# Editor Polish — Brush Fix, Camera Controller, Entity Panel Redesign

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix broken terrain brushes and entity placement, add camera pan/zoom controls, and redesign the entity palette panel with search, proper positioning, and toolbar integration.

**Architecture:** Five independent tasks — two CSS fixes, one diagnostic+fix pass, one new MonoBehaviour, and a two-part panel redesign. All changes are confined to the `editor/` submodule.

**Tech Stack:** Unity 2022.3 LTS, C#, UI Toolkit (UIElements), USS/UXML

---

## Task 1: Fix Input Field Text Visibility

**Files:**
- Modify: `editor/Assets/UI/ZoneCreationPanel.uss`
- Modify: `editor/Assets/UI/EntityPalettePanel.uss`

The USS currently sets near-white text colour on input fields but does not override Unity's default light background, making text unreadable. The fix adds a dark background.

- [ ] **Step 1: Add dark background to input selectors in ZoneCreationPanel.uss**

Open `editor/Assets/UI/ZoneCreationPanel.uss`. Find the block:
```css
/* Unity built-in control text overrides */
.unity-base-field__label { color: rgb(180, 180, 180); }
.unity-text-field__input { color: rgb(220, 220, 220); }
.unity-integer-field__input { color: rgb(220, 220, 220); }
.unity-float-field__input { color: rgb(220, 220, 220); }
```

Replace with:
```css
/* Unity built-in control text overrides */
.unity-base-field__label { color: rgb(180, 180, 180); }
.unity-text-field__input    { color: rgb(220, 220, 220); background-color: rgba(30, 30, 40, 1); }
.unity-integer-field__input { color: rgb(220, 220, 220); background-color: rgba(30, 30, 40, 1); }
.unity-float-field__input   { color: rgb(220, 220, 220); background-color: rgba(30, 30, 40, 1); }
```

- [ ] **Step 2: Fix foldout header text colour in EntityPalettePanel.uss**

Open `editor/Assets/UI/EntityPalettePanel.uss`. Find:
```css
.entity-group > .unity-foldout__toggle > .unity-toggle__label {
    color: rgb(180, 180, 180);
    font-size: 11px;
    -unity-font-style: bold;
}
```

Replace with:
```css
.entity-group > .unity-foldout__toggle > .unity-toggle__label {
    color: rgb(200, 200, 210);
    font-size: 11px;
    -unity-font-style: bold;
}
```

- [ ] **Step 3: Verify in Play mode**

Enter Play mode. Open the Zone Manager panel. Click into the Zone Name field and type — text should now be clearly visible (near-white on dark background). The entity panel foldout headers (NPC, Enemy, Prop) should now be light-grey and readable.

- [ ] **Step 4: Commit**

```bash
cd /home/beek/zoneforge
git add editor/Assets/UI/ZoneCreationPanel.uss editor/Assets/UI/EntityPalettePanel.uss
git commit -m "fix: dark background on input fields, lighter entity foldout text"
```

---

## Task 2: Diagnose and Fix Terrain Brush & Entity Placement

**Files:**
- Modify: `editor/Assets/Scripts/Runtime/TerrainPainter.cs`
- Modify: `editor/Assets/Scripts/Runtime/EntityPlacer.cs`

Both systems gate their input on multiple conditions. The goal is to add targeted diagnostic logging, run the game to identify the failing gate, remove the diagnostics, and apply the fix.

### Phase A — Add Diagnostics

- [ ] **Step 1: Add gate diagnostics to TerrainPainter.Update()**

Open `editor/Assets/Scripts/Runtime/TerrainPainter.cs`. Replace the entire `Update()` method with:

```csharp
void Update()
{
    // --- Diagnostic gates (remove after identifying root cause) ---
    if (!EditorState.HasActiveZone || SpacetimeDBManager.Conn == null)
    {
        if (Input.GetMouseButtonDown(0))
            Debug.Log($"[TerrainPainter] BLOCKED gate1: HasActiveZone={EditorState.HasActiveZone} conn={SpacetimeDBManager.Conn != null}");
        _hoverRing.enabled = false;
        return;
    }
    if (ActiveBrush == null)
    {
        if (Input.GetMouseButtonDown(0))
            Debug.Log("[TerrainPainter] BLOCKED gate2: ActiveBrush is null");
        _hoverRing.enabled = false;
        return;
    }

    bool hasHit = TryGetTerrainHit(out Vector3 hitPoint);
    if (!hasHit)
    {
        if (Input.GetMouseButtonDown(0))
            Debug.Log($"[TerrainPainter] BLOCKED gate3: no terrain hit. " +
                      $"camera={_camera != null} collider={_meshCollider != null} " +
                      $"colliderMesh={_meshCollider?.sharedMesh != null} " +
                      $"camPos={_camera?.transform.position} " +
                      $"rayDir={_camera?.ScreenPointToRay(Input.mousePosition).direction}");
        _hoverRing.enabled = false;
        return;
    }

    UpdateHoverRing(hitPoint, ActiveBrush.Radius);

    bool overUI = UIHoverTracker.IsPointerOverUI;
    if (Input.GetMouseButtonDown(0))
        Debug.Log($"[TerrainPainter] click reached: overUI={overUI} hitPoint={hitPoint}");

    if (Input.GetMouseButton(0) && !overUI)
        ApplyBrushAt(hitPoint.x, hitPoint.z);
}
```

