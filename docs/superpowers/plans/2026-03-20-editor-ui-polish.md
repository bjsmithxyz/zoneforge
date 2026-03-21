# Editor UI Polish & Bug Fixes — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix two critical bugs that prevent terrain painting and entity placement from working, then polish all three editor UI panels (Zone Manager, Brush Panel, Entity Palette).

**Architecture:** All changes are confined to the `editor/` Unity submodule. No server changes. Five independent tasks in dependency order: bug fixes first (they unblock basic editor use), then panel polish. Each task produces a working commit.

**Tech Stack:** Unity 2022.3 LTS, C#, UIToolkit (UIElements), USS stylesheets, UXML layouts, SpacetimeDB C# SDK.

**Testing note:** There is no CLI test runner for Unity runtime code. Verification for each task is described as a Play Mode check in the Unity Editor. Ensure the project compiles (no red errors in the Console window) before entering Play Mode.

**Working directory for all file edits:** `/home/beek/zoneforge/editor/`

---

## File Map

| File | Task(s) | Change |
| --- | --- | --- |
| `Assets/Scripts/Runtime/TerrainRenderer.cs` | 1 | Add `[RequireComponent(typeof(MeshCollider))]`; add `_collider` field; update `sharedMesh` in `BuildFullMesh()` |
| `Assets/Scripts/Runtime/EntityPlacer.cs` | 2 | Restructure `Update()` with Y=0 plane fallback |
| `Assets/Scripts/Runtime/ZoneCreationPanel.cs` | 3 | Backfill existing zones in `OnConnected`; add collapse fields + `SetCollapsed`/`ToggleCollapse` |
| `Assets/UI/ZoneCreationPanel.uxml` | 3 | Wrap content in `panel-header` + `panel-body` |
| `Assets/UI/ZoneCreationPanel.uss` | 3 | Fix field heights; fix cursor; add header/collapse-btn styles |
| `Assets/Scripts/Runtime/TilePalettePanel.cs` | 4 | Add `_panelBody`, `_collapseBtn`, `_isCollapsed`; wire `ToggleCollapse` |
| `Assets/UI/TilePalettePanel.uxml` | 4 | Wrap content in `panel-header` + `panel-body`; add `first-section` class |
| `Assets/UI/TilePalettePanel.uss` | 4 | Remove `:first-child`; add `.first-section`; add toggle label colors; add header styles |
| `Assets/Scripts/Runtime/EntityPalettePanel.cs` | 5 | Rebuild `BuildGrid()` with Foldout grouping; rename `_selectedCell`→`_selectedRow`; update `SelectEntry`, `ClearSelection`, `OnActiveZoneChanged` |
| `Assets/UI/EntityPalettePanel.uxml` | 5 | Increase `entity-scroll` max-height to 320px |
| `Assets/UI/EntityPalettePanel.uss` | 5 | Replace cell styles with row/swatch/group styles; update position and width |

---

## Task 1: Bug — TerrainRenderer MeshCollider sync

**Files:**
- Modify: `Assets/Scripts/Runtime/TerrainRenderer.cs`

**Context:** `TerrainPainter` calls `GetComponent<MeshCollider>()` to raycast for painting. `EntityPlacer` holds a serialized ref to the same collider. Both fail silently because `TerrainRenderer.BuildFullMesh()` never assigns the built `Mesh` to `MeshCollider.sharedMesh`, so the collider shape is always empty.

- [ ] **Step 1: Open the file and read it**

  Read `Assets/Scripts/Runtime/TerrainRenderer.cs`. Confirm:
  - Line 11: `[RequireComponent(typeof(MeshFilter), typeof(MeshRenderer))]` — `MeshCollider` is absent
  - No `MeshCollider` field exists
  - `BuildFullMesh()` ends without a `sharedMesh` assignment

- [ ] **Step 2: Add `[RequireComponent(typeof(MeshCollider))]`**

  Change line 11 from:
  ```csharp
  [RequireComponent(typeof(MeshFilter), typeof(MeshRenderer))]
  ```
  To:
  ```csharp
  [RequireComponent(typeof(MeshFilter), typeof(MeshRenderer), typeof(MeshCollider))]
  ```

