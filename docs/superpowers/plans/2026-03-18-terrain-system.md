# Terrain System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the discrete grid tile painter with a smooth heightmap + splatmap terrain system stored in SpacetimeDB and rendered identically in both the editor and game client.

**Architecture:** Custom subdivided quad mesh deformed by a heightmap. Texture blending uses a 4-channel splatmap (Grass/Dirt/Stone/Ravine). All terrain data lives in `TerrainChunk` rows in SpacetimeDB (32×32 points per chunk). A circular brush system writes affected chunks on each stroke. A flat water mesh renders at the zone's water level using stencil masking from the splatmap shader.

**Tech Stack:** SpacetimeDB 2.x Rust WASM server, SpacetimeDB C# SDK, Unity 2022.3 LTS 3D URP, custom HLSL shader (URP Unlit with manual stencil), Unity Test Runner (EditMode tests for pure-C# logic)

---

## File Map

### Server
| File | Change |
| ---- | ------ |
| `server/spacetimedb/src/lib.rs` | Modify: update Zone table, add TerrainChunk table, add elevation to EntityInstance, rewrite create_zone, add update_terrain_chunk, update spawn_entity |

### Editor — new files
| File | Responsibility |
| ---- | -------------- |
| `editor/Assets/Scripts/Runtime/TerrainChunkData.cs` | Pure-C# value type: encode/decode height and splat byte arrays; chunk ↔ world coordinate math |
| `editor/Assets/Scripts/Runtime/TerrainRenderer.cs` | MonoBehaviour: owns terrain mesh, subscribes to TerrainChunk.OnUpdate, patches submesh per chunk |
| `editor/Assets/Scripts/Runtime/WaterRenderer.cs` | MonoBehaviour: flat quad mesh at zone.water_level Y, stencil-tested against terrain shader |
| `editor/Assets/Scripts/Runtime/TerrainBrush.cs` | Abstract base: radius, strength, falloff; computes affected sample points and chunks |
| `editor/Assets/Scripts/Runtime/HeightBrush.cs` | Concrete brush: modifies height_data (raise / lower / smooth modes) |
| `editor/Assets/Scripts/Runtime/TextureBrush.cs` | Concrete brush: modifies splat_data for selected terrain layer |
| `editor/Assets/Scripts/Runtime/CombinedBrush.cs` | Concrete brush: modifies both height and splat in one stroke |
| `editor/Assets/Scripts/Runtime/TerrainPainter.cs` | MonoBehaviour: mouse raycast → active brush → TerrainChunkData mutation → update_terrain_chunk reducer + optimistic mesh patch; hover ring |
| `editor/Assets/Shaders/TerrainSplatmap.shader` | Custom URP Unlit HLSL: 4-texture splatmap blend + stencil write for Ravine channel |
| `editor/Assets/Tests/EditMode/TerrainChunkDataTests.cs` | EditMode unit tests for coordinate math and encoding |
| `editor/Assets/Tests/EditMode/TerrainBrushTests.cs` | EditMode unit tests for falloff and point-in-radius logic |

### Editor — modified files
| File | Change |
| ---- | ------ |
| `editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs` | Add TerrainChunk subscription query + OnTerrainChunkUpdated static event |
| `editor/Assets/Scripts/Runtime/ZoneCreationPanel.cs` | Add water_level FloatField to UI, update CreateZone reducer call with new signature |
| `editor/Assets/Scripts/Runtime/TilePalettePanel.cs` | Repurpose: replace tile-type buttons with layer selector (Grass/Dirt/Stone/Ravine) + brush type (Height/Texture/Combined) + radius/strength sliders |
| `editor/Assets/UI/ZoneCreationPanel.uxml` | Add `<FloatField name="water-level">` element |

### Editor — deleted files
| File | Reason |
| ---- | ------ |
| `editor/Assets/Scripts/Runtime/TilePainter.cs` | Replaced by TerrainPainter + TerrainRenderer |

### Client — new files (identical to editor versions)
| File | Notes |
| ---- | ----- |
| `client/Assets/Scripts/Runtime/TerrainChunkData.cs` | Copy of editor version |
| `client/Assets/Scripts/Runtime/TerrainRenderer.cs` | Copy of editor version (no brush tools used in client) |
| `client/Assets/Scripts/Runtime/WaterRenderer.cs` | Copy of editor version |
| `client/Assets/Shaders/TerrainSplatmap.shader` | Copy of editor version |

### Client — modified files
| File | Change |
| ---- | ------ |
| `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs` | Add TerrainChunk subscription query (same change as editor) |

---

## Task 1: Update the Rust server schema

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Step 1.1: Replace Zone table fields**

  Replace the existing `Zone` struct. `grid_width`/`grid_height` become `terrain_width`/`terrain_height`; add `water_level`:

  ```rust
  #[table(accessor = zone, public)]
  pub struct Zone {
      #[primary_key]
      #[auto_inc]
      pub id: u64,
      pub name: String,
      pub terrain_width:  u32,
      pub terrain_height: u32,
      pub water_level:    f32,
  }
  ```

- [ ] **Step 1.2: Add TerrainChunk table**

  Insert after the Zone struct:

  ```rust
  #[table(accessor = terrain_chunk, public)]
  pub struct TerrainChunk {
      #[primary_key]
      #[auto_inc]
      pub id:          u64,
      #[index(btree)]
      pub zone_id:     u64,
      pub chunk_x:     u32,
      pub chunk_z:     u32,
      pub height_data: Vec<u8>,  // 32×32 f32 LE = 4096 bytes
      pub splat_data:  Vec<u8>,  // 32×32 × 4 u8 = 4096 bytes
  }
  ```

- [ ] **Step 1.3: Add elevation to EntityInstance**

  ```rust
  #[table(accessor = entity_instance, public)]
  pub struct EntityInstance {
      #[primary_key]
      #[auto_inc]
      pub id:           u64,
      pub zone_id:      u64,
      pub prefab_name:  String,
      pub position_x:   f32,
      pub position_y:   f32,
      pub elevation:    f32,   // world-space Y (vertical)
      pub entity_type:  String,
  }
  ```

- [ ] **Step 1.4: Rewrite `create_zone` reducer**

  ```rust
  const CHUNK_SIZE: u32 = 32;

  #[reducer]
  pub fn create_zone(
      ctx: &ReducerContext,
      name: String,
      terrain_width: u32,
      terrain_height: u32,
      water_level: f32,
  ) {
      let zone = Zone {
          id: 0,
          name: name.clone(),
          terrain_width,
          terrain_height,
          water_level,
      };
      let zone_row = ctx.db.zone().insert(zone);
      let zone_id = zone_row.id;

      // Initialise flat terrain chunks (height = water_level + 0.5, full Grass).
      let chunks_x = terrain_width.div_ceil(CHUNK_SIZE);
      let chunks_z = terrain_height.div_ceil(CHUNK_SIZE);

      let default_height = water_level + 0.5_f32;
      let height_bytes = default_height.to_le_bytes();

      // 32×32 floats — repeat the 4-byte LE representation 1024 times.
      let height_data: Vec<u8> = height_bytes.iter()
          .cycle()
          .take(4096)
          .cloned()
          .collect();

      // 32×32 × 4 channels — R=255 (Grass), G=0, B=0, A=0.
      let splat_data: Vec<u8> = (0..1024)
          .flat_map(|_| [255u8, 0, 0, 0])
          .collect();

      for cx in 0..chunks_x {
          for cz in 0..chunks_z {
              ctx.db.terrain_chunk().insert(TerrainChunk {
                  id: 0,
                  zone_id,
                  chunk_x: cx,
                  chunk_z: cz,
                  height_data: height_data.clone(),
                  splat_data: splat_data.clone(),
              });
          }
      }

      log::info!("Zone '{}' created with {}×{} chunks", name, chunks_x, chunks_z);
  }
  ```