- [ ] **Step 2: Add gate diagnostics to EntityPlacer.Update()**

Open `editor/Assets/Scripts/Runtime/EntityPlacer.cs`. Replace the entire `Update()` method with:

```csharp
void Update()
{
    if (!EditorState.HasActiveZone)
    {
        if (Input.GetMouseButtonDown(0)) Debug.Log("[EntityPlacer] BLOCKED: no active zone");
        return;
    }
    if (SelectedEntry == null)
    {
        if (Input.GetMouseButtonDown(0)) Debug.Log("[EntityPlacer] BLOCKED: no selected entry");
        return;
    }
    bool overUI = UIHoverTracker.IsPointerOverUI;
    if (overUI)
    {
        if (Input.GetMouseButtonDown(0)) Debug.Log("[EntityPlacer] BLOCKED: over UI");
        return;
    }
    if (!Input.GetMouseButtonDown(0)) return;
    if (SpacetimeDBManager.Conn == null) { Debug.Log("[EntityPlacer] BLOCKED: no conn"); return; }
    if (_camera == null) { Debug.Log("[EntityPlacer] BLOCKED: no camera"); return; }

    Debug.Log("[EntityPlacer] attempting placement...");

    var ray = _camera.ScreenPointToRay(Input.mousePosition);
    Vector3 hitPoint;

    RaycastHit hit = default;
    bool colliderHit = _terrainCollider != null &&
                       _terrainCollider.Raycast(ray, out hit, 1000f);

    if (colliderHit)
    {
        hitPoint = hit.point;
    }
    else
    {
        if (Mathf.Abs(ray.direction.y) < 0.0001f) return;
        float t = -ray.origin.y / ray.direction.y;
        if (t <= 0f) return;
        hitPoint = ray.origin + t * ray.direction;
    }

    SpacetimeDBManager.Conn.Reducers.SpawnEntity(
        EditorState.ActiveZoneId,
        SelectedEntry.PrefabName,
        hitPoint.x,
        hitPoint.z,
        hitPoint.y,
        SelectedEntry.EntityType
    );
}
```

- [ ] **Step 3: Run and record the gate diagnostic output**

Enter Play mode. Select a zone. Select a brush from the brush panel. Click on the terrain. Open the Unity Console and note which `[TerrainPainter] BLOCKED gateN` line appears. Also select an entity and click on terrain — note which `[EntityPlacer] BLOCKED` line appears.

**Do not proceed until you have the gate output.** Record it here before moving to Phase B.

### Phase B — Apply Fix Based on Diagnostic Output

Read the gate output and apply **exactly one** of the following fixes:

---

**If gate1 fails (`HasActiveZone=false`):**
The zone selection click is not reaching `EditorState.SetActiveZone`. Verify in the Inspector that the ZoneCreationPanel component has its `UIDocument` and `StyleSheet` fields assigned. If those are set, check whether `SpacetimeDBManager.OnZoneInserted` fires by adding a `Debug.Log` inside `AddZoneRow` in `ZoneCreationPanel.cs`. The zone rows must be present and clickable for `EditorState.SetActiveZone(zoneId)` to be called.

---

**If gate2 fails (`ActiveBrush is null`):**
`TilePalettePanel.UpdateBrush()` is not setting `TerrainPainter.ActiveBrush`. Verify the TilePalettePanel component is in the scene and its `UIDocument` and `StyleSheet` fields are assigned in the Inspector. Add `Debug.Log($"[TilePalettePanel] UpdateBrush called, brush={brush}")` at the end of `UpdateBrush()` to confirm it fires when a radio button is clicked.

---

**If gate3 fails (no terrain hit) and `colliderMesh=False`:**
`BuildFullMesh` hasn't been called yet, or TerrainRenderer and TerrainPainter are on different GameObjects. Verify in the Inspector that TerrainPainter and TerrainRenderer are on the **same** GameObject. If they are, add a `Debug.Log("[TerrainRenderer] BuildFullMesh called")` at the top of `BuildFullMesh()` in `TerrainRenderer.cs` to confirm it fires after zone selection.

---

**If gate3 fails (no terrain hit) and `colliderMesh=True` but `rayDir.y >= 0`:**
The camera ray is going upward — camera is configured below the terrain pointing up, or has wrong FOV. Check the camera position and rotation in the Inspector. For a top-down isometric view, Y position should be positive (above terrain) and the camera should look downward (negative world Y in forward direction).

---

