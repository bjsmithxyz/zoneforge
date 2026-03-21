# Player Movement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement real-time WASD player movement with client-side prediction, server reconciliation, and remote player lerping across the ZoneForge client and server.

**Architecture:** `PlayerManager` (scene singleton) owns player lifecycle — spawning capsule GameObjects and dispatching server updates to `PlayerController` components. `PlayerController` handles both local (input + prediction + throttled reducer calls) and remote (lerp-to-server) players via an `IsLocal` flag. Server `move_player` gains bounds clamping; `create_player` becomes idempotent.

**Tech Stack:** Unity 2022.3 LTS C#, SpacetimeDB C# SDK 2.x, SpacetimeDB Rust server module (WASM), `spacetime` CLI

---

## File Map

| File | Action | Responsibility |
| --- | --- | --- |
| `server/spacetimedb/src/lib.rs` | Modify | Idempotent `create_player`; bounds-clamping `move_player` |
| `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs` | Modify | Add `LocalIdentity`, `OnPlayerDeleted` event |
| `client/Assets/Scripts/ConnectionTest.cs` | Modify | Remove `CreatePlayer` call |
| `client/Assets/Scripts/Player/PlayerController.cs` | Create | Per-player movement logic (local + remote) |
| `client/Assets/Scripts/Player/PlayerManager.cs` | Create | Player lifecycle: spawn, despawn, backfill, create_player |

**Scene change (Unity Editor):** Remove the `ConnectionTest` GameObject from the active scene after Task 3.

---

## Task 1: Server — Idempotent `create_player`

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Step 1: Add early-return guard to `create_player`**

Open `server/spacetimedb/src/lib.rs`. Replace the `create_player` reducer with:

```rust
#[reducer]
pub fn create_player(ctx: &ReducerContext, name: String) {
    // Idempotent: skip if this identity already has a player row
    if ctx.db.player().identity().find(ctx.sender()).is_some() {
        log::info!("create_player: identity already exists, skipping");
        return;
    }
    let player = Player {
        id: 0,
        identity: ctx.sender(),
        name,
        zone_id: 1,
        position_x: 0.0,
        position_y: 0.0,
        health: 100,
        max_health: 100,
    };
    ctx.db.player().insert(player);
    log::info!("Player created: {}", ctx.sender());
}
```

- [ ] **Step 2: Build the server to verify it compiles**

```bash
cd server && spacetime build
```

Expected: build completes with no errors (ignore the `wasm-opt` warning — it's non-critical).

- [ ] **Step 3: Commit**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): make create_player idempotent"
```

---

## Task 2: Server — Bounds-clamping `move_player` + deploy

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Step 1: Update `move_player` to clamp to zone bounds**

Replace the `move_player` reducer with:

```rust
#[reducer]
pub fn move_player(ctx: &ReducerContext, new_x: f32, new_y: f32) -> Result<(), String> {
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or_else(|| "Player not found".to_string())?;

    let zone = ctx.db.zone().id().find(&player.zone_id)
        .ok_or_else(|| "Zone not found".to_string())?;

    let clamped_x = new_x.clamp(0.0, zone.terrain_width as f32);
    let clamped_y = new_y.clamp(0.0, zone.terrain_height as f32);

    ctx.db.player().id().update(Player {
        position_x: clamped_x,
        position_y: clamped_y,
        ..player
    });
    log::info!("Player moved to ({}, {})", clamped_x, clamped_y);
    Ok(())
}
```

- [ ] **Step 2: Build**

```bash
cd server && spacetime build
```

Expected: build completes with no errors.

- [ ] **Step 3: Deploy to local SpacetimeDB**

Make sure `spacetime start` is running in another terminal first.

```bash
cd server && spacetime publish --server local zoneforge-server
```

Expected: `Successfully published module` (or similar). If you see `--delete-data` may be required, the schema hasn't changed — this shouldn't happen since we only changed reducer logic, not table structure.

- [ ] **Step 4: Verify in server logs**

```bash
spacetime logs zoneforge-server
```

Expected: no errors during module load.

- [ ] **Step 5: Commit**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add zone bounds clamping to move_player"
```

---

## Task 3: Client — Update `SpacetimeDBManager`

**Files:**
- Modify: `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs`

