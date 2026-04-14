# Phase 5 Group 14 — Atmosphere Systems Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver per-zone mood presets that animate across a global time-of-day clock, server-authoritative weather broadcast to all clients, and layered ambient audio that reacts to both — closing Phase 5 with a basic multiplayer village plus atmosphere.

**Architecture:** Server owns a single `WorldClock` row (global minutes-of-day) and a `WeatherState` row per zone. Zones reference a `mood_preset_id` (plain `u32`) resolved client-side to a `MoodPreset` ScriptableObject. `AtmosphereController` samples preset curves per frame and applies them to the directional sun, ambient, fog, and post-fx volume. `AmbientAudioMixer` blends base/weather/time audio layers via an `AudioMixer` asset.

**Tech Stack:** SpacetimeDB 2.0.3 Rust server (`server/spacetimedb/src/lib.rs`), Unity 2022.3 URP (`client/` and `editor/`), C# SpacetimeDB SDK, URP Volume framework, Unity AudioMixer.

**Design spec:** [docs/superpowers/specs/2026-04-14-group-14-atmosphere-design.md](../specs/2026-04-14-group-14-atmosphere-design.md)

---

## Preconditions

- Working in the umbrella repo (`~/zoneforge` or a worktree under `~/.config/superpowers/worktrees/zoneforge/<branch>`)
- `spacetime start` running in another terminal
- Current default server is `local` (`spacetime server list` shows `***` next to `local`)
- Phase 5 Group 13 is published and working; you can connect a client to `zoneforge-server` without errors
- A zone with `id = 1` exists (required by `create_player`). If not, create one via the editor first.

---

## File Structure

**Server (modify one file — lib.rs is monolithic in this repo):**
- `server/spacetimedb/src/lib.rs` — add `WeatherKind` enum, `WeatherState` table, `WorldClock` table + scheduled tick table, reducers `tick_world_time`, `change_weather`, `set_zone_mood`. Modify `Zone` struct and `init` reducer. Modify `create_zone` reducer.

**Client game project (`client/`):**
- Create: `client/Assets/Scripts/Data/MoodPreset.cs` — ScriptableObject bundle
- Create: `client/Assets/Scripts/Data/MoodPresetRegistry.cs` — id→asset lookup
- Create: `client/Assets/Scripts/Runtime/AtmosphereController.cs` — applies curves + weather VFX
- Create: `client/Assets/Scripts/Runtime/AmbientAudioMixer.cs` — 3-layer audio
- Create: `client/Assets/Resources/MoodPresets/` — folder for preset assets
- Create: `client/Assets/Resources/MoodPresets/village_day.asset`
- Create: `client/Assets/Resources/MoodPresets/forest.asset`
- Create: `client/Assets/Resources/MoodPresets/night_camp.asset`
- Create: `client/Assets/Resources/WeatherVFX/WeatherVFX_Rain.prefab`
- Create: `client/Assets/Resources/WeatherVFX/WeatherVFX_Fog.prefab`
- Create: `client/Assets/Resources/WeatherVFX/WeatherVFX_Storm.prefab` (stub)
- Create: `client/Assets/Resources/WeatherVFX/WeatherVFX_Snow.prefab` (stub)
- Create: `client/Assets/Audio/ZoneForgeMixer.mixer`
- Modify: `client/Assets/Scripts/autogen/` (regenerated)

**Editor project (`editor/`) — same components, mirrored:**
- Create: `editor/Assets/Scripts/Data/MoodPreset.cs`
- Create: `editor/Assets/Scripts/Data/MoodPresetRegistry.cs`
- Create: `editor/Assets/Scripts/Runtime/AtmosphereController.cs`
- Create: `editor/Assets/Scripts/Runtime/AmbientAudioMixer.cs`
- Create: `editor/Assets/Resources/MoodPresets/` (same 3 preset assets)
- Create: `editor/Assets/Resources/WeatherVFX/` (same 4 prefabs)
- Create: `editor/Assets/Audio/ZoneForgeMixer.mixer`
- Create: `editor/Assets/Scripts/Runtime/WeatherDebugPanel.cs` — admin weather buttons
- Modify: `editor/Assets/Scripts/Runtime/ZoneCreationPanel.cs` — add Mood Preset dropdown
- Modify: `editor/Assets/Scripts/autogen/` (regenerated)

> **Note on file duplication:** ZoneForge mirrors component code between `client/` and `editor/` submodules rather than sharing a common package. Each task below that creates a Unity C# file explicitly lists both destinations. Keep them byte-identical unless a task says otherwise.

---

## Task 1: Add `WeatherKind` enum and `WeatherState` table (server)

**Files:**
- Modify: `server/spacetimedb/src/lib.rs` (insert after existing `#[derive(SpacetimeType)]` enums, before `Zone` struct at line 175)

- [ ] **Step 1: Add the enum and table**

Open `server/spacetimedb/src/lib.rs`. Insert the following block immediately above the `#[table(accessor = zone, public)]` line:

```rust
#[derive(SpacetimeType, Clone, Copy, Debug, PartialEq)]
pub enum WeatherKind {
    Clear,
    Rain,
    Storm,
    Fog,
    Snow,
}

#[table(accessor = weather_state, public)]
pub struct WeatherState {
    #[primary_key]
    pub zone_id: u64,
    pub kind: WeatherKind,
    pub intensity: f32,
    pub started_at: Timestamp,
}
```

> **Why `u64` for `zone_id`:** The existing `Zone` table uses `pub id: u64`. Match it so foreign key joins work without casts.

- [ ] **Step 2: Build the module**

Run from `server/`:

```bash
cd server && spacetime build
```

