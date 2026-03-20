# ADR-004: Entity Prefab Naming Convention

**Date:** 2026-03-19
**Status:** Accepted

---

## Context

The `EntityInstance` table stores two string fields that together identify an entity:

- `prefab_name` — the Unity prefab asset to instantiate
- `entity_type` — the category of entity (NPC, Enemy, Prop, etc.)

Early prototypes used prefixed names for `prefab_name` (e.g., `npc_guard`, `prop_tree`) to make the category visible in the name alone. This is redundant because `entity_type` already carries that information.

---

## Decision

`prefab_name` stores only the **bare asset name** — no category prefix.

Examples:
- `guard` (not `npc_guard`)
- `tree` (not `prop_tree`)
- `skeleton` (not `enemy_skeleton`)

`entity_type` remains the authoritative category field for game logic, filtering, and UI grouping.

---

## Rationale

- `entity_type` already disambiguates category — the prefix is pure duplication
- Bare names match Unity's `Resources.Load<GameObject>("guard")` path directly
- Shorter names are easier to read in logs, debug tools, and the editor palette
- If the category taxonomy changes (e.g., a "Guard" becomes a "Boss"), `entity_type` can be updated without renaming the prefab asset

---

## Consequences

- New entity definitions must follow the bare-name convention
- Any future tooling that derives category from `prefab_name` alone is incorrect — always use `entity_type`
