# ZoneForge Terrain System — Design Spec

**Date:** 2026-03-18
**Status:** Approved
**Scope:** Replaces grid-based tile painting with smooth heightmap + splatmap terrain

---

## Context

ZoneForge currently represents the ground as discrete 1×1 unit colored-cube tiles painted cell-by-cell. This approach is visually blocky and does not support the intended game aesthetic. This spec defines a replacement terrain system using a continuous heightmap mesh with smooth texture blending, stored in SpacetimeDB and shared between the editor and game client in near-real-time.

---

## Coordinate Convention

Unity 3D uses Y-up. The existing `EntityInstance` schema maps `position_x` → world X, and `position_y` → world Z (the horizontal depth axis). This convention is preserved. The new vertical (height) field is named **`elevation`** to avoid ambiguity with the existing positional fields.

---

## Goals

- Replace the discrete grid with a smooth, height-varying terrain mesh
- Support circular brush painting of terrain textures with soft blending at edges
- Support height sculpting (raise, lower, smooth) via brush tools
- Store all terrain data in SpacetimeDB (server-authoritative, no client-side files)
- Render identically in both `ZoneForgeEditor.exe` and `ZoneForgeClient.exe`
- Support variable zone resolution (64, 128, 256, or custom width × height)

---

## Non-Goals

- Unity's built-in Terrain component (not used — custom mesh only)
- Per-stroke real-time sync is nice-to-have, not required
- Multiplayer simultaneous terrain editing (single editor assumed)
- Procedural terrain generation
- Editor identity / role-based authorization (deferred to a future security pass)

---

## Terrain Model

### Representation

The terrain is a subdivided quad mesh on the X/Z plane, deformed vertically by a heightmap. Each point on the grid has:

- A **height value** (world-space Y elevation, stored as `f32`)
- **Blend weights** for up to 4 terrain layers (stored as 4 × `u8`, normalised to [0, 1])

Rendering uses a custom URP shader that samples 4 terrain textures weighted by the blend values (splatmap blending).

### Terrain Layers

| Layer | Splatmap Channel | Behaviour |
|-------|-----------------|-----------|
| Grass | R | Default land surface. Blends with Dirt and Stone. |
| Dirt | G | Paths, riverbeds, disturbed earth. |
| Stone | B | Rocky outcrops, cliff faces, mountain peaks. |
| Ravine / Void | A | Overrides water at low elevation. When Ravine weight > 0.5 the water plane is masked via stencil write — dark pit instead of flooding. |
| Water | — (implicit) | Not a splatmap layer. A separate flat quad mesh rendered at `zone.water_level` Y, visible wherever terrain height is below that value AND Ravine weight ≤ 0.5. Water visibility is controlled via a stencil buffer: terrain mesh writes a stencil value per vertex weighted by Ravine; the water plane only renders where the stencil is clear. |

The `Wall` terrain type is removed. Walls become placed `EntityInstance` prefab objects.

### Zone Resolution

Each zone stores its terrain resolution independently. Supported values: 64×64, 128×128, 256×256, or any custom `u32` width × height. Resolution is set at zone creation and cannot be changed without deleting and recreating the zone's terrain data.

---

## Data Model Changes

### `Zone` table — new fields

```rust
terrain_width:  u32,   // terrain sample count along X
terrain_height: u32,   // terrain sample count along Z
water_level:    f32,   // world-space Y of the water surface
```

`grid_width` and `grid_height` are renamed to `terrain_width` and `terrain_height` respectively. The `ZoneCreationPanel` call site must be updated to match the new field names and the additional `water_level` parameter.

### `TerrainChunk` table — new table

```rust
#[table(accessor = terrain_chunk, public)]
pub struct TerrainChunk {
    #[primary_key]
    #[auto_inc]
    pub id:          u64,
    #[index(btree)]
    pub zone_id:     u64,
    pub chunk_x:     u32,       // chunk column index
    pub chunk_z:     u32,       // chunk row index
    pub height_data: Vec<u8>,   // 32×32 f32 values, little-endian (4 096 bytes)
    pub splat_data:  Vec<u8>,   // 32×32 × 4 u8 values — RGBA blend weights (4 096 bytes)
}
```

**Chunk size:** 32×32 points. A 256×256 terrain produces 64 chunks; each chunk is ~8 KB. Total terrain data for a max-size zone: ~512 KB.

Efficient lookup of a specific chunk uses a `zone_id` btree index plus a client-side filter on `chunk_x / chunk_z`. A unique constraint on `(zone_id, chunk_x, chunk_z)` is enforced in the reducer before insert/update.

### `EntityInstance` table — new field

```rust
pub elevation: f32,   // world-space Y (vertical height). Defaults to terrain height at (position_x, position_y) on placement.
```

### `spawn_entity` reducer — updated signature

```rust
pub fn spawn_entity(
    ctx: &ReducerContext,
    zone_id: u64,
    prefab_name: String,
    position_x: f32,
    position_y: f32,
    elevation: f32,
    entity_type: String,
)
```

