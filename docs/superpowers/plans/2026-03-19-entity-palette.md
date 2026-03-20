# Entity Palette Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an entity palette panel to the ZoneForge editor that lets designers click to select an entity type and stamp instances into the active zone via the `spawn_entity` reducer, with placeholder cubes rendered in real time.

**Architecture:** Four new MonoBehaviours (`EntityDefinition` plain class, `EntityPalettePanel`, `EntityPlacer`, `EntityRenderer`) plus UXML/USS assets for the panel UI. Two existing files are modified: `SpacetimeDBManager` gains an `OnEntityDeleted` event, and `TilePalettePanel` clears `EntityPlacer.SelectedEntry` when a terrain brush is selected to enforce mutual exclusion. `EntityPalettePanel` owns the static definition list and `ByPrefabName` lookup dictionary used by `EntityRenderer`.

**Tech Stack:** Unity 2022.3 LTS, C#, Unity UI Toolkit (UIElements), SpacetimeDB C# SDK, NUnit (Unity Edit Mode tests)

---

## File Map

| Action | Path | Responsibility |
| --- | --- | --- |
| Create | `editor/Assets/Scripts/Runtime/EntityDefinition.cs` | Plain C# data record: `DisplayName`, `PrefabName`, `EntityType`, `Color` |
| Create | `editor/Assets/Scripts/Runtime/EntityPlacer.cs` | Click-to-place: reads `SelectedEntry`, raycasts terrain collider, calls `SpawnEntity` reducer |
| Create | `editor/Assets/Scripts/Runtime/EntityPalettePanel.cs` | UIToolkit panel: builds grid, owns `ByPrefabName` static dict, sets `SelectedEntry` on click |
| Create | `editor/Assets/Scripts/Runtime/EntityRenderer.cs` | Spawns/destroys placeholder cubes in response to `OnEntityInserted`/`OnEntityDeleted` |
| Create | `editor/Assets/UI/EntityPalettePanel.uxml` | Panel layout: title + scrollable entity grid |
| Create | `editor/Assets/UI/EntityPalettePanel.uss` | Panel styles: bottom-right position, grid layout, selection highlight |
| Create | `editor/Assets/Tests/EditMode/EntityPalettePanelTests.cs` | Edit Mode tests for static definition dictionary |
| Modify | `editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs` | Add `OnEntityDeleted` static event and `OnDelete` callback |
| Modify | `editor/Assets/Scripts/Runtime/TilePalettePanel.cs` | Clear `EntityPlacer.SelectedEntry` in `UpdateBrush()` |

---

## Task 1: `EntityDefinition` Data Class

**Files:**
- Create: `editor/Assets/Scripts/Runtime/EntityDefinition.cs`

**Context:** `EntityDefinition` is a plain C# class (not a MonoBehaviour). The Edit Mode tests for this data are written in Task 6 alongside `EntityPalettePanel`, which owns the static definition list and `ByPrefabName` dictionary.

- [ ] **Step 1: Create `EntityDefinition.cs`**

```csharp
// editor/Assets/Scripts/Runtime/EntityDefinition.cs
using UnityEngine;

public class EntityDefinition
{
    public string DisplayName;
    public string PrefabName;
    public string EntityType;
    public Color  Color;
}
```

- [ ] **Step 2: Verify compile — open Unity, check Console for errors**

Expected: no errors. `EntityDefinition` is a plain class with no dependencies beyond `UnityEngine`.

- [ ] **Step 3: Commit**

```bash
cd /home/beek/zoneforge
git add editor/Assets/Scripts/Runtime/EntityDefinition.cs \
        editor/Assets/Scripts/Runtime/EntityDefinition.cs.meta
git commit -m "feat(editor): add EntityDefinition data class"
```

---

## Task 2: `EntityPlacer` MonoBehaviour

**Files:**
- Create: `editor/Assets/Scripts/Runtime/EntityPlacer.cs`

**Context:** `EntityPlacer` must exist before `TilePalettePanel` is modified (Task 4) because `UpdateBrush()` will reference `EntityPlacer.SelectedEntry`. The `_terrainCollider` Inspector reference is wired in Task 8 (scene setup). `Conn` may be null during startup; the null guard on the reducer call handles that.

- [ ] **Step 1: Create `EntityPlacer.cs`**

