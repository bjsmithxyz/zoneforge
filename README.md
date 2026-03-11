# ZoneForge

A 3D multiplayer RPG world builder. Designers create persistent zones using a standalone editor app; players explore them in the game client — both powered by a real-time SpacetimeDB backend.

## Projects

| Repo | Description |
|------|-------------|
| [zoneforge-client](https://github.com/bjsmithxyz/zoneforge-client) | Unity 3D URP game client (C#, SpacetimeDB SDK) |
| [zoneforge-editor](https://github.com/bjsmithxyz/zoneforge-editor) | Unity 3D URP standalone world editor (C#, SpacetimeDB SDK) |
| [zoneforge-server](https://github.com/bjsmithxyz/zoneforge-server) | SpacetimeDB Rust server module |

## Getting Started

Clone with submodules:

```bash
git clone --recurse-submodules https://github.com/bjsmithxyz/zoneforge
```

Or if already cloned:

```bash
git submodule update --init --recursive
```

## Documentation

| Doc | Description |
| --- | ----------- |
| [Architecture Overview](docs/architecture/Overview.md) | System diagram, data flow, repo layout |
| [Getting Started](docs/guides/Getting_Started.md) | Full environment setup (Unity, Rust, SpacetimeDB) |
| [Design Document](docs/design/Detailed_Design.md) | Full game design and systems specification |
| [Progress](docs/design/PROGRESS.md) | Phases, milestones, and current task checklist |
| [docs/](docs/) | All documentation |

## Stack

- **Client & Editor**: Unity 2022.3 LTS (3D URP), C#, SpacetimeDB C# SDK
- **Server**: Rust (WASM), SpacetimeDB 2.x

## Submodule Workflow

When you push changes to a submodule, update the umbrella repo to track the new commits:

```bash
cd /path/to/zoneforge
git add client editor server
git commit -m "Update submodules"
git push
```