Expected: `Compiled successfully` (ignore `wasm-opt` warning — per project convention it's non-critical).

- [ ] **Step 3: Commit**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add WeatherKind enum and WeatherState table"
```

---

## Task 2: Add `Zone.mood_preset_id` column

**Files:**
- Modify: `server/spacetimedb/src/lib.rs` — `Zone` struct (around line 175–184), `create_zone` reducer (around line 972)

- [ ] **Step 1: Add the field to `Zone`**

Replace the existing struct:

```rust
#[table(accessor = zone, public)]
pub struct Zone {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub name: String,
    pub terrain_width:  u32,
    pub terrain_height: u32,
    pub water_level:    f32,
    pub mood_preset_id: u32,
}
```

- [ ] **Step 2: Update `create_zone` to set the new field**

Find the `Zone { ... }` literal inside `create_zone` (around line 994) and append `mood_preset_id: 0,` as the last field. Example of the final literal (use real existing values for the other fields — this is an illustration):

```rust
let zone = Zone {
    id: 0,
    name: name.clone(),
    terrain_width,
    terrain_height,
    water_level,
    mood_preset_id: 0,
};
```

> **Why default `0`:** `0` is the reserved sentinel meaning "unassigned — client uses fallback preset".

- [ ] **Step 3: Build**

```bash
cd server && spacetime build
```

Expected: compiles. If you get "missing field `mood_preset_id`" errors from other code paths, they need the same field added.

- [ ] **Step 4: Search for any other places that construct `Zone`**

Run (from repo root):

```bash
rg -n "Zone \{" server/spacetimedb/src/lib.rs
```

For each match that is a struct literal (not a struct definition), add `mood_preset_id: existing.mood_preset_id,` if it's spreading an existing row, or `mood_preset_id: 0,` if it's constructing fresh. Re-run `spacetime build` until it compiles.

- [ ] **Step 5: Commit**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add mood_preset_id column to Zone"
```

---

## Task 3: Add `WorldClock` table and `tick_world_time` scheduled reducer

**Files:**
- Modify: `server/spacetimedb/src/lib.rs` — insert near other scheduled tables (search for `ManaRegenTick` around line 259) and their reducers

- [ ] **Step 1: Add the tables**

Insert after the existing `ManaRegenTick` table definition (roughly line 265):

```rust
#[table(accessor = world_clock, public)]
pub struct WorldClock {
    #[primary_key]
    pub id: u8,               // always 0 — single-row table
    pub minutes_of_day: u16,  // 0..1440
    pub last_tick: Timestamp,
}

#[table(accessor = world_clock_tick, scheduled(tick_world_time))]
pub struct WorldClockTick {
    #[primary_key]
    #[auto_inc]
    pub scheduled_id: u64,
    pub scheduled_at: ScheduleAt,
}
```

- [ ] **Step 2: Add the reducer**

Insert the reducer next to other scheduled reducers (find `tick_mana_regen` around line 394 and add after it):

```rust
#[reducer]
pub fn tick_world_time(ctx: &ReducerContext, _tick: WorldClockTick) {
    let clock = ctx.db.world_clock().id().find(0u8);
    if let Some(existing) = clock {
        // 1 real second = 1 in-game minute → 24 real minutes per in-game day.
        let next = (existing.minutes_of_day + 1) % 1440;
        ctx.db.world_clock().id().update(WorldClock {
            minutes_of_day: next,
            last_tick: ctx.timestamp,
            ..existing
        });
    }
}
```

> **Why the empty underscore arg:** Scheduled reducers always take the row that triggered them. We don't read it here, so we prefix with `_` to silence the unused warning.

- [ ] **Step 3: Build**

```bash
cd server && spacetime build
```

Expected: compiles.

- [ ] **Step 4: Commit**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add WorldClock table and tick_world_time reducer"
```

---

## Task 4: Add `change_weather` admin reducer

**Files:**
- Modify: `server/spacetimedb/src/lib.rs` — append near `create_zone` reducer

- [ ] **Step 1: Add the reducer**

Insert after `create_zone` (around line 1040):

```rust
#[reducer]
pub fn change_weather(
    ctx: &ReducerContext,
    zone_id: u64,
    kind: WeatherKind,
    intensity: f32,
) -> Result<(), String> {
    if !is_admin(ctx) {
        return Err("not admin".to_string());
    }
    if !(0.0..=1.0).contains(&intensity) {
        return Err("intensity must be 0.0..=1.0".to_string());
    }
    // Upsert: find-then-update, otherwise insert fresh.
    if let Some(existing) = ctx.db.weather_state().zone_id().find(zone_id) {
        ctx.db.weather_state().zone_id().update(WeatherState {
            kind,
            intensity,
            started_at: ctx.timestamp,
            ..existing
        });
    } else {
        ctx.db.weather_state().insert(WeatherState {
            zone_id,
            kind,
            intensity,
            started_at: ctx.timestamp,
        });
    }
    Ok(())
}
```

- [ ] **Step 2: Build**

```bash
cd server && spacetime build
```

Expected: compiles.

- [ ] **Step 3: Commit**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add change_weather admin reducer"
```

---

## Task 5: Add `set_zone_mood` admin reducer

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Step 1: Add the reducer**

Insert immediately after `change_weather`:

```rust
#[reducer]
pub fn set_zone_mood(
    ctx: &ReducerContext,
    zone_id: u64,
    mood_preset_id: u32,
) -> Result<(), String> {
    if !is_admin(ctx) {
        return Err("not admin".to_string());
    }
    let zone = ctx.db.zone().id().find(zone_id)
        .ok_or_else(|| format!("Zone {} not found", zone_id))?;
    ctx.db.zone().id().update(Zone { mood_preset_id, ..zone });
    Ok(())
}
```

- [ ] **Step 2: Build**

```bash
cd server && spacetime build
```

Expected: compiles.

- [ ] **Step 3: Commit**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add set_zone_mood admin reducer"
```

---

## Task 6: Bootstrap `WorldClock` in `init` and schedule the tick

**Files:**
- Modify: `server/spacetimedb/src/lib.rs` — `init` reducer (around line 626)

- [ ] **Step 1: Insert bootstrap at the end of `init`**

Open `init`. Add at the end of the function body (before the closing `}`):

```rust
    // WorldClock bootstrap
    if ctx.db.world_clock().id().find(0u8).is_none() {
        ctx.db.world_clock().insert(WorldClock {
            id: 0,
            minutes_of_day: 480, // 08:00
            last_tick: ctx.timestamp,
        });
    }
    // Schedule tick_world_time every 1 second
    ctx.db.world_clock_tick().insert(WorldClockTick {
        scheduled_id: 0,
        scheduled_at: ScheduleAt::Interval(std::time::Duration::from_secs(1).into()),
    });