```csharp
// editor/Assets/Scripts/Runtime/EntityPlacer.cs
using UnityEngine;

public class EntityPlacer : MonoBehaviour
{
    [SerializeField] MeshCollider _terrainCollider;
    [SerializeField] Camera       _camera;

    public static EntityDefinition SelectedEntry { get; set; }

    void Awake()
    {
        if (_camera == null) _camera = Camera.main;
    }

    void Update()
    {
        if (!EditorState.HasActiveZone)            return;
        if (SelectedEntry == null)                 return;
        if (UIHoverTracker.IsPointerOverUI)        return;
        if (!Input.GetMouseButtonDown(0))          return;
        if (SpacetimeDBManager.Conn == null)       return;
        if (_terrainCollider == null)              return;

        var ray = _camera.ScreenPointToRay(Input.mousePosition);
        if (!_terrainCollider.Raycast(ray, out RaycastHit hit, 1000f)) return;

        SpacetimeDBManager.Conn.Reducers.SpawnEntity(
            EditorState.ActiveZoneId,
            SelectedEntry.PrefabName,
            hit.point.x,   // x
            hit.point.z,   // y (reducer param name — world Z is the depth axis)
            hit.point.y,   // elevation (world Y)
            SelectedEntry.EntityType
        );
    }
}
```

- [ ] **Step 2: Verify compile — check Unity Console for errors**

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add editor/Assets/Scripts/Runtime/EntityPlacer.cs \
        editor/Assets/Scripts/Runtime/EntityPlacer.cs.meta
git commit -m "feat(editor): add EntityPlacer click-to-place MonoBehaviour"
```

---

## Task 3: `SpacetimeDBManager` — Add `OnEntityDeleted`

**Files:**
- Modify: `editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs`

**Context:** `EntityRenderer` (Task 7) subscribes to this event to destroy placeholder cubes when entities are deleted from the server. The `OnDelete` callback delegate signature is `(EventContext, EntityInstance)` — same pattern as the existing `OnInsert` callback at line 73.

- [ ] **Step 1: Add the `OnEntityDeleted` event declaration after line 17**

In `SpacetimeDBManager.cs`, after:
```csharp
public static event Action<EntityInstance> OnEntityInserted;
```
Add:
```csharp
public static event Action<EntityInstance> OnEntityDeleted;
```

- [ ] **Step 2: Register the `OnDelete` callback after line 73**

In `OnSubscriptionApplied`, after:
```csharp
Conn.Db.EntityInstance.OnInsert += (eventCtx, entity) => OnEntityInserted?.Invoke(entity);
```
Add:
```csharp
Conn.Db.EntityInstance.OnDelete += (eventCtx, entity) => OnEntityDeleted?.Invoke(entity);
```

- [ ] **Step 3: Verify compile — check Unity Console**

Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs
git commit -m "feat(editor): add OnEntityDeleted event to SpacetimeDBManager"
```

---

## Task 4: `TilePalettePanel` — Mutual Exclusion

**Files:**
- Modify: `editor/Assets/Scripts/Runtime/TilePalettePanel.cs`

**Context:** Without this change, selecting a terrain brush while an entity is selected leaves both `TerrainPainter.ActiveBrush` and `EntityPlacer.SelectedEntry` non-null, causing both to fire on the same mouse click. `UpdateBrush()` runs once on `OnEnable` to set the initial active brush — this means entity placement starts inactive by default, which is the correct startup behaviour.

- [ ] **Step 1: Add mutual exclusion to `UpdateBrush()`**

In `TilePalettePanel.cs`, find the last line of `UpdateBrush()`:
```csharp
        TerrainPainter.ActiveBrush = brush;
```
Change it to:
```csharp
        TerrainPainter.ActiveBrush = brush;
        EntityPlacer.SelectedEntry = null;
```

- [ ] **Step 2: Verify compile — check Unity Console**

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add editor/Assets/Scripts/Runtime/TilePalettePanel.cs
git commit -m "feat(editor): clear EntityPlacer.SelectedEntry when terrain brush selected"
```

---

## Task 5: Entity Palette Panel UI Assets

**Files:**
- Create: `editor/Assets/UI/EntityPalettePanel.uxml`
- Create: `editor/Assets/UI/EntityPalettePanel.uss`

**Context:** The grid cells (colored thumbnails + labels) are created programmatically in `EntityPalettePanel.cs` — UXML only provides the outer shell. The panel is anchored bottom-right. Panel width of 148px accommodates two 48px cells with 3px margin each (2 × 54px = 108px) plus 20px horizontal padding. The USS `entity-cell--selected` class adds a white border for selection state; it is toggled by C# code, not UXML.

- [ ] **Step 1: Create `EntityPalettePanel.uxml`**

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

- [ ] **Step 2: Create `EntityPalettePanel.uss`**

```css
.panel {
    position: absolute;
    bottom: 10px;
    right: 10px;
    width: 148px;
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
    max-height: 280px;
}

