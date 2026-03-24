# ZoneForge — Architecture Diagrams

All diagrams use [Mermaid](https://mermaid.js.org/) and render directly on GitHub.

---

## 1. System Architecture

Three standalone applications sharing a single SpacetimeDB backend. The editor writes world data; the game client reads and acts on it. All state changes propagate automatically to every subscriber.

```mermaid
graph TB
    subgraph EditorApp["🗺️  zoneforge-editor  (Standalone Unity App)"]
        direction TB
        ZCP["ZoneCreationPanel\ncreate / select zones"]
        TilePP["TilePalettePanel\nbrush type · radius · strength"]
        EntPal["EntityPalettePanel\nNPC / Enemy / Prop"]
        TR_Ed["TerrainRenderer\nbuild mesh from chunk data"]
        TP["TerrainPainter\nraycast → brush → reducer"]
        EntPl["EntityPlacer\nclick-to-place → reducer"]
        EntR["EntityRenderer\nplaceholder cubes"]
    end

    subgraph ClientApp["🎮  zoneforge-client  (Standalone Unity App)"]
        direction TB
        PM["PlayerManager\nspawn / destroy capsules"]
        PC["PlayerController\nWASD · prediction · reconcile"]
        CM["CombatManager\nVFX · death overlay · respawn"]
        CIH["CombatInputHandler\nTab · 1/2/3 · R"]
        TR_Cl["TerrainRenderer\nbuild mesh from chunk data"]
        WR["WaterRenderer\nflat quad at water_level"]
        HUD["HUD\nhotbar · health · mana · floats"]
        Pool["ZoneForgePoolManager\nprojectiles + VFX pools"]
    end

    subgraph STDB["⚙️  SpacetimeDB  ·  zoneforge-server  (Rust WASM Module)"]
        direction LR
        subgraph WorldTables["World Tables"]
            ZoneT["Zone"]
            ChunkT["TerrainChunk"]
            EntIT["EntityInstance"]
        end
        subgraph PlayerTables["Player Tables"]
            PlayerT["Player"]
            CoolT["PlayerCooldown"]
            SEff["StatusEffect"]
        end
        subgraph CombatTables["Combat Tables"]
            AbilT["Ability"]
            CLogT["CombatLog"]
        end
        subgraph Schedulers["Schedulers"]
            SEtick["StatusEffectTick\n1 Hz — DoT ticks"]
            MRtick["ManaRegenTick\n2 Hz — mana restore"]
        end
    end

    EditorApp  <-->|"SpacetimeDB C# SDK · WebSocket"| STDB
    ClientApp  <-->|"SpacetimeDB C# SDK · WebSocket"| STDB
```

---

## 2. Database Schema

```mermaid
erDiagram
    Zone {
        u64     id             PK
        string  name
        u32     terrain_width
        u32     terrain_height
        f32     water_level
    }

    TerrainChunk {
        u64     id          PK
        u64     zone_id     FK
        u32     chunk_x
        u32     chunk_z
        bytes   height_data "1024 × f32 LE = 4096 bytes"
        bytes   splat_data  "1024 × RGBA u8 = 4096 bytes"
    }

    EntityInstance {
        u64     id          PK
        u64     zone_id     FK
        string  prefab_name
        f32     position_x
        f32     position_y
        f32     elevation   "world-space Y"
        string  entity_type "NPC | Enemy | StaticProp …"
    }

    Player {
        u64      id          PK
        Identity identity    UK
        string   name
        u64      zone_id     FK
        f32      position_x
        f32      position_y
        i32      health
        i32      max_health
        i32      mana
        i32      max_mana
        bool     is_dead
    }

    Ability {
        u64         id           PK
        string      name
        i32         damage       "negative = heal"
        u64         cooldown_ms
        i32         mana_cost
        f32         range        "0 = self-cast"
        AbilityType ability_type "MeleeAttack | Projectile | SelfCast"
    }

    PlayerCooldown {
        u64       id         PK
        u64       player_id  FK
        u64       ability_id FK
        Timestamp ready_at
    }

    StatusEffect {
        u64              id              PK
        u64              target_id       FK
        StatusEffectType effect_type     "Burn | Freeze | Stun | Poison"
        Timestamp        expires_at
        i32              damage_per_tick
    }

    CombatLog {
        u64       id           PK
        Timestamp timestamp
        u64       attacker_id  FK
        u64       target_id    FK
        u64       ability_id   FK
        i32       damage_dealt
        i32       overkill
    }

    Zone           ||--o{ TerrainChunk    : "divided into 32×32 chunks"
    Zone           ||--o{ EntityInstance  : "contains"
    Zone           ||--o{ Player          : "hosts"
    Player         ||--o{ PlayerCooldown  : "tracks"
    Player         ||--o{ StatusEffect    : "affected by"
    Player         ||--o{ CombatLog       : "appears in (attacker or target)"
    Ability        ||--o{ PlayerCooldown  : "per-player cooldown row"
    Ability        ||--o{ CombatLog       : "logged per use"
```

---

## 3. Terrain Editing Workflow

How a brush stroke in the editor propagates to all connected game clients in real time.

```mermaid
sequenceDiagram
    actor       D  as Designer
    participant Ed as Editor (Unity)
    participant S  as SpacetimeDB
    participant Cl as Game Clients (all)

    D  ->>  Ed : Click / drag in viewport
    Ed ->>  Ed : Raycast → TerrainPainter hits MeshCollider
    Ed ->>  Ed : Active brush modifies in-memory TerrainChunkData<br/>(height raise/lower/smooth or splat layer paint)
    Ed ->>  S  : update_terrain_chunk(zone_id, cx, cz, height_data, splat_data)
    Note over S : Validate: 4096-byte arrays, chunk in bounds
    S  ->>  S  : Update TerrainChunk row
    S  -->> Ed : TerrainChunk.OnUpdate callback
    S  -->> Cl : TerrainChunk.OnUpdate callback (automatic)
    Ed ->>  Ed : TerrainRenderer rebuilds Mesh + MeshCollider
    Cl ->>  Cl : TerrainRenderer rebuilds Mesh + MeshCollider
```

---

## 4. Player Movement & Reconciliation

Client-side prediction keeps movement responsive while the server remains authoritative.

```mermaid
sequenceDiagram
    actor       P  as Local Player
    participant Cl as Client (Unity)
    participant S  as SpacetimeDB

    P  ->>  Cl : WASD input held
    Cl ->>  Cl : Apply movement immediately<br/>(client-side prediction)
    Cl ->>  S  : move_player(x, y)  [10 Hz]
    Note over S : Clamp position to zone bounds
    S  ->>  S  : Update Player row
    S  -->> Cl : Player.OnUpdate callback
    alt Drift > 1 m and no input held
        Cl ->>  Cl : MoveTowards server position<br/>(reconciliation)
    else Input held
        Cl ->>  Cl : Suppress reconciliation<br/>(avoids direction-change jitter)
    end

    Note over Cl : Remote players lerp to<br/>latest server position (10× deltaTime)
```

---

## 5. Combat Flow

Server-authoritative combat: every check (range, cooldown, mana) happens in the reducer before any state changes.

```mermaid
sequenceDiagram
    actor       P  as Attacker (Client)
    actor       T  as Target (Client)
    participant S  as SpacetimeDB

    P  ->>  S  : use_ability(ability_id, target_id)  [key 1/2/3]
    Note over S : 1. Caller alive?<br/>2. Ability exists?<br/>3. Self-cast target valid?<br/>4. Target alive + in range?<br/>5. Off cooldown?<br/>6. Enough mana?
    S  ->>  S  : Deduct mana from Player row
    S  ->>  S  : Upsert PlayerCooldown row
    S  ->>  S  : apply_damage(target, amount)
    S  ->>  S  : Update target Player row (health, is_dead)
    S  ->>  S  : Insert CombatLog row

    S  -->> P  : PlayerCooldown.OnInsert → hotbar cooldown indicator
    S  -->> P  : Player.OnUpdate (own mana) → mana bar
    S  -->> P  : CombatLog.OnInsert → spawn projectile / impact VFX
    S  -->> T  : Player.OnUpdate → health bar update

    alt target is_dead = true
        S  -->> T  : Player.OnUpdate → death overlay shown
        T  ->>  S  : respawn()  [R key]
        Note over S : Reset position to zone centre<br/>Restore full HP + mana<br/>Delete all StatusEffects
        S  -->> T  : Player.OnUpdate → respawn at centre
    end

    alt ability has DoT (Burn / Poison)
        loop Every 1 s (StatusEffectTick scheduler)
            S  ->>  S  : tick_status_effects()
            S  ->>  S  : apply_damage per active DoT
            S  ->>  S  : Remove expired effects
            S  -->> T  : Player.OnUpdate + CombatLog.OnInsert
        end
    end
```

---

## 6. Project Repository Layout

```mermaid
graph TD
    Root["zoneforge/\n(umbrella repo)"]

    Root --> Docs["docs/\narchitecture · design · guides · decisions"]
    Root --> Client["client/\nzoneforge-client submodule\nUnity 2022.3 LTS · 3D URP"]
    Root --> Editor["editor/\nzoneforge-editor submodule\nUnity 2022.3 LTS · 3D URP"]
    Root --> Server["server/\nzoneforge-server submodule\nRust · WASM"]

    Client  --> CA["Assets/Scripts/\nRuntime · Player · Combat · Zone · UI"]
    Editor  --> EA["Assets/Scripts/\nRuntime · Data · autogen"]
    Server  --> SA["spacetimedb/src/lib.rs\nAll tables + reducers"]

    CA --> Autogen_C["autogen/\nGenerated C# bindings\n(spacetime generate)"]
    EA --> Autogen_E["autogen/\nGenerated C# bindings\n(spacetime generate)"]
```