```

> **Why `ScheduleAt::Interval`:** Repeats forever at the given interval, unlike `ScheduleAt::Time` which is one-shot. Matches existing `tick_mana_regen` pattern — check how that reducer schedules itself and mirror the exact syntax if the above errors out.

- [ ] **Step 2: Check the existing `tick_mana_regen` scheduling pattern**

Run:

```bash
rg -n "mana_regen_tick\(\).insert" server/spacetimedb/src/lib.rs
```

If the existing code uses `ScheduleAt::Time(ctx.timestamp + ...)` instead of `Interval`, use the same pattern for consistency — and note that `Time` is one-shot, so the reducer must re-schedule itself at the end:

```rust
ctx.db.world_clock_tick().insert(WorldClockTick {
    scheduled_id: 0,
    scheduled_at: ScheduleAt::Time(ctx.timestamp + std::time::Duration::from_secs(1)),
});
```

If you use the `Time` pattern, add the same re-schedule block at the end of `tick_world_time` (Task 3) so it keeps ticking.

- [ ] **Step 3: Build**

```bash
cd server && spacetime build
```

Expected: compiles.

- [ ] **Step 4: Commit**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): bootstrap WorldClock and schedule tick_world_time"
```

---

## Task 7: Default `WeatherState` row on zone creation

**Files:**
- Modify: `server/spacetimedb/src/lib.rs` — `create_zone` reducer

- [ ] **Step 1: Insert default `WeatherState` after zone insert**

Inside `create_zone`, after `let zone = ctx.db.zone().insert(zone);` (or the equivalent line that inserts the new zone row), add:

```rust
    ctx.db.weather_state().insert(WeatherState {
        zone_id: zone.id,
        kind: WeatherKind::Clear,
        intensity: 0.0,
        started_at: ctx.timestamp,
    });
```

> **Why after insert:** `zone.id` is auto-incremented by the database — it's `0` before insert, real id after.

- [ ] **Step 2: Build**

```bash
cd server && spacetime build
```

Expected: compiles.

- [ ] **Step 3: Commit**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): default WeatherState row on zone create"
```

---

## Task 8: Publish server with `--delete-data` and verify

> **⚠️ Destructive step.** `--delete-data` wipes every table. Required here because we added a column to `Zone`, which is a breaking schema change. After this step you will lose all existing zones, players, enemies, etc. and must recreate zone id=1 before clients can connect.

**Files:** none

- [ ] **Step 1: Publish**

Run from `server/`:

```bash
cd server && spacetime publish --server local zoneforge-server --delete-data
```

Expected: `Publishing module ... Done.` Answer `y` if prompted to confirm delete.

- [ ] **Step 2: Verify `WorldClock` bootstrap**

```bash
spacetime sql --server local zoneforge-server "SELECT * FROM world_clock"
```

Expected: one row with `id = 0`, `minutes_of_day = 480`.

- [ ] **Step 3: Verify time is advancing**

Wait 5 seconds, run again:

```bash
spacetime sql --server local zoneforge-server "SELECT * FROM world_clock"
```

Expected: `minutes_of_day` increased by ~5 (±1 for timing).

- [ ] **Step 4: Create zone id=1 (needed for client connection)**

Use the editor UI to create a zone, or call the reducer directly:

```bash
spacetime call --server local zoneforge-server create_zone '["Village", 64, 64, 0.0]'
```

(Match the actual `create_zone` argument order — run `rg -n 'pub fn create_zone' server/spacetimedb/src/lib.rs` first to confirm.)

- [ ] **Step 5: Verify default `WeatherState` was inserted**

```bash
spacetime sql --server local zoneforge-server "SELECT * FROM weather_state"
```

Expected: one row with `kind = Clear`, `intensity = 0.0`, `zone_id = 1`.

- [ ] **Step 6: Test `change_weather`**

```bash
spacetime call --server local zoneforge-server change_weather '[1, {"rain":{}}, 0.7]'
spacetime sql --server local zoneforge-server "SELECT kind, intensity FROM weather_state WHERE zone_id = 1"
```

Expected: `kind = Rain`, `intensity = 0.7`.

> **Note:** If the enum JSON encoding errors out, try `{"Rain":[]}` or `{"Rain":{}}` variants — check memory notes on EnemyType for the exact pattern.

- [ ] **Step 7: Test `set_zone_mood`**

```bash
spacetime call --server local zoneforge-server set_zone_mood '[1, 1]'
spacetime sql --server local zoneforge-server "SELECT id, mood_preset_id FROM zone WHERE id = 1"
```

Expected: `mood_preset_id = 1`.

- [ ] **Step 8: Commit (no files changed — skip if nothing staged)**

If Task 8 revealed tweaks needed to earlier tasks' code, commit those now.

---

## Task 9: Regenerate C# bindings for both Unity projects

**Files:**
- Modify: `client/Assets/Scripts/autogen/` (regenerated)
- Modify: `editor/Assets/Scripts/autogen/` (regenerated)

- [ ] **Step 1: Regenerate client bindings**

```bash
cd client && spacetime generate --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

Expected: `Generated N files`. No errors.

- [ ] **Step 2: Regenerate editor bindings**

```bash
cd ../editor && spacetime generate --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

Expected: `Generated N files`.

- [ ] **Step 3: Verify new types exist**

```bash
rg -l "WeatherState|WorldClock|WeatherKind" client/Assets/Scripts/autogen/
rg -l "WeatherState|WorldClock|WeatherKind" editor/Assets/Scripts/autogen/
```

Expected: at least 3 files matched in each directory.

- [ ] **Step 4: Commit bindings in each submodule**

```bash
cd client && git add Assets/Scripts/autogen && git commit -m "chore(autogen): regenerate for Group 14 atmosphere tables"
cd ../editor && git add Assets/Scripts/autogen && git commit -m "chore(autogen): regenerate for Group 14 atmosphere tables"
cd ..
```

---

## Task 10: Create `MoodPreset` ScriptableObject (client)

**Files:**
- Create: `client/Assets/Scripts/Data/MoodPreset.cs`

- [ ] **Step 1: Write the ScriptableObject**

```csharp
using UnityEngine;
using UnityEngine.Rendering;

namespace ZoneForge.Data
{
    [CreateAssetMenu(menuName = "ZoneForge/Mood Preset", fileName = "MoodPreset")]
    public class MoodPreset : ScriptableObject
    {
        [Tooltip("Must match server Zone.mood_preset_id. 0 is the fallback.")]
        public uint Id;

        public string DisplayName;

        [Header("Sun (curves indexed by minutes-of-day 0..1440)")]
        public AnimationCurve SunPitchCurve = AnimationCurve.Linear(0, -30, 1440, 330);
        public AnimationCurve SunYawCurve = AnimationCurve.Linear(0, 0, 1440, 0);