.entity-grid {
    flex-direction: row;
    flex-wrap: wrap;
}

.entity-cell {
    width: 48px;
    margin: 3px;
    align-items: center;
    cursor: link;
    border-width: 2px;
    border-color: rgba(0, 0, 0, 0);
    border-radius: 2px;
}

.entity-cell--selected {
    border-color: rgb(255, 255, 255);
}

.entity-cell__image {
    width: 48px;
    height: 48px;
}

.entity-cell__label {
    font-size: 9px;
    color: rgb(200, 200, 200);
    -unity-text-align: middle-center;
    white-space: normal;
    overflow: hidden;
}
```

- [ ] **Step 3: Verify assets appear in the Unity Project window under `Assets/UI/`**

Unity auto-generates `.meta` files for these assets on import.

- [ ] **Step 4: Commit**

```bash
git add editor/Assets/UI/EntityPalettePanel.uxml \
        editor/Assets/UI/EntityPalettePanel.uxml.meta \
        editor/Assets/UI/EntityPalettePanel.uss \
        editor/Assets/UI/EntityPalettePanel.uss.meta
git commit -m "feat(editor): add EntityPalettePanel UXML and USS assets"
```

---

## Task 6: `EntityPalettePanel` MonoBehaviour + Edit Mode Tests

**Files:**
- Create: `editor/Assets/Scripts/Runtime/EntityPalettePanel.cs`
- Create: `editor/Assets/Tests/EditMode/EntityPalettePanelTests.cs`

**Context:** `ByPrefabName` is populated in a static constructor — it runs the first time the class is accessed, with no scene required. This makes it testable from Unity Edit Mode tests. `SelectEntry` toggles selection: clicking a selected cell deselects it (both panel highlight and `SelectedEntry` cleared). `OnActiveZoneChanged` drives `SetEnabled(false)` on the panel root when no zone is active, blocking all pointer events via UIToolkit's built-in disabled handling. The `Image` element in each cell displays a 1×1 `Texture2D` scaled up, which renders as a solid colour square.

- [ ] **Step 1: Write the failing Edit Mode test**

Create `editor/Assets/Tests/EditMode/EntityPalettePanelTests.cs`:

```csharp
using NUnit.Framework;
using UnityEngine;

public class EntityPalettePanelTests
{
    [Test]
    public void ByPrefabName_ContainsAllEightDefinitions()
    {
        Assert.AreEqual(8, EntityPalettePanel.ByPrefabName.Count);
    }

    [Test]
    public void ByPrefabName_AllKeysMatchPrefabNameField()
    {
        foreach (var kv in EntityPalettePanel.ByPrefabName)
        {
            Assert.AreEqual(kv.Key, kv.Value.PrefabName,
                $"Dictionary key '{kv.Key}' does not match PrefabName '{kv.Value.PrefabName}'");
        }
    }

    [Test]
    public void ByPrefabName_AllEntriesHaveNonEmptyFields()
    {
        foreach (var kv in EntityPalettePanel.ByPrefabName)
        {
            var def = kv.Value;
            Assert.IsNotEmpty(def.DisplayName, $"DisplayName empty for '{kv.Key}'");
            Assert.IsNotEmpty(def.PrefabName,  $"PrefabName empty for '{kv.Key}'");
            Assert.IsNotEmpty(def.EntityType,  $"EntityType empty for '{kv.Key}'");
        }
    }

    [Test]
    public void ByPrefabName_GuardIsNpcBlue()
    {
        Assert.IsTrue(EntityPalettePanel.ByPrefabName.ContainsKey("guard"));
        var def = EntityPalettePanel.ByPrefabName["guard"];
        Assert.AreEqual("Guard", def.DisplayName);
        Assert.AreEqual("NPC",   def.EntityType);
        Assert.AreEqual(Color.blue, def.Color);
    }

