# Enemy AI — Server Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add all server-side tables, helper functions, and reducers for the Group 9 enemy AI system.

**Architecture:** All changes go into `server/spacetimedb/src/lib.rs`. A global `AiTick` scheduled reducer drives the Idle/Chase/Attack state machine every 500ms. SpawnPoints drive automatic enemy creation; death cleanup and respawn use their own per-enemy/per-spawn-point scheduled tables. Admin identities are compile-time constants seeded in `init`.

**Tech Stack:** Rust, SpacetimeDB 2.0.3 WASM module (`spacetimedb` crate), `spacetime` CLI

**Spec:** `docs/superpowers/specs/2026-03-22-enemy-ai-design.md`

> ⚠️ **Breaking schema change in Task 2** — the final publish (Task 12) requires `--delete-data`. Do **not** publish mid-plan without that flag; run `spacetime build` only to check for compile errors.

---

## File Map

| File | Change |
|------|--------|
| `server/spacetimedb/src/lib.rs` | All changes — enums, tables, reducers, helpers |

---

## Task 1: Enums + Admin table + compile-time identity seeding

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Add `AiState` and `EnemyType` enums** after the existing `StatusEffectType` enum (~line 16):

```rust
#[derive(SpacetimeType, Clone, Debug, PartialEq)]
pub enum AiState {
    Idle,
    Chase,
    Attack,
}

#[derive(SpacetimeType, Clone, Debug, PartialEq)]
pub enum EnemyType {
    Melee,
    Ranged,
    Caster,
}
```

- [ ] **Add `Admin` table** after the existing `Player` table:

```rust
// Admin table — identity-keyed, no auto_inc.
// Admin set is defined at compile time in ADMIN_IDENTITIES below.
#[table(accessor = admin, public)]
pub struct Admin {
    #[primary_key]
    pub identity: Identity,
}
```

- [ ] **Add `ADMIN_IDENTITIES` constant and helpers** after the `Admin` struct:

```rust
// Run `spacetime identity list` to get your identity hex (64-char string marked with *).
// Add each admin identity here before publishing. Changes require a republish.
const ADMIN_IDENTITIES: &[&str] = &[
    // "0x<your-64-char-hex-identity-here>",
];

fn identity_from_hex(hex: &str) -> Identity {
    let hex = hex.trim_start_matches("0x");
    assert!(hex.len() == 64, "Admin identity hex must be 64 characters (32 bytes)");
    let mut bytes = [0u8; 32];
    for i in 0..32 {
        bytes[i] = u8::from_str_radix(&hex[i * 2..i * 2 + 2], 16)
            .expect("ADMIN_IDENTITIES contains non-hex characters");
    }
    Identity::from_be_byte_array(bytes)
}

fn is_admin(ctx: &ReducerContext) -> bool {
    ctx.db.admin().identity().find(ctx.sender()).is_some()
}
```

> **Note on `Identity::from_byte_array`:** If this method doesn't exist in your `spacetimedb` crate version, check `cargo doc --open` for available constructors. Common alternatives: `Identity::from_bytes(&bytes)`, or `unsafe { std::mem::transmute(bytes) }` as a last resort. The identity is 32 bytes.

- [ ] **Seed `Admin` table in `init`** — add as the first block inside the `init` reducer (before ability seeding):

```rust
// Seed compile-time admin identities
for &hex in ADMIN_IDENTITIES {
    ctx.db.admin().insert(Admin { identity: identity_from_hex(hex) });
}
log::info!("init: seeded {} admin identity(ies)", ADMIN_IDENTITIES.len());
```

- [ ] **Build to verify compile:** `cd server && spacetime build`
  - Expected: zero errors. A warning about an empty `ADMIN_IDENTITIES` slice is fine.

