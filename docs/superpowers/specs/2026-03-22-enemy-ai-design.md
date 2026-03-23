# Enemy AI — Design Spec (Group 9)

**Date:** 2026-03-22
**Status:** Approved

---

## Overview

Group 9 adds server-authoritative enemy AI to ZoneForge. All enemy state, movement, and combat decisions live on the SpacetimeDB server. Clients subscribe to the `Enemy` table and render what the server dictates — no client controls enemy behaviour.

Security rationale: client-driven AI can be exploited to freeze enemies, prevent attacks, or fabricate kill attribution. Server authority eliminates these attack vectors and ensures future loot/XP systems (Groups 12–13) have trustworthy combat records.

---

## Architecture

### AI Execution Model

A single global `AiTick` scheduled reducer fires every 500ms. Each tick:
1. Builds a definition map (`enemy_def_id → EnemyDefinition`) once.
2. Collects all living players and enemies into Vecs (avoids borrow conflicts during mutation).
3. Runs the state machine for every living enemy.
4. Re-schedules itself for `now + 500ms`.

Spawning is decoupled from the AI tick. `SpawnPoint` rows drive automatic enemy creation; a `RespawnTimer` scheduled table handles post-death respawning. The tick is pure movement and combat.

### Spawning Model (Approach B — SpawnPoint table)

`SpawnPoint` rows are placed by admins. When a spawn point is created it immediately spawns enemies up to `max_count`. When an enemy dies: its row stays alive (`is_dead = true`) for 3 seconds (death animation window), then `EnemyDespawnTimer` fires and deletes it, then schedules a `RespawnTimer` for `respawn_delay_s` seconds later, which recreates the enemy at the spawn coordinates.

`delete_spawn_point` propagates to all associated Enemy rows. Orphaned `RespawnTimer` rows self-cancel by checking whether the spawn point still exists when they fire.

### Admin Gating

`spawn_enemy_manual`, `create_spawn_point`, `delete_spawn_point`, `create_enemy_def`, and `delete_enemy_def` all require the caller's identity to be present in the `Admin` table. `claim_admin` is callable by anyone but only succeeds when the `Admin` table is empty (first-come bootstrap).

---

## Server Schema

### New Enums

```rust
pub enum AiState   { Idle, Chase, Attack }
pub enum EnemyType { Melee, Ranged, Caster }
```

### `Admin` table

| Field    | Type     | Notes          |
|----------|----------|----------------|
| identity | Identity | PK             |

### `EnemyDefinition` table

| Field           | Type       | Notes                          |
|-----------------|------------|--------------------------------|
| id              | u64        | PK, auto_inc                   |
| name            | String     | e.g. "Goblin Scout"            |
| enemy_type      | EnemyType  | Drives AI behaviour branch     |
| prefab_name     | String     | Unity prefab key               |
| max_health      | i32        |                                |
| damage          | i32        | Direct damage (0 for Caster)   |
| aggro_range     | f32        |                                |
| attack_range    | f32        | ~2.5 Melee, ~10 Ranged, ~8 Caster |
| attack_speed_ms | u64        | Cooldown between attacks       |
| move_speed      | f32        | Units/sec                      |

Loot table reference deferred to Group 13.

### `SpawnPoint` table

| Field            | Type  | Notes              |
|------------------|-------|--------------------|
| id               | u64   | PK, auto_inc       |
| zone_id          | u64   | btree index        |
| x, y             | f32   | World position     |
| enemy_def_id     | u64   | References EnemyDefinition |
| max_count        | u32   | Default: 1         |
| respawn_delay_s  | u32   |                    |

### `Enemy` table

| Field             | Type          | Notes                                       |
|-------------------|---------------|---------------------------------------------|
| id                | u64           | PK, auto_inc                                |
| zone_id           | u64           | btree index                                 |
| spawn_point_id    | Option\<u64\> | None = manually spawned, no auto-respawn    |
| enemy_def_id      | u64           |                                             |
| position_x/y      | f32           | Current world position (XZ plane)           |
| home_x/y          | f32           | Copied from SpawnPoint; leash target in Idle |
| health            | i32           | Current only; max_health lives on definition |
| ai_state          | AiState       |                                             |
| target_player_id  | Option\<u64\> |                                             |
| last_attack_us    | u64           | µs since epoch; 0 = never attacked          |
| is_dead           | bool          |                                             |