    [Test]
    public void ByPrefabName_MerchantIsNpcCyan()
    {
        Assert.IsTrue(EntityPalettePanel.ByPrefabName.ContainsKey("merchant"));
        var def = EntityPalettePanel.ByPrefabName["merchant"];
        Assert.AreEqual("NPC", def.EntityType);
        Assert.AreEqual(Color.cyan, def.Color);
    }

    [Test]
    public void ByPrefabName_EntityTypesAreCorrect()
    {
        Assert.AreEqual("NPC",   EntityPalettePanel.ByPrefabName["guard"].EntityType);
        Assert.AreEqual("NPC",   EntityPalettePanel.ByPrefabName["merchant"].EntityType);
        Assert.AreEqual("Enemy", EntityPalettePanel.ByPrefabName["skeleton"].EntityType);
        Assert.AreEqual("Enemy", EntityPalettePanel.ByPrefabName["goblin"].EntityType);
        Assert.AreEqual("Prop",  EntityPalettePanel.ByPrefabName["tree"].EntityType);
        Assert.AreEqual("Prop",  EntityPalettePanel.ByPrefabName["rock"].EntityType);
        Assert.AreEqual("Prop",  EntityPalettePanel.ByPrefabName["chest"].EntityType);
        Assert.AreEqual("Prop",  EntityPalettePanel.ByPrefabName["wall"].EntityType);
    }
}
```

- [ ] **Step 2: Run the failing test in Unity**

Open **Window → General → Test Runner → Edit Mode tab → Run All**.

Expected: 6 tests fail with `NullReferenceException` or "type not found" because `EntityPalettePanel` does not exist yet.

- [ ] **Step 3: Create `EntityPalettePanel.cs`**

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
        foreach (var def in Definitions)
            ByPrefabName[def.PrefabName] = def;
    }

    // -----------------------------------------------------------------------
    // Instance state

    private VisualElement _panelRoot;
    private VisualElement _entityGrid;
    private VisualElement _selectedCell;
    private VisualElement _trackedPanel;

    // -----------------------------------------------------------------------

    void OnEnable()
    {
        var root = GetComponent<UIDocument>().rootVisualElement;

        if (_styleSheet != null)
            root.styleSheets.Add(_styleSheet);

        _panelRoot  = root.Q<VisualElement>("panel");
        _entityGrid = root.Q<VisualElement>("entity-grid");

        BuildGrid();

        EditorState.OnActiveZoneChanged += OnActiveZoneChanged;

        _trackedPanel = _panelRoot ?? root;
        UIHoverTracker.Register(_trackedPanel);

        // Reflect current zone state immediately
        _panelRoot?.SetEnabled(EditorState.HasActiveZone);
    }

    void OnDisable()
    {
        EditorState.OnActiveZoneChanged -= OnActiveZoneChanged;
        UIHoverTracker.Unregister(_trackedPanel);
    }

    // -----------------------------------------------------------------------

    void BuildGrid()
    {
        _entityGrid.Clear();
        _selectedCell = null;

        foreach (var def in Definitions)
        {
            var cell = new VisualElement();
            cell.AddToClassList("entity-cell");

            var img = new Image { image = MakeColorTexture(def.Color) };
            img.AddToClassList("entity-cell__image");
            cell.Add(img);

            var lbl = new Label(def.DisplayName);
            lbl.AddToClassList("entity-cell__label");
            cell.Add(lbl);

            var captured = def;
            cell.RegisterCallback<ClickEvent>(_ => SelectEntry(cell, captured));

            _entityGrid.Add(cell);
        }
    }

    void SelectEntry(VisualElement cell, EntityDefinition def)
    {
        _selectedCell?.RemoveFromClassList("entity-cell--selected");

        if (_selectedCell == cell)
        {
            // Clicking the already-selected cell deselects it
            _selectedCell = null;
            EntityPlacer.SelectedEntry = null;
            return;
        }

        _selectedCell = cell;
        _selectedCell.AddToClassList("entity-cell--selected");

        EntityPlacer.SelectedEntry     = def;
        TerrainPainter.ActiveBrush     = null;
    }

    void OnActiveZoneChanged(ulong zoneId)
    {
        _panelRoot?.SetEnabled(EditorState.HasActiveZone);

        if (!EditorState.HasActiveZone)
        {
            _selectedCell?.RemoveFromClassList("entity-cell--selected");
            _selectedCell             = null;
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

- [ ] **Step 4: Run the tests — verify they pass**

Open **Window → General → Test Runner → Edit Mode tab → Run All**.

Expected: all 6 `EntityPalettePanelTests` pass. If the `TerrainPainter` or `EntityPlacer` static references cause a compile error in Edit Mode context, those are resolved by the references existing in the same assembly (`ZoneForgeRuntime`).

- [ ] **Step 5: Commit**

```bash
git add editor/Assets/Scripts/Runtime/EntityPalettePanel.cs \
        editor/Assets/Scripts/Runtime/EntityPalettePanel.cs.meta \
        editor/Assets/Tests/EditMode/EntityPalettePanelTests.cs \
        editor/Assets/Tests/EditMode/EntityPalettePanelTests.cs.meta
