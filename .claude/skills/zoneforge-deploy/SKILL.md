---
name: zoneforge-deploy
description: Full ZoneForge server deploy workflow — build Rust WASM module, publish to SpacetimeDB, and regenerate C# client bindings. Use this skill whenever the user mentions deploying, publishing, rebuilding the server, pushing server changes, or updating the module. Also trigger when the user says "update the server", "push my changes", "test my changes", or anything that implies running the server pipeline. Don't wait for the user to say "deploy" explicitly — if they just finished editing lib.rs or any server file and want to see it running, this skill applies.
---

## When to use this skill

Any time server code has changed and needs to be live — whether for testing, a schema update, or a new feature.

## Step 1: Determine if schema changed

Ask (or infer from context): **Did you add, remove, or rename any tables or columns?**

- **Schema unchanged** → proceed to Step 2 normally
- **Schema changed, data safe** (additive only) → proceed normally, regenerate bindings in Step 4
- **Breaking schema change** (removed/renamed table or column) → warn the user before Step 3:

> ⚠️ This is a breaking schema change. Publishing with `--delete-data` will wipe all table data in the local database. Confirm before proceeding.

## Step 2: Build the server

Always run `spacetime build`, not `cargo build` — the CLI handles the WASM target correctly.

```bash
cd server && spacetime build
```

The `wasm-opt` warning that appears during build is non-critical — ignore it.

If the build fails, read the compiler errors and fix them before proceeding. Do not proceed to publish with a broken build.

## Step 3: Publish to local SpacetimeDB

```bash
# Standard publish (no data loss)
spacetime publish --server local zoneforge-server

# Breaking schema change only — wipes all table data
spacetime publish --server local zoneforge-server --delete-data
```

The `spacetime sql` / `spacetime call` UNSTABLE warning is expected — not an error.

Verify the publish succeeded before moving on. If it fails, check `spacetime logs zoneforge-server`.

## Step 4: Regenerate client bindings (if schema changed)

Run from the **client/** directory:

```bash
cd ../client && spacetime generate \
  --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

Never edit files in `Assets/Scripts/autogen/` — they are overwritten every time this runs.

## Step 5: Reimport in Unity (if bindings regenerated)

Tell the user: open Unity and run **Assets → Reimport All** to pick up the new generated types. If Unity is already open and autogen imports don't trigger automatically, Reimport All is required.

## Checklist

- [ ] Build succeeded (`spacetime build`)
- [ ] Published (`spacetime publish`)
- [ ] `--delete-data` confirmed if breaking schema change
- [ ] Bindings regenerated if schema changed
- [ ] Unity Reimport All if bindings changed
