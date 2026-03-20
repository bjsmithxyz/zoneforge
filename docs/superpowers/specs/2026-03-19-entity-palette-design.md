# Entity Palette Panel — Design Spec

**Date:** 2026-03-19
**Phase:** 2, Group 5
**Status:** Approved

---

## Overview

This spec covers the entity palette panel and click-to-place workflow for the ZoneForge standalone editor. It completes the remaining Group 5 tasks:

- Entity palette panel with colored thumbnail grid
- Click-to-place entity placement (calls `spawn_entity` reducer)
- Subscribe to `EntityInstance` table (`OnInsert` callback)
- Real-time entity spawning visible in both editor and game client

No server changes are required. The `EntityInstance` table and `spawn_entity` reducer already exist.

---

## Naming Convention

`prefab_name` stores only the bare asset name (e.g., `guard`, `tree`). The `entity_type` field (e.g., `NPC`, `Prop`) already provides the category, making type-prefixes in `prefab_name` redundant. This keeps names short, asset-path-friendly, and not coupled to the category taxonomy.

**Rule:** `prefab_name` = bare Unity prefab asset name, no category prefix. `entity_type` = category string used for game logic and filtering.

See also: [ADR-004 Entity Prefab Naming](../decisions/004-entity-prefab-naming.md)

---

## Architecture

Four new components plus one plain data class are added to the editor. No server changes.

`SpacetimeDBManager` gets one addition: an `OnEntityDeleted` event (see below).

| Component | Type | File | Responsibility |
| --- | --- | --- | --- |
| `EntityDefinition` | Plain C# class | `Runtime/EntityDefinition.cs` | Data record: `DisplayName`, `PrefabName`, `EntityType`, `Color` |
| `EntityPalettePanel` | MonoBehaviour + UIToolkit | `Runtime/EntityPalettePanel.cs` + `UI/EntityPalettePanel.uxml/uss` | Renders palette grid; sets `EntityPlacer.SelectedEntry` on click; clears `TerrainPainter.ActiveBrush` on selection |
| `EntityPlacer` | MonoBehaviour | `Runtime/EntityPlacer.cs` | Handles left-click → MeshCollider raycast → `SpawnEntity` reducer call. Exposes `public static EntityDefinition SelectedEntry` (mirrors `TerrainPainter.ActiveBrush`) |
| `EntityRenderer` | MonoBehaviour | `Runtime/EntityRenderer.cs` | Subscribes to `OnEntityInserted/OnEntityDeleted`; spawns/destroys placeholder cubes |

---

## Entity Definitions (Hardcoded)

`EntityDefinition` is the single source of truth. `EntityPalettePanel` owns the static list and exposes a `public static Dictionary<string, EntityDefinition> ByPrefabName` for other components (e.g., `EntityRenderer`) to look up color by `prefabName`. No duplication of definitions elsewhere.

| Display Name | `PrefabName` | `EntityType` | Color |
| --- | --- | --- | --- |
| Guard | `guard` | NPC | Blue |
| Merchant | `merchant` | NPC | Cyan |
| Skeleton | `skeleton` | Enemy | Red |
| Goblin | `goblin` | Enemy | Dark red (`#8B0000`) |
| Tree | `tree` | Prop | Green |
| Rock | `rock` | Prop | Gray |
| Chest | `chest` | Prop | Yellow |
| Wall | `wall` | Prop | Brown (`#8B4513`) |

---

## UI Layout

- **Position:** Bottom-right, anchored via USS `position: absolute; bottom: 10px; right: 10px`
- **Title:** "Entities"
- **Grid:** 2-column scrollable grid of palette cells
- **Cell size:** 48×48px colored square (programmatically generated `Texture2D`) + label below
- **Selection state:** Selected cell gets a white border highlight via USS class `entity-cell--selected`
- **Disabled state:** When `EditorState.HasActiveZone == false`, call `panelRoot.SetEnabled(false)` — this both grays the panel visually and blocks all pointer events via UIToolkit's built-in disabled handling. Do **not** rely on `opacity` alone (it does not block pointer events).

USS stylesheet assigned via `[SerializeField] StyleSheet _styleSheet` in the MonoBehaviour (same pattern as `TilePalettePanel` and `ZoneCreationPanel`).

Panel registers with `UIHoverTracker` so entity placement is suppressed when the cursor is over it.

---

## Mutual Exclusion: Terrain Brush vs Entity Placement

`TerrainPainter` paints whenever `ActiveBrush != null`. `EntityPlacer` places whenever `SelectedEntry != null`. Both check `Input.GetMouseButton(0)` / `Input.GetMouseButtonDown(0)` — if both are set, a single click fires both on the same frame.

**Rule:** The tools are mutually exclusive.

- When `EntityPalettePanel` selects an entry: set `TerrainPainter.ActiveBrush = null`
- When `TilePalettePanel` sets a brush: set `EntityPlacer.SelectedEntry = null`

`TilePalettePanel.UpdateBrush()` must call `EntityPlacer.SelectedEntry = null` after setting `TerrainPainter.ActiveBrush`. `EntityPalettePanel` must call `TerrainPainter.ActiveBrush = null` after setting `EntityPlacer.SelectedEntry`.

On startup, `TilePalettePanel.UpdateBrush()` runs first and sets an active brush — so entity placement starts inactive by default.

---

## Click-to-Place (`EntityPlacer`)

**Preconditions** — all must be true for a click to register:

1. `EditorState.HasActiveZone`
2. `EntityPlacer.SelectedEntry != null`
3. `!UIHoverTracker.IsPointerOverUI`
4. `Input.GetMouseButtonDown(0)` (single-click stamp, not hold)