- [ ] **Commit:**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add AiState/EnemyType enums, Admin table, compile-time admin seeding"
```

---

## Task 2: Update CombatLog schema

Adds `attacker_is_enemy` and `target_is_enemy` bool fields. Also updates the existing `apply_damage` internal fn to include both fields as `false` (all current calls are player-to-player).

**Files:**
- Modify: `server/spacetimedb/src/lib.rs` — `CombatLog` struct and `apply_damage` fn

- [ ] **Replace the `CombatLog` struct** (~line 104):

```rust
#[table(accessor = combat_log, public)]
pub struct CombatLog {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub timestamp: Timestamp,
    pub attacker_id: u64,
    pub target_id: u64,
    pub ability_id: u64,
    pub damage_dealt: i32,
    pub overkill: i32,
    pub attacker_is_enemy: bool,
    pub target_is_enemy: bool,
}
```

- [ ] **Update the `CombatLog` insert inside `apply_damage`** (~line 286):

```rust
ctx.db.combat_log().insert(CombatLog {
    id: 0,
    timestamp: ctx.timestamp,
    attacker_id,
    target_id,
    ability_id,
    damage_dealt: amount,
    overkill,
    attacker_is_enemy: false,
    target_is_enemy: false,
});
```

- [ ] **Build:** `cd server && spacetime build`

- [ ] **Commit:**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add attacker_is_enemy/target_is_enemy to CombatLog (breaking schema)"
```

---

## Task 3: EnemyDefinition table + create/delete reducers

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Add `EnemyDefinition` table** after the `CombatLog` table:

```rust
#[table(accessor = enemy_def, public)]
pub struct EnemyDefinition {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub name: String,
    pub enemy_type: EnemyType,
    pub prefab_name: String,
    pub max_health: i32,
    pub damage: i32,
    pub aggro_range: f32,
    pub attack_range: f32,
    pub attack_speed_ms: u64,
    pub move_speed: f32,
}
```

- [ ] **Add `create_enemy_def` reducer** (after the existing `spawn_entity` reducer):

```rust
#[reducer]
pub fn create_enemy_def(
    ctx: &ReducerContext,
    name: String,
    enemy_type: EnemyType,
    prefab_name: String,
    max_health: i32,
    damage: i32,
    aggro_range: f32,
    attack_range: f32,
    attack_speed_ms: u64,
    move_speed: f32,
) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized".to_string()); }
    if name.is_empty() { return Err("name cannot be empty".to_string()); }
    if max_health <= 0 { return Err("max_health must be > 0".to_string()); }
    ctx.db.enemy_def().insert(EnemyDefinition {
        id: 0, name, enemy_type, prefab_name,
        max_health, damage, aggro_range, attack_range, attack_speed_ms, move_speed,
    });
    Ok(())
}
```

- [ ] **Add `delete_enemy_def` reducer** immediately after:

```rust
#[reducer]
pub fn delete_enemy_def(ctx: &ReducerContext, def_id: u64) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized".to_string()); }
    ctx.db.enemy_def().id().find(&def_id).ok_or("EnemyDefinition not found")?;
    let in_use = ctx.db.spawn_point().iter().any(|sp| sp.enemy_def_id == def_id);
    if in_use {
        return Err("EnemyDefinition is referenced by one or more SpawnPoints".to_string());
    }
    ctx.db.enemy_def().id().delete(&def_id);
    Ok(())
}
```

> Note: `ctx.db.spawn_point()` won't exist until Task 4. The compiler will error — add a `// TODO: uncomment after Task 4` comment on the `in_use` line and temporarily replace the body with just the admin check + delete. Restore the `in_use` check in Task 4.

- [ ] **Build:** `cd server && spacetime build`

