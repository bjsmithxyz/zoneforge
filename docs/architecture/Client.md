# ZoneForge — Unity Game Client Architecture

## Document History

| Version | Date | Summary |
|---------|------|---------|
| 1.3 | 2026-05-10 | Added Enemy/, Zone/ (portals, drops, transfers), inventory + atmosphere; Heavy tables now filtered by `CurrentZoneId`; `LookupCache` + `EntityInstanceManager`; updated scene singleton list |
| 1.2 | 2026-03-24 | Corrected project structure: `TerrainChunkData`, `TerrainRenderer`, `WaterRenderer`, `NavMeshManager`, `EditorState` moved to `Runtime/`; added `UI/`; fixed `Prefab/` (no 's'); noted `ConnectionTest.cs` |
| 1.1 | 2026-03-07 | Added combat system (CombatManager, CombatInputHandler, pool manager, VFX prefabs) |
| 1.0 | 2026-02-01 | Initial document |

---

## What It Is

`zoneforge-client` is the **standalone game runtime** — the application players use to play ZoneForge. World building is done in the separate `zoneforge-editor` application.

## Engine & Rendering

- **Unity 2022.3 LTS** (2022.3.62f3) with **Universal Render Pipeline (URP), 3D**
- Top-down / isometric camera perspective, 3D world on the X/Z plane (Y = up)
- Target platforms: Windows, macOS, Linux, WebGL

## Project Structure

```text
Assets/
├── Scripts/
│   ├── Runtime/        ← SpacetimeDB integration + shared runtime components
│   │   ├── SpacetimeDBManager.cs       ← singleton; connects, subscribes,
│   │   │                                  fires all table events; `CurrentZoneId`
│   │   │                                  controls heavy-table filters
│   │   ├── LookupCache.cs              ← hot-lookup caches (Ability, EnemyDef,
│   │   │                                  ItemDef, PlayerCooldown) populated in
│   │   │                                  OnSubscriptionApplied
│   │   ├── EntityInstanceManager.cs    ← spawns/destroys placeholder GOs for
│   │   │                                  EntityInstance rows in current zone
│   │   ├── TerrainChunkData.cs         ← in-memory height/splat grid (shared with editor)
│   │   ├── TerrainRenderer.cs          ← subscribes to TerrainChunk, builds Mesh
│   │   ├── WaterRenderer.cs            ← flat quad Mesh at zone.water_level
│   │   ├── NavMeshManager.cs           ← NavMesh baking and agent setup
│   │   ├── EditorState.cs              ← active zone selection, hover state
│   │   ├── AtmosphereController.cs     ← applies MoodPreset (sun/ambient/fog/post-fx)
│   │   │                                  from WorldClock + WeatherState
│   │   └── AmbientAudioMixer.cs        ← 3-layer crossfade: base/weather/time
│   ├── Player/
│   │   ├── PlayerManager.cs            ← singleton; spawns/destroys capsules per Player row
│   │   └── PlayerController.cs         ← WASD prediction + reconciliation; remote lerp
│   ├── Combat/
│   │   ├── ZoneForgePoolManager.cs     ← singleton; pre-allocates projectile + VFX pools
│   │   ├── PooledProjectile.cs         ← self-returns to pool on impact/timeout
│   │   ├── PooledVFX.cs                ← self-returns when ParticleSystem finishes
│   │   ├── CombatManager.cs            ← CombatLog → VFX, death/respawn overlay, hit-pause
│   │   ├── CombatInputHandler.cs       ← Tab cycle-target, 1/2/3 use ability, R respawn
│   │   └── SelectionTarget.cs          ← marker component for Tab-target picking
│   ├── Enemy/
│   │   ├── EnemyManager.cs             ← singleton; spawns/destroys enemy GOs
│   │   ├── EnemyController.cs          ← position lerp, AI state visuals
│   │   └── EnemyHealthBar.cs           ← world-space health bar
│   ├── Zone/
│   │   ├── ZoneController.cs           ← zone bootstrap / scene wiring
│   │   ├── PortalManager.cs            ← portal ring GOs + proximity trigger
│   │   ├── ZoneTransferManager.cs      ← portal proximity → reducer → CurrentZoneId flip
│   │   ├── RippleWarpEffect.cs         ← screen-space transfer warp
│   │   ├── ItemDropRenderer.cs         ← world-space GO per ItemDrop row
│   │   └── ItemPickupManager.cs        ← F key proximity pickup → pickup_item reducer
│   ├── UI/
│   │   ├── HotbarUI.cs                 ← slots 1–5 with cooldown indicators
│   │   ├── SelfHudUI.cs                ← local player HUD (health, mana)
│   │   ├── PlayerHealthBar.cs          ← world-space health bar over capsule
│   │   ├── FloatingTextPopup.cs        ← floating damage / heal numbers
│   │   ├── InventoryManager.cs         ← singleton; caches inventory/equipment/item defs/drops
│   │   ├── InventoryUI.cs              ← grid UI + tooltips
│   │   └── EquipmentUI.cs              ← character-sheet slot UI
│   ├── Data/                           ← ScriptableObjects
│   │   ├── WorldData.cs
│   │   ├── ZoneVisualData.cs
│   │   ├── MoodPreset.cs               ← sun/ambient/fog curves + audio
│   │   └── MoodPresetRegistry.cs       ← id → preset lookup (Resources/MoodPresets/)
│   ├── ConnectionTest.cs               ← dev-only connection smoke test
│   ├── ZoneForgeRuntime.asmdef         ← references SpacetimeDB SDK + ZoneForgeAutogen
│   └── autogen/                        ← `spacetime generate` output (DO NOT EDIT)
│       ├── Tables/  Reducers/  Types/
│       └── ZoneForgeAutogen.asmdef
├── Prefab/                             ← Folder name is "Prefab" (no 's')
│   ├── FireballProjectile.prefab       ← pool key: projectile_fireball
│   ├── FireballMaterial.mat
│   ├── ImpactFireVFX.prefab            ← pool key: vfx_impact_fire
│   └── ImpactGenericVFX.prefab         ← pool key: vfx_impact_generic
├── Resources/
│   ├── MoodPresets/                    ← village_day, forest, night_camp
│   └── WeatherVFX/                     ← Rain, fog prefabs (Storm/Snow stub as Rain)
└── Art/
    ├── Models/                         ← 3D character + prop models
    ├── Sprites/                        ← UI sprites, icons
    └── Materials/
        ├── Graphs/                     ← Material Maker .mmg source files
        ├── Exports/                    ← Albedo, normal, roughness maps
        └── Parameters/                 ← Saved Material Maker presets
```

