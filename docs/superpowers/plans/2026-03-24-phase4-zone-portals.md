# Phase 4 — Zone Portals Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add zone portal travel to ZoneForge — players walk through portal trigger volumes to move between zones, with a Ripple Warp transition effect and predictive pre-loading.

**Architecture:** The server holds a `Portal` table and `enter_zone` reducer. The client pre-loads destination zone data when a player approaches a portal (additive `SubscriptionBuilder().Subscribe()` calls), renders a Ripple Warp animation during transfer, and switches `CurrentZoneId` after both the animation and server confirmation complete. The editor gains an icon toolbar, a collapsible World Graph panel for managing portals visually, and a PortalRenderer for ring indicators in the 3D viewport.

**Tech Stack:** SpacetimeDB 2.x (Rust), Unity 2022.3 LTS C#, SpacetimeDB C# SDK, Unity UI Toolkit (UIElements)

---

## File Structure

### New Files
| File | Responsibility |
| ---- | -------------- |
| `client/Assets/Scripts/Zone/PortalManager.cs` | Subscribe to portal events; spawn ring GOs + trigger colliders; proximity pre-load |
| `client/Assets/Scripts/Zone/ZoneTransferManager.cs` | 7-step portal transfer sequence (reducer call → animation → gate flip → cleanup → reconnect) |
| `client/Assets/Scripts/Zone/RippleWarpEffect.cs` | Screen-space concentric cyan ring animation, Play(Action onComplete) |
| `editor/Assets/Scripts/Runtime/ToolbarController.cs` | Icon buttons (UIDocument) — toggle Zone Manager, World Graph, Brush panels |
| `editor/Assets/UI/ToolbarController.uxml` | Icon button strip layout |
| `editor/Assets/UI/ToolbarController.uss` | Button styles (active = blue tint) |
| `editor/Assets/Scripts/Runtime/PortalRenderer.cs` | Spawn editor portal ring GOs; raycast click → select in WorldGraphPanel |
| `editor/Assets/Scripts/Runtime/WorldGraphPanel.cs` | Collapsed bottom panel — zone node canvas + portal detail panel (UIDocument) |
| `editor/Assets/UI/WorldGraphPanel.uxml` | Node canvas + detail panel layout |
| `editor/Assets/UI/WorldGraphPanel.uss` | Panel styles |

### Modified Files
| File | Change |
| ---- | ------ |
| `server/spacetimedb/src/lib.rs` | Add `Portal` table + `create_portal`, `delete_portal`, `enter_zone` reducers |
| `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs` | `CurrentZoneId`; zone-filtered subscriptions; portal events; `SetCurrentZoneId()`; `ReconnectForNewZone()` |
| `client/Assets/Scripts/Player/PlayerManager.cs` | Zone-aware capsule spawn/destroy in `OnPlayerUpdated` |
| `editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs` | Portal subscription + portal events |
| `editor/Assets/Scripts/Runtime/ZoneCreationPanel.cs` | Add `SetVisible(bool)` for ToolbarController |
| `editor/Assets/Scripts/Runtime/TilePalettePanel.cs` | Add `SetVisible(bool)` for ToolbarController |

---

## Task 1: Server — Portal Table

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

Add the `Portal` table directly after the `Enemy` table definition (around line 233).

- [ ] **Step 1: Add Portal table**

Insert after the `Enemy` struct, before the `EnemyRespawnTick` definition:

```rust
// One row per zone portal connection.
#[table(accessor = portal, public)]
pub struct Portal {
    #[primary_key]
    #[auto_inc]
    pub id:             u64,
    #[index(btree)]
    pub source_zone_id: u64,
    #[index(btree)]
    pub dest_zone_id:   u64,
    pub source_x:       f32,  // portal mouth position in source zone
    pub source_y:       f32,
    pub dest_spawn_x:   f32,  // player arrival + reverse exit point in dest zone
    pub dest_spawn_y:   f32,
    pub bidirectional:  bool,
    pub label:          String,  // e.g. "To Village" — optional display name
}
```

- [ ] **Step 2: Add `create_portal` reducer**

Add near the bottom of the file, after the last existing reducer:

```rust
/// Admin-only. Creates a portal between two zones.
/// Validates both zones exist and positions are within their bounds.
#[reducer]
pub fn create_portal(
    ctx: &ReducerContext,
    source_zone_id: u64,
    dest_zone_id: u64,
    source_x: f32,
    source_y: f32,
    dest_spawn_x: f32,
    dest_spawn_y: f32,
    bidirectional: bool,
    label: String,
) -> Result<(), String> {
    if !is_admin(ctx) {
        return Err("Not authorized: admin only".to_string());
    }
    let source_zone = ctx.db.zone().id().find(&source_zone_id)
        .ok_or_else(|| format!("Zone {} not found", source_zone_id))?;
    let dest_zone = ctx.db.zone().id().find(&dest_zone_id)
        .ok_or_else(|| format!("Zone {} not found", dest_zone_id))?;
    if source_x < 0.0 || source_x > source_zone.terrain_width as f32
        || source_y < 0.0 || source_y > source_zone.terrain_height as f32 {
        return Err(format!("source_x/y out of bounds for zone {}", source_zone_id));
    }
    if dest_spawn_x < 0.0 || dest_spawn_x > dest_zone.terrain_width as f32
        || dest_spawn_y < 0.0 || dest_spawn_y > dest_zone.terrain_height as f32 {
        return Err(format!("dest_spawn_x/y out of bounds for zone {}", dest_zone_id));
    }
    ctx.db.portal().insert(Portal {
        id: 0,
        source_zone_id,
        dest_zone_id,
        source_x,
        source_y,
        dest_spawn_x,
        dest_spawn_y,
        bidirectional,
        label,
    });
    log::info!("create_portal: {} -> {} at ({},{})", source_zone_id, dest_zone_id, source_x, source_y);
    Ok(())
}
```

- [ ] **Step 3: Add `delete_portal` reducer**

```rust
/// Admin-only. Deletes a portal by ID.
#[reducer]
pub fn delete_portal(ctx: &ReducerContext, portal_id: u64) -> Result<(), String> {
    if !is_admin(ctx) {
        return Err("Not authorized: admin only".to_string());
    }
    ctx.db.portal().id().find(&portal_id)
        .ok_or_else(|| format!("Portal {} not found", portal_id))?;
    ctx.db.portal().id().delete(&portal_id);
    log::info!("delete_portal: {}", portal_id);
    Ok(())
}
```

- [ ] **Step 4: Add `enter_zone` reducer**

`dist_sq` already exists in the file — do not re-declare it.

```rust
/// Any player. Moves the player through a portal they are standing near.
/// Checks portals originating from the player's current zone (forward travel).
/// For bidirectional portals, also checks reverse travel from dest_spawn position.
/// Returns Err if no portal is within 2 world units of the player's position.
#[reducer]
pub fn enter_zone(ctx: &ReducerContext) -> Result<(), String> {
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or("Player not found")?;
    if player.is_dead {
        return Err("Dead players cannot use portals".to_string());
    }

    // Forward: portals where this zone is the source
    for portal in ctx.db.portal().source_zone_id().filter(&player.zone_id) {
        if dist_sq(player.position_x, player.position_y, portal.source_x, portal.source_y) < 4.0 {
            ctx.db.player().identity().update(Player {
                zone_id:    portal.dest_zone_id,
                position_x: portal.dest_spawn_x,
                position_y: portal.dest_spawn_y,
                ..player
            });
            log::info!("enter_zone: {:?} -> zone {}", ctx.sender(), portal.dest_zone_id);
            return Ok(());
        }
    }

    // Reverse: bidirectional portals where this zone is the dest
    for portal in ctx.db.portal().dest_zone_id().filter(&player.zone_id) {
        if !portal.bidirectional { continue; }
        if dist_sq(player.position_x, player.position_y, portal.dest_spawn_x, portal.dest_spawn_y) < 4.0 {
            ctx.db.player().identity().update(Player {
                zone_id:    portal.source_zone_id,
                position_x: portal.source_x,
                position_y: portal.source_y,
                ..player
            });
            log::info!("enter_zone: {:?} (reverse) -> zone {}", ctx.sender(), portal.source_zone_id);
            return Ok(());
        }
    }

    Err("No portal within range".to_string())
}
```