- [ ] **Commit:**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add EnemyDefinition table, create/delete reducers"
```

---

## Task 4: SpawnPoint table + create/delete reducers

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Add `SpawnPoint` table** after `EnemyDefinition`:

```rust
#[table(accessor = spawn_point, public)]
pub struct SpawnPoint {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    #[index(btree)]
    pub zone_id: u64,
    pub x: f32,
    pub y: f32,
    pub enemy_def_id: u64,
    pub max_count: u32,
    pub respawn_delay_s: u32,
}
```

- [ ] **Add `create_spawn_point` reducer:**

```rust
#[reducer]
pub fn create_spawn_point(
    ctx: &ReducerContext,
    zone_id: u64,
    x: f32,
    y: f32,
    enemy_def_id: u64,
    max_count: u32,
    respawn_delay_s: u32,
) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized".to_string()); }
    ctx.db.zone().id().find(&zone_id).ok_or("Zone not found")?;
    let def = ctx.db.enemy_def().id().find(&enemy_def_id)
        .ok_or("EnemyDefinition not found")?;
    let sp = ctx.db.spawn_point().insert(SpawnPoint {
        id: 0, zone_id, x, y, enemy_def_id, max_count, respawn_delay_s,
    });
    // Immediately spawn enemies up to max_count
    // TODO: uncomment after Task 5 adds spawn_enemy_at
    // for _ in 0..max_count { spawn_enemy_at(ctx, &sp, &def); }
    let _ = (sp, def); // suppress unused warnings until Task 5
    Ok(())
}
```

- [ ] **Add `delete_spawn_point` reducer:**

```rust
#[reducer]
pub fn delete_spawn_point(ctx: &ReducerContext, sp_id: u64) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized".to_string()); }
    ctx.db.spawn_point().id().find(&sp_id).ok_or("SpawnPoint not found")?;
    // Kill all Enemy rows associated with this spawn point
    // TODO: uncomment after Task 5 adds Enemy table
    // let ids: Vec<u64> = ctx.db.enemy().iter()
    //     .filter(|e| e.spawn_point_id == Some(sp_id))
    //     .map(|e| e.id).collect();
    // for id in ids { ctx.db.enemy().id().delete(&id); }
    ctx.db.spawn_point().id().delete(&sp_id);
    Ok(())
}
```

- [ ] **Restore the `in_use` check in `delete_enemy_def`** (from Task 3 TODO):

```rust
let in_use = ctx.db.spawn_point().iter().any(|sp| sp.enemy_def_id == def_id);
if in_use {
    return Err("EnemyDefinition is referenced by one or more SpawnPoints".to_string());
}
```

- [ ] **Build:** `cd server && spacetime build`

- [ ] **Commit:**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add SpawnPoint table, create/delete reducers"
```

---

## Task 5: Enemy table + scheduled table structs + spawn_enemy_at helper

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Add `Enemy` table** after `SpawnPoint`:

```rust
#[table(accessor = enemy, public)]
pub struct Enemy {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    #[index(btree)]
    pub zone_id: u64,
    pub spawn_point_id: Option<u64>,
    pub enemy_def_id: u64,
    pub position_x: f32,
    pub position_y: f32,
    pub home_x: f32,
    pub home_y: f32,
    pub health: i32,
    pub ai_state: AiState,
    pub target_player_id: Option<u64>,
    pub last_attack_us: u64,
    pub is_dead: bool,
}
```

- [ ] **Add the three scheduled table structs** after `ManaRegenTick`:

```rust
#[table(accessor = ai_tick, scheduled(tick_ai))]
pub struct AiTick {
    #[primary_key]
    #[auto_inc]
    pub scheduled_id: u64,
    pub scheduled_at: ScheduleAt,
}

#[table(accessor = enemy_despawn_timer, scheduled(despawn_dead_enemy))]
pub struct EnemyDespawnTimer {
    #[primary_key]
    #[auto_inc]
    pub scheduled_id: u64,
    pub scheduled_at: ScheduleAt,
    pub enemy_id: u64,
}

#[table(accessor = respawn_timer, scheduled(respawn_at_spawn_point))]
pub struct RespawnTimer {
    #[primary_key]
    #[auto_inc]
    pub scheduled_id: u64,
    pub scheduled_at: ScheduleAt,
    pub spawn_point_id: u64,
}
```

- [ ] **Add `spawn_enemy_at` internal helper** (not a reducer — no `#[reducer]`) after the table definitions:

```rust
fn spawn_enemy_at(ctx: &ReducerContext, sp: &SpawnPoint, def: &EnemyDefinition) {
    ctx.db.enemy().insert(Enemy {
        id: 0,
        zone_id: sp.zone_id,
        spawn_point_id: Some(sp.id),
        enemy_def_id: sp.enemy_def_id,
        position_x: sp.x,
        position_y: sp.y,
        home_x: sp.x,
        home_y: sp.y,
        health: def.max_health,
        ai_state: AiState::Idle,
        target_player_id: None,
        last_attack_us: 0,
        is_dead: false,
    });
}
```

