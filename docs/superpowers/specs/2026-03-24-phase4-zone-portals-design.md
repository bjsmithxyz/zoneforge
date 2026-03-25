# Phase 4 — Zone Portals Design

**Date:** 2026-03-24
**Scope:** Group 10 (Zone Portals) + editor toolbar refactor

---

## Summary

Add zone portal travel to ZoneForge: players walk through portal trigger volumes to move between zones. The system spans the Rust server module, the Unity game client, and the Unity standalone editor.

---

## Key Decisions

| Decision | Choice | Rationale |
| --- | --- | --- |
| Subscription scope | Per-zone filtered queries | Scales beyond 3 zones; all-zones approach won't hold past ~10 |
| Transfer UX | Instant with predictive pre-load + Ripple Warp animation | Best player feel; animation window covers subscription gap |
| Portal crossing effect | Ripple Warp (concentric cyan rings, ~0.6s) | Soft, magical — fits ZoneForge tone |
| Portal data model | One row per connection, `bidirectional` bool | Handles dungeons (one-way drop-ins) and standard return portals equally |
| Editor World Graph | Node canvas + detail panel, collapsed bottom panel | Scales to many portals per zone; visible only when editing |
| Editor toolbar | Icon buttons top-left (Zone Manager + World Graph) and top-right (Brush) | Clean; consistent; placeholder icons for now |

---

## Server

### Portal Table

```text
portal
  id               u64     PK auto-inc
  source_zone_id   u64     btree index
  dest_zone_id     u64
  source_x         f32     world position of trigger volume in source zone
  source_y         f32
  dest_spawn_x     f32     player arrival position in dest zone
  dest_spawn_y     f32
  bidirectional    bool    if true, also acts as dest→source portal
  label            String  optional display name ("To Village", "Dungeon Entrance")
```

### Reducers

**`create_portal`** (admin only)

- Validates both `source_zone_id` and `dest_zone_id` exist
- Validates `source_x/y` is within source zone bounds
- Validates `dest_spawn_x/y` is within dest zone bounds
- Inserts portal row

**`delete_portal`** (admin only)

- Validates portal exists, then deletes

**`enter_zone`** (any player)

- Looks up portals where `source_zone_id = player.zone_id`
- Finds one within 2 units of the player's current position
- If `bidirectional`, also checks portals where `dest_zone_id = player.zone_id` and the player is within 2 units of `dest_spawn_x/y`. For bidirectional portals `dest_spawn_x/y` is the portal mouth on the destination side — arriving players land there and can walk back through the same point. No extra coordinate field is required.
- Updates `player.zone_id`, `player.position_x`, `player.position_y` to the destination spawn point (or back to `source_x/y` when traversing a bidirectional portal in reverse)
- Returns `Result<(), String>` — errors surface to client if player is out of range

---

## Client (zoneforge-client)

### Subscription Management

`SpacetimeDBManager` gains a `CurrentZoneId` field. On connect, initial subscriptions filter heavy tables by zone:

```sql
SELECT * FROM terrain_chunk WHERE zone_id = {CurrentZoneId}
SELECT * FROM entity_instance WHERE zone_id = {CurrentZoneId}
SELECT * FROM enemy WHERE zone_id = {CurrentZoneId}
SELECT * FROM portal WHERE source_zone_id = {CurrentZoneId}
SELECT * FROM portal WHERE dest_zone_id = {CurrentZoneId}
```

The two portal queries together ensure all portal rings are rendered — both portals originating in the current zone and bidirectional portals arriving here.

`spawn_point` is not subscribed by the client — it is an editor-only concern.

Light tables remain unfiltered: `player`, `zone`, `ability`, `combat_log`, `status_effect`, `player_cooldown`, `enemy_def`.

### Pre-load and Rendering Gate

`SpacetimeDBManager.CurrentZoneId` is the rendering gate. It is only updated **after the transfer animation completes**, not when pre-load begins.

**Pre-load** (when player enters the ~5-unit proximity radius):

1. A second `SubscriptionBuilder().Subscribe()` call is made on the existing `DbConnection` for the destination zone queries. SpacetimeDB 2.x subscriptions are additive — both sets are live simultaneously.
2. `OnInsert` callbacks fire for destination zone rows (terrain chunks, enemies, entities, portals). Each manager checks `if (row.ZoneId != SpacetimeDBManager.CurrentZoneId) return;` — rows enter the SDK local cache but no GameObjects are spawned.

**Post-transfer rendering** (after animation completes):

1. `ZoneTransferManager` sets `SpacetimeDBManager.CurrentZoneId` to the new zone once the Player row `zone_id` confirms (transfer sequence step 3).
2. Each manager backfills from the local cache — same pattern as the existing `OnConnected` backfill in `EnemyManager` and `PlayerManager` — spawning any rows that now match `CurrentZoneId`.

**Cleanup** (background, invisible to player):

1. ~2s after transfer, `SpacetimeDBManager` performs a full `Disconnect()` then reconnects via `DbConnection.Builder()...Build()` with only the new zone's filtered queries.
2. The saved auth token in `PlayerPrefs` ensures the same Identity is resumed — the player's server row is unchanged.
3. All managers backfill from the fresh subscription state on `OnSubscriptionApplied`.

### New Components

**`PortalManager`** (scene singleton, mirrors `EnemyManager` pattern)