- [ ] **Step 5: Commit**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add Portal table and create_portal, delete_portal, enter_zone reducers"
```

---

## Task 2: Deploy and Regenerate Autogen

**Files:**
- `client/Assets/Scripts/autogen/` (regenerated)
- `editor/Assets/Scripts/autogen/` (regenerated)

Adding a new table is additive — no `--delete-data` required.

- [ ] **Step 1: Build and publish**

```bash
cd server && spacetime build
spacetime publish --server local zoneforge-server
```

Expected: build prints `wasm-opt` warning (harmless). Publish prints UNSTABLE warning (harmless). Check `spacetime logs zoneforge-server` if publish fails.

- [ ] **Step 2: Regenerate client bindings**

```bash
cd ../client && spacetime generate \
  --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

Expected: new files appear in `client/Assets/Scripts/autogen/` for `Portal`, `CreatePortal`, `DeletePortal`, `EnterZone`.

- [ ] **Step 3: Regenerate editor bindings**

```bash
cd ../editor && spacetime generate \
  --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

- [ ] **Step 4: Verify in Unity**

Open Unity for both `client/` and `editor/`. Run **Assets → Reimport All** in each. The Console should show no compile errors. `Conn.Db.Portal` should now exist in autogen.

- [ ] **Step 5: Commit autogen**

```bash
git add client/Assets/Scripts/autogen editor/Assets/Scripts/autogen
git commit -m "chore: regenerate autogen after Portal table addition"
```

---

## Task 3: Client SpacetimeDBManager — Zone Filtering, CurrentZoneId, Portal Events

**Files:**
- Modify: `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs`

Key changes:
- `CurrentZoneId` static property (saved/loaded from PlayerPrefs)
- Zone-filtered subscriptions for heavy tables on connect
- Portal insert/delete events
- `SetCurrentZoneId(ulong)` — used by ZoneTransferManager after transfer
- `ReconnectForNewZone()` coroutine — background cleanup 2s after transfer

- [ ] **Step 1: Add CurrentZoneId, portal events, and zone key**

Replace the existing event declarations block with the following (keep all existing events, add new ones):

```csharp
public static ulong CurrentZoneId { get; private set; }

private const string ZonePrefKey = "spacetimedb_zone";
// Store server URI and DB name as statics so ReconnectForNewZone can access them
private static string _serverUri;
private static string _dbName;

// --- NEW EVENTS ---
public static event Action<Portal> OnPortalInserted;
public static event Action<Portal> OnPortalDeleted;
public static event Action<ulong>  OnZoneChanged; // fired with OLD zone id
```

- [ ] **Step 2: Set static fields in Awake and load CurrentZoneId**

In `Awake()`, after the singleton guard, add:

```csharp
_serverUri = serverUri;
_dbName    = databaseName;
CurrentZoneId = (ulong)PlayerPrefs.GetInt(ZonePrefKey, 1);
```

- [ ] **Step 3: Update OnConnect to zone-filtered subscriptions**

Replace the `Subscribe` call in `OnConnect` with zone-filtered heavy tables and unfiltered light tables:

```csharp
void OnConnect(DbConnection conn, Identity identity, string token)
{
    PlayerPrefs.SetString(TokenPrefKey, token);
    PlayerPrefs.Save();
    Debug.Log($"[SpacetimeDBManager] Connected. Identity: {identity}. Zone: {CurrentZoneId}");
    LocalIdentity = identity;
    conn.SubscriptionBuilder()
        .OnApplied(OnSubscriptionApplied)
        .Subscribe(new[]
        {
            // Light tables — unfiltered
            "SELECT * FROM player",
            "SELECT * FROM zone",
            "SELECT * FROM ability",
            "SELECT * FROM player_cooldown",
            "SELECT * FROM status_effect",
            "SELECT * FROM combat_log",
            "SELECT * FROM enemy_def",
            // Heavy tables — filtered to current zone
            $"SELECT * FROM terrain_chunk WHERE zone_id = {CurrentZoneId}",
            $"SELECT * FROM entity_instance WHERE zone_id = {CurrentZoneId}",
            $"SELECT * FROM enemy WHERE zone_id = {CurrentZoneId}",
            $"SELECT * FROM portal WHERE source_zone_id = {CurrentZoneId}",
            $"SELECT * FROM portal WHERE dest_zone_id = {CurrentZoneId}",
        });
}
```

- [ ] **Step 4: Wire portal callbacks in OnSubscriptionApplied**

Add at the end of `OnSubscriptionApplied`, before `IsSubscribed = true`:

```csharp
Conn.Db.Portal.OnInsert += (eventCtx, portal) => OnPortalInserted?.Invoke(portal);
Conn.Db.Portal.OnDelete += (eventCtx, portal) => OnPortalDeleted?.Invoke(portal);
```

- [ ] **Step 5: Add SetCurrentZoneId and ReconnectForNewZone**

Add these static methods to the class:

```csharp
/// Called by ZoneTransferManager after transfer animation + server confirmation.
/// Flips the rendering gate and fires OnZoneChanged so managers can purge old GOs.
public static void SetCurrentZoneId(ulong newZoneId)
{
    ulong oldZoneId = CurrentZoneId;
    CurrentZoneId = newZoneId;
    PlayerPrefs.SetInt(ZonePrefKey, (int)newZoneId);
    PlayerPrefs.Save();
    Debug.Log($"[SpacetimeDBManager] Zone changed: {oldZoneId} -> {newZoneId}");
    OnZoneChanged?.Invoke(oldZoneId);
}