- [ ] **Uncomment the TODO blocks** from Tasks 3 and 4:
  - In `create_spawn_point`: replace the placeholder `let _ = (sp, def);` with `for _ in 0..max_count { spawn_enemy_at(ctx, &sp, &def); }`
  - In `delete_spawn_point`: uncomment the enemy cleanup block

- [ ] **Build:** `cd server && spacetime build`

- [ ] **Commit:**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add Enemy table, scheduled table structs, spawn_enemy_at helper"
```

---

## Task 6: Helper functions — step_toward, apply_damage_to_enemy, apply_damage_from_enemy

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

Add all three after the existing `apply_damage` function.

- [ ] **Add `step_toward`:**

```rust
/// Moves (from_x, from_y) toward (to_x, to_y) by at most max_step units.
/// Snaps to target if within one step to avoid overshooting.
fn step_toward(from_x: f32, from_y: f32, to_x: f32, to_y: f32, max_step: f32) -> (f32, f32) {
    let dx = to_x - from_x;
    let dy = to_y - from_y;
    let dist = (dx * dx + dy * dy).sqrt();
    if dist < 0.01 || dist <= max_step {
        (to_x, to_y)
    } else {
        (from_x + dx / dist * max_step, from_y + dy / dist * max_step)
    }
}
```

- [ ] **Add `apply_damage_to_enemy`:**

```rust
/// Applies damage from a player to an enemy. Writes CombatLog. Schedules despawn on kill.
fn apply_damage_to_enemy(
    ctx: &ReducerContext,
    enemy_id: u64,
    attacker_id: u64,
    ability_id: u64,
    amount: i32,
) {
    let Some(enemy) = ctx.db.enemy().id().find(&enemy_id) else { return; };
    if enemy.is_dead { return; }

    let new_health = (enemy.health - amount).max(0);
    let overkill = if amount > enemy.health { amount - enemy.health } else { 0 };
    let now_dead = new_health == 0 && amount > 0;

    ctx.db.enemy().id().update(Enemy {
        health: new_health,
        is_dead: now_dead,
        ..enemy
    });

    ctx.db.combat_log().insert(CombatLog {
        id: 0,
        timestamp: ctx.timestamp,
        attacker_id,
        target_id: enemy_id,
        ability_id,
        damage_dealt: amount,
        overkill,
        attacker_is_enemy: false,
        target_is_enemy: true,
    });

    if now_dead {
        ctx.db.enemy_despawn_timer().insert(EnemyDespawnTimer {
            scheduled_id: 0,
            scheduled_at: ScheduleAt::Time(
                ctx.timestamp + std::time::Duration::from_secs(3)
            ),
            enemy_id,
        });
        log::info!("apply_damage_to_enemy: enemy={} killed by {}", enemy_id, attacker_id);
    }
}
```

- [ ] **Add `apply_damage_from_enemy`:**

```rust
/// Applies damage from an enemy to a player. Writes CombatLog.
fn apply_damage_from_enemy(
    ctx: &ReducerContext,
    player_id: u64,
    enemy_id: u64,
    amount: i32,
) {
    let Some(target) = ctx.db.player().id().find(&player_id) else { return; };
    if target.is_dead { return; }

    let new_health = (target.health - amount).clamp(0, target.max_health);
    let overkill = if amount > 0 && amount > target.health { amount - target.health } else { 0 };
    let new_is_dead = new_health == 0 && amount > 0;

    ctx.db.player().id().update(Player {
        health: new_health,
        is_dead: new_is_dead,
        ..target
    });

    ctx.db.combat_log().insert(CombatLog {
        id: 0,
        timestamp: ctx.timestamp,
        attacker_id: enemy_id,
        target_id: player_id,
        ability_id: 0,
        damage_dealt: amount,
        overkill,
        attacker_is_enemy: true,
        target_is_enemy: false,
    });

    log::info!(
        "apply_damage_from_enemy: enemy={} hit player={} for {} hp",
        enemy_id, player_id, amount
    );
}
```

- [ ] **Build:** `cd server && spacetime build`

- [ ] **Commit:**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add step_toward, apply_damage_to_enemy, apply_damage_from_enemy"
```

