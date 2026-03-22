# Combat Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement server-authoritative tab-target combat with abilities, cooldowns, death/respawn, and pooled projectile VFX.

**Architecture:** Split reducers — `use_ability` validates and deducts resources, `apply_damage` is a plain Rust helper function called internally by `use_ability` and future systems. Client-side: `ZoneForgePoolManager` pre-allocates projectile and VFX pools; `CombatInputHandler` handles Tab-to-target and number-key-to-cast; `CombatManager` reacts to `CombatLog` inserts and spawns VFX.

**Tech Stack:** Rust + SpacetimeDB 2.x (server), Unity 2022.3 C# + SpacetimeDB SDK (client)

**Spec:** `docs/superpowers/specs/2026-03-22-combat-foundation-design.md`

---

## File Map

### Server — modify
- `server/spacetimedb/src/lib.rs` — add enums, 5 new tables, extend Player, 5 new reducers/functions

### Client — create
- `client/Assets/Scripts/Combat/ZoneForgePoolManager.cs`
- `client/Assets/Scripts/Combat/PooledProjectile.cs`
- `client/Assets/Scripts/Combat/PooledVFX.cs`
- `client/Assets/Scripts/Combat/CombatManager.cs`
- `client/Assets/Scripts/Combat/CombatInputHandler.cs`

### Client — modify
- `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs` — add subscriptions + events for 4 new tables

### Regenerated (never edit manually)
- `client/Assets/Scripts/autogen/` — regenerated after server publish

---

## Task 1: Extend Player + add combat enums to lib.rs

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Step 1: Update the import line**

Replace the existing `use spacetimedb::{...}` at line 1 with:

```rust
use spacetimedb::{table, reducer, ReducerContext, Identity, Table, SpacetimeType, Timestamp, ScheduleAt};
```

- [ ] **Step 2: Add `AbilityType` and `StatusEffectType` enums after the import**

Insert immediately after the `use` line, before the `Player` struct:

```rust
#[derive(SpacetimeType, Clone, Debug, PartialEq)]
pub enum AbilityType {
    MeleeAttack,
    Projectile,
    SelfCast,
}

#[derive(SpacetimeType, Clone, Debug, PartialEq)]
pub enum StatusEffectType {
    Burn,
    Freeze,
    Stun,
    Poison,
}
```

- [ ] **Step 3: Add `mana`, `max_mana`, `is_dead` to the Player struct**

The current Player struct ends with `pub max_health: i32,`. Add three new fields after it:

```rust
    pub mana: i32,
    pub max_mana: i32,
    pub is_dead: bool,
```

- [ ] **Step 4: Update `create_player` to initialise the new fields**

In `create_player`, find the `Player { ... }` literal and add the three new fields:

```rust
        mana: 100,
        max_mana: 100,
        is_dead: false,
```

- [ ] **Step 5: Verify build compiles**

```bash
cd /home/beek/zoneforge/server && spacetime build
```

Expected: `Build finished successfully.` (wasm-opt warning is normal).
If compile errors, fix them before continuing.

- [ ] **Step 6: Commit**

```bash
cd /home/beek/zoneforge
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): extend Player with mana/max_mana/is_dead + add combat enums"
```

---

## Task 2: Add the five new server tables

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

Add all five table structs after the `TerrainChunk` table definition (around line 46, before the `create_player` reducer).

- [ ] **Step 1: Add the `Ability` table**

```rust
#[table(accessor = ability, public)]
pub struct Ability {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub name: String,
    pub damage: i32,
    pub cooldown_ms: u64,
    pub mana_cost: i32,
    pub range: f32,
    pub ability_type: AbilityType,
}
```

- [ ] **Step 2: Add the `PlayerCooldown` table**

Note: uses `id: u64` auto_inc PK + btree index on `player_id` (composite PKs are not needed — find-by-player-then-filter-by-ability is the upsert pattern).

```rust
#[table(accessor = player_cooldown, public)]
pub struct PlayerCooldown {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    #[index(btree)]
    pub player_id: u64,
    pub ability_id: u64,
    pub ready_at: Timestamp,
}
```

- [ ] **Step 3: Add the `StatusEffect` table**

```rust
#[table(accessor = status_effect, public)]
pub struct StatusEffect {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    #[index(btree)]
    pub target_id: u64,
    pub effect_type: StatusEffectType,
    pub expires_at: Timestamp,
    pub damage_per_tick: i32,
}
```

- [ ] **Step 4: Add the `CombatLog` table**

```rust
#[table(accessor = combat_log, public)]
pub struct CombatLog {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub timestamp: Timestamp,
    pub attacker_id: u64,
    pub target_id: u64,
    pub ability_id: u64,
    pub damage_dealt: i32,
    pub overkill: i32,
}
```

- [ ] **Step 5: Add the `StatusEffectTick` scheduled table**

This is the scheduler for the DoT tick. It is private (no `public` flag) — clients don't need it.

```rust
#[table(accessor = status_effect_tick, scheduled(tick_status_effects))]
pub struct StatusEffectTick {
    #[primary_key]
    #[auto_inc]
    pub scheduled_id: u64,
    pub scheduled_at: ScheduleAt,
}
```

- [ ] **Step 6: Verify build**

```bash
cd /home/beek/zoneforge/server && spacetime build
```

Expected: `Build finished successfully.`

- [ ] **Step 7: Commit**

```bash
cd /home/beek/zoneforge
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add Ability, PlayerCooldown, StatusEffect, CombatLog, StatusEffectTick tables"
```

---

## Task 3: Add `init` reducer and `apply_damage` helper

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

Add these after the existing table definitions, before `create_player`.

- [ ] **Step 1: Add the `init` reducer**

