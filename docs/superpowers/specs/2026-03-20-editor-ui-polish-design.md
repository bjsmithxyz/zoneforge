# Editor UI Polish & Bug Fixes — Design Spec

**Date:** 2026-03-20
**Phase:** 2, Group 5 (post-merge polish)
**Status:** Approved

---

## Overview

Addresses usability issues and functional bugs across all three editor UI panels (Zone Manager, Brush Panel, Entity Palette). Includes two root-cause bugs that prevent painting and entity placement from working at all.

---

## Bug Fixes

### Bug 1 — Zone list empty on startup

**Root cause:** In `SpacetimeDBManager.OnSubscriptionApplied`, the SpacetimeDB SDK delivers initial `OnInsert` events for existing rows *during* the subscription application process — before `OnSubscriptionApplied` returns and therefore before `Conn.Db.Zone.OnInsert` is registered. By the time `OnConnected` fires (at the end of `OnSubscriptionApplied`), the zone cache in `Conn.Db.Zone` is fully populated, but no handler has seen the insert events. Result: the zone list is permanently empty for pre-existing zones; the user cannot select a zone; `EditorState.HasActiveZone` stays false; painting is blocked.

**Fix:** In `ZoneCreationPanel.OnConnected`, after enabling the create button, iterate `SpacetimeDBManager.Conn.Db.Zone.Iter()` and call `AddZoneRow` for each result, skipping any zone already in `_zoneRows` to guard against duplicate rows if the component is disabled and re-enabled:

```csharp
foreach (var zone in SpacetimeDBManager.Conn.Db.Zone.Iter())
{
    if (!_zoneRows.ContainsKey(zone.Id))
        AddZoneRow(zone);
}
```

New zones created after startup continue to arrive via `OnZoneInserted` as before.

### Bug 2 — Entity placement fails silently

**Two sub-causes:**

1. `EntityPlacer._terrainCollider` is a `[SerializeField]` requiring manual Inspector wiring. If null, `Update` exits at the early-return null guard — the fallback code described below is never reached.
2. `TerrainRenderer.BuildFullMesh()` never updates `MeshCollider.sharedMesh`. Even when the collider is properly wired, it raycasts against an empty/stale mesh and always misses.

**Fix — `TerrainRenderer`:** Add `[RequireComponent(typeof(MeshCollider))]` to the class attribute list so Unity enforces the component's presence at component-add time. Add `private MeshCollider _collider` field; assign in `Awake` via `GetComponent<MeshCollider>()`; call `if (_collider != null) _collider.sharedMesh = _mesh;` at the end of `BuildFullMesh()`. **Scene setup note:** `[RequireComponent]` does not retroactively add the component to existing GameObjects. The `MeshCollider` must be manually added to the `TerrainRenderer` GameObject in the scene before entering Play mode.

**Fix — `EntityPlacer`:** Restructure `Update()` so the `_terrainCollider` null check is part of a conditional branch, not a hard early return. The Y=0 plane fallback (same logic as `TerrainPainter.TryGetTerrainHit`) executes when the collider is null or the raycast misses:

```csharp
var ray = _camera.ScreenPointToRay(Input.mousePosition);
Vector3 hitPoint;

bool colliderHit = _terrainCollider != null &&
                   _terrainCollider.Raycast(ray, out RaycastHit hit, 1000f);

if (colliderHit)
{
    hitPoint = hit.point;
}
else
{
    // Y=0 plane fallback
    if (Mathf.Abs(ray.direction.y) < 0.0001f) return;
    float t = -ray.origin.y / ray.direction.y;
    if (t <= 0f) return;
    hitPoint = ray.origin + t * ray.direction;
}
```

The existing per-condition early-return guards (`HasActiveZone`, `SelectedEntry`, `UIHoverTracker`, `GetMouseButtonDown`, `Conn`, `_camera`) remain before this block — including the `_camera == null` guard that must precede the `ScreenPointToRay` call.