        [Header("Ambient & Fog")]
        public Gradient AmbientColorGradient;
        public Gradient FogColorGradient;
        public AnimationCurve FogDensityCurve = AnimationCurve.Constant(0, 1440, 0.01f);

        [Header("Post-FX")]
        public VolumeProfile PostFxProfile;

        [Header("Audio")]
        public AudioClip BaseAmbientClip;
    }
}
```

> **Why `uint`:** Server column is `u32`. C# binding maps to `uint`.

- [ ] **Step 2: Mirror the file into the editor project**

Copy the same file to `editor/Assets/Scripts/Data/MoodPreset.cs` byte-for-byte.

- [ ] **Step 3: Commit in both submodules**

```bash
cd client && git add Assets/Scripts/Data/MoodPreset.cs && git commit -m "feat: add MoodPreset ScriptableObject"
cd ../editor && git add Assets/Scripts/Data/MoodPreset.cs && git commit -m "feat: add MoodPreset ScriptableObject"
cd ..
```

---

## Task 11: Create `MoodPresetRegistry` (client + editor)

**Files:**
- Create: `client/Assets/Scripts/Data/MoodPresetRegistry.cs`
- Create: `editor/Assets/Scripts/Data/MoodPresetRegistry.cs`

- [ ] **Step 1: Write the registry**

```csharp
using System.Collections.Generic;
using UnityEngine;

namespace ZoneForge.Data
{
    public static class MoodPresetRegistry
    {
        private static Dictionary<uint, MoodPreset> _byId;
        private static MoodPreset _fallback;

        public static MoodPreset Get(uint id)
        {
            EnsureLoaded();
            return _byId.TryGetValue(id, out var preset) ? preset : _fallback;
        }

        public static IReadOnlyCollection<MoodPreset> All
        {
            get { EnsureLoaded(); return _byId.Values; }
        }

        public static void Reload()
        {
            _byId = null;
            EnsureLoaded();
        }