This seeds 3 abilities on first publish and starts the recurring status effect tick. The guard `if ctx.db.ability().iter().next().is_none()` makes it idempotent.

```rust
#[reducer(init)]
pub fn init(ctx: &ReducerContext) {
    // Seed starter abilities if table is empty
    if ctx.db.ability().iter().next().is_none() {
        ctx.db.ability().insert(Ability {
            id: 0,
            name: "Auto-Attack".to_string(),
            damage: 20,
            cooldown_ms: 500,
            mana_cost: 0,
            range: 2.5,
            ability_type: AbilityType::MeleeAttack,
        });
        ctx.db.ability().insert(Ability {
            id: 0,
            name: "Fireball".to_string(),
            damage: 50,
            cooldown_ms: 3000,
            mana_cost: 20,
            range: 15.0,
            ability_type: AbilityType::Projectile,
        });
        ctx.db.ability().insert(Ability {
            id: 0,
            name: "Heal".to_string(),
            damage: -50,
            cooldown_ms: 10000,
            mana_cost: 30,
            range: 0.0,
            ability_type: AbilityType::SelfCast,
        });
        log::info!("init: seeded 3 abilities");
    }

    // Start the recurring status effect tick if not already scheduled
    if ctx.db.status_effect_tick().iter().next().is_none() {
        ctx.db.status_effect_tick().insert(StatusEffectTick {
            scheduled_id: 0,
            scheduled_at: ScheduleAt::Time(
                ctx.timestamp + std::time::Duration::from_secs(1)
            ),
        });
        log::info!("init: scheduled status effect tick");
    }
}
```

- [ ] **Step 2: Add the `apply_damage` helper function**

This is a plain Rust function (no `#[reducer]`), called directly by `use_ability` and future reducers. It modifies the DB via `ctx`.

```rust
fn apply_damage(
    ctx: &ReducerContext,
    target_id: u64,
    attacker_id: u64,
    ability_id: u64,
    amount: i32,
) {
    let Some(target) = ctx.db.player().id().find(&target_id) else {
        return;
    };

    let new_health = (target.health - amount).clamp(0, target.max_health);
    let overkill = if amount > 0 && amount > target.health {
        amount - target.health
    } else {
        0
    };
    let new_is_dead = (new_health == 0 && amount > 0) || target.is_dead;

    ctx.db.player().id().update(Player {
        health: new_health,
        is_dead: new_is_dead,
        ..target
    });

    ctx.db.combat_log().insert(CombatLog {
        id: 0,
        timestamp: ctx.timestamp,
        attacker_id,
        target_id,
        ability_id,
        damage_dealt: amount,
        overkill,
    });

    log::info!(
        "apply_damage: target={} amount={} new_health={} dead={}",
        target_id, amount, new_health, new_is_dead
    );
}
```

- [ ] **Step 3: Verify build**

```bash
cd /home/beek/zoneforge/server && spacetime build
```

Expected: `Build finished successfully.`

- [ ] **Step 4: Commit**

```bash
cd /home/beek/zoneforge
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add init ability seeding and apply_damage helper"
```

---

## Task 4: Implement `use_ability` reducer

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

Add after `apply_damage`.

- [ ] **Step 1: Add the `use_ability` reducer**

```rust
#[reducer]
pub fn use_ability(
    ctx: &ReducerContext,
    ability_id: u64,
    target_id: u64,
) -> Result<(), String> {
    // 1. Caller must exist and not be dead
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or("Player not found")?;
    if player.is_dead {
        return Err("Cannot use ability while dead".to_string());
    }

    // 2. Ability must exist
    let ability = ctx.db.ability().id().find(&ability_id)
        .ok_or("Ability not found")?;

    // 3. Self-cast: target must be the caller
    if ability.ability_type == AbilityType::SelfCast {
        if target_id != player.id {
            return Err("Self-cast ability must target self".to_string());
        }
    } else {
        // 4. Target must exist and not be dead
        let target = ctx.db.player().id().find(&target_id)
            .ok_or("Target not found")?;
        if target.is_dead {
            return Err("Target is already dead".to_string());
        }

        // 5. Range check (XZ distance)
        let dx = player.position_x - target.position_x;
        let dz = player.position_y - target.position_y;
        let dist_sq = dx * dx + dz * dz;
        if dist_sq > ability.range * ability.range {
            return Err(format!(
                "Target out of range (dist={:.1}, range={:.1})",
                dist_sq.sqrt(), ability.range
            ));
        }
    }

    // 6. Cooldown check
    let now_us = ctx.timestamp
        .to_duration_since_unix_epoch()
        .unwrap_or_default()
        .as_micros();
    let on_cooldown = ctx.db.player_cooldown()
        .player_id()
        .filter(&player.id)
        .any(|cd| {
            if cd.ability_id != ability_id { return false; }
            let ready_us = cd.ready_at
                .to_duration_since_unix_epoch()
                .unwrap_or_default()
                .as_micros();
            ready_us > now_us
        });
    if on_cooldown {
        return Err("Ability on cooldown".to_string());
    }

    // 7. Mana check
    if player.mana < ability.mana_cost {
        return Err(format!(
            "Insufficient mana ({}/{})", player.mana, ability.mana_cost
        ));
    }

    // All checks passed — save id before player is moved into the struct update
    let player_id = player.id;
    let new_mana = player.mana - ability.mana_cost;
    ctx.db.player().id().update(Player { mana: new_mana, ..player });
    // `player` is moved above; use `player_id` from here on

    // Upsert cooldown: find existing row for this player+ability, update or insert
    let ready_at = ctx.timestamp
        + std::time::Duration::from_millis(ability.cooldown_ms);
    if let Some(existing_cd) = ctx.db.player_cooldown()
        .player_id()
        .filter(&player_id)
        .find(|cd| cd.ability_id == ability_id)
    {
        ctx.db.player_cooldown().id().update(PlayerCooldown {
            ready_at,
            ..existing_cd
        });
    } else {
        ctx.db.player_cooldown().insert(PlayerCooldown {
            id: 0,
            player_id,
            ability_id,
            ready_at,
        });
    }

    // Apply effect
    apply_damage(ctx, target_id, player_id, ability_id, ability.damage);

    log::info!(
        "use_ability: player={} ability={} target={}",
        player_id, ability_id, target_id
    );
    Ok(())
}
```