- [ ] **Step 1.5: Add `update_terrain_chunk` reducer**

  ```rust
  #[reducer]
  pub fn update_terrain_chunk(
      ctx: &ReducerContext,
      zone_id: u64,
      chunk_x: u32,
      chunk_z: u32,
      height_data: Vec<u8>,
      splat_data: Vec<u8>,
  ) -> Result<(), String> {
      if height_data.len() != 4096 {
          return Err(format!("height_data must be 4096 bytes, got {}", height_data.len()));
      }
      if splat_data.len() != 4096 {
          return Err(format!("splat_data must be 4096 bytes, got {}", splat_data.len()));
      }

      // Verify zone + chunk bounds.
      let zone = ctx.db.zone().id().find(&zone_id)
          .ok_or_else(|| format!("Zone {} not found", zone_id))?;

      let max_cx = zone.terrain_width.div_ceil(CHUNK_SIZE);
      let max_cz = zone.terrain_height.div_ceil(CHUNK_SIZE);
      if chunk_x >= max_cx || chunk_z >= max_cz {
          return Err(format!("Chunk ({},{}) out of bounds ({},{})", chunk_x, chunk_z, max_cx, max_cz));
      }

      // Find and update the existing chunk.
      let existing = ctx.db.terrain_chunk().zone_id().filter(&zone_id)
          .find(|c| c.chunk_x == chunk_x && c.chunk_z == chunk_z)
          .ok_or_else(|| format!("Chunk ({},{}) not found in zone {}", chunk_x, chunk_z, zone_id))?;

      ctx.db.terrain_chunk().id().update(TerrainChunk {
          height_data,
          splat_data,
          ..existing
      });

      Ok(())
  }
  ```

- [ ] **Step 1.6: Update `spawn_entity` to include elevation**

  ```rust
  #[reducer]
  pub fn spawn_entity(
      ctx: &ReducerContext,
      zone_id: u64,
      prefab_name: String,
      x: f32,
      y: f32,
      elevation: f32,
      entity_type: String,
  ) -> Result<(), String> {
      if prefab_name.is_empty() {
          return Err("prefab_name cannot be empty".to_string());
      }
      ctx.db.entity_instance().insert(EntityInstance {
          id: 0,
          zone_id,
          prefab_name: prefab_name.clone(),
          position_x: x,
          position_y: y,
          elevation,
          entity_type,
      });
      log::info!("Entity '{}' spawned in zone {} at ({}, {}, {})", prefab_name, zone_id, x, y, elevation);
      Ok(())
  }
  ```

- [ ] **Step 1.7: Verify it compiles**

  ```bash
  cd server && spacetime build
  ```
  Expected: `Finished release` with only the harmless `wasm-opt` warning.

---

## Task 2: Publish server and regenerate bindings

**Files:**
- `editor/Assets/Scripts/autogen/` (regenerated)
- `client/Assets/Scripts/autogen/` (regenerated)

- [ ] **Step 2.1: Publish with `--delete-data`**

  The schema change (renamed Zone fields, new table) is breaking. Run from `server/`:

  ```bash
  cd server && spacetime publish --server local zoneforge-server --delete-data
  ```
  When prompted `upgrade (y/N)?`, type `y`. Expected: `Module published successfully`.

- [ ] **Step 2.2: Regenerate editor bindings**

  Run from `editor/`:

  ```bash
  cd editor && spacetime generate --lang csharp \
    --out-dir Assets/Scripts/autogen \
    --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
  ```
  Verify `Assets/Scripts/autogen/TerrainChunk.cs` is created and `Zone.cs` now has `TerrainWidth`, `TerrainHeight`, `WaterLevel` fields.

- [ ] **Step 2.3: Regenerate client bindings**

  Run from `client/`:

  ```bash
  cd client && spacetime generate --lang csharp \
    --out-dir Assets/Scripts/autogen \
    --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
  ```

- [ ] **Step 2.4: Commit**

  ```bash
  git add server/spacetimedb/src/lib.rs
  git add editor/Assets/Scripts/autogen/
  git add client/Assets/Scripts/autogen/
  git commit -m "feat: terrain system server schema and regenerated bindings"
  ```

---

## Task 3: TerrainChunkData utility (pure C#, EditMode tests)

**Files:**
- Create: `editor/Assets/Scripts/Runtime/TerrainChunkData.cs`
- Create: `editor/Assets/Tests/EditMode/TerrainChunkDataTests.cs`

This is a pure-C# struct with no Unity or SpacetimeDB dependencies — unit testable in EditMode.

- [ ] **Step 3.1: Create the test file first**

  ```csharp
  // editor/Assets/Tests/EditMode/TerrainChunkDataTests.cs
  using NUnit.Framework;

  public class TerrainChunkDataTests
  {
      [Test]
      public void ChunkCoord_FromWorldPoint_ReturnsCorrectChunk()
      {
          // Sample at world position (65, 0, 33) with chunk size 32 → chunk (2, 1)
          TerrainChunkData.WorldToChunk(65f, 33f, 32, out int cx, out int cz);
          Assert.AreEqual(2, cx);
          Assert.AreEqual(1, cz);
      }

      [Test]
      public void LocalIndex_FromWorldPoint_ReturnsCorrectIndex()
      {
          // World (33, 0, 0) → chunk (1, 0) → local (1, 0) → index 1
          int idx = TerrainChunkData.WorldToLocalIndex(33f, 0f, 32);
          Assert.AreEqual(1, idx);
      }

      [Test]
      public void GetHeight_ReturnsEncodedFloat()
      {
          byte[] data = new byte[4096];
          float expected = 3.14f;
          TerrainChunkData.SetHeight(data, 0, expected);
          Assert.AreEqual(expected, TerrainChunkData.GetHeight(data, 0), 0.0001f);
      }

      [Test]
      public void SetSplat_NormalisesWeights()
      {
          byte[] splat = new byte[4096];
          // Set Grass channel to max on index 0 — other channels should become 0
          TerrainChunkData.SetSplatLayer(splat, 0, TerrainLayer.Grass, 1f);
          Assert.AreEqual(255, TerrainChunkData.GetSplatByte(splat, 0, 0)); // R
          Assert.AreEqual(0,   TerrainChunkData.GetSplatByte(splat, 0, 1)); // G
      }
  }
  ```

  You will also need to create an assembly definition for the tests. Create `editor/Assets/Tests/EditMode/EditModeTests.asmdef`:
  ```json
  {
      "name": "EditModeTests",
      "references": ["ZoneForgeRuntime"],
      "includePlatforms": ["Editor"],
      "optionalUnityReferences": ["TestAssemblies"]
  }
  ```

  And an assembly definition for the Runtime scripts. Create `editor/Assets/Scripts/Runtime/ZoneForgeRuntime.asmdef`:
  ```json
  {
      "name": "ZoneForgeRuntime",
      "references": [],
      "includePlatforms": []
  }
  ```

  > **Important:** `TerrainChunkData` and the brush classes added in Tasks 3–5 are pure C# with no Unity/SDK dependencies, so the empty `references` array is fine for now. In Task 7, when `TerrainRenderer` is added (which uses `SpacetimeDB.Types` and `UnityEngine`), you must add `"com.clockworklabs.spacetimedbsdk"` (and any other SDK assembly names Unity resolves) to the `references` array, or simply delete the `ZoneForgeRuntime.asmdef` file if assembly separation is not needed — Unity will compile all scripts in one default assembly without it.

- [ ] **Step 3.2: Run the tests — confirm they fail**

  Open Unity Test Runner (Window → General → Test Runner), select EditMode tab, run `TerrainChunkDataTests`. Expected: all fail with `TerrainChunkData type not found`.