---

## Task 7: attack_enemy reducer

Player→enemy combat path. Mirrors `use_ability` validation but targets the `Enemy` table.

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Add `attack_enemy` reducer** immediately after `use_ability`:

```rust
#[reducer]
pub fn attack_enemy(
    ctx: &ReducerContext,
    ability_id: u64,
    enemy_id: u64,
) -> Result<(), String> {
    // 1. Caller must exist and not be dead
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or("Player not found")?;
    if player.is_dead {
        return Err("Cannot use ability while dead".to_string());
    }

    // 2. Ability must exist; self-cast abilities cannot target enemies
    let ability = ctx.db.ability().id().find(&ability_id)
        .ok_or("Ability not found")?;
    if ability.ability_type == AbilityType::SelfCast {
        return Err("Self-cast abilities cannot target enemies".to_string());
    }

    // 3. Enemy must exist and not be dead
    let enemy = ctx.db.enemy().id().find(&enemy_id)
        .ok_or("Enemy not found")?;
    if enemy.is_dead {
        return Err("Enemy is already dead".to_string());
    }

    // 4. Same zone
    if enemy.zone_id != player.zone_id {
        return Err("Enemy is in a different zone".to_string());
    }

    // 5. Range check (XZ plane)
    let dx = player.position_x - enemy.position_x;
    let dz = player.position_y - enemy.position_y;
    if dx * dx + dz * dz > ability.range * ability.range {
        return Err(format!(
            "Enemy out of range (dist={:.1}, range={:.1})",
            (dx * dx + dz * dz).sqrt(), ability.range
        ));
    }

    // 6. Cooldown check (reuses PlayerCooldown table)
    let now_us = ctx.timestamp
        .to_duration_since_unix_epoch()
        .unwrap_or_default()
        .as_micros();
    let on_cooldown = ctx.db.player_cooldown()
        .player_id()
        .filter(&player.id)
        .any(|cd| {
            if cd.ability_id != ability_id { return false; }
            let ready_us = cd.ready_at
                .to_duration_since_unix_epoch()
                .unwrap_or_default()
                .as_micros();
            ready_us > now_us
        });
    if on_cooldown { return Err("Ability on cooldown".to_string()); }

    // 7. Mana check
    if player.mana < ability.mana_cost {
        return Err(format!("Insufficient mana ({}/{})", player.mana, ability.mana_cost));
    }

    // All checks passed — consume mana, record cooldown
    let player_id = player.id;
    let new_mana = player.mana - ability.mana_cost;
    ctx.db.player().id().update(Player { mana: new_mana, ..player });

    let ready_at = ctx.timestamp + std::time::Duration::from_millis(ability.cooldown_ms);
    if let Some(existing) = ctx.db.player_cooldown()
        .player_id()
        .filter(&player_id)
        .find(|cd| cd.ability_id == ability_id)
    {
        ctx.db.player_cooldown().id().update(PlayerCooldown { ready_at, ..existing });
    } else {
        ctx.db.player_cooldown().insert(PlayerCooldown {
            id: 0, player_id, ability_id, ready_at,
        });
    }

    apply_damage_to_enemy(ctx, enemy_id, player_id, ability_id, ability.damage);
    log::info!("attack_enemy: player={} ability={} enemy={}", player_id, ability_id, enemy_id);
    Ok(())
}
```

- [ ] **Build:** `cd server && spacetime build`

