# ZoneForge — Development Progress

## Current Focus

Phase 4 Group 11 in progress. Zones, portals, and zone transfer are working in the client.

Completed phases: 1 · 2 · 3 · 4 Groups 1–10, 11 (partial)

Remaining for next session — Phase 4 Group 11:

- SpacetimeDB Cloud account + CI/CD pipeline
- Production deployment
- Load testing (10+ concurrent players)

---

## Phase 1: Core Editor + SpacetimeDB Foundation

### Group 1 — Environment & Foundation

- [x] Roadmap and design document
- [x] Install Unity Hub + Unity 2022.3 LTS with 3D URP
- [x] Start SpacetimeDB local server
- [x] Install Rust toolchain (rustc, cargo, wasm32-unknown-unknown target)
- [x] Install SpacetimeDB CLI
- [x] Initialise Git repositories (server + client + editor submodules)
- [x] Configure VS Code with extensions

### Group 2 — Server Core

- [x] Define `Player` table (id, identity, name, zone_id, position_x/y, health, max_health, mana, max_mana, is_dead)
- [x] Define `Zone` table (id, name, terrain_width, terrain_height, water_level)
- [x] Define `EntityInstance` table (id, zone_id, prefab_name, position_x/y, elevation, entity_type String)
- [x] `create_player` reducer
- [x] `move_player` reducer (validates bounds, returns Result)
- [x] `create_zone` reducer (includes water_level param; initialises TerrainChunk rows)
- [x] `spawn_entity` reducer (includes elevation param; returns Result)
- [x] Build and publish module to local server
- [x] Verify with `spacetime sql` / `spacetime call`

### Group 3 — Unity Project & SDK

- [x] Create Unity projects (2022.3 LTS, 3D URP): zoneforge-client + zoneforge-editor
- [x] Basic ScriptableObject architecture (WorldData, ZoneVisualData) in both projects
- [x] Install SpacetimeDB C# SDK via Package Manager in both projects
- [x] SpacetimeDB connection manager (connect, subscribe, callbacks)
- [x] Verify Unity connects to local server and callbacks fire

### Group 4 — Editor Foundation (zoneforge-editor)

- [x] ZoneController component (`client/` and `editor/`)
- [x] Project structure: Editor scripts separated into zoneforge-editor
- [x] Placeholder tile PNG generator (dev tool in editor/)
- [x] Placeholder sprite PNG generator (dev tool in editor/)
- [x] Three-repo architecture established and confirmed 3D URP

**Milestone: Unity connects to SpacetimeDB, three-repo structure live, 3D URP confirmed**

---

## Phase 2: Entity System + Real-Time Sync

### Group 5 — Terrain System + Entity Placement (zoneforge-editor)

- [x] Runtime zone creation UI (UIToolkit panel, `ZoneCreationPanel.cs` + `ZoneCreationPanel.uxml/uss`)
- [x] Terrain system — server tables and reducers:
  - `TerrainChunk` table (`zone_id`, `chunk_x`, `chunk_z`, `height_data` byte[], `splat_data` byte[])
  - `update_terrain_chunk` reducer (validates 4096-byte arrays + chunk bounds; called by editor on brush stroke)
  - `create_zone` updated: initialises flat `TerrainChunk` rows for the whole zone on creation
- [x] Terrain system — editor runtime (`TerrainChunkData`, `TerrainBrush`, `HeightBrush`, `TextureBrush`, `CombinedBrush`, `TerrainPainter`, `TilePalettePanel`)
- [x] Terrain system — rendering (`TerrainRenderer` mesh from height data, `WaterRenderer` flat mesh)
- [x] Terrain system — client mirror (`TerrainRenderer`/`WaterRenderer`/`TerrainChunkData` in `zoneforge-client`)
- [x] Assembly definition chain: `ZoneForgeAutogen.asmdef` → `ZoneForgeRuntime.asmdef` → `EditModeTests.asmdef`
- [x] UI stylesheet fix: both panels styled and non-overlapping (Zone Manager top-left, Brush panel top-right)
- [x] Entity palette panel with sprite thumbnails
- [x] Click-to-place entity placement (calls `spawn_entity` reducer)
- [x] Subscribe to `EntityInstance` table (OnInsert callback)
- [x] Real-time entity spawning visible in both editor and game client
- [x] Editor UI polish: collapsible Zone Manager (auto-collapse on zone select) and Brush Panel (manual toggle)
- [x] Bug fix: zone list empty on startup (backfill from `Conn.Db.Zone.Iter()` in `OnConnected`)
- [x] Bug fix: terrain painting and entity placement broken (`MeshCollider.sharedMesh` sync in `TerrainRenderer`; Y=0 plane fallback in `EntityPlacer`)
- [x] Entity palette redesigned: foldout groups (NPC / Enemy / Prop), colour swatches, repositioned to avoid Brush Panel overlap

