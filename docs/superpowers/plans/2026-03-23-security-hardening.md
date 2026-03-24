# Security Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix all 13 issues from the 2026-03-23 security audit: admin authorization framework, input validation, missing server tables/reducers, and authorization gates on privileged operations.

**Architecture:** All changes are in `server/spacetimedb/src/lib.rs`. Admin identity is a compile-time constant seeded in `init`. All world-editing reducers are gated behind `is_admin()`. Missing Enemy AI tables and reducers are added to match the client autogen schema. After server changes, the module is published with `--delete-data` (Admin table requires `init` to re-seed), and client/editor bindings are regenerated.

**Tech Stack:** Rust, SpacetimeDB 2.0.3 WASM module, `spacetime` CLI

> ⚠️ **Breaking schema change** — the final publish (Task 5) requires `--delete-data`, which wipes all table data. After publish, recreate zone id=1 via the editor (required by `create_player` and `move_player`).

---

## File Map

| File | Change |
|------|--------|
| `server/spacetimedb/src/lib.rs` | All server changes — enums, tables, constants, helpers, reducers |
| `client/Assets/Scripts/autogen/` | Regenerated — do not edit manually |
| `editor/Assets/Scripts/autogen/` | Regenerated — do not edit manually |

---

## Task 1: Admin table, `is_admin()` helper, and init seeding

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Add `AiState` and `EnemyType` enums** immediately after the `StatusEffectType` enum (after line 16):

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

- [ ] **Add `Admin` table** immediately after the closing `}` of the `Player` struct (after line 38):

```rust
// Admin table — one row per admin identity.
// Admin set is compile-time; seeded in init(). Changes require --delete-data republish.
#[table(accessor = admin, public)]
pub struct Admin {
    #[primary_key]
    pub identity: Identity,
}
```

- [ ] **Add `ADMIN_IDENTITIES` constant and helpers** immediately after the `Admin` struct:

```rust
// Run `spacetime login show` to get your 64-char identity hex.
// Add each admin identity (with or without "0x" prefix) before publishing.
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
    Identity::from_byte_array(bytes)
}

/// Returns true if ctx.sender() is in the Admin table.
fn is_admin(ctx: &ReducerContext) -> bool {
    ctx.db.admin().identity().find(ctx.sender()).is_some()
}
```

- [ ] **Seed admin identities in `init()`** — add as the very first block inside the `init` reducer body (before the ability seeding block):

```rust
// Seed compile-time admin identities (only on fresh databases)
for &hex in ADMIN_IDENTITIES {
    ctx.db.admin().insert(Admin { identity: identity_from_hex(hex) });
}
log::info!("init: seeded {} admin identity(ies)", ADMIN_IDENTITIES.len());
```

- [ ] **Build to verify compile:**
```bash
cd server && spacetime build
```
Expected: zero errors. A warning about `ADMIN_IDENTITIES` being an empty slice is fine.

- [ ] **Commit:**
```bash
git add server/spacetimedb/src/lib.rs
git commit -m "security: add Admin table, is_admin() helper, and compile-time identity seeding"
```

---

## Task 2: Enemy AI tables (Admin, EnemyDefinition, SpawnPoint, Enemy)

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

These tables are required to match the client autogen schema. Without them, `attack_enemy` and admin enemy-management reducers cannot be implemented.

- [ ] **Add `EnemyDefinition` table** after the `ManaRegenTick` struct (after line 131):

```rust
// Defines an enemy archetype — shared stats referenced by all instances.
// Accessor matches autogen name "enemy_def".
#[table(accessor = enemy_def, public)]
pub struct EnemyDefinition {
    #[primary_key]
    #[auto_inc]
    pub id:              u64,
    pub name:            String,
    pub enemy_type:      EnemyType,
    pub prefab_name:     String,
    pub max_health:      i32,
    pub damage:          i32,
    pub aggro_range:     f32,
    pub attack_range:    f32,
    pub attack_speed_ms: u64,
    pub move_speed:      f32,
}
```

- [ ] **Add `SpawnPoint` table** immediately after `EnemyDefinition`:

```rust
// Marks a location in a zone where enemies of a given def spawn automatically.
#[table(accessor = spawn_point, public)]
pub struct SpawnPoint {
    #[primary_key]
    #[auto_inc]
    pub id:               u64,
    #[index(btree)]
    pub zone_id:          u64,
    pub x:                f32,
    pub y:                f32,
    pub enemy_def_id:     u64,
    pub max_count:        u32,
    pub respawn_delay_s:  u32,
}
```