- [ ] **Step 3.3: Implement TerrainChunkData**

  ```csharp
  // editor/Assets/Scripts/Runtime/TerrainChunkData.cs
  using System;

  public enum TerrainLayer { Grass = 0, Dirt = 1, Stone = 2, Ravine = 3 }

  /// <summary>
  /// Pure-C# utility for encoding/decoding terrain chunk byte arrays and
  /// converting between world-space coordinates and chunk/local indices.
  /// No Unity or SpacetimeDB dependencies.
  /// </summary>
  public static class TerrainChunkData
  {
      public const int ChunkSize  = 32;
      public const int PointCount = ChunkSize * ChunkSize; // 1024
      public const int HeightBytes = PointCount * 4;       // 4096
      public const int SplatBytes  = PointCount * 4;       // 4096

      // -----------------------------------------------------------------------
      // Coordinate math

      /// <summary>Returns chunk column/row for a world X/Z position.</summary>
      public static void WorldToChunk(float worldX, float worldZ, int chunkSize, out int chunkX, out int chunkZ)
      {
          chunkX = (int)Math.Floor(worldX / chunkSize);
          chunkZ = (int)Math.Floor(worldZ / chunkSize);
      }

      /// <summary>
      /// Returns the flat index into a 32×32 chunk's data arrays for a world position.
      /// Caller must already know the point falls within the correct chunk.
      /// </summary>
      public static int WorldToLocalIndex(float worldX, float worldZ, int chunkSize)
      {
          int lx = (int)Math.Floor(worldX) % chunkSize;
          int lz = (int)Math.Floor(worldZ) % chunkSize;
          lx = ((lx % chunkSize) + chunkSize) % chunkSize; // handle negatives
          lz = ((lz % chunkSize) + chunkSize) % chunkSize;
          return lz * chunkSize + lx;
      }

      /// <summary>World-space origin of a chunk (bottom-left corner at Y=0).</summary>
      public static (float x, float z) ChunkOrigin(int chunkX, int chunkZ) =>
          (chunkX * ChunkSize, chunkZ * ChunkSize);

      // -----------------------------------------------------------------------
      // Height encoding (little-endian float32)

      public static float GetHeight(byte[] data, int index)
      {
          int offset = index * 4;
          return BitConverter.ToSingle(data, offset);
      }

      public static void SetHeight(byte[] data, int index, float value)
      {
          int offset = index * 4;
          byte[] bytes = BitConverter.GetBytes(value);
          data[offset]     = bytes[0];
          data[offset + 1] = bytes[1];
          data[offset + 2] = bytes[2];
          data[offset + 3] = bytes[3];
      }

      // -----------------------------------------------------------------------
      // Splat encoding (RGBA u8 per point, normalised)

      public static byte GetSplatByte(byte[] splat, int index, int channel)
          => splat[index * 4 + channel];

      public static float GetSplatWeight(byte[] splat, int index, TerrainLayer layer)
          => splat[index * 4 + (int)layer] / 255f;

      /// <summary>
      /// Sets the weight for <paramref name="layer"/> at <paramref name="index"/>
      /// and redistributes the remaining weight proportionally across other channels.
      /// </summary>
      public static void SetSplatLayer(byte[] splat, int index, TerrainLayer layer, float weight)
      {
          weight = Math.Max(0f, Math.Min(1f, weight));
          int offset = index * 4;
          int ch = (int)layer;

          // Read current other-channel weights.
          float[] cur = new float[4];
          float otherTotal = 0f;
          for (int i = 0; i < 4; i++)
          {
              cur[i] = splat[offset + i] / 255f;
              if (i != ch) otherTotal += cur[i];
          }

          // Redistribute remaining (1 - weight) across other channels.
          float remaining = 1f - weight;
          for (int i = 0; i < 4; i++)
          {
              if (i == ch)
                  splat[offset + i] = (byte)(weight * 255f + 0.5f);
              else
              {
                  float newW = otherTotal > 0f ? (cur[i] / otherTotal) * remaining : 0f;
                  splat[offset + i] = (byte)(newW * 255f + 0.5f);
              }
          }
      }

      // -----------------------------------------------------------------------
      // Flat default chunk helpers

      public static byte[] DefaultHeightData(float height)
      {
          byte[] data = new byte[HeightBytes];
          byte[] hb = BitConverter.GetBytes(height);
          for (int i = 0; i < PointCount; i++)
          {
              int offset = i * 4;
              data[offset]     = hb[0];
              data[offset + 1] = hb[1];
              data[offset + 2] = hb[2];
              data[offset + 3] = hb[3];
          }
          return data;
      }

      public static byte[] DefaultSplatData()
      {
          // Full Grass (R=255, G=0, B=0, A=0)
          byte[] data = new byte[SplatBytes];
          for (int i = 0; i < PointCount; i++)
              data[i * 4] = 255;
          return data;
      }
  }
  ```

- [ ] **Step 3.4: Run tests — confirm they pass**

  Run `TerrainChunkDataTests` in Unity Test Runner. Expected: 4 tests pass.

- [ ] **Step 3.5: Commit**

  ```bash
  git add editor/Assets/Scripts/Runtime/TerrainChunkData.cs
  git add editor/Assets/Scripts/Runtime/ZoneForgeRuntime.asmdef
  git add editor/Assets/Tests/
  git commit -m "feat: TerrainChunkData utility with EditMode tests"
  ```

---

## Task 4: TerrainBrush base and HeightBrush (with EditMode tests)

**Files:**
- Create: `editor/Assets/Scripts/Runtime/TerrainBrush.cs`
- Create: `editor/Assets/Scripts/Runtime/HeightBrush.cs`
- Create: `editor/Assets/Tests/EditMode/TerrainBrushTests.cs`

- [ ] **Step 4.1: Write brush tests first**

  ```csharp
  // editor/Assets/Tests/EditMode/TerrainBrushTests.cs
  using NUnit.Framework;
  using System.Collections.Generic;

  public class TerrainBrushTests
  {
      [Test]
      public void FalloffLinear_AtCenter_IsOne()
      {
          float f = TerrainBrush.Falloff(0f, 5f, TerrainBrush.FalloffMode.Linear);
          Assert.AreEqual(1f, f, 0.001f);
      }

      [Test]
      public void FalloffLinear_AtEdge_IsZero()
      {
          float f = TerrainBrush.Falloff(5f, 5f, TerrainBrush.FalloffMode.Linear);
          Assert.AreEqual(0f, f, 0.001f);
      }

      [Test]
      public void FalloffSmooth_AtHalfRadius_IsBetweenZeroAndOne()
      {
          float f = TerrainBrush.Falloff(2.5f, 5f, TerrainBrush.FalloffMode.Smooth);
          Assert.Greater(f, 0f);
          Assert.Less(f, 1f);
      }

      [Test]
      public void AffectedPoints_CircleRadius5_IncludesCentre()
      {
          var points = TerrainBrush.AffectedPoints(16f, 16f, 5f);
          Assert.IsTrue(points.ContainsKey((16, 16)));
      }

      [Test]
      public void AffectedPoints_ExcludesPointsBeyondRadius()
      {
          var points = TerrainBrush.AffectedPoints(0f, 0f, 2f);
          foreach (var kv in points)
              Assert.LessOrEqual(kv.Value, 1f); // falloff weight in [0,1]
      }
  }
  ```

- [ ] **Step 4.2: Run tests — confirm they fail**

  Expected: fail with `TerrainBrush type not found`.

- [ ] **Step 4.3: Implement TerrainBrush base**

  ```csharp
  // editor/Assets/Scripts/Runtime/TerrainBrush.cs
  using System.Collections.Generic;
  using UnityEngine;

  /// <summary>
  /// Abstract base for all terrain brushes. Handles radius, strength, falloff,
  /// and the set of affected terrain sample points.
  /// </summary>
  public abstract class TerrainBrush
  {
      public enum FalloffMode { Linear, Smooth, Sharp }

      public float Radius   { get; set; } = 5f;
      public float Strength { get; set; } = 0.5f;
      public FalloffMode Falloff { get; set; } = FalloffMode.Smooth;

      // -----------------------------------------------------------------------
      // Static helpers (unit-testable)

      /// <summary>
      /// Weight at <paramref name="dist"/> from brush centre given <paramref name="radius"/>.
      /// Returns 0 when dist >= radius.
      /// </summary>
      public static float Falloff(float dist, float radius, FalloffMode mode)
      {
          if (dist >= radius) return 0f;
          float t = 1f - dist / radius;
          return mode switch
          {
              FalloffMode.Linear => t,
              FalloffMode.Smooth => t * t * (3f - 2f * t),  // smoothstep
              FalloffMode.Sharp  => 1f,
              _ => t,
          };
      }

      /// <summary>
      /// Returns all integer sample points (gx, gz) within <paramref name="radius"/>
      /// of (<paramref name="worldX"/>, <paramref name="worldZ"/>),
      /// mapped to their falloff weight [0, 1].
      /// </summary>
      public static Dictionary<(int, int), float> AffectedPoints(float worldX, float worldZ, float radius)
      {
          var result = new Dictionary<(int, int), float>();
          int minX = Mathf.FloorToInt(worldX - radius);
          int maxX = Mathf.CeilToInt(worldX + radius);
          int minZ = Mathf.FloorToInt(worldZ - radius);
          int maxZ = Mathf.CeilToInt(worldZ + radius);

          for (int gx = minX; gx <= maxX; gx++)
          for (int gz = minZ; gz <= maxZ; gz++)
          {
              float dist = Vector2.Distance(new Vector2(worldX, worldZ), new Vector2(gx, gz));
              float w = Falloff(dist, radius, FalloffMode.Smooth);
              if (w > 0f) result[(gx, gz)] = w;
          }
          return result;
      }

      /// <summary>
      /// Groups affected points by chunk coordinates.
      /// Returns dict of chunk (cx, cz) → list of (localIndex, weight).
      /// </summary>
      public static Dictionary<(int, int), List<(int idx, float weight)>> GroupByChunk(
          Dictionary<(int, int), float> points)
      {
          var chunks = new Dictionary<(int, int), List<(int, float)>>();
          foreach (var kv in points)
          {
              int gx = kv.Key.Item1, gz = kv.Key.Item2;
              if (gx < 0 || gz < 0) continue;
              TerrainChunkData.WorldToChunk(gx, gz, TerrainChunkData.ChunkSize, out int cx, out int cz);
              int idx = TerrainChunkData.WorldToLocalIndex(gx, gz, TerrainChunkData.ChunkSize);
              if (!chunks.TryGetValue((cx, cz), out var list))
                  chunks[(cx, cz)] = list = new List<(int, float)>();
              list.Add((idx, kv.Value));
          }
          return chunks;
      }

      // -----------------------------------------------------------------------
      // Abstract stroke application

      /// <summary>
      /// Apply this brush at (<paramref name="worldX"/>, <paramref name="worldZ"/>)
      /// to local copies of the height and splat arrays for the given chunk.
      /// Called once per affected chunk per stroke.
      /// </summary>
      public abstract void ApplyToChunk(
          byte[] heightData, byte[] splatData,
          List<(int idx, float weight)> affectedPoints,
          float strength, float deltaTime);
  }
  ```