/// Call from ZoneTransferManager via StartCoroutine. Rebuilds the connection
/// with zone-filtered queries for the current zone only, flushing old zone data.
public static IEnumerator ReconnectForNewZone()
{
    yield return new WaitForSeconds(2f);
    Debug.Log($"[SpacetimeDBManager] Background reconnect for zone {CurrentZoneId}");
    IsSubscribed = false;
    string savedToken = PlayerPrefs.GetString(TokenPrefKey, null);
    Conn = DbConnection.Builder()
        .WithUri(_serverUri)
        .WithDatabaseName(_dbName)
        .WithToken(savedToken)
        .OnConnect(Instance.OnConnect)
        .OnConnectError(Instance.OnConnectError)
        .OnDisconnect(Instance.OnDisconnect)
        .Build();
    // New OnConnect fires → subscribes with updated CurrentZoneId
}
```

- [ ] **Step 6: Manual verification**

Enter Play mode in Unity. Check Console:
- `[SpacetimeDBManager] Connected. Identity: ... Zone: 1` (or last saved zone)
- `[SpacetimeDBManager] Subscription applied`
- No compile errors

- [ ] **Step 7: Commit**

```bash
git add client/Assets/Scripts/Runtime/SpacetimeDBManager.cs
git commit -m "feat(client): zone-filtered subscriptions, CurrentZoneId, portal events, ReconnectForNewZone"
```

---

## Task 4: Client PlayerManager — Zone Awareness

**Files:**
- Modify: `client/Assets/Scripts/Player/PlayerManager.cs`

When a remote player's `zone_id` changes away from `CurrentZoneId`, destroy their capsule. When it changes to match `CurrentZoneId`, spawn their capsule.

- [ ] **Step 1: Subscribe to OnZoneChanged**

In `Awake()`, add:

```csharp
SpacetimeDBManager.OnZoneChanged += OnZoneChanged;
```

In `OnDestroy()`, add:

```csharp
SpacetimeDBManager.OnZoneChanged -= OnZoneChanged;
```

- [ ] **Step 2: Add OnZoneChanged handler**

Add this method to the class:

```csharp
void OnZoneChanged(ulong oldZoneId)
{
    // Destroy remote player GOs that belong to the old zone
    var toRemove = new List<ulong>();
    foreach (var kvp in _players)
    {
        var player = SpacetimeDBManager.Conn.Db.Player.Id.Find(kvp.Key);
        if (player == null) continue;
        if (player.Identity == SpacetimeDBManager.LocalIdentity) continue; // never destroy local
        if (player.ZoneId != SpacetimeDBManager.CurrentZoneId)
        {
            Destroy(kvp.Value);
            toRemove.Add(kvp.Key);
        }
    }
    foreach (var id in toRemove) _players.Remove(id);

    // Backfill: spawn remote players now in the new zone
    foreach (var player in SpacetimeDBManager.Conn.Db.Player.Iter())
    {
        if (player.Identity == SpacetimeDBManager.LocalIdentity) continue;
        if (player.ZoneId != SpacetimeDBManager.CurrentZoneId) continue;
        if (!_players.ContainsKey(player.Id))
            SpawnPlayer(player);
    }
}
```

- [ ] **Step 3: Zone-gate OnPlayerInserted**

In `OnPlayerInserted`, add a zone check at the top:

```csharp
void OnPlayerInserted(Player player)
{
    if (_players.ContainsKey(player.Id)) return;
    if (player.Identity != SpacetimeDBManager.LocalIdentity
        && player.ZoneId != SpacetimeDBManager.CurrentZoneId) return;
    SpawnPlayer(player);
}
```

- [ ] **Step 4: Handle remote player zone changes in OnPlayerUpdated**

In `OnPlayerUpdated`, after the existing `ReceiveServerPosition` call, add zone departure/arrival for remote players:

```csharp
void OnPlayerUpdated(Player oldPlayer, Player newPlayer)
{
    bool isLocal = newPlayer.Identity == SpacetimeDBManager.LocalIdentity;

    // Remote player moved to a different zone — destroy their capsule
    if (!isLocal && newPlayer.ZoneId != SpacetimeDBManager.CurrentZoneId)
    {
        if (_players.TryGetValue(newPlayer.Id, out var go))
        {
            Destroy(go);
            _players.Remove(newPlayer.Id);
        }
        return;
    }

    // Remote player arrived in this zone — spawn their capsule
    if (!isLocal && newPlayer.ZoneId == SpacetimeDBManager.CurrentZoneId
        && !_players.ContainsKey(newPlayer.Id))
    {
        SpawnPlayer(newPlayer);
        return;
    }

    // Normal update (position, health, etc.)
    if (!_players.TryGetValue(newPlayer.Id, out var playerGo)) return;
    var ctrl = playerGo.GetComponent<PlayerController>();
    if (ctrl == null) return;
    ctrl.ReceiveServerPosition(newPlayer);
    CombatManager.Instance?.RegisterPlayerPosition(
        newPlayer.Id,
        new Vector3(newPlayer.PositionX, 1f, newPlayer.PositionY));
}
```

- [ ] **Step 5: Commit**

```bash
git add client/Assets/Scripts/Player/PlayerManager.cs
git commit -m "feat(client): PlayerManager zone-aware capsule spawn/destroy"
```

---

## Task 5: Client RippleWarpEffect

**Files:**
- Create: `client/Assets/Scripts/Zone/RippleWarpEffect.cs`

Screen-space concentric cyan ring animation using a Canvas UI overlay. Three rings expand from screen centre over 0.6s with a fade out.

- [ ] **Step 1: Write RippleWarpEffect**

```csharp
using System;
using System.Collections;
using UnityEngine;
using UnityEngine.UI;

/// Screen-space portal crossing effect. Attach to a GameObject in the scene
/// (or spawn on demand). Call Play(onComplete) to trigger the animation.
public class RippleWarpEffect : MonoBehaviour
{
    public static RippleWarpEffect Instance { get; private set; }

    private Canvas _canvas;

    private const float Duration    = 0.6f;
    private const int   RingCount   = 3;
    private static readonly Color RingColour = new Color(0.4f, 0.8f, 1f, 0.85f);

    void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);

        _canvas = gameObject.AddComponent<Canvas>();
        _canvas.renderMode   = RenderMode.ScreenSpaceOverlay;
        _canvas.sortingOrder = 999;
        gameObject.AddComponent<CanvasScaler>();
        gameObject.AddComponent<GraphicRaycaster>();
    }

    /// <summary>Plays the Ripple Warp animation then calls onComplete.</summary>
    public void Play(Action onComplete) => StartCoroutine(PlayRoutine(onComplete));

    IEnumerator PlayRoutine(Action onComplete)
    {
        var rings = new (Image img, float startDelay)[RingCount];
        for (int i = 0; i < RingCount; i++)
        {
            var go = new GameObject($"Ring_{i}");
            go.transform.SetParent(_canvas.transform, false);
            var img  = go.AddComponent<Image>();
            img.color   = new Color(RingColour.r, RingColour.g, RingColour.b, 0f);
            img.sprite  = Resources.GetBuiltinResource<Sprite>("UI/Skin/Knob.psd");
            var rt       = go.GetComponent<RectTransform>();
            rt.anchorMin = rt.anchorMax = new Vector2(0.5f, 0.5f);
            rt.sizeDelta = Vector2.one * 10f;
            rings[i] = (img, i * (Duration / RingCount));
        }

        float elapsed = 0f;
        float maxSize = Mathf.Max(Screen.width, Screen.height) * 1.5f;

        while (elapsed < Duration)
        {
            elapsed += Time.deltaTime;
            for (int i = 0; i < RingCount; i++)
            {
                float t = Mathf.Clamp01((elapsed - rings[i].startDelay) / (Duration - rings[i].startDelay));
                if (t <= 0f) continue;
                float size  = Mathf.Lerp(10f, maxSize, t);
                float alpha = Mathf.Lerp(RingColour.a, 0f, t);
                var rt  = rings[i].img.GetComponent<RectTransform>();
                rt.sizeDelta = Vector2.one * size;
                rings[i].img.color = new Color(RingColour.r, RingColour.g, RingColour.b, alpha);
            }
            yield return null;
        }

        foreach (var (img, _) in rings)
            Destroy(img.gameObject);

        onComplete?.Invoke();
    }
}
```

- [ ] **Step 2: Manual verify**

Add `RippleWarpEffect` to a GO in SampleScene. In a temporary test script, call `RippleWarpEffect.Instance.Play(() => Debug.Log("Done"))` and enter Play mode. Concentric cyan rings should expand and fade over ~0.6s.

- [ ] **Step 3: Commit**

```bash
git add client/Assets/Scripts/Zone/RippleWarpEffect.cs
git commit -m "feat(client): RippleWarpEffect - concentric cyan ring portal transition"
```

---

## Task 6: Client PortalManager

**Files:**
- Create: `client/Assets/Scripts/Zone/PortalManager.cs`

Scene singleton. Spawns a glowing ring GO per portal in the current zone. Checks proximity each frame. At ~5 units, pre-loads destination zone. At ~1.5 units, hands off to ZoneTransferManager.

- [ ] **Step 1: Write PortalManager**

```csharp
using System.Collections.Generic;
using UnityEngine;
using SpacetimeDB.Types;

/// Scene singleton. Manages portal ring visuals and proximity logic.
/// Add to SampleScene. Requires SpacetimeDBManager and ZoneTransferManager.
public class PortalManager : MonoBehaviour
{
    public static PortalManager Instance { get; private set; }

    private readonly Dictionary<ulong, GameObject> _portals  = new();
    private readonly HashSet<ulong>                _preloaded = new();

    private const float PreloadRadius = 5f;
    private const float TriggerRadius = 1.5f;