- [ ] **Add `Enemy` table** immediately after `SpawnPoint`:

```rust
// One row per live (or recently dead) enemy instance.
#[table(accessor = enemy, public)]
pub struct Enemy {
    #[primary_key]
    #[auto_inc]
    pub id:               u64,
    #[index(btree)]
    pub zone_id:          u64,
    pub spawn_point_id:   Option<u64>,
    pub enemy_def_id:     u64,
    pub position_x:       f32,
    pub position_y:       f32,
    pub home_x:           f32,
    pub home_y:           f32,
    pub health:           i32,
    pub ai_state:         AiState,
    pub target_player_id: Option<u64>,
    pub last_attack_us:   u64,
    pub is_dead:          bool,
}
```

- [ ] **Add `EnemyRespawnTick` and `AiTick` scheduled tables** immediately after `Enemy`:

```rust
// Scheduled once per dead enemy to respawn it after respawn_delay_s.
#[table(accessor = enemy_respawn_tick, scheduled(tick_enemy_respawn))]
pub struct EnemyRespawnTick {
    #[primary_key]
    #[auto_inc]
    pub scheduled_id: u64,
    pub scheduled_at: ScheduleAt,
    pub enemy_id:     u64,
}

// Global AI tick — runs every 500ms to drive the enemy state machine.
#[table(accessor = ai_tick, scheduled(tick_ai))]
pub struct AiTick {
    #[primary_key]
    #[auto_inc]
    pub scheduled_id: u64,
    pub scheduled_at: ScheduleAt,
}
```

