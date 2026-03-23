# Enemy AI — Client Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire up the Unity client to subscribe to enemy tables, render enemies with health bars, and support enemy targeting in combat.

**Architecture:** `EnemyManager` mirrors `PlayerManager` — subscribes to server `Enemy` and `EnemyDef` tables, spawns/destroys capsule GameObjects, and forwards updates to `EnemyController` for smooth position lerping. `CombatManager` gains an enemy position registry. `CombatInputHandler` gains mixed-target cycling.

**Tech Stack:** Unity 2022.3 LTS, C#, SpacetimeDB C# SDK

**Spec:** `docs/superpowers/specs/2026-03-22-enemy-ai-design.md`

**Prerequisites:** Server plan (`2026-03-22-enemy-ai-server.md`) must be complete and deployed before this plan begins. The autogen refresh in Task 1 is mandatory.

---

## File Map

| File | Change |
|------|--------|
| `client/Assets/Scripts/autogen/` | Regenerated — do not edit manually |
| `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs` | Add Enemy/EnemyDef subscriptions + events |
| `client/Assets/Scripts/Enemy/EnemyManager.cs` | **Create** — singleton, mirrors PlayerManager |
| `client/Assets/Scripts/Enemy/EnemyController.cs` | **Create** — per-enemy component: lerp + death anim |
| `client/Assets/Scripts/Enemy/EnemyHealthBar.cs` | **Create** — world-space health bar |
| `client/Assets/Scripts/Combat/CombatManager.cs` | Add enemy position registry |
| `client/Assets/Scripts/Combat/CombatInputHandler.cs` | Add enemy targeting |

---

## Task 1: Regenerate autogen bindings

Run from the `client/` directory. The server schema changed (new tables + CombatLog fields) so bindings must be regenerated before any C# work.

- [ ] **Regenerate:**

```bash
cd client
spacetime generate --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

Expected: files written to `Assets/Scripts/autogen/` with no errors.

- [ ] **Verify new files exist:**

```bash
ls Assets/Scripts/autogen/Tables/Enemy.g.cs
ls Assets/Scripts/autogen/Tables/EnemyDef.g.cs
ls Assets/Scripts/autogen/Tables/SpawnPoint.g.cs
ls Assets/Scripts/autogen/Types/AiState.g.cs
ls Assets/Scripts/autogen/Types/EnemyType.g.cs
```

- [ ] **Reimport in Unity:** Assets → Reimport All (or wait for Unity to auto-detect)

- [ ] **Verify compile in Unity Console:** zero errors. If the `CombatLog` type now has `AttackerIsEnemy` / `TargetIsEnemy` fields, existing code that creates `CombatLog` objects will not be affected (server-side only). Client code only reads CombatLog — no breaking changes on the client.

- [ ] **Commit:**

```bash
git add client/Assets/Scripts/autogen/
git commit -m "chore(client): regenerate autogen bindings for Group 9 schema"
```

---

## Task 2: SpacetimeDBManager — subscribe to enemy tables + add events

**Files:**
- Modify: `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs`

- [ ] **Add static events** after the existing `OnStatusEffectDeleted` event (~line 27):

```csharp
public static event Action<Enemy> OnEnemyInserted;
public static event Action<Enemy, Enemy> OnEnemyUpdated;
public static event Action<Enemy> OnEnemyDeleted;
```

- [ ] **Add enemy table subscription strings** in `OnConnect` — extend the `Subscribe` array:

```csharp
"SELECT * FROM enemy",
"SELECT * FROM enemy_def",
```

Full updated Subscribe call:
```csharp
conn.SubscriptionBuilder()
    .OnApplied(OnSubscriptionApplied)
    .Subscribe(new[]
    {
        "SELECT * FROM player",
        "SELECT * FROM zone",
        "SELECT * FROM entity_instance",
        "SELECT * FROM terrain_chunk",
        "SELECT * FROM ability",
        "SELECT * FROM player_cooldown",
        "SELECT * FROM status_effect",
        "SELECT * FROM combat_log",
        "SELECT * FROM enemy",
        "SELECT * FROM enemy_def",
    });