        private static void EnsureLoaded()
        {
            if (_byId != null) return;
            _byId = new Dictionary<uint, MoodPreset>();
            var assets = Resources.LoadAll<MoodPreset>("MoodPresets");
            foreach (var asset in assets)
            {
                if (_byId.ContainsKey(asset.Id))
                {
                    Debug.LogWarning($"Duplicate MoodPreset id {asset.Id}: {asset.name}");
                    continue;
                }
                _byId[asset.Id] = asset;
                if (asset.Id == 0) _fallback = asset;
            }
            if (_fallback == null && assets.Length > 0) _fallback = assets[0];
            if (_fallback == null) Debug.LogError("No MoodPreset assets found in Resources/MoodPresets/");
        }
    }
}
```

- [ ] **Step 2: Mirror to editor project**

Copy byte-for-byte to `editor/Assets/Scripts/Data/MoodPresetRegistry.cs`.

- [ ] **Step 3: Commit in both submodules**

```bash
cd client && git add Assets/Scripts/Data/MoodPresetRegistry.cs && git commit -m "feat: add MoodPresetRegistry"
cd ../editor && git add Assets/Scripts/Data/MoodPresetRegistry.cs && git commit -m "feat: add MoodPresetRegistry"
cd ..
```

---

## Task 12: Create the three MoodPreset assets

**Files:**
- Create: `client/Assets/Resources/MoodPresets/village_day.asset`
- Create: `client/Assets/Resources/MoodPresets/forest.asset`
- Create: `client/Assets/Resources/MoodPresets/night_camp.asset`
- Create: same three files under `editor/Assets/Resources/MoodPresets/`

> **This task must be done through the Unity editor GUI.** ScriptableObject assets are YAML-serialized with GUIDs and are painful to hand-author.

- [ ] **Step 1: Open the client Unity project and create the folder**

In Unity: `Project` → right-click `Assets/Resources/` → `Create → Folder → MoodPresets`.

- [ ] **Step 2: Create `village_day`**

Right-click `Assets/Resources/MoodPresets` → `Create → ZoneForge → Mood Preset`. Rename to `village_day`. In the Inspector, set:

- `Id = 1`
- `DisplayName = "Village Day"`
- `SunPitchCurve`: keyframes at `(0, -30)`, `(360, 0)`, `(720, 60)`, `(1080, 0)`, `(1440, -30)` (dawn/noon/dusk arc)
- `AmbientColorGradient`: cool-blue at 0, warm-white at 0.5, orange at 0.75, dark-blue at 1.0
- `FogColorGradient`: match ambient but desaturated
- `FogDensityCurve`: flat 0.005
- `PostFxProfile`: leave null for now (optional — can assign a URP VolumeProfile later)
- `BaseAmbientClip`: pick any free loop from the project, or leave null if none exists

- [ ] **Step 3: Create `forest`**

Duplicate `village_day`, rename `forest`. Set `Id = 2`, `DisplayName = "Forest"`, tint ambient greener, fog slightly denser (0.015).

- [ ] **Step 4: Create `night_camp`**

Duplicate `village_day`, rename `night_camp`. Set `Id = 3`, `DisplayName = "Night Camp"`, sun curve flat at -30 (sun below horizon), ambient dark blue, fog dark and denser (0.02).

- [ ] **Step 5: Copy the three assets to the editor project**

From a terminal:

```bash
cp -r client/Assets/Resources/MoodPresets editor/Assets/Resources/MoodPresets
cp client/Assets/Resources/MoodPresets/*.meta editor/Assets/Resources/MoodPresets/
```

Then open the editor Unity project — it will import the assets with fresh GUIDs. If the importer complains about duplicate GUIDs, delete the `.meta` files under `editor/Assets/Resources/MoodPresets/` and let Unity regenerate them.

- [ ] **Step 6: Commit in both submodules**

```bash
cd client && git add Assets/Resources/MoodPresets && git commit -m "content: add village_day/forest/night_camp MoodPresets"
cd ../editor && git add Assets/Resources/MoodPresets && git commit -m "content: add village_day/forest/night_camp MoodPresets"
cd ..
```

---

## Task 13: Create `AudioMixer` asset with Base/Weather/Time groups

**Files:**
- Create: `client/Assets/Audio/ZoneForgeMixer.mixer`
- Create: `editor/Assets/Audio/ZoneForgeMixer.mixer`

> **Unity editor GUI task.**

- [ ] **Step 1: Open client project, create the mixer**

`Assets/Audio/` → right-click → `Create → Audio Mixer` → name `ZoneForgeMixer`.

- [ ] **Step 2: Add child groups**

Open the mixer window. Right-click `Master` → `Add Child Group` → name `Ambient`. Then right-click `Ambient` → add three children: `Base`, `Weather`, `Time`.

- [ ] **Step 3: Expose volume parameters**

For each of `Base`, `Weather`, `Time`: in Inspector, right-click the `Volume` field → `Expose 'Volume (of X)' to script`. Rename exposed parameters to `BaseVolume`, `WeatherVolume`, `TimeVolume` in the `Exposed Parameters` dropdown (top-right of mixer window).

- [ ] **Step 4: Copy mixer to editor project**

```bash
cp client/Assets/Audio/ZoneForgeMixer.mixer editor/Assets/Audio/ZoneForgeMixer.mixer
cp client/Assets/Audio/ZoneForgeMixer.mixer.meta editor/Assets/Audio/ZoneForgeMixer.mixer.meta
```

Open editor Unity project to let it import. If GUID clash, delete the `.meta` and let Unity regenerate.

- [ ] **Step 5: Commit in both submodules**

```bash
cd client && git add Assets/Audio && git commit -m "content: add ZoneForgeMixer AudioMixer"
cd ../editor && git add Assets/Audio && git commit -m "content: add ZoneForgeMixer AudioMixer"
cd ..
```

---

## Task 14: Create `AmbientAudioMixer` MonoBehaviour

**Files:**
- Create: `client/Assets/Scripts/Runtime/AmbientAudioMixer.cs`
- Create: `editor/Assets/Scripts/Runtime/AmbientAudioMixer.cs`

- [ ] **Step 1: Write the component**

```csharp
using UnityEngine;
using UnityEngine.Audio;
using ZoneForge.Data;

namespace ZoneForge.Runtime
{
    public class AmbientAudioMixer : MonoBehaviour
    {
        [SerializeField] private AudioMixer _mixer;
        [SerializeField] private AudioMixerGroup _baseGroup;
        [SerializeField] private AudioMixerGroup _weatherGroup;
        [SerializeField] private AudioMixerGroup _timeGroup;

        [Header("Weather clips (index by WeatherKind ordinal)")]
        [SerializeField] private AudioClip _rainClip;
        [SerializeField] private AudioClip _stormClip;
        [SerializeField] private AudioClip _fogClip;
        [SerializeField] private AudioClip _snowClip;

        [Header("Time clips")]
        [SerializeField] private AudioClip _dayClip;
        [SerializeField] private AudioClip _nightClip;

        private AudioSource _baseSource, _weatherSource, _timeSource;
        private const float FadeSeconds = 1.5f;

        void Awake()
        {
            _baseSource = NewLoopingSource(_baseGroup);
            _weatherSource = NewLoopingSource(_weatherGroup);
            _timeSource = NewLoopingSource(_timeGroup);
        }

        private AudioSource NewLoopingSource(AudioMixerGroup group)
        {
            var src = gameObject.AddComponent<AudioSource>();
            src.loop = true;
            src.playOnAwake = false;
            src.outputAudioMixerGroup = group;
            return src;
        }

        public void ApplyMood(MoodPreset preset)
        {
            Crossfade(_baseSource, preset != null ? preset.BaseAmbientClip : null);
        }

        public void ApplyWeather(int weatherKindOrdinal)
        {
            AudioClip clip = weatherKindOrdinal switch
            {
                1 => _rainClip,
                2 => _stormClip,
                3 => _fogClip,
                4 => _snowClip,
                _ => null, // Clear
            };
            Crossfade(_weatherSource, clip);
        }

        public void ApplyTimeOfDay(int minutesOfDay)
        {
            // Simple day/night split — 06:00–18:00 = day.
            bool isDay = minutesOfDay >= 360 && minutesOfDay < 1080;
            Crossfade(_timeSource, isDay ? _dayClip : _nightClip);
        }

        private void Crossfade(AudioSource src, AudioClip nextClip)
        {
            if (src.clip == nextClip) return;
            if (nextClip == null)
            {
                src.Stop();
                src.clip = null;
                return;
            }
            src.clip = nextClip;
            src.Play();
            // Fade is provided by the mixer group's exposed volume — left as a manual tune.
            // Real crossfade would use AudioMixer.SetFloat with a smoothed value; stubbed for milestone.
        }
    }
}
```

> **Why the crossfade is stubbed:** Real crossfades need a coroutine lerping an exposed `AudioMixer` parameter. The milestone only needs "audio changes when state changes"; smooth crossfade is a polish-pass item. Noted in the spec as "fade duration tuning during implementation."

- [ ] **Step 2: Mirror to editor project**

Copy byte-for-byte to `editor/Assets/Scripts/Runtime/AmbientAudioMixer.cs`.

- [ ] **Step 3: Commit in both submodules**

```bash
cd client && git add Assets/Scripts/Runtime/AmbientAudioMixer.cs && git commit -m "feat: add AmbientAudioMixer component"
cd ../editor && git add Assets/Scripts/Runtime/AmbientAudioMixer.cs && git commit -m "feat: add AmbientAudioMixer component"
cd ..
```

---

## Task 15: Create `AtmosphereController` MonoBehaviour

**Files:**
- Create: `client/Assets/Scripts/Runtime/AtmosphereController.cs`
- Create: `editor/Assets/Scripts/Runtime/AtmosphereController.cs`

- [ ] **Step 1: Write the component**

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Rendering;
using SpacetimeDB.Types;
using ZoneForge.Data;

namespace ZoneForge.Runtime
{
    public class AtmosphereController : MonoBehaviour
    {
        [SerializeField] private Light _sun;
        [SerializeField] private Volume _postFxVolume;
        [SerializeField] private AmbientAudioMixer _audioMixer;
        [SerializeField] private Transform _weatherVfxParent;

        public ulong CurrentZoneId { get; private set; }

        private MoodPreset _currentPreset;
        private int _cachedMinutes = 480;
        private GameObject _currentWeatherVfx;
        private WeatherKind _currentWeather = WeatherKind.Clear;
        private Dictionary<WeatherKind, GameObject> _vfxPrefabs;

        void Awake()
        {
            _vfxPrefabs = new Dictionary<WeatherKind, GameObject>
            {
                [WeatherKind.Rain] = Resources.Load<GameObject>("WeatherVFX/WeatherVFX_Rain"),
                [WeatherKind.Fog] = Resources.Load<GameObject>("WeatherVFX/WeatherVFX_Fog"),
                [WeatherKind.Storm] = Resources.Load<GameObject>("WeatherVFX/WeatherVFX_Storm"),
                [WeatherKind.Snow] = Resources.Load<GameObject>("WeatherVFX/WeatherVFX_Snow"),
            };
        }

        public void SetZone(ulong zoneId, uint moodPresetId)
        {
            CurrentZoneId = zoneId;
            _currentPreset = MoodPresetRegistry.Get(moodPresetId);
            if (_audioMixer != null) _audioMixer.ApplyMood(_currentPreset);
        }

        public void OnWorldClockChanged(ushort minutesOfDay)
        {
            _cachedMinutes = minutesOfDay;
            if (_audioMixer != null) _audioMixer.ApplyTimeOfDay(minutesOfDay);
        }

        public void OnWeatherChanged(WeatherKind kind, float intensity)
        {
            if (kind == _currentWeather) return;
            _currentWeather = kind;

            if (_currentWeatherVfx != null)
            {
                Destroy(_currentWeatherVfx);
                _currentWeatherVfx = null;
            }
            if (kind != WeatherKind.Clear && _vfxPrefabs.TryGetValue(kind, out var prefab) && prefab != null)
            {
                _currentWeatherVfx = Instantiate(prefab, _weatherVfxParent != null ? _weatherVfxParent : transform);
            }
            if (_audioMixer != null) _audioMixer.ApplyWeather((int)kind);
        }

        void Update()
        {
            if (_currentPreset == null || _sun == null) return;
            float t = _cachedMinutes; // AnimationCurve keys are in minutes 0..1440
            float normalized = _cachedMinutes / 1440f;

            _sun.transform.localRotation = Quaternion.Euler(
                _currentPreset.SunPitchCurve.Evaluate(t),
                _currentPreset.SunYawCurve.Evaluate(t),
                0f);

            RenderSettings.ambientLight = _currentPreset.AmbientColorGradient.Evaluate(normalized);
            RenderSettings.fogColor = _currentPreset.FogColorGradient.Evaluate(normalized);
            RenderSettings.fogDensity = _currentPreset.FogDensityCurve.Evaluate(t);

            if (_postFxVolume != null && _currentPreset.PostFxProfile != null)
            {
                _postFxVolume.profile = _currentPreset.PostFxProfile;
            }
        }
    }
}
```

> **Why `WeatherKind` from `SpacetimeDB.Types`:** The regenerated bindings put generated types in that namespace. If the namespace differs in your `autogen/` output, change the `using`. Run `rg -n "enum WeatherKind" client/Assets/Scripts/autogen/` to confirm.

- [ ] **Step 2: Mirror to editor project**

Copy to `editor/Assets/Scripts/Runtime/AtmosphereController.cs`.

- [ ] **Step 3: Commit in both submodules**

```bash
cd client && git add Assets/Scripts/Runtime/AtmosphereController.cs && git commit -m "feat: add AtmosphereController component"
cd ../editor && git add Assets/Scripts/Runtime/AtmosphereController.cs && git commit -m "feat: add AtmosphereController component"
cd ..
```

---

## Task 16: Wire `AtmosphereController` to SpacetimeDB subscriptions

**Files:**
- Modify: `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs` (location may vary — use the file that calls `SubscribeToAllTables` or similar)
- Modify: `editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs`

- [ ] **Step 1: Find the existing subscription wiring**

```bash
rg -n "OnInsert|OnUpdate|Subscribe" client/Assets/Scripts/Runtime/SpacetimeDBManager.cs
```

Identify how existing tables (e.g. `Zone`, `Player`) register insert/update callbacks.

- [ ] **Step 2: Register `WorldClock` callback**

In `SpacetimeDBManager.OnConnected` (or wherever callbacks are registered — mirror the existing pattern), add:

```csharp
conn.Db.WorldClock.OnInsert += (ctx, row) => UpdateAtmosphereClock(row);
conn.Db.WorldClock.OnUpdate += (ctx, oldRow, newRow) => UpdateAtmosphereClock(newRow);

void UpdateAtmosphereClock(WorldClock row)
{
    var controller = FindObjectOfType<AtmosphereController>();
    if (controller != null) controller.OnWorldClockChanged(row.MinutesOfDay);
}
```

> **Why `FindObjectOfType`:** Keeps this task self-contained. If the codebase has a singleton registry pattern (check `PlayerManager` etc.), use that instead.

- [ ] **Step 3: Register `WeatherState` callback**

Same location:

```csharp
conn.Db.WeatherState.OnInsert += (ctx, row) => UpdateAtmosphereWeather(row);
conn.Db.WeatherState.OnUpdate += (ctx, oldRow, newRow) => UpdateAtmosphereWeather(newRow);

void UpdateAtmosphereWeather(WeatherState row)
{
    var controller = FindObjectOfType<AtmosphereController>();
    if (controller == null) return;
    if (controller.CurrentZoneId != row.ZoneId) return;
    controller.OnWeatherChanged(row.Kind, row.Intensity);
}
```

- [ ] **Step 4: Register `Zone` update callback to pick up `mood_preset_id` changes**

Find the existing `Zone` subscription (there almost certainly is one). In its `OnUpdate` callback, add after existing logic:

```csharp
var controller = FindObjectOfType<AtmosphereController>();
if (controller != null && controller.CurrentZoneId == newRow.Id)
{
    controller.SetZone(newRow.Id, newRow.MoodPresetId);
}
```

If no existing `Zone.OnUpdate` handler exists, add one matching the pattern of other table handlers in the file.

- [ ] **Step 5: Call `SetZone` on initial zone entry**

Find where the client currently decides "player entered zone N" (search for `zone_id`, `ZoneId`, or `enter_zone`). After the player's current zone is known, call:

```csharp
var zone = conn.Db.Zone.Id.Find(playerZoneId);
if (zone != null) FindObjectOfType<AtmosphereController>()?.SetZone(zone.Id, zone.MoodPresetId);
```

- [ ] **Step 6: Mirror all changes to editor `SpacetimeDBManager.cs`**

Same diffs in `editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs`.

- [ ] **Step 7: Commit in both submodules**

```bash
cd client && git add Assets/Scripts/Runtime/SpacetimeDBManager.cs && git commit -m "feat: wire AtmosphereController to WorldClock/WeatherState/Zone subscriptions"
cd ../editor && git add Assets/Scripts/Runtime/SpacetimeDBManager.cs && git commit -m "feat: wire AtmosphereController to WorldClock/WeatherState/Zone subscriptions"
cd ..
```

---

## Task 17: Create Weather VFX prefabs

**Files:**
- Create: `client/Assets/Resources/WeatherVFX/WeatherVFX_Rain.prefab`
- Create: `client/Assets/Resources/WeatherVFX/WeatherVFX_Fog.prefab`
- Create: `client/Assets/Resources/WeatherVFX/WeatherVFX_Storm.prefab` (stub — duplicate of Rain for milestone)
- Create: `client/Assets/Resources/WeatherVFX/WeatherVFX_Snow.prefab` (stub — duplicate of Rain for milestone)
- Mirror all four into `editor/Assets/Resources/WeatherVFX/`

> **Unity editor GUI task.**

- [ ] **Step 1: Rain prefab**

In Unity client project: `Assets/Resources/` → create folder `WeatherVFX`. Right-click → `Create → Prefab Variant` is not right for a fresh prefab — instead:

1. In Hierarchy, create an empty GameObject `WeatherVFX_Rain`
2. Add `ParticleSystem` component
3. Tune: `Start Speed = 10`, `Start Lifetime = 1`, `Emission Rate = 200`, `Shape = Box (x=40, y=1, z=40)`, orient the box above camera head
4. Material: URP Particles/Unlit with a simple rain drop sprite (or a 2x8 white rectangle texture)
5. Drag the GameObject from Hierarchy into `Assets/Resources/WeatherVFX/` to create the prefab
6. Delete the Hierarchy instance

- [ ] **Step 2: Fog prefab**

Create `WeatherVFX_Fog` with a ParticleSystem using a soft white sprite, `Start Size = 5`, `Emission Rate = 5`, `Start Lifetime = 8`, large box shape around camera.

- [ ] **Step 3: Storm and Snow stubs**

Duplicate `WeatherVFX_Rain.prefab` twice → rename copies to `WeatherVFX_Storm.prefab` and `WeatherVFX_Snow.prefab`. Leave internals unchanged — they are placeholders for later polish.

- [ ] **Step 4: Mirror to editor project**

```bash
cp -r client/Assets/Resources/WeatherVFX editor/Assets/Resources/WeatherVFX
cp client/Assets/Resources/WeatherVFX/*.meta editor/Assets/Resources/WeatherVFX/
```

Open editor project to import. Delete duplicate `.meta` files if GUID clash.

- [ ] **Step 5: Commit in both submodules**

```bash
cd client && git add Assets/Resources/WeatherVFX && git commit -m "content: add weather VFX prefabs (rain, fog, storm/snow stubs)"
cd ../editor && git add Assets/Resources/WeatherVFX && git commit -m "content: add weather VFX prefabs (rain, fog, storm/snow stubs)"
cd ..
```

---

## Task 18: Add `AtmosphereController` + `AmbientAudioMixer` GameObjects to scenes

**Files:**
- Modify: `client/Assets/Scenes/SampleScene.unity`
- Modify: `editor/Assets/Scenes/SampleScene.unity` (or whatever the editor's main scene is — confirm with `rg -l ".unity" editor/Assets/Scenes/`)

> **Unity editor GUI task.**

- [ ] **Step 1: Client scene**

Open `SampleScene`. Create an empty GameObject `AtmosphereSystem`. Add `AtmosphereController` and `AmbientAudioMixer` components. In the Inspector:

- `AtmosphereController._sun`: drag the existing Directional Light
- `AtmosphereController._postFxVolume`: drag the scene's global Volume (create one if none exists: `GameObject → Volume → Global Volume`)
- `AtmosphereController._audioMixer`: drag the `AmbientAudioMixer` from the same GameObject
- `AtmosphereController._weatherVfxParent`: drag the Main Camera so weather VFX follows the player
- `AmbientAudioMixer._mixer`: drag `ZoneForgeMixer`
- `AmbientAudioMixer._baseGroup / _weatherGroup / _timeGroup`: drag Base/Weather/Time groups from the mixer
- Leave weather and time clips empty for now (milestone acceptance doesn't require audio variety)

- [ ] **Step 2: Editor scene**

Same setup in the editor project's main scene.

- [ ] **Step 3: Commit scene changes in both submodules**

```bash
cd client && git add Assets/Scenes && git commit -m "scene: add AtmosphereSystem GameObject to SampleScene"
cd ../editor && git add Assets/Scenes && git commit -m "scene: add AtmosphereSystem GameObject to main scene"
cd ..
```

---

## Task 19: Editor — Zone Inspector Mood Preset dropdown

**Files:**
- Modify: `editor/Assets/Scripts/Runtime/ZoneCreationPanel.cs`

- [ ] **Step 1: Locate the zone edit flow**

```bash
rg -n "ZoneCreationPanel|update_zone\|UpdateZone" editor/Assets/Scripts/Runtime/ZoneCreationPanel.cs
```

Understand how the panel currently writes to a zone. The dropdown should live alongside existing zone-edit fields.

- [ ] **Step 2: Add a DropdownField for mood preset**

In the UXML builder / runtime UI code, add after the existing fields (inside whatever builder method constructs the form):

```csharp
var moodDropdown = new UnityEngine.UIElements.DropdownField("Mood Preset",
    ZoneForge.Data.MoodPresetRegistry.All
        .Select(p => $"{p.Id}: {p.DisplayName}")
        .ToList(),
    0);
moodDropdown.RegisterValueChangedCallback(evt =>
{
    var idStr = evt.newValue.Split(':')[0];
    if (uint.TryParse(idStr, out var id))
    {
        SpacetimeDBManager.Conn.Reducers.SetZoneMood(_editingZoneId, id);
    }
});
rootContainer.Add(moodDropdown); // rootContainer = whatever VisualElement holds the other fields
```

> **Why a formatted string:** Unity's `DropdownField` is string-based. Embedding the id as a prefix is the cheapest lookup round-trip. Don't use the index — the `All` collection is unordered.

> **Add imports:** `using System.Linq; using ZoneForge.Data;`

- [ ] **Step 3: Initialize selection from the zone being edited**

When the panel opens for an existing zone, set `moodDropdown.value` to the matching `"{id}: {DisplayName}"` string based on `zone.MoodPresetId`:

```csharp
var current = ZoneForge.Data.MoodPresetRegistry.Get(zone.MoodPresetId);
if (current != null) moodDropdown.value = $"{current.Id}: {current.DisplayName}";
```

- [ ] **Step 4: Commit**

```bash
cd editor && git add Assets/Scripts/Runtime/ZoneCreationPanel.cs && git commit -m "feat(editor): Mood Preset dropdown in Zone panel"
cd ..
```

---

## Task 20: Editor — Admin Weather Debug Panel

**Files:**
- Create: `editor/Assets/Scripts/Runtime/WeatherDebugPanel.cs`
- Modify: `editor/Assets/Scripts/Runtime/ToolbarController.cs` (or wherever the editor toolbar lives — to open the panel)

- [ ] **Step 1: Write the panel**

```csharp
using SpacetimeDB.Types;
using UnityEngine;
using UnityEngine.UIElements;

namespace ZoneForge.Runtime
{
    public class WeatherDebugPanel : MonoBehaviour
    {
        [SerializeField] private UIDocument _document;
        private float _intensity = 0.5f;

        void OnEnable()
        {
            var root = _document.rootVisualElement;
            root.pickingMode = PickingMode.Ignore; // per project convention for full-screen panels

            var panel = new VisualElement();
            panel.style.position = Position.Absolute;
            panel.style.right = 10;
            panel.style.top = 10;
            panel.style.width = 220;
            panel.style.backgroundColor = new Color(0, 0, 0, 0.7f);
            panel.style.paddingLeft = panel.style.paddingRight = panel.style.paddingTop = panel.style.paddingBottom = 8;

            panel.Add(new Label("Weather (admin)") { style = { color = Color.white } });

            var slider = new Slider("Intensity", 0f, 1f) { value = _intensity };
            slider.RegisterValueChangedCallback(e => _intensity = e.newValue);
            panel.Add(slider);

            panel.Add(MakeButton("Clear", WeatherKind.Clear));
            panel.Add(MakeButton("Rain", WeatherKind.Rain));
            panel.Add(MakeButton("Storm", WeatherKind.Storm));
            panel.Add(MakeButton("Fog", WeatherKind.Fog));
            panel.Add(MakeButton("Snow", WeatherKind.Snow));

            root.Add(panel);
        }

        private Button MakeButton(string label, WeatherKind kind)
        {
            var btn = new Button(() =>
            {
                var controller = FindObjectOfType<AtmosphereController>();
                if (controller == null) { Debug.LogWarning("No AtmosphereController"); return; }
                SpacetimeDBManager.Conn.Reducers.ChangeWeather(controller.CurrentZoneId, kind, _intensity);
            }) { text = label };
            return btn;
        }
    }
}
```

> **`SpacetimeDBManager.Conn`:** Confirm the actual static/property name with `rg -n "public static .* Conn" editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs` and adjust.

- [ ] **Step 2: Add a scene GameObject for the panel**

In the editor Unity project: create `WeatherDebugPanel` GameObject → add `UIDocument` → assign a new empty Panel Settings asset → assign the `WeatherDebugPanel` component. Set `UIDocument.Sort Order = 20` (higher than background overlays, lower than `ZoneCreationPanel`'s 100 per memory notes).

- [ ] **Step 3: Wire toolbar toggle (optional for milestone)**

Add a button to `ToolbarController` that `SetActive(!active)`s the panel GameObject. If you prefer to keep it always-on for now, skip.

- [ ] **Step 4: Commit**

```bash
cd editor && git add Assets/Scripts/Runtime/WeatherDebugPanel.cs Assets/Scenes && git commit -m "feat(editor): Weather debug panel for admin testing"
cd ..
```

---

## Task 21: Integration smoke test

**Files:** none — this is a manual verification task.

- [ ] **Step 1: Ensure server is published and running**

```bash
spacetime sql --server local zoneforge-server "SELECT minutes_of_day FROM world_clock"
```

Expected: row present, `minutes_of_day` advancing.

- [ ] **Step 2: Open the editor Unity project, press Play**

Log in, connect to the server, open/select zone 1.

Expected:

- No console errors
- Sun rotation updates visibly over ~30 seconds
- Ambient color shifts over a few minutes

- [ ] **Step 3: Click Rain in the Weather Debug Panel**

Expected:

- Rain particle system appears above the camera
- `SELECT kind FROM weather_state WHERE zone_id = 1` → `Rain`

- [ ] **Step 4: Click Clear**

Expected:

- Rain VFX disappears
- Row updates to `Clear`

- [ ] **Step 5: Change mood preset via dropdown**

Select `2: Forest`. Expected:

- Ambient tint shifts toward green
- `SELECT mood_preset_id FROM zone WHERE id = 1` → `2`

- [ ] **Step 6: Open game client (`client/` Unity project) and connect**

Same zone, same observations — should see identical sun angle and weather as the editor client (server-authoritative sync working).

- [ ] **Step 7: If all checks pass, commit submodule pointers in the umbrella repo**

```bash
cd ~/zoneforge  # (or worktree root)
git add client editor server
git commit -m "chore: advance submodules — Group 14 atmosphere"
```

- [ ] **Step 8: If checks fail, debug using zoneforge-debug skill**

Don't patch symptoms — run the full stack triage (server logs, sql inspection, C# bindings check, Unity console).

---

## Task 22: Mark Group 14 complete in PROGRESS.md

**Files:**
- Modify: `docs/design/PROGRESS.md`

- [ ] **Step 1: Check off all Group 14 items**

Replace every `- [ ]` under Group 14 with `- [x]`.

- [ ] **Step 2: Update Current Focus section**

Replace the top-of-file "Current Focus" paragraph with something like:

```markdown
Phase 5 Group 14 complete. Atmosphere systems (lighting, weather, ambient sound) implemented.

Next: Phase 5 polish pass across all prior groups before Phase 6 begins.

Completed phases: 1 · 2 · 3 · 4 (local) · 5 Groups 12–14
```

- [ ] **Step 3: Commit**

```bash
git add docs/design/PROGRESS.md
git commit -m "docs: mark Phase 5 Group 14 complete in PROGRESS.md"
```

---

## Done

Phase 5 is feature-complete. Next session begins the open-ended polish pass across Groups 1–14 before Phase 6.