- [ ] **Commit:**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add attack_enemy reducer (player->enemy combat)"
```

---

## Task 8: spawn_enemy_manual + despawn_enemy admin reducers

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Add `spawn_enemy_manual` reducer:**

```rust
#[reducer]
pub fn spawn_enemy_manual(
    ctx: &ReducerContext,
    zone_id: u64,
    x: f32,
    y: f32,
    enemy_def_id: u64,
) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized".to_string()); }
    ctx.db.zone().id().find(&zone_id).ok_or("Zone not found")?;
    let def = ctx.db.enemy_def().id().find(&enemy_def_id)
        .ok_or("EnemyDefinition not found")?;
    ctx.db.enemy().insert(Enemy {
        id: 0,
        zone_id,
        spawn_point_id: None,  // no auto-respawn for manually spawned enemies
        enemy_def_id,
        position_x: x,
        position_y: y,
        home_x: x,
        home_y: y,
        health: def.max_health,
        ai_state: AiState::Idle,
        target_player_id: None,
        last_attack_us: 0,
        is_dead: false,
    });
    Ok(())
}
```

- [ ] **Add `despawn_enemy` reducer:**

```rust
#[reducer]
pub fn despawn_enemy(ctx: &ReducerContext, enemy_id: u64) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized".to_string()); }
    ctx.db.enemy().id().find(&enemy_id).ok_or("Enemy not found")?;
    ctx.db.enemy().id().delete(&enemy_id);
    Ok(())
}
```

- [ ] **Build:** `cd server && spacetime build`

- [ ] **Commit:**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add spawn_enemy_manual and despawn_enemy admin reducers"
```

---

## Task 9: tick_ai scheduled reducer

The core 500ms AI loop. State machine: Idle → Chase → Attack.

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Add constants before the `tick_ai` reducer:**

```rust
const AI_TICK_INTERVAL_MS: u64 = 500;
const AI_TICK_INTERVAL_S: f32 = AI_TICK_INTERVAL_MS as f32 / 1000.0;
```

- [ ] **Add `tick_ai` reducer:**

```rust
#[reducer]
pub fn tick_ai(ctx: &ReducerContext, _tick: AiTick) {
    use std::collections::HashMap;

    // Build definition lookup map once per tick
    let defs: HashMap<u64, EnemyDefinition> = ctx.db.enemy_def()
        .iter()
        .map(|d| (d.id, d))
        .collect();

    // Snapshot living players — no btree index on Player.zone_id at Group 9 scale;
    // full scan + Rust filter is acceptable (expected single-digit concurrent players).
    let players: Vec<Player> = ctx.db.player()
        .iter()
        .filter(|p| !p.is_dead)
        .collect();

    // Snapshot living enemies — collect before mutating to avoid borrow conflicts
    let enemies: Vec<Enemy> = ctx.db.enemy()
        .iter()
        .filter(|e| !e.is_dead)
        .collect();

    let now_us = ctx.timestamp
        .to_duration_since_unix_epoch()
        .unwrap_or_default()
        .as_micros() as u64;

    for enemy in enemies {
        let Some(def) = defs.get(&enemy.enemy_def_id) else { continue; };
        let step = def.move_speed * AI_TICK_INTERVAL_S;

        // Find nearest living player in the same zone within aggro range
        let nearest = players
            .iter()
            .filter(|p| p.zone_id == enemy.zone_id)
            .map(|p| {
                let dx = p.position_x - enemy.position_x;
                let dz = p.position_y - enemy.position_y;
                (p, dx * dx + dz * dz)
            })
            .filter(|(_, dist_sq)| *dist_sq <= def.aggro_range * def.aggro_range)
            .min_by(|a, b| a.1.partial_cmp(&b.1).unwrap_or(std::cmp::Ordering::Equal));

        let (new_state, new_target, new_x, new_y, new_last_attack, attack_event) = match nearest {
            None => {
                // Idle — walk back toward home position
                let (nx, ny) = step_toward(
                    enemy.position_x, enemy.position_y,
                    enemy.home_x, enemy.home_y,
                    step,
                );
                (AiState::Idle, None, nx, ny, enemy.last_attack_us, None)
            }
            Some((target, dist_sq)) => {
                if dist_sq <= def.attack_range * def.attack_range {
                    // Attack — hold position; deal damage if attack cooldown has elapsed
                    let ready_us = enemy.last_attack_us + def.attack_speed_ms * 1000;
                    let can_attack = now_us >= ready_us;
                    let new_last = if can_attack { now_us } else { enemy.last_attack_us };
                    let evt = if can_attack {
                        Some((target.id, def.damage, def.enemy_type.clone()))
                    } else {
                        None
                    };
                    (AiState::Attack, Some(target.id),
                     enemy.position_x, enemy.position_y, new_last, evt)
                } else {
                    // Chase — step toward player
                    let (nx, ny) = step_toward(
                        enemy.position_x, enemy.position_y,
                        target.position_x, target.position_y,
                        step,
                    );
                    (AiState::Chase, Some(target.id), nx, ny, enemy.last_attack_us, None)
                }
            }
        };

        let enemy_id = enemy.id;
        ctx.db.enemy().id().update(Enemy {
            ai_state: new_state,
            target_player_id: new_target,
            position_x: new_x,
            position_y: new_y,
            last_attack_us: new_last_attack,
            ..enemy
        });

        if let Some((player_id, damage, enemy_type)) = attack_event {
            match enemy_type {
                EnemyType::Caster => {
                    // Caster: apply Burn status effect instead of direct damage.
                    // StatusEffect.target_id always references a Player in Group 9 —
                    // tick_status_effects calls apply_damage which looks up Player only.
                    ctx.db.status_effect().insert(StatusEffect {
                        id: 0,
                        target_id: player_id,
                        effect_type: StatusEffectType::Burn,
                        expires_at: ctx.timestamp + std::time::Duration::from_secs(5),
                        damage_per_tick: 8,
                    });
                    log::info!("tick_ai: caster enemy={} applied Burn to player={}", enemy_id, player_id);
                }
                _ => {
                    apply_damage_from_enemy(ctx, player_id, enemy_id, damage);
                }
            }
        }
    }

    // Re-schedule self for next tick
    ctx.db.ai_tick().insert(AiTick {
        scheduled_id: 0,
        scheduled_at: ScheduleAt::Time(
            ctx.timestamp + std::time::Duration::from_millis(AI_TICK_INTERVAL_MS)
        ),
    });
}
```