```

- [ ] **Register enemy callbacks** in `OnSubscriptionApplied` after the existing `StatusEffect` callbacks:

```csharp
Conn.Db.Enemy.OnInsert += (eventCtx, enemy) => OnEnemyInserted?.Invoke(enemy);
Conn.Db.Enemy.OnUpdate += (eventCtx, oldEnemy, newEnemy) => OnEnemyUpdated?.Invoke(oldEnemy, newEnemy);
Conn.Db.Enemy.OnDelete += (eventCtx, enemy) => OnEnemyDeleted?.Invoke(enemy);
```

- [ ] **Build check in Unity** (compile, no Play mode yet): confirm zero errors in Console.

- [ ] **Commit:**

```bash
git add client/Assets/Scripts/Runtime/SpacetimeDBManager.cs
git commit -m "feat(client): subscribe to enemy/enemy_def tables, add OnEnemy* events"
```

---

## Task 3: EnemyManager singleton

**Files:**
- Create: `client/Assets/Scripts/Enemy/EnemyManager.cs`

- [ ] **Create `Assets/Scripts/Enemy/` folder** in Unity Project view (right-click → Create → Folder) or via filesystem.

- [ ] **Create `EnemyManager.cs`:**

```csharp
using System.Collections.Generic;
using UnityEngine;
using SpacetimeDB.Types;

/// <summary>
/// Scene singleton. Subscribes to SpacetimeDBManager enemy events and maintains
/// one capsule GameObject per Enemy row. Mirrors PlayerManager pattern.
/// Requires CombatManager to be present in the scene before OnConnected fires.
/// </summary>
public class EnemyManager : MonoBehaviour
{
    public static EnemyManager Instance { get; private set; }

    private readonly Dictionary<ulong, GameObject> _enemies = new();

    void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);

        SpacetimeDBManager.OnEnemyInserted += OnEnemyInserted;
        SpacetimeDBManager.OnEnemyUpdated  += OnEnemyUpdated;
        SpacetimeDBManager.OnEnemyDeleted  += OnEnemyDeleted;
        SpacetimeDBManager.OnConnected     += OnConnected;
    }

    void OnDestroy()
    {
        SpacetimeDBManager.OnEnemyInserted -= OnEnemyInserted;
        SpacetimeDBManager.OnEnemyUpdated  -= OnEnemyUpdated;
        SpacetimeDBManager.OnEnemyDeleted  -= OnEnemyDeleted;
        SpacetimeDBManager.OnConnected     -= OnConnected;
    }

    void OnConnected()
    {
        // Backfill: initial rows arrive before SpacetimeDBManager registers callbacks
        foreach (var enemy in SpacetimeDBManager.Conn.Db.Enemy.Iter())
        {
            if (!_enemies.ContainsKey(enemy.Id))
                SpawnEnemy(enemy);
        }
    }

    void OnEnemyInserted(Enemy enemy)
    {
        if (_enemies.ContainsKey(enemy.Id)) return;
        SpawnEnemy(enemy);
    }

    void OnEnemyUpdated(Enemy oldEnemy, Enemy newEnemy)
    {
        if (!_enemies.TryGetValue(newEnemy.Id, out var go)) return;
        go.GetComponent<EnemyController>()?.ReceiveUpdate(newEnemy);
        CombatManager.Instance?.RegisterEnemyPosition(
            newEnemy.Id,
            new Vector3(newEnemy.PositionX, 1f, newEnemy.PositionY));
    }

    void OnEnemyDeleted(Enemy enemy)
    {
        if (!_enemies.TryGetValue(enemy.Id, out var go)) return;
        Destroy(go);
        _enemies.Remove(enemy.Id);
    }

    /// <summary>Returns the GameObject for an enemy id, or null if not spawned.</summary>
    public GameObject GetEnemyObject(ulong enemyId) =>
        _enemies.TryGetValue(enemyId, out var go) ? go : null;

    void SpawnEnemy(Enemy enemy)
    {
        // Look up the definition for color/name — iterate since no direct key access
        EnemyDefinition def = null;
        foreach (var d in SpacetimeDBManager.Conn.Db.EnemyDef.Iter())
        {
            if (d.Id == enemy.EnemyDefId) { def = d; break; }
        }

        var go = GameObject.CreatePrimitive(PrimitiveType.Capsule);
        go.name = $"Enemy_{enemy.Id}_{def?.Name ?? "Unknown"}";

        var rend = go.GetComponent<Renderer>();
        var mat  = new Material(Shader.Find("Universal Render Pipeline/Lit"));
        mat.color = GetEnemyColor(def?.EnemyType ?? EnemyType.Melee);
        rend.material = mat;

        // Disable collider — server owns position; no physics needed
        var col = go.GetComponent<CapsuleCollider>();
        if (col != null) col.enabled = false;

        var ctrl = go.AddComponent<EnemyController>();
        ctrl.Init(enemy);

        var hb = go.AddComponent<EnemyHealthBar>();
        hb.Init(enemy, def);

        _enemies[enemy.Id] = go;
        CombatManager.Instance?.RegisterEnemyPosition(
            enemy.Id,
            new Vector3(enemy.PositionX, 1f, enemy.PositionY));

        Debug.Log($"[EnemyManager] Spawned {go.name} at ({enemy.PositionX}, {enemy.PositionY})");
    }

    static Color GetEnemyColor(EnemyType type) => type switch
    {
        EnemyType.Melee  => new Color(1f, 0.45f, 0.1f),  // orange
        EnemyType.Ranged => new Color(1f, 0.90f, 0.1f),  // yellow
        EnemyType.Caster => new Color(0.6f, 0.1f, 0.9f), // purple
        _                => Color.white,
    };
}
```

- [ ] **Compile check in Unity Console:** zero errors.

- [ ] **Commit:**

```bash
git add client/Assets/Scripts/Enemy/EnemyManager.cs
git commit -m "feat(client): add EnemyManager singleton"
```

---

## Task 4: EnemyController component

**Files:**
- Create: `client/Assets/Scripts/Enemy/EnemyController.cs`

- [ ] **Create `EnemyController.cs`:**

```csharp
using UnityEngine;
using SpacetimeDB.Types;