`TilePainter.cs` (the only existing call site) is removed as part of this change, so there are no legacy callers to migrate. New entity placement code will always supply `elevation`.

---

## New / Updated Reducers

### `create_zone` — updated signature

```rust
pub fn create_zone(
    ctx: &ReducerContext,
    name: String,
    terrain_width: u32,
    terrain_height: u32,
    water_level: f32,
)
```

On creation, initialises all `TerrainChunk` rows for the zone with a flat heightmap (`height = water_level + 0.5` at every point) and full Grass blend weight (R = 255, G = 0, B = 0, A = 0). Note: a 256×256 zone creates 64 chunks (~512 KB) in a single reducer transaction — smoke-test this early. If SpacetimeDB enforces a transaction size limit, fall back to lazy chunk initialization on first write.

### `update_terrain_chunk`

```rust
pub fn update_terrain_chunk(
    ctx: &ReducerContext,
    zone_id: u64,
    chunk_x: u32,
    chunk_z: u32,
    height_data: Vec<u8>,
    splat_data: Vec<u8>,
)
```

Replaces `height_data` and `splat_data` for the specified chunk. Validates: chunk coordinates are within zone bounds, data lengths are exactly 4 096 bytes each. Authorization (editor-only) is deferred — any connected identity may call this in the current phase.

---

## Brush System (Editor only)

All brushes share: **radius** (world units), **strength** (0–1 opacity/intensity), **falloff curve** (linear / smooth / sharp).

| Brush | Modifies | Modes |
|-------|----------|-------|
| Height | `height_data` only | Raise · Lower · Smooth |
| Texture | `splat_data` only | Paint selected layer (blends, normalised across all 4 channels) |
| Combined | Both | Raise + texture · Lower + texture |
| Cliff ⭐ | Both | Sharp height raise + Stone blend |
| Cliff-to-Water ⭐ | Both | Cliff edge + Ravine blend at base |

⭐ Stretch goal.

### Brush stroke pipeline (Editor)

1. Raycast mouse → world-space hit point on terrain mesh
2. Determine all terrain sample points within brush radius (with falloff weighting)
3. Identify which 32×32 chunks contain those points
4. Modify height/splat values in local chunk buffers
5. Call `update_terrain_chunk` reducer for each modified chunk
6. Patch local mesh immediately (optimistic update) — server confirm arrives via `OnUpdate`

---

## Rendering (shared, Editor + Client)

### Terrain mesh

- Subdivided quad grid: one quad per terrain sample interval
- Vertex Y set from `height_data` at mesh build / chunk update time
- Normals recalculated per chunk update. Smooth normals at chunk seams are computed by reading the adjacent chunk's edge row from the in-memory SpacetimeDB subscription cache (all zone chunks are subscribed at zone load, so neighbour data is always available locally)
- A `TerrainRenderer` MonoBehaviour owns the mesh, subscribes to `TerrainChunk.OnUpdate`, and patches the relevant submesh region

### Subscription strategy

On active zone change, the editor and client subscribe to all `TerrainChunk` rows for that `zone_id` using a filtered query. For a 256×256 zone this is 64 rows × ~8 KB = ~512 KB initial sync. All chunks remain in the subscription cache for the duration of the session, providing neighbour data for seam normal calculations and brush operations.

### Splatmap shader (custom URP Lit)

- Samples 4 terrain textures (Grass, Dirt, Stone, Ravine)
- Blends by RGBA weights from `splat_data` texture
- Textures tile at configurable world-space scale
- Writes a stencil value proportional to the Ravine (A) channel to mask the water plane

### Water

- Separate flat quad mesh at `zone.water_level` Y, sized to zone extents
- Uses a stencil test: only renders where the terrain shader did NOT write a Ravine stencil value
- Simple URP material with transparency and scrolling normal map (stretch: reflections)

### Hover preview

- Brush circle projected onto terrain surface (decal or line renderer following terrain contour)
- Updates every frame in editor; not present in client

---

## Affected Existing Components

| Component | Change |
| --------- | ------ |
| `TilePainter.cs` | **Removed** — replaced by `TerrainPainter` + `TerrainRenderer` |
| `ZoneCreationPanel.cs` | Updated — new `create_zone` signature (terrain_width, terrain_height, water_level) |
| `TilePalettePanel.cs` | Repurposed — becomes terrain layer / brush type selector |
| `SpacetimeDBManager.cs` | Subscription query updated to include `TerrainChunk` for active zone |
| `EditorState.cs` | No change |
| `UIHoverTracker.cs` | No change |
| `server/src/lib.rs` | Schema + reducer changes as described above |

---

## Out of Scope (Future Phases)

- Terrain LOD (level of detail for distant zones)
- Procedural generation / noise-based terrain initialisation
- Multi-editor simultaneous painting / conflict resolution
- Terrain streaming (loading only nearby chunks for very large zones)
- Physics-accurate water simulation
- Editor identity / role-based authorization for terrain reducers
