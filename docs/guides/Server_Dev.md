# ZoneForge — Server Development Guide

## Prerequisites

- Rust toolchain (`rustc` 1.70+, `cargo`)
- `wasm32-unknown-unknown` target installed
- SpacetimeDB CLI (`spacetime` command)
- VS Code with rust-analyzer
- See [Getting_Started.md](Getting_Started.md) for full setup

## Daily Workflow

```bash
# Terminal 1 — keep the server running
spacetime start

# Terminal 2 — server module development
cd ~/Projects/ZoneForge/server

# Edit src/lib.rs
# Then build and republish:
spacetime build
spacetime publish --server local zoneforge-server

# If schema changed incompatibly (table added/removed/renamed):
spacetime publish --server local zoneforge-server --delete-data
```

> `--delete-data` wipes all table data and cannot be undone. Use only for breaking schema changes during development.

## VS Code Setup

Create `server/.vscode/settings.json`:

```json
{
  "rust-analyzer.linkedProjects": ["./Cargo.toml"],
  "rust-analyzer.check.command": "clippy",
  "rust-analyzer.cargo.features": "all",
  "editor.formatOnSave": true,
  "[rust]": {
    "editor.defaultFormatter": "rust-lang.rust-analyzer"
  }
}
```

rust-analyzer status appears in the bottom status bar. Hover over any type in `lib.rs` to confirm IntelliSense is active.

## Adding a Table

```rust
#[table(accessor = my_table, public)]
pub struct MyTable {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub some_field: String,
}
```

- Use `#[table(...)]` — do **not** add `#[derive(SpacetimeType)]` to table structs
- Use `#[derive(SpacetimeType)]` only for custom types used as *fields within* table rows (enums, nested structs)

## Adding a Reducer

```rust
#[reducer]
pub fn my_action(ctx: &ReducerContext, arg: String) {
    ctx.db.my_table().insert(MyTable {
        id: 0,  // auto_inc fills this
        some_field: arg,
    });
}
```

- Look up by unique field: `ctx.db.player().identity().find(ctx.sender())`
- Update a row: `ctx.db.player().id().update(Player { ..old, field: new_val })`

## Testing with the CLI

```bash
# Call a reducer
spacetime call --server local zoneforge-server create_zone "Village" 64 64

# Query a table
spacetime sql --server local zoneforge-server "SELECT * FROM zone"
spacetime sql --server local zoneforge-server "SELECT * FROM player"
```

## Linting & Formatting

```bash
cargo clippy   # linter (also runs on save if rust-analyzer.check.command = "clippy")
cargo fmt      # formatter (runs on save with editor.formatOnSave = true)
```

## wasm-opt Warning

The build may print: `Could not find wasm-opt to optimise the module.`

This is non-critical. The module builds and runs correctly without it. For production, install `wasm-opt` from [github.com/WebAssembly/binaryen/releases](https://github.com/WebAssembly/binaryen/releases).

## After Schema Changes

1. Run `spacetime build`
2. Run `spacetime publish --server local zoneforge-server --delete-data` (if breaking)
3. Regenerate Unity client bindings:

```bash
# Run from client/ or editor/ directory
spacetime generate --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

## Common Issues

| Problem | Fix |
|---------|-----|
| `spacetime: command not found` | `export PATH="$HOME/.spacetime/bin:$PATH"` |
| Rust linker error | `sudo apt install build-essential pkg-config libssl-dev` |
| `cargo build` produces native binary, not WASM | Use `spacetime build` instead |
| Port 3000 in use | `sudo lsof -i :3000` → kill or use `spacetime start --listen-addr 127.0.0.1:3001` |
| rust-analyzer not activating | Check `rust-analyzer.linkedProjects` points to `./Cargo.toml` |

## Claude Code Skills

When working in `server/` with Claude Code, two skills apply automatically:

- **`spacetimedb-rust-table`** — correct table definition syntax, attributes, and common mistakes
- **`spacetimedb-rust-reducer`** — correct reducer signatures, update patterns, and determinism rules

For the full deploy pipeline (build → publish → bindings) Claude uses **`zoneforge-deploy`**.
For debugging connection or data issues, Claude uses **`zoneforge-debug`**.

See [Claude_Skills.md](Claude_Skills.md) for the full skills reference.

## See Also

- [../architecture/Server.md](../architecture/Server.md) — Module architecture, table list, conventions
- [Client_Dev.md](Client_Dev.md) — Unity client development workflow
- [Getting_Started.md](Getting_Started.md) — Full environment setup