**If the click log shows `overUI=true` (gate4):**
UIHoverTracker is incorrectly returning true for the whole screen. Add this diagnostic to `UIHoverTracker.IsPointerOverUI`:
```csharp
public static bool IsPointerOverUI
{
    get
    {
        var mp = Input.mousePosition;
        var screenPos = new Vector2(mp.x, Screen.height - mp.y);
        foreach (var panel in _panels)
        {
            if (panel.worldBound.Contains(screenPos))
            {
                Debug.Log($"[UIHoverTracker] blocked by panel: {panel.name} worldBound={panel.worldBound}");
                return true;
            }
        }
        return false;
    }
}
```
Run again and check which panel's worldBound covers the terrain click point. The fix will be to ensure that panel registers only its visible content element, not a full-screen wrapper.

---

**If `[TerrainPainter] click reached: overUI=False` appears but terrain doesn't change visually:**
The issue is in `ApplyBrushAt`. Add a log inside the chunk loop:
```csharp
void ApplyBrushAt(float worldX, float worldZ)
{
    if (SpacetimeDBManager.Conn == null) return;
    var points  = TerrainBrush.AffectedPoints(worldX, worldZ, ActiveBrush.Radius, ActiveBrush.Falloff);
    var byChunk = TerrainBrush.GroupByChunk(points);
    Debug.Log($"[TerrainPainter] ApplyBrushAt({worldX:F1},{worldZ:F1}) chunks={byChunk.Count} points={points.Count}");
    // ... rest unchanged
```
If `chunks=0`, the world coordinates are outside the terrain grid. Check that zone dimensions match the terrain mesh extents.

---

- [ ] **Step 4: Remove all diagnostic logs**

After identifying and fixing the root cause, remove all `Debug.Log` statements added in Steps 1–3 of this task. The final `Update()` methods should be clean.

**Final TerrainPainter.Update()** (clean version):
```csharp
void Update()
{
    if (!EditorState.HasActiveZone || SpacetimeDBManager.Conn == null) return;
    if (ActiveBrush == null) { _hoverRing.enabled = false; return; }

    if (!TryGetTerrainHit(out Vector3 hitPoint))
    {
        _hoverRing.enabled = false;
        return;
    }

    UpdateHoverRing(hitPoint, ActiveBrush.Radius);

    if (Input.GetMouseButton(0) && !UIHoverTracker.IsPointerOverUI)
        ApplyBrushAt(hitPoint.x, hitPoint.z);
}
```

**Final EntityPlacer.Update()** (clean version):
```csharp
void Update()
{
    if (!EditorState.HasActiveZone)            return;
    if (SelectedEntry == null)                 return;
    if (UIHoverTracker.IsPointerOverUI)        return;
    if (!Input.GetMouseButtonDown(0))          return;
    if (SpacetimeDBManager.Conn == null)       return;
    if (_camera == null)                       return;

    var ray = _camera.ScreenPointToRay(Input.mousePosition);
    Vector3 hitPoint;

    RaycastHit hit = default;
    bool colliderHit = _terrainCollider != null &&
                       _terrainCollider.Raycast(ray, out hit, 1000f);

    if (colliderHit)
    {
        hitPoint = hit.point;
    }
    else
    {
        if (Mathf.Abs(ray.direction.y) < 0.0001f) return;
        float t = -ray.origin.y / ray.direction.y;
        if (t <= 0f) return;
        hitPoint = ray.origin + t * ray.direction;
    }

    SpacetimeDBManager.Conn.Reducers.SpawnEntity(
        EditorState.ActiveZoneId,
        SelectedEntry.PrefabName,
        hitPoint.x,
        hitPoint.z,
        hitPoint.y,
        SelectedEntry.EntityType
    );
}
```

- [ ] **Step 5: Verify brush and entity placement work**

Enter Play mode. Select a zone. Select a brush — hover ring should appear over the terrain. Hold left mouse button — terrain height or texture should change. Select an entity — click on terrain — entity should spawn. Both systems must work before committing.

- [ ] **Step 6: Commit**

```bash
cd /home/beek/zoneforge
git add editor/Assets/Scripts/Runtime/TerrainPainter.cs \
        editor/Assets/Scripts/Runtime/EntityPlacer.cs
# Add any other files modified during the fix (e.g. TerrainRenderer.cs if touched)
git commit -m "fix: terrain brush and entity placement — <describe actual root cause found>"
```

---

## Task 3: Camera Controller

**Files:**
- Create: `editor/Assets/Scripts/Runtime/CameraController.cs`

New MonoBehaviour for the main camera. Handles arrow key pan, edge-scroll pan, and scroll-wheel zoom. Attach to the main camera GameObject in SampleScene.

- [ ] **Step 1: Create CameraController.cs**

Create `editor/Assets/Scripts/Runtime/CameraController.cs` with the following content:

```csharp
using UnityEngine;

/// <summary>
/// Pan (arrow keys + edge scroll) and zoom (scroll wheel) for the world editor camera.
/// Attach to the main camera GameObject. Camera rotation is never modified — set the
/// angle you want in the Inspector and this controller preserves it.
///
/// Pan moves along the camera's projected right and forward axes so the direction
/// matches the visual orientation regardless of camera angle.
///
/// Zoom moves the camera along its local Y axis, clamped between MinHeight and MaxHeight.
/// Pan speed scales with current height so the view doesn't feel sluggish when zoomed out.
/// </summary>
public class CameraController : MonoBehaviour
{
    [SerializeField] private float _panSpeed       = 20f;
    [SerializeField] private int   _edgeMargin     = 20;
    [SerializeField] private float _minHeight      = 5f;
    [SerializeField] private float _maxHeight      = 80f;
    [SerializeField] private float _zoomSpeed      = 10f;

    private float ReferenceHeight => (_minHeight + _maxHeight) * 0.5f;

    void Update()
    {
        Vector3 pos = transform.position;

        // ── Pan ────────────────────────────────────────────────────────────────
        // Project camera axes onto the world XZ plane so pan always moves in the
        // direction the camera appears to be looking, regardless of its tilt.
        Vector3 camRight   = Vector3.ProjectOnPlane(transform.right,   Vector3.up).normalized;
        Vector3 camForward = Vector3.ProjectOnPlane(transform.forward, Vector3.up).normalized;

        float dx = 0f, dz = 0f;

        if (!UIHoverTracker.IsPointerOverUI)
        {
            float w = Screen.width;
            float h = Screen.height;
            Vector2 mp = Input.mousePosition;

            // Arrow keys
            if (Input.GetKey(KeyCode.LeftArrow))  dx -= 1f;
            if (Input.GetKey(KeyCode.RightArrow)) dx += 1f;
            if (Input.GetKey(KeyCode.DownArrow))  dz -= 1f;
            if (Input.GetKey(KeyCode.UpArrow))    dz += 1f;

            // Edge scroll — only when mouse is within the window
            if (mp.x >= 0 && mp.x <= w && mp.y >= 0 && mp.y <= h)
            {
                if (mp.x < _edgeMargin)        dx -= 1f;
                if (mp.x > w - _edgeMargin)    dx += 1f;
                if (mp.y < _edgeMargin)        dz -= 1f;
                if (mp.y > h - _edgeMargin)    dz += 1f;
            }
        }

        if (dx != 0f || dz != 0f)
        {
            // Clamp diagonal to unit length so diagonal pan isn't faster
            var dir = new Vector2(dx, dz);
            if (dir.sqrMagnitude > 1f) dir.Normalize();

            float speedScale = Mathf.Max(0.1f, pos.y / ReferenceHeight);
            float speed = _panSpeed * speedScale * Time.deltaTime;

            Vector3 move = (dir.x * camRight + dir.y * camForward) * speed;
            pos.x += move.x;
            pos.z += move.z;
        }

        // ── Zoom ───────────────────────────────────────────────────────────────
        float scroll = Input.mouseScrollDelta.y;
        if (Mathf.Abs(scroll) > 0.001f)
            pos.y = Mathf.Clamp(pos.y - scroll * _zoomSpeed, _minHeight, _maxHeight);

        transform.position = pos;
    }
}
```

- [ ] **Step 2: Attach to the camera in SampleScene**

Open SampleScene in the Unity Editor. Select the main camera GameObject (the one with the `Camera` component tagged `MainCamera`). In the Inspector, click **Add Component** and add `CameraController`. Leave all serialized fields at their defaults.

- [ ] **Step 3: Verify in Play mode**

Enter Play mode:
- Press left/right/up/down arrow keys — camera should pan in the corresponding direction, with movement matching the camera's visual orientation.
- Move the mouse to within 20px of any screen edge — camera should pan in the same direction.
- Scroll the mouse wheel — camera should zoom in (scroll up) and out (scroll down), stopping at MinHeight=5 and MaxHeight=80.
- Pan speed should feel proportionally faster when zoomed out and slower when zoomed in.

- [ ] **Step 4: Commit**

```bash
cd /home/beek/zoneforge
git add editor/Assets/Scripts/Runtime/CameraController.cs
git commit -m "feat: add camera pan (arrows + edge scroll) and zoom (scroll wheel)"
```

---

## Task 4: Entity Panel UXML and USS Redesign

**Files:**
- Modify: `editor/Assets/UI/EntityPalettePanel.uxml`
- Modify: `editor/Assets/UI/EntityPalettePanel.uss`

Redesign the entity panel: move to bottom-right, match brush panel size and style, add panel header with collapse button, add search field.

- [ ] **Step 1: Rewrite EntityPalettePanel.uxml**

Replace the entire contents of `editor/Assets/UI/EntityPalettePanel.uxml` with:

```xml
<ui:UXML xmlns:ui="UnityEngine.UIElements">
    <ui:VisualElement name="panel" class="panel">
        <ui:VisualElement name="panel-header" class="panel-header">
            <ui:Label text="Entities" class="panel-title"/>
            <ui:Button name="collapse-btn" text="▾" class="collapse-btn"/>
        </ui:VisualElement>
        <ui:VisualElement name="panel-body">
            <ui:Label text="Search" class="search-label"/>
            <ui:TextField name="search-field" class="search-field"/>
            <ui:ScrollView name="entity-scroll" class="entity-scroll">
                <ui:VisualElement name="entity-grid" class="entity-grid"/>
            </ui:ScrollView>
        </ui:VisualElement>
    </ui:VisualElement>
</ui:UXML>
```