git commit -m "feat(editor): add EntityPalettePanel MonoBehaviour and Edit Mode tests"
```

---

## Task 7: `EntityRenderer` MonoBehaviour

**Files:**
- Create: `editor/Assets/Scripts/Runtime/EntityRenderer.cs`

**Context:** `EntityRenderer` subscribes to `SpacetimeDBManager.OnEntityInserted` and `OnEntityDeleted` (added in Task 3). On `OnActiveZoneChanged`, it clears all cubes and re-populates from the server cache for the new zone. `GameObject.CreatePrimitive(PrimitiveType.Cube)` creates cubes with a `BoxCollider` by default — this collider is removed immediately since entity cubes must not interfere with terrain-collider raycasts from `EntityPlacer`. Each cube gets its own `Material` instance to prevent shared-material mutation (following the same pattern as `TerrainPainter`'s hover ring). Cubes are 0.5 units on each side. Position mapping: `(entity.PositionX, entity.Elevation, entity.PositionY)` → world `(X, Y, Z)`.

- [ ] **Step 1: Create `EntityRenderer.cs`**

```csharp
// editor/Assets/Scripts/Runtime/EntityRenderer.cs
using System.Collections.Generic;
using UnityEngine;
using SpacetimeDB.Types;

public class EntityRenderer : MonoBehaviour
{
    private readonly Dictionary<ulong, GameObject> _cubes = new();

    // -----------------------------------------------------------------------

    void OnEnable()
    {
        SpacetimeDBManager.OnEntityInserted  += OnEntityInserted;
        SpacetimeDBManager.OnEntityDeleted   += OnEntityDeleted;
        EditorState.OnActiveZoneChanged      += OnActiveZoneChanged;
    }

    void OnDisable()
    {
        SpacetimeDBManager.OnEntityInserted  -= OnEntityInserted;
        SpacetimeDBManager.OnEntityDeleted   -= OnEntityDeleted;
        EditorState.OnActiveZoneChanged      -= OnActiveZoneChanged;
    }

    // -----------------------------------------------------------------------

    void OnEntityInserted(EntityInstance entity)
    {
        if (entity.ZoneId != EditorState.ActiveZoneId) return;
        SpawnCube(entity);
    }

    void OnEntityDeleted(EntityInstance entity)
    {
        if (_cubes.TryGetValue(entity.Id, out var go))
        {
            Destroy(go);
            _cubes.Remove(entity.Id);
        }
    }

    void OnActiveZoneChanged(ulong zoneId)
    {
        ClearAll();

        if (SpacetimeDBManager.Conn == null) return;

        foreach (var entity in SpacetimeDBManager.Conn.Db.EntityInstance.Iter())
        {
            if (entity.ZoneId == zoneId)
                SpawnCube(entity);
        }
    }

    // -----------------------------------------------------------------------

    void SpawnCube(EntityInstance entity)
    {
        var go = GameObject.CreatePrimitive(PrimitiveType.Cube);
        go.transform.localScale = Vector3.one * 0.5f;
        go.transform.position   = new Vector3(entity.PositionX, entity.Elevation, entity.PositionY);

        // Look up colour; fall back to white with a warning if the prefab is unknown
        Color color = Color.white;
        if (EntityPalettePanel.ByPrefabName.TryGetValue(entity.PrefabName, out var def))
            color = def.Color;
        else
            Debug.LogWarning($"[EntityRenderer] Unknown prefab '{entity.PrefabName}' — rendering white");

        // Own material instance to avoid shared-material mutation
        var mr = go.GetComponent<MeshRenderer>();
        mr.material = new Material(Shader.Find("Universal Render Pipeline/Unlit")) { color = color };

        // Remove BoxCollider — terrain raycasts use MeshCollider.Raycast() directly
        // so this collider is never needed and would only add overhead
        Destroy(go.GetComponent<Collider>());

        _cubes[entity.Id] = go;
    }