### Group 6 — Player Movement

- [x] Player Controller — WASD input + Transform updates
- [x] `move_player` server-side validation
- [x] Subscribe to `Player` table (OnUpdate callback)
- [x] Client-side prediction + server reconciliation
- [x] NavMesh setup and collision detection (zone boundaries, walls)
- [x] Multi-client test: 2+ Unity clients in same zone

**Milestone: Players move in real time across multiple clients**

---

## Phase 3: Combat + Server Authority

### Group 7 — Combat Foundation

- [x] `Ability` table (id, damage, cooldown, mana_cost, range)
- [x] `StatusEffect` table (burn, freeze, stun, poison)
- [x] `CombatLog` table (timestamp, attacker, target, damage)
- [x] `use_ability` reducer (server-authoritative: validate range, cooldown, mana)
- [x] `apply_damage` helper function (plain Rust fn, not a reducer — called internally)
- [x] Death and respawn logic
- [x] Combat VFX — projectiles (fireball, arrow prefabs)

### Group 8 — Combat UI & Abilities

- [x] Ability hotbar UI (slots 1–5)
- [x] Cooldown indicators
- [x] Health bar UI (player and enemy)
- [x] Floating damage numbers
- [x] Melee attack ability
- [x] Fireball spell (ranged, burn DoT)
- [x] Heal spell (self or ally)
- [x] Object pooling for projectiles and VFX

### Group 9 — Enemy AI (Basic)

- [x] Enemy AI — Idle, Chase, Attack states (client-side NavMesh; `NavMeshManager` in zoneforge-client)
- [x] Melee, ranged, and caster behavior variants
- [x] `AI State` column on `Enemy` table + `update_ai_state` reducer (admin override; `tick_ai` drives state machine)
- [x] `spawn_enemy_manual` / `despawn_enemy` reducers

**Milestone: Server-authoritative combat working with 2+ players**

---

## Phase 4: Zone Stitching + Cloud Deployment

### Group 10 — Zone Portals

- [x] `Portal` table (source_zone, dest_zone, spawn_point)
- [x] `enter_zone` reducer (zone transfer with validation)
- [x] Portal trigger volumes (Unity collision) — PortalManager ring GOs + proximity trigger
- [x] Zone loading UI and scene cleanup — RippleWarpEffect screen-space warp + ZoneTransferManager
- [x] Zone subscription management (subscribe/unsubscribe on transfer)
- [x] World Graph editor window — WorldGraphPanel with node canvas + edge drawing + PortalRenderer

### Group 11 — Production Deployment

- [x] Create 3 zones: Village, Forest, Cave
- [x] Place and test portals between all zones
- [x] Multi-player zone transfer testing (single-client verified; server handles concurrent players)
- [ ] SpacetimeDB Cloud account and CI/CD pipeline
- [ ] Production deployment
- [ ] Load testing (10+ concurrent players)

**Milestone: Multi-zone world accessible via SpacetimeDB Cloud**

---

## Phase 5: Systems Polish

### Group 12 — Triggers, Quests & NPC Authoring

- [ ] `Trigger` table (trigger_type, zone_id, conditions, actions)
- [ ] `OnEnter` and `OnInteract` trigger types
- [ ] Condition system (has_item, quest_status, player_level)
- [ ] Action system (spawn_entity, play_sound, give_item)
- [ ] Visual Scripting UI for triggers
- [ ] `Quest` table (objectives, rewards, prerequisites)
- [ ] `QuestProgress` table (player_id, quest_id, objective_states)
- [ ] Quest reducers: start, update_objective, complete
- [ ] Quest UI — journal and dialogue tree
- [ ] 3 interconnected quest chains
- [ ] **Editor tool:** NPC creation panel — define NPC archetypes (name, dialogue lines, vendor inventory, quest-giver flag)
- [ ] **Editor tool:** Quest designer panel — visual quest chain builder (nodes for objectives, rewards, prerequisites)