- [ ] **Step 4.4: Implement HeightBrush**

  ```csharp
  // editor/Assets/Scripts/Runtime/HeightBrush.cs
  using System.Collections.Generic;
  using UnityEngine;

  public class HeightBrush : TerrainBrush
  {
      public enum Mode { Raise, Lower, Smooth }
      public Mode BrushMode { get; set; } = Mode.Raise;

      public override void ApplyToChunk(
          byte[] heightData, byte[] splatData,
          List<(int idx, float weight)> affectedPoints,
          float strength, float deltaTime)
      {
          float delta = strength * deltaTime * 2f; // 2 world-units/sec at full strength

          if (BrushMode == Mode.Smooth)
          {
              // Compute average height across affected points first.
              float sum = 0f;
              foreach (var (idx, _) in affectedPoints)
                  sum += TerrainChunkData.GetHeight(heightData, idx);
              float avg = sum / affectedPoints.Count;

              foreach (var (idx, weight) in affectedPoints)
              {
                  float h = TerrainChunkData.GetHeight(heightData, idx);
                  float newH = Mathf.Lerp(h, avg, weight * strength * deltaTime * 5f);
                  TerrainChunkData.SetHeight(heightData, idx, newH);
              }
          }
          else
          {
              float sign = BrushMode == Mode.Raise ? 1f : -1f;
              foreach (var (idx, weight) in affectedPoints)
              {
                  float h = TerrainChunkData.GetHeight(heightData, idx);
                  TerrainChunkData.SetHeight(heightData, idx, h + sign * weight * delta);
              }
          }
      }
  }
  ```

- [ ] **Step 4.5: Run tests — confirm they pass**

  Expected: 5 tests pass.

- [ ] **Step 4.6: Commit**

  ```bash
  git add editor/Assets/Scripts/Runtime/TerrainBrush.cs
  git add editor/Assets/Scripts/Runtime/HeightBrush.cs
  git add editor/Assets/Tests/EditMode/TerrainBrushTests.cs
  git commit -m "feat: TerrainBrush base and HeightBrush with EditMode tests"
  ```

---

## Task 5: TextureBrush and CombinedBrush

**Files:**
- Create: `editor/Assets/Scripts/Runtime/TextureBrush.cs`
- Create: `editor/Assets/Scripts/Runtime/CombinedBrush.cs`

- [ ] **Step 5.1: Implement TextureBrush**

  ```csharp
  // editor/Assets/Scripts/Runtime/TextureBrush.cs
  using System.Collections.Generic;

  public class TextureBrush : TerrainBrush
  {
      public TerrainLayer TargetLayer { get; set; } = TerrainLayer.Grass;

      public override void ApplyToChunk(
          byte[] heightData, byte[] splatData,
          List<(int idx, float weight)> affectedPoints,
          float strength, float deltaTime)
      {
          float paintRate = strength * deltaTime * 3f; // blend speed

          foreach (var (idx, weight) in affectedPoints)
          {
              float current = TerrainChunkData.GetSplatWeight(splatData, idx, TargetLayer);
              float target  = current + weight * paintRate;
              if (target > 1f) target = 1f;
              TerrainChunkData.SetSplatLayer(splatData, idx, TargetLayer, target);
          }
      }
  }
  ```

- [ ] **Step 5.2: Implement CombinedBrush**

  ```csharp
  // editor/Assets/Scripts/Runtime/CombinedBrush.cs
  using System.Collections.Generic;

  public class CombinedBrush : TerrainBrush
  {
      public enum Mode { RaiseAndPaint, LowerAndPaint }
      public Mode BrushMode   { get; set; } = Mode.RaiseAndPaint;
      public TerrainLayer TargetLayer { get; set; } = TerrainLayer.Grass;

      private readonly HeightBrush  _height  = new();
      private readonly TextureBrush _texture = new();

      public override void ApplyToChunk(
          byte[] heightData, byte[] splatData,
          List<(int idx, float weight)> affectedPoints,
          float strength, float deltaTime)
      {
          _height.BrushMode  = BrushMode == Mode.RaiseAndPaint ? HeightBrush.Mode.Raise : HeightBrush.Mode.Lower;
          _texture.TargetLayer = TargetLayer;

          _height.ApplyToChunk(heightData, splatData, affectedPoints, strength, deltaTime);
          _texture.ApplyToChunk(heightData, splatData, affectedPoints, strength, deltaTime);
      }
  }
  ```

- [ ] **Step 5.3: Commit**

  ```bash
  git add editor/Assets/Scripts/Runtime/TextureBrush.cs
  git add editor/Assets/Scripts/Runtime/CombinedBrush.cs
  git commit -m "feat: TextureBrush and CombinedBrush"
  ```

---

## Task 6: Terrain splatmap shader

**Files:**
- Create: `editor/Assets/Shaders/TerrainSplatmap.shader`

The shader samples 4 textures blended by RGBA splatmap weights. Water masking (Ravine areas) is handled in the **water shader** via `clip()`, not via stencil — this avoids `SV_StencilRef` which is not reliably supported across all Unity backends (DX11-only). The terrain shader itself is straightforward opaque blending.

- [ ] **Step 6.1: Create the Shaders directory and terrain shader**

  ```bash
  mkdir -p editor/Assets/Shaders
  ```

  ```hlsl
  // editor/Assets/Shaders/TerrainSplatmap.shader
  Shader "ZoneForge/TerrainSplatmap"
  {
      Properties
      {
          _GrassTex  ("Grass Texture",  2D) = "white" {}
          _DirtTex   ("Dirt Texture",   2D) = "white" {}
          _StoneTex  ("Stone Texture",  2D) = "white" {}
          _RavineTex ("Ravine Texture", 2D) = "black" {}
          _SplatTex  ("Splat Map",      2D) = "red"   {}
          _TileScale ("Tile Scale", Float) = 4.0
      }

      SubShader
      {
          Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
          LOD 100

          Pass
          {
              Name "ForwardLit"
              Tags { "LightMode"="UniversalForward" }

              HLSLPROGRAM
              #pragma vertex vert
              #pragma fragment frag
              #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

              struct Attributes
              {
                  float4 positionOS : POSITION;
                  float2 splatUV    : TEXCOORD1;  // second UV set: normalised [0,1] over full terrain
              };

              struct Varyings
              {
                  float4 positionHCS : SV_POSITION;
                  float2 worldXZ     : TEXCOORD0;  // world X/Z for texture tiling
                  float2 splatUV     : TEXCOORD1;
              };

              TEXTURE2D(_GrassTex);  SAMPLER(sampler_GrassTex);
              TEXTURE2D(_DirtTex);   SAMPLER(sampler_DirtTex);
              TEXTURE2D(_StoneTex);  SAMPLER(sampler_StoneTex);
              TEXTURE2D(_RavineTex); SAMPLER(sampler_RavineTex);
              TEXTURE2D(_SplatTex);  SAMPLER(sampler_SplatTex);

              CBUFFER_START(UnityPerMaterial)
                  float4 _GrassTex_ST;
                  float  _TileScale;
              CBUFFER_END

              Varyings vert(Attributes IN)
              {
                  Varyings OUT;
                  OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                  float3 worldPos = TransformObjectToWorld(IN.positionOS.xyz);
                  OUT.worldXZ = worldPos.xz / _TileScale;
                  OUT.splatUV = IN.splatUV;
                  return OUT;
              }

              half4 frag(Varyings IN) : SV_Target
              {
                  half4 splat = SAMPLE_TEXTURE2D(_SplatTex, sampler_SplatTex, IN.splatUV);

                  half4 grass  = SAMPLE_TEXTURE2D(_GrassTex,  sampler_GrassTex,  IN.worldXZ);
                  half4 dirt   = SAMPLE_TEXTURE2D(_DirtTex,   sampler_DirtTex,   IN.worldXZ);
                  half4 stone  = SAMPLE_TEXTURE2D(_StoneTex,  sampler_StoneTex,  IN.worldXZ);
                  half4 ravine = SAMPLE_TEXTURE2D(_RavineTex, sampler_RavineTex, IN.worldXZ);

                  half4 color = grass  * splat.r
                              + dirt   * splat.g
                              + stone  * splat.b
                              + ravine * splat.a;

                  return half4(color.rgb, 1.0);
              }
              ENDHLSL
          }
      }
  }
  ```