## SpacetimeDB Integration

The client communicates with the server exclusively through the **SpacetimeDB C# SDK** via `SpacetimeDBManager` (singleton in `Scripts/Runtime/`).

### Connection lifecycle

1. `DbConnection.Builder()` — connect to `http://localhost:3000` (dev) or cloud URL (prod). Auth token persisted in `PlayerPrefs.spacetimedb_token` so the same `Identity` is reused across sessions.
2. `OnConnect` — store `LocalIdentity`; build a subscription set (see below).
3. `OnSubscriptionApplied` — populate `LookupCache`, register every table callback, prime atmosphere from the current `Zone`, fire `OnConnected`.
4. `Update()` — calls `_conn.FrameTick()` every frame to drain SDK messages.

### Subscriptions

`SpacetimeDBManager.CurrentZoneId` controls which zone the client renders. Light tables stream unfiltered; heavy tables are filtered to the current zone so we never download other zones' terrain / entities / drops.

| Filter | Tables |
|--------|--------|
| Unfiltered | `player`, `zone`, `ability`, `player_cooldown`, `status_effect`, `combat_log`, `enemy_def`, `item_def`, `inventory`, `equipment`, `world_clock` |
| `WHERE zone_id = CurrentZoneId` | `terrain_chunk`, `entity_instance`, `enemy`, `item_drop`, `weather_state` |
| `WHERE source_zone_id = CurrentZoneId` OR `WHERE dest_zone_id = CurrentZoneId` | `portal` (so back-portals are also visible) |

On zone transfer, `ZoneTransferManager` calls the `enter_zone` reducer, waits for the `Player` row's `zone_id` to update, then calls `SpacetimeDBManager.SetCurrentZoneId(...)` which fires `OnZoneChanged` so subscribers can purge old GOs and re-subscribe.