### Scheduled tables

| Table               | Reducer                  | Notes                        |
|---------------------|--------------------------|------------------------------|
| `AiTick`            | `tick_ai`                | Global, 500ms, self-recurring |
| `EnemyDespawnTimer` | `despawn_dead_enemy`     | Per-enemy, fires 3s after death |
| `RespawnTimer`      | `respawn_at_spawn_point` | Per-spawn-point, fires after respawn_delay_s |

All three follow the standard SpacetimeDB scheduled table pattern (`scheduled_id: u64, scheduled_at: ScheduleAt`, plus payload fields).

### `CombatLog` schema change

Two bool fields added:

| Field             | Type | Notes                                        |
|-------------------|------|----------------------------------------------|
| attacker_is_enemy | bool | true when attacker_id references Enemy table |
| target_is_enemy   | bool | true when target_id references Enemy table   |

Breaking change — requires `--delete-data` on next publish.

---

## Server Reducers

### Admin management
- `claim_admin` — succeeds only when `Admin` table is empty
- `add_admin(identity)` — admin-only
- `remove_admin(identity)` — admin-only

### EnemyDefinition management (admin-only)
- `create_enemy_def(name, enemy_type, prefab_name, max_health, damage, aggro_range, attack_range, attack_speed_ms, move_speed)`
- `delete_enemy_def(def_id)` — rejects if any SpawnPoints reference it

### SpawnPoint management (admin-only)
- `create_spawn_point(zone_id, x, y, enemy_def_id, max_count, respawn_delay_s)` — validates zone + def exist, inserts SpawnPoint, immediately spawns enemies up to max_count
- `delete_spawn_point(sp_id)` — deletes SpawnPoint and kills all associated Enemy rows

### Manual admin overrides
- `spawn_enemy_manual(zone_id, x, y, enemy_def_id)` — no spawn_point_id, no auto-respawn
- `despawn_enemy(enemy_id)` — immediate deletion regardless of health

### Player→Enemy combat
- `attack_enemy(ability_id, enemy_id)` — mirrors `use_ability` validation (caller alive, ability exists, zone match, range check, cooldown, mana) but targets `Enemy` table. Reuses `PlayerCooldown` table. Calls `apply_damage_to_enemy` (internal). On kill: schedules `EnemyDespawnTimer` for 3s. Writes `CombatLog` with `attacker_is_enemy: false, target_is_enemy: true`.
- `use_ability` — unchanged; remains the player→player path

### `tick_ai` (500ms scheduled)

Per-enemy state machine:

```
No player in aggro_range  → Idle:   step toward (home_x, home_y)
Player in aggro_range     → Chase:  step toward player
Player in attack_range    → Attack: hold position; if cooldown elapsed, deal damage
```

Movement step: `move_speed × 0.5s` per tick (simple vector step, no NavMesh — enemies walk through obstacles in Group 9).

Attack branch by `enemy_type`:
- `Melee` / `Ranged`: call `apply_damage_from_enemy` (internal fn; updates Player health, writes CombatLog with `attacker_is_enemy: true, target_is_enemy: false`)
- `Caster`: insert `StatusEffect(Burn, 5s, 8 dmg/tick)` — existing `tick_status_effects` handles DoT

### `despawn_dead_enemy` (scheduled, fires 3s after death)
- If enemy row exists and `is_dead = true`: delete it
- If `spawn_point_id` is set: insert `RespawnTimer` scheduled for `respawn_delay_s` seconds later

### `respawn_at_spawn_point` (scheduled, fires after respawn delay)
- If SpawnPoint deleted: exit silently (self-cancelling)
- Count living enemies with matching `spawn_point_id`; if below `max_count`: insert fresh Enemy row at spawn coordinates, full health, `AiState::Idle`