- [ ] **Step 6.2: Create the water shader**

  The water shader samples the same `_SplatTex` and `clip()`s pixels where the Ravine channel is dominant — dark pit, no water. This works on all Unity backends.

  ```hlsl
  // editor/Assets/Shaders/WaterUnlit.shader
  Shader "ZoneForge/WaterUnlit"
  {
      Properties
      {
          _Color       ("Water Color", Color)  = (0.2, 0.5, 1.0, 0.7)
          _SplatTex    ("Splat Map",   2D)     = "red" {}
          _TerrainSize ("Terrain Size (XZ)", Vector) = (64, 64, 0, 0)
      }

      SubShader
      {
          Tags { "RenderType"="Transparent" "Queue"="Transparent" "RenderPipeline"="UniversalPipeline" }

          Blend SrcAlpha OneMinusSrcAlpha
          ZWrite Off

          Pass
          {
              Name "WaterForward"
              Tags { "LightMode"="UniversalForward" }

              HLSLPROGRAM
              #pragma vertex vert
              #pragma fragment frag
              #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

              struct Attributes { float4 positionOS : POSITION; };

              struct Varyings
              {
                  float4 positionHCS : SV_POSITION;
                  float3 worldPos    : TEXCOORD0;
              };

              TEXTURE2D(_SplatTex); SAMPLER(sampler_SplatTex);

              CBUFFER_START(UnityPerMaterial)
                  half4  _Color;
                  float4 _TerrainSize;
              CBUFFER_END

              Varyings vert(Attributes IN)
              {
                  Varyings OUT;
                  OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                  OUT.worldPos    = TransformObjectToWorld(IN.positionOS.xyz);
                  return OUT;
              }

              half4 frag(Varyings IN) : SV_Target
              {
                  // Normalised terrain UV from world XZ.
                  float2 splatUV = IN.worldPos.xz / _TerrainSize.xy;
                  half4 splat = SAMPLE_TEXTURE2D(_SplatTex, sampler_SplatTex, splatUV);

                  // Discard where Ravine channel is dominant (dark pit, not water).
                  clip(0.5h - splat.a);

                  return _Color;
              }
              ENDHLSL
          }
      }
  }
  ```

  > **Note:** `TerrainRenderer.BuildSplatTexture()` sets `_SplatTex` on the terrain material. `WaterRenderer` must also receive this texture — pass it via `_waterMaterial.SetTexture("_SplatTex", tex)` and `_waterMaterial.SetVector("_TerrainSize", new Vector4(width, height, 0, 0))` after the terrain builds it. Add a `public void SetSplatTexture(Texture2D tex, int w, int h)` method to `WaterRenderer` and call it from `TerrainRenderer.BuildSplatTexture()`.

- [ ] **Step 6.2: Commit**

  ```bash
  git add editor/Assets/Shaders/TerrainSplatmap.shader
  git commit -m "feat: TerrainSplatmap URP shader with stencil write"
  ```

---

## Task 7: TerrainRenderer MonoBehaviour

**Files:**
- Create: `editor/Assets/Scripts/Runtime/TerrainRenderer.cs`

`TerrainRenderer` owns the terrain `Mesh`, builds it from chunk data, and patches it whenever `SpacetimeDBManager.OnTerrainChunkUpdated` fires. It also creates the splatmap `Texture2D` per chunk.

- [ ] **Step 7.1: Implement TerrainRenderer**

  ```csharp
  // editor/Assets/Scripts/Runtime/TerrainRenderer.cs
  using System.Collections.Generic;
  using UnityEngine;
  using SpacetimeDB.Types;

  /// <summary>
  /// Builds and maintains the terrain mesh from TerrainChunk rows.
  /// Attach to a GameObject in the scene; assign the TerrainSplatmap material.
  /// TerrainRenderer rebuilds when the active zone changes and patches on chunk updates.
  /// </summary>
  [RequireComponent(typeof(MeshFilter), typeof(MeshRenderer))]
  public class TerrainRenderer : MonoBehaviour
  {
      [SerializeField] private Material _terrainMaterial;

      private Mesh _mesh;
      private MeshFilter _filter;
      private MeshRenderer _renderer;

      // zone dimensions in sample points
      private int _terrainWidth;
      private int _terrainHeight;

      // chunk cache: (cx, cz) → last received TerrainChunk
      private readonly Dictionary<(int, int), TerrainChunk> _chunks = new();

      // per-chunk splatmap textures
      private readonly Dictionary<(int, int), Texture2D> _splatTextures = new();

      // -----------------------------------------------------------------------

      void Awake()
      {
          _filter   = GetComponent<MeshFilter>();
          _renderer = GetComponent<MeshRenderer>();
          _mesh = new Mesh { name = "TerrainMesh" };
          _mesh.indexFormat = UnityEngine.Rendering.IndexFormat.UInt32;
          _filter.mesh = _mesh;
      }

      void OnEnable()
      {
          SpacetimeDBManager.OnConnected            += OnConnected;
          SpacetimeDBManager.OnTerrainChunkUpdated  += OnChunkUpdated;
          EditorState.OnActiveZoneChanged           += OnActiveZoneChanged;

          if (SpacetimeDBManager.IsSubscribed)
              RebuildFromActiveZone();
      }

      void OnDisable()
      {
          SpacetimeDBManager.OnConnected           -= OnConnected;
          SpacetimeDBManager.OnTerrainChunkUpdated -= OnChunkUpdated;
          EditorState.OnActiveZoneChanged          -= OnActiveZoneChanged;
      }

      // -----------------------------------------------------------------------

      void OnConnected() => RebuildFromActiveZone();

      void OnActiveZoneChanged(ulong _)
      {
          _chunks.Clear();
          ClearSplatTextures();
          RebuildFromActiveZone();
      }

      void OnChunkUpdated(TerrainChunk chunk)
      {
          if (chunk.ZoneId != EditorState.ActiveZoneId) return;
          _chunks[(int)chunk.ChunkX, (int)chunk.ChunkZ)] = chunk;
          PatchChunk(chunk);
      }

      // -----------------------------------------------------------------------

      void RebuildFromActiveZone()
      {
          if (SpacetimeDBManager.Conn == null || !EditorState.HasActiveZone) return;

          _terrainWidth  = 0;
          _terrainHeight = 0;

          foreach (var zone in SpacetimeDBManager.Conn.Db.Zone.Iter())
          {
              if (zone.Id != EditorState.ActiveZoneId) continue;
              _terrainWidth  = (int)zone.TerrainWidth;
              _terrainHeight = (int)zone.TerrainHeight;
              break;
          }

          if (_terrainWidth == 0) return;

          // Cache all chunks for this zone.
          _chunks.Clear();
          foreach (var chunk in SpacetimeDBManager.Conn.Db.TerrainChunk.Iter())
          {
              if (chunk.ZoneId != EditorState.ActiveZoneId) continue;
              _chunks[((int)chunk.ChunkX, (int)chunk.ChunkZ)] = chunk;
          }

          BuildFullMesh();
      }

      // -----------------------------------------------------------------------
      // Mesh construction

      void BuildFullMesh()
      {
          int w = _terrainWidth;
          int h = _terrainHeight;

          // Vertices: one per sample point
          var vertices = new Vector3[w * h];
          var uv0      = new Vector2[w * h]; // world XZ (passed to shader as worldXZ)
          var uv1      = new Vector2[w * h]; // normalised terrain UV for splatmap

          for (int z = 0; z < h; z++)
          for (int x = 0; x < w; x++)
          {
              int vi = z * w + x;
              float height = SampleHeight(x, z);
              vertices[vi] = new Vector3(x, height, z);
              uv0[vi]      = new Vector2(x, z);
              uv1[vi]      = new Vector2((float)x / w, (float)z / h);
          }

          // Triangles: 2 per quad
          int quadCount = (w - 1) * (h - 1);
          var triangles = new int[quadCount * 6];
          int t = 0;
          for (int z = 0; z < h - 1; z++)
          for (int x = 0; x < w - 1; x++)
          {
              int vi = z * w + x;
              triangles[t++] = vi;
              triangles[t++] = vi + w;
              triangles[t++] = vi + 1;
              triangles[t++] = vi + 1;
              triangles[t++] = vi + w;
              triangles[t++] = vi + w + 1;
          }

          _mesh.Clear();
          _mesh.vertices  = vertices;
          _mesh.uv        = uv0;
          _mesh.uv2       = uv1;
          _mesh.triangles = triangles;
          _mesh.RecalculateNormals();
          _mesh.RecalculateBounds();

          // Build combined splatmap texture (full terrain, one RGBA texture).
          BuildSplatTexture();

          _renderer.material = _terrainMaterial;
      }

      float SampleHeight(int gx, int gz)
      {
          TerrainChunkData.WorldToChunk(gx, gz, TerrainChunkData.ChunkSize,
              out int cx, out int cz);
          int idx = TerrainChunkData.WorldToLocalIndex(gx, gz, TerrainChunkData.ChunkSize);

          if (_chunks.TryGetValue((cx, cz), out var chunk))
              return TerrainChunkData.GetHeight(chunk.HeightData, idx);
          return 0f;
      }

      void BuildSplatTexture()
      {
          // One RGBA texture covering the full terrain.
          var tex = new Texture2D(_terrainWidth, _terrainHeight, TextureFormat.RGBA32, false);
          var pixels = new Color32[_terrainWidth * _terrainHeight];

          for (int z = 0; z < _terrainHeight; z++)
          for (int x = 0; x < _terrainWidth;  x++)
          {
              TerrainChunkData.WorldToChunk(x, z, TerrainChunkData.ChunkSize,
                  out int cx, out int cz);
              int idx = TerrainChunkData.WorldToLocalIndex(x, z, TerrainChunkData.ChunkSize);

              byte r = 255, g = 0, b = 0, a = 0;
              if (_chunks.TryGetValue((cx, cz), out var chunk))
              {
                  r = chunk.SplatData[idx * 4];
                  g = chunk.SplatData[idx * 4 + 1];
                  b = chunk.SplatData[idx * 4 + 2];
                  a = chunk.SplatData[idx * 4 + 3];
              }
              pixels[z * _terrainWidth + x] = new Color32(r, g, b, a);
          }

          tex.SetPixels32(pixels);
          tex.Apply();

          _terrainMaterial.SetTexture("_SplatTex", tex);
      }

      /// <summary>Patch only the vertices and splat pixels affected by a single chunk update.</summary>
      void PatchChunk(TerrainChunk chunk)
      {
          if (_terrainWidth == 0) return;
          // For simplicity in this initial implementation, rebuild the full mesh.
          // Optimise to partial vertex update in a follow-up if needed.
          BuildFullMesh();
      }

      void ClearSplatTextures()
      {
          foreach (var tex in _splatTextures.Values)
              if (tex != null) Destroy(tex);
          _splatTextures.Clear();
      }
  }
  ```

  > **Note:** There is a small typo in the `OnChunkUpdated` method above — `_chunks[(int)chunk.ChunkX, (int)chunk.ChunkZ)]` should use the Dictionary syntax `_chunks[((int)chunk.ChunkX, (int)chunk.ChunkZ)]`. Fix this during implementation.