- [ ] **Step 2: Verify build**

```bash
cd /home/beek/zoneforge/server && spacetime build
```

Expected: `Build finished successfully.`

- [ ] **Step 3: Commit**

```bash
cd /home/beek/zoneforge
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add use_ability reducer with range/cooldown/mana validation"
```

---

## Task 5: Implement `respawn` and `tick_status_effects` reducers

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Step 1: Add the `respawn` reducer**

```rust
#[reducer]
pub fn respawn(ctx: &ReducerContext) -> Result<(), String> {
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or("Player not found")?;
    if !player.is_dead {
        return Err("Player is not dead".to_string());
    }

    let zone = ctx.db.zone().id().find(&player.zone_id)
        .ok_or("Zone not found")?;

    let spawn_x = zone.terrain_width as f32 / 2.0;
    let spawn_y = zone.terrain_height as f32 / 2.0;

    ctx.db.player().id().update(Player {
        health: player.max_health,
        mana: player.max_mana,
        is_dead: false,
        position_x: spawn_x,
        position_y: spawn_y,
        ..player
    });

    // Remove all active status effects for this player
    let effect_ids: Vec<u64> = ctx.db.status_effect()
        .target_id()
        .filter(&player.id)
        .map(|e| e.id)
        .collect();
    for effect_id in effect_ids {
        ctx.db.status_effect().id().delete(&effect_id);
    }

    log::info!("respawn: player={} at ({}, {})", player.id, spawn_x, spawn_y);
    Ok(())
}
```

- [ ] **Step 2: Add the `tick_status_effects` reducer**

This is called by the scheduler. It processes all active DoT effects and re-schedules itself.

```rust
#[reducer]
pub fn tick_status_effects(ctx: &ReducerContext, _arg: StatusEffectTick) {
    let now_us = ctx.timestamp
        .to_duration_since_unix_epoch()
        .unwrap_or_default()
        .as_micros();

    // Collect all current effects (avoid borrow issues while deleting)
    let all_effects: Vec<StatusEffect> = ctx.db.status_effect().iter().collect();

    for effect in all_effects {
        let expires_us = effect.expires_at
            .to_duration_since_unix_epoch()
            .unwrap_or_default()
            .as_micros();

        if expires_us <= now_us {
            // Expired — remove it
            ctx.db.status_effect().id().delete(&effect.id);
        } else if matches!(effect.effect_type, StatusEffectType::Burn | StatusEffectType::Poison) {
            // Active DoT — apply tick damage (ability_id 0 = DoT, no ability row)
            apply_damage(ctx, effect.target_id, effect.target_id, 0, effect.damage_per_tick);
        }
    }

    // Re-schedule for next tick (self-recurring pattern)
    ctx.db.status_effect_tick().insert(StatusEffectTick {
        scheduled_id: 0,
        scheduled_at: ScheduleAt::Time(
            ctx.timestamp + std::time::Duration::from_secs(1)
        ),
    });
}
```

- [ ] **Step 3: Verify build**

```bash
cd /home/beek/zoneforge/server && spacetime build
```

Expected: `Build finished successfully.`

- [ ] **Step 4: Commit**

```bash
cd /home/beek/zoneforge
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add respawn and tick_status_effects reducers"
```

---

## Task 6: Deploy server and regenerate client bindings

> This is a breaking schema change (Player table modified). All existing player rows will be deleted.

- [ ] **Step 1: Publish with `--delete-data`**

```bash
cd /home/beek/zoneforge/server
spacetime publish --server local zoneforge-server --delete-data
```

Expected output ends with: `Updated database with name: zoneforge-server`

- [ ] **Step 2: Verify `init` seeded the abilities**

```bash
spacetime sql zoneforge-server "SELECT * FROM ability"
```

Expected: 3 rows — Auto-Attack, Fireball, Heal.

- [ ] **Step 3: Smoke-test `use_ability` from CLI**

First create a player (replace `"Player"` with any name):

```bash
spacetime call --server local zoneforge-server create_player '["Player"]'
```

Query the player id:

```bash
spacetime sql zoneforge-server "SELECT id, name, health, mana, is_dead FROM player"
```

Note the player's `id` (likely 1). Then call use_ability on themselves with Heal (ability 3 = SelfCast, so target = their own id):

```bash
spacetime call --server local zoneforge-server use_ability '[3, 1]'
```

Check logs for the result:

```bash
spacetime logs zoneforge-server --num-lines 10
```

Expected: `use_ability: player=1 ability=3 target=1` (and mana deducted in player row).

- [ ] **Step 4: Regenerate C# bindings**

