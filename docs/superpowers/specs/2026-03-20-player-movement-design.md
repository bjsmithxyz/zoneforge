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
Scene singleton MonoBehaviour. Owns the full player lifecycle:

- On `SpacetimeDBManager.OnConnected`: check `Conn.Db.Player.Iter()` for a row matching the local identity. Call `create_player` only if none is found.
- On `SpacetimeDBManager.OnPlayerInserted`: spawn a capsule GameObject, attach `PlayerController`, set `isLocal = (player.Identity == Conn.Identity)`.
- On `SpacetimeDBManager.OnPlayerUpdated`: forward new position to the matching `PlayerController`.
- On player delete: destroy the capsule GameObject.

### `PlayerController` (client, new)
MonoBehaviour attached to each player capsule. Parameterized at spawn time via `isLocal`.

**Local player (`isLocal = true`):**
- Read `Input.GetAxisRaw("Horizontal")` / `"Vertical"` each `Update`.
- Project camera forward/right onto the X/Z plane (zero Y, normalize) and blend with input axes to get a camera-relative direction vector.
- Apply `transform.position += dir * speed * Time.deltaTime` immediately (prediction).
- Send `MovePlayer` reducer at 10 Hz (every 0.1 s) only if position has changed.
- On `ReceiveServerPosition`: if distance to server position > 0.5 units, lerp toward it over 0.2 s. Differences below 0.5 units are ignored to avoid jitter.

**Remote player (`isLocal = false`):**
- On `ReceiveServerPosition`: store target position.
- Each `Update`: `Vector3.Lerp(current, target, Time.deltaTime * 10f)`.

### Server (`lib.rs`, modified)

**`create_player` — idempotent:**
Check `ctx.db.player().identity().find(ctx.sender())` before inserting. If a row exists, return early. Prevents duplicate player rows on reconnect.

**`move_player` — bounds clamping:**
After finding the player row, look up their zone via `ctx.db.zone().id().find(&player.zone_id)`. Clamp `new_x` to `[0, zone.terrain_width as f32]` and `new_y` to `[0, zone.terrain_height as f32]`. Return `Err` if zone is not found.

---

## Coordinate Mapping

The server stores positions as 2D floats (`position_x`, `position_y`). Unity is 3D on the X/Z plane (Y = up).

| Server field | Unity axis |
|---|---|
| `Player.PositionX` | `transform.position.x` |
| `Player.PositionY` | `transform.position.z` |
| (fixed) | `transform.position.y = 1.0f` |

`MovePlayer` reducer calls pass `(transform.position.x, transform.position.z)`.

---

## Movement Parameters

| Parameter | Value | Notes |
|---|---|---|
| Move speed | `5f` units/sec | `[SerializeField]`, tunable |
| Reducer send rate | 10 Hz (every 0.1 s) | Only sends if position changed |
| Reconciliation threshold | `0.5f` units | Below this, ignore server delta |
| Reconciliation lerp time | `0.2f` s | Smooth correction |
| Remote lerp speed | `10f` (lerp factor) | `Time.deltaTime * 10f` |

---

## Camera

The main camera is parented to the local player GameObject and positioned for an isometric view. No separate camera follow system — parenting is sufficient for this pass.

---

## Files

### New
- `client/Assets/Scripts/Player/PlayerManager.cs`
- `client/Assets/Scripts/Player/PlayerController.cs`

### Modified
- `server/spacetimedb/src/lib.rs` — idempotent `create_player`, bounds-clamping `move_player`
- `client/Assets/Scripts/ConnectionTest.cs` — remove `create_player` call (PlayerManager owns this)

### Not changed
- No server schema changes → no `spacetime generate` needed
- `SpacetimeDBManager.cs` — already exposes `OnPlayerInserted` / `OnPlayerUpdated` events; no changes needed

---

## Out of Scope (future items)

- NavMesh / collision detection (Group 6, item 5)
- Multi-client stress testing (Group 6, item 6)
- Full rollback/replay reconciliation (current design uses lerp, not replay)