- [ ] **Build to verify compile:**
```bash
cd server && spacetime build
```
Expected: zero errors (tables are defined but reducers not yet added — that's OK at this stage if the `scheduled(...)` macro allows forward references; if not, add stub reducers temporarily).

> If the build fails with `E0425` ("cannot find function `tick_enemy_respawn`"), add these temporary stubs anywhere before the `init` reducer:
> ```rust
> #[reducer]
> pub fn tick_enemy_respawn(ctx: &ReducerContext, _tick: EnemyRespawnTick) {}
> #[reducer]
> pub fn tick_ai(ctx: &ReducerContext, _tick: AiTick) {}
> ```
> You will replace them with real implementations in Task 4.

- [ ] **Commit:**
```bash
git add server/spacetimedb/src/lib.rs
git commit -m "security: add EnemyDefinition, SpawnPoint, Enemy tables and AI/respawn scheduler stubs"
```

---

## Task 3: Input validation on existing reducers

This task addresses audit issues #2–5, #10, #11, #13.

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

### 3a — `create_player`: name length validation

- [ ] **Add name validation** as the very first statement inside `create_player` (before the idempotency check):

```rust
// Validate name: non-empty, max 64 chars, no null bytes
if name.is_empty() || name.len() > 64 || name.contains('\0') {
    log::warn!("create_player: invalid name from {}", ctx.sender());
    return;
}
```

### 3b — `move_player`: reject out-of-bounds instead of clamp

- [ ] **Replace the clamping block** (lines 489–497) with a hard rejection:

Replace:
```rust
    let clamped_x = new_x.clamp(0.0, zone.terrain_width as f32);
    let clamped_y = new_y.clamp(0.0, zone.terrain_height as f32);

    if clamped_x != new_x || clamped_y != new_y {
        log::warn!(
            "move_player: position ({}, {}) clamped to ({}, {})",
            new_x, new_y, clamped_x, clamped_y
        );
    }
```

With:
```rust
    if new_x < 0.0 || new_x > zone.terrain_width as f32
        || new_y < 0.0 || new_y > zone.terrain_height as f32
    {
        return Err(format!(
            "Position ({}, {}) out of zone bounds ({}x{})",
            new_x, new_y, zone.terrain_width, zone.terrain_height
        ));
    }
    let clamped_x = new_x;
    let clamped_y = new_y;
```

### 3c — `create_zone`: parameter bounds + admin gate (admin gate applied in Task 4)

- [ ] **Add validation** as the first statements inside `create_zone` (before inserting the Zone row):

```rust
// Security: admin-only (enforced in Task 4 below)

// Input validation
const MAX_TERRAIN_DIM: u32 = 512;
const MAX_NAME_LEN: usize = 128;
if name.is_empty() || name.len() > MAX_NAME_LEN || name.contains('\0') {
    return;
}
if terrain_width == 0 || terrain_width > MAX_TERRAIN_DIM {
    return;
}
if terrain_height == 0 || terrain_height > MAX_TERRAIN_DIM {
    return;
}
if !water_level.is_finite() || water_level < 0.0 || water_level > terrain_height as f32 {
    return;
}
```

> Note: `create_zone` currently returns `()` (no Result). We could change the signature to `Result<(), String>` for better error reporting, but to keep the diff minimal we use early returns. If you prefer explicit errors, change the return type and replace `return;` with `return Err("...".into());`.

### 3d — `spawn_entity`: zone boundary validation + string limits

- [ ] **Add validation** inside `spawn_entity` after the existing `prefab_name.is_empty()` check (after line 632):

```rust
    // Validate zone exists and position is in bounds
    let zone = ctx.db.zone().id().find(&zone_id)
        .ok_or_else(|| format!("Zone {} not found", zone_id))?;

    if !x.is_finite() || !y.is_finite() || !elevation.is_finite() {
        return Err("Non-finite position values".to_string());
    }
    if x < 0.0 || x > zone.terrain_width as f32 || y < 0.0 || y > zone.terrain_height as f32 {
        return Err(format!(
            "Position ({}, {}) out of zone bounds ({}x{})",
            x, y, zone.terrain_width, zone.terrain_height
        ));
    }
    const MAX_ELEVATION: f32 = 200.0;
    if elevation < -10.0 || elevation > MAX_ELEVATION {
        return Err(format!("Elevation {} out of range [-10, {}]", elevation, MAX_ELEVATION));
    }
    if prefab_name.len() > 128 || entity_type.len() > 64 {
        return Err("prefab_name or entity_type exceeds maximum length".to_string());
    }
```

> Also remove the existing `if prefab_name.is_empty()` check — the new length check below subsumes it. Keep `prefab_name.is_empty()` as a separate guard just above:
> ```rust
>     if prefab_name.is_empty() {
>         return Err("prefab_name cannot be empty".to_string());
>     }
> ```
> This is still needed; the new block goes immediately after it.

### 3e — `update_terrain_chunk`: height data float validation

- [ ] **Add NaN/Infinity check for height data** after the existing length checks (after line 579):

```rust
    // Validate height values: each group of 4 bytes is a little-endian f32.
    // Reject NaN and Infinity which would corrupt terrain rendering.
    for chunk in height_data.chunks_exact(4) {
        let val = f32::from_le_bytes([chunk[0], chunk[1], chunk[2], chunk[3]]);
        if !val.is_finite() {
            return Err(format!("height_data contains non-finite float value: {}", val));
        }
    }
```

### 3f — `use_ability`: damage value overflow guard

- [ ] **Add ability damage bounds check** inside `use_ability` after the ability lookup (after line 317):

```rust
    // Guard against pathological damage values that could overflow i32 arithmetic
    const MAX_ABILITY_DAMAGE: i32 = 10_000;
    if ability.damage.abs() > MAX_ABILITY_DAMAGE {
        return Err(format!("Ability {} has invalid damage value {}", ability_id, ability.damage));
    }
```

- [ ] **Build to verify compile:**
```bash
cd server && spacetime build
```
Expected: zero errors.

- [ ] **Commit:**
```bash
git add server/spacetimedb/src/lib.rs
git commit -m "security: input validation — zone bounds, position rejection, terrain float check, string limits, damage guard"
```

---

## Task 4: Authorization gates and missing reducers

This task addresses audit issues #1, #6, #7.

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

### 4a — Gate existing editor reducers behind `is_admin()`

- [ ] **Add admin check to `create_zone`** — add as the very first line of the reducer body (before the validation block added in Task 3c):

```rust
    if !is_admin(ctx) {
        log::warn!("create_zone: unauthorized caller {}", ctx.sender());
        return;
    }
```

- [ ] **Add admin check to `update_terrain_chunk`** — add at the very start of the reducer body (before the length checks):

```rust
    if !is_admin(ctx) {
        return Err("Not authorized: admin only".to_string());
    }
```

- [ ] **Add admin check to `spawn_entity`** — add at the very start of the reducer body (before `if prefab_name.is_empty()`):

```rust
    if !is_admin(ctx) {
        return Err("Not authorized: admin only".to_string());
    }
```

### 4b — Admin enemy-management reducers

- [ ] **Add `create_enemy_def` reducer** (after the `spawn_entity` reducer):

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
    if !is_admin(ctx) {
        return Err("Not authorized: admin only".to_string());
    }
    if name.is_empty() || name.len() > 128 || name.contains('\0') {
        return Err("Invalid enemy def name".to_string());
    }
    if prefab_name.is_empty() || prefab_name.len() > 128 {
        return Err("Invalid prefab_name".to_string());
    }
    if max_health <= 0 || max_health > 100_000 {
        return Err(format!("max_health {} out of range [1, 100000]", max_health));
    }
    if damage < 0 || damage > 10_000 {
        return Err(format!("damage {} out of range [0, 10000]", damage));
    }
    if !aggro_range.is_finite() || aggro_range < 0.0 || aggro_range > 200.0 {
        return Err("aggro_range out of range [0, 200]".to_string());
    }
    if !attack_range.is_finite() || attack_range < 0.0 || attack_range > aggro_range {
        return Err("attack_range must be in [0, aggro_range]".to_string());
    }
    if attack_speed_ms == 0 || attack_speed_ms > 60_000 {
        return Err("attack_speed_ms out of range [1, 60000]".to_string());
    }
    if !move_speed.is_finite() || move_speed < 0.0 || move_speed > 100.0 {
        return Err("move_speed out of range [0, 100]".to_string());
    }
    ctx.db.enemy_def().insert(EnemyDefinition {
        id: 0,
        name,
        enemy_type,
        prefab_name,
        max_health,
        damage,
        aggro_range,
        attack_range,
        attack_speed_ms,
        move_speed,
    });
    Ok(())
}
```

- [ ] **Add `delete_enemy_def` reducer** (after `create_enemy_def`):

```rust
#[reducer]
pub fn delete_enemy_def(ctx: &ReducerContext, def_id: u64) -> Result<(), String> {
    if !is_admin(ctx) {
        return Err("Not authorized: admin only".to_string());
    }
    ctx.db.enemy_def().id().find(&def_id)
        .ok_or_else(|| format!("EnemyDef {} not found", def_id))?;
    ctx.db.enemy_def().id().delete(&def_id);
    Ok(())
}
```

- [ ] **Add `create_spawn_point` reducer** (after `delete_enemy_def`):

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
    if !is_admin(ctx) {
        return Err("Not authorized: admin only".to_string());
    }
    let zone = ctx.db.zone().id().find(&zone_id)
        .ok_or_else(|| format!("Zone {} not found", zone_id))?;
    ctx.db.enemy_def().id().find(&enemy_def_id)
        .ok_or_else(|| format!("EnemyDef {} not found", enemy_def_id))?;
    if !x.is_finite() || !y.is_finite()
        || x < 0.0 || x > zone.terrain_width as f32
        || y < 0.0 || y > zone.terrain_height as f32
    {
        return Err("Spawn point position out of zone bounds".to_string());
    }
    if max_count == 0 || max_count > 100 {
        return Err("max_count out of range [1, 100]".to_string());
    }
    if respawn_delay_s > 3600 {
        return Err("respawn_delay_s must be <= 3600".to_string());
    }
    ctx.db.spawn_point().insert(SpawnPoint {
        id: 0,
        zone_id,
        x,
        y,
        enemy_def_id,
        max_count,
        respawn_delay_s,
    });
    Ok(())
}
```