```bash
cd /home/beek/zoneforge/client
spacetime generate \
  --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

Expected: Files written to `Assets/Scripts/autogen/` with no errors.
New files will appear for `Ability`, `PlayerCooldown`, `StatusEffect`, `CombatLog`, `UseAbility`, `Respawn`, `TickStatusEffects`.

- [ ] **Step 5: Open Unity, run Assets → Reimport All**

Unity must reimport to pick up the new generated types. If compile errors appear in `autogen/`, they indicate a code generation issue — do not manually edit those files.

- [ ] **Step 6: Commit generated bindings**

```bash
cd /home/beek/zoneforge
git add client/Assets/Scripts/autogen/
git commit -m "chore: regenerate client bindings for combat schema"
```

---

## Task 7: Create `ZoneForgePoolManager`

**Files:**
- Create: `client/Assets/Scripts/Combat/ZoneForgePoolManager.cs`

- [ ] **Step 1: Create the directory and file**

```bash
mkdir -p /home/beek/zoneforge/client/Assets/Scripts/Combat
```

Create `client/Assets/Scripts/Combat/ZoneForgePoolManager.cs`:

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Singleton pool manager. Pre-allocates GameObjects at Awake and recycles
/// them throughout the session. Register pool definitions in the Inspector.
/// </summary>
public class ZoneForgePoolManager : MonoBehaviour
{
    public static ZoneForgePoolManager Instance { get; private set; }

    [Serializable]
    public class PoolDefinition
    {
        public string key;
        public GameObject prefab;
        public int initialCapacity = 10;
        public int maxSize = 60;
    }

    [SerializeField] private List<PoolDefinition> _poolDefinitions = new();

    private readonly Dictionary<string, Queue<GameObject>> _pools = new();
    private readonly Dictionary<string, PoolDefinition> _defs = new();

    void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);

        foreach (var def in _poolDefinitions)
        {
            _defs[def.key] = def;
            var queue = new Queue<GameObject>(def.initialCapacity);
            for (int i = 0; i < def.initialCapacity; i++)
                queue.Enqueue(CreatePooled(def));
            _pools[def.key] = queue;
            Debug.Log($"[PoolManager] Pre-allocated {def.initialCapacity}x '{def.key}'");
        }
    }

    /// <summary>Retrieve an object from the pool. Returns null if pool is empty and at max size.</summary>
    public GameObject Get(string key)
    {
        if (!_pools.TryGetValue(key, out var queue))
        {
            Debug.LogWarning($"[PoolManager] Unknown pool key: '{key}'");
            return null;
        }

        GameObject go;
        if (queue.Count > 0)
        {
            go = queue.Dequeue();
        }
        else
        {
            // Pool exhausted — grow if under max
            if (!_defs.TryGetValue(key, out var def) || CountActive(key) >= def.maxSize)
            {
                Debug.LogWarning($"[PoolManager] Pool '{key}' exhausted and at max size");
                return null;
            }
            go = CreatePooled(def);
        }

        go.SetActive(true);
        return go;
    }

    /// <summary>Return an object to the pool.</summary>
    public void Return(string key, GameObject go)
    {
        if (!_pools.TryGetValue(key, out var queue))
        {
            Destroy(go);
            return;
        }
        go.SetActive(false);
        go.transform.SetParent(transform); // re-parent to pool manager GO
        queue.Enqueue(go);
    }

    private GameObject CreatePooled(PoolDefinition def)
    {
        var go = Instantiate(def.prefab, transform);
        go.SetActive(false);
        return go;
    }

    private int CountActive(string key)
    {
        // Approximate: total created minus what's in the queue
        // Good enough for max-size guard
        return _defs[key].initialCapacity - (_pools.TryGetValue(key, out var q) ? q.Count : 0);
    }
}
```

- [ ] **Step 2: Verify file exists**

```bash
ls /home/beek/zoneforge/client/Assets/Scripts/Combat/
```

Expected: `ZoneForgePoolManager.cs`

- [ ] **Step 3: Commit**

```bash
cd /home/beek/zoneforge
git add client/Assets/Scripts/Combat/ZoneForgePoolManager.cs
git commit -m "feat(client): add ZoneForgePoolManager with pre-allocated object pools"
```

---

## Task 8: Create `PooledProjectile` and `PooledVFX`

**Files:**
- Create: `client/Assets/Scripts/Combat/PooledProjectile.cs`
- Create: `client/Assets/Scripts/Combat/PooledVFX.cs`

- [ ] **Step 1: Create `PooledProjectile.cs`**

```csharp
using System.Collections;
using UnityEngine;

/// <summary>
/// Attach to all projectile prefabs. Self-returns to pool on collision or timeout.
/// Requires a Rigidbody (gravity disabled) on the same GameObject.
/// </summary>
[RequireComponent(typeof(Rigidbody))]
public class PooledProjectile : MonoBehaviour
{
    [Tooltip("Must match the pool key registered in ZoneForgePoolManager")]
    public string poolKey = "projectile_fireball";
    public float speed = 12f;
    public float maxLifetime = 5f;

    [Tooltip("VFX pool key to spawn on impact. Leave empty to skip.")]
    public string impactVfxKey = "vfx_impact_fire";

    private Rigidbody _rb;
    private bool _hasReturned;
    private Coroutine _timeoutCoroutine;

    void Awake()
    {
        _rb = GetComponent<Rigidbody>();
        _rb.useGravity = false;
    }

    void OnEnable()
    {
        _hasReturned = false;
        _timeoutCoroutine = StartCoroutine(TimeoutReturn());
    }

    void OnDisable()
    {
        if (_timeoutCoroutine != null)
        {
            StopCoroutine(_timeoutCoroutine);
            _timeoutCoroutine = null;
        }
        _rb.linearVelocity = Vector3.zero;
        _rb.angularVelocity = Vector3.zero;
    }

    /// <summary>Aim and launch toward a world-space target position.</summary>
    public void Launch(Vector3 origin, Vector3 targetPos)
    {
        transform.position = origin;
        Vector3 dir = (targetPos - origin).normalized;
        transform.forward = dir;
        _rb.linearVelocity = dir * speed;
    }

    void OnCollisionEnter(Collision collision)
    {
        ReturnToPool(collision.contacts.Length > 0 ? collision.contacts[0].point : transform.position);
    }

    private IEnumerator TimeoutReturn()
    {
        yield return new WaitForSeconds(maxLifetime);
        ReturnToPool(transform.position);
    }

    private void ReturnToPool(Vector3 impactPoint)
    {
        if (_hasReturned) return;
        _hasReturned = true;

        // Spawn impact VFX
        if (!string.IsNullOrEmpty(impactVfxKey) && ZoneForgePoolManager.Instance != null)
        {
            var vfx = ZoneForgePoolManager.Instance.Get(impactVfxKey);
            if (vfx != null)
                vfx.transform.position = impactPoint;
        }

        if (ZoneForgePoolManager.Instance != null)
            ZoneForgePoolManager.Instance.Return(poolKey, gameObject);
        else
            gameObject.SetActive(false);
    }
}
```

