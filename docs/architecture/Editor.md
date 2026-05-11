# ZoneForge — World Editor Architecture

## Document History

| Version | Date | Summary |
|---------|------|---------|
| 1.2 | 2026-05-10 | Added EnemyDef/Loot/WorldGraph/Toolbar/WeatherDebug panels; EnemyRenderer + PortalRenderer; AtmosphereController + AmbientAudioMixer; CameraController; asmdef set updated (`ZoneForgeData` split out) |
| 1.1 | 2026-03-07 | Initial three-panel UI documentation |

---

## What It Is

`zoneforge-editor` is a **standalone desktop application** built in Unity 2022.3 LTS (3D URP). It is not a Unity Editor tool — it is a compiled executable that designers run independently, like WoWEdit or StarEdit.

It connects to the same SpacetimeDB backend as the game client. Changes made in the editor (zones, terrain, entities, portals, enemy defs, loot tables, atmosphere) are immediately visible to all connected game clients in real time.

## Engine & Rendering

- **Unity 2022.3 LTS** (2022.3.62f3) with **Universal Render Pipeline (URP), 3D**
- Top-down / isometric camera perspective
- 3D world on the X/Z plane (Y = up)
- Target platforms: Windows, macOS, Linux

## Project Structure

```text
Assets/
├── Scripts/
│   ├── Runtime/                              ← MonoBehaviours and runtime managers
│   │   ├── SpacetimeDBManager.cs             ← singleton; connect + subscribe + tick
│   │   ├── EditorState.cs                    ← active zone selection, hover state
│   │   ├── UIHoverTracker.cs                 ← blocks terrain painting while hovering UI
│   │   ├── CameraController.cs               ← orbit / pan / zoom in 3D
│   │   ├── ToolbarController.cs              ← top toolbar (panel toggles)
│   │   │
│   │   ├── ZoneCreationPanel.cs              ← UIToolkit: zone form, list of zones,
│   │   │                                       MoodPreset dropdown
│   │   ├── TilePalettePanel.cs               ← UIToolkit: brush type/mode/layer/radius/strength
│   │   ├── EntityPalettePanel.cs             ← UIToolkit: NPC/Enemy/Prop palette
│   │   ├── EnemyDefCreationPanel.cs          ← UIToolkit: define enemy archetypes
│   │   ├── LootCreationPanel.cs              ← UIToolkit: item defs + loot tables
│   │   ├── WorldGraphPanel.cs                ← UIToolkit: portal node graph, edges, drag-to-create
│   │   ├── WeatherDebugPanel.cs              ← UIToolkit (admin): weather kind + intensity, mood
│   │   │
│   │   ├── TerrainPainter.cs                 ← raycast-based painting; calls update_terrain_chunk
│   │   ├── TerrainRenderer.cs                ← subscribes to TerrainChunk; builds Mesh + MeshCollider
│   │   ├── WaterRenderer.cs                  ← flat quad mesh at zone.water_level
│   │   ├── TerrainBrush.cs                   ← abstract brush base (radius, strength, falloff)
│   │   ├── HeightBrush.cs                    ← raise/lower/smooth terrain height
│   │   ├── TextureBrush.cs                   ← paint splatmap layer
│   │   ├── CombinedBrush.cs                  ← height + texture in one stroke
│   │   ├── TerrainChunkData.cs               ← in-memory height/splat grid + index helpers
│   │   │
│   │   ├── EntityPlacer.cs                   ← click-to-place; spawn_entity reducer
│   │   ├── EntityRenderer.cs                 ← placeholder cubes per EntityInstance row
│   │   ├── EntityDefinition.cs               ← data class (DisplayName, PrefabName, EntityType, Color)
│   │   ├── EnemyRenderer.cs                  ← visualises live Enemy rows + spawn points
│   │   ├── PortalRenderer.cs                 ← visualises Portal rows on the terrain
│   │   │
│   │   ├── AtmosphereController.cs           ← applies MoodPreset (sun/ambient/fog/post-fx)
│   │   └── AmbientAudioMixer.cs              ← 3-layer crossfade: base/weather/time
│   │
│   ├── Data/                                 ← ScriptableObjects (asmdef: ZoneForgeData)
│   │   ├── WorldData.cs
│   │   ├── ZoneVisualData.cs
│   │   ├── MoodPreset.cs
│   │   └── MoodPresetRegistry.cs
│   │
│   ├── Editor/                               ← Editor-only dev tools (excluded from builds)
│   │   ├── PlaceholderSpriteGenerator.cs
│   │   └── PlaceholderTileGenerator.cs
│   │
│   └── autogen/                              ← `spacetime generate` output (DO NOT EDIT)
│       ├── Tables/  Reducers/  Types/
│       └── ZoneForgeAutogen.asmdef
│
├── UI/                                       ← UI Toolkit assets
│   ├── ZoneCreationPanel.uxml/.uss           ← zone form, anchored top-left
│   ├── TilePalettePanel.uxml/.uss            ← brush controls, top-right
│   ├── EntityPalettePanel.uxml/.uss          ← entity grid, bottom-right
│   ├── ToolbarController.uxml/.uss           ← top toolbar
│   ├── WorldGraphPanel.uxml/.uss             ← portal node canvas
│   └── LootCreationPanel.uss                 ← (UXML built programmatically)
│
├── Resources/
│   ├── MoodPresets/                          ← village_day, forest, night_camp
│   └── WeatherVFX/                           ← Rain, fog (Storm/Snow stubbed)
│
├── Materials/
│   ├── TerrainSplatmap.mat                   ← references TerrainSplatmap.shader
│   └── WaterUnlit.mat                        ← references WaterUnlit.shader
├── Shaders/
│   ├── TerrainSplatmap.shader                ← splat-blend + Lambert + ambient
│   └── WaterUnlit.shader                     ← flat water colour
│
├── Art/
│   ├── Tiles/                                ← Tile textures
│   ├── Sprites/                              ← Entity thumbnails for palette UI
│   └── Prefabs/                              ← Reusable prefabs
│
└── Tests/EditMode/                           ← EditMode tests (asmdef: EditModeTests)
```