- [ ] **Step 2: Rewrite EntityPalettePanel.uss**

Replace the entire contents of `editor/Assets/UI/EntityPalettePanel.uss` with:

```css
/* ── Panel container ────────────────────────────────────────── */
.panel {
    position: absolute;
    bottom: 60px;
    right: 16px;
    width: 260px;
    background-color: rgba(20, 20, 25, 0.92);
    border-radius: 6px;
    padding: 16px;
}

/* ── Header row ─────────────────────────────────────────────── */
.panel-header {
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 8px;
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
    background-color: rgba(0, 0, 0, 0);
    border-width: 0;
    color: rgb(180, 180, 180);
    font-size: 12px;
    -unity-text-align: middle-center;
}

/* ── Search field ───────────────────────────────────────────── */
.search-label {
    font-size: 11px;
    color: rgb(160, 160, 180);
    margin-bottom: 3px;
}

.search-field {
    margin-bottom: 8px;
}

.search-field .unity-text-field__input {
    background-color: rgba(30, 30, 40, 1);
    color: rgb(220, 220, 220);
    border-radius: 3px;
}

/* ── Entity list ─────────────────────────────────────────────── */
.entity-scroll {
    max-height: 280px;
}

.entity-grid {
    flex-direction: column;
}

/* Foldout group header text */
.entity-group > .unity-foldout__toggle > .unity-toggle__label {
    color: rgb(200, 200, 210);
    font-size: 11px;
    -unity-font-style: bold;
}

.entity-row {
    flex-direction: row;
    align-items: center;
    padding: 3px 4px;
    border-radius: 2px;
    border-left-width: 2px;
    border-left-color: rgba(0, 0, 0, 0);
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

- [ ] **Step 3: Verify layout in Play mode**

Enter Play mode. The entity panel should now appear at the bottom-right of the screen (above where the toolbar icon will be), match the brush panel's dark background and width, show a "Entities" title with a ▾ collapse button, and have a "Search" label with an empty text field above the entity list.

(The collapse button and search will not be wired yet — that's Task 5.)

- [ ] **Step 4: Commit**

```bash
cd /home/beek/zoneforge
git add editor/Assets/UI/EntityPalettePanel.uxml \
        editor/Assets/UI/EntityPalettePanel.uss
git commit -m "refactor: entity panel — bottom-right position, brush-panel styling, search UI"
```

---

## Task 5: Entity Panel C# — Collapse, Search, SetVisible

**Files:**
- Modify: `editor/Assets/Scripts/Runtime/EntityPalettePanel.cs`

Wire the new UXML elements added in Task 4: collapse toggle, search filter, and `SetVisible` for ToolbarController.

- [ ] **Step 1: Rewrite EntityPalettePanel.cs**

Replace the entire contents of `editor/Assets/Scripts/Runtime/EntityPalettePanel.cs` with:

```csharp
// editor/Assets/Scripts/Runtime/EntityPalettePanel.cs
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UIElements;

[RequireComponent(typeof(UIDocument))]
public class EntityPalettePanel : MonoBehaviour
{
    [SerializeField] StyleSheet _styleSheet;

    // -----------------------------------------------------------------------
    // Static definition list — single source of truth for all entity types

    public static Dictionary<string, EntityDefinition> ByPrefabName { get; private set; }
    public static Dictionary<string, Texture2D>        Thumbnails   { get; private set; }

    private static readonly EntityDefinition[] Definitions =
    {
        new EntityDefinition { DisplayName = "Guard",    PrefabName = "guard",    EntityType = "NPC",   Color = Color.blue },
        new EntityDefinition { DisplayName = "Merchant", PrefabName = "merchant", EntityType = "NPC",   Color = Color.cyan },
        new EntityDefinition { DisplayName = "Skeleton", PrefabName = "skeleton", EntityType = "Enemy", Color = new Color(0.961f, 0.961f, 0.863f) },
        new EntityDefinition { DisplayName = "Goblin",   PrefabName = "goblin",   EntityType = "Enemy", Color = new Color(0.565f, 0.933f, 0.565f) },
        new EntityDefinition { DisplayName = "Tree",     PrefabName = "tree",     EntityType = "Prop",  Color = Color.green },
        new EntityDefinition { DisplayName = "Rock",     PrefabName = "rock",     EntityType = "Prop",  Color = new Color(0.753f, 0.753f, 0.753f) },
        new EntityDefinition { DisplayName = "Chest",    PrefabName = "chest",    EntityType = "Prop",  Color = Color.yellow },
        new EntityDefinition { DisplayName = "Wall",     PrefabName = "wall",     EntityType = "Prop",  Color = new Color(0.251f, 0.251f, 0.251f) },
    };

    static EntityPalettePanel()
    {
        ByPrefabName = new Dictionary<string, EntityDefinition>();
        Thumbnails   = new Dictionary<string, Texture2D>();
        foreach (var def in Definitions)
            ByPrefabName[def.PrefabName] = def;
    }