- [ ] **Step 2: Create `PooledVFX.cs`**

```csharp
using System.Collections;
using UnityEngine;

/// <summary>
/// Attach to all pooled VFX prefabs. Automatically returns to pool when
/// the particle system finishes. Requires a ParticleSystem on the same GO.
/// </summary>
[RequireComponent(typeof(ParticleSystem))]
public class PooledVFX : MonoBehaviour
{
    [Tooltip("Must match the pool key registered in ZoneForgePoolManager")]
    public string poolKey = "vfx_impact_generic";

    private ParticleSystem _ps;
    private Coroutine _returnCoroutine;

    void Awake() => _ps = GetComponent<ParticleSystem>();

    void OnEnable()
    {
        _ps.Play();
        _returnCoroutine = StartCoroutine(WaitAndReturn());
    }

    void OnDisable()
    {
        if (_returnCoroutine != null)
        {
            StopCoroutine(_returnCoroutine);
            _returnCoroutine = null;
        }
        _ps.Stop(true, ParticleSystemStopBehavior.StopEmittingAndClear);
    }

    private IEnumerator WaitAndReturn()
    {
        yield return new WaitUntil(() => !_ps.IsAlive(true));
        if (ZoneForgePoolManager.Instance != null)
            ZoneForgePoolManager.Instance.Return(poolKey, gameObject);
        else
            gameObject.SetActive(false);
    }
}
```

- [ ] **Step 3: Commit**

```bash
cd /home/beek/zoneforge
git add client/Assets/Scripts/Combat/PooledProjectile.cs \
        client/Assets/Scripts/Combat/PooledVFX.cs
git commit -m "feat(client): add PooledProjectile and PooledVFX self-returning components"
```

---

## Task 9: Update `SpacetimeDBManager` with new events and subscriptions

**Files:**
- Modify: `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs`

- [ ] **Step 1: Add new static event declarations**

In `SpacetimeDBManager.cs`, find the block of static event declarations (around lines 13–21) and add after `OnTerrainChunkUpdated`:

```csharp
public static event Action<SpacetimeDB.Types.CombatLog> OnCombatLogInserted;
public static event Action<SpacetimeDB.Types.Ability> OnAbilityInserted;
public static event Action<SpacetimeDB.Types.PlayerCooldown> OnPlayerCooldownInserted;
public static event Action<SpacetimeDB.Types.PlayerCooldown, SpacetimeDB.Types.PlayerCooldown> OnPlayerCooldownUpdated;
public static event Action<SpacetimeDB.Types.StatusEffect> OnStatusEffectInserted;
public static event Action<SpacetimeDB.Types.StatusEffect> OnStatusEffectDeleted;
```

- [ ] **Step 2: Add the four new table subscriptions**

In `OnConnect`, find the `Subscribe(new[]` call and add four new entries:

```csharp
"SELECT * FROM ability",
"SELECT * FROM player_cooldown",
"SELECT * FROM status_effect",
"SELECT * FROM combat_log"
```

- [ ] **Step 3: Register callbacks in `OnSubscriptionApplied`**

In `OnSubscriptionApplied`, after the existing `Conn.Db.TerrainChunk` registrations, add:

```csharp
Conn.Db.CombatLog.OnInsert += (eventCtx, log) => OnCombatLogInserted?.Invoke(log);
Conn.Db.Ability.OnInsert += (eventCtx, ability) => OnAbilityInserted?.Invoke(ability);
Conn.Db.PlayerCooldown.OnInsert += (eventCtx, cd) => OnPlayerCooldownInserted?.Invoke(cd);
Conn.Db.PlayerCooldown.OnUpdate += (eventCtx, oldCd, newCd) => OnPlayerCooldownUpdated?.Invoke(oldCd, newCd);
Conn.Db.StatusEffect.OnInsert += (eventCtx, effect) => OnStatusEffectInserted?.Invoke(effect);
Conn.Db.StatusEffect.OnDelete += (eventCtx, effect) => OnStatusEffectDeleted?.Invoke(effect);
```

- [ ] **Step 4: Verify the file compiles in Unity**

Open Unity and check the Console for compile errors. Fix any namespace mismatches (the exact autogen type name may differ — check `Assets/Scripts/autogen/Types/` for the generated class names and adjust the `SpacetimeDB.Types.` prefix if needed).

- [ ] **Step 5: Commit**

```bash
cd /home/beek/zoneforge
git add client/Assets/Scripts/Runtime/SpacetimeDBManager.cs
git commit -m "feat(client): add CombatLog/Ability/PlayerCooldown/StatusEffect subscriptions and events"
```

---

## Task 10: Create `CombatManager`

**Files:**
- Create: `client/Assets/Scripts/Combat/CombatManager.cs`

