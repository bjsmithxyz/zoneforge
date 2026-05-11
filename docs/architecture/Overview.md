# ZoneForge — System Architecture Overview

## Document History

| Version | Date | Summary |
|---------|------|---------|
| 1.2 | 2026-05-10 | Refreshed for Phase 5 (Groups 12–14): combat/enemy/inventory/loot/portal/atmosphere subsystems; server module split into files; expanded client/editor folder layouts; admin-gated reducers |
| 1.1 | 2026-03-24 | Updated table and reducer list to reflect current implementation; fixed `paint_terrain` → `update_terrain_chunk` |
| 1.0 | 2026-02-01 | Initial document |

---

## Three-Application Architecture

ZoneForge consists of three applications sharing a single SpacetimeDB backend:

```text
┌──────────────────────────┐   ┌──────────────────────────┐
│   zoneforge-editor       │   │   zoneforge-client        │
│   (Standalone Unity app) │   │   (Standalone Unity app)  │
│                          │   │                           │
│  World-building tools:   │   │  Game runtime:            │
│  - Zone creation         │   │  - Player movement        │
│  - Terrain painting      │   │  - Combat + enemy AI      │
│  - Entity placement      │   │  - Inventory / loot       │
│  - Portal authoring      │   │  - Zone transfer (portal) │
│  - EnemyDef + loot table │   │  - Atmosphere (sky/wx)    │
│  - Mood / weather debug  │   │  - HUD / UI               │
└────────────┬─────────────┘   └────────────┬──────────────┘
             │  SpacetimeDB C# SDK (WebSocket)│
             └──────────────┬────────────────┘
                            ▼
             ┌──────────────────────────────────┐
             │   SpacetimeDB + Rust Module       │
             │   (zoneforge-server)              │
             │                                  │
             │  Tables (authoritative state):   │
             │    Player, Zone, TerrainChunk,   │
             │    EntityInstance, Portal,       │
             │    Ability, PlayerCooldown,      │
             │    StatusEffect, CombatLog,      │
             │    EnemyDefinition, SpawnPoint,  │
             │    Enemy, ItemDefinition,        │
             │    Inventory, Equipment,         │
             │    LootTable, ItemDrop,          │
             │    WeatherState, WorldClock,     │
             │    Admin, + scheduler rows       │
             │                                  │
             │  Reducers (mutations):           │
             │    create_player, move_player,   │
             │    create_zone, delete_zone,     │
             │    update_terrain_chunk,         │
             │    spawn_entity,                 │
             │    use_ability, attack_enemy,    │
             │    respawn,                      │
             │    create_portal, enter_zone,    │
             │    spawn_enemy_manual, …,        │
             │    pickup_item, equip_item, …,   │
             │    change_weather, set_zone_mood │
             └──────────────────────────────────┘
```

**Editor writes → SpacetimeDB ↔ Client reads in real time.** A zone created or modified in the editor is immediately visible to all connected game clients with no manual sync step.

## Key Architectural Principles

**Server authority** — All game state mutations happen via reducers on the server. Both the editor and the client send intents, not state.

**Automatic sync** — SpacetimeDB pushes table changes to all subscribed clients. No polling, no manual WebSocket management.

**In-memory performance** — SpacetimeDB holds all active game state in memory, backed by a commit log for durability.

**Admin gating** — World-authoring reducers (`delete_zone`, `create_portal`, `change_weather`, `set_zone_mood`, `create_item_def`, `create_loot_table`, `create_enemy_def`, `create_spawn_point`, …) check the `Admin` table. Admin identities are seeded at compile time via `ADMIN_IDENTITIES` in `lib.rs` and auto-promoted on `client_connected`. Gameplay reducers (`move_player`, `use_ability`, `pickup_item`, …) are open to any connected identity.

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
3. Editor calls `update_terrain_chunk(zone_id, chunk_x, chunk_z, height_data, splat_data)` reducer
4. Server updates the `TerrainChunk` table row
5. SpacetimeDB pushes diff to all subscribers of `SELECT * FROM terrain_chunk WHERE zone_id = …`
6. All Unity clients receive `OnUpdate` callback; `TerrainRenderer` rebuilds the Mesh

Example (player movement):

1. Player presses W — Unity calls `move_player(new_x, new_y)` reducer
2. Server validates position (bounds, finite values)
3. Server updates `Player` table row
4. SpacetimeDB pushes diff to all clients subscribed to `SELECT * FROM player`
5. All Unity clients receive `OnUpdate` callback and update their local representation

## Subsystem Map

