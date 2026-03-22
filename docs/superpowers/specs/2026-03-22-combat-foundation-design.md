# Combat Foundation — Design Spec
**Date:** 2026-03-22
**Phase:** 3, Group 7
**Status:** Approved

---

## Overview

Server-authoritative combat with tab-targeting, cooldown-based abilities, status effects, death/respawn, and pooled projectile VFX. Targeting model is tab-target (cycle with Tab key); direction-based is a future extension point. Input is keyboard-only (no mouse required).

---

## Server Schema

### Modified Tables

**`Player`** — three new columns (breaking schema change, requires `--delete-data`):
- `mana: i32` — current mana
- `max_mana: i32` — mana cap (default 100)
- `is_dead: bool` — dead state flag; suppresses movement and ability use

### New Tables

**`Ability`** — static ability definitions (seeded by `init` reducer):

| Field | Type | Notes |
|---|---|---|
| `id` | `u64` PK auto_inc | |
| `name` | `String` | |
| `damage` | `i32` | negative = healing |
| `cooldown_ms` | `u64` | milliseconds |
| `mana_cost` | `i32` | |
| `range` | `f32` | max XZ distance; 0 = self-only (SelfCast) |
| `ability_type` | `AbilityType` | enum: MeleeAttack, Projectile, SelfCast |

`AbilityType` is a `#[derive(SpacetimeType)]` enum — not a table.

**`PlayerCooldown`** — per-player per-ability cooldown tracking.
Uses a composite primary key `(player_id, ability_id)` so upsert is a find-then-update-or-insert:

| Field | Type | Notes |
|---|---|---|
| `player_id` | `u64` PK (part 1) | FK → Player.id |
| `ability_id` | `u64` PK (part 2) | FK → Ability.id |
| `ready_at` | `Timestamp` | next usable time |

Add `#[index(btree)]` on `player_id` for efficient per-player queries.

Upsert pattern in `use_ability`:
```rust
// find by composite PK, update if found, insert if not
if let Some(existing) = ctx.db.player_cooldown().player_id_ability_id().find(&(player_id, ability_id)) {
    ctx.db.player_cooldown().player_id_ability_id().update(PlayerCooldown { ready_at, ..existing });
} else {
    ctx.db.player_cooldown().insert(PlayerCooldown { player_id, ability_id, ready_at });
}
```

Client subscribes to `OnPlayerCooldownInserted` and `OnPlayerCooldownUpdated` (both fire depending on insert vs update path).

**`StatusEffect`** — active effect instances on players:

| Field | Type | Notes |
|---|---|---|
| `id` | `u64` PK auto_inc | |
| `target_id` | `u64` `#[index(btree)]` | FK → Player.id |
| `effect_type` | `StatusEffectType` | enum: Burn, Freeze, Stun, Poison |
| `expires_at` | `Timestamp` | |
| `damage_per_tick` | `i32` | 0 for non-DoT effects (Freeze, Stun) |

`StatusEffectType` is a `#[derive(SpacetimeType)]` enum.

**`CombatLog`** — immutable combat history (grows unbounded in Group 7; archival is a future concern):

| Field | Type | Notes |
|---|---|---|
| `id` | `u64` PK auto_inc | |
| `timestamp` | `Timestamp` | `ctx.timestamp` |
| `attacker_id` | `u64` | FK → Player.id |
| `target_id` | `u64` | FK → Player.id |
| `ability_id` | `u64` | FK → Ability.id |
| `damage_dealt` | `i32` | actual damage after clamping (negative = heal) |
| `overkill` | `i32` | excess damage beyond 0 HP; always 0 for heals |

**`StatusEffectTick`** — scheduled table driving the 1-second DoT tick:

| Field | Type | Notes |
|---|---|---|
| `scheduled_id` | `u64` PK auto_inc | |
| `scheduled_at` | `ScheduleAt` | |

```rust
#[table(accessor = status_effect_tick, scheduled(tick_status_effects))]
pub struct StatusEffectTick {
    #[primary_key]
    #[auto_inc]
    pub scheduled_id: u64,
    pub scheduled_at: ScheduleAt,
}
```

The `init` reducer inserts one `StatusEffectTick` row with `ScheduleAt::Time(ctx.timestamp + Duration::from_secs(1))`. At the end of each `tick_status_effects` invocation, the reducer re-inserts a new row with `ScheduleAt::Time(ctx.timestamp + Duration::from_secs(1))` to keep the tick recurring.

---

## Server Reducers

### `init` (lifecycle hook)
Runs once on first publish. Seeds 3 abilities if `Ability` table is empty, and inserts the first `StatusEffectTick` row:

