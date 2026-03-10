# ZoneForge — SpacetimeDB Server Architecture

## Overview

The server is a single Rust WASM module published to SpacetimeDB. It contains all authoritative game logic: table definitions (schema) and reducers (mutations).

**Module name:** `zoneforge-server`
**Language:** Rust (compiled to `wasm32-unknown-unknown`)
**Entry point:** `server/src/lib.rs`

## Core Tables

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| `Player` | `id` (PK, auto_inc), `identity` (unique) | Player state: position, health, zone |
| `Zone` | `id` (PK, auto_inc), `name` | Zone metadata: dimensions |
| `EntityInstance` | `id` (PK, auto_inc), `zone_id` | Placed entities: props, NPCs, enemies |

**Planned tables** (added per phase):
- `Ability`, `StatusEffect`, `CombatLog` — Phase 2 combat
- `Portal`, `WorldEvent` — Phase 2 zone stitching
- `Inventory`, `Equipment`, `Item` — Phase 2 inventory
- `Quest`, `QuestProgress` — Phase 3 quest system
- `Party`, `Guild` — Phase 4 multiplayer features

## Core Reducers

| Reducer | Caller | Purpose |
|---------|--------|---------|
| `create_player(name)` | Client on connect | Register new player |
| `move_player(x, y)` | Client on input | Server-validated movement |
| `create_zone(name, w, h)` | Editor tool | Create new zone |
| `spawn_entity(zone_id, prefab, x, y, type)` | Editor tool | Place entity in zone |

## Module Conventions

```rust
// Tables use #[table(name = snake_case, public)]
// Do NOT add #[derive(SpacetimeType)] to table structs — only to embedded types
#[table(name = player, public)]
pub struct Player { ... }

// Custom embedded types (not tables) use #[derive(SpacetimeType)]
#[derive(SpacetimeType)]
pub enum EntityType { StaticProp, Interactive, NPC, Enemy, Player, TriggerVolume }

// Reducers access tables via ctx.db.<table_name>()
// Look up by unique field: ctx.db.player().identity().find(ctx.sender)
// Update: ctx.db.player().id().update(Player { ..old_player, new_field: value })
```

## Build & Publish

```bash
# Build (compiles Rust → WASM)
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
├── .cargo/
│   └── config.toml        ← sets default target: wasm32-unknown-unknown
├── Cargo.toml             ← spacetimedb = "1.0", crate-type = ["cdylib"]
└── src/
    └── lib.rs             ← all tables and reducers
```

As the module grows, `lib.rs` will be split into modules (e.g. `player.rs`, `combat.rs`, `zone.rs`) and referenced from `lib.rs` via `mod`.

## Client Binding Generation

After any schema change, regenerate the C# bindings for Unity:

```bash
# Run from the Unity client root (ZoneForge/client/)
spacetime generate \
  --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
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