- [ ] **Build:** `cd server && spacetime build`
  - If the compiler complains about `use std::collections::HashMap` inside a function, move it to the top of the file with the other `use` statements.

- [ ] **Commit:**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add tick_ai scheduled reducer (500ms AI state machine)"
```

---

## Task 10: despawn_dead_enemy + respawn_at_spawn_point reducers

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Add `despawn_dead_enemy` reducer:**

```rust
#[reducer]
pub fn despawn_dead_enemy(ctx: &ReducerContext, timer: EnemyDespawnTimer) {
    let Some(enemy) = ctx.db.enemy().id().find(&timer.enemy_id) else { return; };
    if !enemy.is_dead { return; } // Recovered somehow — skip

    let spawn_point_id = enemy.spawn_point_id;
    ctx.db.enemy().id().delete(&timer.enemy_id);

    // If this enemy came from a spawn point, schedule a respawn
    if let Some(sp_id) = spawn_point_id {
        let Some(sp) = ctx.db.spawn_point().id().find(&sp_id) else { return; };
        ctx.db.respawn_timer().insert(RespawnTimer {
            scheduled_id: 0,
            scheduled_at: ScheduleAt::Time(
                ctx.timestamp + std::time::Duration::from_secs(sp.respawn_delay_s as u64)
            ),
            spawn_point_id: sp_id,
        });
        log::info!(
            "despawn_dead_enemy: enemy={} despawned, respawn in {}s at spawn_point={}",
            timer.enemy_id, sp.respawn_delay_s, sp_id
        );
    }
}
```

- [ ] **Add `respawn_at_spawn_point` reducer:**

```rust
#[reducer]
pub fn respawn_at_spawn_point(ctx: &ReducerContext, timer: RespawnTimer) {
    // SpawnPoint may have been deleted since the timer was scheduled — self-cancel silently
    let Some(sp) = ctx.db.spawn_point().id().find(&timer.spawn_point_id) else { return; };
    let Some(def) = ctx.db.enemy_def().id().find(&sp.enemy_def_id) else { return; };

    // Count living enemies currently at this spawn point
    let living_count = ctx.db.enemy()
        .zone_id()
        .filter(&sp.zone_id)
        .filter(|e| e.spawn_point_id == Some(sp.id) && !e.is_dead)
        .count() as u32;

    if living_count < sp.max_count {
        spawn_enemy_at(ctx, &sp, &def);
        log::info!("respawn_at_spawn_point: spawned enemy at spawn_point={}", sp.id);
    }
}
```

- [ ] **Build:** `cd server && spacetime build`

- [ ] **Commit:**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add despawn_dead_enemy and respawn_at_spawn_point reducers"
```

