# ZoneForge — SpacetimeDB Server Architecture

## Document History

| Version | Date | Summary |
|---------|------|---------|
| 1.2 | 2026-03-24 | Added `ManaRegenTick` table; added `tick_mana_regen`, `client_connected` reducers; renamed `paint_terrain` → `update_terrain_chunk`; corrected `spawn_entity` signature |
| 1.1 | 2026-03-07 | Added combat tables (Ability, PlayerCooldown, StatusEffect, CombatLog, StatusEffectTick) and reducers |
| 1.0 | 2026-02-01 | Initial document |

---

## Overview

The server is a single Rust WASM module published to SpacetimeDB. It contains all authoritative game logic: table definitions (schema) and reducers (mutations).

**Module name:** `zoneforge-server`
**Language:** Rust (compiled to `wasm32-unknown-unknown`)
**Entry point:** `server/spacetimedb/src/lib.rs`

## Core Tables

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| `Player` | `id` (PK, auto_inc), `identity` (unique) | Player state: position, health, mana, is_dead, zone |
| `Zone` | `id` (PK, auto_inc), `name` | Zone metadata: dimensions, `water_level` |
| `TerrainChunk` | `id` (PK, auto_inc), `zone_id`, `chunk_x`, `chunk_z` | Per-chunk heightmap (`height_data` byte[]) and splatmap (`splat_data` byte[]) |
| `EntityInstance` | `id` (PK, auto_inc), `zone_id` | Placed entities: props, NPCs, enemies |
| `Ability` | `id` (PK, auto_inc) | Ability definitions: damage, cooldown_ms, mana_cost, range, ability_type |
| `PlayerCooldown` | `id` (PK, auto_inc), `player_id` (btree) | Per-player per-ability cooldown expiry timestamps |
| `StatusEffect` | `id` (PK, auto_inc), `target_id` (btree) | Active DoT/debuff effects: type, expires_at, damage_per_tick |
| `CombatLog` | `id` (PK, auto_inc) | Immutable record of every damage/heal event |
| `StatusEffectTick` | `scheduled_id` (PK, auto_inc) | Scheduler row — triggers `tick_status_effects` every 1 s |
| `ManaRegenTick` | `scheduled_id` (PK, auto_inc) | Scheduler row — triggers `tick_mana_regen` every 2 s |

**Planned tables** (added per phase):

- `Portal`, `WorldEvent` — Phase 4 zone stitching (Group 10)
- `Inventory`, `Equipment`, `Item` — Phase 5 inventory (Group 13)
- `Quest`, `QuestProgress` — Phase 5 quest system (Group 12)
- `Party`, `Guild` — Phase 6 multiplayer features (Group 15–16)

## Core Reducers

| Reducer | Caller | Purpose |
|---------|--------|---------|
| `create_player(name)` | Client on connect | Register new player (zone_id defaults to 1); idempotent — skips if identity already has a row |
| `move_player(x, y)` | Client on input | Server-validated movement; clamps to zone bounds; returns `Result` |
| `create_zone(name, w, h, water_level)` | Editor on zone creation | Create zone + initialise flat `TerrainChunk` rows for every chunk in the grid |
| `update_terrain_chunk(zone_id, chunk_x, chunk_z, height_data, splat_data)` | Editor on brush stroke | Update a chunk's heightmap and/or splatmap; validates size (4096 bytes each) and bounds |
| `spawn_entity(zone_id, prefab, x, y, elevation, entity_type)` | Editor on entity placement | Place entity in zone; `elevation` is world-space Y; `entity_type` is a plain string |
| `use_ability(ability_id, target_id)` | Client on key press | Server-authoritative: validates range, cooldown, mana; calls `apply_damage` |
| `respawn()` | Client on R key | Resets dead player to zone centre with full health/mana; clears status effects |
| `tick_status_effects(_)` | Scheduler (1 Hz) | Applies DoT ticks, removes expired effects, re-schedules self |
| `tick_mana_regen(_)` | Scheduler (2 Hz) | Restores 10 mana to all living players; re-schedules self |
| `client_connected` | SpacetimeDB lifecycle | Bootstraps mana regen tick if missing (handles hot-publish without data wipe) |

`apply_damage` is a plain Rust helper function (not a reducer) called internally by `use_ability` and `tick_status_effects`.

## Module Conventions

```rust
// Tables use #[table(accessor = snake_case, public)]
// Do NOT add #[derive(SpacetimeType)] to table structs — only to embedded types
#[table(accessor = player, public)]
pub struct Player { ... }

// Custom embedded types (not tables) use #[derive(SpacetimeType)]
#[derive(SpacetimeType)]
pub enum EntityType { StaticProp, Interactive, NPC, Enemy, Player, TriggerVolume }

// Reducers access tables via ctx.db.<table_name>()
// Look up by unique field: ctx.db.player().identity().find(ctx.sender())
// Update: ctx.db.player().id().update(Player { ..old_player, new_field: value })
// Note: ctx.sender() is a METHOD call in SpacetimeDB 2.x (not a field)
```

## Build & Publish

```bash
# Build (compiles Rust → WASM) — run from server/
spacetime build

# Publish to local dev server
spacetime publish --server local zoneforge-server

# Publish with schema reset (breaking change)
spacetime publish --server local zoneforge-server --delete-data
```

> Always use `spacetime build`, not `cargo build` directly — it handles the WASM target and SpacetimeDB post-processing.

## Project Layout

```
server/
├── spacetime.json         ← points module-path to ./spacetimedb
└── spacetimedb/
    ├── .cargo/
    │   └── config.toml    ← sets default target: wasm32-unknown-unknown
    ├── Cargo.toml         ← spacetimedb = "2.0", crate-type = ["cdylib"]
    └── src/
        └── lib.rs         ← all tables and reducers
```

As the module grows, `lib.rs` will be split into modules (e.g. `player.rs`, `combat.rs`, `zone.rs`) and referenced from `lib.rs` via `mod`.

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
- Add indexes to columns used in hot-path queries (Phase 3 optimization)
- Batch reducer operations where possible to reduce round-trips
- Target: 100+ concurrent players per zone (validated in Sprint 18 load testing)

## See Also

- [Overview.md](Overview.md) — System architecture and data flow
- [../guides/Server_Dev.md](../guides/Server_Dev.md) — Daily Rust/SpacetimeDB workflow
- [../guides/Getting_Started.md](../guides/Getting_Started.md) — Full environment setup
- [../decisions/001-spacetimedb.md](../decisions/001-spacetimedb.md) — Why SpacetimeDB
