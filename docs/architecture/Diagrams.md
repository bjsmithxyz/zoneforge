# ZoneForge — Architecture Diagrams

All diagrams use [Mermaid](https://mermaid.js.org/) and render directly on GitHub.

---

## 1. System Architecture

Three standalone applications sharing a single SpacetimeDB backend. The editor authors world data; the game client renders + acts on it. All state changes propagate automatically to every subscriber.

```mermaid
graph TB
    subgraph EditorApp["🗺️  zoneforge-editor  (Standalone Unity App)"]
        direction TB
        Toolbar["ToolbarController"]
        ZCP["ZoneCreationPanel\ncreate · select · mood"]
        TilePP["TilePalettePanel\nbrush type · radius · strength"]
        EntPal["EntityPalettePanel\nNPC / Enemy / Prop"]
        EnDef["EnemyDefCreationPanel"]
        Loot["LootCreationPanel"]
        Graph["WorldGraphPanel\nportal nodes + edges"]
        WxDbg["WeatherDebugPanel (admin)"]
        TR_Ed["TerrainRenderer\nbuild mesh from chunk data"]
        TP["TerrainPainter\nraycast → brush → reducer"]
        EntPl["EntityPlacer + EntityRenderer"]
        EnR["EnemyRenderer"]
        PortR["PortalRenderer"]
        Atm_Ed["AtmosphereController + AmbientAudioMixer"]
    end

    subgraph ClientApp["🎮  zoneforge-client  (Standalone Unity App)"]
        direction TB
        PM["PlayerManager / PlayerController"]
        EM["EnemyManager / EnemyController"]
        CM["CombatManager + CombatInputHandler"]
        TR_Cl["TerrainRenderer / WaterRenderer"]
        Pool["ZoneForgePoolManager (projectile + VFX)"]
        HUD["HotbarUI · SelfHudUI · FloatingTextPopup"]
        Inv["InventoryManager + InventoryUI + EquipmentUI"]
        Port["PortalManager + ZoneTransferManager + RippleWarpEffect"]
        Drops["ItemDropRenderer + ItemPickupManager"]
        Atm_Cl["AtmosphereController + AmbientAudioMixer"]
    end

    subgraph STDB["⚙️  SpacetimeDB  ·  zoneforge-server  (Rust WASM Module)"]
        direction LR
        subgraph LibRs["lib.rs"]
            ZoneT["Zone"]
            PlayerT["Player"]
            EntIT["EntityInstance"]
            AdminT["Admin"]
        end
        subgraph TerrainMod["terrain.rs"]
            ChunkT["TerrainChunk"]
        end
        subgraph CombatMod["combat.rs"]
            AbilT["Ability"]
            CoolT["PlayerCooldown"]
            SEff["StatusEffect"]
            CLogT["CombatLog"]
            SEtick["StatusEffectTick (1s)"]
            MRtick["ManaRegenTick (2s)"]
            CLtick["CombatLogPruneTick (60s)"]
        end
        subgraph EnemyMod["enemy.rs"]
            EnemyDefT["EnemyDefinition"]
            SpawnT["SpawnPoint"]
            EnemyT["Enemy"]
            ARtick["EnemyRespawnTick"]
            AItick["AiTick (500ms)"]
        end
        subgraph PortalMod["portal.rs"]
            PortalT["Portal"]
        end
        subgraph InvMod["inventory.rs / loot.rs"]
            ItemDefT["ItemDefinition"]
            InvT["Inventory"]
            EquipT["Equipment"]
            LootT["LootTable"]
            DropT["ItemDrop"]
        end
        subgraph AtmMod["atmosphere.rs"]
            WClk["WorldClock"]
            Wx["WeatherState"]
            WClkTick["WorldClockTick (1s)"]
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
        u32     mood_preset_id
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

    Admin {
        Identity identity PK
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

    EnemyDefinition {
        u64       id              PK
        string    name
        EnemyType enemy_type     "Melee | Ranged | Caster"
        string    prefab_name
        i32       max_health
        i32       damage
        f32       aggro_range
        f32       attack_range
        u64       attack_speed_ms
        f32       move_speed
    }

    SpawnPoint {
        u64 id              PK
        u64 zone_id         FK
        f32 x
        f32 y
        u64 enemy_def_id    FK
        u32 max_count
        u32 respawn_delay_s
    }

    Enemy {
        u64     id               PK
        u64     zone_id          FK
        u64     spawn_point_id   "Option<FK>"
        u64     enemy_def_id     FK
        f32     position_x
        f32     position_y
        f32     home_x
        f32     home_y
        i32     health
        AiState ai_state        "Idle | Chase | Attack"
        u64     target_player_id "Option<FK>"
        u64     last_attack_us
        bool    is_dead
    }

    Portal {
        u64    id              PK
        u64    source_zone_id  FK
        u64    dest_zone_id    FK
        f32    source_x
        f32    source_y
        f32    dest_spawn_x
        f32    dest_spawn_y
        bool   bidirectional
        string label
    }

    ItemDefinition {
        u64      id          PK
        string   name
        string   description
        ItemType item_type   "Weapon | Armor | Accessory | Consumable"
        Rarity   rarity      "Common | Uncommon | Rare | Epic"
        string   icon_name
        i32      damage_bonus
        i32      armor_bonus
        i32      healing
        u32      value
    }

    Inventory {
        u64 id          PK
        u64 player_id   FK
        u64 item_def_id FK
        u32 quantity
    }

    Equipment {
        u64 player_id    PK
        u64 weapon_id    "Option<FK>"
        u64 armor_id     "Option<FK>"
        u64 accessory_id "Option<FK>"
    }

    LootTable {
        u64 id           PK
        u64 enemy_def_id FK
        u64 item_def_id  FK
        u32 drop_chance  "0-100"
        u32 min_quantity
        u32 max_quantity
    }

    ItemDrop {
        u64 id          PK
        u64 zone_id     FK
        u64 item_def_id FK
        u32 quantity
        f32 pos_x
        f32 pos_y
    }

    WorldClock {
        u8        id              PK "always 0"
        u16       minutes_of_day  "0-1439"
        Timestamp last_tick
    }

    WeatherState {
        u64         zone_id    PK
        WeatherKind kind       "Clear | Rain | Storm | Fog | Snow"
        f32         intensity  "0.0 - 1.0"
        Timestamp   started_at
    }

    Zone            ||--o{ TerrainChunk    : "divided into 32×32 chunks"
    Zone            ||--o{ EntityInstance  : "contains"
    Zone            ||--o{ Player          : "hosts"
    Zone            ||--o{ SpawnPoint      : "contains"
    Zone            ||--o{ Enemy           : "hosts"
    Zone            ||--o{ ItemDrop        : "contains"
    Zone            ||--|| WeatherState    : "per-zone weather"
    Zone            ||--o{ Portal          : "as source"
    Zone            ||--o{ Portal          : "as destination"
    Player          ||--o{ PlayerCooldown  : "tracks"
    Player          ||--o{ StatusEffect    : "affected by"
    Player          ||--o{ CombatLog       : "appears in (attacker/target)"
    Player          ||--o{ Inventory       : "carries"
    Player          ||--|| Equipment       : "wears"
    Ability         ||--o{ PlayerCooldown  : "per-player cooldown row"
    Ability         ||--o{ CombatLog       : "logged per use"
    EnemyDefinition ||--o{ SpawnPoint      : "spawns"
    EnemyDefinition ||--o{ Enemy           : "instantiates"
    EnemyDefinition ||--o{ LootTable       : "drops"
    SpawnPoint      ||--o{ Enemy           : "respawns"
    ItemDefinition  ||--o{ Inventory       : "stocked as"
    ItemDefinition  ||--o{ LootTable       : "drops as"
    ItemDefinition  ||--o{ ItemDrop        : "ground drop"
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
    Note over S : Validate: 4096-byte arrays, finite floats, chunk in bounds
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
    Note over S : Validate finite + clamp to zone bounds
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

Server-authoritative combat: every check (alive, range, cooldown, mana) happens in the reducer before any state changes.

```mermaid
sequenceDiagram
    actor       P  as Attacker (Client)
    actor       T  as Target (Player or Enemy)
    participant S  as SpacetimeDB

    P  ->>  S  : use_ability(ability_id, target_id)  [vs Player]
    Note right of P : attack_enemy(ability_id, enemy_id) for Enemy targets
    Note over S : 1. Caller alive?<br/>2. Ability exists?<br/>3. Self-cast target valid?<br/>4. Target alive + in range?<br/>5. Off cooldown?<br/>6. Enough mana?
    S  ->>  S  : Deduct mana from Player row
    S  ->>  S  : Upsert PlayerCooldown row
    S  ->>  S  : apply_damage(target, amount)
    S  ->>  S  : Update target row (health, is_dead)
    S  ->>  S  : Insert CombatLog row

    S  -->> P  : PlayerCooldown.OnInsert/Update → hotbar
    S  -->> P  : Player.OnUpdate (own mana) → mana bar
    S  -->> P  : CombatLog.OnInsert → projectile / impact VFX
    S  -->> T  : Player/Enemy.OnUpdate → health bar update

    alt target is_dead = true (Player)
        S  -->> T  : Player.OnUpdate → death overlay
        T  ->>  S  : respawn()  [R key]
        Note over S : Reset to zone centre<br/>Restore full HP + mana<br/>Delete StatusEffects
        S  -->> T  : Player.OnUpdate → respawn
    else target is_dead = true (Enemy)
        S  ->>  S  : spawn_loot_drops(enemy)<br/>→ ItemDrop rows
        S  ->>  S  : Schedule EnemyRespawnTick<br/>(after spawn_point.respawn_delay_s)
        S  -->> P  : ItemDrop.OnInsert → ground GO
    end

    alt ability has DoT (Burn / Poison)
        loop Every 1 s (StatusEffectTick)
            S  ->>  S  : tick_status_effects()
            S  ->>  S  : apply_damage per active DoT
            S  ->>  S  : Remove expired effects
            S  -->> T  : Player.OnUpdate + CombatLog.OnInsert
        end
    end

    loop Every 60 s
        S  ->>  S  : prune_combat_log() — cap to 1000 rows
    end

    loop Every 2 s
        S  ->>  S  : tick_mana_regen() — +10 mana per living player
    end
```

---

## 6. Zone Transfer (Portal)

How a player walks through a portal and resubscribes to the destination zone's heavy tables.

```mermaid
sequenceDiagram
    actor       P  as Player
    participant Cl as Client (Unity)
    participant S  as SpacetimeDB

    Note over Cl : PortalManager.Update<br/>checks proximity to portal rings

    P  ->>  Cl : Walk into portal trigger
    Cl ->>  Cl : RippleWarpEffect.Play()<br/>(screen-space warp begins)
    Cl ->>  S  : enter_zone()
    Note over S : Find nearest portal in caller's zone<br/>Update Player.zone_id + position
    S  -->> Cl : Player.OnUpdate (own row)
    Cl ->>  Cl : ZoneTransferManager sees new zone_id<br/>→ SpacetimeDBManager.SetCurrentZoneId()
    Cl ->>  Cl : Fire OnZoneChanged event<br/>→ managers purge old-zone GOs
    Cl ->>  S  : Re-subscribe heavy tables WHERE zone_id = new<br/>(terrain_chunk, entity_instance, enemy, portal, item_drop, weather_state)
    S  -->> Cl : Snapshot inserts for new zone
    Cl ->>  Cl : Renderers rebuild from new rows
    Cl ->>  Cl : RippleWarpEffect resolves
```

---

## 7. Atmosphere (Time of Day + Weather)

How the world clock and per-zone weather flow into the client renderers.

```mermaid
sequenceDiagram
    participant S    as SpacetimeDB
    participant Atm  as AtmosphereController
    participant Aud  as AmbientAudioMixer
    participant VFX  as Weather VFX prefab

    loop Every 1 s
        S  ->>  S   : tick_world_time() — minutes_of_day += 1 (mod 1440)
        S  -->> Atm : WorldClock.OnUpdate
        Atm ->>  Atm: Sample MoodPreset curves at minutes_of_day<br/>→ sun rot/intensity, ambient colour, fog, post-fx
        Atm ->>  Aud: Crossfade time-layer audio
    end

    Note over S : Admin changes weather:<br/>change_weather(zone_id, kind, intensity)
    S  -->> Atm : WeatherState.OnUpdate (filtered to current zone)
    Atm ->>  VFX: Spawn/replace Resources/WeatherVFX/<kind>
    Atm ->>  Aud: Crossfade weather-layer audio

    Note over S : Admin changes mood:<br/>set_zone_mood(zone_id, mood_preset_id)
    S  -->> Atm : Zone.OnUpdate (mood_preset_id)
    Atm ->>  Atm: Load new MoodPreset from MoodPresetRegistry<br/>Replace base audio, refresh curves
```

---

## 8. Project Repository Layout

```mermaid
graph TD
    Root["zoneforge/\n(umbrella repo)"]

    Root --> Docs["docs/\narchitecture · design · guides · decisions"]
    Root --> Client["client/\nzoneforge-client submodule\nUnity 2022.3 LTS · 3D URP"]
    Root --> Editor["editor/\nzoneforge-editor submodule\nUnity 2022.3 LTS · 3D URP"]
    Root --> Server["server/\nzoneforge-server submodule\nRust · WASM"]

    Client --> CScripts["Assets/Scripts/\nRuntime · Player · Combat · Enemy · Zone · UI · Data"]
    Client --> CAuto["autogen/ (generated)"]
    Editor --> EScripts["Assets/Scripts/\nRuntime · Data · Editor"]
    Editor --> EUI["Assets/UI/ .uxml/.uss"]
    Editor --> EAuto["autogen/ (generated)"]
    Server --> SRC["spacetimedb/src/\nlib.rs · terrain · combat · enemy · portal\ninventory · loot · atmosphere"]
```
