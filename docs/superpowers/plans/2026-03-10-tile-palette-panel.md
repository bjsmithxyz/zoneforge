# Tile Palette Panel Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a bottom-center hotbar UIToolkit panel to `zoneforge-editor` that lets the designer select the active tile type, exposing `TilePalettePanel.SelectedTile` for the raycast painter to consume.

**Architecture:** Separate `UIDocument` + UXML following the exact same pattern as `ZoneCreationPanel`. Five hardcoded tile types with runtime-generated solid-colour `Texture2D` thumbnails. Selection state is a static property; no server interaction.

**Tech Stack:** Unity 2022.3 LTS, C#, UIToolkit (UIElements), `UnityEngine.UIElements`

**Spec:** `docs/superpowers/specs/2026-03-10-tile-palette-panel-design.md`

---

## Chunk 1: Tile Palette Panel

### File Structure

| Action | Path | Responsibility |
|---|---|---|
| Create | `editor/Assets/UI/TilePalettePanel.uxml` | Static layout shell — hotbar container + active label |
| Create | `editor/Assets/UI/TilePalettePanel.uss` | Positioning, slot sizing, selection highlight |
| Create | `editor/Assets/Scripts/Runtime/TilePalettePanel.cs` | Tile data, thumbnail generation, slot building, selection state |

---

### Task 1: UXML layout

**Files:**
- Create: `editor/Assets/UI/TilePalettePanel.uxml`

The UXML only defines the shell. Slots are injected by `TilePalettePanel.cs` at runtime so adding tile types later requires no UXML edits.

- [ ] **Step 1: Create the UXML file**

Create `editor/Assets/UI/TilePalettePanel.uxml` with the stylesheet reference included from the start (USS must exist before Play mode — create the USS file in Task 2 before entering Play):

```xml
<ui:UXML xmlns:ui="UnityEngine.UIElements" editor-extension-mode="False">
    <Style src="TilePalettePanel.uss" />
    <ui:VisualElement name="hotbar-root" class="hotbar-root">
        <ui:Label name="active-label" text="Painting: Grass" class="active-label" />
        <ui:VisualElement name="hotbar-slots" class="hotbar-slots" />
    </ui:VisualElement>
</ui:UXML>
```

- [ ] **Step 2: Commit**

```bash
cd /home/beek/zoneforge/editor
git add Assets/UI/TilePalettePanel.uxml Assets/UI/TilePalettePanel.uxml.meta
git commit -m "feat(editor): add TilePalettePanel UXML shell"
```

---

### Task 2: USS styles

**Files:**
- Create: `editor/Assets/UI/TilePalettePanel.uss`

- [ ] **Step 1: Create the USS file**

Create `editor/Assets/UI/TilePalettePanel.uss`:

```css
.hotbar-root {
    position: absolute;
    bottom: 16px;
    left: 50%;
    translate: -50% 0;
    align-items: center;
}

.active-label {
    font-size: 11px;
    color: rgb(200, 200, 210);
    margin-bottom: 4px;
    -unity-text-align: middle-center;
}

.hotbar-slots {
    flex-direction: row;
    background-color: rgba(20, 20, 25, 0.90);
    border-radius: 6px;
    padding: 6px 8px;
    flex-wrap: nowrap;
}

.tile-slot {
    width: 52px;
    align-items: center;
    padding: 3px;
    border-radius: 4px;
    border-width: 2px;
    border-color: rgba(0, 0, 0, 0);
    margin: 0 2px;
    cursor: link;
}

.tile-slot:hover {
    background-color: rgba(255, 255, 255, 0.08);
}

.tile-slot--selected {
    border-color: rgb(220, 200, 140);
    background-color: rgba(220, 200, 140, 0.12);
}

.tile-swatch {
    width: 36px;
    height: 36px;
    border-radius: 3px;
}

.tile-label {
    font-size: 10px;
    color: rgb(180, 180, 190);
    margin-top: 2px;
    -unity-text-align: middle-center;
}

.tile-slot--selected .tile-label {
    color: rgb(220, 200, 140);
}
```

- [ ] **Step 2: Commit**

```bash
git add Assets/UI/TilePalettePanel.uss Assets/UI/TilePalettePanel.uss.meta
git commit -m "feat(editor): add TilePalettePanel USS styles"
```

---

### Task 3: TilePalettePanel MonoBehaviour

**Files:**
- Create: `editor/Assets/Scripts/Runtime/TilePalettePanel.cs`

This is the only C# file. It owns: tile definitions, thumbnail generation, slot building, selection state, and texture cleanup.

- [ ] **Step 1: Create TilePalettePanel.cs**