/// <summary>
/// Per-enemy component. Lerps toward the last server-reported position each Update
/// (smooths 500ms tick intervals). Plays a scale-to-zero death animation when
/// is_dead becomes true, matching the 3s server despawn window.
/// </summary>
public class EnemyController : MonoBehaviour
{
    private Vector3 _targetPosition;
    private bool    _isDead;
    private float   _deathTimer;

    private const float DeathAnimDuration = 2.5f; // completes before 3s despawn fires
    private const float LerpSpeed         = 8f;   // units/sec toward server position

    public void Init(Enemy enemy)
    {
        var pos = new Vector3(enemy.PositionX, 1f, enemy.PositionY);
        transform.position = pos;
        _targetPosition    = pos;
        _isDead            = enemy.IsDead;
    }

    public void ReceiveUpdate(Enemy enemy)
    {
        if (!_isDead && enemy.IsDead)
        {
            _isDead      = true;
            _deathTimer  = DeathAnimDuration;
        }

        if (!_isDead)
            _targetPosition = new Vector3(enemy.PositionX, 1f, enemy.PositionY);
    }

    void Update()
    {
        if (_isDead)
        {
            _deathTimer = Mathf.Max(0f, _deathTimer - Time.deltaTime);
            float t = 1f - _deathTimer / DeathAnimDuration;
            transform.localScale = Vector3.Lerp(Vector3.one, Vector3.zero, t);
            return;
        }

        transform.position = Vector3.Lerp(transform.position, _targetPosition, Time.deltaTime * LerpSpeed);
    }
}
```

- [ ] **Compile check:** zero errors.

- [ ] **Commit:**

```bash
git add client/Assets/Scripts/Enemy/EnemyController.cs
git commit -m "feat(client): add EnemyController (position lerp + death animation)"
```

---

## Task 5: EnemyHealthBar component

**Files:**
- Create: `client/Assets/Scripts/Enemy/EnemyHealthBar.cs`

- [ ] **Create `EnemyHealthBar.cs`** — follows the same canvas construction pattern as `PlayerHealthBar`:

```csharp
using UnityEngine;
using UnityEngine.UI;
using SpacetimeDB.Types;

/// <summary>
/// World-space floating health bar above an enemy capsule.
/// Added by EnemyManager — call Init(enemy, def) immediately after AddComponent.
/// Bills toward the camera every LateUpdate. Hides on death.
/// </summary>
public class EnemyHealthBar : MonoBehaviour
{
    private ulong _enemyId;
    private int   _maxHealth;
    private Image _fill;
    private Canvas _canvas;

    public void Init(Enemy enemy, EnemyDefinition def)
    {
        _enemyId   = enemy.Id;
        _maxHealth = def?.MaxHealth ?? 100;
        BuildBar(def?.Name ?? "Enemy");
        UpdateFill(enemy.Health, _maxHealth);
    }

    void OnEnable()  => SpacetimeDBManager.OnEnemyUpdated += OnEnemyUpdated;
    void OnDisable() => SpacetimeDBManager.OnEnemyUpdated -= OnEnemyUpdated;

    void LateUpdate()
    {
        if (_canvas == null) return;
        var cam = Camera.main;
        if (cam != null)
            _canvas.transform.forward = cam.transform.forward;
    }

