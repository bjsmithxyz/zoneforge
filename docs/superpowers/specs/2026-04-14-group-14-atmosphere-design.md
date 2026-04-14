# Phase 5 Group 14 — Atmosphere Systems Design

**Date:** 2026-04-14
**Phase:** 5 (Systems Polish)
**Group:** 14
**Status:** Approved, pending implementation plan

## Summary

Phase 5 Group 14 delivers the atmosphere layer for ZoneForge: per-zone lighting moods that interpolate across a global time-of-day clock, server-authoritative weather broadcast to all clients in a zone, and a layered ambient audio system that reacts to both. Together these produce the sensory foundation for the "basic multiplayer village" milestone that closes Phase 5.

## Scope Changes From Original PROGRESS.md

The original Group 14 checklist has been revised. Items removed or deferred:

- **Interior zones as separate zones** — removed. Interiors are now part of the existing parent zone (open buildings, no loading barrier). True instanced interiors (private to a player or party) are deferred to Phase 6.
- **Furniture placement editor tool** — deferred to a later dedicated doodad placement phase.
- **Door/window prefabs + state table + toggle reducer** — deferred to doodad phase (doors are doodads with state).
- **Baked lightmaps for interiors** — removed. Static bakes conflict with dynamic time-of-day lighting.
- **Dynamic lights (torches, magic orbs)** — deferred to doodad phase (torches are doodads).

Items retained and expanded: lighting presets, weather state + reducer, post-processing (folded into lighting preset), ambient sound (expanded to layered mixer).

## Architecture Overview

```
WorldClock (server, ticks minutes_of_day) ─┐
WeatherState (server, per-zone row)        ├─► AtmosphereController (client)
Zone.mood_preset_id (server)               ┘         │
                                                     ├─► Sun rotation / ambient / fog / post-fx
                                                     └─► AmbientAudioMixer (3 layers)
```

Single subscription per client. Weather and time-of-day sync come for free via SpacetimeDB row broadcast. Clients never make authoritative atmosphere decisions.

## Server Components

### WeatherKind enum

```rust
#[derive(SpacetimeType, Clone, Copy, Debug, PartialEq)]
pub enum WeatherKind {
    Clear,
    Rain,
    Storm,
    Fog,
    Snow,
}
```

### WeatherState table

One row per zone. Created on zone creation (default `Clear`). Updated via `change_weather` reducer.

```rust
#[table(name = weather_state, public)]
pub struct WeatherState {
    #[primary_key]
    pub zone_id: u32,
    pub kind: WeatherKind,
    pub intensity: f32, // 0.0 – 1.0
    pub started_at: Timestamp,
}
```

### WorldClock table

Single-row table holding global time-of-day. `id = 0` as primary key convention.

```rust
#[table(name = world_clock, public)]
pub struct WorldClock {
    #[primary_key]
    pub id: u8, // always 0
    pub minutes_of_day: u16, // 0 – 1439
    pub last_tick: Timestamp,
}
```

### Zone table addition

Add `mood_preset_id: u32` column. Default `0` (untyped). Client resolves id → `MoodPreset` ScriptableObject via registry.

### Reducers

- **`change_weather(zone_id: u32, kind: WeatherKind, intensity: f32)`** — admin-gated (`ADMIN_IDENTITIES` check). Updates or inserts `WeatherState` row, sets `started_at = ctx.timestamp`.
- **`tick_world_time`** — scheduled reducer (every real second). Advances `minutes_of_day` by N (tuned for desired day length, e.g. 1 real second = 1 in-game minute → 24-minute real days). Wraps at 1440.
- **`set_zone_mood(zone_id: u32, mood_preset_id: u32)`** — admin-gated. Updates the zone row's `mood_preset_id`.

### Bootstrap

On `init` reducer: insert `WorldClock { id: 0, minutes_of_day: 480, last_tick: ctx.timestamp }` (480 = 08:00 in-game). Also insert default `WeatherState` rows for any pre-existing zones in the same init. On zone creation reducers, also insert a default `WeatherState` row.

## Client Components (Unity — both `client/` and `editor/`)

### MoodPreset ScriptableObject

One asset per mood. Bundles everything authored together:

```csharp
[CreateAssetMenu(menuName = "ZoneForge/Mood Preset")]
public class MoodPreset : ScriptableObject
{
    public uint Id;                          // matches server mood_preset_id
    public string DisplayName;

    // Time-of-day curves (x = 0..1440 minutes, y = value)
    public AnimationCurve SunPitchCurve;
    public AnimationCurve SunYawCurve;
    public Gradient AmbientColorGradient;    // evaluate with t = minutes / 1440
    public Gradient FogColorGradient;
    public AnimationCurve FogDensityCurve;

    public VolumeProfile PostFxProfile;      // URP post-processing
    public AudioClip BaseAmbientClip;        // zone loop layer
}
```

### MoodPresetRegistry

Static lookup from `uint id → MoodPreset`. Loads all assets from `Resources/MoodPresets/` on first access. Fallback to a hardcoded default preset if id not found.

### AtmosphereController MonoBehaviour