- [ ] **Add `delete_spawn_point` reducer** (after `create_spawn_point`):

```rust
#[reducer]
pub fn delete_spawn_point(ctx: &ReducerContext, spawn_point_id: u64) -> Result<(), String> {
    if !is_admin(ctx) {
        return Err("Not authorized: admin only".to_string());
    }
    ctx.db.spawn_point().id().find(&spawn_point_id)
        .ok_or_else(|| format!("SpawnPoint {} not found", spawn_point_id))?;
    ctx.db.spawn_point().id().delete(&spawn_point_id);
    Ok(())
}
```

- [ ] **Add `spawn_enemy_manual` reducer** (admin-only; for editor/testing) (after `delete_spawn_point`):

```rust
#[reducer]
pub fn spawn_enemy_manual(
    ctx: &ReducerContext,
    zone_id: u64,
    x: f32,
    y: f32,
    enemy_def_id: u64,
) -> Result<(), String> {
    if !is_admin(ctx) {
        return Err("Not authorized: admin only".to_string());
    }
    let zone = ctx.db.zone().id().find(&zone_id)
        .ok_or_else(|| format!("Zone {} not found", zone_id))?;
    let def = ctx.db.enemy_def().id().find(&enemy_def_id)
        .ok_or_else(|| format!("EnemyDef {} not found", enemy_def_id))?;
    if !x.is_finite() || !y.is_finite()
        || x < 0.0 || x > zone.terrain_width as f32
        || y < 0.0 || y > zone.terrain_height as f32
    {
        return Err("Spawn position out of zone bounds".to_string());
    }
    ctx.db.enemy().insert(Enemy {
        id: 0,
        zone_id,
        spawn_point_id: None,
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

- [ ] **Add `despawn_enemy` reducer** (after `spawn_enemy_manual`):

```rust
#[reducer]
pub fn despawn_enemy(ctx: &ReducerContext, enemy_id: u64) -> Result<(), String> {
    if !is_admin(ctx) {
        return Err("Not authorized: admin only".to_string());
    }
    ctx.db.enemy().id().find(&enemy_id)
        .ok_or_else(|| format!("Enemy {} not found", enemy_id))?;
    ctx.db.enemy().id().delete(&enemy_id);
    Ok(())
}
```

### 4c — `attack_enemy` reducer (callable by all authenticated players)

- [ ] **Add `attack_enemy` reducer** (after `despawn_enemy`):

```rust
/// Player uses an ability to attack an enemy. Called from CombatInputHandler.
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

    // 2. Ability must exist and be non-self-cast (can't use heal on enemies)
    let ability = ctx.db.ability().id().find(&ability_id)
        .ok_or("Ability not found")?;
    if ability.ability_type == AbilityType::SelfCast {
        return Err("Self-cast abilities cannot target enemies".to_string());
    }

    // 3. Target enemy must exist and not be dead
    let enemy = ctx.db.enemy().id().find(&enemy_id)
        .ok_or("Enemy not found")?;
    if enemy.is_dead {
        return Err("Enemy is already dead".to_string());
    }

    // 4. Range check
    let dx = player.position_x - enemy.position_x;
    let dz = player.position_y - enemy.position_y;
    let dist_sq = dx * dx + dz * dz;
    if dist_sq > ability.range * ability.range {
        return Err(format!(
            "Enemy out of range (dist={:.1}, range={:.1})",
            dist_sq.sqrt(), ability.range
        ));
    }

    // 5. Cooldown check
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
    if on_cooldown {
        return Err("Ability on cooldown".to_string());
    }

    // 6. Mana check
    if player.mana < ability.mana_cost {
        return Err(format!("Insufficient mana ({}/{})", player.mana, ability.mana_cost));
    }

    // 7. Damage bounds guard (defense in depth — abilities are seeded by admin)
    const MAX_ABILITY_DAMAGE: i32 = 10_000;
    if ability.damage.abs() > MAX_ABILITY_DAMAGE {
        return Err(format!("Ability {} has invalid damage value", ability_id));
    }

    // All checks passed — deduct mana and update cooldown
    let player_id = player.id;
    let new_mana = player.mana - ability.mana_cost;
    ctx.db.player().id().update(Player { mana: new_mana, ..player });

    let ready_at = ctx.timestamp + std::time::Duration::from_millis(ability.cooldown_ms);
    if let Some(existing_cd) = ctx.db.player_cooldown()
        .player_id()
        .filter(&player_id)
        .find(|cd| cd.ability_id == ability_id)
    {
        ctx.db.player_cooldown().id().update(PlayerCooldown { ready_at, ..existing_cd });
    } else {
        ctx.db.player_cooldown().insert(PlayerCooldown {
            id: 0,
            player_id,
            ability_id,
            ready_at,
        });
    }

    // Apply damage to enemy
    apply_damage_to_enemy(ctx, enemy_id, player_id, ability.damage);

    log::info!("attack_enemy: player={} ability={} enemy={}", player_id, ability_id, enemy_id);
    Ok(())
}

