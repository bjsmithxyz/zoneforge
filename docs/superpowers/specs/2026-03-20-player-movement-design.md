# Player Movement — Design Spec

**Date:** 2026-03-20
**Phase:** 2, Group 6 (items 1–4)
**Scope:** Client-side player movement with server-side prediction and reconciliation

---

## Overview

Implement real-time player movement across the ZoneForge client and server. Players move with WASD (camera-relative), position is applied locally immediately (client-side prediction), and authoritative position is received back from the server via the `Player` table subscription and used for reconciliation. Remote players lerp smoothly to their server-reported positions.

---

## Architecture

Three components work together:

### `PlayerManager` (client, new)

Scene singleton MonoBehaviour. Owns the full player lifecycle.

**Identity availability:** `SpacetimeDBManager.Conn.Identity` is populated when `OnConnect` fires (before `OnSubscriptionApplied`). It is valid for all events that follow, including `OnPlayerInserted` and `OnConnected`.

**Subscription timing and backfill:** `PlayerManager` subscribes to `SpacetimeDBManager` events in `Awake`. However, `SpacetimeDBManager` registers `Conn.Db.Player.OnInsert` inside `OnSubscriptionApplied` — after the initial snapshot rows are already in the local cache. This means initial rows do NOT trigger `OnPlayerInserted`. This matches the established pattern in the codebase (see editor zone backfill). Therefore `PlayerManager.OnConnected` must iterate `Conn.Db.Player.Iter()` to spawn all existing player rows, in addition to catching future inserts via `OnPlayerInserted`.

**Dependency:** This component subscribes to `SpacetimeDBManager.OnPlayerDeleted`, which does not yet exist and must be added to `SpacetimeDBManager` as part of this work unit (see Modified files).

**Lifecycle:**

- `Awake`: subscribe to `SpacetimeDBManager.OnPlayerInserted`, `OnPlayerUpdated`, `OnPlayerDeleted`, `OnConnected`
- On `OnPlayerInserted`: spawn a capsule GameObject, attach `PlayerController`, set `isLocal = (player.Identity == Conn.Identity)`. Key the GameObject in a `Dictionary<ulong, GameObject>` by `player.Id`. Guard with a dictionary-contains check to avoid duplicates with the `OnConnected` backfill.
- On `OnPlayerUpdated` (`Action<Player, Player>`): look up by `newPlayer.Id`, call `controller.ReceiveServerPosition(...)`. Only `newPlayer` is used; `oldPlayer` is ignored.
- On `OnPlayerDeleted`: look up by `player.Id`, destroy the GameObject, remove from dictionary.
- On `OnConnected`: iterate `Conn.Db.Player.Iter()` and spawn any player whose `Id` is not already in the dictionary. Then check whether any row has `Identity == Conn.Identity`; call `create_player("Player")` only if none is found. (`Conn.Identity` and `player.Identity` are both `SpacetimeDB.Identity` — the same type — so equality comparison is direct.)

### `PlayerController` (client, new)

MonoBehaviour attached to each player capsule. Parameterized at spawn time via `isLocal`.

**Y enforcement:** After every position write (prediction, reconciliation, remote lerp), force `transform.position.y = 1.0f`. This must be explicit in all three code paths — the camera is parented to this GameObject and any Y drift causes a visible lurch.

**Local player (`isLocal = true`):**

- Read `Input.GetAxisRaw("Horizontal")` / `"Vertical"` each `Update`.
- Project camera forward/right onto the X/Z plane (zero Y, normalize) and blend with input axes to get a camera-relative direction vector.
- Apply `transform.position += dir * speed * Time.deltaTime` immediately (prediction), then force `transform.position.y = 1.0f`.
- Send `MovePlayer` reducer at 10 Hz (every 0.1 s) only if position has moved more than `0.01f` units from `_lastSentPosition` (X/Z distance check). Update `_lastSentPosition` on each send. `_lastSentPosition` is initialised at construction time to the player's spawn position (`new Vector3(player.PositionX, 1.0f, player.PositionY)`) to prevent a spurious send on the first frame.
- Subscribe to `Conn.Reducers.OnMovePlayer` and log a warning on error (failed reducer calls — e.g., zone not found — are handled by the next valid `OnPlayerUpdated` reconciliation; no explicit rollback is needed).

**`ReceiveServerPosition(Vector3 serverPos)`:**