### `init` / `client_connected` updates
Bootstrap `AiTick` scheduled row if not already present (same pattern as mana regen).

---

## Internal Helper Functions

### `apply_damage_to_enemy(ctx, enemy_id, attacker_id, ability_id, amount)`
- Looks up Enemy; skips if dead
- Clamps health to `[0, current]` (max_health not needed for damage-only path in Group 9)
- If health reaches 0: sets `is_dead = true`, inserts `EnemyDespawnTimer` for 3s
- Writes CombatLog (`attacker_is_enemy: false, target_is_enemy: true`)

### `apply_damage_from_enemy(ctx, player_id, enemy_id, amount)`
- Mirrors existing `apply_damage` but with `attacker_is_enemy: true, target_is_enemy: false` in CombatLog
- Skips if player is dead

### `step_toward(from_x, from_y, to_x, to_y, max_step) → (f32, f32)`
- Pure deterministic helper; snaps to target if within one step

---

## Client

### `SpacetimeDBManager` additions
- Subscribe to `Enemy` and `EnemyDefinition` tables
- Add static events: `OnEnemyInserted`, `OnEnemyUpdated`, `OnEnemyDeleted`
- `CombatLog` autogen bindings pick up `AttackerIsEnemy` / `TargetIsEnemy` automatically after regen

### `EnemyManager` (new singleton)

Mirrors `PlayerManager`:
- `Awake`: subscribe to enemy events + `OnConnected`
- `OnConnected`: backfill existing Enemy rows
- `OnEnemyInserted`: look up EnemyDefinition → instantiate capsule (Melee: orange, Ranged: yellow, Caster: purple) → attach `EnemyController` + health bar → call `CombatManager.RegisterEnemyPosition`
- `OnEnemyUpdated`: forward to `EnemyController`; if `is_dead` just became true, trigger death visual
- `OnEnemyDeleted`: destroy GameObject
- `GetEnemyObject(ulong enemyId)` public accessor

### `EnemyController` (new component)
- Stores last server position; lerps toward it each `Update` (smooths 500ms server ticks)
- On `is_dead = true`: freeze movement, scale capsule to zero over 2.5s (matches 3s despawn window)
- No `NavMeshAgent` — display-only, server owns position

### Enemy health bar
- Reuses `PlayerHealthBar` pattern (world-space canvas above capsule)
- Current health from `Enemy.Health`; max from `EnemyDefinition.MaxHealth`
- Hidden when `is_dead = true`

### `CombatManager` updates
- Add `_enemyPositions: Dictionary<ulong, Vector3>`, populated via new `RegisterEnemyPosition(ulong, Vector3)` method
- Update `OnCombatLogInserted`: use `log.AttackerIsEnemy` / `log.TargetIsEnemy` to route attacker and target position lookups to `_playerPositions` or `_enemyPositions` as appropriate
- VFX and floating number logic unchanged

### `CombatInputHandler` updates
- Tab cycles through all targetable entities: players (from `PlayerManager`) + enemies (from `EnemyManager`)
- Track `_targetIsEnemy: bool` and `_targetEnemyId: ulong` alongside existing `_targetPlayerId`
- On ability key press:
  - `_targetIsEnemy = true` → `Conn.Reducers.AttackEnemy(abilityId, _targetEnemyId)`
  - `_targetIsEnemy = false` → `Conn.Reducers.UseAbility(abilityId, _targetPlayerId)` (unchanged)

---

## Upgrade Path

- **Group 12 (Triggers):** SpawnPoint rows are already in the database; the editor visualises and places them. Trigger actions can reference SpawnPoint ids to enable/disable spawning.
- **Group 13 (Loot):** Add `loot_table_id: Option<u64>` to `EnemyDefinition`; `apply_damage_to_enemy` emits a loot event on kill.
- **NavMesh pathfinding:** Replace `step_toward` in `tick_ai` with a client-side nav request if full obstacle avoidance becomes needed (requires rethinking server authority — deferred).