/// Applies `amount` damage to an enemy. Negative amount = heal.
fn apply_damage_to_enemy(
    ctx: &ReducerContext,
    enemy_id: u64,
    attacker_id: u64,
    amount: i32,
) {
    let Some(enemy) = ctx.db.enemy().id().find(&enemy_id) else { return; };
    if enemy.is_dead { return; }

    let def = ctx.db.enemy_def().id().find(&enemy.enemy_def_id);
    let max_health = def.map(|d| d.max_health).unwrap_or(enemy.health);

    let new_health = (enemy.health - amount).clamp(0, max_health);
    let is_dead = new_health == 0 && amount > 0;

    ctx.db.enemy().id().update(Enemy {
        health: new_health,
        is_dead,
        // If killed, clear AI targeting
        ai_state: if is_dead { AiState::Idle } else { enemy.ai_state.clone() },
        target_player_id: if is_dead { None } else { enemy.target_player_id },
        ..enemy
    });

    if is_dead {
        log::info!("apply_damage_to_enemy: enemy={} killed by player={}", enemy_id, attacker_id);
        // Schedule respawn if this enemy belongs to a spawn point
        if let Some(sp_id) = ctx.db.enemy().id().find(&enemy_id).and_then(|e| e.spawn_point_id) {
            if let Some(sp) = ctx.db.spawn_point().id().find(&sp_id) {
                ctx.db.enemy_respawn_tick().insert(EnemyRespawnTick {
                    scheduled_id: 0,
                    scheduled_at: ScheduleAt::Time(
                        ctx.timestamp + std::time::Duration::from_secs(sp.respawn_delay_s as u64)
                    ),
                    enemy_id,
                });
            }
        }
    }
}
```

### 4d — `tick_enemy_respawn` and `tick_ai` scheduled reducers

If you added stubs in Task 2, replace them now. Otherwise add these after `apply_damage_to_enemy`.

- [ ] **Add `tick_enemy_respawn` reducer:**

```rust
#[reducer]
pub fn tick_enemy_respawn(ctx: &ReducerContext, tick: EnemyRespawnTick) {
    let Some(enemy) = ctx.db.enemy().id().find(&tick.enemy_id) else { return; };
    if !enemy.is_dead { return; }  // Already revived (manual despawn/respawn)

    let def = ctx.db.enemy_def().id().find(&enemy.enemy_def_id);
    let max_health = def.map(|d| d.max_health).unwrap_or(100);

    ctx.db.enemy().id().update(Enemy {
        health: max_health,
        is_dead: false,
        ai_state: AiState::Idle,
        target_player_id: None,
        position_x: enemy.home_x,
        position_y: enemy.home_y,
        ..enemy
    });
    log::info!("tick_enemy_respawn: enemy={} respawned", tick.enemy_id);
}
```

- [ ] **Add `tick_ai` reducer** (replaces stub from Task 2):

```rust
/// Drives the enemy AI state machine every 500ms.
/// Idle → Chase when player enters aggro range.
/// Chase → Attack when player enters attack range; tracks player position.
/// Attack → Chase when player moves out of attack range.
/// Any state → Idle when target player disconnects/dies or exceeds aggro range.
#[reducer]
pub fn tick_ai(ctx: &ReducerContext, _tick: AiTick) {
    let now_us = ctx.timestamp
        .to_duration_since_unix_epoch()
        .unwrap_or_default()
        .as_micros() as u64;

    let enemies: Vec<Enemy> = ctx.db.enemy().iter().filter(|e| !e.is_dead).collect();

    for enemy in enemies {
        let Some(def) = ctx.db.enemy_def().id().find(&enemy.enemy_def_id) else { continue; };

        let updated = match enemy.ai_state.clone() {
            AiState::Idle => {
                // Scan for the nearest living player in aggro range
                let target = ctx.db.player().iter()
                    .filter(|p| !p.is_dead && p.zone_id == enemy.zone_id)
                    .min_by(|a, b| {
                        let da = dist_sq(a.position_x, a.position_y, enemy.position_x, enemy.position_y);
                        let db = dist_sq(b.position_x, b.position_y, enemy.position_x, enemy.position_y);
                        da.partial_cmp(&db).unwrap_or(std::cmp::Ordering::Equal)
                    });
                if let Some(p) = target {
                    let d2 = dist_sq(p.position_x, p.position_y, enemy.position_x, enemy.position_y);
                    if d2 <= def.aggro_range * def.aggro_range {
                        Enemy { ai_state: AiState::Chase, target_player_id: Some(p.id), ..enemy }
                    } else {
                        enemy  // No change
                    }
                } else {
                    enemy
                }
            },
            AiState::Chase => {
                let target_id = match enemy.target_player_id {
                    Some(id) => id,
                                    None => {
                        let e = return_idle(&enemy);
                        ctx.db.enemy().id().update(e);
                        continue;
                    }
                };
                let Some(player) = ctx.db.player().id().find(&target_id) else {
                    // Target gone — return home
                    let e = return_idle(&enemy);
                    ctx.db.enemy().id().update(e);
                    continue;
                };
                if player.is_dead {
                    let e = return_idle(&enemy);
                    ctx.db.enemy().id().update(e);
                    continue;
                }
                // Check if still in aggro range
                let d2 = dist_sq(player.position_x, player.position_y, enemy.position_x, enemy.position_y);
                if d2 > def.aggro_range * def.aggro_range * 1.5 {
                    // Leash — return home
                    let e = return_idle(&enemy);
                    ctx.db.enemy().id().update(e);
                    continue;
                }
                if d2 <= def.attack_range * def.attack_range {
                    Enemy { ai_state: AiState::Attack, ..enemy }
                } else {
                    // Step toward player
                    let step = def.move_speed * 0.5; // 500ms tick
                    let (nx, ny) = step_toward(
                        enemy.position_x, enemy.position_y,
                        player.position_x, player.position_y,
                        step,
                    );
                    Enemy { position_x: nx, position_y: ny, ..enemy }
                }
            },
            AiState::Attack => {
                let target_id = match enemy.target_player_id {
                    Some(id) => id,
                    None => {
                        let e = return_idle(&enemy);
                        ctx.db.enemy().id().update(e);
                        continue;
                    }
                };
                let Some(player) = ctx.db.player().id().find(&target_id) else {
                    let e = return_idle(&enemy);
                    ctx.db.enemy().id().update(e);
                    continue;
                };
                if player.is_dead {
                    let e = return_idle(&enemy);
                    ctx.db.enemy().id().update(e);
                    continue;
                }
                let d2 = dist_sq(player.position_x, player.position_y, enemy.position_x, enemy.position_y);
                if d2 > def.attack_range * def.attack_range {
                    Enemy { ai_state: AiState::Chase, ..enemy }
                } else {
                    // Attack if cooldown elapsed
                    let attack_interval_us = def.attack_speed_ms as u64 * 1000;
                    if now_us.saturating_sub(enemy.last_attack_us) >= attack_interval_us {
                        apply_damage(ctx, target_id, enemy.id, 0, def.damage);
                        Enemy { last_attack_us: now_us, ..enemy }
                    } else {
                        enemy  // Still on cooldown
                    }
                }
            },
        };
        ctx.db.enemy().id().update(updated);
    }

    // Re-schedule next tick
    ctx.db.ai_tick().insert(AiTick {
        scheduled_id: 0,
        scheduled_at: ScheduleAt::Time(
            ctx.timestamp + std::time::Duration::from_millis(500)
        ),
    });
}