    void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);

        SpacetimeDBManager.OnPortalInserted += OnPortalInserted;
        SpacetimeDBManager.OnPortalDeleted  += OnPortalDeleted;
        SpacetimeDBManager.OnConnected      += OnConnected;
        SpacetimeDBManager.OnZoneChanged    += OnZoneChanged;
    }

    void OnDestroy()
    {
        SpacetimeDBManager.OnPortalInserted -= OnPortalInserted;
        SpacetimeDBManager.OnPortalDeleted  -= OnPortalDeleted;
        SpacetimeDBManager.OnConnected      -= OnConnected;
        SpacetimeDBManager.OnZoneChanged    -= OnZoneChanged;
    }

    void OnConnected()
    {
        foreach (var portal in SpacetimeDBManager.Conn.Db.Portal.Iter())
        {
            if (portal.SourceZoneId == SpacetimeDBManager.CurrentZoneId
                || (portal.Bidirectional && portal.DestZoneId == SpacetimeDBManager.CurrentZoneId))
                SpawnPortalGo(portal);
        }
    }

    void OnPortalInserted(Portal portal)
    {
        if (_portals.ContainsKey(portal.Id)) return;
        if (portal.SourceZoneId == SpacetimeDBManager.CurrentZoneId
            || (portal.Bidirectional && portal.DestZoneId == SpacetimeDBManager.CurrentZoneId))
            SpawnPortalGo(portal);
    }

    void OnPortalDeleted(Portal portal)
    {
        if (!_portals.TryGetValue(portal.Id, out var go)) return;
        Destroy(go);
        _portals.Remove(portal.Id);
        _preloaded.Remove(portal.Id);
    }

    void OnZoneChanged(ulong oldZoneId)
    {
        // Destroy all portal GOs — OnConnected backfill spawns the new zone's portals
        foreach (var go in _portals.Values) Destroy(go);
        _portals.Clear();
        _preloaded.Clear();
    }

    void Update()
    {
        if (!SpacetimeDBManager.IsSubscribed) return;
        var localPlayer = GetLocalPlayerRow();
        if (localPlayer == null) return;
        var localPos = new Vector3(localPlayer.PositionX, 0f, localPlayer.PositionY);

        foreach (var kvp in _portals)
        {
            var portal = SpacetimeDBManager.Conn.Db.Portal.Id.Find(kvp.Key);
            if (portal == null) continue;
            bool reverse = portal.DestZoneId == SpacetimeDBManager.CurrentZoneId;
            float px = reverse ? portal.DestSpawnX : portal.SourceX;
            float py = reverse ? portal.DestSpawnY : portal.SourceY;
            var portalPos = new Vector3(px, 0f, py);
            float dist = Vector3.Distance(localPos, portalPos);

            if (dist < PreloadRadius && !_preloaded.Contains(kvp.Key))
            {
                _preloaded.Add(kvp.Key);
                ulong destZoneId = reverse ? portal.SourceZoneId : portal.DestZoneId;
                PreloadZone(destZoneId);
            }

            if (dist < TriggerRadius && ZoneTransferManager.Instance != null
                && !ZoneTransferManager.Instance.IsTransferring)
            {
                ZoneTransferManager.Instance.BeginTransfer(portal);
            }
        }
    }

    void SpawnPortalGo(Portal portal)
    {
        bool reverse = portal.DestZoneId == SpacetimeDBManager.CurrentZoneId && portal.Bidirectional;
        float px = reverse ? portal.DestSpawnX : portal.SourceX;
        float py = reverse ? portal.DestSpawnY : portal.SourceY;

        var go = new GameObject($"Portal_{portal.Id}");
        go.transform.position = new Vector3(px, 0f, py);

        // Invisible trigger cylinder for reference (no MeshRenderer)
        var col = go.AddComponent<CapsuleCollider>();
        col.isTrigger = true;
        col.radius    = TriggerRadius;
        col.height    = 2f;
        col.center    = Vector3.up;

        // Glowing ring — thin cylinder scaled flat
        var ringGo = GameObject.CreatePrimitive(PrimitiveType.Cylinder);
        ringGo.transform.SetParent(go.transform);
        ringGo.transform.localPosition = new Vector3(0f, 0.1f, 0f);
        ringGo.transform.localScale    = new Vector3(2f, 0.03f, 2f);
        Destroy(ringGo.GetComponent<Collider>());
        var rend = ringGo.GetComponent<Renderer>();
        var mat  = new Material(Shader.Find("Universal Render Pipeline/Lit"));
        mat.EnableKeyword("_EMISSION");
        mat.SetColor("_EmissionColor", new Color(0.3f, 0.8f, 1f) * 2f);
        mat.color = new Color(0.3f, 0.8f, 1f, 0.5f);
        rend.material = mat;

        _portals[portal.Id] = go;
    }

    void PreloadZone(ulong destZoneId)
    {
        Debug.Log($"[PortalManager] Pre-loading zone {destZoneId}");
        SpacetimeDBManager.Conn.SubscriptionBuilder()
            .OnApplied(_ => Debug.Log($"[PortalManager] Pre-load complete for zone {destZoneId}"))
            .Subscribe(new[]
            {
                $"SELECT * FROM terrain_chunk WHERE zone_id = {destZoneId}",
                $"SELECT * FROM entity_instance WHERE zone_id = {destZoneId}",
                $"SELECT * FROM enemy WHERE zone_id = {destZoneId}",
                $"SELECT * FROM portal WHERE source_zone_id = {destZoneId}",
                $"SELECT * FROM portal WHERE dest_zone_id = {destZoneId}",
            });
    }

    static Player GetLocalPlayerRow()
    {
        if (!SpacetimeDBManager.IsSubscribed) return null;
        return SpacetimeDBManager.Conn.Db.Player.Identity.Find(SpacetimeDBManager.LocalIdentity);
    }
}
```

- [ ] **Step 2: Add PortalManager to SampleScene**

In Unity: create an empty GameObject named `PortalManager` in SampleScene, add the `PortalManager` component.

- [ ] **Step 3: Manual verify**

Enter Play mode. Create a portal via `spacetime call` or editor. A glowing cyan ring should appear in the viewport. The Console should log `[PortalManager] Pre-loading zone X` as the player approaches within 5 units.

- [ ] **Step 4: Commit**

```bash
git add client/Assets/Scripts/Zone/PortalManager.cs
git commit -m "feat(client): PortalManager - portal ring GOs, proximity pre-load, trigger detection"
```

---

## Task 7: Client ZoneTransferManager

**Files:**
- Create: `client/Assets/Scripts/Zone/ZoneTransferManager.cs`

Coordinates the 7-step transfer sequence. `BeginTransfer()` is called by PortalManager when the player enters the trigger radius.

- [ ] **Step 1: Write ZoneTransferManager**

```csharp
using System.Collections;
using UnityEngine;
using SpacetimeDB.Types;

/// Scene singleton. Executes the zone transfer sequence:
/// 1. Call enter_zone reducer  2. Play Ripple Warp animation
/// 3. Wait for player row zone_id update  4. Flip CurrentZoneId gate
/// 5. Destroy old zone GOs  6. Background reconnect (2s delay)
public class ZoneTransferManager : MonoBehaviour
{
    public static ZoneTransferManager Instance { get; private set; }

    public bool IsTransferring { get; private set; }

    private ulong  _pendingZoneId;
    private bool   _transferConfirmed;
    private bool   _animationComplete;