`SpacetimeDBManager` needs two additions: store `LocalIdentity` (so `PlayerManager` can identify the local player), and expose `OnPlayerDeleted`.

- [ ] **Step 1: Add `LocalIdentity` property and `OnPlayerDeleted` event**

Add these two lines to the static properties/events block (after the existing events):

```csharp
public static Identity LocalIdentity { get; private set; }
public static event Action<Player> OnPlayerDeleted;
```

- [ ] **Step 2: Store the identity in `OnConnect`**

In the `OnConnect` method, add one line before the subscription call:

```csharp
void OnConnect(DbConnection conn, Identity identity, string token)
{
    Debug.Log($"[SpacetimeDBManager] Connected. Identity: {identity}");
    LocalIdentity = identity;   // ← add this line
    conn.SubscriptionBuilder()
        ...
```

- [ ] **Step 3: Wire `OnDelete` in `OnSubscriptionApplied`**

In `OnSubscriptionApplied`, add one line after the existing `Player` event wiring:

```csharp
Conn.Db.Player.OnInsert += (eventCtx, player) => OnPlayerInserted?.Invoke(player);
Conn.Db.Player.OnUpdate += (eventCtx, oldPlayer, newPlayer) => OnPlayerUpdated?.Invoke(oldPlayer, newPlayer);
Conn.Db.Player.OnDelete += (eventCtx, player) => OnPlayerDeleted?.Invoke(player);   // ← add this
```

- [ ] **Step 4: Verify Unity compiles**

Open Unity. Check the Console window (`Window → General → Console`). There should be no red errors. Yellow warnings are fine.

- [ ] **Step 5: Commit**

```bash
git add client/Assets/Scripts/Runtime/SpacetimeDBManager.cs
git commit -m "feat(client): add LocalIdentity and OnPlayerDeleted to SpacetimeDBManager"
```

---

## Task 4: Client — Fix `ConnectionTest.cs` and remove from scene

**Files:**
- Modify: `client/Assets/Scripts/ConnectionTest.cs`
- Scene: remove `ConnectionTest` GameObject (Unity Editor step)

`ConnectionTest` was a debug stub that creates its own `DbConnection` and calls `CreatePlayer` on connect. With `PlayerManager` owning player creation, this must be cleaned up — otherwise two connections with different identities run simultaneously.

- [ ] **Step 1: Remove the `CreatePlayer` call**

In `ConnectionTest.cs`, find `OnSubscriptionApplied` and delete the `CreatePlayer` line:

```csharp
void OnSubscriptionApplied(SubscriptionEventContext ctx)
{
    Debug.Log("Subscription applied — calling create_player reducer");
    ctx.Reducers.CreatePlayer("TestPlayer");   // ← delete this line
}
```

After the edit, `OnSubscriptionApplied` should just contain the debug log.

- [ ] **Step 2: Remove the `ConnectionTest` GameObject from the active scene**

In Unity:
1. Open the Hierarchy window.
2. Find the GameObject with the `ConnectionTest` component.
3. Delete it (`Delete` key or right-click → Delete).
4. Save the scene (`Ctrl+S` / `Cmd+S`).

Do not delete the `ConnectionTest.cs` script file itself — keep it for reference.

- [ ] **Step 3: Verify Unity compiles**

Check the Console for red errors. There should be none.

- [ ] **Step 4: Commit**

```bash
git add client/Assets/Scripts/ConnectionTest.cs
# Also commit the scene file that changed when you removed the GameObject
git add client/Assets/Scenes/
git commit -m "fix(client): remove ConnectionTest CreatePlayer call and GameObject from scene"
```

---

## Task 5: Client — Create `PlayerController`

**Files:**
- Create: `client/Assets/Scripts/Player/PlayerController.cs`

`PlayerController` is a MonoBehaviour attached to each player capsule. It handles both local (WASD + prediction + throttled send + reconciliation) and remote (lerp) player modes.

- [ ] **Step 1: Create the `Player/` directory and `PlayerController.cs`**

Create `client/Assets/Scripts/Player/PlayerController.cs` with this content:

```csharp
using UnityEngine;
using SpacetimeDB;
using SpacetimeDB.Types;

/// <summary>
/// Attached to each player capsule. isLocal=true: reads WASD input, predicts
/// movement, throttles MovePlayer reducer calls. isLocal=false: lerps to server pos.
/// </summary>
public class PlayerController : MonoBehaviour
{
    public bool IsLocal { get; private set; }

    [SerializeField] private float _speed = 5f;

    // Local player: throttled reducer sends
    private Vector3 _lastSentPosition;
    private float _sendTimer;
    private const float SendInterval = 0.1f;

    // Local player: reconciliation
    private Vector3 _reconcileTarget;
    private float _reconcileSpeed;
    private bool _reconciling;

    // Remote player: lerp target
    private Vector3 _targetPosition;

    /// <summary>
    /// Call immediately after AddComponent. Sets initial position and wires
    /// up the MovePlayer error callback for local players.
    /// </summary>
    public void Init(Player player, bool isLocal)
    {
        IsLocal = isLocal;
        Vector3 spawnPos = new Vector3(player.PositionX, 1f, player.PositionY);
        transform.position = spawnPos;
        _lastSentPosition = spawnPos;
        _targetPosition = spawnPos;

        if (isLocal)
            SpacetimeDBManager.Conn.Reducers.OnMovePlayer += OnMovePlayerResult;
    }

    void OnDestroy()
    {
        if (IsLocal && SpacetimeDBManager.Conn != null)
            SpacetimeDBManager.Conn.Reducers.OnMovePlayer -= OnMovePlayerResult;
    }

    void OnMovePlayerResult(ReducerEventContext ctx, float newX, float newY)
    {
        if (ctx.Event.Status is Status.Failed(var reason))
            Debug.LogWarning($"[PlayerController] MovePlayer failed: {reason}");
    }

    void Update()
    {
        if (IsLocal) UpdateLocal();
        else UpdateRemote();
    }

    void UpdateLocal()
    {
        // 1. Apply WASD input immediately (client-side prediction)
        Vector3 dir = GetCameraRelativeInput();
        if (dir.sqrMagnitude > 0.001f)
        {
            transform.position += dir * _speed * Time.deltaTime;
            EnforceY();
        }

        // 2. Apply reconciliation correction (MoveTowards at fixed speed)
        if (_reconciling)
        {
            transform.position = Vector3.MoveTowards(
                transform.position, _reconcileTarget, _reconcileSpeed * Time.deltaTime);
            EnforceY();
            if (Vector3.Distance(transform.position, _reconcileTarget) < 0.001f)
                _reconciling = false;
        }

        // 3. Throttled reducer send (10 Hz, only if moved)
        _sendTimer += Time.deltaTime;
        if (_sendTimer >= SendInterval)
        {
            _sendTimer = 0f;
            Vector3 pos = transform.position;
            float dx = pos.x - _lastSentPosition.x;
            float dz = pos.z - _lastSentPosition.z;
            if (dx * dx + dz * dz > 0.01f * 0.01f)
            {
                SpacetimeDBManager.Conn.Reducers.MovePlayer(pos.x, pos.z);
                _lastSentPosition = pos;
            }
        }
    }

    void UpdateRemote()
    {
        transform.position = Vector3.Lerp(
            transform.position, _targetPosition, Time.deltaTime * 10f);
        EnforceY();
    }

    /// <summary>
    /// Called by PlayerManager when the server sends a new position for this player.
    /// </summary>
    public void ReceiveServerPosition(Player newPlayer)
    {
        Vector3 serverPos = new Vector3(newPlayer.PositionX, 1f, newPlayer.PositionY);

        if (!IsLocal)
        {
            _targetPosition = serverPos;
            return;
        }

        float dist = Vector3.Distance(transform.position, serverPos);
        if (dist > 1.0f)
        {
            // Start or restart reconciliation
            _reconcileTarget = serverPos;
            _reconcileSpeed = dist / 0.2f; // covers gap in 0.2 s
            _reconciling = true;
        }
        else
        {
            // Cancel any in-progress reconciliation — server agrees closely enough
            _reconciling = false;
        }
    }

    // Force Y to 1.0f after every position write (camera is parented here)
    void EnforceY() =>
        transform.position = new Vector3(transform.position.x, 1f, transform.position.z);

    static Vector3 GetCameraRelativeInput()
    {
        float h = Input.GetAxisRaw("Horizontal");
        float v = Input.GetAxisRaw("Vertical");
        if (h == 0f && v == 0f) return Vector3.zero;

        Camera cam = Camera.main;
        if (cam == null) return Vector3.zero;

        Vector3 forward = cam.transform.forward;
        forward.y = 0f;
        if (forward.sqrMagnitude < 0.001f) return Vector3.zero;
        forward.Normalize();

        Vector3 right = cam.transform.right;
        right.y = 0f;
        right.Normalize();

        return (forward * v + right * h).normalized;
    }
}
```