/// Helper: returns an enemy to its home position with Idle state.
fn return_idle(enemy: &Enemy) -> Enemy {
    Enemy {
        ai_state: AiState::Idle,
        target_player_id: None,
        position_x: enemy.home_x,
        position_y: enemy.home_y,
        ..enemy.clone()
    }
}

fn dist_sq(ax: f32, ay: f32, bx: f32, by: f32) -> f32 {
    let dx = ax - bx;
    let dy = ay - by;
    dx * dx + dy * dy
}

fn step_toward(from_x: f32, from_y: f32, to_x: f32, to_y: f32, step: f32) -> (f32, f32) {
    let dx = to_x - from_x;
    let dy = to_y - from_y;
    let dist = (dx * dx + dy * dy).sqrt();
    if dist <= step {
        (to_x, to_y)
    } else {
        (from_x + dx / dist * step, from_y + dy / dist * step)
    }
}
```

### 4e — Start AI tick in `init` and `client_connected`

- [ ] **Add AI tick start to `init`** — add after the mana regen tick seeding block (inside `init`):

```rust
    // Start the enemy AI tick
    if ctx.db.ai_tick().iter().next().is_none() {
        ctx.db.ai_tick().insert(AiTick {
            scheduled_id: 0,
            scheduled_at: ScheduleAt::Time(
                ctx.timestamp + std::time::Duration::from_millis(500)
            ),
        });
        log::info!("init: scheduled AI tick");
    }