- [ ] **Step 3: Add the `_collider` field and assign it in `Awake`**

  After the existing `private MeshRenderer _renderer;` field declaration, add:
  ```csharp
  private MeshCollider _collider;
  ```

  In `Awake()`, after `_filter.mesh = _mesh;`, add:
  ```csharp
  _collider = GetComponent<MeshCollider>();
  ```

- [ ] **Step 4: Update `sharedMesh` at the end of `BuildFullMesh()`**

  `BuildFullMesh()` ends with `_renderer.sharedMaterial = _terrainMaterial;`. After that line, add:
  ```csharp
  if (_collider != null)
      _collider.sharedMesh = _mesh;
  ```

- [ ] **Step 5: Scene setup — add MeshCollider to the TerrainRenderer GameObject**

  `[RequireComponent]` does not retroactively add components to existing scene objects. In the Unity Editor (not Play mode):
  1. Select the `TerrainRenderer` GameObject in the Hierarchy
  2. In the Inspector, click **Add Component** → search for **Mesh Collider** → add it
  3. The `sharedMesh` field in the Mesh Collider Inspector can remain empty — `TerrainRenderer.BuildFullMesh()` will assign it at runtime

- [ ] **Step 6: Verify**

  Enter Play Mode. Select a zone. Confirm in the Console that no NullReferenceException fires from `TerrainRenderer`. The hover ring in `TerrainPainter` should now track the terrain surface accurately (not just Y=0 plane).

- [ ] **Step 7: Commit**

  ```bash
  cd /home/beek/zoneforge/editor
  git add Assets/Scripts/Runtime/TerrainRenderer.cs
  git commit -m "fix: sync MeshCollider.sharedMesh after terrain mesh rebuild"
  ```

---

## Task 2: Bug — EntityPlacer Y=0 plane fallback

**Files:**
- Modify: `Assets/Scripts/Runtime/EntityPlacer.cs`

**Context:** The current `Update()` has an early `return` when `_terrainCollider == null`, making it impossible to place entities if the Inspector reference is unwired (even with a functional terrain). Restructuring to a conditional branch adds a Y=0 fallback that matches `TerrainPainter`'s existing fallback logic, so entity placement works whenever the terrain is flat or the collider is missing.

- [ ] **Step 1: Open the file and read it**

  Read `Assets/Scripts/Runtime/EntityPlacer.cs`. Confirm `Update()` has this sequence of hard returns:
  ```csharp
  if (!EditorState.HasActiveZone)     return;
  if (SelectedEntry == null)          return;
  if (UIHoverTracker.IsPointerOverUI) return;
  if (!Input.GetMouseButtonDown(0))   return;
  if (SpacetimeDBManager.Conn == null) return;
  if (_terrainCollider == null)       return;   // ← hard return, no fallback
  if (_camera == null)                return;

  var ray = _camera.ScreenPointToRay(Input.mousePosition);
  if (!_terrainCollider.Raycast(ray, out RaycastHit hit, 1000f)) return;  // ← no fallback
  ```

- [ ] **Step 2: Rewrite `Update()` with the conditional branch**

  Replace the entire body of `Update()` with:
  ```csharp
  void Update()
  {
      if (!EditorState.HasActiveZone)      return;
      if (SelectedEntry == null)           return;
      if (UIHoverTracker.IsPointerOverUI)  return;
      if (!Input.GetMouseButtonDown(0))    return;
      if (SpacetimeDBManager.Conn == null) return;
      if (_camera == null)                 return;

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
          // Y=0 plane fallback — matches TerrainPainter.TryGetTerrainHit logic
          if (Mathf.Abs(ray.direction.y) < 0.0001f) return;
          float t = -ray.origin.y / ray.direction.y;
          if (t <= 0f) return;
          hitPoint = ray.origin + t * ray.direction;
      }

      SpacetimeDBManager.Conn.Reducers.SpawnEntity(
          EditorState.ActiveZoneId,
          SelectedEntry.PrefabName,
          hitPoint.x,
          hitPoint.z,   // y reducer param = world Z (horizontal depth axis)
          hitPoint.y,   // elevation = world Y
          SelectedEntry.EntityType
      );
  }
  ```

