# ZoneForge

A tile-based multiplayer world builder. Players collaboratively build and explore persistent zones powered by a real-time database backend.

## Projects

| Repo | Description |
|------|-------------|
| [zoneforge-client](https://github.com/bjsmithxyz/zoneforge-client) | Unity game client (C#, URP, SpacetimeDB SDK) |
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
| [Roadmap](docs/design/Roadmap.md) | Phases, sprints, and milestones |
| [docs/](docs/) | All documentation |

## Stack

- **Client**: Unity 2022.3 LTS (URP), C#, SpacetimeDB C# SDK
- **Server**: Rust, SpacetimeDB 1.x (resolved: 1.12.0)

## Submodule Workflow

When you push changes to `client` or `server`, update the parent repo to track the new commits:

```bash
cd /path/to/zoneforge
git add client server
git commit -m "Update submodules"
git push
```