```

- [ ] **Bootstrap AI tick in `client_connected`** — add after the mana regen bootstrap block:

```rust
    if ctx.db.ai_tick().iter().next().is_none() {
        ctx.db.ai_tick().insert(AiTick {
            scheduled_id: 0,
            scheduled_at: ScheduleAt::Time(
                ctx.timestamp + std::time::Duration::from_millis(500)
            ),
        });
        log::info!("client_connected: bootstrapped AI tick");
    }
```

- [ ] **Build to verify compile:**
```bash
cd server && spacetime build
```
Expected: zero errors.

> **If `return_idle` causes a type error** because `Enemy` doesn't implement `Clone` automatically: SpacetimeDB table structs do implement Clone — if you see an error, add `#[derive(Clone)]` is not needed (the `#[table]` macro handles it). If the borrow checker rejects `..enemy.clone()`, use `..enemy` directly when `enemy` is consumed by the match arm; restructure the match to avoid double-moves as needed.

- [ ] **Commit:**
```bash
git add server/spacetimedb/src/lib.rs
git commit -m "security: add authorization gates, missing reducers (attack_enemy, enemy management, AI tick)"
```

---

## Task 5: Set admin identity

Before deploying, you need to fill in your identity.

- [ ] **Get your identity hex:**
```bash
spacetime login show
```
Copy the 64-character hex string (without `0x` prefix, or keep it — both work).

