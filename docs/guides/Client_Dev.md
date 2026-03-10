# ZoneForge — Unity Client Development Guide

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
# Unity Hub → Projects → ZoneForge (client/)
```

In a second terminal when server code has changed:

```bash
cd ~/Projects/ZoneForge/server
spacetime build
spacetime publish --server local zoneforge-server
# Regenerate bindings if schema changed:
cd ../client
spacetime generate --lang csharp --out-dir Assets/Scripts/autogen \
  --bin-path ../server/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

## Testing in Unity

- **Play mode** (`Ctrl+P`) — connects to local SpacetimeDB, runs full game loop
- **Console** (`Window → General → Console`) — SpacetimeDB connection logs appear here
- **Scene view** — use editor tools (ZoneForge → Map Editor) outside play mode

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
  --bin-path ../server/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

Wait for Unity to reimport (`Assets → Reimport All` if auto-import doesn't trigger).

## Editor Tool Development

Editor-only scripts go in `Assets/Scripts/Editor/`. They are excluded from game builds automatically. Use `[MenuItem("ZoneForge/...")]` to add entries to the Unity menu bar.

## Common Issues

| Problem | Fix |
|---------|-----|
| "Connected!" not appearing in Console | Check `spacetime start` is running; verify module name is `zoneforge-server` |
| Autogen types unresolved after schema change | Run `spacetime generate`, then `Assets → Reimport All` |
| rust-analyzer errors in server code unrelated to client | Open server in a separate VS Code window |
| Editor window missing from menu | Unity didn't recompile — check Console for C# errors |

## See Also

- [../architecture/Client.md](../architecture/Client.md) — Client architecture detail
- [Server_Dev.md](Server_Dev.md) — Server-side development workflow
- [Getting_Started.md](Getting_Started.md) — Full environment setup