    void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);
    }

    /// Called by PortalManager when the local player enters a portal trigger radius.
    public void BeginTransfer(Portal portal)
    {
        if (IsTransferring) return;
        var localPlayer = SpacetimeDBManager.Conn.Db.Player.Identity.Find(SpacetimeDBManager.LocalIdentity);
        if (localPlayer == null) return;
        bool reverse = portal.DestZoneId == SpacetimeDBManager.CurrentZoneId;
        _pendingZoneId    = reverse ? portal.SourceZoneId : portal.DestZoneId;
        IsTransferring    = true;
        _transferConfirmed = false;
        _animationComplete = false;

        Debug.Log($"[ZoneTransferManager] Transfer to zone {_pendingZoneId}");

        // Step 1: call enter_zone reducer
        SpacetimeDBManager.Conn.Reducers.EnterZone();

        // Step 2: play Ripple Warp
        if (RippleWarpEffect.Instance != null)
            RippleWarpEffect.Instance.Play(OnAnimationComplete);
        else
            OnAnimationComplete(); // fallback if effect not in scene

        // Step 3: listen for player zone update
        SpacetimeDBManager.OnPlayerUpdated += OnPlayerUpdated;
    }

    void OnPlayerUpdated(Player oldPlayer, Player newPlayer)
    {
        if (newPlayer.Identity != SpacetimeDBManager.LocalIdentity) return;
        if (newPlayer.ZoneId == _pendingZoneId)
        {
            _transferConfirmed = true;
            TryCompleteTransfer();
        }
    }

    void OnAnimationComplete()
    {
        _animationComplete = true;
        TryCompleteTransfer();
    }

    void TryCompleteTransfer()
    {
        if (!_transferConfirmed || !_animationComplete) return;
        SpacetimeDBManager.OnPlayerUpdated -= OnPlayerUpdated;
        CompleteTransfer();
    }

    void CompleteTransfer()
    {
        // Steps 4 & 5: flip rendering gate (managers fire OnZoneChanged to purge old GOs)
        SpacetimeDBManager.SetCurrentZoneId(_pendingZoneId);

        // Step 6: background reconnect
        StartCoroutine(SpacetimeDBManager.ReconnectForNewZone());

        IsTransferring = false;
        Debug.Log($"[ZoneTransferManager] Transfer complete. Now in zone {_pendingZoneId}");
    }
}
```

- [ ] **Step 2: Add ZoneTransferManager to SampleScene**

Create an empty GO named `ZoneTransferManager` in SampleScene, add the component. Also add `RippleWarpEffect` to an empty GO named `RippleWarpEffect`.

- [ ] **Step 3: Manual verify (end-to-end portal test)**

1. Ensure at least 2 zones exist (zone id=1 and id=2).
2. Create a bidirectional portal via the editor or `spacetime call zoneforge-server create_portal '{"source_zone_id":1,"dest_zone_id":2,"source_x":10,"source_y":10,"dest_spawn_x":5,"dest_spawn_y":5,"bidirectional":true,"label":"Test"}'`
3. Enter Play mode. Walk the local player to (10, 10) in zone 1.
4. Expected:
   - Cyan ring appears at portal position
   - At 5 units away: Console logs `Pre-loading zone 2`
   - At 1.5 units: Ripple Warp animation plays
   - Player row `zone_id` updates to 2 on server
   - Console logs `Transfer complete. Now in zone 2`
   - Background reconnect fires after 2s

- [ ] **Step 4: Commit**

```bash
git add client/Assets/Scripts/Zone/ZoneTransferManager.cs
git commit -m "feat(client): ZoneTransferManager - portal transfer sequence with Ripple Warp"
```

---

## Task 8: Editor SpacetimeDBManager — Portal Subscription and Events

**Files:**
- Modify: `editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs`

- [ ] **Step 1: Add portal events**

Add to the event declarations block:

```csharp
public static event Action<Portal> OnPortalInserted;
public static event Action<Portal> OnPortalDeleted;
```

- [ ] **Step 2: Add portal subscription**

In `OnConnect`, add portal queries to the existing `Subscribe` call:

```csharp
conn.SubscriptionBuilder()
    .OnApplied(OnSubscriptionApplied)
    .Subscribe(new[]
    {
        "SELECT * FROM player",
        "SELECT * FROM zone",
        "SELECT * FROM entity_instance",
        "SELECT * FROM terrain_chunk",
        "SELECT * FROM portal",        // ← ADD
    });
```

- [ ] **Step 3: Wire portal callbacks in OnSubscriptionApplied**

Add after the existing terrain chunk callbacks:

```csharp
Conn.Db.Portal.OnInsert += (eventCtx, portal) => OnPortalInserted?.Invoke(portal);
Conn.Db.Portal.OnDelete += (eventCtx, portal) => OnPortalDeleted?.Invoke(portal);
```

- [ ] **Step 4: Commit**

```bash
git add editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs
git commit -m "feat(editor): portal subscription and events in SpacetimeDBManager"
```

---

## Task 9: Editor ToolbarController

**Files:**
- Create: `editor/Assets/Scripts/Runtime/ToolbarController.cs`
- Create: `editor/Assets/UI/ToolbarController.uxml`
- Create: `editor/Assets/UI/ToolbarController.uss`

Icon button toolbar: 🗺 + ⬡ top-left, 🖌 top-right. Active panel icon gets blue highlight. Only one panel open at a time per side.

- [ ] **Step 1: Create ToolbarController.uxml**

Check that `editor/Assets/UI/` exists (it should from existing panels). Create:

```xml
<ui:UXML xmlns:ui="UnityEngine.UIElements">
  <ui:VisualElement name="toolbar-root" class="toolbar-root">

    <!-- Top-left icon strip -->
    <ui:VisualElement name="left-strip" class="icon-strip left-strip">
      <ui:Button name="btn-zone"  class="icon-btn" text="🗺" tooltip="Zone Manager"/>
      <ui:Button name="btn-graph" class="icon-btn" text="⬡" tooltip="World Graph"/>
    </ui:VisualElement>

    <!-- Top-right brush icon -->
    <ui:Button name="btn-brush" class="icon-btn right-btn" text="🖌" tooltip="Brush"/>

  </ui:VisualElement>
</ui:UXML>
```

- [ ] **Step 2: Create ToolbarController.uss**

```css
.toolbar-root {
    position: absolute;
    top: 0; left: 0; right: 0; bottom: 0;
    background-color: rgba(0,0,0,0);
}

.icon-strip {
    position: absolute;
    top: 6px;
    flex-direction: row;
}

.left-strip { left: 6px; }

.icon-btn {
    width: 28px;
    height: 28px;
    background-color: rgba(20,20,35,0.92);
    border-color: rgba(255,255,255,0.12);
    border-width: 1px;
    border-radius: 5px;
    font-size: 14px;
    margin-right: 4px;
    -unity-text-align: middle-center;
    padding: 0;
}

.icon-btn.active {
    background-color: rgba(60,80,160,0.9);
    border-color: rgba(100,140,255,0.6);
}

.right-btn {
    position: absolute;
    top: 6px;
    right: 6px;
    margin-right: 0;
}
```

- [ ] **Step 3: Create ToolbarController.cs**

```csharp
using UnityEngine;
using UnityEngine.UIElements;

/// UIDocument-based icon toolbar. Toggle buttons control ZoneCreationPanel,
/// WorldGraphPanel, and TilePalettePanel. Assign panel references in Inspector.
[RequireComponent(typeof(UIDocument))]
public class ToolbarController : MonoBehaviour
{
    public static ToolbarController Instance { get; private set; }

    [SerializeField] StyleSheet       _styleSheet;
    [SerializeField] ZoneCreationPanel _zonePanel;
    [SerializeField] WorldGraphPanel   _graphPanel;
    [SerializeField] TilePalettePanel  _brushPanel;

    private Button _btnZone;
    private Button _btnGraph;
    private Button _btnBrush;

    private bool _zoneOpen;
    private bool _graphOpen;
    private bool _brushOpen;

    void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;
    }

    void OnEnable()
    {
        var root = GetComponent<UIDocument>().rootVisualElement;
        if (_styleSheet != null) root.styleSheets.Add(_styleSheet);

        _btnZone  = root.Q<Button>("btn-zone");
        _btnGraph = root.Q<Button>("btn-graph");
        _btnBrush = root.Q<Button>("btn-brush");

        _btnZone .clicked += ToggleZone;
        _btnGraph.clicked += ToggleGraph;
        _btnBrush.clicked += ToggleBrush;
    }

    void ToggleZone()
    {
        _zoneOpen = !_zoneOpen;
        if (_zoneOpen) { _graphOpen = false; RefreshGraph(); }
        RefreshZone();
    }

    void ToggleGraph()
    {
        _graphOpen = !_graphOpen;
        if (_graphOpen) { _zoneOpen = false; RefreshZone(); }
        RefreshGraph();
    }

    void ToggleBrush()
    {
        _brushOpen = !_brushOpen;
        RefreshBrush();
    }

    /// Called by WorldGraphPanel or PortalRenderer to open the graph panel.
    public void OpenWorldGraph()
    {
        if (_graphOpen) return;
        _graphOpen = true;
        _zoneOpen  = false;
        RefreshZone();
        RefreshGraph();
    }

    void RefreshZone()
    {
        _zonePanel?.SetVisible(_zoneOpen);
        SetActive(_btnZone, _zoneOpen);
    }

    void RefreshGraph()
    {
        _graphPanel?.SetVisible(_graphOpen);
        SetActive(_btnGraph, _graphOpen);
    }

    void RefreshBrush()
    {
        _brushPanel?.SetVisible(_brushOpen);
        SetActive(_btnBrush, _brushOpen);
    }

    static void SetActive(Button btn, bool active)
    {
        if (btn == null) return;
        if (active) btn.AddToClassList("active");
        else        btn.RemoveFromClassList("active");
    }
}
```

- [ ] **Step 4: Add ToolbarController to Editor Scene**

In the Unity Editor project:
1. Create empty GO named `ToolbarController` in the scene.
2. Add `UIDocument` component. Assign a Panel Settings asset (same one used by other panels).
3. Assign the `ToolbarController.uxml` as the Source Asset.
4. Add the `ToolbarController` script.
5. In Inspector, assign `_styleSheet`, `_zonePanel`, and `_brushPanel`. Leave `_graphPanel` empty for now — assign it after completing Task 11 (WorldGraphPanel).

- [ ] **Step 5: Commit**

```bash
git add editor/Assets/Scripts/Runtime/ToolbarController.cs \
        editor/Assets/UI/ToolbarController.uxml \
        editor/Assets/UI/ToolbarController.uss