| # | Name | Type | Damage | Cooldown | Mana | Range |
|---|---|---|---|---|---|---|
| 1 | Auto-Attack | MeleeAttack | 20 | 500ms | 0 | 2.5 |
| 2 | Fireball | Projectile | 50 | 3000ms | 20 | 15.0 |
| 3 | Heal | SelfCast | -50 | 10000ms | 30 | 0.0 |

### `use_ability(ability_id: u64, target_id: u64)`
Validation (returns `Err` on failure):
1. Caller player exists and `!is_dead`
2. `Ability` row exists
3. If `ability.ability_type == SelfCast`: verify `target_id == caller.id`; reject otherwise
4. If `ability.ability_type != SelfCast`: target player exists and `!is_dead`; XZ distance ≤ `ability.range`
5. `PlayerCooldown` row for `(caller.id, ability_id)` either doesn't exist or `ready_at <= ctx.timestamp`
6. `caller.mana >= ability.mana_cost`

On success:
- Deduct `ability.mana_cost` from caller mana
- Upsert `PlayerCooldown` with `ready_at = ctx.timestamp + Duration::from_millis(ability.cooldown_ms)`
- Call `apply_damage(target_id, caller.id, ability_id, ability.damage)`

### `apply_damage(target_id: u64, attacker_id: u64, ability_id: u64, amount: i32)`
Called from `use_ability` (and future reducers: enemy attacks, traps, AoE). Not dispatched via SpacetimeDB reducer mechanism — called as a direct Rust function from other reducers.

- Find target player row (return early if not found)
- `new_health = (target.health - amount).clamp(0, target.max_health)`
- `overkill = if amount > 0 { max(0, amount - target.health) } else { 0 }`
- Update player: `health = new_health`
- Write `CombatLog` row with `damage_dealt = amount`, `overkill`
- If `new_health == 0` and `amount > 0`: set `is_dead = true`

### `respawn()`
- Find caller's player row; return `Err("not dead")` if `!is_dead`
- Look up zone by `player.zone_id`; return `Err("zone not found")` if missing
- Reset: `health = max_health`, `mana = max_mana`, `is_dead = false`
- Warp: `position_x = zone.terrain_width as f32 / 2.0`, `position_y = zone.terrain_height as f32 / 2.0`
- Delete all `StatusEffect` rows where `target_id == player.id`

### `tick_status_effects` (scheduled via `StatusEffectTick`)
- Iterate all `StatusEffect` rows
- Remove rows where `expires_at <= ctx.timestamp`
- For remaining Burn/Poison rows: call `apply_damage(row.target_id, row.target_id, 0, row.damage_per_tick)` (self-inflicted, ability_id 0 = DoT)
- Re-insert a new `StatusEffectTick` row with `ScheduleAt::Time(ctx.timestamp + Duration::from_secs(1))` to keep recurring

---

## Client Architecture

### New Scripts

**`CombatInputHandler.cs`** (attach to PlayerManager GO or dedicated Combat GO)
- Maintains `_selectedTargetId: ulong` (player ID, 0 = none)
- Tab: cycles through non-local, non-dead players from `Conn.Db.Player.Iter()`; wraps; places a flat cylinder "selection ring" under the target capsule; removes ring when target dies or deselects
- Key 1: Auto-Attack — `Reducers.UseAbility(conn, 1, _selectedTargetId)`
- Key 2: Fireball — `Reducers.UseAbility(conn, 2, _selectedTargetId)`
- Key 3: Heal — `Reducers.UseAbility(conn, 3, localPlayerId)` (always self, ignores selected target)
- R: `Reducers.Respawn(conn)` — only when local player `is_dead`
- Guards: skip ability keys if local player is dead; skip 1/2 if no target selected; skip if ability on cooldown (check `PlayerCooldown` cache via `Conn.Db.PlayerCooldown.Iter()`)

**`ZoneForgePoolManager.cs`** (singleton, DontDestroyOnLoad)
- `Dictionary<string, Queue<GameObject>>` pools, pre-allocated in `Awake`
- `Get(key)` / `Return(key, go)` public API
- Initial pools: `projectile_fireball` (cap 20), `vfx_impact_fire` (cap 15), `vfx_impact_generic` (cap 20)
- Pool config via `[Serializable] PoolDefinition` inspector list (key, prefab, initial capacity, max size)

**`PooledProjectile.cs`** (attach to projectile prefabs, requires Rigidbody)
- On `OnEnable`: cache `Rigidbody`, start `maxLifetime` timeout coroutine
- On `OnCollisionEnter`: spawn impact VFX from pool at contact point, self-return
- On timeout: self-return without VFX
- `ZoneForgePoolManager.Instance.Return(poolKey, gameObject)` resets velocity before re-queuing

