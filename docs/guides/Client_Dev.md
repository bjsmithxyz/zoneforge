# ZoneForge — Unity Game Client Development Guide

> This guide covers the **game client** (`client/` submodule). For world-building (zone creation, tile painting, entity placement) see [Editor_Dev.md](Editor_Dev.md).

## Prerequisites

- Unity 2022.3 LTS installed via Unity Hub
- VS Code with C# and Unity extensions
- SpacetimeDB local server running (`spacetime start`)
- See [Getting_Started.md](Getting_Started.md) for full setup

## Daily Workflow

```bash
# 1. Start the database server (keep this terminal open)
spacetime start

# 2. Open the Unity project
# Unity Hub → Projects → ZoneForge Client (client/)
```

In a second terminal when server code has changed:

```bash
cd server && spacetime build && spacetime publish --server local zoneforge-server
# Regenerate bindings if schema changed:
cd ../client
spacetime generate --lang csharp --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

## Testing in Unity

- **Play mode** (`Ctrl+P`) — connects to local SpacetimeDB, runs full game loop
- **Console** (`Window → General → Console`) — SpacetimeDB connection logs appear here
- **Scene view** — used for level/scene inspection; world editing is done in the standalone editor app

## VS Code Setup for C#

Unity auto-generates `.vscode/settings.json`, `.vscode/launch.json` when you run **Assets → Open C# Project**. Do not overwrite these files. Add only inside `settings.json`:

```json
"editor.formatOnSave": true,
"[csharp]": {
  "editor.defaultFormatter": "ms-dotnettools.csharp"
}
```

## Regenerating Client Bindings

Regenerate after any server schema change (new table, changed field, new reducer):

```bash
# From client/ directory
spacetime generate --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

Wait for Unity to reimport (`Assets → Reimport All` if auto-import doesn't trigger).

## Common Issues

| Problem | Fix |
|---------|-----|
| "Connected!" not appearing in Console | Check `spacetime start` is running; verify module name is `zoneforge-server` |
| Autogen types unresolved after schema change | Run `spacetime generate`, then `Assets → Reimport All` |
| rust-analyzer errors in server code unrelated to client | Open server in a separate VS Code window |

## Claude Code Skills

When working in `client/` with Claude Code, two skills apply automatically:

- **`unity-spacetimedb-subscribe`** — correct subscription ordering, callback registration, and `FrameTick` usage
- **`unity-autogen-refresh`** — regenerating C# bindings after server schema changes

For the full deploy pipeline Claude uses **`zoneforge-deploy`**.
For debugging when things aren't working, Claude uses **`zoneforge-debug`**.

See [Claude_Skills.md](Claude_Skills.md) for the full skills reference.

## See Also

- [Editor_Dev.md](Editor_Dev.md) — Standalone world editor development workflow
- [../architecture/Client.md](../architecture/Client.md) — Client architecture detail
- [Server_Dev.md](Server_Dev.md) — Server-side development workflow
- [Getting_Started.md](Getting_Started.md) — Full environment setup