- [ ] **Step 2: Verify Unity compiles**

Switch to Unity and check Console for red errors. Common issues:
- `Status.Failed` pattern match syntax — if this doesn't compile in Unity's C# version, replace with: `if (ctx.Event.Status is Status.Failed failedStatus) Debug.LogWarning(failedStatus.Message);`
- Missing `using SpacetimeDB;` — already included above.

- [ ] **Step 3: Commit**

```bash
git add client/Assets/Scripts/Player/PlayerController.cs
git commit -m "feat(client): add PlayerController with prediction and reconciliation"
```

---

## Task 6: Client — Create `PlayerManager`

**Files:**
- Create: `client/Assets/Scripts/Player/PlayerManager.cs`

`PlayerManager` is the scene singleton that owns all player GameObjects. It subscribes to `SpacetimeDBManager` events in `Awake`, backfills existing rows in `OnConnected`, and calls `create_player` if the local player has no row yet.

- [ ] **Step 1: Create `PlayerManager.cs`**

Create `client/Assets/Scripts/Player/PlayerManager.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;
using SpacetimeDB.Types;

/// <summary>
/// Scene singleton. Subscribes to SpacetimeDBManager player events in Awake
/// (before SpacetimeDBManager.Start) and maintains a capsule GameObject per
/// player row. Backfills existing rows in OnConnected because the SDK fires
/// initial OnInsert callbacks before SpacetimeDBManager registers them.
/// </summary>
public class PlayerManager : MonoBehaviour
{
    public static PlayerManager Instance { get; private set; }

    private readonly Dictionary<ulong, GameObject> _players = new();

    void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);

        SpacetimeDBManager.OnPlayerInserted += OnPlayerInserted;
        SpacetimeDBManager.OnPlayerUpdated  += OnPlayerUpdated;
        SpacetimeDBManager.OnPlayerDeleted  += OnPlayerDeleted;
        SpacetimeDBManager.OnConnected      += OnConnected;
    }

    void OnDestroy()
    {
        SpacetimeDBManager.OnPlayerInserted -= OnPlayerInserted;
        SpacetimeDBManager.OnPlayerUpdated  -= OnPlayerUpdated;
        SpacetimeDBManager.OnPlayerDeleted  -= OnPlayerDeleted;
        SpacetimeDBManager.OnConnected      -= OnConnected;
    }

    void OnConnected()
    {
        // Backfill: initial rows arrive before SpacetimeDBManager registers
        // Conn.Db.Player.OnInsert, so OnPlayerInserted never fires for them.
        foreach (var player in SpacetimeDBManager.Conn.Db.Player.Iter())
        {
            if (!_players.ContainsKey(player.Id))
                SpawnPlayer(player);
        }

        // Call create_player only if no row exists for our identity
        bool hasLocalPlayer = false;
        foreach (var player in SpacetimeDBManager.Conn.Db.Player.Iter())
        {
            if (player.Identity == SpacetimeDBManager.LocalIdentity)
            {
                hasLocalPlayer = true;
                break;
            }
        }
        if (!hasLocalPlayer)
            SpacetimeDBManager.Conn.Reducers.CreatePlayer("Player");
    }

    void OnPlayerInserted(Player player)
    {
        if (_players.ContainsKey(player.Id)) return; // guard against backfill duplicate
        SpawnPlayer(player);
    }

    void OnPlayerUpdated(Player oldPlayer, Player newPlayer)
    {
        if (!_players.TryGetValue(newPlayer.Id, out var go)) return;
        go.GetComponent<PlayerController>().ReceiveServerPosition(newPlayer);
    }

    void OnPlayerDeleted(Player player)
    {
        if (!_players.TryGetValue(player.Id, out var go)) return;
        Destroy(go);
        _players.Remove(player.Id);
    }

    void SpawnPlayer(Player player)
    {
        bool isLocal = player.Identity == SpacetimeDBManager.LocalIdentity;

        var go = GameObject.CreatePrimitive(PrimitiveType.Capsule);
        go.name = isLocal ? "LocalPlayer" : $"RemotePlayer_{player.Id}";

        var ctrl = go.AddComponent<PlayerController>();
        ctrl.Init(player, isLocal);

        if (isLocal)
            AttachCamera(go);

        _players[player.Id] = go;
        Debug.Log($"[PlayerManager] Spawned {go.name} at ({player.PositionX}, {player.PositionY})");
    }

    static void AttachCamera(GameObject playerGo)
    {
        Camera cam = Camera.main;
        if (cam == null)
        {
            Debug.LogWarning("[PlayerManager] No main camera found — skipping camera attachment");
            return;
        }
        cam.transform.SetParent(playerGo.transform);
        cam.transform.localPosition = new Vector3(0f, 10f, -7f);
        cam.transform.localRotation = Quaternion.Euler(55f, 0f, 0f);
    }
}
```