> `Assets/Scripts/Editor/` exists for dev tools (placeholder asset generators) and is excluded from builds automatically.

## SpacetimeDB Integration

The editor uses the same SpacetimeDB C# SDK as the client, but with a wider, **unfiltered** subscription set since designers may inspect any zone:

```text
player, zone, entity_instance, terrain_chunk, portal,
item_def, loot_table, enemy_def, enemy,
world_clock, weather_state
```

(`SpacetimeDBManager.OnConnect` in [editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs](../../editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs).)

Callbacks are registered in `OnSubscriptionApplied` for: `Zone`, `EntityInstance`, `Enemy`, `TerrainChunk`, `Portal`, `ItemDef`, `LootTable`, `EnemyDef`, `WorldClock`, `WeatherState`. Each fires through the singleton's events so panels and renderers can react.

The SpacetimeDB SDK must be added manually to `Packages/manifest.json`:

```json
"com.clockworklabs.spacetimedbsdk":
  "https://github.com/clockworklabs/com.clockworklabs.spacetimedbsdk.git"
```

## UI Architecture

All UI is built with **Unity UI Toolkit (UIElements)** — never uGUI, IMGUI, or EditorGUI (those are Editor-only and won't compile in builds).

### Implemented panels

| Panel | File(s) | Position | Purpose |
|-------|---------|----------|---------|
| Toolbar | `ToolbarController.*` | Top | Toggle other panels |
| Zone Manager | `ZoneCreationPanel.*` | Top-left | Create/select zones, MoodPreset dropdown |
| Brush Panel | `TilePalettePanel.*` | Top-right | Brush type, mode, layer, radius, strength |
| Entity Palette | `EntityPalettePanel.*` | Bottom-right | NPC/Enemy/Prop palette; click to select |
| Enemy Def | `EnemyDefCreationPanel.cs` | Modal | Define enemy archetypes |
| Loot | `LootCreationPanel.cs` | Modal | Define item templates + per-enemy loot tables |
| World Graph | `WorldGraphPanel.*` | Full-screen | Portal node canvas + drag-to-create edges |
| Weather Debug | `WeatherDebugPanel.cs` | Admin only | WeatherKind buttons + intensity slider |

**Mutual exclusion:** selecting a terrain brush clears the entity selection and vice versa. The Zone Manager auto-collapses when a zone is activated; chevron expands/collapses manually.

### UI Toolkit gotchas (carried over from session memory)

- Every UIDocument with a full-screen transparent root must set `root.pickingMode = PickingMode.Ignore` in `OnEnable`, or the document silently swallows pointer events from documents rendered beneath it. Also set `Ignore` on any full-screen child wrapper (e.g. `.panel-wrapper`).
- UIDocument **Sort Order** (Inspector) decides picking priority across documents; higher wins regardless of `PickingMode`. `ZoneCreationPanel = 100`, background overlays = 10.
- USS stylesheets do not auto-resolve from `<Style src="…">` in runtime-built UXML — use `[SerializeField] StyleSheet _styleSheet` and assign in the Inspector, then `root.styleSheets.Add(_styleSheet)`.
- Any new folder under `Assets/Scripts/` needs its own asmdef + a reference from `ZoneForgeRuntime.asmdef`. The editor uses asmdefs (the client does not).

## Terrain System

Terrain is **procedurally generated from height and splat data** stored per chunk in the `TerrainChunk` table. There are no individual tile GameObjects for ground terrain.

**Chunk layout:** A zone of `W × H` units is divided into `32 × 32`-unit chunks (`CHUNK_SIZE` on the server). Each chunk has one `TerrainChunk` row containing `height_data` (1024 × f32 LE = 4096 B) and `splat_data` (1024 × RGBA u8 = 4096 B).

### Terrain rendering flow

1. `TerrainRenderer` subscribes to `TerrainChunk.OnInsert/OnUpdate`
2. On callback, decodes `height_data` into Mesh vertex Y positions
3. Decodes `splat_data` into per-vertex UV2 (used by `TerrainSplatmap.shader` to blend Grass/Dirt/Stone/Ravine)
4. Uploads Mesh to GPU and updates the `MeshCollider`

### Terrain painting flow

1. Designer left-clicks / drags in the Game view
2. `TerrainPainter` raycasts the `MeshCollider` (with `Y = 0` plane fallback)
3. `TilePalettePanel` supplies the active `TerrainBrush` (type, mode, layer, radius, strength)
4. Brush computes affected samples and updates local `TerrainChunkData` copies
5. `update_terrain_chunk(zone_id, cx, cz, height_data, splat_data)` is called for each modified chunk
6. Server validates 4096-byte arrays + bounds and updates the `TerrainChunk` row
7. All subscribers (editor + game client) receive `OnUpdate` and rebuild their meshes

**Water:** `WaterRenderer` draws a flat quad Mesh at `zone.water_level`. Any terrain vertex below that level appears submerged.

## Atmosphere

`AtmosphereController` subscribes (via `SpacetimeDBManager`) to:

- `WorldClock` — drives time-of-day curves (`MoodPreset.sun/ambient/fog` keyframes)
- `WeatherState` (filtered to active zone) — spawns the matching weather VFX prefab from `Resources/WeatherVFX/`
- `Zone.mood_preset_id` — switches `MoodPreset` ScriptableObject

`AmbientAudioMixer` crossfades a 3-layer mix (`Base`, `Weather`, `Time`) through the `AudioMixer` asset's `Master → Ambient → {Base, Weather, Time}` groups.

The `WeatherDebugPanel` calls `change_weather` and `set_zone_mood` reducers (admin-gated server-side) to test atmosphere changes live.

## Key Unity Packages

| Package | Purpose |
|---------|---------|
| Universal Render Pipeline (URP) | Core 3D rendering pipeline |
| UI Toolkit | Runtime world-building UI panels |
| TextMeshPro | UI text |
| SpacetimeDB Unity SDK | Real-time backend integration |

### Assembly Definitions

```text
ZoneForgeAutogen     ← references: SpacetimeDB SDK
ZoneForgeData        ← references: (none)
ZoneForgeRuntime     ← references: SpacetimeDB SDK, ZoneForgeAutogen, ZoneForgeData
EditModeTests        ← references: ZoneForgeRuntime, NUnit
```

## See Also

- [Overview.md](Overview.md) — System architecture and data flow
- [Client.md](Client.md) — Game client architecture (for comparison)
- [Diagrams.md](Diagrams.md) — Schema + sequence diagrams
- [../guides/Editor_Dev.md](../guides/Editor_Dev.md) — Daily editor development workflow
- [../guides/Getting_Started.md](../guides/Getting_Started.md) — Full environment setup