**Raycast:** `EntityPlacer` holds a `[SerializeField] MeshCollider _terrainCollider` assigned in the Inspector to the same MeshCollider that `TerrainPainter` uses (the one on the `TerrainRenderer` GameObject). It calls `_terrainCollider.Raycast(ray, out hit, 1000f)` directly — matching the pattern in `TerrainPainter.TryGetTerrainHit()`. No `Physics.Raycast` with layer masks; no new Physics layers required. If no hit, click is silently ignored.

**Coordinate mapping** — matches existing XZ convention:

- `position_x` = `hit.point.x`
- `position_y` = `hit.point.z` (world Z maps to the `y` reducer parameter — the reducer's `y` arg is the horizontal depth axis, not vertical)
- `elevation` = `hit.point.y`

**Reducer call:**

```csharp
SpacetimeDBManager.Conn.Reducers.SpawnEntity(
    EditorState.ActiveZoneId,
    entry.PrefabName,
    hit.point.x,     // x
    hit.point.z,     // y (reducer param name — this is world Z)
    hit.point.y,     // elevation (world Y)
    entry.EntityType
);
```

---

## Entity Rendering (`EntityRenderer`)

Placeholder cubes represent placed entities in the editor scene until real prefabs exist.

**Event subscriptions:** Follow the `OnEnable`/`OnDisable` pattern used by `ZoneCreationPanel` and `TilePalettePanel` — subscribe in `OnEnable`, unsubscribe in `OnDisable`.

**On `SpacetimeDBManager.OnEntityInserted`:**

- If `entity.ZoneId != EditorState.ActiveZoneId`, ignore
- Instantiate a 0.5×0.5×0.5 unit cube at `(entity.PositionX, entity.Elevation, entity.PositionY)`
- Look up color via `EntityPalettePanel.ByPrefabName[entity.PrefabName].Color`; fall back to `Color.white` if not found (log a warning)
- Create a new `Material` for the cube (`new Material(Shader.Find("Universal Render Pipeline/Unlit")) { color = ... }`) to avoid shared material mutation
- Store the cube in `Dictionary<ulong, GameObject>` keyed by `entity.Id`

**On `EditorState.OnActiveZoneChanged`:**

- Destroy all existing placeholder cubes and clear the dictionary
- Guard: if `SpacetimeDBManager.Conn == null`, return (mirrors the pattern in `TerrainPainter.Update()`)
- Re-populate from `SpacetimeDBManager.Conn.Db.EntityInstance.Iter()` filtered to the new `zoneId`, using the same spawn logic as `OnEntityInserted`

**On `SpacetimeDBManager.OnEntityDeleted`:**

- Look up cube by `entity.Id`, call `Destroy(go)`, remove from dictionary

**No separate physics layer needed:** `EntityPlacer` raycasts directly against the terrain `MeshCollider` component, so placeholder cubes are never hit by placement raycasts regardless of their layer.

---

## `SpacetimeDBManager` Change

One addition: expose `OnEntityDeleted` to mirror `OnEntityInserted`.

```csharp
public static event Action<EntityInstance> OnEntityDeleted;
// In OnSubscriptionApplied:
Conn.Db.EntityInstance.OnDelete += (eventCtx, entity) => OnEntityDeleted?.Invoke(entity);
```

---

## `TilePalettePanel` Change

`UpdateBrush()` must clear `EntityPlacer.SelectedEntry` after setting the active brush, to enforce mutual exclusion:

```csharp
TerrainPainter.ActiveBrush = brush;
EntityPlacer.SelectedEntry = null;
```

---

## Scene Wiring

Three new GameObjects in the editor scene:

| GameObject | Components | Notes |
| --- | --- | --- |
| `EntityPalettePanel` | `UIDocument`, `EntityPalettePanel` | Assign `_styleSheet` in Inspector. UIDocument Sort Order = 2. |
| `EntityPlacer` | `EntityPlacer` | Assign `_terrainCollider` in Inspector — drag the `MeshCollider` from the `TerrainRenderer` GameObject. |
| `EntityRenderer` | `EntityRenderer` | No special Inspector setup required. |

---

## What Is Not In Scope

- Real prefab assets (placeholder cubes only for now)
- Entity deletion from the editor UI
- Entity selection / move / inspect after placement
- Client-side rendering of entities in `zoneforge-client` (subscribed but no renderer — future task)

---

## File Summary

**New files (editor):**

| File | Purpose |
| --- | --- |
| `editor/Assets/Scripts/Runtime/EntityDefinition.cs` | Plain C# data class |
| `editor/Assets/Scripts/Runtime/EntityPalettePanel.cs` | UIToolkit panel MonoBehaviour |
| `editor/Assets/Scripts/Runtime/EntityPlacer.cs` | Click-to-place MonoBehaviour |
| `editor/Assets/Scripts/Runtime/EntityRenderer.cs` | Placeholder cube spawner |
| `editor/Assets/UI/EntityPalettePanel.uxml` | Panel layout |
| `editor/Assets/UI/EntityPalettePanel.uss` | Panel styles |

**Modified files:**

| File | Change |
| --- | --- |
| `editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs` | Add `OnEntityDeleted` event + callback |
| `editor/Assets/Scripts/Runtime/TilePalettePanel.cs` | Clear `EntityPlacer.SelectedEntry` in `UpdateBrush()` |

**New docs:**

| File | Purpose |
| --- | --- |
| `docs/decisions/004-entity-prefab-naming.md` | ADR: bare prefab names, no category prefix |
