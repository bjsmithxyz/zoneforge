---
name: zoneforge-debug
description: Systematic triage for debugging ZoneForge issues across the full stack — SpacetimeDB server process, Rust server module, client bindings, and Unity client. Use this skill whenever something isn't working and the cause isn't immediately obvious: Unity can't connect, callbacks aren't firing, data isn't appearing, server errors, reducer not running, or anything behaving unexpectedly. This is a three-layer stack (SpacetimeDB process → Rust module → Unity client) and failures at any layer look similar from the outside — work through this triage top-to-bottom before drawing conclusions.
---

## Triage order

Diagnose from infrastructure up. Jumping to Unity first when the server isn't running wastes time.

```
Layer 1: SpacetimeDB process running?
Layer 2: Module built and published?
Layer 3: Client bindings current?
Layer 4: Unity wired correctly?
Layer 5: Runtime errors in server logs?
```

---

## Layer 1 — Is SpacetimeDB running?

```bash
spacetime server list
```

If the local server isn't listed as running, start it:

```bash
spacetime start
```

**Signs this is the problem:** Unity Console shows a connection error immediately on Play, or `spacetime logs` fails with a connection refused message.

---

## Layer 2 — Is the module built and published?

```bash
# Check the binary exists and is recent
ls -lh server/target/wasm32-unknown-unknown/release/zoneforge_server.wasm

# Rebuild and republish
cd server && spacetime build
spacetime publish --server local zoneforge-server
```

If schema changed incompatibly (table/column removed or renamed), the old data may be incompatible:

```bash
spacetime publish --server local zoneforge-server --delete-data
```

**Signs this is the problem:** Unity connects but gets no data, or server logs show schema mismatch errors. The WASM binary exists but predates recent source changes.

---

## Layer 3 — Are client bindings current?

Stale bindings are the most common cause of Unity compile errors after server work.

```bash
# Regenerate from the compiled WASM
cd client && spacetime generate \
  --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

Then in Unity: **Assets → Reimport All**.

**Signs this is the problem:** Unity shows compile errors referencing types in `autogen/`, or a table/reducer you just added on the server doesn't exist as a C# class.

Never edit files in `Assets/Scripts/autogen/` — any manual changes are silently overwritten on next generate.

---

## Layer 4 — Is Unity wired correctly?

Work through these checks in order:

### 4a. Is FrameTick called every frame?

```csharp
void Update()
{
    _conn?.FrameTick();  // Without this, NO callbacks ever fire
}
```

This is the single most common cause of "my callbacks never fire". Missing `FrameTick()` = a completely silent failure.

### 4b. Is Subscribe called inside OnConnect — not before?

```csharp
// ✅ Correct
void OnConnect(DbConnection conn, Identity identity, string token)
{
    conn.SubscriptionBuilder()
        .OnApplied(OnSubscriptionApplied)
        .Subscribe(new[] { "SELECT * FROM player", "SELECT * FROM zone" });
}

// ❌ Wrong — subscribing in Start() before connection is established
void Start()
{
    _conn.SubscriptionBuilder().Subscribe(...);  // Connection not open yet
}
```

### 4c. Are callbacks registered inside OnSubscriptionApplied — not before?

```csharp
// ✅ Correct
void OnSubscriptionApplied(SubscriptionEventContext ctx)
{
    ctx.Db.Player.OnInsert += OnPlayerInserted;
}

// ❌ Wrong — registering before subscription is applied
void OnConnect(DbConnection conn, ...)
{
    conn.Db.Player.OnInsert += OnPlayerInserted;  // Too early
}
```

### 4d. Is the new table in the Subscribe query list?

If you added a table on the server, you must also add it to the `Subscribe(...)` call:

```csharp
.Subscribe(new[]
{
    "SELECT * FROM player",
    "SELECT * FROM zone",
    "SELECT * FROM new_table",  // ← add new tables here
})
```

### 4e. Is the table marked `public` on the server?

A private table produces no error — clients just never receive its rows. Check the Rust definition:

```rust
#[table(accessor = my_table, public)]  // ← must have public
pub struct MyTable { ... }
```

### 4f. Is the reducer actually being called?

Add a log to verify:

```csharp
Debug.Log("Calling reducer PlaceZone");
Reducer.PlaceZone(_conn, name, x, y);
```

On the server side:

```rust
log::info!("place_zone called by {:?}", ctx.sender);
```

Then check `spacetime logs zoneforge-server` to confirm it's reaching the server.

---

## Layer 5 — Check server logs

```bash
spacetime logs zoneforge-server
spacetime logs zoneforge-server --num-lines 50  # More history
```

Look for:
- `ERROR` lines — reducer failures, panics, schema issues
- Your `log::info!` / `log::debug!` output from reducers
- Absence of expected log lines (reducer not being called at all)

The `UNSTABLE API` warning that appears is expected — not an error.

---

## Common symptom → cause map

| Symptom | Most likely cause |
|---------|------------------|
| Immediate connection error in Unity | SpacetimeDB not running (`spacetime start`) |
| Connects but no data | Table not `public`, not subscribed, or bindings stale |
| Callbacks never fire | Missing `FrameTick()` in `Update()` |
| C# compile errors in `autogen/` | Schema changed — run `spacetime generate` + Reimport All |
| Reducer does nothing | Not called from client, or subscribing before `OnConnect` |
| Data visible in `spacetime sql` but not in Unity | Table not in `Subscribe(...)` query list |
| Server error in logs | Reducer panicking — return `Err(...)` instead of panicking |
| `spacetime build` succeeds but publish fails | Check logs for schema incompatibility — may need `--delete-data` |
| Callback fires but with stale data | Spreading `..Default::default()` instead of `..existing_row` in update |

---

## Quick verification commands

```bash
# Is the server running and the module published?
spacetime server list
spacetime logs zoneforge-server

# Call a reducer directly (bypasses Unity entirely)
spacetime call --server local zoneforge-server create_zone "TestZone" 64 64

# Query a table directly (bypasses subscriptions entirely)
spacetime sql --server local zoneforge-server "SELECT * FROM zone"
```

Using `spacetime call` and `spacetime sql` isolates server-side behaviour from Unity — if the data appears in `spacetime sql` but not in Unity, the problem is in the client layer.