    void OnEnemyUpdated(Enemy _, Enemy e)
    {
        if (e.Id != _enemyId) return;
        if (e.IsDead)
        {
            if (_canvas != null) _canvas.gameObject.SetActive(false);
            return;
        }
        UpdateFill(e.Health, _maxHealth);
    }

    void UpdateFill(int health, int maxHealth)
    {
        if (_fill == null) return;
        float pct    = maxHealth > 0 ? Mathf.Clamp01((float)health / maxHealth) : 1f;
        _fill.fillAmount = pct;
        _fill.color      = Color.Lerp(Color.red, new Color(0.15f, 0.75f, 0.15f), pct);
    }

    void BuildBar(string enemyName)
    {
        const float PixW    = 160f;
        const float PixH    = 36f;
        const float Scale   = 0.01f;
        const float YOffset = 2.8f;

        var canvasGo = new GameObject("EnemyHealthBarCanvas");
        canvasGo.transform.SetParent(transform, false);
        canvasGo.transform.localPosition = new Vector3(0f, YOffset, 0f);
        canvasGo.transform.localScale    = Vector3.one * Scale;

        _canvas = canvasGo.AddComponent<Canvas>();
        _canvas.renderMode   = RenderMode.WorldSpace;
        _canvas.sortingOrder = 5;

        var canvasRect = canvasGo.GetComponent<RectTransform>();
        canvasRect.sizeDelta = new Vector2(PixW, PixH);

        // Name text — top half
        var nameGo   = new GameObject("NameText");
        nameGo.transform.SetParent(canvasGo.transform, false);
        var nameRect = nameGo.AddComponent<RectTransform>();
        nameRect.anchorMin = new Vector2(0f, 0.5f);
        nameRect.anchorMax = new Vector2(1f, 1f);
        nameRect.offsetMin = nameRect.offsetMax = Vector2.zero;
        var nameText = nameGo.AddComponent<Text>();
        nameText.text      = enemyName;
        nameText.font      = Resources.GetBuiltinResource<Font>("LegacyRuntime.ttf");
        nameText.fontSize  = 14;
        nameText.color     = new Color(1f, 0.6f, 0.6f);
        nameText.alignment = TextAnchor.MiddleCenter;

        // Bar background — bottom half
        var bgGo   = new GameObject("BarBG");
        bgGo.transform.SetParent(canvasGo.transform, false);
        var bgRect = bgGo.AddComponent<RectTransform>();
        bgRect.anchorMin = new Vector2(0f, 0f);
        bgRect.anchorMax = new Vector2(1f, 0.5f);
        bgRect.offsetMin = new Vector2(4f,  2f);
        bgRect.offsetMax = new Vector2(-4f, -2f);
        bgGo.AddComponent<Image>().color = new Color(0.15f, 0.05f, 0.05f, 0.9f);

        // Fill bar
        var fillGo   = new GameObject("BarFill");
        fillGo.transform.SetParent(bgGo.transform, false);
        var fillRect = fillGo.AddComponent<RectTransform>();
        fillRect.anchorMin = Vector2.zero;
        fillRect.anchorMax = Vector2.one;
        fillRect.offsetMin = new Vector2(1f, 1f);
        fillRect.offsetMax = new Vector2(-1f, -1f);
        _fill = fillGo.AddComponent<Image>();
        _fill.type       = Image.Type.Filled;
        _fill.fillMethod = Image.FillMethod.Horizontal;
        _fill.fillOrigin = 0;
    }
}
```

- [ ] **Compile check:** zero errors.

- [ ] **Commit:**

```bash
git add client/Assets/Scripts/Enemy/EnemyHealthBar.cs
git commit -m "feat(client): add EnemyHealthBar world-space health bar"
```

---

## Task 6: CombatManager — add enemy position registry

**Files:**
- Modify: `client/Assets/Scripts/Combat/CombatManager.cs`

- [ ] **Add `_enemyPositions` field** after `_playerPositions` (~line 19):

```csharp
private readonly Dictionary<ulong, Vector3> _enemyPositions = new();
```

- [ ] **Add `RegisterEnemyPosition` method** after `RegisterPlayerPosition`:

```csharp
/// <summary>Called by EnemyManager after spawning or moving an enemy capsule.</summary>
public void RegisterEnemyPosition(ulong enemyId, Vector3 worldPos)
{
    _enemyPositions[enemyId] = worldPos;
}
```

- [ ] **Update `OnCombatLogInserted`** to route position lookups using the new bool fields. Replace the existing attacker/target position lookup block:

```csharp
// Determine attacker position
Vector3 attackerPos = default;
if (log.AttackerIsEnemy)
{
    if (!_enemyPositions.TryGetValue(log.AttackerId, out attackerPos))
        Debug.LogWarning($"[CombatManager] No position for enemy attacker {log.AttackerId}");
}
else
{
    if (!_playerPositions.TryGetValue(log.AttackerId, out attackerPos))
        Debug.LogWarning($"[CombatManager] No position for player attacker {log.AttackerId}");
}

