# ADR 001 — Use SpacetimeDB as the backend

**Date:** March 2026
**Status:** Accepted

---

## Context

ZoneForge needs a multiplayer backend capable of:

- Real-time state synchronization across all connected clients (editor and game)
- Server-authoritative game logic (preventing cheating)
- Persistent world storage
- Simple deployment — the project is solo/small-team with no dedicated infrastructure engineer
- High concurrency (target: 100+ players per zone)

The alternatives evaluated were:

| Option | Approach | Rejected reason |
|--------|----------|----------------|
| REST API + PostgreSQL | Traditional web server + relational DB | Complex infrastructure (server, DB, cache, load balancer), no built-in real-time sync, requires custom netcode |
| Firebase / Supabase | BaaS with real-time subscriptions | Good for CRUD apps, not designed for authoritative game logic; limited transaction guarantees; vendor lock-in with opaque pricing |
| Custom game server (Nakama, Photon) | Dedicated multiplayer framework | Adds a separate runtime and SDK to manage; Nakama uses Go/Lua for game logic rather than a language we were already using |
| Raw WebSockets + custom server | Full control | Requires implementing sync, persistence, and state management from scratch; high maintenance burden |

---

## Decision

Use **SpacetimeDB** with a Rust WASM module as the authoritative server.

---

## Reasons

**1. Zero infrastructure to manage**
SpacetimeDB replaces the web server, database, cache, and WebSocket layer with a single binary. Deploy the entire backend with one command: `spacetime publish`.

**2. Automatic real-time sync**
Clients subscribe to SQL queries (`SELECT * FROM player`) and receive live diffs when any row changes. No polling, no manual broadcast logic, no custom netcode required.

**3. Authoritative by design**
All state mutations go through Rust reducers that run on the server. Clients cannot modify game state directly — they call reducers, which validate and apply changes.

**4. In-memory performance**
All active state is in memory; disk is only a commit log. SpacetimeDB benchmarks at 100–1000x throughput compared to DB + API stacks, comfortably supporting 100+ concurrent players per zone.

**5. Rust alignment**
Server logic is written in Rust, which provides memory safety and performance without a garbage-collected runtime. The Rust ecosystem and tooling (cargo, clippy, rustfmt) are mature.

**6. Production precedent**
SpacetimeDB powers BitCraft Online by Clockwork Labs. It is battle-tested in a production MMORPG, not just a framework in early alpha.

**7. Simplified mental model**
Tables are the data model. Reducers are the API. Clients subscribe to tables. This replaces REST endpoints, ORM models, and WebSocket event handlers with three concepts.

---

## Trade-offs & Risks

| Risk | Mitigation |
|------|-----------|
| SpacetimeDB is newer than Postgres/Redis — smaller community | Official Discord is active; Clockwork Labs team is responsive; docs are improving |
| Schema migrations require `--delete-data` during early dev | Acceptable pre-launch; SpacetimeDB is adding migration support |
| Rust learning curve for contributors unfamiliar with the language | Server logic is isolated in `server/src/lib.rs`; contributors can focus on Unity client without touching Rust |
| SpacetimeDB Cloud pricing at scale | Self-hosted option available; cloud pricing is usage-based |

---

## Consequences

- Server module lives in `server/` (Rust WASM), client in `client/` (Unity C#)
- All game-critical logic is in Rust reducers — no gameplay code in Unity
- Client bindings are generated (`spacetime generate`) and must be regenerated after schema changes
- Local development requires SpacetimeDB CLI (`spacetime start`) running alongside Unity