- [ ] **Step 1: Create `CombatManager.cs`**

```csharp
using System.Collections.Generic;
using UnityEngine;
using SpacetimeDB.Types;

/// <summary>
/// Singleton. Subscribes to CombatLog inserts and spawns VFX. Also handles
/// the local player's death/respawn overlay.
/// </summary>
public class CombatManager : MonoBehaviour
{
    public static CombatManager Instance { get; private set; }

    [Header("Respawn Overlay")]
    [Tooltip("Assign a UI Canvas Text/Image that reads 'Press R to respawn'. Enable/disable in Inspector.")]
    [SerializeField] private GameObject _respawnOverlay;

    // Cache of player world positions — updated by PlayerManager via OnPlayerUpdated
    // Key: player id, Value: world position
    private readonly Dictionary<ulong, Vector3> _playerPositions = new();

    void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;

        if (_respawnOverlay != null)
            _respawnOverlay.SetActive(false);
    }

    void OnEnable()
    {
        SpacetimeDBManager.OnCombatLogInserted += OnCombatLogInserted;
        SpacetimeDBManager.OnPlayerUpdated += OnPlayerUpdated;
    }

    void OnDisable()
    {
        SpacetimeDBManager.OnCombatLogInserted -= OnCombatLogInserted;
        SpacetimeDBManager.OnPlayerUpdated -= OnPlayerUpdated;
    }

    /// <summary>Called by PlayerManager after spawning or moving a player capsule.</summary>
    public void RegisterPlayerPosition(ulong playerId, Vector3 worldPos)
    {
        _playerPositions[playerId] = worldPos;
    }

    private void OnCombatLogInserted(CombatLog log)
    {
        // Look up the ability — ability_id 0 means DoT tick (no ability row)
        Ability ability = null;
        if (log.AbilityId != 0)
        {
            foreach (var a in SpacetimeDBManager.Conn.Db.Ability.Iter())
            {
                if (a.Id == log.AbilityId) { ability = a; break; }
            }
        }

        // Determine positions
        _playerPositions.TryGetValue(log.AttackerId, out var attackerPos);
        _playerPositions.TryGetValue(log.TargetId, out var targetPos);

        if (ability != null && ability.AbilityType == AbilityType.Projectile)
        {
            // Spawn projectile from attacker toward target
            var go = ZoneForgePoolManager.Instance?.Get("projectile_fireball");
            if (go != null)
            {
                var proj = go.GetComponent<PooledProjectile>();
                if (proj != null)
                    proj.Launch(attackerPos + Vector3.up, targetPos + Vector3.up);
            }
        }
        else
        {
            // MeleeAttack, SelfCast, or DoT tick — instant impact VFX at target
            var vfx = ZoneForgePoolManager.Instance?.Get("vfx_impact_generic");
            if (vfx != null)
                vfx.transform.position = targetPos + Vector3.up;
        }
    }

    private void OnPlayerUpdated(Player oldPlayer, Player newPlayer)
    {
        // Track position for VFX targeting
        _playerPositions[newPlayer.Id] = new Vector3(newPlayer.PositionX, 1f, newPlayer.PositionY);

        // Local player death/respawn overlay
        if (newPlayer.Identity != SpacetimeDBManager.LocalIdentity) return;

        bool justDied = !oldPlayer.IsDead && newPlayer.IsDead;
        bool justRespawned = oldPlayer.IsDead && !newPlayer.IsDead;

        if (justDied && _respawnOverlay != null)
            _respawnOverlay.SetActive(true);
        else if (justRespawned && _respawnOverlay != null)
            _respawnOverlay.SetActive(false);
    }
}
```

- [ ] **Step 2: Commit**

```bash
cd /home/beek/zoneforge
git add client/Assets/Scripts/Combat/CombatManager.cs
git commit -m "feat(client): add CombatManager with VFX spawning and death overlay"
```

---

## Task 11: Create `CombatInputHandler`

**Files:**
- Create: `client/Assets/Scripts/Combat/CombatInputHandler.cs`

- [ ] **Step 1: Create `CombatInputHandler.cs`**