One per scene (both client game scene and editor scene). Responsibilities:

- Subscribe to `WorldClock` (single row, stays subbed forever)
- Subscribe to `WeatherState` filtered by current zone
- Resolve current zone's `mood_preset_id` → `MoodPreset`
- Each frame:
  - Read `minutes_of_day` from cached `WorldClock` row (interpolate locally between server ticks for smoothness)
  - Sample all preset curves/gradients at current time
  - Apply to scene directional light (Sun), `RenderSettings.ambientLight`, `RenderSettings.fog*`, the active `Volume` component's profile
- On `WeatherState` row change: trigger crossfade of weather VFX prefab (see below)
- On zone change: swap `MoodPreset` and re-subscribe to new zone's `WeatherState`

### Weather VFX prefabs

One prefab per `WeatherKind` (excluding `Clear`): `WeatherVFX_Rain`, `WeatherVFX_Storm`, `WeatherVFX_Fog`, `WeatherVFX_Snow`. Each contains:

- Particle system(s) parented to the camera (so effects follow the player)
- Optional post-fx delta (slight contrast/saturation shift, blended on top of mood preset's base profile)
- Spawn rate and emission driven by `WeatherState.intensity`

`AtmosphereController` instantiates the appropriate prefab on weather change and fades out the previous one over ~2 seconds.

### AmbientAudioMixer MonoBehaviour

Three `AudioSource` components, each routed to a mixer group:

- **Base** — the zone's `MoodPreset.BaseAmbientClip`, looped
- **Weather** — mapped from `WeatherKind` (e.g. `Rain` → rain loop clip, `Clear` → silent)
- **Time** — mapped from time-of-day buckets (dawn chorus, day birds, dusk crickets, night wind)

All three crossfade via `AudioMixer.SetFloat` exposed volume parameters (~1.5 s fade). Master `AudioMixer` asset has groups `Master → Ambient → {Base, Weather, Time}`.

## Editor Authoring

- **Zone Inspector panel** gets a new dropdown: "Mood Preset" — populated from `MoodPresetRegistry.AllPresets`. Selecting calls `set_zone_mood` reducer.
- **Admin weather debug panel** — new floating panel (admin only). Buttons per `WeatherKind`, slider for intensity, targets current zone. Calls `change_weather` reducer.

## Content Deliverables (Minimal for Milestone)

- 3 MoodPresets committed as assets: `village_day`, `forest`, `night_camp`
- 1 ambient audio clip per preset (royalty-free placeholder OK)
- Rain and fog `WeatherVFX` prefabs fully wired; storm/snow can be stubs for milestone
- `Master/Ambient/Base/Weather/Time` AudioMixer asset

## Data Flow Summary

1. Server ticks `WorldClock.minutes_of_day` once per real second
2. Client caches latest `WorldClock` row, interpolates between ticks for smooth sun motion
3. Client reads current zone's `WeatherState` and `Zone.mood_preset_id`
4. `AtmosphereController` samples `MoodPreset` curves at interpolated time → applies to sun, ambient, fog, post-fx
5. `AmbientAudioMixer` blends three audio layers based on mood, weather, and time bucket
6. On weather change: crossfade VFX prefab + weather audio layer
7. On zone change: resubscribe to new zone's `WeatherState`, swap mood preset, rebuild audio stack

## Error Handling

- Unknown `mood_preset_id` on client → fallback to default preset, log warning
- `WorldClock` row missing (shouldn't happen post-init) → client uses local clock fallback at 12:00 static
- `WeatherState` row missing for a zone → treat as `Clear`, intensity 0
- Missing `BaseAmbientClip` on a preset → base layer silent, no crash

## Testing Strategy

- **Server:** unit tests for `tick_world_time` wrap at 1440, `change_weather` admin gate rejects non-admin, `set_zone_mood` updates row
- **Integration:** two clients in same zone observe identical weather after admin triggers change
- **Client manual:** visual check that sun moves smoothly across a 24-minute in-game day, weather crossfades cleanly, ambient audio layers blend without pops
- **Editor manual:** authoring a new `MoodPreset` and assigning it to a zone reflects immediately in the editor viewport

## Milestone Wording (Replaces Current)

> **Milestone: Basic multiplayer village with quests, inventory, and atmosphere — feature-complete for Phase 5.**
>
> Before Phase 6 begins, Phase 5 enters a polish pass touching every prior group: combat feel, UI refinement, editor UX, server perf, bug sweep. The polish pass is not tracked as a numbered group — it is open-ended until the quality bar is met.

## Out of Scope (Explicitly)

- Instanced interior zones (Phase 6)
- Furniture and doodad placement (later dedicated phase)
- Door/window state and toggle reducers (doodad phase)
- Dynamic point lights, torches, magic orbs (doodad phase)
- Baked lightmaps (incompatible with dynamic time-of-day)
- One-shot SFX event table (deferred — can be added in polish pass if needed)
- Per-player weather overrides
- Seasonal variation / long-term weather patterns

## Open Questions

None at design-approval time. Tuning values (day length in real seconds, fade durations, intensity scaling) to be decided during implementation.
