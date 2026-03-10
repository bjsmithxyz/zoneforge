# ADR 002 — Umbrella repo with submodules for client and server

**Date:** March 2026
**Status:** Accepted

---

## Context

ZoneForge has two distinct codebases that have very different toolchains, contributors, and release cadences:

- **Client** — Unity 2022.3 LTS project, C#, large binary assets (textures, meshes, audio) tracked via Git LFS
- **Server** — Rust WASM module, no binary assets, pure source code

A structural decision was needed before the first commit: how to organise these into repositories.

The options considered were:

| Option | Description |
|--------|-------------|
| Single monorepo | One repository containing both `client/` and `server/` directly |
| Two separate repos | `zoneforge-client` and `zoneforge-server` as independent repositories |
| Umbrella repo with submodules | Parent repo pins both submodules; each submodule is its own repo |

---

## Decision

Use an **umbrella repo** (`zoneforge`) with `client/` and `zoneforge-server` as git submodules.

---

## Reasons

**1. Toolchain isolation**
Unity generates a large volume of auto-generated files (`.meta`, `.csproj`, `.sln`, build artefacts). Rust generates its own `target/` directory. Keeping them in separate repos means each has its own `.gitignore`, CI configuration, and LFS policy without interfering with each other.

**2. Independent history and release**
Server and client can be tagged and released independently. A server-only hotfix does not require a Unity client rebuild. A client asset update does not require server republishing.

**3. Contributor scoping**
A future contributor working only on Unity client code does not need to clone the Rust server, and vice versa. Submodules allow selective cloning.

**4. Monorepo rejected**
A monorepo would work at small scale but mixes two completely different toolchains in one git history, complicates `.gitignore` and LFS tracking, and makes CI harder to scope (every push would trigger both Unity and Rust pipelines).

**5. Two separate repos (no umbrella) rejected**
Without an umbrella, there is no single place to pin compatible client+server versions together. The umbrella repo's submodule refs act as a compatibility ledger — a specific commit in `zoneforge` guarantees a known-working client+server pair.

---

## Submodule Workflow

When pushing changes to `client/` or `server/`, update the umbrella to track the new commits:

```bash
cd /path/to/zoneforge
git add client server
git commit -m "Update submodule refs"
git push
```

Clone with submodules:

```bash
git clone --recurse-submodules https://github.com/bjsmithxyz/zoneforge
```

---

## Trade-offs & Risks

| Risk | Mitigation |
|------|-----------|
| Submodule UX is notoriously confusing for new contributors | README and Getting Started guide include explicit submodule clone instructions |
| Forgetting to update the umbrella after pushing to a submodule | Keep `git add client server && git commit` as a step in the dev workflow (documented in CLAUDE.md) |
| Submodule detached HEAD state confuses contributors | Document: always work in a branch inside the submodule, not in the umbrella checkout |

---

## Consequences

- Three GitHub repositories: `zoneforge` (umbrella), `zoneforge-client`, `zoneforge-server`
- All CI/CD runs in the submodule repos; the umbrella is a coordinator only
- `git clone --recurse-submodules` is the required clone command