// Determine target position
Vector3 targetPos = default;
if (log.TargetIsEnemy)
{
    if (!_enemyPositions.TryGetValue(log.TargetId, out targetPos))
        Debug.LogWarning($"[CombatManager] No position for enemy target {log.TargetId}");
}
else
{
    if (!_playerPositions.TryGetValue(log.TargetId, out targetPos))
        Debug.LogWarning($"[CombatManager] No position for player target {log.TargetId}");
}
```

Remove the old two-line position lookup block that assumed both IDs were player IDs:
```csharp
// DELETE these two lines:
if (!_playerPositions.TryGetValue(log.AttackerId, out var attackerPos))
    Debug.LogWarning($"[CombatManager] No position for attacker {log.AttackerId}");
if (!_playerPositions.TryGetValue(log.TargetId, out var targetPos))
    Debug.LogWarning($"[CombatManager] No position for target {log.TargetId}");
```

Also update `OnPlayerUpdated` to clean up enemy position on player death — no change needed there, it only tracks player positions.

- [ ] **Compile check:** zero errors.

- [ ] **Commit:**

```bash
git add client/Assets/Scripts/Combat/CombatManager.cs
git commit -m "feat(client): add enemy position registry to CombatManager"
```

---

## Task 7: CombatInputHandler — mixed player/enemy targeting

The existing `CycleTarget` only cycles players. This task extends it to cycle both players and enemies, and dispatches `AttackEnemy` vs `UseAbility` based on target type.

**Files:**
- Modify: `client/Assets/Scripts/Combat/CombatInputHandler.cs`

- [ ] **Add target-tracking fields** after `_selectionRing` field (~line 23):

```csharp
private bool _targetIsEnemy;  // false = player target, true = enemy target
```

- [ ] **Replace `CycleTarget` method** with a version that includes enemies:

```csharp
private void CycleTarget()
{
    // Build a unified candidate list: (id, isEnemy, position for ring)
    var candidates = new System.Collections.Generic.List<(ulong id, bool isEnemy)>();

    foreach (var p in SpacetimeDBManager.Conn.Db.Player.Iter())
    {
        if (p.Identity == SpacetimeDBManager.LocalIdentity) continue;
        if (p.IsDead) continue;
        candidates.Add((p.Id, false));
    }

    foreach (var e in SpacetimeDBManager.Conn.Db.Enemy.Iter())
    {
        if (e.IsDead) continue;
        candidates.Add((e.Id, true));
    }

    if (candidates.Count == 0)
    {
        _selectedTargetId = 0;
        _targetIsEnemy    = false;
        _selectionRing.SetActive(false);
        return;
    }

    // Find current target index in unified list; advance by 1
    int currentIndex = candidates.FindIndex(c => c.id == _selectedTargetId && c.isEnemy == _targetIsEnemy);
    int nextIndex    = (currentIndex + 1) % candidates.Count;

    _selectedTargetId = candidates[nextIndex].id;
    _targetIsEnemy    = candidates[nextIndex].isEnemy;
    _selectionRing.SetActive(true);

    Debug.Log($"[CombatInput] Target: {(_targetIsEnemy ? "enemy" : "player")} id={_selectedTargetId}");
}
```

- [ ] **Replace `TryUseAbility` method** to dispatch to the correct reducer:

```csharp
private void TryUseAbility(ulong abilityId, bool requireTarget, Player localPlayer)
{
    if (requireTarget && _selectedTargetId == 0)
    {
        Debug.Log("[CombatInput] No target selected");
        return;
    }

    if (IsOnCooldown(abilityId, localPlayer.Id))
    {
        Debug.Log($"[CombatInput] Ability {abilityId} on cooldown");
        return;
    }

    if (!requireTarget)
    {
        // Self-cast (Heal) — always targets local player
        SpacetimeDBManager.Conn.Reducers.UseAbility(abilityId, localPlayer.Id);
        Debug.Log($"[CombatInput] UseAbility({abilityId}, self)");
        return;
    }

    if (_targetIsEnemy)
    {
        SpacetimeDBManager.Conn.Reducers.AttackEnemy(abilityId, _selectedTargetId);
        Debug.Log($"[CombatInput] AttackEnemy({abilityId}, enemy={_selectedTargetId})");
    }
    else
    {
        SpacetimeDBManager.Conn.Reducers.UseAbility(abilityId, _selectedTargetId);
        Debug.Log($"[CombatInput] UseAbility({abilityId}, player={_selectedTargetId})");
    }
}
```

- [ ] **Update `UpdateSelectionRing`** to look up GameObjects in the correct manager:

```csharp
private void UpdateSelectionRing()
{
    if (_selectedTargetId == 0) { _selectionRing.SetActive(false); return; }

    GameObject targetGo = _targetIsEnemy
        ? EnemyManager.Instance?.GetEnemyObject(_selectedTargetId)
        : PlayerManager.Instance?.GetPlayerObject(_selectedTargetId);

    if (targetGo == null)
    {
        _selectedTargetId = 0;
        _targetIsEnemy    = false;
        _selectionRing.SetActive(false);
        return;
    }

    var pos = targetGo.transform.position;
    _selectionRing.transform.position = new Vector3(pos.x, 0.05f, pos.z);
    _selectionRing.SetActive(true);
}
```

- [ ] **Clear `_targetIsEnemy` in the dead-player early-return block** (in `Update`, where `_selectedTargetId = 0` is already set):

```csharp
_selectedTargetId = 0;
_targetIsEnemy    = false;  // add this line
```

- [ ] **Compile check:** zero errors.

- [ ] **Commit:**

```bash
git add client/Assets/Scripts/Combat/CombatInputHandler.cs
git commit -m "feat(client): add enemy targeting to CombatInputHandler"
```

---

## Task 8: Scene setup + integration test

**Files:**
- Unity SampleScene (no code changes — scene wiring only)

- [ ] **Add EnemyManager GameObject** to SampleScene: right-click in Hierarchy → Create Empty, name it `EnemyManager`, add the `EnemyManager` component. Position is irrelevant.

- [ ] **Verify scene has all required singletons** (check Hierarchy):
  - `SpacetimeDBManager` ✓ (existing)
  - `PlayerManager` ✓ (existing)
  - `CombatManager` ✓ (existing)
  - `EnemyManager` ✓ (just added)

- [ ] **Enter Play mode** — connect to local SpacetimeDB. Check Console for errors.

- [ ] **Create an EnemyDefinition via `spacetime call`** (type manually — do not paste):

```bash
spacetime call zoneforge-server create_enemy_def '"Goblin Scout"' '{"Melee":{}}' '"GoblinScout"' 50 10 8.0 2.5 2000 3.0
```

Expected: no error in terminal. Verify in Unity Console that no null-ref errors appear.

> ⚠️ If `create_enemy_def` returns "Not authorized": your identity is not in `ADMIN_IDENTITIES`. Run `spacetime identity list`, copy your hex, add it to the constant in `lib.rs`, republish with `--delete-data`, and recreate the zone.

- [ ] **Create a SpawnPoint near zone center:**

```bash
spacetime call zoneforge-server create_spawn_point 1 32.0 32.0 1 1 10
```

Expected: an enemy capsule (orange) appears in the Unity scene near position (32, 32). Health bar shows above it.

- [ ] **Verify AI behaviour:**
  - Walk the local player toward the enemy — at ~8 units range it should start moving toward you (Chase state)
  - At ~2.5 units it should stop and deal damage (Attack state — watch floating numbers and your health bar)
  - Move away past aggro range — enemy should return to spawn (Idle state)

- [ ] **Verify Tab targeting:**
  - Press Tab — selection ring should appear under the enemy
  - Press Tab again — ring cycles to the next player (if any), or wraps back to enemy

- [ ] **Verify attack_enemy:**
  - Tab to select the enemy, press 1 (Auto-Attack) — floating damage number appears above the enemy, its health bar decreases
  - Kill the enemy (spam 1/2) — health bar hides, capsule scales down over 2.5s, then disappears after 3s
  - After `respawn_delay_s` (10s from the call above), enemy reappears at spawn point

- [ ] **Final commit:**

```bash
git add -A
git commit -m "feat(client): Group 9 enemy AI client complete — EnemyManager, targeting, combat wired"
```

---

**Client plan complete. Group 9 Enemy AI (Basic) is fully implemented.**
