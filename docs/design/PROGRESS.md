# ZoneForge — Development Progress

## Current Focus

- [x] Complete development environment setup (Rust toolchain, WASM target, SpacetimeDB CLI)
- [x] Define core tables: Player, Zone, EntityInstance
- [x] Define core reducers: create_player, move_player, create_zone, spawn_entity
- [x] Build and publish server module to local SpacetimeDB
- [x] Unity 3D URP projects: game client (zoneforge-client) + standalone editor (zoneforge-editor)
- [x] Three-repo structure: server + client + editor submodules
- [x] SpacetimeDB C# SDK connected in both Unity projects

**Next:** Phase 2 Group 5 — 3D tile palette panel (in zoneforge-editor)

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

- [x] Define `Player` table (id, identity, name, position, health)
- [x] Define `Zone` table (id, name, grid_width, grid_height)
- [x] Define `EntityInstance` table (id, zone_id, prefab_name, position, type)
- [x] `create_player` reducer
- [x] `move_player` reducer
- [x] `create_zone` reducer
- [x] `spawn_entity` reducer
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

### Group 5 — Entity Placement (zoneforge-editor)

- [x] Runtime zone creation UI (UIToolkit panel replacing EditorWindow)
- [x] 3D tile palette panel with prefab thumbnails
- [ ] Raycast-based tile painting on X/Z grid plane
- [ ] Entity palette panel with sprite thumbnails
- [ ] Click-to-place entity placement (calls `spawn_entity` reducer)
- [ ] Subscribe to `EntityInstance` table (OnInsert callback)
- [ ] Real-time entity spawning visible in both editor and game client

### Group 6 — Player Movement

- [ ] Player Controller — WASD input + Transform updates
- [ ] `move_player` server-side validation
- [ ] Subscribe to `Player` table (OnUpdate callback)
- [ ] Client-side prediction + server reconciliation
- [ ] NavMesh setup and collision detection (zone boundaries, walls)
- [ ] Multi-client test: 2+ Unity clients in same zone

**Milestone: Players move in real time across multiple clients**

---

## Phase 3: Combat + Server Authority

### Group 7 — Combat Foundation

- [ ] `Ability` table (id, damage, cooldown, mana_cost, range)
- [ ] `StatusEffect` table (burn, freeze, stun, poison)
- [ ] `CombatLog` table (timestamp, attacker, target, damage)
- [ ] `use_ability` reducer (server-authoritative: validate range, cooldown, mana)
- [ ] `apply_damage` reducer
- [ ] Death and respawn logic
- [ ] Combat VFX — projectiles (fireball, arrow prefabs)

### Group 8 — Combat UI & Abilities

- [ ] Ability hotbar UI (slots 1–5)
- [ ] Cooldown indicators
- [ ] Health bar UI (player and enemy)
- [ ] Floating damage numbers
- [ ] Melee attack ability
- [ ] Fireball spell (ranged, burn DoT)
- [ ] Heal spell (self or ally)
- [ ] Object pooling for projectiles and VFX

### Group 9 — Enemy AI (Basic)

- [ ] Enemy AI — Idle, Chase, Attack states
- [ ] Melee, ranged, and caster behavior variants
- [ ] `AI State` table + `update_ai_state` reducer (server-side)
- [ ] `spawn_enemy` / `despawn_enemy` reducers

**Milestone: Server-authoritative combat working with 2+ players**

---

## Phase 4: Zone Stitching + Cloud Deployment

### Group 10 — Zone Portals

- [ ] `Portal` table (source_zone, dest_zone, spawn_point)
- [ ] `enter_zone` reducer (zone transfer with validation)
- [ ] Portal trigger volumes (Unity collision)
- [ ] Zone loading UI and scene cleanup
- [ ] Zone subscription management (subscribe/unsubscribe on transfer)
- [ ] World Graph editor window (node-based zone visualisation)

### Group 11 — Production Deployment

- [ ] Create 3 zones: Village, Forest, Cave
- [ ] Place and test portals between all zones
- [ ] Multi-player zone transfer testing
- [ ] SpacetimeDB Cloud account and CI/CD pipeline
- [ ] Production deployment
- [ ] Load testing (10+ concurrent players)

**Milestone: Multi-zone world accessible via SpacetimeDB Cloud**

---

## Phase 5: Systems Polish

### Group 12 — Triggers & Quests

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

### Group 13 — Inventory & Loot

- [ ] `Item` table (item definitions)
- [ ] `Inventory` table (player_id, item_id, quantity, slot)
- [ ] `Equipment` table (weapon, armor, accessory slots)
- [ ] Inventory, equip, and loot reducers
- [ ] Inventory grid UI with drag-and-drop and tooltips
- [ ] Equipment character sheet UI
- [ ] Loot table system and drop-on-death
- [ ] Pickup item reducer (collision-based)

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
- [ ] Party teleport

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
- [ ] `WorldEvent` table + trigger reducer
- [ ] World boss spawning and dynamic events
- [ ] Seasonal content and achievement system

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