    void ClearAll()
    {
        foreach (var go in _cubes.Values)
            if (go != null) Destroy(go);
        _cubes.Clear();
    }
}
```

- [ ] **Step 2: Verify compile — check Unity Console for errors**

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add editor/Assets/Scripts/Runtime/EntityRenderer.cs \
        editor/Assets/Scripts/Runtime/EntityRenderer.cs.meta
git commit -m "feat(editor): add EntityRenderer placeholder cube spawner"
```

---

## Task 8: Scene Wiring

**Files:**
- Modify: `editor/Assets/Scenes/SampleScene.unity` (via Unity Inspector)

**Context:** All three new MonoBehaviours need GameObjects in the scene. `EntityPalettePanel` also needs a `UIDocument` component (same pattern as `TilePalettePanel` and `ZoneCreationPanel`). The `EntityPlacer._terrainCollider` must be assigned to the same `MeshCollider` that `TerrainPainter` uses — it lives on the `TerrainRenderer` GameObject. The UIDocument Sort Order for `EntityPalettePanel` is 2 (above the brush panel at 1, which itself is above the zone creation panel at 0).

**Steps are performed in the Unity Editor, not via bash.**

- [ ] **Step 1: Create the `EntityPalettePanel` GameObject**

1. In the Hierarchy, right-click → Create Empty → rename to `EntityPalettePanel`
2. Add Component → `UIDocument` → set **Sort Order = 2**
3. Add Component → `EntityPalettePanel`
4. In the `EntityPalettePanel` component Inspector, assign **Style Sheet** → drag `EntityPalettePanel.uss` from `Assets/UI/`

- [ ] **Step 2: Create the `EntityPlacer` GameObject**

1. In the Hierarchy, right-click → Create Empty → rename to `EntityPlacer`
2. Add Component → `EntityPlacer`
3. In the `EntityPlacer` component Inspector:
   - **Terrain Collider** → drag the `MeshCollider` from the `TerrainRenderer` GameObject (the same collider used by `TerrainPainter`)
   - **Camera** → leave blank (defaults to `Camera.main` in `Awake`)

- [ ] **Step 3: Create the `EntityRenderer` GameObject**

1. In the Hierarchy, right-click → Create Empty → rename to `EntityRenderer`
2. Add Component → `EntityRenderer`
3. No additional Inspector setup required

- [ ] **Step 4: Enter Play Mode — smoke test**

Verify the following:

1. **Panel visible** — an "Entities" panel appears bottom-right with 8 coloured cells in a 2-column grid
2. **Panel disabled without zone** — before selecting a zone, all cells are grayed out and non-interactive
3. **Panel enabled with zone** — after clicking a zone in the Zone Manager panel, cells become interactive
4. **Selection highlight** — clicking a cell adds a white border; clicking a different cell moves the highlight; clicking the same cell again deselects
5. **Mutual exclusion** — selecting an entity cell clears the terrain brush (hover ring disappears); selecting a terrain brush clears the entity selection (cell unhighlights)
6. **Placement** — with an entity selected, left-click on the terrain places a coloured cube at the click point (Console shows no errors)
7. **Real-time sync** — the placed cube persists after deselecting and reselecting the zone (loaded from `EntityInstance.Iter()`)

- [ ] **Step 5: Commit**

```bash
git add editor/Assets/Scenes/SampleScene.unity
git commit -m "feat(editor): wire EntityPalettePanel, EntityPlacer, EntityRenderer in scene"
```

---

## Task 9: Update PROGRESS.md

**Files:**
- Modify: `docs/design/PROGRESS.md`

- [ ] **Step 1: Mark Group 5 entity tasks complete**

In `docs/design/PROGRESS.md`, change the remaining Group 5 checkboxes from `[ ]` to `[x]`:

```markdown
- [x] Entity palette panel with sprite thumbnails
- [x] Click-to-place entity placement (calls `spawn_entity` reducer)
- [x] Subscribe to `EntityInstance` table (OnInsert callback)
- [x] Real-time entity spawning visible in both editor and game client
```

- [ ] **Step 2: Update "Current Focus" section**

Change the "Next:" line to:
```markdown
**Next:** Phase 2 Group 6 — Player movement
```

- [ ] **Step 3: Commit**

```bash
git add docs/design/PROGRESS.md
git commit -m "docs: mark Group 5 entity palette tasks complete"
```