    static void EnsureThumbnails()
    {
        if (Thumbnails.Count > 0) return;
        foreach (var def in Definitions)
            Thumbnails[def.PrefabName] = MakeColorTexture(def.Color);
    }

    // -----------------------------------------------------------------------
    // Instance state

    private static EntityPalettePanel _instance;

    private VisualElement _panelRoot;
    private VisualElement _panelBody;
    private VisualElement _entityGrid;
    private VisualElement _selectedRow;
    private VisualElement _trackedPanel;
    private Button        _collapseBtn;
    private TextField     _searchField;
    private bool          _isCollapsed;

    // Row tracking for search filtering
    private struct RowEntry
    {
        public VisualElement Row;
        public EntityDefinition Def;
        public Foldout Group;
    }
    private readonly List<RowEntry> _allRows = new();

    // -----------------------------------------------------------------------

    void OnEnable()
    {
        EnsureThumbnails();
        _instance = this;
        var root = GetComponent<UIDocument>().rootVisualElement;
        root.pickingMode = PickingMode.Ignore;

        if (_styleSheet != null)
            root.styleSheets.Add(_styleSheet);

        _panelRoot  = root.Q<VisualElement>("panel");
        _panelBody  = root.Q<VisualElement>("panel-body");
        _entityGrid = root.Q<VisualElement>("entity-grid");
        _collapseBtn = root.Q<Button>("collapse-btn");
        _searchField = root.Q<TextField>("search-field");

        if (_collapseBtn != null)
            _collapseBtn.clicked += ToggleCollapse;

        if (_searchField != null)
            _searchField.RegisterValueChangedCallback(e => FilterBySearch(e.newValue));

        BuildGrid();

        EditorState.OnActiveZoneChanged += OnActiveZoneChanged;

        _trackedPanel = _panelRoot ?? root;
        UIHoverTracker.Register(_trackedPanel);

        // Reflect current zone state immediately
        _panelRoot?.SetEnabled(EditorState.HasActiveZone);
    }

    void OnDisable()
    {
        if (_collapseBtn != null)
            _collapseBtn.clicked -= ToggleCollapse;

        EditorState.OnActiveZoneChanged -= OnActiveZoneChanged;
        UIHoverTracker.Unregister(_trackedPanel);
        if (_instance == this) _instance = null;
    }

    // -----------------------------------------------------------------------
    // Visibility and collapse

    /// Called by ToolbarController. Shows or hides the entire panel.
    public void SetVisible(bool visible)
    {
        if (_panelRoot == null) return;
        _panelRoot.style.display = visible ? DisplayStyle.Flex : DisplayStyle.None;
    }

    void ToggleCollapse()
    {
        _isCollapsed = !_isCollapsed;
        if (_panelBody   != null) _panelBody.style.display = _isCollapsed ? DisplayStyle.None : DisplayStyle.Flex;
        if (_collapseBtn != null) _collapseBtn.text        = _isCollapsed ? "▸" : "▾";
    }

    // -----------------------------------------------------------------------
    // Search filtering

    void FilterBySearch(string query)
    {
        query = query?.Trim() ?? "";
        bool noFilter = string.IsNullOrEmpty(query);

        var foldoutVisible = new Dictionary<Foldout, bool>();

        foreach (var entry in _allRows)
        {
            bool show = noFilter ||
                        entry.Def.DisplayName.IndexOf(query, System.StringComparison.OrdinalIgnoreCase) >= 0;
            entry.Row.style.display = show ? DisplayStyle.Flex : DisplayStyle.None;

            if (!foldoutVisible.ContainsKey(entry.Group))
                foldoutVisible[entry.Group] = false;
            if (show)
                foldoutVisible[entry.Group] = true;
        }

        foreach (var kv in foldoutVisible)
            kv.Key.style.display = kv.Value ? DisplayStyle.Flex : DisplayStyle.None;
    }

    // -----------------------------------------------------------------------
    // Grid construction

    void BuildGrid()
    {
        if (_entityGrid == null)
        {
            Debug.LogError("[EntityPalettePanel] 'entity-grid' element not found in UXML");
            return;
        }
        _entityGrid.Clear();
        _allRows.Clear();
        _selectedRow = null;

        var groupOrder = new[] { "NPC", "Enemy", "Prop" };
        foreach (var groupName in groupOrder)
        {
            var foldout = new Foldout { text = groupName, value = true };
            foldout.AddToClassList("entity-group");

            foreach (var def in Definitions)
            {
                if (def.EntityType != groupName) continue;

                var row = new VisualElement();
                row.AddToClassList("entity-row");

                var swatch = new VisualElement();
                swatch.AddToClassList("entity-swatch");
                swatch.style.backgroundColor = def.Color;
                row.Add(swatch);

                var lbl = new Label(def.DisplayName);
                lbl.AddToClassList("entity-row__label");
                row.Add(lbl);

                var captured = def;
                row.RegisterCallback<ClickEvent>(_ => SelectEntry(row, captured));

                foldout.Add(row);
                _allRows.Add(new RowEntry { Row = row, Def = def, Group = foldout });
            }

            _entityGrid.Add(foldout);
        }
    }