- [ ] **Step 2: Verify Unity compiles**

Check Console for red errors.

- [ ] **Step 3: Add `PlayerManager` to the scene**

In Unity:
1. Create an empty GameObject (`GameObject → Create Empty`), name it `PlayerManager`.
2. Drag the `PlayerManager.cs` script onto it in the Inspector.
3. Save the scene (`Ctrl+S`).

- [ ] **Step 4: Commit**

```bash
git add client/Assets/Scripts/Player/PlayerManager.cs
git add client/Assets/Scenes/
git commit -m "feat(client): add PlayerManager — spawns player capsules from server rows"
```

---

## Task 7: Integration test

**No new files.** This task verifies the full flow end-to-end in Play Mode.

Before starting: make sure `spacetime start` is running and the server is deployed (Task 2).

- [ ] **Step 1: Enter Play Mode**

Press `Ctrl+P` in Unity. Watch the Console.

Expected sequence:
1. `[SpacetimeDBManager] Connecting...`
2. `[SpacetimeDBManager] Connected. Identity: <hex>`
3. `[SpacetimeDBManager] Subscription applied`
4. `[PlayerManager] Spawned LocalPlayer at (0, 0)` — OR the player's last known position if they reconnected

If you see `[PlayerManager] Spawned LocalPlayer` — player creation is working.

- [ ] **Step 2: Verify WASD moves the capsule**

Press W/A/S/D. The capsule should move in camera-relative directions. The camera follows because it is parented.

If the capsule doesn't move: check that the PlayerController `_speed` field is non-zero in the Inspector.

- [ ] **Step 3: Verify position persists across reconnect**

Exit Play Mode (`Ctrl+P`). Enter Play Mode again. The player should spawn at the position it was at when you exited (not back at origin), because `create_player` is now idempotent and the row persists.

- [ ] **Step 4: Verify server logs show movement**

```bash
spacetime logs zoneforge-server
```

Expected: `Player moved to (X, Y)` entries corresponding to your WASD movement.

- [ ] **Step 5: Verify zone bounds clamping**

Move the player to the edge of the zone. The capsule should stop at the boundary rather than continuing (the server clamps and the reconciliation lerp corrects the predicted overshoot).

- [ ] **Step 6: Commit submodule state**

```bash
# From the umbrella repo root
git add client
git commit -m "chore: advance client submodule for player movement"
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| No player spawns, no `[PlayerManager]` log | `PlayerManager` not in scene | Add the GameObject (Task 6 Step 3) |
| Player spawns at 0,0 every connect | `create_player` not idempotent yet | Rebuild and redeploy server (Tasks 1–2) |
| WASD has no effect | `GetCameraRelativeInput` returning zero | Check `Camera.main` exists in scene; ensure there is a Camera tagged "MainCamera" |
| Camera doesn't follow | `AttachCamera` skipped | Check Console for "No main camera found" warning; ensure Camera is tagged "MainCamera" |
| Two players spawn on connect | `ConnectionTest` still in scene | Remove the `ConnectionTest` GameObject (Task 4 Step 2) |
| `Status.Failed` pattern doesn't compile | C# version too old for `is Status.Failed(var reason)` | Use `if (ctx.Event.Status is Status.Failed s) Debug.LogWarning(s.Message);` |
| Callbacks never fire | `FrameTick` not running | Verify `SpacetimeDBManager.Update` calls `Conn?.FrameTick()` |