git commit -m "feat(editor): ToolbarController - icon button strip for Zone Manager, World Graph, Brush"
```

---

## Task 10: Editor PortalRenderer

**Files:**
- Create: `editor/Assets/Scripts/Runtime/PortalRenderer.cs`

Editor-side ring rendering. Listens to portal events, spawns ring GOs in the 3D viewport for portals in the active zone. Raycasts on click to select a portal in WorldGraphPanel.

- [ ] **Step 1: Write PortalRenderer**

```csharp
using System.Collections.Generic;
using UnityEngine;
using SpacetimeDB.Types;

/// Scene singleton. Renders portal ring indicators in the editor 3D viewport.
/// Click on a ring → selects that portal in the WorldGraphPanel.
public class PortalRenderer : MonoBehaviour
{
    public static PortalRenderer Instance { get; private set; }

    private readonly Dictionary<ulong, GameObject> _rings = new();

    // Whether we're waiting for a viewport click to place a portal's source position
    public bool IsPlacementMode { get; private set; }
    private ulong _placementPortalId;
    private System.Action<float, float> _placementCallback;

    void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;

        SpacetimeDBManager.OnPortalInserted += OnPortalInserted;
        SpacetimeDBManager.OnPortalDeleted  += OnPortalDeleted;
        SpacetimeDBManager.OnConnected      += OnConnected;
        EditorState.OnActiveZoneChanged     += _ => RebuildRings();
    }

    void OnDestroy()
    {
        SpacetimeDBManager.OnPortalInserted -= OnPortalInserted;
        SpacetimeDBManager.OnPortalDeleted  -= OnPortalDeleted;
        SpacetimeDBManager.OnConnected      -= OnConnected;
    }

    void OnConnected() => RebuildRings();

    void OnPortalInserted(Portal portal) => TrySpawnRing(portal);

    void OnPortalDeleted(Portal portal)
    {
        if (!_rings.TryGetValue(portal.Id, out var go)) return;
        Destroy(go);
        _rings.Remove(portal.Id);
    }

    void RebuildRings()
    {
        foreach (var go in _rings.Values) Destroy(go);
        _rings.Clear();
        if (!SpacetimeDBManager.IsSubscribed) return;
        foreach (var portal in SpacetimeDBManager.Conn.Db.Portal.Iter())
            TrySpawnRing(portal);
    }

    void TrySpawnRing(Portal portal)
    {
        if (portal.SourceZoneId != (ulong)EditorState.ActiveZoneId) return;
        if (_rings.ContainsKey(portal.Id)) return;
        SpawnRing(portal, portal.SourceX, portal.SourceY, false);
    }

    void SpawnRing(Portal portal, float x, float y, bool selected)
    {
        var go  = GameObject.CreatePrimitive(PrimitiveType.Cylinder);
        go.name = $"PortalRing_{portal.Id}";
        go.transform.position   = new Vector3(x, 0.05f, y);
        go.transform.localScale = new Vector3(2f, 0.04f, 2f);
        Destroy(go.GetComponent<Collider>());
        // Add trigger for click detection
        var col = go.AddComponent<CapsuleCollider>();
        col.isTrigger = true;
        var rend = go.GetComponent<Renderer>();
        var mat  = new Material(Shader.Find("Universal Render Pipeline/Lit"));
        Color c  = selected ? new Color(1f, 0.85f, 0.3f) : new Color(0.3f, 0.8f, 1f);
        mat.EnableKeyword("_EMISSION");
        mat.SetColor("_EmissionColor", c * 2f);
        mat.color = c;
        rend.material = mat;
        // Store portal id on GO via a simple tag or name (we use name for lookup)
        _rings[portal.Id] = go;
    }

    void Update()
    {
        if (!Input.GetMouseButtonDown(0)) return;
        var ray = Camera.main?.ScreenPointToRay(Input.mousePosition);
        if (ray == null) return;

        if (IsPlacementMode)
        {
            HandlePlacementClick(ray.Value);
            return;
        }

        // Check if a portal ring was clicked
        if (Physics.Raycast(ray.Value, out var hit, 500f))
        {
            foreach (var kvp in _rings)
            {
                if (kvp.Value == hit.collider.gameObject)
                {
                    WorldGraphPanel.Instance?.SelectPortal(kvp.Key);
                    ToolbarController.Instance?.OpenWorldGraph();
                    return;
                }
            }
        }
    }

    /// Enters placement mode. The next viewport click sets the source position.
    public void BeginPlacement(ulong portalId, System.Action<float, float> onPositionSet)
    {
        IsPlacementMode    = true;
        _placementPortalId = portalId;
        _placementCallback = onPositionSet;
    }

    void HandlePlacementClick(Ray ray)
    {
        if (Physics.Raycast(ray, out var hit, 500f))
        {
            float wx = hit.point.x;
            float wy = hit.point.z;
            IsPlacementMode = false;
            _placementCallback?.Invoke(wx, wy);
        }
    }

    /// Updates the visual of a ring to highlight it as selected.
    public void SetRingSelected(ulong portalId, bool selected)
    {
        if (!_rings.TryGetValue(portalId, out var go)) return;
        var rend = go.GetComponent<Renderer>();
        if (rend == null) return;
        Color c = selected ? new Color(1f, 0.85f, 0.3f) : new Color(0.3f, 0.8f, 1f);
        rend.material.SetColor("_EmissionColor", c * 2f);
        rend.material.color = c;
    }
}
```

- [ ] **Step 2: Add PortalRenderer to Editor Scene**

Create empty GO `PortalRenderer`, add the script.

- [ ] **Step 3: Commit**

```bash
git add editor/Assets/Scripts/Runtime/PortalRenderer.cs
git commit -m "feat(editor): PortalRenderer - portal ring GOs, viewport click selection"
```

---

## Task 11: Editor WorldGraphPanel

**Files:**
- Create: `editor/Assets/Scripts/Runtime/WorldGraphPanel.cs`
- Create: `editor/Assets/UI/WorldGraphPanel.uxml`
- Create: `editor/Assets/UI/WorldGraphPanel.uss`

Collapsed bottom panel. Zone nodes positioned absolutely on a canvas. Portal edges drawn via `generateVisualContent`. Detail panel on the right.

- [ ] **Step 1: Create WorldGraphPanel.uxml**

```xml
<ui:UXML xmlns:ui="UnityEngine.UIElements">
  <ui:VisualElement name="wg-root" class="wg-root">

    <!-- Bottom panel (collapsed by default) -->
    <ui:VisualElement name="wg-panel" class="wg-panel">

      <ui:VisualElement name="wg-body" class="wg-body">

        <!-- Node canvas (left, flex 1.7) -->
        <ui:VisualElement name="wg-canvas" class="wg-canvas">
          <ui:Label name="canvas-header" class="section-label" text="ZONES"/>
          <!-- Zone nodes and edges are added at runtime -->
        </ui:VisualElement>

        <!-- Detail panel (right, 148px) -->
        <ui:VisualElement name="wg-detail" class="wg-detail">
          <ui:Label name="detail-header" class="section-label" text="PORTAL DETAIL"/>
          <ui:Label name="detail-title"  class="detail-title"  text="—"/>
          <ui:VisualElement class="detail-row">
            <ui:Label class="detail-label" text="Spawn"/>
            <ui:Label name="detail-spawn" class="detail-value" text="—"/>
          </ui:VisualElement>
          <ui:VisualElement class="detail-row">
            <ui:Label class="detail-label" text="Direction"/>
            <ui:Label name="detail-dir"   class="detail-value" text="—"/>
          </ui:VisualElement>
          <ui:VisualElement name="detail-buttons" class="detail-buttons">
            <ui:Button name="btn-delete" class="action-btn danger-btn" text="Delete"/>
            <ui:Button name="btn-place"  class="action-btn place-btn"  text="Place"/>
          </ui:VisualElement>
        </ui:VisualElement>

      </ui:VisualElement>
    </ui:VisualElement>

  </ui:VisualElement>