```csharp
using System.Collections.Generic;
using UnityEngine;
using SpacetimeDB.Types;

/// <summary>
/// Handles combat input:
///   Tab       — cycle target through other living players
///   1         — Auto-Attack selected target
///   2         — Fireball selected target
///   3         — Heal self
///   R         — Respawn (only when dead)
/// Attach to any persistent GameObject (e.g. the SpacetimeDBManager GO).
/// </summary>
public class CombatInputHandler : MonoBehaviour
{
    // Ability IDs as seeded by init reducer — must match server seed order
    private const ulong AbilityAutoAttack = 1;
    private const ulong AbilityFireball   = 2;
    private const ulong AbilityHeal       = 3;

    private ulong _selectedTargetId;   // 0 = no target
    private GameObject _selectionRing; // flat cylinder under the target

    void Start()
    {
        // Create the selection ring (flat yellow cylinder, placed under target)
        _selectionRing = GameObject.CreatePrimitive(PrimitiveType.Cylinder);
        _selectionRing.name = "SelectionRing";
        _selectionRing.transform.localScale = new Vector3(1.2f, 0.05f, 1.2f);
        var rend = _selectionRing.GetComponent<Renderer>();
        rend.material.color = Color.yellow;
        // Disable collider so the ring doesn't interfere with physics
        Destroy(_selectionRing.GetComponent<Collider>());
        _selectionRing.SetActive(false);
    }

    void Update()
    {
        if (!SpacetimeDBManager.IsSubscribed) return;

        var localPlayer = GetLocalPlayer();
        if (localPlayer == null) return;

        if (localPlayer.IsDead)
        {
            if (Input.GetKeyDown(KeyCode.R))
                SpacetimeDBManager.Conn.Reducers.Respawn();
            // Suppress all other input while dead
            return;
        }

        if (Input.GetKeyDown(KeyCode.Tab))
            CycleTarget();

        if (Input.GetKeyDown(KeyCode.Alpha1))
            TryUseAbility(AbilityAutoAttack, requireTarget: true, localPlayer);

        if (Input.GetKeyDown(KeyCode.Alpha2))
            TryUseAbility(AbilityFireball, requireTarget: true, localPlayer);

        if (Input.GetKeyDown(KeyCode.Alpha3))
            TryUseAbility(AbilityHeal, requireTarget: false, localPlayer);

        // Keep selection ring stuck to target
        UpdateSelectionRing();
    }

    private void CycleTarget()
    {
        var candidates = new List<Player>();
        foreach (var p in SpacetimeDBManager.Conn.Db.Player.Iter())
        {
            if (p.Identity == SpacetimeDBManager.LocalIdentity) continue;
            if (p.IsDead) continue;
            candidates.Add(p);
        }

        if (candidates.Count == 0)
        {
            _selectedTargetId = 0;
            _selectionRing.SetActive(false);
            return;
        }

        // Find the index of the current target in the list; advance by 1
        int currentIndex = candidates.FindIndex(p => p.Id == _selectedTargetId);
        int nextIndex = (currentIndex + 1) % candidates.Count;
        _selectedTargetId = candidates[nextIndex].Id;
        _selectionRing.SetActive(true);
        Debug.Log($"[CombatInput] Target selected: player {_selectedTargetId}");
    }

    private void TryUseAbility(ulong abilityId, bool requireTarget, Player localPlayer)
    {
        ulong targetId;
        if (requireTarget)
        {
            if (_selectedTargetId == 0)
            {
                Debug.Log("[CombatInput] No target selected");
                return;
            }
            targetId = _selectedTargetId;
        }
        else
        {
            // Self-cast — use local player's id
            targetId = localPlayer.Id;
        }

        // Client-side cooldown check (server also validates, this prevents spamming)
        if (IsOnCooldown(abilityId, localPlayer.Id))
        {
            Debug.Log($"[CombatInput] Ability {abilityId} on cooldown");
            return;
        }

        SpacetimeDBManager.Conn.Reducers.UseAbility(abilityId, targetId);
        Debug.Log($"[CombatInput] UseAbility({abilityId}, target={targetId})");
    }

    private bool IsOnCooldown(ulong abilityId, ulong playerId)
    {
        long nowMs = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
        foreach (var cd in SpacetimeDBManager.Conn.Db.PlayerCooldown.Iter())
        {
            if (cd.PlayerId != playerId || cd.AbilityId != abilityId) continue;
            // ready_at is a SpacetimeDB Timestamp — access its microseconds
            long readyUs = (long)cd.ReadyAt.MicrosecondsSinceUnixEpoch;
            long nowUs = nowMs * 1000L;
            return readyUs > nowUs;
        }
        return false;
    }

    private void UpdateSelectionRing()
    {
        if (_selectedTargetId == 0) { _selectionRing.SetActive(false); return; }

        // Find target GO via PlayerManager
        if (PlayerManager.Instance == null) return;
        var targetGo = PlayerManager.Instance.GetPlayerObject(_selectedTargetId);
        if (targetGo == null)
        {
            _selectedTargetId = 0;
            _selectionRing.SetActive(false);
            return;
        }

        var pos = targetGo.transform.position;
        _selectionRing.transform.position = new Vector3(pos.x, 0.05f, pos.z);
        _selectionRing.SetActive(true);
    }

    private Player GetLocalPlayer()
    {
        foreach (var p in SpacetimeDBManager.Conn.Db.Player.Iter())
        {
            if (p.Identity == SpacetimeDBManager.LocalIdentity)
                return p;
        }
        return null;
    }
}
```

- [ ] **Step 2: Expose `GetPlayerObject` on `PlayerManager` and wire position tracking**

`CombatInputHandler` needs `PlayerManager.Instance.GetPlayerObject(id)` and `CombatManager` needs position updates. Add to `client/Assets/Scripts/Player/PlayerManager.cs`:

**A) New public method** (e.g. after `OnNavMeshBaked`):

```csharp
/// <summary>Returns the GameObject for a player id, or null if not spawned.</summary>
public GameObject GetPlayerObject(ulong playerId) =>
    _players.TryGetValue(playerId, out var go) ? go : null;
```

**B) In `SpawnPlayer`**, after `_players[player.Id] = go;`, add:

```csharp
CombatManager.Instance?.RegisterPlayerPosition(
    player.Id,
    new Vector3(player.PositionX, 1f, player.PositionY));
```

**C) In `OnPlayerUpdated`**, after `ctrl.ReceiveServerPosition(newPlayer);`, add:

```csharp
CombatManager.Instance?.RegisterPlayerPosition(
    newPlayer.Id,
    new Vector3(newPlayer.PositionX, 1f, newPlayer.PositionY));
```

- [ ] **Step 3: Commit**

```bash
cd /home/beek/zoneforge
git add client/Assets/Scripts/Combat/CombatInputHandler.cs \
        client/Assets/Scripts/Player/PlayerManager.cs
git commit -m "feat(client): add CombatInputHandler (Tab/1/2/3/R) and PlayerManager.GetPlayerObject"
```

---

## Task 12: Scene setup, prefabs, and end-to-end test

> This task is done in the Unity Editor. No code files — only scene wiring and prefab creation.

- [ ] **Step 1: Create placeholder prefabs**

In the Unity Project window, create `Assets/Prefabs/` if it doesn't exist.

