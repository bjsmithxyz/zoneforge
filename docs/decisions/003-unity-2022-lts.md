# ADR 003 — Use Unity 2022.3 LTS as the client engine

**Date:** March 2026
**Status:** Accepted

---

## Context

ZoneForge requires a 3D client engine capable of:

- Isometric rendering with a customisable render pipeline
- A rich editor extension API (custom Editor Windows, Tilemap, Inspector tools)
- Mature C# scripting with good IDE tooling
- Active LTS support for the duration of the project (~12 months minimum)
- SpacetimeDB C# SDK compatibility

The primary choice was between **Unity 2022.3 LTS** and **Unity 6** (the latest release at project start).

---

## Decision

Use **Unity 2022.3 LTS**, pinned at **2022.3.62f3**.

---

## Reasons

**1. LTS stability**
Unity 2022.3 is a Long Term Support release, meaning Unity commits to bug-fix patches for a minimum of 2 years with no new feature churn. Unity 6 is a major release that introduces breaking changes and is still receiving stability patches.

**2. SpacetimeDB SDK compatibility**
The SpacetimeDB C# SDK was validated against Unity 2022.3 at project start. Unity 6 changes the render pipeline APIs and scripting backend in ways that had not been fully tested with the SDK version in use.

**3. Ecosystem maturity**
Community packages, tutorials, and StackOverflow answers are overwhelmingly written for 2022.3. Edge cases encountered during development are more likely to have existing solutions documented.

**4. Tooling stability**
VS Code's C# extension and Unity debug adapter have known-good configurations for 2022.3. Unity 6 introduced changes to the .NET runtime target that caused intermittent issues with the `ms-dotnettools.csdevkit` extension at project start.

**5. URP is fully mature on 2022.3**
Universal Render Pipeline on 2022.3 LTS has all features required by ZoneForge (post-processing, lightmaps, custom shaders). URP on Unity 6 received significant internal refactoring; waiting for it to stabilise on an LTS is prudent.

---

## Upgrade Path

Unity 6 LTS is expected to be released in late 2026. The project should evaluate upgrading after:

1. Unity 6 receives its first LTS designation
2. SpacetimeDB Unity SDK explicitly validates Unity 6 support
3. Phase 2 (combat + cloud deployment) is complete — mid-project engine upgrades carry high risk

---

## Trade-offs & Risks

| Risk | Mitigation |
|------|-----------|
| 2022.3 reaches end of LTS support before project ships | Monitor Unity LTS roadmap; upgrade to Unity 6 LTS when it stabilises |
| Unity 6 features (e.g. improved multiplayer tooling) unavailable | ZoneForge's multiplayer is handled by SpacetimeDB, not Unity Netcode — Unity 6 networking features are not needed |
| New hires expect Unity 6 | 2022.3 → Unity 6 migration is well-documented; the Unity Editor API is largely stable between versions |

---

## Consequences

- `client/ProjectSettings/ProjectVersion.txt` is pinned to `2022.3.62f3`
- All documentation references "Unity 2022.3 LTS" — do not update to "Unity 6" until an upgrade decision is made
- Unity Hub must have the 2022.3.x LTS editor installed; Unity 6 editor is not required