- [ ] **Step 3: Verify**

  Enter Play Mode. Select a zone. Select an entity in the Entity Palette. Click on the terrain. Confirm in the Console (and via `spacetime logs zoneforge-server`) that `spawn_entity` is called and an entity appears in the scene.

- [ ] **Step 4: Commit**

  ```bash
  git add Assets/Scripts/Runtime/EntityPlacer.cs
  git commit -m "fix: add Y=0 fallback to EntityPlacer when MeshCollider unavailable"
  ```

---

## Task 3: Zone Manager — backfill fix + collapsible panel

**Files:**
- Modify: `Assets/Scripts/Runtime/ZoneCreationPanel.cs`
- Modify: `Assets/UI/ZoneCreationPanel.uxml`
- Modify: `Assets/UI/ZoneCreationPanel.uss`

**Context:** Existing zones never appear in the list because SpacetimeDB fires initial `OnInsert` events before our handler is registered. The panel also needs a collapsible header (auto-collapses when a zone is activated), field-height fixes, and cursor fix.

### Step group A — backfill existing zones

- [ ] **Step 1: Update `OnConnected` in `ZoneCreationPanel.cs`**

  Find `void OnConnected()` (currently: enables create button, sets status text). Add a backfill loop after enabling the button:
  ```csharp
  void OnConnected()
  {
      _createBtn.SetEnabled(true);

      // SpacetimeDB fires initial OnInsert before our handler is registered.
      // Backfill existing zones from the subscription cache.
      if (SpacetimeDBManager.Conn != null)
      {
          foreach (var zone in SpacetimeDBManager.Conn.Db.Zone.Iter())
          {
              if (!_zoneRows.ContainsKey(zone.Id))
                  AddZoneRow(zone);
          }
      }

      RefreshStatus();
  }
  ```

- [ ] **Step 2: Verify zone list**

  Enter Play Mode with at least one zone already in the database. Confirm the Zone Manager list shows the existing zone immediately after "Subscription applied" logs. Click it — confirm painting becomes available (hover ring appears over terrain).

### Step group B — collapsible header

- [ ] **Step 3: Restructure `ZoneCreationPanel.uxml`**

  Replace the entire file content with:
  ```xml
  <ui:UXML xmlns:ui="UnityEngine.UIElements" editor-extension-mode="False">
      <ui:VisualElement name="panel" class="panel">

          <ui:VisualElement name="panel-header" class="panel-header">
              <ui:Label text="Zone Manager" class="panel-title"/>
              <ui:Button name="collapse-btn" text="▾" class="collapse-btn"/>
          </ui:VisualElement>

          <ui:VisualElement name="panel-body">
              <ui:Label name="status-label" text="Connecting..." class="status-label"/>

              <ui:VisualElement class="section">
                  <ui:Label text="Create Zone" class="section-title"/>
                  <ui:VisualElement class="form-row">
                      <ui:Label text="Name" class="form-label"/>
                      <ui:TextField name="zone-name" class="form-field"/>
                  </ui:VisualElement>
                  <ui:VisualElement class="form-row">
                      <ui:Label text="Width" class="form-label"/>
                      <ui:IntegerField name="zone-width" value="32" class="form-field-small"/>
                      <ui:Label text="Height" class="form-label"/>
                      <ui:IntegerField name="zone-height" value="32" class="form-field-small"/>
                  </ui:VisualElement>
                  <ui:FloatField label="Water Level" name="water-level" value="0.5"/>
                  <ui:Button name="create-btn" text="Create Zone" class="create-btn"/>
              </ui:VisualElement>

              <ui:VisualElement class="section">
                  <ui:Label text="Existing Zones" class="section-title"/>
                  <ui:ScrollView name="zones-list" class="zones-list"/>
              </ui:VisualElement>
          </ui:VisualElement>

      </ui:VisualElement>
  </ui:UXML>
  ```