- Subscribes to `Portal` inserts/deletes for the current zone
- Spawns an invisible cylinder trigger collider at each `source_x/y`
- Renders a glowing ring indicator (URP emissive material, cyan by default)
- Two proximity radii per portal:
  - **Pre-load radius (~5 units):** subscribes to destination zone queries
  - **Trigger radius (~1.5 units):** begins transfer sequence

**`ZoneTransferManager`** (scene singleton)

Transfer sequence on trigger entry:

1. Call `Reducer.EnterZone(conn, portalId)`
2. Play Ripple Warp screen-space effect (0.6s, concentric cyan rings expanding from screen centre)
3. Wait for `Player.OnUpdate` confirming new `zone_id` on local player row
4. Set `SpacetimeDBManager.CurrentZoneId` to the new zone — unblocks the rendering gate
5. Each manager backfills from the SDK local cache (destination rows already received during pre-load) — new zone GameObjects spawn immediately
6. Destroy old zone GameObjects (terrain meshes, entity capsules, enemy capsules, portal volumes)
7. After ~2s, trigger background reconnect to flush old zone subscription from memory

**`RippleWarpEffect`** (screen-space effect controller)

- Plays the concentric cyan ring animation over ~0.6s using a URP full-screen render pass or a UI overlay canvas
- Exposed as a single `Play(Action onComplete)` call — `ZoneTransferManager` passes its continuation as the callback

### PlayerManager Zone Awareness

`PlayerManager.OnPlayerUpdated` gains zone filtering:

- If an updated player's `zone_id` no longer matches the local player's zone → destroy that player's capsule
- If an updated player's `zone_id` newly matches the local player's zone → spawn their capsule

This ensures other players appear and disappear correctly during zone transfers without any additional server work.

### New Events on SpacetimeDBManager

```csharp
public static event Action<Portal> OnPortalInserted;
public static event Action<Portal> OnPortalDeleted;
```

Both are wired up inside `OnSubscriptionApplied`, matching the existing callback pattern.

---

## Editor (zoneforge-editor)

### Toolbar Refactor

Replace text panel headers with icon buttons. All icons are placeholders — final art TBD.

**Top-left (horizontal strip):**

- 🗺 Zone Manager icon — toggles the existing Zone Manager dropdown panel
- ⬡ World Graph icon — toggles the World Graph bottom panel

**Top-right:**

- 🖌 Brush icon — toggles the existing Brush panel

Active panel icon highlights blue. Inactive icons use a dim border. At most one panel per group open at a time.

### World Graph Panel

Collapsed bottom panel (0px height) by default. Expands to ~90px when:

- The ⬡ icon is clicked
- A portal ring in the 3D viewport is clicked (auto-selects that portal in the graph)

Collapses when:

- The ⬡ icon is clicked again
- The user clicks away in the viewport with no portal selected

**Layout when expanded:**

Left side (flex 1.7) — draggable zone node canvas:

- One node per zone row, colour-coded by zone
- Directed arrows between nodes for each portal
- Selected portal edge highlighted yellow

Right side (fixed 148px) — portal detail panel:

- `source → dest` label
- Editable spawn point coordinates (X/Y)
- Bidirectional toggle
- Place button (sets `source_x/y` from the next viewport click)
- Delete button

Node positions are auto-laid-out on each open — not persisted.

### Portal Placement in 3D Viewport

Mirrors the existing entity placement pattern:

1. Select a portal edge in the World Graph (or create a new connection via the detail panel)
2. Click Place — cursor enters placement mode
3. Click in the 3D viewport to set `source_x/y` (raycasts against terrain collider)
4. Calls `create_portal` reducer with the selected source/dest zones and clicked position

Portal ring indicators are rendered in the viewport at all `source_x/y` positions for the current zone via `PortalRenderer.cs`.

---

## Out of Scope (Phase 4)

- Animated portal mesh (shader effect around the ring) — placeholder emissive ring for now
- Sound effects on portal entry
- Zone minimap or overhead world map
- Cloud deployment (Group 11 — separate session)

---

## Files Affected

### Server Module

- `server/spacetimedb/src/lib.rs` — add `Portal` table; add `create_portal`, `delete_portal`, `enter_zone` reducers

### Client (`client/Assets/Scripts/`)

- `Runtime/SpacetimeDBManager.cs` — zone-filtered subscriptions, `CurrentZoneId`, portal events, background reconnect logic
- `Zone/ZoneTransferManager.cs` — new file
- `Zone/PortalManager.cs` — new file
- `Zone/RippleWarpEffect.cs` — new file (screen-space effect controller)
- `Player/PlayerManager.cs` — zone-aware capsule spawning/destroying

### Editor (`editor/Assets/Scripts/`)

- `UI/ToolbarController.cs` — new file (icon button strip, panel toggle logic)
- `UI/WorldGraphPanel.cs` — new file (node canvas + detail panel)
- `UI/WorldGraphPanel.uxml` — new file
- `UI/WorldGraphPanel.uss` — new file
- `Zone/PortalRenderer.cs` — new file (editor-side portal ring rendering and click detection; separate from the client `PortalManager` as the editor has its own `SpacetimeDBManager` instance)
- `UI/ZoneCreationPanel.cs` — minor: remove text header, hook into toolbar toggle
- `UI/TilePalettePanel.cs` — minor: remove text header, hook into toolbar toggle

### Autogen (both client and editor)

- Regenerate after `Portal` table and reducers are added