---

## Zone Manager Panel

### Collapsible header (auto-collapse on zone activate)

Add a header row (title + chevron button) to `ZoneCreationPanel`. Content below the header moves into a `panel-body` container. Chevron is `▾` when expanded, `▸` when collapsed.

**C# changes:** Add `_panelBody` (`VisualElement`), `_collapseBtn` (`Button`), and `_isCollapsed` (`bool`) fields. Wire `_collapseBtn.clicked += ToggleCollapse` in `OnEnable`. Collapse logic is added inside the **existing** `OnActiveZoneChanged(ulong zoneId)` callback (no new subscription):

```csharp
private bool _isCollapsed = false;

void OnActiveZoneChanged(ulong zoneId)
{
    // existing highlight logic ...

    // Auto-collapse when a zone is active; expand when none is selected
    SetCollapsed(zoneId != 0);
}

void SetCollapsed(bool collapsed)
{
    _isCollapsed = collapsed;
    _panelBody.style.display = collapsed ? DisplayStyle.None : DisplayStyle.Flex;
    _collapseBtn.text = collapsed ? "▸" : "▾";
}

void ToggleCollapse() => SetCollapsed(!_isCollapsed);
```

**Behaviour:**

- Zone activated → panel auto-collapses to header only
- Zone deactivated (e.g., zone deleted) → panel auto-expands
- User can manually expand/collapse at any time via the chevron

**UXML change for `ZoneCreationPanel.uxml`:** The existing panel content (`.section` divs) moves into `panel-body`. Add a `panel-header` above it. The `panel-title` and `status-label` labels move into the header or remain in the body — keep `panel-title` in the header and `status-label` at the top of `panel-body` so the zone count is still visible when expanded:

```xml
<ui:VisualElement name="panel" class="panel">
    <ui:VisualElement name="panel-header" class="panel-header">
        <ui:Label text="Zone Manager" class="panel-title"/>
        <ui:Button name="collapse-btn" text="▾" class="collapse-btn"/>
    </ui:VisualElement>
    <ui:VisualElement name="panel-body">
        <ui:Label name="status-label" text="Connecting..." class="status-label"/>
        <!-- existing .section divs remain here unchanged -->
    </ui:VisualElement>
</ui:VisualElement>
```

USS additions for `ZoneCreationPanel.uss` mirror the Brush Panel: `.panel-header`, `.panel-title`, `.collapse-btn` (same rules).

### Text field clipping fix

`.form-field` and `.form-field-small` use `height: 26px`, which clips text in Unity UIToolkit's input element at runtime.

**Fix:** Change both to `height: auto; min-height: 24px`. The inner `TextField`/`IntegerField` elements size naturally without clipping.

### Cursor warning fix

`cursor: link` in `ZoneCreationPanel.uss` is unsupported in UIToolkit runtime and generates "Runtime cursors other than the default cursor" warnings on every frame.

**Fix:** Change `.zone-row { cursor: link }` to `cursor: arrow`.

---

## Brush Panel

### Collapsible header (manual toggle)

Add a header row to `TilePalettePanel` with a "Brush Panel" title and ▾/▸ chevron toggle button. Existing content moves into a `panel-body` container beneath the header.

**UXML change:** Wrap existing content in `<VisualElement name="panel-body">`. Add `<VisualElement name="panel-header">` above it, containing `<Label text="Brush Panel" class="panel-title"/>` and `<Button name="collapse-btn" text="▾" class="collapse-btn"/>`.

**C# changes in `TilePalettePanel`:** Add fields `_panelBody` (`VisualElement`) and `_collapseBtn` (`Button`). In `OnEnable`:

```csharp
_panelBody   = root.Q<VisualElement>("panel-body");
_collapseBtn = root.Q<Button>("collapse-btn");
_collapseBtn.clicked += ToggleCollapse;
```