- [ ] **Step 4: Add collapse fields and methods to `ZoneCreationPanel.cs`**

  Add three new fields after the existing `private VisualElement _trackedPanel;` line:
  ```csharp
  private VisualElement _panelBody;
  private Button _collapseBtn;
  private bool _isCollapsed = false;
  ```

  In `OnEnable()`, after the `_trackedPanel` and `UIHoverTracker.Register` lines, add:
  ```csharp
  _panelBody   = root.Q<VisualElement>("panel-body");
  _collapseBtn = root.Q<Button>("collapse-btn");
  _collapseBtn.clicked += ToggleCollapse;
  ```

  In `OnDisable()`, after `_createBtn.clicked -= OnCreateClicked;`, add:
  ```csharp
  _collapseBtn.clicked -= ToggleCollapse;
  ```

  Add two new methods after `SetStatus`:
  ```csharp
  void SetCollapsed(bool collapsed)
  {
      _isCollapsed = collapsed;
      if (_panelBody != null)
          _panelBody.style.display = collapsed ? DisplayStyle.None : DisplayStyle.Flex;
      if (_collapseBtn != null)
          _collapseBtn.text = collapsed ? "▸" : "▾";
  }

  void ToggleCollapse() => SetCollapsed(!_isCollapsed);
  ```

  In the existing `OnActiveZoneChanged(ulong zoneId)`, add a call to `SetCollapsed` at the end. Read the current method body first and preserve the row-highlighting logic verbatim — only append the `SetCollapsed` call:
  ```csharp
  void OnActiveZoneChanged(ulong zoneId)
  {
      // --- preserve existing code that clears and sets zone-row--active class ---

      // Add this line at the end:
      SetCollapsed(zoneId != 0);
  }
  ```

