# Tile Palette Panel — Design Spec

**Date:** 2026-03-10
**Feature:** Group 5 item #2 — 3D tile palette panel with prefab thumbnails
**Project:** zoneforge-editor (Unity standalone runtime)

---

## Overview

A bottom-center hotbar UIToolkit panel that lets the designer select the active tile type before painting. Always visible. Selection state is exposed as a static property consumed by the raycast tile painter (item #3).

---

## Architecture

Separate `UIDocument` following the same pattern as `ZoneCreationPanel`. Three files plus one scene GameObject.

| File | Purpose |
|---|---|
| `Assets/UI/TilePalettePanel.uxml` | Layout — horizontal slot row + active label |
| `Assets/UI/TilePalettePanel.uss` | Styles — hotbar positioning, slot highlight |
| `Assets/Scripts/Runtime/TilePalettePanel.cs` | MonoBehaviour — builds slots, handles selection |
| Scene: `TilePaletteHotbar` GameObject | `UIDocument` + `TilePalettePanel` component |

---

## Tile Types (Phase 2)

Hardcoded. Named to match the `PlaceholderTileGenerator` output. Each carries a runtime colour used to generate its thumbnail.

| Name | Color (approx) |
|---|---|
| Grass | `(0.30, 0.60, 0.10)` |
| Dirt | `(0.55, 0.35, 0.17)` |
| Stone | `(0.50, 0.50, 0.50)` |
| Water | `(0.25, 0.50, 1.00)` |
| Wall | `(0.31, 0.24, 0.16)` |

When real tile textures exist they replace the colour swatch by assigning a `Sprite` to the slot's background image — no code changes required.

---

## UI Structure

```
[bottom-center, always visible]
  "Painting: Grass"           ← active label above hotbar
  [ 🟩 Grass ][ 🟫 Dirt ][ ⬜ Stone ][ 🟦 Water ][ 🟤 Wall ]
     selected ↑ (bright border)
```

- Panel: `position: absolute; bottom: 16px; left: 50%` (CSS translate centres it)
- Each slot: 48×60px — 32×32 colour swatch + 12px label below
- Selected slot: accent-coloured border, slightly lighter background
- No scroll — five tiles fit comfortably in a single row

---

## State Contract

```csharp
// TilePalettePanel.cs
public static string SelectedTile { get; private set; } = "Grass";
```

The raycast tile painter (item #3) reads `TilePalettePanel.SelectedTile` to determine which tile to stamp. No other coupling.

---

## Thumbnail Generation

Runtime `Texture2D` (32×32 solid fill). Generated in `BuildHotbar()`, cached in a `List<Texture2D>`, destroyed in `OnDisable()` to avoid GPU memory leaks.

```csharp
static Texture2D MakeSwatch(Color color) {
    var tex = new Texture2D(32, 32);
    var pixels = new Color[32 * 32];
    for (int i = 0; i < pixels.Length; i++) pixels[i] = color;
    tex.SetPixels(pixels);
    tex.Apply();
    return tex;
}
```

---

## Out of Scope

- Keyboard shortcuts (1–5) — deferred to polish phase
- Real tile textures — deferred until Art/Tiles/ is populated
- Tile categories / grouping — not needed for 5 types
- Server persistence — palette selection is local editor state only