### Group 13 — Inventory & Loot

- [ ] `Item` table (item definitions)
- [ ] `Inventory` table (player_id, item_id, quantity, slot)
- [ ] `Equipment` table (weapon, armor, accessory slots)
- [ ] Inventory, equip, and loot reducers
- [ ] Inventory grid UI with drag-and-drop and tooltips
- [ ] Equipment character sheet UI
- [ ] Loot table system and drop-on-death
- [ ] Pickup item reducer (collision-based)
- [ ] **Editor tool:** Loot creation panel — define item templates and loot tables (drop chances, rarity tiers, enemy/chest assignment)

### Group 14 — Interiors & Atmosphere

- [ ] Interior zone type (building interiors as separate zones)
- [ ] Furniture placement editor tool
- [ ] Door/window prefabs with state table + toggle reducer
- [ ] Lighting preset ScriptableObjects + runtime application
- [ ] Baked lightmaps for interiors
- [ ] Dynamic lights (torches, magic orbs)
- [ ] Post-processing profiles per zone
- [ ] `WeatherState` table + `change_weather` reducer
- [ ] Ambient sound system

**Milestone: Polished multiplayer village with quests, inventory, and atmosphere**

---

## Phase 6: Advanced Multiplayer (Future)

### Group 15 — Party System

- [ ] `Party` / `PartyInvite` tables
- [ ] Party reducers (create, invite, join, leave)
- [ ] Shared quest progress and loot distribution
- [ ] Party UI (member list with health bars)

### Group 16 — PvP & Guilds

- [ ] `PvPFlag` table + toggle reducer
- [ ] PvP zone designation and duel system
- [ ] `Leaderboard` table + post-combat update reducer
- [ ] `Guild` / `GuildInvite` / `GuildPermissions` tables
- [ ] Guild reducers (create, invite, rank management)
- [ ] Guild bank and guild hall zone
- [ ] Guild UI (roster, chat channel)

### Group 17 — Economy & World Events

- [ ] `AuctionHouse` table (listings, bids, buyouts)
- [ ] Trade, bid, and buyout reducers
- [ ] Currency system (gold, gems, tokens)
- [ ] `Crafting` table + craft reducer

**Milestone: Full multiplayer RPG feature set complete**

---

## Risks

| Priority | Risk | Mitigation |
|----------|------|------------|
| High | Client-server desync | Server reconciliation |
| High | Network latency | Client-side prediction |
| High | Cheating exploits | Server authority on all game logic |
| Medium | Performance under load | Early profiling, object pooling |
| Medium | Scope creep | Strict phase ordering, milestones gate next phase |
| Medium | Cloud costs | Usage monitoring |
| Low | Version compatibility | Lock dependency versions in docs |
| Low | Data loss | SpacetimeDB commit log + automated backups |
| Resolved | Technology selection | SpacetimeDB + Rust + Unity |
| Resolved | Architecture design | Client-server hybrid, autogen bindings |

---

## Future / Post-Ship Considerations

These features are intentionally out of scope for the current roadmap. They are preserved here for reference and may be revisited after the game ships if the business model supports them.

### AI NPC Interaction System

**Status:** Design complete — no implementation planned.

Each NPC could respond to free-text player input using a runtime LLM (Claude API), while still being able to trigger structured game actions (start quest, give item, open shop) via LLM tool use. Per-NPC backstories and per-player relationship memory would be stored in SpacetimeDB. A stateless Python gateway service would handle all LLM communication (SpacetimeDB WASM cannot make external HTTP calls).

Blocked on: resolved business model (per-call API costs at player scale).

Full design: [docs/superpowers/specs/2026-03-25-ai-npc-interaction-design.md](../superpowers/specs/2026-03-25-ai-npc-interaction-design.md)