    void SelectEntry(VisualElement row, EntityDefinition def)
    {
        _selectedRow?.RemoveFromClassList("entity-row--selected");

        if (_selectedRow == row)
        {
            _selectedRow = null;
            EntityPlacer.SelectedEntry = null;
            return;
        }

        _selectedRow = row;
        _selectedRow.AddToClassList("entity-row--selected");

        EntityPlacer.SelectedEntry = def;
        TerrainPainter.ActiveBrush  = null;
    }

    public static void ClearSelection()
    {
        if (_instance == null) return;
        _instance._selectedRow?.RemoveFromClassList("entity-row--selected");
        _instance._selectedRow = null;
        EntityPlacer.SelectedEntry = null;
    }

    void OnActiveZoneChanged(ulong zoneId)
    {
        _panelRoot?.SetEnabled(EditorState.HasActiveZone);

        if (!EditorState.HasActiveZone)
        {
            _selectedRow?.RemoveFromClassList("entity-row--selected");
            _selectedRow               = null;
            EntityPlacer.SelectedEntry = null;
        }
    }

    static Texture2D MakeColorTexture(Color color)
    {
        var tex = new Texture2D(1, 1);
        tex.SetPixel(0, 0, color);
        tex.Apply();
        return tex;
    }
}
```

- [ ] **Step 2: Verify collapse and search in Play mode**

Enter Play mode:
- The entity panel should appear at bottom-right.
- Click the ▾ button in the panel header — the entity list and search field should collapse/expand.
- Click an entity (e.g. Guard) — it should highlight as selected.
- Type "g" in the Search field — only "Guard", "Goblin" rows should remain visible; foldout groups with no matching entities should hide.
- Clear the search — all entities should reappear.
- The `SetVisible` method is not yet wired (no toolbar button yet) — that's Task 6.

- [ ] **Step 3: Commit**

```bash
cd /home/beek/zoneforge
git add editor/Assets/Scripts/Runtime/EntityPalettePanel.cs
git commit -m "feat: entity panel — collapse toggle, search filter, SetVisible for toolbar"
```

---

## Task 6: Toolbar Entity Button

**Files:**
- Modify: `editor/Assets/UI/ToolbarController.uxml`
- Modify: `editor/Assets/UI/ToolbarController.uss`
- Modify: `editor/Assets/Scripts/Runtime/ToolbarController.cs`

Add a bottom-right icon button that shows/hides the entity panel, matching the style of the existing brush/zone/graph buttons.

- [ ] **Step 1: Add entity button to ToolbarController.uxml**

Replace the entire contents of `editor/Assets/UI/ToolbarController.uxml` with:

```xml
<ui:UXML xmlns:ui="UnityEngine.UIElements">
  <ui:VisualElement name="toolbar-root" class="toolbar-root">

    <!-- Top-left icon strip -->
    <ui:VisualElement name="left-strip" class="icon-strip left-strip">
      <ui:Button name="btn-zone"  class="icon-btn" text="🗺" tooltip="Zone Manager"/>
      <ui:Button name="btn-graph" class="icon-btn" text="⬡" tooltip="World Graph"/>
    </ui:VisualElement>

    <!-- Top-right brush icon -->
    <ui:Button name="btn-brush" class="icon-btn right-btn" text="🖌" tooltip="Brush"/>

    <!-- Bottom-right entity icon -->
    <ui:Button name="btn-entity" class="icon-btn bottom-right-btn" text="☰" tooltip="Entities"/>

  </ui:VisualElement>
</ui:UXML>
```

- [ ] **Step 2: Add .bottom-right-btn style to ToolbarController.uss**

Open `editor/Assets/UI/ToolbarController.uss`. Append to the end of the file:

```css
.bottom-right-btn {
    position: absolute;
    bottom: 6px;
    right: 6px;
    margin-right: 0;
}
```

- [ ] **Step 3: Update ToolbarController.cs**

Replace the entire contents of `editor/Assets/Scripts/Runtime/ToolbarController.cs` with:

```csharp
using UnityEngine;
using UnityEngine.UIElements;

/// UIDocument-based icon toolbar. Toggle buttons control ZoneCreationPanel,
/// WorldGraphPanel, TilePalettePanel, and EntityPalettePanel.
/// Assign panel references in Inspector.
[RequireComponent(typeof(UIDocument))]
public class ToolbarController : MonoBehaviour
{
    public static ToolbarController Instance { get; private set; }

    [SerializeField] StyleSheet        _styleSheet;
    [SerializeField] ZoneCreationPanel _zonePanel;
    [SerializeField] WorldGraphPanel   _graphPanel;
    [SerializeField] TilePalettePanel  _brushPanel;
    [SerializeField] EntityPalettePanel _entityPanel;

    private Button _btnZone;
    private Button _btnGraph;
    private Button _btnBrush;
    private Button _btnEntity;

    private bool _zoneOpen;
    private bool _graphOpen;
    private bool _brushOpen;
    private bool _entityOpen;