Create `editor/Assets/Scripts/Runtime/TilePalettePanel.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UIElements;

[RequireComponent(typeof(UIDocument))]
public class TilePalettePanel : MonoBehaviour
{
    // --- Public API consumed by the raycast tile painter ---
    public static string SelectedTile { get; private set; } = "Grass";

    // --- Tile definitions ---
    private static readonly (string name, Color color)[] Tiles =
    {
        ("Grass", new Color(0.30f, 0.60f, 0.10f)),
        ("Dirt",  new Color(0.55f, 0.35f, 0.17f)),
        ("Stone", new Color(0.50f, 0.50f, 0.50f)),
        ("Water", new Color(0.25f, 0.50f, 1.00f)),
        ("Wall",  new Color(0.31f, 0.24f, 0.16f)),
    };

    // --- Internal state ---
    private Label _activeLabel;
    private VisualElement _slotsContainer;
    private VisualElement _selectedSlot;
    private readonly List<Texture2D> _swatchTextures = new();

    // ----------------------------------------------------------------

    void OnEnable()
    {
        var root = GetComponent<UIDocument>().rootVisualElement;
        _activeLabel   = root.Q<Label>("active-label");
        _slotsContainer = root.Q<VisualElement>("hotbar-slots");

        BuildHotbar();
    }

    void OnDisable()
    {
        foreach (var tex in _swatchTextures)
            if (tex != null) Destroy(tex);
        _swatchTextures.Clear();
    }

    // ----------------------------------------------------------------

    void BuildHotbar()
    {
        _slotsContainer.Clear();
        _selectedSlot = null;

        foreach (var (name, color) in Tiles)
        {
            var slot = CreateSlot(name, color);
            _slotsContainer.Add(slot);

            if (name == SelectedTile)
                ApplySelection(slot, name);
        }
    }

    VisualElement CreateSlot(string tileName, Color color)
    {
        var slot = new VisualElement { name = $"slot-{tileName}" };
        slot.AddToClassList("tile-slot");

        var swatch = new VisualElement();
        swatch.AddToClassList("tile-swatch");
        swatch.style.backgroundImage = new StyleBackground(MakeSwatch(color));
        slot.Add(swatch);

        var label = new Label(tileName);
        label.AddToClassList("tile-label");
        slot.Add(label);

        slot.RegisterCallback<ClickEvent>(_ => OnSlotClicked(slot, tileName));
        return slot;
    }

    void OnSlotClicked(VisualElement slot, string tileName)
    {
        _selectedSlot?.RemoveFromClassList("tile-slot--selected");
        ApplySelection(slot, tileName);
    }

    void ApplySelection(VisualElement slot, string tileName)
    {
        _selectedSlot = slot;
        _selectedSlot.AddToClassList("tile-slot--selected");
        SelectedTile = tileName;
        _activeLabel.text = $"Painting: {tileName}";
    }

    Texture2D MakeSwatch(Color color)
    {
        const int size = 32;
        var tex = new Texture2D(size, size, TextureFormat.RGBA32, false);
        var pixels = new Color[size * size];
        for (int i = 0; i < pixels.Length; i++)
            pixels[i] = color;
        tex.SetPixels(pixels);
        tex.Apply();
        _swatchTextures.Add(tex);
        return tex;
    }
}
```

- [ ] **Step 2: Commit**

```bash
git add Assets/Scripts/Runtime/TilePalettePanel.cs Assets/Scripts/Runtime/TilePalettePanel.cs.meta
git commit -m "feat(editor): add TilePalettePanel MonoBehaviour"
```

---

### Task 4: Maintenance commit

- [ ] **Step 1: Add .gitignore entry for brainstorm outputs**

```bash
# From /home/beek/zoneforge (umbrella repo root)
echo ".superpowers/" >> .gitignore
git add .gitignore
git commit -m "chore: ignore .superpowers/ brainstorm output dir"
```

---

### Task 5: Scene wiring and verification

This task is done inside Unity — no code to write.

**Prerequisites:** Unity editor open with `SampleScene`, SpacetimeDB SDK package resolved.

- [ ] **Step 1: Create a PanelSettings asset for the hotbar**

Each UIDocument can share a PanelSettings or have its own — give the hotbar its own so its sort order is independent.

In Unity Project window:
- Right-click `Assets/Settings` → **Create → UI Toolkit → Panel Settings**
- Name it `TilePalettePanelSettings`

- [ ] **Step 2: Create scene GameObject**

In the Hierarchy:
- Right-click → **Create Empty**, name it `TilePaletteHotbar`
- Add component: **UI Document**
- Set **Panel Settings** → `TilePalettePanelSettings`
- Set **Source Asset** → `Assets/UI/TilePalettePanel.uxml`
- Add component: **Tile Palette Panel** (script)

- [ ] **Step 3: Enter Play mode and verify**

Expected behaviour:

1. Hotbar appears bottom-centre with five coloured slots: Grass, Dirt, Stone, Water, Wall
2. Grass slot starts highlighted (gold border), label reads "Painting: Grass"
3. Clicking **Stone** → border moves to Stone, label reads "Painting: Stone"
4. Clicking **Water** → border moves to Water, label reads "Painting: Water"
5. No errors in the Unity Console

To confirm `SelectedTile` updates: temporarily add `Debug.Log(TilePalettePanel.SelectedTile);` inside `OnSlotClicked`, enter Play, click tiles, confirm Console output, then remove the log line before committing.

- [ ] **Step 4: Commit scene and generated assets**

```bash
cd /home/beek/zoneforge/editor
git add Assets/Scenes/SampleScene.unity
git add Assets/Settings/TilePalettePanelSettings.asset
git add Assets/Settings/TilePalettePanelSettings.asset.meta
git commit -m "feat(editor): add TilePaletteHotbar to SampleScene"
```

---

## Done

`TilePalettePanel.SelectedTile` is live. The raycast tile painter (item #3) can now read it without any further wiring.