- Force `serverPos.y = 1.0f` before any comparison or assignment.
- Compute `dist = Vector3.Distance(transform.position, serverPos)`.
- If `dist > 1.0f` (see threshold note below): store `_reconcileTarget = serverPos`, store `_reconcileSpeed = dist / 0.2f` (computed once; not recomputed from remaining distance each frame). Cancel any in-progress correction — the new target and speed replace the old ones.
- If `dist ≤ 1.0f` and a reconciliation is in progress: cancel it (new server position is close enough that continuing toward the old target would be wrong).
- If `dist ≤ 1.0f` and no reconciliation is in progress: ignore the delta to prevent jitter.
- Each `Update` while reconciling: `transform.position = Vector3.MoveTowards(transform.position, _reconcileTarget, _reconcileSpeed * Time.deltaTime)`, then force `y = 1.0f`. Clear the reconciliation state when `MoveTowards` reaches the target.

**Threshold note:** At 5 units/sec with 10 Hz sends, the server position lags by up to 0.5 units in normal play (one send interval × speed). Using a 1.0 f threshold (2× the expected drift) prevents spurious corrections from normal round-trip latency while still catching genuine desyncs.

**Known limitation:** The camera is parented to the local player GameObject (see Camera section). During a reconciliation correction the camera will stutter. This is acceptable for this pass; a smoothed camera follow can be added later.

**Remote player (`isLocal = false`):**

- `ReceiveServerPosition`: force `serverPos.y = 1.0f`, store as `_targetPosition`.
- Each `Update`: `transform.position = Vector3.Lerp(transform.position, _targetPosition, Time.deltaTime * 10f)`, then force `y = 1.0f`.

### Server (`lib.rs`, modified)

**`create_player` — pre-existing reducer, made idempotent:**
Signature: `pub fn create_player(ctx: &ReducerContext, name: String)`. Already exists. Before inserting, check `ctx.db.player().identity().find(ctx.sender())` — valid because `identity` is `#[unique]` on `Player`. If a row is found, return early (no-op).

**`move_player` — bounds clamping added:**
Current signature: `pub fn move_player(ctx: &ReducerContext, new_x: f32, new_y: f32)`. Change return type to `Result<(), String>`. After finding the player row, look up their zone via `ctx.db.zone().id().find(&player.zone_id)`. Return `Err("Zone not found")` if missing. Clamp `new_x` to `[0.0, zone.terrain_width as f32]` and `new_y` to `[0.0, zone.terrain_height as f32]` (inclusive upper bound — player may stand on the zone boundary edge). Then update the row. Note: on the client, these same bounds are available as `zone.TerrainWidth` / `zone.TerrainHeight` from the autogen `Zone` type — not from `ZoneController.GridWidth` / `GridHeight`, which are named differently.

---

## Coordinate Mapping

| Server field | Unity axis |
| --- | --- |
| `Player.PositionX` | `transform.position.x` |
| `Player.PositionY` | `transform.position.z` |
| (fixed) | `transform.position.y = 1.0f` (enforced after every write) |

`MovePlayer` reducer calls pass `(transform.position.x, transform.position.z)`.

---

## Movement Parameters

| Parameter | Value | Notes |
| --- | --- | --- |
| Move speed | `5f` units/sec | `[SerializeField]`, tunable |
| Reducer send rate | 10 Hz (every 0.1 s) | Only sends if moved > 0.01 units from last send |
| Reconciliation threshold | `1.0f` units | 2× expected drift at 5 u/s + 0.1 s send interval |
| Reconciliation speed | `distance / 0.2f` units/sec | Stored once on correction start; not recomputed per-frame |
| Remote lerp factor | `10f` | `Time.deltaTime * 10f` |

---

## Camera

The main camera is parented to the local player GameObject and positioned/angled for an isometric view (e.g., offset `(0, 10, -7)`, rotation `(55°, 0°, 0°)`). No separate camera follow system — parenting is sufficient for this pass.

---

## Files

### New

- `client/Assets/Scripts/Player/PlayerManager.cs`
- `client/Assets/Scripts/Player/PlayerController.cs`

### Modified

- `server/spacetimedb/src/lib.rs` — idempotent `create_player`; return type + bounds clamping in `move_player`
- `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs` — add `OnPlayerDeleted` static event, wire `Conn.Db.Player.OnDelete` in `OnSubscriptionApplied`
- `client/Assets/Scripts/ConnectionTest.cs` — remove the single `ctx.Reducers.CreatePlayer(...)` call in `OnSubscriptionApplied`; no other changes. Also remove the `ConnectionTest` GameObject from the active scene — leaving it active would create a second independent connection with a separate identity, causing `SpacetimeDBManager` to never see the `ConnectionTest` player row.

### Not changed

- No server schema changes → no `spacetime generate` needed

---

## Out of Scope (future items)

- NavMesh / collision detection (Group 6, item 5)
- Multi-client stress testing (Group 6, item 6)
- Full rollback/replay reconciliation (current design uses `MoveTowards`, not replay)
- Smoothed camera follow (camera stutter during reconciliation is a known limitation of parenting)
