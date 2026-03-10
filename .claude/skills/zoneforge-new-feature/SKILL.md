---
name: zoneforge-new-feature
description: Guide for implementing any feature that spans the ZoneForge server and Unity client. Use this skill whenever the user wants to add a new game mechanic, system, entity, or piece of data — even if they only describe the client-side or server-side half of it. Trigger on phrases like "add X", "implement X", "I want players to be able to X", "create a system for X", "store X in the database". The most common mistake in SpacetimeDB projects is building the backend without wiring up the client — this skill prevents that.
---

## Why this skill exists

SpacetimeDB features span two codebases: the Rust server module and the Unity C# client. It's easy to implement tables and reducers on the server, then forget to subscribe to the data or call the reducers from Unity. This skill walks through both sides every time.

## Step 0: Clarify scope before writing any code

Ask (or infer from context):
- What data needs to be stored? (→ determines tables)
- What actions can players/the server take? (→ determines reducers)
- Who can see the data — all clients, or just the owner? (→ `public` vs private table)
- Does this replace or extend an existing table? (→ affects whether `--delete-data` is needed)

## Step 1: Define the table(s) on the server

In `server/src/lib.rs` (or a new module file if appropriate):

```rust
use spacetimedb::{table, Table, ReducerContext, Identity};

#[table(accessor = zone_object, public)]
pub struct ZoneObject {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub owner: Identity,
    pub name: String,
    pub x: i32,
    pub y: i32,
}
```

Rules:
- Use `#[table(accessor = ..., public)]` — clients can't subscribe to private tables
- **Never** add `#[derive(SpacetimeType)]` to a table struct — the macro handles serialisation
- Column names become the index accessor names — keep them short and clear
- Index names must be unique across the entire module

Use `#[derive(SpacetimeType)]` only for embedded custom types (structs/enums used as field types, not as tables).

## Step 2: Define the reducer(s) on the server

```rust
use spacetimedb::{reducer, ReducerContext, Table};

#[reducer]
pub fn place_object(ctx: &ReducerContext, name: String, x: i32, y: i32) -> Result<(), String> {
    if name.is_empty() {
        return Err("Name cannot be empty".to_string());
    }
    ctx.db.zone_object().insert(ZoneObject {
        id: 0,  // auto-inc placeholder
        owner: ctx.sender,
        name,
        x,
        y,
    });
    Ok(())
}
```

Rules:
- Context is always `&ReducerContext`, never `&mut`
- Reducers must be deterministic — no filesystem, network calls, `rand` crate, or `SystemTime::now()`
- Use `ctx.rng` for randomness, `ctx.timestamp` for time
- Return `Result<(), String>` for fallible reducers so errors surface cleanly (don't panic)
- Use `ctx.sender` for the caller's identity — never trust an identity passed as a parameter

**Update pattern** — always spread the existing row:
```rust
if let Some(obj) = ctx.db.zone_object().id().find(&object_id) {
    ctx.db.zone_object().id().update(ZoneObject { name: new_name, ..obj });
}
```

## Step 3: Build and publish

```bash
cd server && spacetime build
spacetime publish --server local zoneforge-server
# If tables changed shape: spacetime publish --server local zoneforge-server --delete-data
```

Use the `zoneforge-deploy` skill for the full deploy sequence.

## Step 4: Regenerate client bindings

This is required whenever tables or reducers change:

```bash
cd client && spacetime generate \
  --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

Then in Unity: **Assets → Reimport All**.

Never edit files in `Assets/Scripts/autogen/`.

## Step 5: Subscribe to the table in Unity

In `SpacetimeDBManager` (or a relevant manager script), subscribe inside the `OnConnect` callback — not in `Start()` or `Awake()`:

```csharp
void OnConnect(DbConnection conn, Identity identity, string token)
{
    conn.SubscriptionBuilder()
        .OnApplied(OnSubscriptionApplied)
        .Subscribe(new[] {
            "SELECT * FROM player",
            "SELECT * FROM zone_object"   // ← add your new table here
        });
}
```

## Step 6: Register callbacks in OnSubscriptionApplied

Register row event callbacks only after the subscription is applied — not before:

```csharp
void OnSubscriptionApplied(SubscriptionEventContext ctx)
{
    ctx.Db.ZoneObject.OnInsert += OnZoneObjectInserted;
    ctx.Db.ZoneObject.OnUpdate += OnZoneObjectUpdated;
    ctx.Db.ZoneObject.OnDelete += OnZoneObjectDeleted;
}

void OnZoneObjectInserted(EventContext ctx, ZoneObject obj)
{
    // Spawn or update the object in the scene
}
```

## Step 7: Call the reducer from UI / game logic

This is the most commonly forgotten step. Every reducer on the server must be explicitly called from the client or it will never run.

```csharp
// Generated reducer classes live in Assets/Scripts/autogen/
Reducer.PlaceObject(conn, objectName, x, y);
```

Wire this up to a button, keyboard shortcut, or game event as appropriate.

## Step 8: Render / react to the data

Read data from the local cache (populated by subscriptions) rather than making explicit requests:

```csharp
foreach (var obj in conn.Db.ZoneObject.Iter())
{
    // obj is a ZoneObject row — render it
}
```

## Checklist

- [ ] Table(s) defined in server with correct `#[table(...)]` syntax
- [ ] No `#[derive(SpacetimeType)]` on table structs
- [ ] Reducer(s) defined, deterministic, returning `Result<(), String>`
- [ ] Server built and published
- [ ] Bindings regenerated (`spacetime generate`)
- [ ] Unity Reimport All
- [ ] Client subscribes to new table(s) inside `OnConnect`
- [ ] Callbacks registered inside `OnSubscriptionApplied`
- [ ] Reducer called from client UI/game logic ← the most commonly missed step
- [ ] Data rendered from subscription cache
