# Claude Skills — ZoneForge

Claude Code skills are project-specific instruction sets that Claude loads automatically when you describe a relevant task. They encode the correct patterns, common mistakes, and step-by-step workflows for this project so Claude doesn't have to rediscover them each session.

## How skills work

Skills live as `SKILL.md` files in `.claude/skills/<skill-name>/` directories. Claude reads the skill's description to decide when to apply it, then follows the instructions inside for that workflow.

You don't need to invoke skills by name — Claude recognises when a task matches a skill and uses it automatically. If you want to invoke one explicitly, say e.g. "use the zoneforge-deploy skill".

## Skill locations

Skills are scoped to the part of the project they apply to:

| Scope | Location |
|-------|----------|
| Umbrella repo (full-stack workflows) | `zoneforge/.claude/skills/` |
| Server submodule | `zoneforge/server/.claude/skills/` |
| Client submodule | `zoneforge/client/.claude/skills/` |

---

## Skills reference

### Umbrella repo

#### `zoneforge-deploy`
**Location:** `.claude/skills/zoneforge-deploy/SKILL.md`

Full server pipeline: build → publish → regenerate bindings. Handles the decision about whether `--delete-data` is needed for breaking schema changes and ensures the Unity client is prompted to Reimport All when bindings change.

**Triggers on:** "deploy", "publish", "rebuild server", "push changes", finishing edits to `lib.rs`

---

#### `zoneforge-new-feature`
**Location:** `.claude/skills/zoneforge-new-feature/SKILL.md`

Guides implementing any feature that spans server and client. Walks through the full 8-step sequence: define table → define reducer → build/publish → regenerate bindings → subscribe in Unity → register callbacks → call reducer from UI → render data.

The most important skill in the project — it prevents the #1 mistake in SpacetimeDB development: implementing the server side but forgetting to wire up the client.

**Triggers on:** "add feature", "implement X", "I want players to be able to X", "create a system for X", "store X"

---

#### `zoneforge-debug`

**Location:** `.claude/skills/zoneforge-debug/SKILL.md`

Systematic triage for debugging issues across the full stack. Works through four layers in order — SpacetimeDB process → Rust module → client bindings → Unity wiring — with specific checks at each layer and a symptom-to-cause lookup table. Uses `spacetime call` and `spacetime sql` to isolate server behaviour from the Unity client.

**Triggers on:** "something's broken", "not working", "can't connect", "callbacks not firing", "no data", "reducer not running", "debugging"

---

### Server submodule (`server/`)

#### `spacetimedb-rust-table`
**Location:** `server/.claude/skills/spacetimedb-rust-table/SKILL.md`

Correct patterns for defining SpacetimeDB tables in Rust. Covers the `#[table(...)]` macro, visibility (`public`), column attributes (`#[primary_key]`, `#[auto_inc]`, `#[unique]`, `#[index(btree)]`), custom embedded types (`SpacetimeType`), index naming rules, and the `Table` trait import.

Prevents the most common LLM mistakes: adding `#[derive(SpacetimeType)]` to table structs, forgetting `use spacetimedb::Table`, and duplicate index names.

**Triggers on:** "add table", "define schema", "create table", "store X on the server", "new entity"

---

#### `spacetimedb-rust-reducer`
**Location:** `server/.claude/skills/spacetimedb-rust-reducer/SKILL.md`

Correct patterns for implementing SpacetimeDB reducers in Rust. Covers the function signature (`&ReducerContext`, `Result<(), String>`), the update-via-spread pattern, lifecycle hooks (`init`, `client_connected`, `client_disconnected`), scheduled reducers, logging, and determinism rules.

Prevents: mutable context, returning data, using `rand`/`SystemTime`, panicking instead of returning `Err`, and the borrow-after-move pitfall.

**Triggers on:** "add reducer", "server action", "handle player event", "implement game mechanic on server"

---

### Client submodule (`client/`)

#### `unity-spacetimedb-subscribe`
**Location:** `client/.claude/skills/unity-spacetimedb-subscribe/SKILL.md`

Correct C# patterns for subscribing to SpacetimeDB tables and registering row callbacks in Unity. Enforces the mandatory ordering: `DbConnection.Builder()` → subscribe inside `OnConnect` → register callbacks inside `OnSubscriptionApplied` → `FrameTick()` every `Update()`.

Prevents the most common Unity/SpacetimeDB issues: missing `FrameTick()`, subscribing too early, and registering callbacks before the subscription is applied.

**Triggers on:** "subscribe to table", "listen for updates", "callbacks not firing", "react to server data", "wire up callbacks"

---

#### `unity-autogen-refresh`
**Location:** `client/.claude/skills/unity-autogen-refresh/SKILL.md`

Regenerates the C# client bindings in `Assets/Scripts/autogen/` after a server schema change, and prompts the Unity Reimport All step. Includes the prerequisite chain (build → publish → generate) and the rule never to edit autogen files manually.

**Triggers on:** "regenerate bindings", "schema changed", "autogen out of date", "spacetime generate", Unity errors after server changes

---

## Adding new skills

1. Create `.claude/skills/<skill-name>/SKILL.md` in the appropriate scope directory
2. Add YAML frontmatter with `name` and `description` fields
3. Write the skill body following the patterns in existing skills
4. Add an entry to this document

Skills use markdown. Keep them under ~500 lines; split into reference files if they grow larger. See the [skill-creator plugin](https://github.com/anthropics/claude-code) for tooling to test and iterate on skills.