Add method:

```csharp
private bool _isCollapsed = false;

void ToggleCollapse()
{
    _isCollapsed = !_isCollapsed;
    _panelBody.style.display = _isCollapsed ? DisplayStyle.None : DisplayStyle.Flex;
    _collapseBtn.text = _isCollapsed ? "▸" : "▾";
}
```

`_isCollapsed` is not persisted across sessions. Panel width (260px) is unchanged when collapsed.

**USS additions:**

```css
.panel-header {
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 4px;
}
.panel-title {
    font-size: 13px;
    color: rgb(220, 200, 140);
    -unity-font-style: bold;
}
.collapse-btn {
    width: 20px;
    height: 20px;
    padding: 0;
    background-color: rgba(0,0,0,0);
    border-width: 0;
    color: rgb(180, 180, 180);
    font-size: 12px;
    -unity-text-align: middle-center;
}
```

### Radio button text colour

Unity UIToolkit's `RadioButton` (which extends `Toggle`) defaults to black label text in runtime panels. The dark panel background makes this unreadable.

**Fix:** Add to `TilePalettePanel.uss`:

```css
.unity-toggle__label {
    color: rgb(200, 200, 200);
}
.unity-toggle--checked > .unity-toggle__label {
    color: rgb(140, 200, 255);
}
```

### Remove unsupported `:first-child` pseudo-class

`TilePalettePanel.uss` contains `.section-label:first-child { margin-top: 0; border-top-width: 0; padding-top: 0; }` which is unsupported and logs a console warning on stylesheet load. Simply removing the rule would visually regress the first "Brush Type" label (it would gain the `margin-top: 10px` and top border that `.section-label` applies unconditionally).

**Fix:** Remove the `:first-child` rule. In `TilePalettePanel.OnEnable`, after building the element tree, assign a `first-section` USS class to the first `section-label` element (or add the class directly in UXML on the "Brush Type" label):

```css
.section-label.first-section {
    margin-top: 0;
    border-top-width: 0;
    padding-top: 0;
}
```

In UXML: `<ui:Label text="Brush Type" class="section-label first-section"/>`.

---

## Entity Palette Panel

### Reposition to avoid brush panel overlap

On shorter windows the entity panel (`bottom: 10px; right: 10px`) can overlap with the brush panel (`top: 16px; right: 16px; width: 260px`). To eliminate the risk permanently:

**Fix:** Change to `right: 286px` (260px brush width + 16px brush right margin + 10px gap). Entity panel sits directly to the left of the brush panel regardless of window height.

Panel width increases from 148px → 180px to accommodate entity row labels legibly.

### Redesign: grouped list with Foldouts

Replace the 2-column 48×48 grid with a vertical grouped list. Entities are grouped by `EntityType` using UIToolkit `Foldout` elements.

**Groups (in order):** NPC, Enemy, Prop

Each `Foldout` starts expanded — set `foldout.value = true` explicitly when creating each one, as the default varies by Unity version. Users can collapse individual groups. The overall panel remains scrollable (`max-height: 320px`).

**Row layout (replaces 48×48 cell):**

- 14×14 colour swatch (solid background colour, `border-radius: 2px`)
- Entity display name label (11px). Uses `overflow: hidden; white-space: nowrap` — text clips silently; UIToolkit 2022 runtime does not support `text-overflow: ellipsis`, so there is no ellipsis character.

**Selection state:** Left 2px accent border + subtle background tint (replaces the current outer white border).

**C# changes in `EntityPalettePanel`:**

- `BuildGrid()`: group `Definitions` by `EntityType`, create a `Foldout` per group with `foldout.value = true` and `foldout.AddToClassList("entity-group")` (required for the `.entity-group` USS selector to match), add `VisualElement` rows per entity.
- `SelectEntry()`: track `_selectedRow` (rename from `_selectedCell`); toggle `.entity-row--selected` class.
- `ClearSelection()`: update `RemoveFromClassList` call to use `entity-row--selected`; rename internal `_selectedCell` reference to `_selectedRow`.
- `OnActiveZoneChanged()`: same — update class name and field name.