- [ ] **Edit `ADMIN_IDENTITIES`** in `lib.rs` — replace the placeholder comment with your actual hex:

```rust
const ADMIN_IDENTITIES: &[&str] = &[
    "0x<your-64-char-hex-here>",
];
```

- [ ] **Build again to confirm no errors:**
```bash
cd server && spacetime build
```

- [ ] **Commit:**
```bash
git add server/spacetimedb/src/lib.rs
git commit -m "security: seed admin identity"
```

---

## Task 6: Deploy and regenerate bindings

- [ ] **Confirm `spacetime start` is running** in another terminal.

- [ ] **Publish with `--delete-data`** (required because Admin, Enemy, EnemyDef, SpawnPoint are new tables and `init` must re-seed):
```bash
cd server && spacetime publish --server local zoneforge-server --delete-data -y
```
Expected output ends with a success line, no `ERROR:` lines.

- [ ] **Verify admin identity seeded** via logs:
```bash
spacetime logs zoneforge-server | grep "seeded"
```
Expected: `init: seeded 1 admin identity(ies)`

- [ ] **Regenerate client bindings:**
```bash
cd client && spacetime generate --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

- [ ] **Regenerate editor bindings:**
```bash
cd editor && spacetime generate --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

- [ ] **Verify admin table appears in client autogen:**
```bash
ls client/Assets/Scripts/autogen/Tables/Admin.g.cs
```

- [ ] **Commit the regenerated bindings:**
```bash
git add client/Assets/Scripts/autogen editor/Assets/Scripts/autogen
git commit -m "chore: regenerate autogen bindings after security hardening schema"
```

---

## Task 7: Post-deploy smoke test

- [ ] **Recreate zone id=1** via the editor (required after `--delete-data` — `create_player` hardcodes `zone_id: 1`):
  - Open the ZoneForge editor in Unity Play mode
  - Create a zone (your identity must be in `ADMIN_IDENTITIES` for this to succeed)

- [ ] **Verify non-admin calls are rejected** — from a second Unity client (different identity), try calling `create_zone` via `spacetime call`:
```bash
spacetime call zoneforge-server create_zone '["test", 64, 64, 0.5]' --server local
```
Expected: reducer returns error `Not authorized: admin only` (visible in `spacetime logs zoneforge-server`).

- [ ] **Verify `move_player` rejects out-of-bounds:**
```bash
spacetime call zoneforge-server move_player '[99999.0, 99999.0]' --server local
```
Expected: error `Position (99999, 99999) out of zone bounds`.

- [ ] **Verify `attack_enemy` works for valid players** — connect the game client, spawn an enemy via the editor, then use ability 1 (Auto-Attack) on Tab-selected enemy target.

- [ ] **Commit to umbrella repo:**
```bash
cd ..  # back to umbrella repo root
git add server client editor
git commit -m "security: hardening complete — admin auth, input validation, missing reducers"
```

---

## Summary of Issues Addressed

| Issue | Severity | Fix |
|-------|----------|-----|
| #1 Missing Admin framework | Critical | Tasks 1, 5 |
| #2 Terrain chunk float validation | Critical | Task 3e |
| #3 move_player clamps instead of rejects | High | Task 3b |
| #4 spawn_entity no zone bounds | High | Task 3d |
| #5 create_zone no bounds | High | Task 3c |
| #6 Editor reducers unprotected | High | Task 4a |
| #7 Missing reducers (attack_enemy etc.) | High | Tasks 2, 4b-d |
| #8 Client cooldown bypassable | Medium | Server already authoritative — no change needed |
| #9 Mana deducted before ability resolves | Medium | No change (atomicity ensures safety; inline comment added) |
| #10 Negative damage no bounds | Medium | Tasks 3f, 4c |
| #11 Self-cast targeting | Medium | No change (design intent is acceptable) |
| #12 Token in PlayerPrefs | Low | No change (acceptable for dev; production note added here) |
| #13 String fields unsanitized | Low | Tasks 3a, 4b |