- [ ] **Step 5: Update `ZoneCreationPanel.uss`**

  **Fix field heights** — find and replace both height declarations:
  ```css
  /* OLD */
  .form-field {
      flex-grow: 1;
      height: 26px;
  }
  /* NEW */
  .form-field {
      flex-grow: 1;
      height: auto;
      min-height: 24px;
  }
  ```
  ```css
  /* OLD */
  .form-field-small {
      width: 60px;
      height: 26px;
      margin-right: 10px;
  }
  /* NEW */
  .form-field-small {
      width: 60px;
      height: auto;
      min-height: 24px;
      margin-right: 10px;
  }
  ```

  **Fix cursor** — in `.zone-row`:
  ```css
  /* OLD */
  cursor: link;
  /* NEW */
  cursor: arrow;
  ```

  **Add header and collapse-btn styles** — append to end of file:
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
      background-color: rgba(0, 0, 0, 0);
      border-width: 0;
      color: rgb(180, 180, 180);
      font-size: 12px;
      -unity-text-align: middle-center;
  }
  ```

- [ ] **Step 6: Verify**

  Enter Play Mode. Confirm:
  - Text in Name/Width/Height fields is no longer clipped
  - "Runtime cursors" warning is gone from the Console
  - Zone Manager shows the "Zone Manager" title with a ▾ chevron button
  - Clicking an existing zone auto-collapses the panel to header-only
  - Clicking the chevron manually re-expands

- [ ] **Step 7: Commit**

  ```bash
  git add Assets/Scripts/Runtime/ZoneCreationPanel.cs \
          Assets/UI/ZoneCreationPanel.uxml \
          Assets/UI/ZoneCreationPanel.uss
  git commit -m "fix: backfill existing zones on connect; add collapsible Zone Manager panel"
  ```

---

## Task 4: Brush Panel — collapse toggle + USS fixes

**Files:**
- Modify: `Assets/Scripts/Runtime/TilePalettePanel.cs`
- Modify: `Assets/UI/TilePalettePanel.uxml`
- Modify: `Assets/UI/TilePalettePanel.uss`

**Context:** Radio button labels are black (unreadable on dark background). The `:first-child` pseudo-class is unsupported and logs a warning. Add a collapsible header.

- [ ] **Step 1: Restructure `TilePalettePanel.uxml`**

  Replace the entire file content with:
  ```xml
  <ui:UXML xmlns:ui="UnityEngine.UIElements">
      <ui:VisualElement name="panel" class="panel">

          <ui:VisualElement name="panel-header" class="panel-header">
              <ui:Label text="Brush Panel" class="panel-title"/>
              <ui:Button name="collapse-btn" text="▾" class="collapse-btn"/>
          </ui:VisualElement>

          <ui:VisualElement name="panel-body">

              <ui:Label text="Brush Type" class="section-label first-section"/>
              <ui:GroupBox>
                  <ui:RadioButton name="brush-texture"  label="Texture" value="true"/>
                  <ui:RadioButton name="brush-height"   label="Height"/>
                  <ui:RadioButton name="brush-combined" label="Combined"/>
              </ui:GroupBox>

              <ui:Label text="Mode" class="section-label"/>
              <ui:GroupBox>
                  <ui:RadioButton name="mode-raise"  label="Raise" value="true"/>
                  <ui:RadioButton name="mode-lower"  label="Lower"/>
                  <ui:RadioButton name="mode-smooth" label="Smooth"/>
              </ui:GroupBox>

              <ui:Label text="Layer" class="section-label"/>
              <ui:GroupBox>
                  <ui:RadioButton name="layer-grass"  label="Grass"  value="true"/>
                  <ui:RadioButton name="layer-dirt"   label="Dirt"/>
                  <ui:RadioButton name="layer-stone"  label="Stone"/>
                  <ui:RadioButton name="layer-ravine" label="Ravine"/>
              </ui:GroupBox>

              <ui:Label text="Radius" class="section-label"/>
              <ui:Slider name="radius-slider" low-value="1" high-value="20" value="5"/>

              <ui:Label text="Strength" class="section-label"/>
              <ui:Slider name="strength-slider" low-value="0.05" high-value="1" value="0.5"/>

          </ui:VisualElement>

      </ui:VisualElement>
  </ui:UXML>
  ```

- [ ] **Step 2: Rewrite `TilePalettePanel.uss`**

  Replace the entire file content with:
  ```css
  .panel {
      position: absolute;
      top: 16px;
      right: 16px;
      width: 260px;
      background-color: rgba(20, 20, 25, 0.92);
      border-radius: 6px;
      padding: 16px;
  }

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
      background-color: rgba(0, 0, 0, 0);
      border-width: 0;
      color: rgb(180, 180, 180);
      font-size: 12px;
      -unity-text-align: middle-center;
  }

  .section-label {
      font-size: 12px;
      color: rgb(160, 160, 180);
      -unity-font-style: bold;
      margin-top: 10px;
      margin-bottom: 4px;
      border-top-width: 1px;
      border-top-color: rgba(255, 255, 255, 0.12);
      padding-top: 8px;
  }

  .section-label.first-section {
      margin-top: 0;
      border-top-width: 0;
      padding-top: 0;
  }

  /* Radio button / toggle label colours */
  .unity-toggle__label {
      color: rgb(200, 200, 200);
  }

  .unity-toggle--checked > .unity-toggle__label {
      color: rgb(140, 200, 255);
  }
  ```

  Note: The old `:first-child` rule is NOT included — it is replaced by `.section-label.first-section`.

- [ ] **Step 3: Add collapse fields and `ToggleCollapse` to `TilePalettePanel.cs`**

  Add three fields after the existing `private Slider _strengthSlider;` declaration:
  ```csharp
  private VisualElement _panelBody;
  private Button        _collapseBtn;
  private bool          _isCollapsed = false;
  ```

  In `OnEnable()`, after `UIHoverTracker.Register(_trackedPanel);`, add:
  ```csharp
  _panelBody   = root.Q<VisualElement>("panel-body");
  _collapseBtn = root.Q<Button>("collapse-btn");
  _collapseBtn.clicked += ToggleCollapse;
  ```

  In `OnDisable()`, add:
  ```csharp
  _collapseBtn.clicked -= ToggleCollapse;
  ```

  Add the method after `OnDisable`:
  ```csharp
  void ToggleCollapse()
  {
      _isCollapsed = !_isCollapsed;
      _panelBody.style.display = _isCollapsed ? DisplayStyle.None : DisplayStyle.Flex;
      _collapseBtn.text = _isCollapsed ? "▸" : "▾";
  }
  ```

- [ ] **Step 4: Verify**

  Enter Play Mode. Confirm:
  - "Unknown pseudo class `first-child`" warning is gone from Console
  - Radio button labels (Texture, Height, Combined, Raise, etc.) are now white/light grey, not black
  - Checked radio buttons show light-blue label text
  - "Brush Panel" title appears with ▾ chevron
  - Clicking chevron collapses/expands the brush controls

- [ ] **Step 5: Commit**

  ```bash
  git add Assets/Scripts/Runtime/TilePalettePanel.cs \
          Assets/UI/TilePalettePanel.uxml \
          Assets/UI/TilePalettePanel.uss
  git commit -m "feat: collapsible Brush Panel; fix radio button text colour; fix :first-child warning"
  ```

---

## Task 5: Entity Palette — reposition + Foldout redesign

**Files:**
- Modify: `Assets/Scripts/Runtime/EntityPalettePanel.cs`
- Modify: `Assets/UI/EntityPalettePanel.uxml`
- Modify: `Assets/UI/EntityPalettePanel.uss`

**Context:** Replace the 2-column 48×48 grid with a grouped list (NPC / Enemy / Prop) using UIToolkit `Foldout` elements. Reposition to avoid overlap with the Brush Panel. This is a coordinated change across all three files — apply them together.

- [ ] **Step 1: Update `EntityPalettePanel.uxml`**

  The grid is populated entirely in C#. The UXML only needs the scroll view height updated. Replace the file content with:
  ```xml
  <ui:UXML xmlns:ui="UnityEngine.UIElements">
      <ui:VisualElement name="panel" class="panel">
          <ui:Label text="Entities" class="panel-title"/>
          <ui:ScrollView name="entity-scroll" class="entity-scroll">
              <ui:VisualElement name="entity-grid" class="entity-grid"/>
          </ui:ScrollView>
      </ui:VisualElement>
  </ui:UXML>
  ```

- [ ] **Step 2: Rewrite `EntityPalettePanel.uss`**

  Replace the entire file content with:
  ```css
  .panel {
      position: absolute;
      bottom: 10px;
      right: 286px;
      width: 180px;
      background-color: rgba(20, 20, 25, 0.92);
      border-radius: 6px;
      padding: 10px;
  }

  .panel-title {
      font-size: 13px;
      color: rgb(220, 200, 140);
      -unity-font-style: bold;
      margin-bottom: 8px;
  }

  .entity-scroll {
      max-height: 320px;
  }

  .entity-grid {
      flex-direction: column;
  }

  /* Foldout group header */
  .entity-group > .unity-foldout__toggle > .unity-toggle__label {
      color: rgb(180, 180, 180);
      font-size: 11px;
      -unity-font-style: bold;
  }

  /* Entity row */
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

  /* 14×14 colour swatch */
  .entity-swatch {
      width: 14px;
      height: 14px;
      border-radius: 2px;
      margin-right: 6px;
      flex-shrink: 0;
  }

  /* Entity name label */
  .entity-row__label {
      font-size: 11px;
      color: rgb(200, 200, 200);
      overflow: hidden;
      white-space: nowrap;
  }
  ```

  Note: `text-overflow: ellipsis` is NOT supported in UIToolkit 2022 runtime — text clips silently instead.

- [ ] **Step 3: Rewrite `BuildGrid()` in `EntityPalettePanel.cs`**

  Replace the entire `BuildGrid()` method with:
  ```csharp
  void BuildGrid()
  {
      if (_entityGrid == null)
      {
          Debug.LogError("[EntityPalettePanel] 'entity-grid' element not found — check EntityPalettePanel.uxml");
          return;
      }
      _entityGrid.Clear();
      _selectedRow = null;

      // Group definitions by EntityType in display order
      var groupOrder = new[] { "NPC", "Enemy", "Prop" };
      foreach (var groupName in groupOrder)
      {
          var foldout = new Foldout { text = groupName, value = true };
          foldout.AddToClassList("entity-group"); // required for USS selector match

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
          }

          _entityGrid.Add(foldout);
      }
  }
  ```

- [ ] **Step 4: Update `SelectEntry()` to track `_selectedRow`**

  The field `_selectedCell` is renamed to `_selectedRow` and the CSS class changes from `entity-cell--selected` to `entity-row--selected`. Update the field declaration:

  Change:
  ```csharp
  private VisualElement _selectedCell;
  ```
  To:
  ```csharp
  private VisualElement _selectedRow;
  ```

  Replace the entire `SelectEntry` method:
  ```csharp
  void SelectEntry(VisualElement row, EntityDefinition def)
  {
      _selectedRow?.RemoveFromClassList("entity-row--selected");

      if (_selectedRow == row)
      {
          // Clicking the already-selected row deselects it
          _selectedRow = null;
          EntityPlacer.SelectedEntry = null;
          return;
      }

      _selectedRow = row;
      _selectedRow.AddToClassList("entity-row--selected");

      EntityPlacer.SelectedEntry     = def;
      TerrainPainter.ActiveBrush     = null;
  }
  ```

- [ ] **Step 5: Update `ClearSelection()` and `OnActiveZoneChanged()`**

  Replace `ClearSelection()`:
  ```csharp
  public static void ClearSelection()
  {
      if (_instance == null) return;
      _instance._selectedRow?.RemoveFromClassList("entity-row--selected");
      _instance._selectedRow = null;
      EntityPlacer.SelectedEntry = null;
  }
  ```

  Replace `OnActiveZoneChanged()`:
  ```csharp
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
  ```

- [ ] **Step 6: Verify**

  Enter Play Mode. Confirm:
  - Entity palette no longer overlaps the Brush Panel
  - Entities are grouped under NPC / Enemy / Prop foldout headers
  - Each group is expandable/collapsible by clicking the header
  - Clicking an entity row selects it (left-border accent appears), clears the brush selection
  - Clicking the same row deselects it
  - Selecting a brush (in the Brush Panel) clears the entity selection

- [ ] **Step 7: Commit**

  ```bash
  git add Assets/Scripts/Runtime/EntityPalettePanel.cs \
          Assets/UI/EntityPalettePanel.uxml \
          Assets/UI/EntityPalettePanel.uss
  git commit -m "feat: entity palette grouped list with Foldouts; reposition to avoid brush panel"
  ```

---

## Final verification checklist

After all five tasks are complete and in Play Mode:

- [ ] Existing zones appear in Zone Manager on startup (no need to create a new zone)
- [ ] Clicking a zone activates it and collapses the Zone Manager to its title bar
- [ ] Chevron button re-expands the Zone Manager
- [ ] Text in Name / Width / Height fields is not clipped
- [ ] No "Runtime cursors" warning in Console
- [ ] No "Unknown pseudo class `first-child`" warning in Console
- [ ] Brush Panel shows title + collapse chevron; radio button text is readable (white)
- [ ] Clicking the Brush Panel chevron collapses/expands the brush controls
- [ ] Entity palette is to the left of the Brush Panel, not overlapping
- [ ] Entity palette shows NPC / Enemy / Prop foldout groups
- [ ] Selecting an entity and clicking on terrain spawns the entity (visible as a coloured cube)
- [ ] Selecting a brush and clicking terrain modifies height/texture (hover ring appears)
- [ ] `spacetime logs zoneforge-server` shows reducer calls for both terrain updates and entity spawns