| Subsystem | Server module | Key tables | Client / editor home |
|-----------|---------------|------------|----------------------|
| Player & zone | `lib.rs` | `Player`, `Zone`, `Admin`, `EntityInstance` | `Player/`, `Runtime/`, `Zone/` |
| Terrain | `terrain.rs` | `TerrainChunk` (chunked 32×32) | `Runtime/TerrainRenderer.cs`, editor brushes |
| Combat | `combat.rs` | `Ability`, `PlayerCooldown`, `StatusEffect`, `CombatLog` + scheduler ticks | `Combat/`, `UI/HotbarUI.cs`, `UI/FloatingTextPopup.cs` |
| Enemies | `enemy.rs` | `EnemyDefinition`, `SpawnPoint`, `Enemy`, `EnemyRespawnTick`, `AiTick` | `Enemy/` (client), `EnemyDefCreationPanel.cs` (editor) |
| Portals | `portal.rs` | `Portal` | `Zone/PortalManager.cs`, `Zone/ZoneTransferManager.cs`, `WorldGraphPanel.cs` (editor) |
| Inventory | `inventory.rs` | `ItemDefinition`, `Inventory`, `Equipment` | `UI/InventoryManager.cs`, `InventoryUI.cs`, `EquipmentUI.cs` |
| Loot | `loot.rs` | `LootTable`, `ItemDrop` | `Zone/ItemDropRenderer.cs`, `Zone/ItemPickupManager.cs`, `LootCreationPanel.cs` (editor) |
| Atmosphere | `atmosphere.rs` | `WorldClock`, `WeatherState`, `WorldClockTick` (+ `mood_preset_id` on `Zone`) | `Runtime/AtmosphereController.cs`, `Runtime/AmbientAudioMixer.cs`, `WeatherDebugPanel.cs` (editor) |

## Repository Structure

```text
zoneforge/                   ← umbrella repo
├── client/                  ← Unity 3D URP game client (submodule)
│   └── Assets/
│       ├── Scripts/
│       │   ├── Runtime/     ← SpacetimeDBManager, Terrain/Water/NavMesh,
│       │   │                  EntityInstanceManager, AtmosphereController,
│       │   │                  AmbientAudioMixer, LookupCache
│       │   ├── Player/      ← PlayerManager, PlayerController
│       │   ├── Combat/      ← CombatManager, CombatInputHandler, pooling, SelectionTarget
│       │   ├── Enemy/       ← EnemyManager, EnemyController, EnemyHealthBar
│       │   ├── Zone/        ← ZoneController, PortalManager, ZoneTransferManager,
│       │   │                  ItemDropRenderer, ItemPickupManager, RippleWarpEffect
│       │   ├── UI/          ← HotbarUI, SelfHudUI, PlayerHealthBar, FloatingTextPopup,
│       │   │                  InventoryManager, InventoryUI, EquipmentUI
│       │   ├── Data/        ← WorldData, ZoneVisualData, MoodPreset, MoodPresetRegistry
│       │   └── autogen/     ← `spacetime generate` output (do not edit)
│       ├── Prefab/          ← Combat prefabs (FireballProjectile, ImpactFire/GenericVFX)
│       ├── Resources/
│       │   ├── MoodPresets/ ← village_day, forest, night_camp
│       │   └── WeatherVFX/  ← Rain/fog prefabs
│       └── Art/             ← Models, Sprites, Materials
├── editor/                  ← Unity 3D URP standalone world editor (submodule)
│   └── Assets/
│       ├── Scripts/
│       │   ├── Runtime/     ← SpacetimeDBManager, terrain renderer/painter + brushes,
│       │   │                  EntityRenderer/Placer, EnemyRenderer, PortalRenderer,
│       │   │                  AtmosphereController, AmbientAudioMixer, CameraController,
│       │   │                  ZoneCreationPanel, TilePalettePanel, EntityPalettePanel,
│       │   │                  EnemyDefCreationPanel, LootCreationPanel,
│       │   │                  WorldGraphPanel, WeatherDebugPanel, ToolbarController,
│       │   │                  UIHoverTracker
│       │   ├── Data/        ← WorldData, ZoneVisualData, MoodPreset, MoodPresetRegistry
│       │   ├── Editor/      ← Placeholder asset generators (Editor-only dev tools)
│       │   └── autogen/     ← `spacetime generate` output (do not edit)
│       ├── UI/              ← .uxml/.uss for all editor panels
│       ├── Resources/       ← MoodPresets + WeatherVFX (mirrors client/)
│       └── Tests/EditMode/  ← EditMode tests
└── server/                  ← SpacetimeDB Rust module (submodule)
    └── spacetimedb/
        └── src/
            ├── lib.rs        ← Player, Zone, EntityInstance, Admin, init, client_connected
            ├── terrain.rs    ← TerrainChunk + update_terrain_chunk
            ├── combat.rs     ← Ability, PlayerCooldown, StatusEffect, CombatLog, ticks
            ├── enemy.rs      ← EnemyDefinition, SpawnPoint, Enemy, AI + respawn ticks
            ├── portal.rs     ← Portal + create_portal/enter_zone
            ├── inventory.rs  ← ItemDefinition, Inventory, Equipment
            ├── loot.rs       ← LootTable, ItemDrop, pickup_item
            └── atmosphere.rs ← WorldClock, WeatherState, world tick + admin commands
```

## See Also

- [Diagrams.md](Diagrams.md) — Mermaid diagrams: architecture, schema, and workflow sequences
- [Client.md](Client.md) — Game client architecture detail
- [Editor.md](Editor.md) — World editor architecture detail
- [Server.md](Server.md) — SpacetimeDB module architecture detail
- [../decisions/001-spacetimedb.md](../decisions/001-spacetimedb.md) — Why SpacetimeDB was chosen
- [../design/Detailed_Design.md](../design/Detailed_Design.md) — Full system design document
