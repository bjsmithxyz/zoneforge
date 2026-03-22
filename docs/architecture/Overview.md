# ZoneForge — System Architecture Overview

## Three-Application Architecture

ZoneForge consists of three applications sharing a single SpacetimeDB backend:

```text
┌──────────────────────────┐   ┌──────────────────────────┐
│   zoneforge-editor       │   │   zoneforge-client        │
│   (Standalone Unity app) │   │   (Standalone Unity app)  │
│                          │   │                           │
│  World-building tools:   │   │  Game runtime:            │
│  - Zone creation         │   │  - Player movement        │
│  - Terrain painting      │   │  - Combat                 │
│  - Entity placement      │   │  - UI / inventory         │
│  - Zone management       │   │  - Rendering              │
└────────────┬─────────────┘   └────────────┬──────────────┘
             │  SpacetimeDB C# SDK (WebSocket)│
             └──────────────┬────────────────┘
                            ▼
             ┌──────────────────────────────────┐
             │   SpacetimeDB + Rust Module       │
             │   (zoneforge-server)              │
             │                                  │
             │  ┌────────────────────────────┐  │
             │  │  Tables (authoritative     │  │
             │  │  game state)               │  │
             │  │  Player, Zone,             │  │
             │  │  TerrainChunk, Entity,     │  │
             │  │  Ability, CombatLog,       │  │
             │  │  PlayerCooldown,           │  │
             │  │  StatusEffect...           │  │
             │  └────────────────────────────┘  │
             │  ┌────────────────────────────┐  │
             │  │  Reducers (mutations)      │  │
             │  │  create_zone, paint_terrain│  │
             │  │  move_player, spawn_entity │  │
             │  │  use_ability, respawn      │  │
             │  └────────────────────────────┘  │
             └──────────────────────────────────┘
```

**Editor writes → SpacetimeDB ↔ Client reads in real time.** A zone created or modified in the editor is immediately visible to all connected game clients with no manual sync step.

## Key Architectural Principles

**Server authority** — All game state mutations happen via reducers on the server. Both the editor and the client send intents, not state.

**Automatic sync** — SpacetimeDB pushes table changes to all subscribed clients. No polling, no manual WebSocket management.

**In-memory performance** — SpacetimeDB holds all active game state in memory, backed by a commit log for durability. This provides 100–1000x better throughput than traditional DB + API stacks.

**Single-binary deployment** — The entire server is a Rust WASM module. Deploy with `spacetime publish`.

**Standalone applications** — Both the editor and the game client are compiled standalone desktop apps. Neither requires the Unity Editor to run.

## Data Flow

**Editor authoring:**

```text
Designer Input → Editor UI → Reducer Call → SpacetimeDB → Table Update → All Subscribers
```

**Player gameplay:**

```text
Player Input → Client → Reducer Call → SpacetimeDB → Table Update → All Clients
```

Example (terrain painting in editor):

1. Designer clicks and drags on terrain — `TerrainPainter` raycasts to the Mesh
2. Brush modifies in-memory `TerrainChunkData` (height/splat arrays)
3. Editor calls `paint_terrain(zone_id, chunk_x, chunk_z, height_data, splat_data)` reducer
4. Server updates the `TerrainChunk` table row
5. SpacetimeDB pushes diff to all subscribers of `SELECT * FROM terrain_chunk`
6. All Unity clients receive `OnUpdate` callback; `TerrainRenderer` rebuilds the Mesh

Example (player movement):

1. Player presses W — Unity calls `move_player(new_x, new_z)` reducer
2. Server validates position (bounds, collision)
3. Server updates `Player` table row
4. SpacetimeDB pushes diff to all clients subscribed to `SELECT * FROM player`
5. All Unity clients receive `OnUpdate` callback and update their local representation

## Repository Structure

```text
zoneforge/                   ← umbrella repo
├── client/                  ← Unity 3D URP game client (submodule)
│   └── Assets/
│       ├── Scripts/
│       │   ├── Data/        ← WorldData, ZoneVisualData, TerrainChunkData
│       │   ├── Network/     ← SpacetimeDBManager (Zone, TerrainChunk, EntityInstance)
│       │   ├── Zone/        ← ZoneController, TerrainRenderer, WaterRenderer
│       │   └── autogen/     ← Generated C# bindings (spacetime generate)
│       └── Art/
│           ├── Models/
│           └── Materials/
├── editor/                  ← Unity 3D URP standalone world editor (submodule)
│   └── Assets/
│       ├── Scripts/
│       │   ├── Runtime/     ← SpacetimeDBManager, TerrainRenderer, TerrainPainter, brush classes, UI panels
│       │   ├── Data/        ← WorldData, ZoneVisualData, TerrainChunkData
│       │   └── autogen/     ← Generated C# bindings (spacetime generate)
│       └── UI/              ← ZoneCreationPanel + TilePalettePanel (uxml/uss)
└── server/                  ← SpacetimeDB Rust module (submodule)
    └── spacetimedb/
        └── src/
            └── lib.rs       ← All tables and reducers
```

## See Also

- [Client.md](Client.md) — Game client architecture detail
- [Editor.md](Editor.md) — World editor architecture detail
- [Server.md](Server.md) — SpacetimeDB module architecture detail
- [../decisions/001-spacetimedb.md](../decisions/001-spacetimedb.md) — Why SpacetimeDB was chosen
- [../design/Detailed_Design.md](../design/Detailed_Design.md) — Full system design document
