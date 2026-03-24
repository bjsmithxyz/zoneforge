# Detailed Design Document

## ZoneForge

### Multiplayer-Ready Isometric RPG World Editor

---

**Project Name:** ZoneForge
**Client Engine:** Unity 2022.3 LTS
**Backend:** SpacetimeDB 2.x with Rust
**Document Version:** 2.1
**Date:** March 24, 2026
**Target Platform:** Windows, macOS, Linux, WebGL (Client) | Cloud/Self-Hosted (Server)

---

## Document History

| Version | Date | Summary |
|---------|------|---------|
| 2.1 | 2026-03-24 | Updated all SpacetimeDB examples to 2.x API; corrected table schemas (Zone, Player, EntityInstance); replaced 2D tile-layer system with terrain chunk system; removed unused Unity packages; noted missing AI server tables |
| 2.0 | 2026-03-07 | SpacetimeDB integration — replaced local file persistence with server tables and reducers |
| 1.0 | 2026-01-15 | Initial design document |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Project Overview](#2-project-overview)
3. [Technical Foundation](#3-technical-foundation)
4. [System Architecture](#4-system-architecture)
5. [Map and World System](#5-map-and-world-system)
6. [Entity System](#6-entity-system)
7. [Event and Trigger System](#7-event-and-trigger-system)
8. [Combat and Spell System](#8-combat-and-spell-system)
   - [8A. Object Pooling System](#8a-object-pooling-system)
9. [Asset Pipeline](#9-asset-pipeline)
   - [9A. Procedural Texturing Pipeline (Material Maker)](#9a-procedural-texturing-pipeline)
   - [9B. Smart Materials & ArmorPaint Workflow](#9b-smart-materials--armorpaint-workflow)
   - [9C. Geometry Nodes: Procedural Foliage & Decoration](#9c-geometry-nodes-procedural-foliage--decoration)
10. [Lighting and VFX System](#10-lighting-and-vfx-system)
11. [Editor UI and Tools](#11-editor-ui-and-tools)
12. [Development Roadmap](#12-development-roadmap)
13. [Technical Specifications](#13-technical-specifications)
14. [Risks and Mitigations](#14-risks-and-mitigations)
15. [Success Metrics](#15-success-metrics)
16. [Appendices](#16-appendices)

---

## 1. Executive Summary

This document outlines the complete technical design for **ZoneForge**, a multiplayer-ready Isometric RPG World Editor built with a hybrid architecture: Unity for the client-side editor and runtime, and **SpacetimeDB with Rust** for the authoritative server backend. The tool is inspired by industry-standard editors like WoWEdit and StarEdit, designed to empower solo developers and small teams to create professional-quality isometric RPGs with built-in multiplayer capabilities.

### Key Innovation: Multiplayer-First Architecture

ZoneForge leverages SpacetimeDB, a real-time backend framework and database that embeds server logic directly into the database for extreme speed and automatic client synchronization. This eliminates the need for traditional web servers, caching layers, and complex infrastructure management.

### The editor will provide:

- Grid-based map creation with multiple zone support
- Visual event/trigger system for game logic
- Asset management pipeline for 3D models, textures, and audio
- NPC and enemy AI configuration tools
- Combat and spell system designer
- Lighting and VFX placement system
- In-editor playtesting capabilities
- Real-time multiplayer support with automatic state synchronization
- Cloud-hosted or self-hosted server deployment

---

## 2. Project Overview

### 2.1 Vision Statement

Create a multiplayer-ready world editor that democratizes isometric RPG development by providing professional-grade tools with an intuitive interface and built-in online play capabilities. ZoneForge bridges the gap between technical complexity and creative freedom, allowing developers to focus on world-building and narrative design while SpacetimeDB handles all backend infrastructure, real-time synchronization, and multiplayer logic automatically.

### 2.2 Target Audience

- Solo indie developers building their first RPG
- Small development teams (2-5 people) without dedicated tools engineers
- Game designers with narrative/level design experience but limited programming skills
- Developers familiar with RTS/RPG editors seeking modern workflow improvements

### 2.3 Success Criteria

- Create a complete playable zone (town with 5 NPCs, 1 quest, 1 combat encounter) in under 4 hours
- Non-programmers can build event-driven gameplay without writing code
- Support worlds with 50+ interconnected zones without performance degradation
- Export playable builds to Windows, macOS, Linux, and WebGL
- Deploy multiplayer servers with a single command
- Support 100+ concurrent players per zone with real-time state synchronization
- Zero-downtime server updates for live games

---

## 3. Technical Foundation

### 3.1 Architecture Overview: Hybrid Client-Server Model

ZoneForge uses a **hybrid architecture** that combines the best of both worlds:

**Client-Side (Unity)**:

- World editor tooling and visual design interface
- Asset management and import pipeline
- 3D rendering, lighting, and VFX
- Local playtesting and development
- Compiled game client for end-users

**Server-Side (SpacetimeDB + Rust)**:

- Authoritative game state and logic
- Real-time multiplayer synchronization
- Combat calculations and validation
- Player authentication and permissions
- Persistent world storage

This separation ensures that:

- Game logic runs on the server (preventing cheating)
- Clients automatically stay synchronized
- Developers can update game logic without rebuilding clients
- Multiplayer "just works" without custom netcode

### 3.2 Why SpacetimeDB?

SpacetimeDB is a real-time backend framework and database for apps and games that handles all the persistence, logic, deployment, and real-time sync in a single cohesive backend.

**Key Advantages for ZoneForge:**

1. **Zero Infrastructure Management**
    
    - No separate web servers, containers, Kubernetes, or VMs
    - Clients connect directly to the database and execute application logic in your module
    - Deploy backend as a single binary
2. **Automatic Real-Time Sync**
    
    - SpacetimeDB automatically publishes updates to your clients so they're always in sync
    - No polling, no WebSockets to manage, no manual state updates
    - Subscribe to tables and get live updates automatically
3. **Extreme Performance**
    
    - All application state is held in memory for fast access, while a commit log on disk provides durability and crash recovery
    - 100x-1000x better performance than traditional database + web server architectures
    - Optimized for high-concurrency game scenarios
4. **Developer Productivity**
    
    - Write your entire application in a single language and deploy it as a single binary
    - Tables (your data) and reducers (your logic) replace complex REST APIs
    - ACID guarantees eliminate race conditions and state corruption
5. **Production-Proven**
    
    - Powering BitCraft Online MMORPG by Clockwork Labs
    - Multiple teams already run SpacetimeDB in production

### 3.3 Core Technology Stack

|Category|Technology|Purpose|
|---|---|---|
|**Client Engine**|Unity 2022.3 LTS|Editor UI, 3D rendering, asset pipeline|
|**Client Rendering**|Universal Render Pipeline (URP)|Optimized 3D isometric rendering|
|**Client Scripting**|C#|Unity editor extensions, UI logic|
|**Backend Database**|SpacetimeDB|Real-time database with embedded logic|
|**Backend Language**|Rust|Server-side game logic (reducers, tables)|
|**Client SDK**|SpacetimeDB C# SDK|Unity ↔ SpacetimeDB communication|
|**Visual Scripting**|Unity Visual Scripting|Designer-friendly trigger system (compiled to Rust reducers)|
|**AI Pathfinding**|Unity NavMesh (client) + Custom Rust (server)|Dual pathfinding for editor preview and authoritative server|
|**Version Control**|Git with Git LFS|Source control for Unity assets and Rust code|
|**Deployment**|SpacetimeDB Cloud or Self-Hosted|One-command server deployment|

### 3.4 Development Environment

#### Required Software

**Client Development:**

- Unity 2022.3 LTS or newer
- Visual Studio 2022 or JetBrains Rider (C# IDE)
- Blender 3.6+ (3D modeling, Geometry Nodes, asset export)
- Material Maker (procedural texture graph editor — tileable textures, normal maps)
- ArmorPaint (3D texture painting with smart materials — edge wear, dirt, AO)
- GIMP or Photoshop (texture compositing and cleanup)

**Server Development:**

- Rust toolchain (rustc 1.70+, cargo)
- SpacetimeDB CLI (`spacetime` command)
- Visual Studio Code with rust-analyzer (recommended)

**Version Control:**

- Git with Git LFS for version control

#### Unity Packages

- **Universal Render Pipeline (URP)** — Core 3D rendering pipeline
- **UI Toolkit** — Runtime world-building UI panels (editor app)
- **Cinemachine** — Camera control
- **Timeline** — Cutscenes
- **TextMeshPro** — High-quality UI text
- **SpacetimeDB Unity SDK** — Real-time backend integration

#### Rust Crates (Server Dependencies)

```toml
[dependencies]
spacetimedb = "2.0"           # Core SpacetimeDB framework
log = "0.4"
```

### 3.5 SpacetimeDB Architecture Fundamentals

#### Tables (Data Schema)

In SpacetimeDB, tables define your game's persistent state. Example:

```rust
use spacetimedb::{table, reducer, ReducerContext, Identity, Table};

// Tables use #[table(accessor = snake_case, public)]
// Do NOT add #[derive(SpacetimeType)] to table structs.
// SpacetimeType is only for custom embedded types used as fields inside rows.
#[table(accessor = player, public)]
pub struct Player {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    #[unique]
    pub identity: Identity,  // SpacetimeDB user authentication
    pub name: String,
    pub zone_id: u64,
    pub position_x: f32,
    pub position_y: f32,
    pub health: i32,
    pub max_health: i32,
    pub mana: i32,
    pub max_mana: i32,
    pub is_dead: bool,
}
```

#### Reducers (Server Logic)

Reducers are server-side functions that modify the database. They execute atomically with ACID guarantees:

```rust
// ctx.sender() is a method call in SpacetimeDB 2.x (not a field)
#[reducer]
pub fn move_player(ctx: &ReducerContext, new_x: f32, new_y: f32) -> Result<(), String> {
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or_else(|| "Player not found".to_string())?;

    let zone = ctx.db.zone().id().find(&player.zone_id)
        .ok_or_else(|| "Zone not found".to_string())?;

    let clamped_x = new_x.clamp(0.0, zone.terrain_width as f32);
    let clamped_y = new_y.clamp(0.0, zone.terrain_height as f32);

    ctx.db.player().id().update(Player {
        position_x: clamped_x,
        position_y: clamped_y,
        ..player
    });

    // All subscribed clients automatically receive this update!
    Ok(())
}
```

#### Client Integration (Unity C#)

Unity clients subscribe to tables and call reducers via the SpacetimeDB 2.x C# SDK:

```csharp
using SpacetimeDB;
using UnityEngine;

// SpacetimeDBManager.cs — singleton that manages the connection
public class SpacetimeDBManager : MonoBehaviour
{
    public static DbConnection Conn { get; private set; }

    void Start()
    {
        Conn = DbConnection.Builder()
            .WithUri("http://localhost:3000")
            .WithModuleName("zoneforge-server")
            .OnConnect(OnConnect)
            .Build();
    }

    void Update() => Conn?.FrameTick();

    void OnConnect(DbConnection conn, Identity identity, string token)
    {
        conn.SubscriptionBuilder()
            .OnApplied(ctx => {
                // Register callbacks after subscription is ready
                conn.Db.Player.OnUpdate += OnPlayerMoved;
            })
            .Subscribe("SELECT * FROM player");
    }

    void OnPlayerMoved(EventContext ctx, Player oldPlayer, Player newPlayer)
    {
        // Automatically called when ANY player row changes
        // Update Unity GameObject position
        // transform.position = new Vector3(newPlayer.PositionX, 0, newPlayer.PositionY);
    }
}

// In PlayerController.cs — call a reducer on input
void Update()
{
    if (Input.GetKey(KeyCode.W))
    {
        Reducer.MovePlayer(SpacetimeDBManager.Conn,
            transform.position.x,
            transform.position.z + speed * Time.deltaTime);
    }
}
```

### 3.6 Why Rust for Server Logic?

**Performance:**

- Zero-cost abstractions and no garbage collection
- Compiles to WebAssembly for SpacetimeDB modules
- Memory-safe concurrency without data races

**Safety:**

- Compile-time prevention of null pointer errors, buffer overflows
- Type system catches logic errors before runtime
- Perfect for authoritative server logic that must never fail

**Ecosystem:**

- Excellent game development crates (nalgebra, rand, pathfinding)
- First-class SpacetimeDB support
- Growing game dev community

**Developer Experience:**

- Expressive type system makes complex game logic manageable
- Cargo build system and package manager
- Excellent tooling (rust-analyzer, cargo-fmt, clippy)

---

## 4. System Architecture

### 4.1 High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                   UNITY CLIENT LAYER                     │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Editor Tools (Zone Editor, Entity Palette, etc.)│  │
│  └───────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Unity Rendering (URP, Lighting, VFX, Audio)     │  │
│  └───────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────┐  │
│  │  SpacetimeDB C# SDK (Subscriptions + Reducers)   │  │
│  └────────────────────┬──────────────────────────────┘  │
└───────────────────────┼──────────────────────────────────┘
                        │
                WebSocket Connection
              (Auto-sync, Real-time)
                        │
┌───────────────────────┼──────────────────────────────────┐
│              SPACETIMEDB SERVER LAYER                     │
│  ┌────────────────────▼──────────────────────────────┐  │
│  │         SpacetimeDB Runtime Engine                │  │
│  │  (WASM Module Execution + Database in Memory)     │  │
│  └───────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────┐  │
│  │       Rust Game Logic (compiled to WASM)          │  │
│  │   ┌──────────────────────────────────────────┐    │  │
│  │   │  Tables: Player, Zone, TerrainChunk,    │    │  │
│  │   │  EntityInstance, Ability, CombatLog,   │    │  │
│  │   │  PlayerCooldown, StatusEffect...        │    │  │
│  │   │  Reducers: create_zone, move_player,   │    │  │
│  │   │  update_terrain_chunk, use_ability...  │    │  │
│  │   └──────────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Commit Log (Disk Persistence + Crash Recovery)   │  │
│  └───────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
```

### 4.2 Client-Server Responsibilities

|Component|Responsibilities|Technology|
|---|---|---|
|**Unity Client**|Rendering, UI, asset management, editor tools, local playtesting|C# + Unity|
|**SpacetimeDB Server**|Game state, combat logic, player validation, persistence, real-time sync|Rust + SpacetimeDB|
|**SpacetimeDB SDK**|WebSocket connection, table subscriptions, reducer calls|C# SDK for Unity|

### 4.3 Data Flow Example: Player Movement

1. **Player Input** (Unity): User presses 'W' key
2. **Client Validation** (Unity): Check if movement is valid (cooldown, stunned, etc.)
3. **Call Reducer** (C# SDK): `Reducers.move_player(new_x, new_y)`
4. **Server Execution** (Rust):
    - Validate movement (speed hacks, wall clipping)
    - Update Player table position
    - Check for collision with NPCs/enemies
    - Trigger zone events if applicable
5. **Auto-Sync** (SpacetimeDB): All subscribed clients receive updated Player table
6. **Update Display** (Unity): OnUpdate callback moves GameObject to new position

### 4.4 Core Systems Overview

|System|Client (Unity)|Server (SpacetimeDB/Rust)|
|---|---|---|
|**Map System**|Tile painting, zone editor, prefab placement|Zone data storage, portal connections|
|**Entity System**|GameObject rendering, animation|Entity state, position, health|
|**Event System**|Visual scripting UI|Trigger execution, condition checking|
|**Combat System**|VFX, animations, UI feedback|Damage calculation, validation, death handling|
|**AI System**|NavMesh visualization|Pathfinding, behavior tree execution|
|**Inventory**|UI, drag-and-drop|Item ownership, validation|
|**Quest System**|Objective tracking UI|Quest state, progression, rewards|

---

## 5. Map and World System

### 5.1 Data Structure

The world is organized using a **hybrid storage model**:

- **Client-Side (Unity)**: Asset references, prefabs, visual data (ScriptableObjects)
- **Server-Side (SpacetimeDB)**: Game state, entity positions, zone metadata (Rust tables)

#### Server Tables (Rust + SpacetimeDB)

```rust
use spacetimedb::{table, SpacetimeType, Identity};

// Zone stores terrain dimensions and water level.
// terrain_width/height are in world units; chunks are 32×32 each.
#[table(accessor = zone, public)]
pub struct Zone {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub name: String,
    pub terrain_width: u32,
    pub terrain_height: u32,
    pub water_level: f32,
}

// Per-chunk heightmap and splatmap. create_zone initialises one row per chunk.
#[table(accessor = terrain_chunk, public)]
pub struct TerrainChunk {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    #[index(btree)]
    pub zone_id: u64,
    pub chunk_x: u32,
    pub chunk_z: u32,
    pub height_data: Vec<u8>,  // 32×32 f32 LE = 4096 bytes
    pub splat_data: Vec<u8>,   // 32×32 × 4 u8 = 4096 bytes
}

// entity_type is a plain String ("NPC", "Enemy", "StaticProp", etc.)
// elevation is world-space Y (vertical height above terrain).
#[table(accessor = entity_instance, public)]
pub struct EntityInstance {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub zone_id: u64,
    pub prefab_name: String,
    pub position_x: f32,
    pub position_y: f32,
    pub elevation: f32,
    pub entity_type: String,
}

// Portal table — planned for Phase 4
// #[table(accessor = portal, public)]
// pub struct Portal { ... }
```

#### Client Data (Unity ScriptableObjects)

```csharp
// Visual and asset data that doesn't need server validation.
// The terrain mesh itself is built at runtime from TerrainChunk server data.
[CreateAssetMenu(menuName = "ZoneForge/Zone Visual Data")]
public class ZoneVisualData : ScriptableObject
{
    public string zoneName;
    public Material terrainMaterial;   // splatmap shader material
    public Material waterMaterial;     // flat water shader material
    public Material skyboxMaterial;
    public Color ambientColor;
}
```

### 5.2 Map Creation Workflow

1. Designer opens the standalone **zoneforge-editor** application (not the Unity Editor)
2. In the **Zone Manager panel** (top-left), fill in the zone name, terrain width/height, and water level
3. Click **Create Zone** — this calls the `create_zone` reducer, which:
    - Inserts a `Zone` row in SpacetimeDB
    - Initialises flat `TerrainChunk` rows for every 32×32 chunk in the grid
4. The editor's `TerrainRenderer` receives `OnInsert` callbacks and builds the initial mesh
5. Designer uses the **Brush Panel** (top-right) to paint height and texture layers — each brush stroke calls `update_terrain_chunk`
6. Changes are immediately visible to all connected game clients via SpacetimeDB's automatic sync

### 5.3 Multiplayer Zone Loading

When a player enters a zone:

```rust
// Planned for Phase 4 — zone portals and transfers
#[reducer]
pub fn enter_zone(ctx: &ReducerContext, zone_id: u64, spawn_x: f32, spawn_y: f32) -> Result<(), String> {
    // ctx.sender() is a method in SpacetimeDB 2.x
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or("Player not found")?;

    ctx.db.player().id().update(Player {
        zone_id,
        position_x: spawn_x,
        position_y: spawn_y,
        ..player
    });

    // Automatically syncs to all players subscribed to this zone!
    Ok(())
}
```

Unity client automatically receives zone updates via the SpacetimeDB 2.x SDK:

```csharp
// In OnSubscriptionApplied callback (SpacetimeDBManager.cs):
conn.Db.EntityInstance.OnInsert += OnEntitySpawned;
conn.Db.Player.OnUpdate += OnPlayerMoved;

void OnEntitySpawned(EventContext ctx, EntityInstance entity)
{
    if (entity.ZoneId == currentZoneId)
    {
        // Instantiate Unity prefab
        var prefab = Resources.Load<GameObject>(entity.PrefabName);
        var instance = Instantiate(prefab,
            new Vector3(entity.PositionX, entity.Elevation, entity.PositionY),
            Quaternion.identity);
    }
}
```

### 5.3 Terrain Chunk System

Terrain is **procedurally generated from height and splat data** stored per chunk in the `TerrainChunk` table. There are no individual tile GameObjects for ground terrain.

**Chunk layout:** A zone of width W × height H is divided into chunks of 32×32 world units. Each chunk has one `TerrainChunk` row.

| Data | Storage | Format |
|---|---|---|
| `height_data` | `Vec<u8>` | 1024 × f32 LE = 4096 bytes — one height value per sample point |
| `splat_data` | `Vec<u8>` | 1024 × 4 bytes = 4096 bytes — RGBA blend weights for 4 texture layers |

**Texture layers blended by splatmap:**

| Channel | Texture | Example Use |
|---|---|---|
| R (255,0,0,0) | Layer 0 — Grass | Default flat terrain |
| G (0,255,0,0) | Layer 1 — Dirt | Paths, bare earth |
| B (0,0,255,0) | Layer 2 — Stone | Rocky outcrops, roads |
| A (0,0,0,255) | Layer 3 — Ravine | Deep crevices, dark areas |

**Rendering flow:**
1. `TerrainRenderer` subscribes to `TerrainChunk.OnInsert` / `OnUpdate`
2. Decodes `height_data` bytes → Mesh vertex Y positions
3. Decodes `splat_data` bytes → per-vertex UV2 (used by the splatmap shader)
4. Uploads Mesh to GPU; updates `MeshCollider.sharedMesh`

**Water:** `WaterRenderer` renders a flat quad at `zone.water_level`. Any terrain vertex below that elevation appears submerged.

### 5.4 Zone Stitching and Portals

#### Portal System

```csharp
public class PortalData : ScriptableObject
{
    public string portalID;
    public ZoneData sourceZone;
    public Vector2Int sourcePosition;        // grid coordinates
    public ZoneData destinationZone;
    public string destinationSpawnPoint;     // spawn point ID
    public TransitionType transitionType;    // Instant, FadeToBlack, Cutscene
    public List<Condition> requiredConditions;
}
```

#### World Graph View

Custom editor window (**Tools > RPG Editor > World Graph**) displays:

- Node-based visualization of zones (boxes) and portals (connecting lines)
- Click-and-drag to create new portal connections
- Double-click zone node to open in Scene view
- Auto-layout algorithms to organize complex world graphs

---

## 6. Entity System

### 6.1 Entity Hierarchy

All placeable objects inherit from BaseEntity MonoBehaviour:

| Entity Type       | Properties                                               |
| ----------------- | -------------------------------------------------------- |
| **StaticProp**    | Non-interactive decorations (barrels, trees, rocks)      |
| **Interactive**   | Chests, doors, levers with OnInteract event hook         |
| **NPC**           | Dialogue tree, idle behavior, quest data, shop inventory |
| **Enemy**         | Health, AI behavior tree, attack abilities, loot table   |
| **Player**        | Spawn point marker, starting equipment, class template   |
| **TriggerVolume** | Invisible collision zone for event detection             |

### 6.2 Entity Placement System

#### EntityPlacement Data

```csharp
[Serializable]
public class EntityPlacement
{
    public string entityID;                              // unique instance ID
    public GameObject prefab;                            // reference to entity prefab
    public Vector3 position;
    public Quaternion rotation;
    public Vector3 scale;
    public Dictionary<string, object> overrideData;      // per-instance property overrides
}
```

#### Placement Workflow

1. Designer opens Entity Palette window (**Tools > RPG Editor > Entity Palette**)
2. Browses categorized entity library (NPCs, Enemies, Props, Interactives)
3. Click entity thumbnail, then click in Scene view to place
4. Multi-select and drag to duplicate/array placement
5. Gizmos show entity bounds, interaction radius, patrol paths in Scene view
6. Inspector shows entity-specific properties (dialogue, loot tables, AI behavior)

### 6.3 NPC System

#### NPCData ScriptableObject

```csharp
public class NPCData : ScriptableObject
{
    public string displayName;
    public GameObject modelPrefab;
    public AnimatorController animationController;
    public DialogueGraph dialogueTree;
    public IdleBehavior idleBehavior;        // StandStill, Patrol, SitDown, RandomWalk
    public List<Vector3> patrolWaypoints;
    public List<ItemData> shopInventory;     // if merchant
    public QuestData questGiver;             // if quest NPC
}
```

#### Dialogue System

Node-based dialogue editor (similar to Unity's Shader Graph):

- **DialogueNode**: Entry point, NPC speech, player choices, conditional branches
- **Conditions**: Check quest status, player level, inventory items, world state flags
- **Actions**: Start quest, give item, set world flag, trigger event, open shop
- **AI Integration**: Import dialogue trees from JSON, let AI generate conversation flow based on character background

### 6.4 Enemy AI System

#### EnemyData ScriptableObject

```csharp
public class EnemyData : ScriptableObject
{
    public string enemyName;
    public int health;
    public float movementSpeed;
    public float aggroRadius;                // detection range
    public float fleeThreshold;              // health % to start fleeing
    public BehaviorTreeAsset behaviorTree;
    public List<AbilityData> attackAbilities;
    public LootTableData lootTable;
}
```

#### Behavior Tree Integration

Use Unity-compatible behavior tree plugin (e.g., Behavior Designer or custom):

- **States**: Idle, Patrol, Chase, Attack, Flee, Dead
- **Transitions**: Based on player distance, health thresholds, ability cooldowns
- **Custom nodes**: CastSpell, TeleportToPosition, SummonMinion

---

## 7. Event and Trigger System

### 7.1 System Overview

The Event System is the core gameplay logic layer, enabling designers to create interactive experiences without coding. Implemented using **Unity Visual Scripting** (formerly Bolt).

### 7.2 Trigger Data Structure

```csharp
public class TriggerData : ScriptableObject
{
    public string triggerID;
    public TriggerType triggerType;          // OnEnter, OnExit, OnInteract, OnKill, OnTimer, OnItemPickup
    public Collider triggerZone;             // BoxCollider or SphereCollider (invisible)
    public List<ConditionNode> conditions;
    public List<ActionNode> actions;
    public RepeatMode repeatMode;            // Once, Repeatable, OncePerPlayer
}
```

### 7.3 Event Types

|Event Type|Description|Use Cases|
|---|---|---|
|**OnEnter**|Player enters trigger volume|Zone transitions, ambushes, cutscenes|
|**OnExit**|Player leaves trigger volume|Lock doors, spawn reinforcements|
|**OnInteract**|Player presses interact key near object|Open chests, talk to NPCs, pull levers|
|**OnKill**|Specific enemy dies|Quest objectives, spawn boss|
|**OnTimer**|Time-based trigger|Day/night cycle events, timed challenges|
|**OnItemPickup**|Player collects item|Quest progression, unlock areas|

### 7.4 Condition System

Pre-built condition modules (drag-and-drop in Visual Scripting):

- **HasItem(itemID)**: Check player inventory
- **QuestStatus(questID, status)**: Check quest state (NotStarted, Active, Completed)
- **PlayerLevel(minLevel, maxLevel)**: Level range check
- **TimeOfDay(startHour, endHour)**: In-game time check
- **WorldFlag(flagName, value)**: Global state variable check
- **RandomChance(percentage)**: Probability check (0-100)

### 7.5 Action System

Pre-built action modules:

- **SpawnEntity(prefab, position, rotation)**
- **DestroyEntity(entityID)**
- **PlayCutscene(timelineAsset)**
- **PlaySound(audioClip, volume)**
- **GiveItem(itemID, quantity)**
- **RemoveItem(itemID, quantity)**
- **SetQuestMarker(position, iconType)**
- **ChangeWeather(weatherType, transitionTime)**
- **SetWorldFlag(flagName, value)**
- **ShowUIMessage(text, duration)**
- **TeleportPlayer(zoneID, spawnPointID)**

### 7.6 Example Trigger Configuration

**Use Case: Goblin Ambush on Cave Entry**

```
Event: OnEnter (trigger zone at cave entrance)

Conditions:
  - QuestStatus('lost_sword', Active)
  - WorldFlag('cave_ambush_triggered', false)

Actions:
  - SpawnEntity('goblin_warrior_prefab', waypoint_12, default)
  - SpawnEntity('goblin_warrior_prefab', waypoint_13, default)
  - SpawnEntity('goblin_warrior_prefab', waypoint_14, default)
  - PlaySound('battle_horn.wav', 0.8)
  - SetQuestMarker(cave_interior_position, 'investigate')
  - SetWorldFlag('cave_ambush_triggered', true)
  - ShowUIMessage('You've been ambushed!', 3.0)
```

---

## 8. Combat and Spell System

### 8.1 Combat System Architecture

Real-time combat with cooldown-based abilities, inspired by Diablo and Path of Exile.

#### Core Components

- **CombatManager** (Singleton): Handles damage calculation, status effects, death events
- **DamageSystem**: Raycast/overlap checks for hitboxes, applies damage to Health component
- **StatusEffectController**: Manages buffs/debuffs (burn, freeze, poison, slow, stun)

### 8.2 Ability Data Structure

```csharp
public class AbilityData : ScriptableObject
{
    public string abilityName;
    public Sprite iconSprite;                    // for UI hotbar
    public AbilityType abilityType;              // Melee, Ranged, AOE, Buff, Debuff, Heal
    public int damageAmount;                     // base damage before modifiers
    public float castTime;                       // seconds, 0 for instant-cast
    public float cooldown;                       // seconds between uses
    public int manaCost;
    public float range;                          // max distance, 0 for self-cast
    public float aoeRadius;                      // area of effect, 0 if single-target
    public GameObject projectilePrefab;          // visual effect for ranged
    public GameObject impactVFX;                 // particle effect on hit
    public List<StatusEffectData> statusEffects;
    public AudioClip soundEffect;
}
```

### 8.3 Spell Editor Tool

Custom Editor Window (**Tools > RPG Editor > Spell Designer**):

- **Database Browser**: Searchable list of all abilities
- **Visual Preview**: Shows projectile trajectory, AOE circle, VFX in real-time
- **Stat Balancing Tools**: DPS calculator, mana efficiency rating
- **Templates**: Fireball, Heal, Summon, Teleport (quick-start presets)

### 8.4 Example Abilities

| Ability             | Type   | Damage | Cast | Cooldown | Mana | Special                |
| ------------------- | ------ | ------ | ---- | -------- | ---- | ---------------------- |
| **Fireball**        | Ranged | 50     | 1.5s | 3s       | 20   | Burn DoT (5 dmg/s, 3s) |
| **Ice Nova**        | AOE    | 80     | 2s   | 8s       | 40   | 5m radius, Freeze 2s   |
| **Heal**            | Buff   | 100    | 2s   | 10s      | 30   | Self or ally target    |
| **Melee Slash**     | Melee  | 35     | 0s   | 1s       | 0    | Basic attack           |
| **Lightning Chain** | Ranged | 60     | 1s   | 5s       | 25   | Jumps to 3 enemies     |

---

## 8A. Object Pooling System

### 8A.1 Overview

Object pooling is a critical performance pattern for ZoneForge's combat and entity systems. Rather than calling `Instantiate()` and `Destroy()` on GameObjects at runtime — which generates garbage, triggers physics recalculations, and stalls frames — pooling pre-allocates a fixed set of objects at scene load and recycles them throughout the session.

**Why pooling is non-negotiable at scale:**

Without pooling, a fireball cast every 3 seconds by 20 concurrent players generates 400+ `Instantiate`/`Destroy` calls per minute. Each `Destroy()` on a particle system defers cleanup to the next GC collection, causing frame spikes. Enemy spawn waves create allocation bursts that can drop frame rate below the 60 FPS target.

| System | Without Pooling | With Pooling |
| --- | --- | --- |
| **Projectile (Fireball)** | Instantiate on cast, Destroy on hit — 2 allocs per shot | Retrieve from pool on cast, return on hit — 0 allocs |
| **Impact VFX** | Instantiate particle system, auto-destroy after lifetime | Retrieve, play, auto-return after lifetime completes |
| **Enemy units** | Instantiate on spawn wave, Destroy on death | Retrieve pre-warmed enemy, reset state, return on death |
| **Damage numbers** | Instantiate text GameObject per hit | Retrieve from pool, set value, animate, return |

### 8A.2 Systems That Use Pools

| Pool Key               | Prefab              | Default Capacity | Max Size | Return Trigger                    |
| ---------------------- | ------------------- | ---------------- | -------- | --------------------------------- |
| `projectile_fireball`  | FireballProjectile  | 20               | 60       | On hit or timeout (5s)            |
| `projectile_arrow`     | ArrowProjectile     | 20               | 60       | On hit or timeout (4s)            |
| `projectile_lightning` | LightningProjectile | 10               | 30       | On hit or timeout (3s)            |
| `vfx_impact_fire`      | ImpactFireVFX       | 15               | 50       | Particle system lifetime          |
| `vfx_impact_generic`   | ImpactGenericVFX    | 20               | 60       | Particle system lifetime          |
| `ui_damage_number`     | DamageNumberUI      | 30               | 100      | After float animation (1.2s)      |
| `enemy_goblin`         | GoblinEnemy         | 10               | 40       | On death + loot spawn             |
| `enemy_skeleton`       | SkeletonEnemy       | 10               | 40       | On death + loot spawn             |
| `loot_pickup`          | LootPickupObject    | 20               | 80       | On player pickup or timeout (60s) |

### 8A.3 Unity's Built-In `ObjectPool<T>`

Unity 2021+ includes a generic `ObjectPool<T>` class in `UnityEngine.Pool`. ZoneForge uses this as the foundation rather than a custom implementation.

```csharp
// Unity's ObjectPool<T> constructor signature:
new ObjectPool<T>(
    createFunc:      () => T,         // how to create a new instance
    actionOnGet:     (T obj) => void, // called when retrieved from pool
    actionOnRelease: (T obj) => void, // called when returned to pool
    actionOnDestroy: (T obj) => void, // called if pool exceeds max size
    collectionCheck: bool,            // throws if double-release detected (dev only)
    defaultCapacity: int,             // initial pre-allocated count
    maxSize:         int              // pool will not grow beyond this
);
```

### 8A.4 ZoneForgePoolManager

A centralized `PoolManager` MonoBehaviour owns all pools and provides a clean API for game systems to request and return objects. Attach to a persistent GameObject in the Bootstrap scene.

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Pool;

/// <summary>
/// Central pool registry for ZoneForge.
/// Attach to a persistent GameObject in the Bootstrap scene.
/// </summary>
public class ZoneForgePoolManager : MonoBehaviour
{
    public static ZoneForgePoolManager Instance { get; private set; }

    [System.Serializable]
    public class PoolConfig
    {
        public string key;
        public GameObject prefab;
        public int defaultCapacity = 10;
        public int maxSize = 50;
    }

    [SerializeField] private List<PoolConfig> poolConfigs;

    private readonly Dictionary<string, ObjectPool<GameObject>> _pools = new();

    void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);
        InitializePools();
    }

    private void InitializePools()
    {
        foreach (var config in poolConfigs)
        {
            var prefab = config.prefab;
            _pools[config.key] = new ObjectPool<GameObject>(
                createFunc:      () => Instantiate(prefab),
                actionOnGet:     obj => obj.SetActive(true),
                actionOnRelease: obj => obj.SetActive(false),
                actionOnDestroy: obj => Destroy(obj),
                collectionCheck: true,   // disable in Release builds
                defaultCapacity: config.defaultCapacity,
                maxSize:         config.maxSize
            );
        }
    }

    /// <summary>Retrieve a pooled object. Always pair with Return().</summary>
    public GameObject Get(string key)
    {
        if (!_pools.TryGetValue(key, out var pool))
        {
            Debug.LogError($"[Pool] Key not registered: {key}");
            return null;
        }
        return pool.Get();
    }

    /// <summary>Return a pooled object. Must match the key it was retrieved with.</summary>
    public void Return(string key, GameObject obj)
    {
        if (_pools.TryGetValue(key, out var pool))
            pool.Release(obj);
        else
            Destroy(obj); // safe fallback
    }

    public int CountActive(string key)   => _pools.TryGetValue(key, out var p) ? p.CountActive   : 0;
    public int CountInactive(string key) => _pools.TryGetValue(key, out var p) ? p.CountInactive : 0;
}
```
### 8A.5 Projectile Pool Implementation

Projectiles are the highest-frequency pooled objects in combat. The following component makes any projectile self-returning — it automatically gives itself back to the pool on impact or timeout, with no external cleanup required.

```csharp
using UnityEngine;

/// <summary>
/// Attach to all projectile prefabs (Fireball, Arrow, LightningBolt, etc.)
/// Self-returns to pool on collision or timeout.
/// </summary>
[RequireComponent(typeof(Rigidbody))]
public class PooledProjectile : MonoBehaviour
{
    [Header("Pool Settings")]
    [Tooltip("Must match the key registered in ZoneForgePoolManager")]
    public string poolKey;
    public float maxLifetime = 5f;

    private float _spawnTime;
    private bool _hasReturned;

    void OnEnable()
    {
        _spawnTime = Time.time;
        _hasReturned = false;
    }

    void Update()
    {
        if (!_hasReturned && Time.time - _spawnTime >= maxLifetime)
            ReturnToPool();
    }

    void OnCollisionEnter(Collision collision)
    {
        if (_hasReturned) return;

        // Spawn impact VFX from its own pool before returning
        var vfx = ZoneForgePoolManager.Instance.Get("vfx_impact_fire");
        if (vfx != null)
        {
            vfx.transform.position = collision.contacts[0].point;
            // VFX component handles its own return after particle lifetime
        }

        ReturnToPool();
    }

    private void ReturnToPool()
    {
        _hasReturned = true;
        var rb = GetComponent<Rigidbody>();
        rb.linearVelocity = Vector3.zero;
        rb.angularVelocity = Vector3.zero;
        ZoneForgePoolManager.Instance.Return(poolKey, gameObject);
    }
}
```

### 8A.6 VFX Auto-Return Component

Particle system VFX needs a lightweight component that returns itself to the pool after its effect finishes playing — without requiring external management.

```csharp
using System.Collections;
using UnityEngine;

/// <summary>
/// Attach to all pooled VFX prefabs.
/// Automatically returns to pool when the particle system finishes.
/// </summary>
[RequireComponent(typeof(ParticleSystem))]
public class PooledVFX : MonoBehaviour
{
    public string poolKey;

    private ParticleSystem _ps;

    void Awake() => _ps = GetComponent<ParticleSystem>();

    void OnEnable() => StartCoroutine(ReturnWhenFinished());

    private IEnumerator ReturnWhenFinished()
    {
        // Wait for the particle system's full duration + start lifetime
        yield return new WaitForSeconds(_ps.main.duration + _ps.main.startLifetime.constantMax);

        // Stop emission and wait for all particles to die
        _ps.Stop(withChildren: true, stopBehavior: ParticleSystemStopBehavior.StopEmitting);
        yield return new WaitUntil(() => !_ps.IsAlive(true));

        ZoneForgePoolManager.Instance.Return(poolKey, gameObject);
    }
}
```

### 8A.7 Enemy Pool Integration

Enemies require additional reset logic when retrieved from the pool — health, AI state, and animation must be restored to their initial values.

```csharp
using UnityEngine;

/// <summary>
/// Attach to all pooled enemy prefabs.
/// Handles state reset on pool retrieval and self-return on death.
/// </summary>
public class PooledEnemy : MonoBehaviour
{
    public string poolKey;

    private EnemyData _data;
    private int _currentHealth;
    private EnemyAIController _ai;
    private Animator _animator;

    void Awake()
    {
        _data     = GetComponent<EnemyData>();
        _ai       = GetComponent<EnemyAIController>();
        _animator = GetComponent<Animator>();
    }

    // Called automatically by ObjectPool when retrieved
    void OnEnable()
    {
        _currentHealth = _data.health;       // restore to full health
        _ai.ResetBehavior();                  // return AI to Idle state
        _animator.Rebind();                   // reset all animation state
        _animator.Update(0f);
    }

    public void TakeDamage(int amount)
    {
        _currentHealth -= amount;
        if (_currentHealth <= 0)
            Die();
    }

    private void Die()
    {
        // Spawn loot, play death VFX, then return
        SpawnLoot();
        var deathVFX = ZoneForgePoolManager.Instance.Get("vfx_impact_generic");
        if (deathVFX != null)
            deathVFX.transform.position = transform.position;

        ZoneForgePoolManager.Instance.Return(poolKey, gameObject);
    }

    private void SpawnLoot()
    {
        // Loot pickup objects also use the pool
        var loot = ZoneForgePoolManager.Instance.Get("loot_pickup");
        if (loot != null)
            loot.transform.position = transform.position;
    }
}
```

### 8A.8 Calling the Pool from Combat System

The combat system (Section 8) calls the pool manager instead of `Instantiate` when spawning projectiles:

```csharp
// In CombatManager.cs — called when player uses a ranged ability
public void FireProjectile(AbilityData ability, Vector3 origin, Vector3 direction)
{
    // Map ability type to pool key
    string poolKey = ability.abilityType switch
    {
        AbilityType.Fireball  => "projectile_fireball",
        AbilityType.Arrow     => "projectile_arrow",
        AbilityType.Lightning => "projectile_lightning",
        _                     => "projectile_fireball"
    };

    var projectile = ZoneForgePoolManager.Instance.Get(poolKey);
    if (projectile == null) return;

    projectile.transform.SetPositionAndRotation(origin, Quaternion.LookRotation(direction));
    projectile.GetComponent<Rigidbody>().linearVelocity = direction * ability.projectileSpeed;
}
```

### 8A.9 Roadmap Integration

Object pooling should be introduced in **Sprint 8** (Month 4), before the combat demo milestone, to ensure all VFX and projectile systems are built pool-aware from the start rather than retrofitted later.

| Sprint    | Task                                                                    |
| --------- | ----------------------------------------------------------------------- |
| Sprint 8  | Implement `ZoneForgePoolManager`, configure projectile and VFX pools    |
| Sprint 8  | Add `PooledProjectile` and `PooledVFX` components to all combat prefabs |
| Sprint 12 | Add `PooledEnemy` to all enemy prefabs before load testing              |
|Sprint 18|Profile pool sizes under 100-player load, tune `defaultCapacity` and `maxSize` values|

---

## 9. Asset Pipeline

### 9.1 Asset Types and Organization

**Project Structure (Assets/RPG_Game/):**

```
Assets/
├── Art/
│   ├── Models/              # FBX, OBJ files
│   ├── Textures/            # PNG, TGA (diffuse, normal, emission maps)
│   ├── Materials/           # Unity material assets
│   └── Animations/          # Animation clips, Animator Controllers
├── Audio/
│   ├── Music/               # WAV, OGG background tracks
│   ├── SFX/                 # Combat, UI, ambient sounds
│   └── Voice/               # NPC dialogue recordings
├── Prefabs/
│   ├── Entities/            # NPCs, enemies, props
│   ├── VFX/                 # Particle systems, shaders
│   └── UI/                  # HUD elements, menus
├── Data/                    # ScriptableObjects
│   ├── Zones/
│   ├── Abilities/
│   ├── Items/
│   └── Quests/
└── Scenes/
    ├── Zones/               # One scene per zone
    └── UI/                  # MainMenu, LoadingScreen
```

### 9.2 3D Model Import Workflow

1. Export from Blender as FBX (Y-up, -Z forward, 1 Blender Unit = 1 meter)
2. Import into Unity (drag FBX into Assets/Art/Models/)
3. Configure Import Settings:
    - Scale Factor: 1 (if modeled at 1m = 1 unit)
    - Generate Colliders: Yes (box collider for props)
    - Animation: Import if embedded in FBX
4. Auto-generate prefab with default components (MeshRenderer, Collider)
5. Tag with entity category (Enemy, NPC, Prop) for Entity Palette filtering

### 9.3 Texture and Material Pipeline

- **Textures**: 1024x1024 or 2048x2048 PNG for diffuse maps
- **Normal Maps**: Generated from diffuse using Unity's Normal Map filter or external tools
- **Materials**: Use URP/Lit shader for PBR workflow (Albedo, Normal, Metallic, Smoothness)
- **Atlasing**: Combine small textures into texture atlases for performance

### 9.4 Animation Import

- **Humanoid Rig**: Use Unity's Humanoid Avatar system for character animations (allows retargeting)
- **Required Animation Clips**: Idle, Walk, Run, Attack, CastSpell, Hit, Death
- **Animator Controller**: Set up state machine (Idle → Walk → Attack transitions)
- **Animation Events**: Mark hit frames for damage application, sound cues

### 9.5 AI-Assisted Asset Creation

#### Recommended Workflows

- **Texture Generation**: Midjourney/Stable Diffusion → clean up in Photoshop → import
- **Dialogue Writing**: AI generates conversation tree → designer edits for tone/consistency
- **Quest Design**: Input quest outline → AI generates objectives, dialogue, rewards → designer refines
- **Procedural Variation**: One base model → AI texture variants (weathered, damaged, painted)

---

## 9A. Procedural Texturing Pipeline

### 9A.1 Philosophy: Math Over Painting

ZoneForge adopts a procedural-first asset creation philosophy. Traditional hand-painted textures create a bottleneck: every surface variant (clean stone, cracked stone, mossy stone) requires separate artwork. A procedural approach instead defines surfaces as mathematical recipes — noise patterns, gradient functions, and layer blends — that generate controlled variations from a single source graph.

**Benefits for ZoneForge:**

- Tile layers (Ground, Decoration, Collision) require seamless, tileable textures. Procedural generation guarantees seamlessness mathematically.
- Zone aesthetic variants (Sunny Village, Spooky Forest, Dungeon) can share the same base material graph with different parameter sets rather than entirely separate texture files.
- Normal maps for isometric lighting are generated from the same graph as the diffuse, ensuring they always match.
- Asset library scales without linear effort — one designer can produce 50+ surface variants from 5 base graphs.

### 9A.2 Required Software

| Tool | Purpose | Cost | URL |
| --- | --- | --- | --- |
| **Material Maker** | Procedural texture graph editor — creates tileable textures, normal maps, roughness maps from math nodes | Free / Open Source | materialmaker.org |
| **ArmorPaint** | 3D texture painting on models with smart materials (edge wear, dirt, AO) | Free (self-compile) or ~$19 | armorpaint.org |
| **Blender 3.6+** | 3D modeling, Geometry Nodes, FBX export (already in dev stack) | Free | blender.org |

### 9A.3 Material Maker Workflow

#### Installation and Project Setup

```
# Download Material Maker from https://www.materialmaker.org/
# Extract and run — no installer required

# Recommended project folder structure:
Assets/Art/MaterialMaker/
├── Graphs/       # .ptex source graph files (version controlled with Git)
├── Exports/      # Exported PNG sets (add to .gitignore — regenerate from graphs)
└── Parameters/   # Saved parameter presets per zone theme (.json)
```

#### Building a Ground Tile Graph (Step-by-Step)

The following describes building a reusable stone ground tile that can be parameterized for different zone themes:

1. Open Material Maker → New Graph → name it `stone_ground_base.ptex`
2. Add a **Noise node** (Simplex): Scale 8.0 — controls large rock face variation
3. Add a second **Noise node** (Simplex): Scale 32.0 — controls fine surface grain
4. **Blend** both using a Mix node at weight 0.4 — creates layered natural surface
5. Feed into a **Slope Blur** node to simulate directional weathering
6. Connect to **Color Ramp**: 0.0 → dark grey `#3A3A3A`, 0.5 → mid stone `#8A8A7A`, 1.0 → highlight `#C8C4B0`
7. Add **Warp** node (strength 0.15) to break up tiling regularity
8. Connect to **Output** node with channels: Albedo, Normal (auto-generated from height), Roughness

#### Exporting for Unity URP

```
// Material Maker Export Settings:
// Output Resolution: 1024x1024 (standard) or 2048x2048 (hero assets)
// Format: PNG (Albedo, Roughness, AO) | EXR (Normal maps, for precision)
//
// Naming convention: {material_name}_{channel}.png
//
// Exports from stone_ground_base:
//   stone_ground_base_albedo.png
//   stone_ground_base_normal.png
//   stone_ground_base_roughness.png
//   stone_ground_base_ao.png
```

#### Unity Import Configuration

```
// Albedo texture:
// Texture Type: Default | sRGB: ON | Compression: BC7 (high quality)

// Normal map:
// Texture Type: Normal map | Create from Grayscale: OFF (exported as normal)
// Compression: BC5 (preserves XY channels accurately)

// Roughness / AO (pack into single texture for performance):
// Pack in GIMP: R=Metallic, G=AO, B=unused, A=Roughness
// Texture Type: Default | sRGB: OFF | Compression: BC7

// Apply to URP/Lit material:
// Surface Inputs → Albedo:      stone_ground_base_albedo
//               → Normal Map:   stone_ground_base_normal | Normal Scale: 0.8
//               → Metallic Map: packed_mask (R channel)
//               → Smoothness:   packed_mask (A channel)
```

### 9A.4 Zone Theme Parameterization

Material Maker graphs support exported parameters, allowing the same graph to produce different zone aesthetics by adjusting values rather than rebuilding artwork. Save each as a `.json` preset in `Assets/Art/MaterialMaker/Parameters/` for one-click re-export.

| Zone Theme | Primary Color | Noise Scale | Roughness | Warp Strength |
| --- | --- | --- | --- | --- |
| **Sunny Village** | `#9C8E6A` (warm stone) | 8.0 / 28.0 | 0.65 | 0.12 |
| **Spooky Forest** | `#4A4A3E` (dark moss) | 6.0 / 20.0 | 0.80 | 0.22 |
| **Dungeon** | `#3A3A3A` (wet rock) | 10.0 / 36.0 | 0.90 | 0.08 |
| **Sunset Ruins** | `#8C6A4A` (sandstone) | 7.0 / 24.0 | 0.70 | 0.18 |

### 9A.5 Roadmap Integration

| Sprint | Task |
| --- | --- |
| Sprint 1 | Install Material Maker, set up folder structure, create first test graph |
| Sprint 2 | Produce ground tile sets for Village zone (grass, dirt path, stone cobble) |
| Sprint 8 | Produce dungeon tile sets (wet stone, carved rock, mossy floor) |
| Sprint 15 | Produce interior tile sets (wooden floor, plaster wall, stone fireplace) |

---

## 9B. Smart Materials & ArmorPaint Workflow

### 9B.1 What Smart Materials Automate

ArmorPaint brings smart material automation to ZoneForge's 3D models. Smart materials use the model's own geometry data (curvature, ambient occlusion, edge exposure) to automatically apply realistic wear patterns, saving hours of manual painting per asset.

- **Edge Wear** — highlights on geometry edges where real-world wear occurs
- **Dirt Accumulation** — automatically darkens recessed areas (crevices, corners) using curvature maps
- **Ambient Occlusion Masking** — darkens areas that receive less indirect light, without manual selection
- **Grunge Overlay** — randomised surface variation using pre-built grunge masks

### 9B.2 ArmorPaint Installation

```bash
# Option A: Pre-built binaries (~$19 one-time)
# Download from https://armorpaint.org → Run ArmorPaint executable

# Option B: Build from source (free)
git clone --recursive https://github.com/armory3d/armorpaint
cd armorpaint
# Follow platform build instructions in README (requires Kha framework)
```

### 9B.3 NPC and Enemy Asset Workflow

#### Step 1 — Bake Utility Maps in Blender

Before painting in ArmorPaint, bake maps from the high-poly onto the low-poly model. These maps drive all smart material behaviour.

```bash
# In Blender — Cycles render engine required for baking

# 1. Select low-poly mesh, then Shift-select high-poly
# 2. Open Shader Editor — add Image Texture node, set as active
# 3. Render Properties → Bake panel:

# Bake Type: Normal    → {asset}_normal_bake.png
# Bake Type: AO        → {asset}_ao_bake.png
# Bake Type: Roughness → {asset}_roughness_bake.png

# Settings: Margin 4px | Resolution 1024x1024 | 32 samples (AO)
```

#### Step 2 — Import into ArmorPaint

1. `File → Import` → select your `.fbx` model (same file used for Unity)
2. `File → Import Texture` → import all baked maps from Step 1
3. `Canvas → New` → set resolution 1024x1024, color space: Linear

#### Step 3 — Apply Smart Material Base

1. Open the Materials panel → Smart Materials browser
2. Drag a base material (Stone, Metal, Wood, Leather) onto the model
3. ArmorPaint auto-generates edge highlights and crevice dirt using baked AO and curvature
4. Adjust **Wear Amount** slider (0.0–1.0) to control how battle-worn the asset appears

#### Step 4 — Layer Additional Detail

```
// Recommended layer stack for NPC/Enemy models (top to bottom):
//
// Layer 5: Decals         — clan symbols, scars, insignia (painted manually)
// Layer 4: Edge Wear      — smart material | Blend: Screen   | Opacity: 0.6
// Layer 3: Dirt/Grime     — smart material | Blend: Multiply | Opacity: 0.4
// Layer 2: Color Variation — noise-based tint for visual uniqueness per unit
// Layer 1: Base Material  — albedo color from Material Maker graph
```

#### Step 5 — Export for Unity

```
// ArmorPaint Export Settings:
// File → Export Textures
//
// Format: PNG | Bits: 8 | Color Space: sRGB (Albedo), Linear (all others)
//
// Export channels:
//   albedo.png  → Albedo / BaseColor
//   normal.png  → Normal map (DirectX format — enable Flip Green Channel in Unity if lighting looks inverted)
//   orm.png     → Packed: R=Occlusion, G=Roughness, B=Metallic
```

### 9B.4 Asset Type Guidelines

| Asset Type | Base Material | Edge Wear | Dirt Amount | Notes |
| --- | --- | --- | --- | --- |
| **Stone props** (walls, barrels) | Stone (rough) | Low (0.3) | Medium (0.5) | Add grunge overlay for moss |
| **Metal props** (weapons, armor) | Metal (polished) | High (0.8) | Low (0.2) | Increase wear for dungeon enemies |
| **Wood props** (crates, fences) | Wood (oak) | Medium (0.5) | High (0.7) | Add dark crevice dirt |
| **NPC clothing** | Fabric | Low (0.2) | Low–Medium | Layer unique color variation |
| **Enemy armor** | Metal (worn) | High (0.9) | High (0.8) | Maximise battle-damage look |

### 9B.5 Roadmap Integration

| Sprint | Task |
| --- | --- |
| Sprint 1 | Install ArmorPaint, paint first test NPC model using baked maps |
| Sprint 3 | Apply smart materials to all Entity Palette props (barrels, crates, walls) |
| Sprint 7 | Apply worn/damaged smart materials to all enemy prefabs |
| Sprint 15 | Apply interior props (furniture, candles, bookshelves) |

---

## 9C. Geometry Nodes: Procedural Foliage & Decoration

### 9C.1 Overview

Blender's Geometry Nodes system enables procedural scattering of meshes across surfaces using a node graph. For ZoneForge, this replaces the manual placement of hundreds of individual foliage, rock, and debris meshes in zone decoration layers — producing natural-looking results in minutes rather than hours.

Geometry Nodes is built into Blender 3.6+, which is already listed as required software in Section 3.4. No additional tools or cost required.

### 9C.2 Use Cases in ZoneForge

| Use Case | Zone Layer | Generated Objects |
| --- | --- | --- |
| **Forest floor** | Decoration (Z:1) | Grass tufts, small rocks, mushrooms, fallen leaves |
| **Tree canopy** | Decoration (Z:1) | Leaf clusters scattered on branch surfaces |
| **Dungeon debris** | Decoration (Z:1) | Rubble, dust piles, bone fragments |
| **Village details** | Decoration (Z:1) | Wildflowers, pebbles, mud patches around buildings |
| **Cliff face variation** | Collision (Z:2) | Moss patches, embedded rocks on wall surfaces |

### 9C.3 Setting Up a Foliage Scatter Graph

#### Prerequisites

- A terrain or ground mesh to scatter onto (the "surface mesh")
- Individual foliage mesh assets: `grass_tuft.fbx`, `small_rock.fbx`, `mushroom.fbx`
- Blender 3.6+ with Geometry Nodes workspace tab enabled

#### Building the Node Graph

```
// Node graph layout for forest floor scatter:

[Group Input]
     ↓
[Distribute Points on Faces]
  Density: 2.5 points per m²
  Seed: 42  (change per zone for variation)
     ↓
[Rotate Instances]
  Rotation: Random Value (Euler) — 0–360° on Z axis only
     ↓
[Scale Instances]
  Scale: Random Value — 0.7 to 1.3 (natural size variation)
     ↓
[Instance on Points]
  Instance: [Collection Info → forest_foliage_collection]
  Pick Instance: True  (randomly picks from collection)
     ↓
[Realize Instances]   ← required before export
     ↓
[Group Output]
```

#### Controlling Density with Weight Painting

Use Blender's Weight Paint mode to define where foliage should be dense or sparse — essential for keeping paths, doorways, and water areas clear:

```
// In Blender:
// 1. Select terrain mesh → switch to Weight Paint mode
// 2. Paint vertex weights:
//      1.0 (red)  = high density
//      0.0 (blue) = no foliage
// 3. In Geometry Nodes, connect the vertex group to:
//    [Distribute Points on Faces] → Density Factor socket
//
// Result: foliage only grows where painted red — paths stay clear automatically
```

### 9C.4 Tree Canopy Workflow

Canopy foliage is scattered directly on branch surfaces rather than placed as billboard cards, producing fully 3D trees suitable for isometric close-up views.

```
// Separate Geometry Nodes modifier on the branch mesh:

[Distribute Points on Faces]
  Density: 8.0  (leaves are small, need high density)
  Distribution Method: Random
     ↓
[Align Euler to Vector]
  Vector: [Normal]  (leaves face outward from branch surface)
  Axis: Z
     ↓
[Rotate Instances]
  Rotation: Random offset ±15°  (prevents flat uniform look)
     ↓
[Scale Instances]
  Scale: Random 0.8–1.2
     ↓
[Instance on Points]
  Instance: leaf_mesh  (single low-poly leaf, ~4 triangles)
     ↓
[Realize Instances] → [Group Output]
```

### 9C.5 Exporting Scattered Meshes to Unity

Geometry Nodes results must be baked and exported as static meshes. The node graph itself does not transfer to Unity.

```bash
# In Blender — export workflow:

# 1. Apply the Geometry Nodes modifier:
#    Select mesh → Properties → Modifier → Apply
#    (converts node graph to real geometry)

# 2. Reduce polygon count if needed:
#    Modifier → Decimate → target < 10,000 tris for decoration props

# 3. Export as FBX:
#    File → Export → FBX
#    Settings:
#      Apply Modifiers: ON
#      Scale: 1.0
#      Forward: Y  |  Up: Z
#      Triangulate Faces: ON  (required for Unity import)

# 4. In Unity after import:
#    Generate Lightmap UVs: ON  (for baked lighting)
#    Mesh Compression: Medium
#    Read/Write Enabled: OFF
```

> **Important:** Save the Blender `.blend` file with the Geometry Nodes modifier **before** applying it. The pre-apply file is your edit source — only apply and export when the result is final. This lets you re-generate with different density, scale, or seed values later.

### 9C.6 Performance Guidelines

- Target triangle budget: **< 8,000 tris** per scattered decoration mesh after realization
- Use **LOD Groups** in Unity: LOD0 = full mesh, LOD1 = 50% reduction, LOD2 = billboard sprite
- Mark all procedurally scattered props as **Static** in the Unity Inspector to enable GPU instancing
- For real-time animated foliage (swaying grass), limit to a 20m radius around the player and use Unity's built-in **Terrain Detail system** rather than scattered FBX meshes

### 9C.7 Roadmap Integration

| Sprint | Task |
| --- | --- |
| Sprint 2 | Build first forest floor scatter graph, export grass/rock decoration set |
| Sprint 2 | Build tree canopy graph, produce 3 tree variants for Village zone |
| Sprint 9 | Build dungeon debris scatter graph for Cave zone |
| Sprint 15 | Build interior clutter graph for tavern and building interiors |
| Sprint 18 | Profile LOD distances on decoration meshes during 100-player load test |

---

## 10. Lighting and VFX System

### 10.1 Lighting Architecture

Three-tier lighting system for performance and visual quality:

#### Global Lighting (Per Zone)

- **Directional Light**: Sun/moon, shadows enabled
- **Skybox**: Time-of-day gradient (sunrise, day, sunset, night)
- **Ambient Lighting**: Environment lighting from skybox
- **Fog**: Unity's exponential fog for atmosphere

#### Local Lighting (Dynamic)

- **Point Lights**: Torches, campfires, magic orbs
- **Spot Lights**: Focused beams, spotlights in dungeons
- **Light Cookies**: Texture-based shadows (window patterns, foliage)

#### Baked Lighting (Static)

- **Lightmaps**: Pre-calculated for static geometry (buildings, terrain)
- **Reflection Probes**: Baked reflections for water, metal surfaces
- **Light Probes**: Sample points for dynamic object lighting

### 10.2 Lighting Presets System

```csharp
public class LightingProfile : ScriptableObject
{
    public string profileName;
    public Material skyboxMaterial;
    public Color sunColor;
    public float sunIntensity;
    public Color ambientColor;
    public Color fogColor;
    public float fogDensity;
}
```

#### Example Presets

|Preset|Skybox|Sun Intensity|Fog|Use Case|
|---|---|---|---|---|
|**Sunny Village**|Blue gradient|1.2|None|Outdoor towns|
|**Spooky Forest**|Purple/dark|0.4|Dense green|Horror zones|
|**Dungeon**|Black|0.1|Light gray|Interior caves|
|**Sunset**|Orange/red|0.8|Warm orange|Dramatic scenes|

### 10.3 VFX System

#### Particle System Types

- **Environmental**: Falling leaves, dust motes, fireflies, rain, snow
- **Combat**: Spell projectiles, explosions, blood splatter, elemental trails
- **Interactive**: Chest opening sparkles, quest item glows, portal swirls

#### VFX Placement Tool

Editor window for placing and previewing particle systems:

- **VFX Library Browser**: Categorized particle effect prefabs
- **Live Preview**: Click to place, see loop/spawn in Scene view
- **Parameter Tweaking**: Emission rate, color, speed, lifetime

### 10.4 Post-Processing Stack

URP Post-Processing Volume (per-zone or global):

- **Bloom**: Magical glow, bright highlights
- **Vignette**: Darken edges in dungeons
- **Color Grading**: Warm/cool color shifts per zone mood
- **Depth of Field**: Focus on foreground, blur background (optional)
- **Chromatic Aberration**: Subtle lens distortion for cinematic feel

---

## 11. Editor UI and Tools

### 11.1 Custom Editor Windows

|Window|Purpose|Key Features|
|---|---|---|
|**Zone Editor**|Map creation and tile painting|Grid size config, layer visibility, procedural terrain|
|**Entity Palette**|Browse and place entities|Category filters, thumbnail previews, search|
|**World Graph**|Visualize zone connections|Node-based layout, portal creation, auto-arrange|
|**Trigger Designer**|Visual scripting for events|Drag-and-drop conditions/actions, templates|
|**Spell Designer**|Create and balance abilities|DPS calculator, VFX preview, stat editor|
|**Quest Editor**|Design quest chains|Objective tracking, reward configuration|

### 11.2 Scene View Enhancements

- **Custom Gizmos**: Visual indicators for entity bounds, trigger zones, patrol paths
- **Grid Snapping**: Hold Shift for grid-aligned placement
- **Layer Visibility**: Toggle ground/decoration/collision layers
- **Measurement Tool**: Click-and-drag to measure distances

### 11.3 Inspector Enhancements

- **Custom Property Drawers**: Rich previews for AbilityData, ItemData
- **Quick Actions**: 'Test Ability', 'Preview Dialogue', 'Simulate AI' buttons
- **Reference Finder**: Show all zones/entities using this asset

---

## 12. Development Roadmap

### 12.1 Phase 1: Core Editor + SpacetimeDB Foundation (Months 1-3)

#### Month 1: Foundation

**Unity Client:**

- Unity project setup with URP and core packages
- ScriptableObject architecture for visual/asset data
- Basic map editor: Create zone, paint tiles on grid
- Import test assets (1 character model, 5 prop models, basic textures)

**SpacetimeDB Server:**

- Install Rust toolchain and SpacetimeDB CLI
- Initialize SpacetimeDB module project
- Define core tables: Zone, Player, EntityInstance
- Implement basic reducers: create_player, move_player, enter_zone
- Deploy to local SpacetimeDB server for testing

#### Month 2: Entity System + Real-Time Sync

**Unity Client:**

- Entity Palette window with drag-and-drop placement
- SpacetimeDB C# SDK integration
- Subscribe to Player and EntityInstance tables
- Real-time entity position updates
- Player controller: WASD movement → calls move_player reducer

**SpacetimeDB Server:**

- NPC table with basic properties
- Enemy table with health and position
- Server-side movement validation (speed limits, wall collision)
- Zone boundary enforcement

#### Month 3: Triggers and Multiplayer Playtest

**Unity Client:**

- Visual Scripting UI for trigger design
- Convert visual scripts to Rust reducer calls
- In-editor playtest: Press Play to test zone
- Multi-client testing (2+ players in same zone)

**SpacetimeDB Server:**

- Trigger table and event system
- Implement 5 core trigger reducers
- 10 action reducers (spawn_entity, play_sound, give_item, etc.)
- Condition checking (has_item, quest_status, etc.)

**Milestone: Multiplayer Village Zone**

- 1 village map (64x64) with 5 buildings, 3 NPCs
- Simple quest: Talk to NPC → collect item → return
- **2-4 players can complete quest together in real-time**
- Deploy to SpacetimeDB Cloud for remote testing

### 12.2 Phase 2: Combat + Server Authority (Months 4-6)

#### Month 4: Combat Foundation

**Unity Client:**

- Combat VFX: Hit sparks, spell projectiles
- Ability UI: Hotbar, cooldown indicators
- Health bars and damage numbers

**SpacetimeDB Server:**

- Combat tables: Ability, StatusEffect, CombatEvent
- Damage calculation reducers (server-authoritative)
- Ability system: 1 melee attack, 2 spells (fireball, heal)
- Status effect system (burn, freeze, stun)
- Anti-cheat validation (cooldowns, mana costs)

#### Month 5: Zone Stitching + Cloud Deployment

**Unity Client:**

- Portal system implementation
- World Graph editor window
- Scene loading/unloading optimization

**SpacetimeDB Server:**

- Portal table and zone transfer reducers
- Cross-zone player migration
- Load balancing considerations (future: multiple zone servers)
- **Deploy to SpacetimeDB Cloud production**

#### Month 6: Systems Polish + Inventory

**Unity Client:**

- Inventory system UI (grid-based, drag-and-drop)
- Equipment slots (weapon, armor, accessories)
- UI framework: Health bars, hotbar, minimap

**SpacetimeDB Server:**

- Item and Inventory tables
- Server-side inventory validation
- Equipment stat bonuses
- Trade system between players
- Loot table and drop system

**Milestone: Multiplayer Combat Demo**

- 15-minute gameplay loop: Village → Forest (fight 3 enemies) → Cave (boss fight)
- **Support 10+ concurrent players per server**
- Server-authoritative combat (no client-side cheating possible)

### 12.3 Phase 3: Advanced Features + Scaling (Months 7-9)

#### Month 7: Advanced Triggers + Quests

**Unity Client:**

- Branching dialogue system
- Quest UI: Objectives, tracking, rewards
- Cutscene tools using Unity Timeline

**SpacetimeDB Server:**

- Quest table and progression system
- Quest sharing in parties
- Persistent quest state across sessions
- Condition/Action library expansion (20+ modules)

#### Month 8: Interior Builder + Lighting

**Unity Client:**

- Building interior editor (place furniture, props)
- Door/window placement tools
- Lighting preset system (Spooky, Sunny, Tavern, etc.)
- Baked lightmaps for static geometry

**Server (Minimal Changes):**

- Interior zone definitions
- Door lock/unlock state

#### Month 9: AI Enhancements + Production Readiness

**Unity Client:**

- AI visualization tools in editor

**SpacetimeDB Server:**

- Advanced enemy AI (3 behavior templates: Melee, Ranged, Caster)
- Server-side pathfinding
- AI spawning and despawning optimization
- **Performance tuning for 100+ concurrent players**
- Database backup and restore procedures
- Server monitoring and logging

**Milestone: Production-Ready Vertical Slice**

- 30-minute campaign: 5 zones, 3 quests, 10 unique enemies, 1 boss
- **Support 50-100 concurrent players**
- Cloud deployment with CI/CD pipeline
- Zero-downtime updates via SpacetimeDB module publishing

### 12.4 Phase 4: Multiplayer Features (Months 10-12) - Future Roadmap

#### Planned Features:

**Party System:**

- Player groups with shared quests
- Party chat and voice
- Shared loot distribution

**PvP System:**

- Dueling between consenting players
- PvP zones
- Leaderboards

**Guild System:**

- Player-created organizations
- Guild halls (persistent zones)
- Guild progression and perks

**Economy:**

- Player-to-player trading
- Auction house
- Crafting system

**World Events:**

- Server-wide boss spawns
- Dynamic events affecting all players
- Seasonal content

---

## 13. Technical Specifications

### 13.1 Performance Targets

| Metric                    | Target               | Notes                                  |
| ------------------------- | -------------------- | -------------------------------------- |
| **Editor FPS**            | 60 FPS               | Unity editor with 50+ entities visible |
| **Client Runtime FPS**    | 60 FPS               | 1080p on mid-range hardware            |
| **Server Tick Rate**      | 20-60 TPS            | SpacetimeDB transaction processing     |
| **Zone Load Time**        | < 3 seconds          | Client-side scene loading              |
| **Server Response Time**  | < 50ms               | Reducer execution latency              |
| **Concurrent Players**    | 100+ per zone        | SpacetimeDB in-memory architecture     |
| **Memory Usage (Client)** | < 2GB RAM            | Per zone, typical content              |
| **Memory Usage (Server)** | < 4GB RAM            | Per 100 concurrent players             |
| **Build Size (Client)**   | < 500MB              | Base game without user content         |
| **Module Size (Server)**  | < 10MB               | Compiled Rust WASM module              |
| **Network Bandwidth**     | < 50 KB/s per player | With efficient delta updates           |
| **Max Zone Size**         | 256x256 grid         | With performance optimizations         |

### 13.2 Data Serialization

**Client-Side (Unity):**

- **ScriptableObjects**: Visual data, asset references, editor-only data
- **Binary Asset Bundles**: Optimized asset loading for builds

**Server-Side (SpacetimeDB):**

- **SpacetimeDB Tables**: All persistent game state (automatic ACID transactions)
- **Commit Log**: Disk-based durability and crash recovery
- **Binary Serialization**: Efficient over-the-wire format (automatic)

**Client ↔ Server Communication:**

- **WebSockets**: Real-time bidirectional communication
- **Binary Protocol**: SpacetimeDB's optimized wire format
- **Delta Updates**: Only changed data is transmitted

### 13.3 Version Control Strategy

**Unity Project (Client):**

- Git repository for all C# code and Unity scenes
- Git LFS for binary assets (.fbx, .png, .wav, .prefab)
- Unity Smart Merge for scene/prefab conflict resolution

**SpacetimeDB Module (Server):**

- Separate Git repository for Rust code
- Standard Rust project structure with Cargo.toml
- CI/CD pipeline for automated testing and deployment

**Branch Strategy:**

- `main` - Stable production builds
- `develop` - Active development
- `feature/*` - Experimental features

### 13.4 Deployment Architecture

#### Local Development

```bash
# Terminal 1: Start local SpacetimeDB server
spacetime start

# Terminal 2: Deploy Rust module
cd server/
spacetime publish zoneforge_server --clear-database

# Terminal 3: Run Unity editor
# Connect to localhost:3000
```

#### Cloud Deployment (SpacetimeDB Cloud)

```bash
# Deploy server module to cloud
spacetime publish zoneforge_server \
  --project-domain your-game.spacetimedb.com \
  --yes

# Unity clients connect to: wss://your-game.spacetimedb.com
```

#### Self-Hosted Deployment

```bash
# Docker deployment
docker run -d \
  -p 3000:3000 \
  -v /data/spacetimedb:/data \
  clockworklabs/spacetimedb:latest

# Deploy module to self-hosted instance
spacetime publish zoneforge_server \
  --server http://your-server.com:3000
```

### 13.5 Scalability Considerations

**Current Architecture (Phase 1-3):**

- Single SpacetimeDB instance per game world
- All zones in one database
- 100-500 concurrent players supported

**Future Scaling Options (Phase 4+):**

**Horizontal Scaling:**

- Multiple SpacetimeDB instances (one per major zone or region)
- Client connects to appropriate instance based on zone
- Cross-instance communication for global chat, trading

**Sharding Strategy:**

```
Instance 1: Starting Zones (1-10)
Instance 2: Mid-Game Zones (11-30)
Instance 3: End-Game Zones (31-50)
Instance 4: PvP Zones
```

**Database Partitioning:**

- Separate tables per zone for isolation
- Global tables for player data, guilds, economy

### 13.6 Security Architecture

**Authentication:**

- SpacetimeDB Identity system (built-in auth)
- Optional: Integration with external auth providers (Steam, Discord, etc.)

**Authorization:**

- Server-side validation in every reducer
- Player can only modify their own data
- Admin roles for GM commands

**Anti-Cheat:**

- All game logic on server (client sends inputs only)
- Server validates movement speed, ability cooldowns, resource costs
- Client-side prediction with server reconciliation

**Example Secure Reducer:**

```rust
#[reducer]
pub fn use_ability(ctx: &ReducerContext, ability_id: u64, target_id: u64) {
    let player = ctx.db.player().identity().find(&ctx.sender).unwrap();
    let ability = ctx.db.ability().id().find(&ability_id).unwrap();
    
    // Validate: Player has ability unlocked
    if !player.abilities.contains(&ability_id) {
        return; // Silent fail to prevent info leaking
    }
    
    // Validate: Cooldown expired
    if player.last_ability_use + ability.cooldown > ctx.timestamp {
        return;
    }
    
    // Validate: Sufficient mana
    if player.mana < ability.mana_cost {
        return;
    }
    
    // Execute ability (validated server-side)
    apply_ability_effects(ctx, player, ability, target_id);
}
```

### 13.7 Monitoring and Observability

**Client-Side:**

- Unity Analytics for user behavior
- Crash reporting (Unity Cloud Diagnostics)
- FPS and performance metrics

**Server-Side:**

- SpacetimeDB built-in metrics (transaction rate, memory usage)
- Custom logging via `log` crate
- Database query performance tracking

**Recommended Tools:**

- Grafana for metrics visualization
- Prometheus for time-series data
- Sentry for error tracking

---

## 14. Risks and Mitigations

| Risk                                  | Impact | Probability | Mitigation                                                                                                 |
| ------------------------------------- | ------ | ----------- | ---------------------------------------------------------------------------------------------------------- |
| **Scope Creep**                       | High   | High        | Strict MVP definition, phased roadmap with multiplayer as Phase 4, monthly reviews                         |
| **SpacetimeDB Learning Curve**        | Medium | Medium      | Start with simple reducers, leverage documentation and examples, Clockwork Labs community support          |
| **Rust Learning Curve**               | Medium | Medium      | Focus on game logic only (not systems programming), use AI for boilerplate, comprehensive examples in docs |
| **Client-Server Desync**              | High   | Medium      | Extensive testing, server reconciliation, client-side prediction with rollback                             |
| **SpacetimeDB Scaling Limits**        | Medium | Low         | Monitor performance early, plan for sharding if >100 concurrent players needed                             |
| **Performance Issues (Client)**       | High   | Medium      | Early profiling, LOD systems, object pooling, occlusion culling                                            |
| **Performance Issues (Server)**       | High   | Medium      | Optimize hot reducer paths, use indexes on tables, batch operations where possible                         |
| **Unity Version Compatibility**       | Medium | Low         | Stick to LTS version, document all package versions                                                        |
| **SpacetimeDB Version Compatibility** | Medium | Low         | Use stable releases, test module upgrades in staging                                                       |
| **Network Latency**                   | Medium | High        | Client-side prediction, lag compensation, dedicated servers in multiple regions                            |
| **Security Vulnerabilities**          | High   | Medium      | Server-authoritative design, input validation, rate limiting, code audits                                  |
| **Cheating/Exploits**                 | High   | High        | All game logic on server, anti-cheat validation, server-side cooldowns and costs                           |
| **Data Loss**                         | High   | Low         | SpacetimeDB commit log ensures durability, regular backups, disaster recovery plan                         |
| **Vendor Lock-in (SpacetimeDB)**      | Medium | Low         | Export capabilities via spacetime SQL, open-source core allows self-hosting                                |
| **Cloud Costs**                       | Medium | Medium      | Monitor usage, optimize data storage, consider self-hosting for production                                 |
| **AI Tool Reliability**               | Medium | Medium      | Always manual review of AI-generated content, fallback workflows                                           |
| **Learning Curve (Users)**            | Medium | High        | Comprehensive documentation, video tutorials, example projects, active community                           |

---

## 15. Success Metrics

### 15.1 Editor Usability

- Time to create basic zone (target: <30 minutes for 64x64 map)
- Time to configure NPC with dialogue (target: <10 minutes)
- Time to set up trigger event (target: <5 minutes)

### 15.2 Technical Performance

- Maintain 60 FPS in editor with 50+ entities visible
- Zone load time <3 seconds on target hardware
- Zero data loss on crash (autosave every 5 minutes)

### 15.3 Content Creation

- Produce 1 playable zone per week (Phase 2+)
- Design 3 unique enemies per week using AI-assisted workflow

---

## 16. Appendices

### 16.1 Glossary

|Term|Definition|
|---|---|
|**SpacetimeDB**|Real-time backend framework and database for apps and games that embeds server logic directly into the database|
|**Reducer**|Server-side function in SpacetimeDB that modifies the database (analogous to API endpoints)|
|**Table**|Data structure in SpacetimeDB that stores game state (analogous to database tables)|
|**Identity**|SpacetimeDB's built-in authentication system for tracking users|
|**WASM**|WebAssembly - binary format that Rust compiles to for SpacetimeDB modules|
|**ScriptableObject**|Unity data container that stores game data outside of scenes (client-side only in ZoneForge)|
|**NavMesh**|Navigation mesh used for AI pathfinding|
|**Tilemap**|Grid-based system for 2D/isometric tile placement|
|**Prefab**|Reusable game object template in Unity|
|**Behavior Tree**|Hierarchical AI decision-making structure|
|**URP**|Universal Render Pipeline - Unity's modern rendering system|
|**Gizmo**|Visual debugging aid in Unity Scene view|
|**Commit Log**|SpacetimeDB's disk-based transaction log for durability|
|**Server-Authoritative**|Architecture where the server validates all game logic (prevents cheating)|
|**Client-Side Prediction**|Technique where client optimistically applies changes before server confirmation|
|**Reconciliation**|Process of correcting client state when server rejects a prediction|
|**Crate**|Rust package/library (equivalent to npm packages)|
|**Cargo**|Rust's build system and package manager|
|**Trait**|Rust's interface/protocol system|
|**Borrow Checker**|Rust's compile-time memory safety system|

### 16.2 References

**Unity:**

- Unity Documentation: https://docs.unity3d.com/
- Unity Visual Scripting: https://unity.com/features/unity-visual-scripting

**SpacetimeDB:**

- SpacetimeDB Documentation: https://spacetimedb.com/docs
- SpacetimeDB GitHub: https://github.com/clockworklabs/SpacetimeDB
- SpacetimeDB Blog: https://spacetimedb.com/blog
- SpacetimeDB Discord Community: https://spacetimedb.com/community

**Rust:**

- The Rust Programming Language (Book): https://doc.rust-lang.org/book/
- Rust by Example: https://doc.rust-lang.org/rust-by-example/
- Cargo Documentation: https://doc.rust-lang.org/cargo/

### 16.3 Code Snippets

#### Example: Zone Table (Rust + SpacetimeDB)

```rust
use spacetimedb::{table, SpacetimeType};

#[derive(SpacetimeType)]
#[table(name = zone, public)]
pub struct Zone {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub name: String,
    pub grid_width: u32,
    pub grid_height: u32,
    pub cell_size: f32,
    pub max_players: u32,
}

#[reducer]
pub fn create_zone(ctx: &ReducerContext, name: String, width: u32, height: u32) {
    ctx.db.zone().insert(Zone {
        id: 0, // auto_inc
        name,
        grid_width: width,
        grid_height: height,
        cell_size: 1.0,
        max_players: 100,
    });
}
```

#### Example: Player Movement (Server-Authoritative)

```rust
use spacetimedb::{reducer, ReducerContext, Identity};

#[reducer]
pub fn move_player(ctx: &ReducerContext, new_x: f32, new_y: f32) {
    let player_identity = ctx.sender;
    
    // Find the player
    let Some(player) = ctx.db.player().identity().find(&player_identity) else {
        log::warn!("Player not found: {:?}", player_identity);
        return;
    };
    
    // Validate movement (server-side anti-cheat)
    let distance = ((new_x - player.position_x).powi(2) + 
                    (new_y - player.position_y).powi(2)).sqrt();
    
    let max_distance = player.movement_speed * 0.1; // 100ms tick assumption
    
    if distance > max_distance {
        log::warn!("Player {:?} attempted speed hack", player_identity);
        return; // Reject movement
    }
    
    // Check zone boundaries
    let zone = ctx.db.zone().id().find(&player.zone_id).unwrap();
    if new_x < 0.0 || new_x > zone.grid_width as f32 ||
       new_y < 0.0 || new_y > zone.grid_height as f32 {
        return; // Out of bounds
    }
    
    // Update player position
    ctx.db.player().identity().update(Player {
        position_x: new_x,
        position_y: new_y,
        ..player
    });
    
    // All subscribed clients automatically receive this update!
}
```

#### Example: Unity Client Integration

```csharp
using SpacetimeDB.SDK;
using UnityEngine;

public class PlayerController : MonoBehaviour
{
    private float moveSpeed = 5f;
    private Identity localPlayerIdentity;
    
    void Start()
    {
        // Connect to SpacetimeDB
        SpacetimeDBClient.instance.onConnect += OnConnected;
        SpacetimeDBClient.instance.Connect("wss://your-game.spacetimedb.com", "zoneforge_server");
        
        // Subscribe to player updates
        Player.OnInsert += OnPlayerSpawned;
        Player.OnUpdate += OnPlayerMoved;
    }
    
    void OnConnected(DbConnection connection, Identity identity, string token)
    {
        localPlayerIdentity = identity;
        
        // Create player on server
        Reducers.create_player(connection, "PlayerName");
    }
    
    void OnPlayerMoved(Player oldPlayer, Player newPlayer, ReducerEvent callInfo)
    {
        // Only update if this is our local player
        if (newPlayer.identity == localPlayerIdentity)
        {
            transform.position = new Vector3(newPlayer.position_x, 0, newPlayer.position_y);
        }
    }
    
    void Update()
    {
        if (localPlayerIdentity == null) return;
        
        // Get input
        float moveX = Input.GetAxis("Horizontal");
        float moveY = Input.GetAxis("Vertical");
        
        if (moveX != 0 || moveY != 0)
        {
            // Calculate new position (client-side prediction)
            Vector3 newPos = transform.position + new Vector3(moveX, 0, moveY) * moveSpeed * Time.deltaTime;
            
            // Call server reducer
            Reducers.move_player(SpacetimeDBClient.instance.connection, newPos.x, newPos.z);
            
            // Optimistic update (will be corrected by server if invalid)
            transform.position = newPos;
        }
    }
}
```

#### Example: Combat System (Server Reducer)

```rust
#[reducer]
pub fn use_ability(ctx: &ReducerContext, ability_id: u64, target_id: u64) {
    let attacker_identity = ctx.sender;
    
    let Some(attacker) = ctx.db.player().identity().find(&attacker_identity) else {
        return;
    };
    
    let Some(target) = ctx.db.player().id().find(&target_id) else {
        return;
    };
    
    let Some(ability) = ctx.db.ability().id().find(&ability_id) else {
        return;
    };
    
    // Validate: Same zone
    if attacker.zone_id != target.zone_id {
        return;
    }
    
    // Validate: In range
    let distance = ((attacker.position_x - target.position_x).powi(2) +
                    (attacker.position_y - target.position_y).powi(2)).sqrt();
    if distance > ability.range {
        return;
    }
    
    // Validate: Cooldown
    if attacker.last_ability_time + ability.cooldown > ctx.timestamp.elapsed().as_secs() {
        return;
    }
    
    // Validate: Mana
    if attacker.mana < ability.mana_cost {
        return;
    }
    
    // Apply damage
    let new_health = (target.health - ability.damage).max(0);
    
    ctx.db.player().id().update(Player {
        health: new_health,
        ..target
    });
    
    // Update attacker mana and cooldown
    ctx.db.player().identity().update(Player {
        mana: attacker.mana - ability.mana_cost,
        last_ability_time: ctx.timestamp.elapsed().as_secs(),
        ..attacker
    });
    
    // Log combat event
    ctx.db.combat_log().insert(CombatLog {
        id: 0,
        timestamp: ctx.timestamp.elapsed().as_secs(),
        attacker_id: attacker.id,
        target_id: target.id,
        ability_id,
        damage: ability.damage,
    });
    
    // Check for death
    if new_health == 0 {
        handle_player_death(ctx, target.id);
    }
}
```

---

**— End of Document —**

_This Detailed Design Document serves as the technical blueprint for ZoneForge, a multiplayer-ready Isometric RPG World Editor. The hybrid architecture combining Unity's powerful editor capabilities with SpacetimeDB's revolutionary real-time backend enables solo developers and small teams to create professional-quality multiplayer RPGs without the traditional infrastructure complexity. This document should be treated as a living document, updated as development progresses and new requirements emerge._

**Key Innovation**: By leveraging SpacetimeDB, ZoneForge eliminates the need for traditional game servers, caching layers, and complex DevOps, allowing developers to focus on game design and content creation rather than infrastructure management.