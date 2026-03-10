# ZoneForge — System Architecture Overview

## Hybrid Client-Server Model

ZoneForge uses a strict separation between a Unity client and an authoritative SpacetimeDB server. No game-critical logic runs on the client.

```
┌─────────────────────────────┐       ┌──────────────────────────────┐
│         Unity Client         │       │    SpacetimeDB + Rust Module  │
│                             │       │                              │
│  ┌─────────────────────┐   │  SDK   │  ┌────────────────────────┐ │
│  │  Editor Tools       │◄──┼────────┼─►│  Tables (game state)   │ │
│  │  (ZoneEditor, etc.) │   │        │  │  Player, Zone, Entity  │ │
│  └─────────────────────┘   │        │  │  Combat, Inventory...  │ │
│  ┌─────────────────────┐   │  WebSocket  └────────────────────────┘ │
│  │  Runtime / Gameplay │◄──┼────────┼─►┌────────────────────────┐ │
│  │  Rendering, UI, Input│   │  subscriptions  │  Reducers (logic)    │ │
│  └─────────────────────┘   │        │  │  create_player,        │ │
│                             │        │  │  move_player,          │ │
└─────────────────────────────┘        │  │  use_ability, etc.     │ │
                                       │  └────────────────────────┘ │
                                       └──────────────────────────────┘
```

## Key Architectural Principles

**Server authority** — All game state mutations happen via reducers on the server. The client sends intents (e.g. `move_player`), not state.

**Automatic sync** — SpacetimeDB pushes table changes to all subscribed clients. No polling, no manual WebSocket management.

**In-memory performance** — SpacetimeDB holds all active game state in memory, backed by a commit log for durability. This provides 100–1000x better throughput than traditional DB + API stacks.

**Single-binary deployment** — The entire server is a Rust WASM module. Deploy with `spacetime publish`.

## Data Flow

```
Player Input → Unity → Reducer Call → SpacetimeDB → Table Update → All Subscribers
```

1. Player presses W — Unity calls `move_player(new_x, new_y)` reducer
2. Server validates position (bounds, collision)
3. Server updates `Player` table row
4. SpacetimeDB pushes diff to all clients subscribed to `SELECT * FROM player`
5. All Unity clients receive `OnUpdate` callback and update their local representation

## Repository Structure

```
zoneforge/           ← umbrella repo (this repo)
├── client/          ← Unity project (submodule)
│   └── Assets/
│       ├── Scripts/
│       │   ├── Data/        ← ScriptableObjects (WorldData, ZoneVisualData)
│       │   ├── Network/     ← SpacetimeDB connection management
│       │   ├── Zone/        ← ZoneController, zone runtime logic
│       │   ├── Editor/      ← Unity Editor window scripts
│       │   └── autogen/     ← Generated C# bindings (spacetime generate)
│       └── Art/
│           ├── Models/
│           ├── Materials/
│           │   ├── Graphs/      ← Material Maker .mmg source files
│           │   ├── Exports/     ← Exported texture maps
│           │   └── Parameters/
│           └── Tiles/
└── server/          ← SpacetimeDB Rust module (submodule)
    └── src/
        └── lib.rs   ← All tables and reducers
```

## See Also

- [Client.md](Client.md) — Unity-specific architecture detail
- [Server.md](Server.md) — SpacetimeDB module architecture detail
- [../decisions/001-spacetimedb.md](../decisions/001-spacetimedb.md) — Why SpacetimeDB was chosen
- [../design/Detailed_Design.md](../design/Detailed_Design.md) — Full system design document