</ui:UXML>
```

- [ ] **Step 2: Create WorldGraphPanel.uss**

```css
.wg-root {
    position: absolute;
    top: 0; left: 0; right: 0; bottom: 0;
    background-color: rgba(0,0,0,0);
    justify-content: flex-end;
}

.wg-panel {
    background-color: rgb(18,18,30);
    border-top-width: 2px;
    border-top-color: rgba(100,140,255,0.25);
}

.wg-body {
    flex-direction: row;
    height: 90px;
}

.wg-canvas {
    flex: 1.7;
    padding: 8px;
    border-right-width: 1px;
    border-right-color: rgba(255,255,255,0.07);
    position: relative;
    overflow: hidden;
}

.wg-detail {
    width: 148px;
    padding: 8px 10px;
}

.section-label {
    font-size: 9px;
    color: rgba(255,255,255,0.35);
    margin-bottom: 4px;
    -unity-font-style: bold;
}

.detail-title {
    font-size: 11px;
    color: rgba(255,220,80,0.9);
    margin-bottom: 5px;
}

.detail-row {
    flex-direction: row;
    justify-content: space-between;
    margin-bottom: 3px;
}

.detail-label { font-size: 10px; color: rgba(255,255,255,0.5); }
.detail-value { font-size: 10px; }

.detail-buttons {
    flex-direction: row;
    margin-top: 4px;
}

.action-btn {
    flex: 1;
    font-size: 9px;
    padding: 2px 0;
    border-radius: 3px;
    margin-right: 4px;
    -unity-text-align: middle-center;
}

.danger-btn {
    background-color: rgba(255,80,80,0.15);
    border-color: rgba(255,80,80,0.3);
    color: rgba(255,120,120,0.9);
}

.place-btn {
    background-color: rgba(80,140,255,0.15);
    border-color: rgba(80,140,255,0.3);
    color: rgba(120,170,255,0.9);
    margin-right: 0;
}

/* Zone node pills — positioned absolutely via script */
.zone-node {
    position: absolute;
    padding: 3px 8px;
    border-radius: 4px;
    font-size: 10px;
    border-width: 1px;
    cursor: default;
}
```

- [ ] **Step 3: Create WorldGraphPanel.cs**

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UIElements;
using SpacetimeDB.Types;

[RequireComponent(typeof(UIDocument))]
public class WorldGraphPanel : MonoBehaviour
{
    public static WorldGraphPanel Instance { get; private set; }

    [SerializeField] StyleSheet _styleSheet;

    private VisualElement _panel;
    private VisualElement _canvas;
    private Label         _detailTitle;
    private Label         _detailSpawn;
    private Label         _detailDir;
    private Button        _btnDelete;
    private Button        _btnPlace;

    private ulong? _selectedPortalId;

    // Zone node elements keyed by zone id
    private readonly Dictionary<ulong, VisualElement> _zoneNodes = new();

    // Edge drawing element (overlays canvas)
    private EdgeDrawer _edgeDrawer;

    void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;
    }

    // Named handlers for event subscription — lambdas can't be unsubscribed reliably.
    void OnPortalInsertedRebuild(Portal _) => Rebuild();
    void OnPortalDeletedRebuild(Portal _)  => Rebuild();
    void OnZoneInsertedRebuild(Zone _)     => Rebuild();

    void OnEnable()
    {
        var root = GetComponent<UIDocument>().rootVisualElement;
        if (_styleSheet != null) root.styleSheets.Add(_styleSheet);

        _panel       = root.Q<VisualElement>("wg-panel");
        _canvas      = root.Q<VisualElement>("wg-canvas");
        _detailTitle = root.Q<Label>("detail-title");
        _detailSpawn = root.Q<Label>("detail-spawn");
        _detailDir   = root.Q<Label>("detail-dir");
        _btnDelete   = root.Q<Button>("btn-delete");
        _btnPlace    = root.Q<Button>("btn-place");

        _btnDelete.clicked += OnDeleteClicked;
        _btnPlace.clicked  += OnPlaceClicked;

        // Hidden by default — ToolbarController calls SetVisible
        _panel.style.display = DisplayStyle.None;

        // Edge drawer sits above zone nodes in the canvas
        _edgeDrawer = new EdgeDrawer();
        _canvas.Add(_edgeDrawer);
        _edgeDrawer.StretchToParentSize();

        SpacetimeDBManager.OnConnected      += Rebuild;
        SpacetimeDBManager.OnPortalInserted += OnPortalInsertedRebuild;
        SpacetimeDBManager.OnPortalDeleted  += OnPortalDeletedRebuild;
        SpacetimeDBManager.OnZoneInserted   += OnZoneInsertedRebuild;
    }

    void OnDisable()
    {
        SpacetimeDBManager.OnConnected      -= Rebuild;
        SpacetimeDBManager.OnPortalInserted -= OnPortalInsertedRebuild;
        SpacetimeDBManager.OnPortalDeleted  -= OnPortalDeletedRebuild;
        SpacetimeDBManager.OnZoneInserted   -= OnZoneInsertedRebuild;
    }

    public void SetVisible(bool visible)
    {
        _panel.style.display = visible ? DisplayStyle.Flex : DisplayStyle.None;
        if (visible) Rebuild();
    }

    public void SelectPortal(ulong portalId)
    {
        _selectedPortalId = portalId;
        RefreshDetail();
        _edgeDrawer.MarkDirtyRepaint();
        PortalRenderer.Instance?.SetRingSelected(portalId, true);
    }

    void Rebuild()
    {
        if (_panel.style.display == DisplayStyle.None) return;
        BuildZoneNodes();
        _edgeDrawer.MarkDirtyRepaint();
        RefreshDetail();
    }

    void BuildZoneNodes()
    {
        // Remove old nodes (not the edge drawer)
        var toRemove = new List<VisualElement>();
        foreach (var c in _canvas.Children())
            if (c != _edgeDrawer) toRemove.Add(c);
        foreach (var c in toRemove) _canvas.Remove(c);
        _zoneNodes.Clear();

        if (!SpacetimeDBManager.IsSubscribed) return;
        var zones = new List<Zone>();
        foreach (var z in SpacetimeDBManager.Conn.Db.Zone.Iter()) zones.Add(z);

        // Simple circular auto-layout
        int n = zones.Count;
        float cx = 140f, cy = 35f, rx = 100f, ry = 20f;
        for (int i = 0; i < n; i++)
        {
            float angle = (float)i / n * Mathf.PI * 2f;
            float x = cx + rx * Mathf.Cos(angle) - 30f;
            float y = cy + ry * Mathf.Sin(angle) - 10f;
            var node = new Label(zones[i].Name);
            node.AddToClassList("zone-node");
            node.style.left = x;
            node.style.top  = y;
            node.style.backgroundColor = GetZoneColor(i, 0.2f);
            node.style.borderLeftColor = node.style.borderRightColor =
                node.style.borderTopColor = node.style.borderBottomColor = GetZoneColor(i, 0.6f);
            var zoneId = zones[i].Id;
            node.RegisterCallback<ClickEvent>(_ => OnZoneNodeClicked(zoneId));
            _canvas.Add(node);
            _zoneNodes[zones[i].Id] = node;
        }

        // Give layout a frame to resolve positions for edge drawing
        _canvas.schedule.Execute(() => _edgeDrawer.MarkDirtyRepaint()).ExecuteLater(1);
    }

    void OnZoneNodeClicked(ulong zoneId)
    {
        // Future: create new portal connection from this zone
    }

    void RefreshDetail()
    {
        if (_selectedPortalId == null || !SpacetimeDBManager.IsSubscribed)
        {
            _detailTitle.text = "—";
            _detailSpawn.text = "—";
            _detailDir.text   = "—";
            return;
        }
        var portal = SpacetimeDBManager.Conn.Db.Portal.Id.Find(_selectedPortalId.Value);
        if (portal == null) { _selectedPortalId = null; RefreshDetail(); return; }
        var src = SpacetimeDBManager.Conn.Db.Zone.Id.Find(portal.SourceZoneId);
        var dst = SpacetimeDBManager.Conn.Db.Zone.Id.Find(portal.DestZoneId);
        _detailTitle.text = $"{src?.Name ?? "?"} → {dst?.Name ?? "?"}";
        _detailSpawn.text = $"X:{portal.DestSpawnX:F0} Y:{portal.DestSpawnY:F0}";
        _detailDir.text   = portal.Bidirectional ? "↔ Both" : "→ One-way";
    }

    void OnDeleteClicked()
    {
        if (_selectedPortalId == null) return;
        SpacetimeDBManager.Conn.Reducers.DeletePortal(_selectedPortalId.Value);
        _selectedPortalId = null;
    }

    void OnPlaceClicked()
    {
        if (_selectedPortalId == null) return;
        PortalRenderer.Instance?.BeginPlacement(_selectedPortalId.Value, OnPlacementSet);
    }

    void OnPlacementSet(float wx, float wy)
    {
        if (_selectedPortalId == null) return;
        var portal = SpacetimeDBManager.Conn.Db.Portal.Id.Find(_selectedPortalId.Value);
        if (portal == null) return;
        // Re-create portal with new source position (delete + create)
        SpacetimeDBManager.Conn.Reducers.DeletePortal(portal.Id);
        SpacetimeDBManager.Conn.Reducers.CreatePortal(
            portal.SourceZoneId, portal.DestZoneId,
            wx, wy,
            portal.DestSpawnX, portal.DestSpawnY,
            portal.Bidirectional, portal.Label);
    }

    static Color GetZoneColor(int index, float alpha)
    {
        Color[] palette = {
            new Color(0.3f, 0.55f, 1f, alpha),
            new Color(0.3f, 0.78f, 0.4f, alpha),
            new Color(0.7f, 0.4f, 0.3f, alpha),
            new Color(0.8f, 0.7f, 0.2f, alpha),
        };
        return palette[index % palette.Length];
    }

    // -------------------------------------------------------------------------

    /// Custom VisualElement that draws portal edges as lines using generateVisualContent.
    class EdgeDrawer : VisualElement
    {
        internal ulong? SelectedPortalId;
        internal Dictionary<ulong, VisualElement> ZoneNodes;

        public EdgeDrawer()
        {
            generateVisualContent += DrawEdges;
            pickingMode = PickingMode.Ignore;
        }

        void DrawEdges(MeshGenerationContext ctx)
        {
            if (!SpacetimeDBManager.IsSubscribed || ZoneNodes == null) return;
            var painter = ctx.painter2D;
            foreach (var portal in SpacetimeDBManager.Conn.Db.Portal.Iter())
            {
                if (!ZoneNodes.TryGetValue(portal.SourceZoneId, out var srcNode)) continue;
                if (!ZoneNodes.TryGetValue(portal.DestZoneId,   out var dstNode)) continue;
                var srcCenter = GetNodeCenter(srcNode);
                var dstCenter = GetNodeCenter(dstNode);
                bool selected = SelectedPortalId == portal.Id;
                painter.strokeColor = selected
                    ? new Color(1f, 0.86f, 0.3f, 0.9f)
                    : new Color(0.6f, 0.6f, 1f, 0.45f);
                painter.lineWidth = selected ? 2f : 1.5f;
                painter.BeginPath();
                painter.MoveTo(srcCenter);
                painter.LineTo(dstCenter);
                painter.Stroke();
            }
        }

        static Vector2 GetNodeCenter(VisualElement node)
        {
            var r = node.layout;
            return new Vector2(r.x + r.width * 0.5f, r.y + r.height * 0.5f);
        }
    }
}
```