**`PooledVFX.cs`** (attach to VFX prefabs, requires ParticleSystem)
- On `OnEnable`: start coroutine `WaitUntil(!ps.IsAlive(true))`, then `ZoneForgePoolManager.Instance.Return(poolKey, gameObject)`

**`CombatManager.cs`** (singleton)
- Subscribes to `SpacetimeDBManager.OnCombatLogInserted`
- On new CombatLog row:
  1. Look up `Ability` row by `combatLog.ability_id` from `Conn.Db.Ability`; if null (e.g. DoT ticks use `ability_id = 0`), fall back to `vfx_impact_generic` at target position and return early
  2. If `ability.ability_type == AbilityType.Projectile`: get projectile from pool, position at attacker's current world pos, aim toward target's current world pos, apply forward velocity
  3. If `MeleeAttack` or `SelfCast`: spawn `vfx_impact_generic` at target position (instant, no travel)
- Subscribes to `SpacetimeDBManager.OnPlayerUpdated`:
  - Local player `is_dead` transitions to `true` → show "Press R to respawn" screen overlay
  - Local player `is_dead` transitions to `false` → hide overlay

### SpacetimeDBManager Changes
Add to `Subscribe` query list:
```csharp
"SELECT * FROM ability",
"SELECT * FROM player_cooldown",
"SELECT * FROM status_effect",
"SELECT * FROM combat_log"
```

Expose static events:
- `OnCombatLogInserted(CombatLog)`
- `OnAbilityInserted(Ability)`
- `OnPlayerCooldownInserted(PlayerCooldown)`
- `OnPlayerCooldownUpdated(PlayerCooldown, PlayerCooldown)`
- `OnStatusEffectInserted(StatusEffect)`
- `OnStatusEffectDeleted(StatusEffect)`

### Prefabs Required
- `FireballProjectile` — sphere primitive, `PooledProjectile` + Rigidbody (gravity disabled), optional trail renderer; pool key `projectile_fireball`
- `ImpactFireVFX` — particle system, `PooledVFX`; pool key `vfx_impact_fire`
- `ImpactGenericVFX` — particle system, `PooledVFX`; pool key `vfx_impact_generic`

---

## Data Flow: Player Uses Fireball

1. Player presses Tab → `_selectedTargetId` set, selection ring placed under target capsule
2. Player presses 2 → `CombatInputHandler` checks cooldown cache and mana, calls `Reducers.UseAbility(conn, 2, targetId)`
3. Server `use_ability`: validates range/cooldown/mana, deducts mana, upserts `PlayerCooldown`, calls `apply_damage`
4. `apply_damage`: reduces target health, writes `CombatLog`, sets `is_dead = true` if health reaches 0
5. All clients receive `CombatLog.OnInsert` → `CombatManager` looks up Ability (Projectile type) → spawns fireball from pool, fires toward target
6. Projectile hits (or times out) → `PooledProjectile` spawns `vfx_impact_fire` from pool, self-returns
7. All clients receive `Player.OnUpdate` with target's new health
8. If `is_dead = true`: target client's `CombatManager` shows "Press R to respawn" overlay

## Data Flow: Player Heals Self

1. Player presses 3 → `CombatInputHandler` calls `Reducers.UseAbility(conn, 3, localPlayerId)`
2. Server `use_ability`: confirms `target_id == caller.id` (SelfCast validation), deducts mana, upserts `PlayerCooldown`, calls `apply_damage` with `amount = -50`
3. `apply_damage`: `new_health = clamp(health + 50, 0, max_health)`, writes `CombatLog` with negative `damage_dealt`, `overkill = 0`
4. All clients receive `CombatLog.OnInsert` → `CombatManager` sees `SelfCast` → spawns `vfx_impact_generic` at caster position

---

## Deployment Notes

- Adding `mana`, `max_mana`, `is_dead` to `Player` is a **breaking schema change** — publish requires `--delete-data`
- `tick_status_effects` scheduled reducer is self-recurring via re-insert pattern; seeded in `init`
- Bindings must be regenerated after publish; Unity Reimport All required

---

## Out of Scope (Group 8+)

- Hotbar UI, cooldown ring indicators, health/mana bars
- Floating damage numbers
- AoE abilities (Ice Nova)
- Direction-based targeting
- Status effect *application* from abilities (tables defined and ticked, but `use_ability` does not apply effects in Group 7)
- Enemy AI (Group 9)
- CombatLog archival/TTL