---

## Task 11: Bootstrap AiTick in init + client_connected

**Files:**
- Modify: `server/spacetimedb/src/lib.rs` — `init` and `client_connected` reducers

- [ ] **In `init`**, add AiTick bootstrap after the mana regen block:

```rust
// Start the AI tick if not already scheduled
if ctx.db.ai_tick().iter().next().is_none() {
    ctx.db.ai_tick().insert(AiTick {
        scheduled_id: 0,
        scheduled_at: ScheduleAt::Time(
            ctx.timestamp + std::time::Duration::from_millis(AI_TICK_INTERVAL_MS)
        ),
    });
    log::info!("init: scheduled AI tick");
}
```

- [ ] **In `client_connected`**, add AiTick self-heal bootstrap after the mana regen check:

```rust
if ctx.db.ai_tick().iter().next().is_none() {
    ctx.db.ai_tick().insert(AiTick {
        scheduled_id: 0,
        scheduled_at: ScheduleAt::Time(
            ctx.timestamp + std::time::Duration::from_millis(AI_TICK_INTERVAL_MS)
        ),
    });
    log::info!("client_connected: bootstrapped AI tick");
}
```

- [ ] **Build:** `cd server && spacetime build`

- [ ] **Commit:**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): bootstrap AiTick in init and client_connected"
```

---

## Task 12: Publish + integration test

Full deploy with `--delete-data` (required — CombatLog schema changed).

> ⚠️ This wipes all existing data. After publishing, zone id=1 must be recreated (see step below) before players can connect.

**Run from the `server/` directory unless stated otherwise.**

- [ ] **Ensure SpacetimeDB is running:** `spacetime start`

- [ ] **Build and publish:**

```bash
cd server
spacetime build
spacetime publish --server local zoneforge-server --delete-data
```

Expected: `Published module to zoneforge-server` (no errors).

- [ ] **Verify all new tables exist:**

```bash
spacetime sql zoneforge-server "SELECT count(*) FROM admin"
spacetime sql zoneforge-server "SELECT count(*) FROM enemy_def"
spacetime sql zoneforge-server "SELECT count(*) FROM enemy"
spacetime sql zoneforge-server "SELECT count(*) FROM spawn_point"
spacetime sql zoneforge-server "SELECT count(*) FROM ai_tick"
```

Expected: each query returns a count row with no SQL errors. `ai_tick` should show 1 row (the running tick).

- [ ] **Check init log output:**

```bash
spacetime logs zoneforge-server
```

Expected lines (order may vary):
```
init: seeded 0 admin identity(ies)
init: seeded 3 abilities
init: scheduled status effect tick
init: scheduled mana regen tick
init: scheduled AI tick
```

- [ ] **Recreate zone** — open the editor project in Unity Play mode and create a zone via the Zone Manager UI. Alternatively: `spacetime call zoneforge-server create_zone '"Forest"' 64 64 0.5` (type manually; do not paste — see gotcha in project memory about curly-quote encoding).

- [ ] **Verify player can connect** — open the client project in Unity Play mode. Confirm the player spawns and can move. Check Unity console for errors.

- [ ] **Final commit:**

```bash
git add -A
git commit -m "chore: Group 9 server schema deployed — enemy AI tables and reducers live"
```

---

**Server plan complete. Proceed to `2026-03-22-enemy-ai-client.md` for the client implementation.**