**USS changes:** Remove `.entity-cell`, `.entity-cell__image`, `.entity-cell--selected`, `.entity-grid` (grid is column-direction anyway after redesign). Add:

```css
.entity-grid {
    flex-direction: column;
}
.entity-group > .unity-foldout__toggle > .unity-toggle__label {
    color: rgb(180, 180, 180);
    font-size: 11px;
    -unity-font-style: bold;
}
.entity-row {
    flex-direction: row;
    align-items: center;
    padding: 3px 4px;
    border-radius: 2px;
    border-left-width: 2px;
    border-left-color: rgba(0,0,0,0);
    cursor: arrow;
}
.entity-row:hover {
    background-color: rgba(255, 255, 255, 0.08);
}
.entity-row--selected {
    background-color: rgba(255, 255, 255, 0.12);
    border-left-color: rgb(255, 255, 255);
}
.entity-swatch {
    width: 14px;
    height: 14px;
    border-radius: 2px;
    margin-right: 6px;
    flex-shrink: 0;
}
.entity-row__label {
    font-size: 11px;
    color: rgb(200, 200, 200);
    overflow: hidden;
    white-space: nowrap;
}
```

**UXML change:** `EntityPalettePanel.uxml` retains `entity-grid` as the container; the grid is populated entirely in C#. The `entity-scroll` max-height increases to 320px.

---

## File Summary

**Modified files:**

| File | Change |
| --- | --- |
| `editor/Assets/Scripts/Runtime/ZoneCreationPanel.cs` | Populate existing zones in `OnConnected`; add `SetCollapsed`/`ToggleCollapse`; collapse in existing `OnActiveZoneChanged` |
| `editor/Assets/Scripts/Runtime/TilePalettePanel.cs` | Add `_panelBody`, `_collapseBtn` fields; wire `ToggleCollapse`; add `first-section` class to first label in UXML or OnEnable |
| `editor/Assets/Scripts/Runtime/EntityPalettePanel.cs` | Rebuild `BuildGrid()` with Foldout grouping; update `SelectEntry()`, `ClearSelection()`, `OnActiveZoneChanged()` to use `_selectedRow` / `entity-row--selected` |
| `editor/Assets/Scripts/Runtime/EntityPlacer.cs` | Restructure `Update()` to use conditional branch for collider + Y=0 plane fallback |
| `editor/Assets/Scripts/Runtime/TerrainRenderer.cs` | Add `_collider` field; update `MeshCollider.sharedMesh` in `BuildFullMesh()` |
| `editor/Assets/UI/ZoneCreationPanel.uxml` | Wrap content in `panel-header` + `panel-body` structure |
| `editor/Assets/UI/ZoneCreationPanel.uss` | Fix field heights (`height: auto; min-height: 24px`); fix cursor (`arrow`); add `.panel-header`, `.panel-title`, `.collapse-btn` styles |
| `editor/Assets/UI/TilePalettePanel.uxml` | Wrap content in `panel-header` + `panel-body`; add `first-section` class to first label |
| `editor/Assets/UI/TilePalettePanel.uss` | Add `.unity-toggle__label` colour rules; remove `:first-child` rule; add `.first-section` rule; add header/collapse-btn styles |
| `editor/Assets/UI/EntityPalettePanel.uxml` | Increase `entity-scroll` `max-height` to 320px; grid content populated entirely in C# |
| `editor/Assets/UI/EntityPalettePanel.uss` | Replace cell styles with row/swatch/group styles; update position (`right: 286px`) and width (180px) |

**No server changes required.**

---

## Out of Scope

- Default texture on zone create (requires `create_zone` reducer change — separate ticket)
- Entity search / filter (future work)
