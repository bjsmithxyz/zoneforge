# ZoneForge — SpacetimeDB Server Architecture

## Document History

| Version | Date | Summary |
|---------|------|---------|
| 1.3 | 2026-05-10 | Module split into files (`combat.rs`, `enemy.rs`, `portal.rs`, `inventory.rs`, `loot.rs`, `terrain.rs`, `atmosphere.rs`); added Admin/Portal/Enemy/Inventory/Loot/Atmosphere tables; added `delete_zone`, `attack_enemy`, `pickup_item`, `enter_zone`, `change_weather`, `set_zone_mood`, `prune_combat_log` etc.; `create_player` now spawns at zone-center, not hardcoded id=1 |
| 1.2 | 2026-03-24 | Added `ManaRegenTick`; added `tick_mana_regen`, `client_connected` reducers; renamed `paint_terrain` → `update_terrain_chunk`; corrected `spawn_entity` signature |
| 1.1 | 2026-03-07 | Added combat tables (Ability, PlayerCooldown, StatusEffect, CombatLog, StatusEffectTick) and reducers |
| 1.0 | 2026-02-01 | Initial document |

---

## Overview

The server is a single Rust WASM module published to SpacetimeDB. It contains all authoritative game logic: table definitions (schema) and reducers (mutations).

**Module name:** `zoneforge-server`
**Language:** Rust (compiled to `wasm32-unknown-unknown`)
**Entry point:** `server/spacetimedb/src/lib.rs` (split into per-subsystem files — see [Project Layout](#project-layout))

## Tables

### Core (lib.rs)

| Table | Key fields | Purpose |
|-------|-----------|---------|
| `Player` | `id` (PK, auto_inc), `identity` (unique), `zone_id` (btree) | Position, health/mana, `is_dead`, zone |
| `Zone` | `id` (PK, auto_inc) | `name`, `terrain_width/height`, `water_level`, `mood_preset_id` |
| `EntityInstance` | `id` (PK, auto_inc), `zone_id` | Placed entities: `prefab_name`, `position_x/y`, `elevation`, `entity_type` string |
| `Admin` | `identity` (PK) | Admin allow-list; seeded from compile-time `ADMIN_IDENTITIES` in `init` and `client_connected` |

### Terrain (terrain.rs)

| Table | Key fields | Purpose |
|-------|-----------|---------|
| `TerrainChunk` | `id` (PK), `zone_id` (btree), `chunk_x`, `chunk_z` | Per-chunk `height_data` (4096 B, 1024 × f32 LE) and `splat_data` (4096 B, 1024 × RGBA u8). `CHUNK_SIZE = 32`. |

### Combat (combat.rs)

| Table | Key fields | Purpose |
|-------|-----------|---------|
| `Ability` | `id` (PK) | `name`, `damage` (negative = heal), `cooldown_ms`, `mana_cost`, `range`, `ability_type` (`MeleeAttack`/`Projectile`/`SelfCast`) |
| `PlayerCooldown` | `id` (PK), `player_id` (btree) | Per-player per-ability `ready_at` timestamp |
| `StatusEffect` | `id` (PK), `target_id` (btree) | `effect_type` (`Burn`/`Freeze`/`Stun`/`Poison`), `expires_at`, `damage_per_tick` |
| `CombatLog` | `id` (PK) | Immutable: `timestamp`, `attacker_id`, `target_id`, `ability_id`, `damage_dealt`, `overkill` |
| `StatusEffectTick` | `scheduled_id` (PK) | Scheduler — drives `tick_status_effects` every 1 s |
| `ManaRegenTick` | `scheduled_id` (PK) | Scheduler — drives `tick_mana_regen` every 2 s |
| `CombatLogPruneTick` | `scheduled_id` (PK) | Scheduler — caps `CombatLog` to latest 1000 rows every 60 s |

### Enemies (enemy.rs)

| Table | Key fields | Purpose |
|-------|-----------|---------|
| `EnemyDefinition` | `id` (PK) | `name`, `enemy_type` (`Melee`/`Ranged`/`Caster`), `prefab_name`, `max_health`, `damage`, ranges, `attack_speed_ms`, `move_speed` |
| `SpawnPoint` | `id` (PK), `zone_id` (btree) | `(x, y)`, `enemy_def_id`, `max_count`, `respawn_delay_s` |
| `Enemy` | `id` (PK), `zone_id` (btree) | Live row: position, home position, health, `ai_state` (`Idle`/`Chase`/`Attack`), `target_player_id`, `last_attack_us`, `is_dead` |
| `EnemyRespawnTick` | `scheduled_id` (PK), `enemy_id` | Scheduler — respawns one enemy at its spawn point |
| `AiTick` | `scheduled_id` (PK) | Scheduler — drives `tick_ai` every 500 ms |

### Portals (portal.rs)

| Table | Key fields | Purpose |
|-------|-----------|---------|
| `Portal` | `id` (PK), `source_zone_id` (btree), `dest_zone_id` (btree) | Source coords, dest spawn coords, `bidirectional`, `label` |

### Inventory & loot (inventory.rs, loot.rs)

| Table | Key fields | Purpose |
|-------|-----------|---------|
| `ItemDefinition` | `id` (PK) | `name`, `description`, `item_type` (`Weapon`/`Armor`/`Accessory`/`Consumable`), `rarity`, `icon_name`, `damage_bonus`, `armor_bonus`, `healing`, `value` |
| `Inventory` | `id` (PK), `player_id` (btree) | `(player_id, item_def_id) → quantity` |
| `Equipment` | `player_id` (PK) | `weapon_id`, `armor_id`, `accessory_id` (each `Option<u64>`) |
| `LootTable` | `id` (PK), `enemy_def_id` (btree) | Drop entry: `item_def_id`, `drop_chance` (0–100), `min/max_quantity` |
| `ItemDrop` | `id` (PK), `zone_id` (btree) | Live ground drop: `item_def_id`, `quantity`, `pos_x/y` |

### Atmosphere (atmosphere.rs)

| Table | Key fields | Purpose |
|-------|-----------|---------|
| `WorldClock` | `id: u8` (PK, always 0) | Single-row world time: `minutes_of_day` (0–1439), `last_tick` |
| `WeatherState` | `zone_id` (PK) | Per-zone `WeatherKind` (`Clear`/`Rain`/`Storm`/`Fog`/`Snow`), `intensity` (0–1), `started_at` |
| `WorldClockTick` | `scheduled_id` (PK) | Scheduler — drives `tick_world_time` every 1 s (advances 1 in-game minute per real second) |

## Reducers

### Lifecycle (lib.rs)

| Reducer | Caller | Purpose |
|---------|--------|---------|
| `init` | SpacetimeDB on first publish | Seeds `Admin` from `ADMIN_IDENTITIES`, seeds 3 starter abilities (Auto-Attack/Fireball/Heal), bootstraps all scheduler ticks, inserts `WorldClock` |
| `client_connected` | SpacetimeDB lifecycle | Auto-promote admin if sender matches `ADMIN_IDENTITIES`; re-bootstrap any missing scheduler tick (handles hot-publish without `--delete-data`) |

### Player & zone (lib.rs)

| Reducer | Caller | Purpose |
|---------|--------|---------|
| `create_player(name)` | Client on connect | Register new player; spawns at the lowest-id zone's center (no longer hardcoded id=1); idempotent per identity |
| `move_player(new_x, new_y)` | Client on input | Server-validated movement; bounds-checked; returns `Result` |
| `create_zone(name, w, h, water_level)` | Editor | Validates dims (≤512) and inputs; creates zone + flat `TerrainChunk` grid (height = `water_level + 0.5`, full Grass splat); inserts default Clear `WeatherState` |
| `delete_zone(zone_id)` | Editor (admin) | Cascade-delete zone + terrain + entities + drops + enemies + spawn points + portals + weather; rejects if any player is in the zone |
| `spawn_entity(zone_id, prefab, x, y, elevation, entity_type)` | Editor | Place entity; `elevation` is world-space Y; `entity_type` is a free string. Bounds, finite, and length validated. |

### Terrain (terrain.rs)

| Reducer | Caller | Purpose |
|---------|--------|---------|
| `update_terrain_chunk(zone_id, cx, cz, height_data, splat_data)` | Editor on brush stroke | Update a chunk's heightmap / splatmap; validates 4096-byte arrays and finite floats |

### Combat (combat.rs)

| Reducer | Caller | Purpose |
|---------|--------|---------|
| `use_ability(ability_id, target_id)` | Client on key press | Server-authoritative: alive, ability exists, target valid, range, cooldown, mana. Calls `apply_damage`. |
| `attack_enemy(ability_id, enemy_id)` | Client on attack input | Same authority checks but against an `Enemy` row; awards loot on kill via `spawn_loot_drops` |
| `respawn()` | Client on R key | Resets dead player to zone centre with full HP/mana; clears their `StatusEffect` rows |
| `tick_status_effects(_)` | Scheduler (1 Hz) | Applies DoT ticks, removes expired effects, re-schedules self |
| `tick_mana_regen(_)` | Scheduler (2 Hz) | Restores +10 mana to all living players; re-schedules self |
| `prune_combat_log(_)` | Scheduler (60 s) | Caps `CombatLog` at the latest 1000 rows to bound client subscription bandwidth |

`apply_damage` is a plain Rust helper (not a reducer) called by `use_ability`, `attack_enemy`, and `tick_status_effects`.

### Enemies (enemy.rs)

| Reducer | Caller | Purpose |
|---------|--------|---------|
| `create_enemy_def(...)` | Editor (admin) | Define a new enemy archetype |
| `delete_enemy_def(def_id)` | Editor (admin) | Delete a definition (refuses if in use) |
| `create_spawn_point(zone_id, x, y, enemy_def_id, max_count, respawn_delay_s)` | Editor (admin) | Place a spawner |
| `delete_spawn_point(spawn_point_id)` | Editor (admin) | Remove a spawner + its live enemies |
| `spawn_enemy_manual(...)` | Editor (admin) | Manual one-shot spawn |
| `despawn_enemy(enemy_id)` | Editor (admin) | Remove a live enemy |
| `update_ai_state(enemy_id, ai_state)` | Editor (admin) | Manual AI override |
| `tick_ai(_)` | Scheduler (2 Hz) | Drives Idle/Chase/Attack state machine across all live enemies |
| `tick_enemy_respawn(tick)` | Scheduler (one-shot per kill) | Resurrects a killed enemy at its spawn point after the configured delay |

### Portals (portal.rs)

| Reducer | Caller | Purpose |
|---------|--------|---------|
| `create_portal(...)` | Editor (admin) | Author a portal between zones |
| `delete_portal(portal_id)` | Editor (admin) | Remove a portal |
| `enter_zone()` | Client on portal proximity | Atomically move the calling player to the nearest portal's destination zone + spawn point |

### Inventory (inventory.rs)

| Reducer | Caller | Purpose |
|---------|--------|---------|
| `create_item_def(...)` | Editor (admin) | Define an item |
| `delete_item_def(item_def_id)` | Editor (admin) | Delete an item definition |
| `give_item(player_id, item_def_id, qty)` | Editor (admin) | Admin grant |
| `equip_item(item_def_id)` | Client | Equip from inventory into the matching slot |
| `unequip_item(slot)` | Client | Unequip back to inventory |

### Loot (loot.rs)

| Reducer | Caller | Purpose |
|---------|--------|---------|
| `create_loot_table(enemy_def_id, item_def_id, drop_chance, min_qty, max_qty)` | Editor (admin) | Add a drop entry to an enemy archetype |
| `delete_loot_table(loot_table_id)` | Editor (admin) | Remove a drop entry |
| `pickup_item(item_drop_id)` | Client on F key | Proximity-validated pickup → adds to caller's `Inventory` |

`spawn_loot_drops` is a plain Rust helper called when an `Enemy` dies in `attack_enemy`.

### Atmosphere (atmosphere.rs)

| Reducer | Caller | Purpose |
|---------|--------|---------|
| `tick_world_time(_)` | Scheduler (1 Hz) | Advances `WorldClock.minutes_of_day` by 1 (mod 1440); re-schedules self |
| `change_weather(zone_id, kind, intensity)` | Editor (admin) | Upsert `WeatherState` for a zone; 0 ≤ intensity ≤ 1 |
| `set_zone_mood(zone_id, mood_preset_id)` | Editor (admin) | Update `Zone.mood_preset_id` |

## Admin model

`ADMIN_IDENTITIES` is a `const` array of 64-char hex identity strings (one per Anthropic / Linux account that builds + admins the server). Identities are seeded into the `Admin` table on `init` and any matching connecting identity is auto-inserted on `client_connected`, so the admin set survives `--delete-data` republishes only because the constant is re-evaluated each time `init` runs.

Use `spacetime login show` to look up your identity. Add it to `ADMIN_IDENTITIES` in [`lib.rs`](../../server/spacetimedb/src/lib.rs) and publish.

The `is_admin(ctx)` helper is the gate used by every authoring reducer.

## Module Conventions

```rust
// Tables: #[table(accessor = snake_case, public)]
// DO NOT also add #[derive(SpacetimeType)] to a table struct.
#[table(accessor = player, public)]
pub struct Player { /* ... */ }

// Embedded enums/structs use #[derive(SpacetimeType)]
#[derive(SpacetimeType, Clone, Copy, Debug, PartialEq)]
pub enum WeatherKind { Clear, Rain, Storm, Fog, Snow }

// Table access via methods (parentheses!)
ctx.db.player().identity().find(ctx.sender());
ctx.db.player().id().update(Player { mana: 0, ..old_player });

// Cross-module table accessors require explicit imports so the trait is in scope:
use crate::player as _;
```

> Wherever a module references a table defined in another module, the accessor trait must be imported (typically `use crate::<table> as _`) so the trait methods are visible on `ctx.db`. See the `use ... as _` lines at the top of every module file.

## Schedulers

Five scheduler tables drive recurring work. Each scheduled reducer re-inserts a new row at the end so the cadence is self-sustaining; if you ever clear one of these tables without re-bootstrapping, the corresponding system silently stops.

| Tick | Cadence | Reducer |
|------|---------|---------|
| `StatusEffectTick` | 1 s | `tick_status_effects` |
| `ManaRegenTick` | 2 s | `tick_mana_regen` |
| `CombatLogPruneTick` | 60 s | `prune_combat_log` |
| `AiTick` | 500 ms | `tick_ai` |
| `WorldClockTick` | 1 s | `tick_world_time` |
| `EnemyRespawnTick` | one-shot per kill | `tick_enemy_respawn` |

`client_connected` re-bootstraps any missing tick row, so a hot-publish (without `--delete-data`) never leaves the system dead.

## Build & Publish

```bash
# Build (compiles Rust → WASM) — run from server/
spacetime build

# Publish to local dev server
spacetime publish --server local zoneforge-server

# Publish with schema reset (breaking change — wipes ALL data including Admin)
spacetime publish --server local zoneforge-server --delete-data
```

> Always use `spacetime build`, not `cargo build` directly — it handles the WASM target and SpacetimeDB post-processing.

After `--delete-data`: `init` re-runs, re-seeds admins + abilities + scheduler ticks + `WorldClock`. You will need to recreate at least one zone before clients can spawn players.

## Project Layout

```text
server/
├── spacetime.json         ← points module-path to ./spacetimedb
└── spacetimedb/
    ├── .cargo/
    │   └── config.toml    ← sets default target: wasm32-unknown-unknown
    ├── Cargo.toml         ← spacetimedb = "2.0", crate-type = ["cdylib"]
    └── src/
        ├── lib.rs         ← Player, Zone, EntityInstance, Admin, init, client_connected, helpers
        ├── terrain.rs     ← TerrainChunk + update_terrain_chunk
        ├── combat.rs      ← Ability, PlayerCooldown, StatusEffect, CombatLog + ticks
        ├── enemy.rs       ← EnemyDefinition, SpawnPoint, Enemy + AI/respawn ticks
        ├── portal.rs      ← Portal + create_portal / enter_zone
        ├── inventory.rs   ← ItemDefinition, Inventory, Equipment + reducers
        ├── loot.rs        ← LootTable, ItemDrop, pickup_item, spawn_loot_drops helper
        └── atmosphere.rs  ← WorldClock, WeatherState + tick_world_time / change_weather / set_zone_mood
```

## Client Binding Generation

After any schema change, regenerate the C# bindings for both Unity projects:

```bash
# Run from client/ or editor/ directory
spacetime generate \
  --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

The generated files in `Assets/Scripts/autogen/` are not hand-edited — they are overwritten on each generation.

## Performance Notes

- SpacetimeDB holds all game state in memory; disk is only the commit log
- Heavy tables (`TerrainChunk`, `EntityInstance`, `Enemy`, `Portal`, `ItemDrop`, `WeatherState`) are subscribed with `WHERE zone_id = …` filters on the client; light tables (`Player`, `Ability`, `Inventory`, `WorldClock`, …) are unfiltered
- `prune_combat_log` caps `CombatLog` at 1000 rows so the client's full-table subscription stays bounded
- BTree indexes on hot filters: `Player.zone_id`, `TerrainChunk.zone_id`, `Enemy.zone_id`, `Portal.source_zone_id`/`dest_zone_id`, `Inventory.player_id`, `ItemDrop.zone_id`, `LootTable.enemy_def_id`, `SpawnPoint.zone_id`, `StatusEffect.target_id`, `PlayerCooldown.player_id`

## See Also

- [Overview.md](Overview.md) — System architecture and data flow
- [Diagrams.md](Diagrams.md) — Schema diagram and sequence diagrams
- [../guides/Server_Dev.md](../guides/Server_Dev.md) — Daily Rust/SpacetimeDB workflow
- [../guides/Getting_Started.md](../guides/Getting_Started.md) — Full environment setup
- [../decisions/001-spacetimedb.md](../decisions/001-spacetimedb.md) — Why SpacetimeDB