After `BuildZoneNodes`, the `EdgeDrawer` needs access to the node dictionary. Update `BuildZoneNodes` to wire it:

```csharp
// At the end of BuildZoneNodes, before the schedule call:
_edgeDrawer.ZoneNodes       = _zoneNodes;
_edgeDrawer.SelectedPortalId = _selectedPortalId;
```

And update `SelectPortal` to also set it:

```csharp
_edgeDrawer.SelectedPortalId = portalId;
```

- [ ] **Step 4: Add WorldGraphPanel to Editor Scene**

1. Create empty GO `WorldGraphPanel`, add `UIDocument` (same Panel Settings as other panels).
2. Assign `WorldGraphPanel.uxml` as Source Asset.
3. Add `WorldGraphPanel` script. Assign `_styleSheet`.
4. In ToolbarController Inspector, assign this GO to `_graphPanel`.

- [ ] **Step 5: Manual verify**

Open Unity editor. Enter Play mode. Click the ⬡ icon button in the toolbar. The World Graph panel should expand at the bottom. Zone nodes should appear. Click a node — edges should draw between zones.

- [ ] **Step 6: Commit**

```bash
git add editor/Assets/Scripts/Runtime/WorldGraphPanel.cs \
        editor/Assets/UI/WorldGraphPanel.uxml \
        editor/Assets/UI/WorldGraphPanel.uss
git commit -m "feat(editor): WorldGraphPanel - zone node canvas, portal edges, detail panel"
```

---

## Task 12: Editor — ZoneCreationPanel and TilePalettePanel Wiring

**Files:**
- Modify: `editor/Assets/Scripts/Runtime/ZoneCreationPanel.cs`
- Modify: `editor/Assets/Scripts/Runtime/TilePalettePanel.cs`

Add a `SetVisible(bool)` method to each panel so ToolbarController can control them.

- [ ] **Step 1: Add SetVisible to ZoneCreationPanel**

Find `ToggleCollapse()` in `ZoneCreationPanel.cs` and add the following method below it:

```csharp
/// Called by ToolbarController icon button.
public void SetVisible(bool visible)
{
    if (visible == !_isCollapsed) return; // already in correct state
    ToggleCollapse();
}
```

- [ ] **Step 2: Add SetVisible to TilePalettePanel**

Read the file to find where `ToggleCollapse()` is defined, then add the same method:

```csharp
/// Called by ToolbarController icon button.
public void SetVisible(bool visible)
{
    if (visible == !_isCollapsed) return;
    ToggleCollapse();
}
```

- [ ] **Step 3: Wire panels in ToolbarController Inspector**

In Unity: select the `ToolbarController` GO in the Editor scene. Drag `ZoneCreationPanel` GO into the `_zonePanel` slot, and `TilePalettePanel` GO into the `_brushPanel` slot.

- [ ] **Step 4: Manual verify**

Enter Play mode in the Editor project:
- Click 🗺 — Zone Manager panel should open/close
- Click 🖌 — Brush panel should open/close
- Click ⬡ — World Graph panel should expand/collapse at bottom
- Active icon should show blue tint
- Opening one left-side panel should close the other

- [ ] **Step 5: Final commit**

```bash
git add editor/Assets/Scripts/Runtime/ZoneCreationPanel.cs \
        editor/Assets/Scripts/Runtime/TilePalettePanel.cs
git commit -m "feat(editor): wire ZoneCreationPanel and TilePalettePanel to ToolbarController"
```

---

## End-to-End Smoke Test

After all tasks are complete:

1. Start SpacetimeDB: `spacetime start`
2. Open Editor project in Unity → Play mode
3. Create two zones if needed (Zone Manager panel)
4. Click ⬡ → World Graph opens
5. Note: to create a portal, use `spacetime call` until the portal creation UI is implemented:
   ```
   spacetime call --server local zoneforge-server create_portal \
     '{"source_zone_id":1,"dest_zone_id":2,"source_x":10.0,"source_y":10.0,"dest_spawn_x":5.0,"dest_spawn_y":5.0,"bidirectional":true,"label":"Test Portal"}'
   ```
6. World Graph should show an edge between zones; ring should appear in 3D viewport
7. Click ring → portal detail should populate in the right panel
8. Open Client project → Play mode
9. Walk player to (10,10) → ring appears, pre-load triggers at 5u, Ripple Warp at 1.5u
10. Player arrives in zone 2 at (5,5)
11. 2s later: background reconnect fires (Console log)
