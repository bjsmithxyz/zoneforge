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

## Stack

- **Client**: Unity 6 (URP), C#, SpacetimeDB C# SDK
- **Server**: Rust, SpacetimeDB 1.0

## Submodule Workflow

When you push changes to `client` or `server`, update the parent repo to track the new commits:

```bash
cd /home/beek/repos/ZoneForge
git add client server
git commit -m "Update submodules"
git push
```