    void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;
    }

    void OnEnable()
    {
        var root = GetComponent<UIDocument>().rootVisualElement;
        root.pickingMode = PickingMode.Ignore;
        if (_styleSheet != null) root.styleSheets.Add(_styleSheet);

        _btnZone   = root.Q<Button>("btn-zone");
        _btnGraph  = root.Q<Button>("btn-graph");
        _btnBrush  = root.Q<Button>("btn-brush");
        _btnEntity = root.Q<Button>("btn-entity");

        _btnZone  .clicked += ToggleZone;
        _btnGraph .clicked += ToggleGraph;
        _btnBrush .clicked += ToggleBrush;
        _btnEntity.clicked += ToggleEntity;

        // Zone panel is visible on startup; reflect that in button state.
        _zoneOpen = true;
        SetActive(_btnZone, true);
    }

    void OnDisable()
    {
        if (_btnZone   != null) _btnZone  .clicked -= ToggleZone;
        if (_btnGraph  != null) _btnGraph .clicked -= ToggleGraph;
        if (_btnBrush  != null) _btnBrush .clicked -= ToggleBrush;
        if (_btnEntity != null) _btnEntity.clicked -= ToggleEntity;
    }

    void ToggleZone()
    {
        _zoneOpen = !_zoneOpen;
        if (_zoneOpen) { _graphOpen = false; RefreshGraph(); }
        RefreshZone();
    }

    void ToggleGraph()
    {
        _graphOpen = !_graphOpen;
        if (_graphOpen) { _zoneOpen = false; RefreshZone(); }
        RefreshGraph();
    }

    void ToggleBrush()
    {
        _brushOpen = !_brushOpen;
        RefreshBrush();
    }

    void ToggleEntity()
    {
        _entityOpen = !_entityOpen;
        RefreshEntity();
    }

    /// Called by WorldGraphPanel or PortalRenderer to open the graph panel.
    public void OpenWorldGraph()
    {
        if (_graphOpen) return;
        _graphOpen = true;
        _zoneOpen  = false;
        RefreshZone();
        RefreshGraph();
    }

    void RefreshZone()
    {
        _zonePanel?.SetVisible(_zoneOpen);
        SetActive(_btnZone, _zoneOpen);
    }

    void RefreshGraph()
    {
        _graphPanel?.SetVisible(_graphOpen);
        SetActive(_btnGraph, _graphOpen);
    }

    void RefreshBrush()
    {
        _brushPanel?.SetVisible(_brushOpen);
        SetActive(_btnBrush, _brushOpen);
    }

    void RefreshEntity()
    {
        _entityPanel?.SetVisible(_entityOpen);
        SetActive(_btnEntity, _entityOpen);
    }

    static void SetActive(Button btn, bool active)
    {
        if (btn == null) return;
        if (active) btn.AddToClassList("active");
        else        btn.RemoveFromClassList("active");
    }
}
```

- [ ] **Step 4: Assign EntityPalettePanel in the Inspector**

In the Unity Editor (not Play mode), select the ToolbarController GameObject in SampleScene. In the Inspector, the `ToolbarController` component now has an **Entity Panel** field. Drag the EntityPalettePanel GameObject from the Hierarchy into this field.

- [ ] **Step 5: Verify in Play mode**

Enter Play mode:
- A ☰ icon should appear at the bottom-right of the screen.
- Clicking it should show the entity panel (at bottom-right, above the icon) with the ☰ button highlighted.
- Clicking it again should hide the entity panel and unhighlight the button.
- The entity panel's own ▾ collapse button should still collapse just the body when the panel is open.
- The brush (🖌), zone (🗺), and graph (⬡) buttons should continue to work as before.

- [ ] **Step 6: Commit**

```bash
cd /home/beek/zoneforge
git add editor/Assets/UI/ToolbarController.uxml \
        editor/Assets/UI/ToolbarController.uss \
        editor/Assets/Scripts/Runtime/ToolbarController.cs
git commit -m "feat: toolbar entity button — bottom-right ☰ icon toggles entity panel"
```

---

## Self-Review

**Spec coverage:**
- ✅ Text fields white on white → Task 1 (ZoneCreationPanel.uss dark bg)
- ✅ Brushes not working → Task 2 (diagnostic + fix)
- ✅ Entity placement not working → Task 2 (EntityPlacer diagnostic + fix)
- ✅ Camera movement → Task 3 (CameraController.cs)
- ✅ Entity panel position bottom-right → Task 4 (EntityPalettePanel.uss)
- ✅ Entity panel match brush panel style → Task 4 (USS width/bg/padding)
- ✅ Entity panel search function → Tasks 4 + 5
- ✅ Entity panel foldout header text colour → Task 1 (EntityPalettePanel.uss)
- ✅ Entity panel collapse button → Tasks 4 + 5
- ✅ Toolbar entity icon → Task 6
- ✅ Scroll wheel zoom → Task 3

**Type consistency:** `EntityPalettePanel.SetVisible(bool)` defined in Task 5, called in Task 6. `FilterBySearch(string)` defined and called in Task 5. `RowEntry` struct is private to the class. All method names consistent across tasks.

**No placeholders:** All code blocks are complete. No TBD items.