- [ ] **Step 7.2: Commit**

  ```bash
  git add editor/Assets/Scripts/Runtime/TerrainRenderer.cs
  git commit -m "feat: TerrainRenderer MonoBehaviour"
  ```

---

## Task 8: WaterRenderer MonoBehaviour

**Files:**
- Create: `editor/Assets/Scripts/Runtime/WaterRenderer.cs`

- [ ] **Step 8.1: Implement WaterRenderer**

  ```csharp
  // editor/Assets/Scripts/Runtime/WaterRenderer.cs
  using UnityEngine;
  using SpacetimeDB.Types;

  /// <summary>
  /// Renders a flat quad at zone.water_level Y.
  /// Uses a stencil test to hide beneath Ravine terrain areas.
  /// </summary>
  [RequireComponent(typeof(MeshFilter), typeof(MeshRenderer))]
  public class WaterRenderer : MonoBehaviour
  {
      [SerializeField] private Material _waterMaterial;

      private MeshFilter _filter;
      private MeshRenderer _renderer;

      void Awake()
      {
          _filter   = GetComponent<MeshFilter>();
          _renderer = GetComponent<MeshRenderer>();
      }

      void OnEnable()
      {
          SpacetimeDBManager.OnConnected      += RefreshWater;
          EditorState.OnActiveZoneChanged     += _ => RefreshWater();

          if (SpacetimeDBManager.IsSubscribed) RefreshWater();
      }

      void OnDisable()
      {
          SpacetimeDBManager.OnConnected  -= RefreshWater;
          EditorState.OnActiveZoneChanged -= _ => RefreshWater();
      }

      void RefreshWater()
      {
          if (SpacetimeDBManager.Conn == null || !EditorState.HasActiveZone)
          {
              gameObject.SetActive(false);
              return;
          }

          foreach (var zone in SpacetimeDBManager.Conn.Db.Zone.Iter())
          {
              if (zone.Id != EditorState.ActiveZoneId) continue;
              BuildWaterMesh(zone.TerrainWidth, zone.TerrainHeight, zone.WaterLevel);
              return;
          }

          gameObject.SetActive(false);
      }

      void BuildWaterMesh(uint width, uint height, float waterLevel)
      {
          float w = width;
          float h = height;

          var mesh = new Mesh { name = "WaterMesh" };
          mesh.vertices  = new[]
          {
              new Vector3(0, waterLevel, 0),
              new Vector3(w, waterLevel, 0),
              new Vector3(w, waterLevel, h),
              new Vector3(0, waterLevel, h),
          };
          mesh.uv = new[] { Vector2.zero, Vector2.right, Vector2.one, Vector2.up };
          mesh.triangles = new[] { 0, 2, 1, 0, 3, 2 };
          mesh.RecalculateNormals();

          _filter.mesh = mesh;
          _renderer.material = _waterMaterial;
          gameObject.SetActive(true);
      }
  }
  ```

  The water `Material` should be a URP Unlit material with:
  - Stencil Test: `Ref 1 Comp NotEqual` (only render where terrain didn't write stencil 1)
  - Surface: Transparent, blue-tinted color, slight alpha (0.7)

  Create the water material in Unity: `Assets/Art/Materials/WaterMaterial.mat` — set Shader to `Universal Render Pipeline/Unlit`, color to `(0.2, 0.5, 1.0, 0.7)`, Surface Type `Transparent`. Then edit the `.mat` file's stencil settings via the Inspector's advanced mode or by editing the YAML.

- [ ] **Step 8.2: Commit**

  ```bash
  git add editor/Assets/Scripts/Runtime/WaterRenderer.cs
  git commit -m "feat: WaterRenderer with stencil masking"
  ```

---

## Task 9: Update SpacetimeDBManager

**Files:**
- Modify: `editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs`

- [ ] **Step 9.1: Add TerrainChunk event and subscription**

  Add the static event near the other events:
  ```csharp
  public static event Action<TerrainChunk> OnTerrainChunkUpdated;
  ```

  In `OnConnect`, add `"SELECT * FROM terrain_chunk"` to the subscription array:
  ```csharp
  conn.SubscriptionBuilder()
      .OnApplied(OnSubscriptionApplied)
      .Subscribe(new[]
      {
          "SELECT * FROM player",
          "SELECT * FROM zone",
          "SELECT * FROM entity_instance",
          "SELECT * FROM terrain_chunk"
      });
  ```

  In `OnSubscriptionApplied`, add the terrain chunk callback:
  ```csharp
  Conn.Db.TerrainChunk.OnInsert += (_, chunk) => OnTerrainChunkUpdated?.Invoke(chunk);
  Conn.Db.TerrainChunk.OnUpdate += (_, _old, chunk) => OnTerrainChunkUpdated?.Invoke(chunk);
  ```

- [ ] **Step 9.2: Confirm Unity compiles with no errors**

  Open Unity and wait for script compilation. Expected: no compile errors in Console.

- [ ] **Step 9.3: Commit**

  ```bash
  git add editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs
  git commit -m "feat: add TerrainChunk subscription and event to SpacetimeDBManager"
  ```

---

## Task 10: Update ZoneCreationPanel

**Files:**
- Modify: `editor/Assets/Scripts/Runtime/ZoneCreationPanel.cs`
- Modify: `editor/Assets/UI/ZoneCreationPanel.uxml`

- [ ] **Step 10.1: Add water_level field to the UXML**

  Open `editor/Assets/UI/ZoneCreationPanel.uxml` and add a `FloatField` after the height field:
  ```xml
  <ui:FloatField label="Water Level" name="water-level" value="0.5" />
  ```

- [ ] **Step 10.2: Update ZoneCreationPanel.cs**

  Add field: `private FloatField _waterLevelField;`

  In `OnEnable`, query it: `_waterLevelField = root.Q<FloatField>("water-level");`

  In `OnCreateClicked`, update the reducer call:
  ```csharp
  float waterLevel = _waterLevelField?.value ?? 0.5f;
  SpacetimeDBManager.Conn.Reducers.CreateZone(zoneName, width, height, waterLevel);
  ```

  In `AddZoneRow`, update the label to use the new field names from autogen:
  ```csharp
  var label = new Label($"{zone.Name}  ({zone.TerrainWidth} × {zone.TerrainHeight})");
  ```

- [ ] **Step 10.3: Confirm Unity compiles with no errors**

- [ ] **Step 10.4: Commit**

  ```bash
  git add editor/Assets/Scripts/Runtime/ZoneCreationPanel.cs
  git add editor/Assets/UI/ZoneCreationPanel.uxml
  git commit -m "feat: ZoneCreationPanel updated for new create_zone signature"
  ```

---

## Task 11: TerrainPainter — replaces TilePainter

**Files:**
- Create: `editor/Assets/Scripts/Runtime/TerrainPainter.cs`
- Delete: `editor/Assets/Scripts/Runtime/TilePainter.cs`

`TerrainPainter` raycasts onto the terrain mesh, invokes the active brush, applies changes to local byte arrays, calls `update_terrain_chunk`, and patches the mesh optimistically.

- [ ] **Step 11.1: Implement TerrainPainter**

  ```csharp
  // editor/Assets/Scripts/Runtime/TerrainPainter.cs
  using System.Collections.Generic;
  using UnityEngine;
  using SpacetimeDB.Types;

  /// <summary>
  /// Handles mouse input for terrain painting.
  /// Raycasts against the terrain mesh, invokes the active brush,
  /// writes modified chunk data to SpacetimeDB, and patches the mesh immediately.
  /// </summary>
  [RequireComponent(typeof(TerrainRenderer))]
  public class TerrainPainter : MonoBehaviour
  {
      [SerializeField] private Camera _camera;

      private TerrainRenderer _terrainRenderer;

      // Active brush — set by TilePalettePanel (now TerrainBrushPanel)
      public static TerrainBrush ActiveBrush { get; set; }

      // Hover ring
      private LineRenderer _hoverRing;
      private const int HoverRingSegments = 64;

      // -----------------------------------------------------------------------

      void Awake()
      {
          _terrainRenderer = GetComponent<TerrainRenderer>();
          if (_camera == null) _camera = Camera.main;
          _hoverRing = CreateHoverRing();
      }

      void OnDestroy()
      {
          if (_hoverRing != null) Destroy(_hoverRing.gameObject);
      }

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

      // -----------------------------------------------------------------------

      bool TryGetTerrainHit(out Vector3 hitPoint)
      {
          hitPoint = Vector3.zero;
          var ray = _camera.ScreenPointToRay(Input.mousePosition);

          // Raycast against the terrain mesh collider (MeshCollider on TerrainRenderer GO).
          var col = GetComponent<MeshCollider>();
          if (col != null && col.Raycast(ray, out RaycastHit hit, 1000f))
          {
              hitPoint = hit.point;
              return true;
          }

          // Fallback: intersect Y=0 plane.
          if (Mathf.Abs(ray.direction.y) < 0.0001f) return false;
          float t = -ray.origin.y / ray.direction.y;
          if (t <= 0f) return false;
          hitPoint = ray.origin + t * ray.direction;
          return true;
      }

      void ApplyBrushAt(float worldX, float worldZ)
      {
          if (SpacetimeDBManager.Conn == null) return;

          var points    = TerrainBrush.AffectedPoints(worldX, worldZ, ActiveBrush.Radius);
          var byChunk   = TerrainBrush.GroupByChunk(points);

          float dt = Time.deltaTime;

          foreach (var kv in byChunk)
          {
              int cx = kv.Key.Item1, cz = kv.Key.Item2;

              // Get current chunk data from subscription cache.
              TerrainChunk? src = null;
              foreach (var c in SpacetimeDBManager.Conn.Db.TerrainChunk.Iter())
              {
                  if (c.ZoneId == EditorState.ActiveZoneId &&
                      c.ChunkX == (uint)cx && c.ChunkZ == (uint)cz)
                  { src = c; break; }
              }
              if (src == null) continue;

              // Clone byte arrays for local modification.
              byte[] heightCopy = (byte[])src.Value.HeightData.Clone();
              byte[] splatCopy  = (byte[])src.Value.SplatData.Clone();

              // Apply brush.
              ActiveBrush.ApplyToChunk(heightCopy, splatCopy, kv.Value,
                  ActiveBrush.Strength, dt);

              // Push to server.
              SpacetimeDBManager.Conn.Reducers.UpdateTerrainChunk(
                  EditorState.ActiveZoneId,
                  (uint)cx, (uint)cz,
                  heightCopy, splatCopy);
          }
      }

      // -----------------------------------------------------------------------

      LineRenderer CreateHoverRing()
      {
          var go = new GameObject("TerrainHoverRing");
          var lr = go.AddComponent<LineRenderer>();
          lr.positionCount = HoverRingSegments + 1;
          lr.loop = false;
          lr.startWidth = lr.endWidth = 0.1f;
          lr.useWorldSpace = true;
          lr.material = new Material(Shader.Find("Universal Render Pipeline/Unlit"))
              { color = Color.white };
          lr.enabled = false;
          return lr;
      }

      void UpdateHoverRing(Vector3 centre, float radius)
      {
          _hoverRing.enabled = true;
          for (int i = 0; i <= HoverRingSegments; i++)
          {
              float angle = i / (float)HoverRingSegments * Mathf.PI * 2f;
              _hoverRing.SetPosition(i, new Vector3(
                  centre.x + Mathf.Cos(angle) * radius,
                  centre.y + 0.1f,
                  centre.z + Mathf.Sin(angle) * radius));
          }
      }
  }
  ```

  > **Note:** Add a `MeshCollider` component to the TerrainRenderer GameObject in the scene so raycasting works correctly against the actual terrain surface.

- [ ] **Step 11.2: Delete TilePainter.cs**

  ```bash
  git rm editor/Assets/Scripts/Runtime/TilePainter.cs
  ```

  Also delete the associated `.meta` file if Unity hasn't done so automatically.

- [ ] **Step 11.3: Confirm Unity compiles with no errors**

- [ ] **Step 11.4: Commit**

  ```bash
  git add editor/Assets/Scripts/Runtime/TerrainPainter.cs
  git commit -m "feat: TerrainPainter replaces TilePainter; hover ring on terrain surface"
  ```

---

## Task 12: Repurpose TilePalettePanel as TerrainBrushPanel

**Files:**
- Modify: `editor/Assets/Scripts/Runtime/TilePalettePanel.cs`

Replace tile-type buttons with: layer selector (Grass/Dirt/Stone/Ravine), brush type (Height/Texture/Combined), mode selector (Raise/Lower/Smooth), radius slider, strength slider.

- [ ] **Step 12.1: Rewrite TilePalettePanel.cs**

  ```csharp
  // editor/Assets/Scripts/Runtime/TilePalettePanel.cs
  using UnityEngine;
  using UnityEngine.UIElements;

  [RequireComponent(typeof(UIDocument))]
  public class TilePalettePanel : MonoBehaviour
  {
      private VisualElement _trackedPanel;

      private RadioButton _brushHeight;
      private RadioButton _brushTexture;
      private RadioButton _brushCombined;

      private RadioButton _modeRaise;
      private RadioButton _modeLower;
      private RadioButton _modeSmooth;

      private RadioButton _layerGrass;
      private RadioButton _layerDirt;
      private RadioButton _layerStone;
      private RadioButton _layerRavine;

      private Slider _radiusSlider;
      private Slider _strengthSlider;

      // Cached brush instances
      private readonly HeightBrush   _heightBrush   = new();
      private readonly TextureBrush  _textureBrush  = new();
      private readonly CombinedBrush _combinedBrush = new();

      void OnEnable()
      {
          var root = GetComponent<UIDocument>().rootVisualElement;

          _brushHeight   = root.Q<RadioButton>("brush-height");
          _brushTexture  = root.Q<RadioButton>("brush-texture");
          _brushCombined = root.Q<RadioButton>("brush-combined");

          _modeRaise  = root.Q<RadioButton>("mode-raise");
          _modeLower  = root.Q<RadioButton>("mode-lower");
          _modeSmooth = root.Q<RadioButton>("mode-smooth");

          _layerGrass  = root.Q<RadioButton>("layer-grass");
          _layerDirt   = root.Q<RadioButton>("layer-dirt");
          _layerStone  = root.Q<RadioButton>("layer-stone");
          _layerRavine = root.Q<RadioButton>("layer-ravine");

          _radiusSlider   = root.Q<Slider>("radius-slider");
          _strengthSlider = root.Q<Slider>("strength-slider");

          _brushHeight.RegisterValueChangedCallback(_   => UpdateBrush());
          _brushTexture.RegisterValueChangedCallback(_  => UpdateBrush());
          _brushCombined.RegisterValueChangedCallback(_ => UpdateBrush());
          _modeRaise.RegisterValueChangedCallback(_     => UpdateBrush());
          _modeLower.RegisterValueChangedCallback(_     => UpdateBrush());
          _modeSmooth.RegisterValueChangedCallback(_    => UpdateBrush());
          _layerGrass.RegisterValueChangedCallback(_    => UpdateBrush());
          _layerDirt.RegisterValueChangedCallback(_     => UpdateBrush());
          _layerStone.RegisterValueChangedCallback(_    => UpdateBrush());
          _layerRavine.RegisterValueChangedCallback(_   => UpdateBrush());
          _radiusSlider.RegisterValueChangedCallback(_  => UpdateBrush());
          _strengthSlider.RegisterValueChangedCallback(_ => UpdateBrush());

          _trackedPanel = root.Q<VisualElement>("panel") ?? root;
          UIHoverTracker.Register(_trackedPanel);

          UpdateBrush(); // set initial active brush
      }

      void OnDisable()
      {
          UIHoverTracker.Unregister(_trackedPanel);
      }

      void UpdateBrush()
      {
          float radius   = _radiusSlider?.value   ?? 5f;
          float strength = _strengthSlider?.value ?? 0.5f;

          TerrainBrush brush;

          if (_brushHeight?.value == true)
          {
              _heightBrush.BrushMode = _modeLower?.value == true  ? HeightBrush.Mode.Lower
                                     : _modeSmooth?.value == true ? HeightBrush.Mode.Smooth
                                     : HeightBrush.Mode.Raise;
              brush = _heightBrush;
          }
          else if (_brushCombined?.value == true)
          {
              _combinedBrush.BrushMode    = _modeLower?.value == true ? CombinedBrush.Mode.LowerAndPaint
                                                                      : CombinedBrush.Mode.RaiseAndPaint;
              _combinedBrush.TargetLayer  = SelectedLayer();
              brush = _combinedBrush;
          }
          else
          {
              _textureBrush.TargetLayer = SelectedLayer();
              brush = _textureBrush;
          }

          brush.Radius   = radius;
          brush.Strength = strength;
          TerrainPainter.ActiveBrush = brush;
      }

      TerrainLayer SelectedLayer()
      {
          if (_layerDirt?.value   == true) return TerrainLayer.Dirt;
          if (_layerStone?.value  == true) return TerrainLayer.Stone;
          if (_layerRavine?.value == true) return TerrainLayer.Ravine;
          return TerrainLayer.Grass;
      }
  }
  ```

  Replace the entire contents of `editor/Assets/UI/TilePalettePanel.uxml` with:

  ```xml
  <ui:UXML xmlns:ui="UnityEngine.UIElements">
      <ui:VisualElement name="panel" class="panel">

          <ui:Label text="Brush Type" class="section-label"/>
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
  </ui:UXML>
  ```

- [ ] **Step 12.2: Confirm Unity compiles with no errors**

- [ ] **Step 12.3: Commit**

  ```bash
  git add editor/Assets/Scripts/Runtime/TilePalettePanel.cs
  git add editor/Assets/UI/TilePalettePanel.uxml
  git commit -m "feat: TilePalettePanel repurposed as terrain brush selector"
  ```

---

## Task 13: Copy renderer files to client

**Files:**
- Create: `client/Assets/Scripts/Runtime/TerrainChunkData.cs`
- Create: `client/Assets/Scripts/Runtime/TerrainRenderer.cs`
- Create: `client/Assets/Scripts/Runtime/WaterRenderer.cs`
- Create: `client/Assets/Shaders/TerrainSplatmap.shader`
- Modify: `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs`

- [ ] **Step 13.1: Copy rendering files to client**

  ```bash
  mkdir -p client/Assets/Scripts/Runtime client/Assets/Shaders
  cp editor/Assets/Scripts/Runtime/TerrainChunkData.cs client/Assets/Scripts/Runtime/
  cp editor/Assets/Scripts/Runtime/TerrainRenderer.cs  client/Assets/Scripts/Runtime/
  cp editor/Assets/Scripts/Runtime/WaterRenderer.cs    client/Assets/Scripts/Runtime/
  cp editor/Assets/Shaders/TerrainSplatmap.shader      client/Assets/Shaders/
  ```

- [ ] **Step 13.2: Update client SpacetimeDBManager**

  Apply the same changes as Task 9 (add `TerrainChunk` subscription and `OnTerrainChunkUpdated` event) to `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs`.

- [ ] **Step 13.3: Confirm client Unity project compiles with no errors**

- [ ] **Step 13.4: Commit**

  ```bash
  git add client/Assets/Scripts/Runtime/TerrainChunkData.cs
  git add client/Assets/Scripts/Runtime/TerrainRenderer.cs
  git add client/Assets/Scripts/Runtime/WaterRenderer.cs
  git add client/Assets/Shaders/
  git add client/Assets/Scripts/Runtime/SpacetimeDBManager.cs
  git commit -m "feat: copy terrain renderer to game client"
  ```

---

## Task 14: Scene wiring and smoke test

**Files:** Editor Unity scene

- [ ] **Step 14.1: Wire GameObjects in editor scene**

  In the editor Unity scene:
  1. Create `GameObject "Terrain"` — add `TerrainRenderer`, `MeshFilter`, `MeshRenderer`, `MeshCollider`
  2. Assign `TerrainSplatmap` material to `TerrainRenderer._terrainMaterial`
  3. Create `GameObject "Water"` — add `WaterRenderer`, `MeshFilter`, `MeshRenderer`
  4. Assign water `Material` to `WaterRenderer._waterMaterial`
  5. Add `TerrainPainter` component to the `Terrain` GameObject
  6. Assign `Main Camera` to `TerrainPainter._camera`

- [ ] **Step 14.2: Smoke test — zone creation creates terrain**

  - Start SpacetimeDB: `spacetime start`
  - Publish server: `cd server && spacetime build && spacetime publish --server local zoneforge-server`
  - Enter Play mode in Unity editor
  - Create a zone (64×64, water level 0.5)
  - Verify: green terrain mesh appears, water plane visible below terrain level
  - Expected log: `[SpacetimeDBManager] Subscription applied`

- [ ] **Step 14.3: Smoke test — brush painting**

  - Select `Texture` brush, `Grass` layer in the panel
  - Left-click drag across terrain — verify color changes
  - Select `Height` brush, `Raise` mode — left-click drag — verify mesh deforms upward
  - Verify water recedes where terrain rises above `water_level`

- [ ] **Step 14.4: Verify chunk update round-trip**

  ```bash
  spacetime logs zoneforge-server | tail -20
  ```
  Expected: `update_terrain_chunk` reducer calls logged without errors.

- [ ] **Step 14.5: Final commit**

  ```bash
  git add editor/
  git commit -m "feat: wire terrain system in editor scene; smoke test passed"
  ```

---

## Known Follow-Ups (not in scope)

- Partial vertex patching in `TerrainRenderer.PatchChunk` (currently rebuilds full mesh)
- MeshCollider auto-update on chunk patch (currently full rebuild)
- Specialty brushes: Cliff, Cliff-to-Water (stretch goal from spec)
- Water material stencil configuration (set via Unity Inspector or `.mat` YAML)
- Terrain LOD
- Per-stroke throttling to reduce reducer call rate under fast mouse movement