**FireballProjectile prefab:**
1. Hierarchy → Create Empty, name it `FireballProjectile`
2. Add Component → Sphere (or use a primitive: GameObject → 3D Object → Sphere, scale to 0.3)
3. Add Component → `Rigidbody` — disable gravity (`Use Gravity = false`)
4. Add Component → `PooledProjectile` — set Pool Key = `projectile_fireball`, Impact VFX Key = `vfx_impact_fire`
5. Drag from Hierarchy to `Assets/Prefabs/FireballProjectile.prefab`
6. Delete from Hierarchy

**ImpactFireVFX prefab:**
1. Hierarchy → Effects → Particle System, name it `ImpactFireVFX`
2. In Particle System, set Duration = 0.5, Stop Action = None (PooledVFX handles return)
3. Add Component → `PooledVFX` — set Pool Key = `vfx_impact_fire`
4. Drag to `Assets/Prefabs/ImpactFireVFX.prefab`, delete from Hierarchy

**ImpactGenericVFX prefab:**
1. Same as above, name `ImpactGenericVFX`, Pool Key = `vfx_impact_generic`
2. Save as `Assets/Prefabs/ImpactGenericVFX.prefab`

- [ ] **Step 2: Add `ZoneForgePoolManager` to the scene**

1. In SampleScene Hierarchy, create an empty GO named `PoolManager`
2. Add Component → `ZoneForgePoolManager`
3. In the Inspector, set Pool Definitions (size = 3):
   - Element 0: Key = `projectile_fireball`, Prefab = FireballProjectile, Initial = 20, Max = 60
   - Element 1: Key = `vfx_impact_fire`, Prefab = ImpactFireVFX, Initial = 15, Max = 50
   - Element 2: Key = `vfx_impact_generic`, Prefab = ImpactGenericVFX, Initial = 20, Max = 60

- [ ] **Step 3: Add `CombatManager` to the scene**

1. Create empty GO named `CombatManager`
2. Add Component → `CombatManager`
3. For the Respawn Overlay field: create a simple UI Canvas → Text object that reads "Press R to respawn" (or use an existing UI Canvas if one exists). Assign it to the field. The GO starts inactive.

- [ ] **Step 4: Add `CombatInputHandler` to the scene**

1. Add Component → `CombatInputHandler` to the `SpacetimeDBManager` GO (or any persistent GO)

- [ ] **Step 5: Run the end-to-end test**

Clean up stale player rows first:
```bash
spacetime sql zoneforge-server "DELETE FROM player"
```

Then:
1. Hit Play in the Unity Editor
2. Both clients connect — confirm 2 capsules appear (standalone + editor)
3. Press Tab in one client — the selection ring should appear under the other player's capsule
4. Press 1 (Auto-Attack):
   - Check Unity Console for `[CombatInput] UseAbility(1, target=X)`
   - Check `spacetime logs zoneforge-server` for `use_ability: player=... ability=1 target=...`
   - `vfx_impact_generic` should play at the target's position (melee)
5. Press 2 (Fireball):
   - A fireball sphere should launch from the attacker toward the target
   - Impact VFX should play on collision or after 5s timeout
6. Keep attacking until the target reaches 0 HP:
   - `is_dead = true` → "Press R to respawn" overlay should appear on the target client
7. Press R on the dead client:
   - Player warps to zone center, health/mana reset
   - Overlay disappears

- [ ] **Step 6: Commit final wiring**

If any scene files changed (`.unity`) or new meta files were created:

```bash
cd /home/beek/zoneforge
git add client/Assets/
git commit -m "feat(client): scene wiring for PoolManager, CombatManager, CombatInputHandler"
```

---

## Verification Checklist

Before marking Group 7 complete in `docs/design/PROGRESS.md`:

- [ ] `spacetime sql zoneforge-server "SELECT * FROM ability"` returns 3 rows
- [ ] Tab key cycles through living remote players, selection ring moves
- [ ] Key 1 deals damage (check health in `spacetime sql zoneforge-server "SELECT id, health, mana FROM player"`)
- [ ] Key 2 spawns a fireball projectile visible to both clients
- [ ] Key 3 heals the local player (mana deducted, health increases)
- [ ] Killing a player shows "Press R to respawn" overlay on their client
- [ ] R key respawns the dead player at zone center with full health/mana
- [ ] `spacetime logs` shows no unexpected errors during combat

---

## Troubleshooting

**`UseAbility` returns "Ability not found"**
The `init` reducer only seeds on first publish. If you published without `--delete-data` and abilities existed previously, the guard passes. Query: `spacetime sql zoneforge-server "SELECT * FROM ability"` — if empty, the `--delete-data` publish didn't trigger `init`. Re-publish with `--delete-data`.

**`CombatInputHandler` can't read `cd.ReadyAt.MicrosecondsSinceUnixEpoch`**
The exact property name depends on the autogen version. Check the generated `PlayerCooldown.g.cs` file in `Assets/Scripts/autogen/Types/` for the actual `ReadyAt` property type and accessor. Adjust `IsOnCooldown` accordingly.

**Fireball launches but goes the wrong direction**
`PooledProjectile.Launch` takes `targetPos + Vector3.up` — confirm attacker and target positions are in `CombatManager._playerPositions`. If the dictionary is empty, `CombatManager.RegisterPlayerPosition` may not be called. Add a call from `PlayerManager.SpawnPlayer` and `PlayerManager.OnPlayerUpdated`.

**"Press R to respawn" overlay never appears**
Verify `CombatManager._respawnOverlay` is assigned in the Inspector. Also confirm `OnPlayerUpdated` is firing — add a `Debug.Log` to the `justDied` branch temporarily.