### Player system

- `PlayerManager` (singleton): in `OnConnected` it backfills existing `Player` rows via `Conn.Db.Player.Iter()` and calls `CreatePlayer` if no row exists for the local identity.
- `PlayerController` per capsule:
  - **Local** (`isLocal = true`): WASD input applied immediately (prediction), `MovePlayer` reducer at 10 Hz, `Vector3.MoveTowards` reconciliation when drift > 1 m. Reconciliation is suppressed while input is held.
  - **Remote**: lerp toward the latest server position (`Time.deltaTime × 10`).
- Camera (`Camera.main`) parented to local player capsule, offset `(0, 10, −7)`, rotation `(55°, 0°, 0°)`.

### Required scene GameObjects

| GameObject | Component(s) | Notes |
| --- | --- | --- |
| `SpacetimeDBManager` | `SpacetimeDBManager`, `CombatInputHandler` | Connects, subscribes, fires events; Tab/1/2/3/R input |
| `PlayerManager` | `PlayerManager` | Spawns/destroys player capsules |
| `EnemyManager` | `EnemyManager` | Spawns/destroys enemy capsules per `Enemy` row |
| `CombatManager` | `CombatManager` | CombatLog → VFX + death overlay + hit-pause |
| `PoolManager` | `ZoneForgePoolManager` | Pre-allocates projectile + VFX pools at startup |
| `InventoryManager` | `InventoryManager` | Caches inventory + equipment + drops; fires UI events |
| `PortalManager` | `PortalManager` | Portal ring GOs + proximity check |
| `ZoneTransferManager` | `ZoneTransferManager` | Coordinates portal transfer + ripple warp |
| `ItemPickupManager` | `ItemPickupManager` | F-key proximity → `pickup_item` |
| `AtmosphereController` | `AtmosphereController`, `AmbientAudioMixer` | Applies `MoodPreset` from `WorldClock`/`WeatherState` |

> `EnemyManager` must be present before `CombatManager.OnConnected` runs — `CombatManager` looks up enemy GOs by id when spawning impact VFX.

### Generated bindings

`spacetime generate --lang csharp --out-dir Assets/Scripts/autogen --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm` (run from `client/`). Output is split into `autogen/Tables/`, `autogen/Reducers/`, `autogen/Types/`. **Never edit** — regenerate after any server schema change.

## Key Unity Packages

| Package | Purpose |
|---------|---------|
| Universal Render Pipeline (URP) | Core 3D rendering pipeline |
| Cinemachine | Camera control and cutscenes |
| Timeline | Cutscene sequencing |
| TextMeshPro | High-quality UI text |
| UI Toolkit | (light use — most HUD is uGUI) |
| SpacetimeDB Unity SDK | Real-time backend integration |
| AI Navigation (`com.unity.ai.navigation`) | NavMesh baking + agents |

## Asset Pipeline

Textures are authored procedurally:

- **Material Maker** — node-based procedural texture graphs (`.mmg` → albedo/normal/roughness PNG sets)
- **ArmorPaint** — 3D texture painting with smart materials
- **Blender** — 3D modelling and Geometry Nodes for procedural foliage/scatter

All exported maps land in `Assets/Art/Materials/Exports/`.

## Object Pooling

Combat projectiles and VFX use `ZoneForgePoolManager` to avoid runtime allocations. `PooledProjectile` and `PooledVFX` must be present on every combat prefab. Pool keys are string constants matched between the pool manager and the prefab (`projectile_fireball`, `vfx_impact_fire`, `vfx_impact_generic`). Pool capacities are tuned during load testing (§8A in Detailed Design).

## See Also

- [Overview.md](Overview.md) — System architecture diagram and data flow
- [Editor.md](Editor.md) — World editor architecture (for comparison)
- [Diagrams.md](Diagrams.md) — Schema + sequence diagrams (player movement, combat, terrain edit)
- [../guides/Client_Dev.md](../guides/Client_Dev.md) — Daily Unity development workflow
- [../guides/Getting_Started.md](../guides/Getting_Started.md) — Full environment setup
