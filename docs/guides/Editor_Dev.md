# ZoneForge — World Editor Development Guide

> This guide covers the **standalone world editor** (`editor/` submodule). For game client development see [Client_Dev.md](Client_Dev.md).

## What the Editor Is

`zoneforge-editor` is a standalone desktop application (not a Unity Editor tool) for building ZoneForge worlds. Designers use it to create zones, paint the 3D tile grid, and place entities. Changes are persisted to SpacetimeDB and are immediately visible to all connected game clients.

## Prerequisites

- Unity 2022.3 LTS installed via Unity Hub
- VS Code with C# and Unity extensions
- SpacetimeDB local server running (`spacetime start`)
- See [Getting_Started.md](Getting_Started.md) for full setup

## Daily Workflow

```bash
# 1. Start the database server (keep this terminal open)
spacetime start

# 2. Open the Unity project
# Unity Hub → Projects → ZoneForge Editor (editor/)
```

In a second terminal when server code has changed:

```bash
cd server && spacetime build && spacetime publish --server local zoneforge-server
# Regenerate bindings if schema changed:
cd ../editor
spacetime generate --lang csharp --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

## Architecture Rules

**No Editor-only APIs.** The editor is a standalone runtime app that must compile to a desktop build. These APIs are forbidden:
- `EditorWindow` — use `MonoBehaviour` + UIToolkit instead
- `AssetDatabase` — assets are baked into the build
- `EditorSceneManager` — use `SceneManager` at runtime
- `[MenuItem]` — use UI buttons instead

**UI stack:** Unity UI Toolkit (UIElements) for panels and toolbars, uGUI for HUD elements. Do not use IMGUI / EditorGUI.

**Terrain system:** Ground terrain is procedurally generated from height and splat data — there are no individual tile GameObjects for terrain. Each zone is divided into chunks; each chunk has a `TerrainChunk` row on the server containing `height_data` and `splat_data` byte arrays. `TerrainRenderer` builds a Mesh from this data and the `TerrainSplatmap` shader blends texture layers.

**Terrain painting pattern:**
```csharp
// TerrainPainter raycasts against the MeshCollider on the Terrain GameObject.
// TilePalettePanel provides the active TerrainBrush (type, radius, strength).
// On hit, the brush updates local TerrainChunkData, then calls:
Reducers.PaintTerrain(zoneId, chunkX, chunkZ, heightData, splatData);
// Server updates TerrainChunk; all subscribers (editor + client) rebuild their Meshes.
```

**Entity placement pattern:**

```csharp
// EntityPlacer raycasts against the MeshCollider on the Terrain GameObject.
// EntityPalettePanel provides the selected EntityDefinition (prefab name, type, colour).
// On click, EntityPlacer calls:
Reducers.SpawnEntity(zoneId, prefabName, hit.point.x, hit.point.z, hit.point.y, entityType);
// Server inserts an EntityInstance row; EntityRenderer spawns a placeholder cube on insert.
// EntityPalettePanel.ClearSelection() is called when a terrain brush is activated (mutual exclusion).
```

## Regenerating Editor Bindings

Run after any server schema change:

```bash
# From editor/ directory
spacetime generate --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

Then in Unity: **Assets → Reimport All**.

Never edit files in `Assets/Scripts/autogen/`.

## SpacetimeDB Patterns

The SpacetimeDB C# SDK patterns are identical to the game client. Use the same `DbConnection.Builder()` → `OnConnect` → `OnSubscriptionApplied` → `FrameTick()` ordering.

See [Client_Dev.md](Client_Dev.md) or the `unity-spacetimedb-subscribe` Claude skill for the full pattern.

## Building a Standalone Editor

```text
File → Build Settings → Platform: Linux (or Windows/macOS)
→ Build
```

The build excludes all `Assets/Scripts/Editor/` scripts automatically — those are dev tools only (placeholder generators, etc.) that don't need to ship.

## Common Issues

| Problem | Fix |
|---------|-----|
| Bindings not updating after schema change | Run `spacetime generate` from `editor/`, then Reimport All |
| Tile placement raycasting misses | Ensure the grid plane Y value matches your world grid origin |
| UI not visible in build | Check UI Toolkit UXML asset is included in build; verify no EditorGUI calls |

## Claude Code Skills

When working in `editor/` with Claude Code:

- **`unity-spacetimedb-subscribe`** — correct subscription ordering, callback registration, and `FrameTick` usage (same patterns as client)
- **`unity-autogen-refresh`** — regenerating C# bindings after server schema changes

Editor-specific skills for zone creation and tile placement patterns are planned for Phase 2.

See [Claude_Skills.md](Claude_Skills.md) for the full skills reference.

## See Also

- [Client_Dev.md](Client_Dev.md) — Game client development workflow
- [../architecture/Editor.md](../architecture/Editor.md) — Editor architecture detail
- [Server_Dev.md](Server_Dev.md) — Server-side development workflow
- [Getting_Started.md](Getting_Started.md) — Full environment setup
