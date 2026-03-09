# ZoneForge Development Kanban Board

## Sprint Structure Overview

This kanban board follows a 2-week sprint model across the 12-month development roadmap.

- **Phase 1**: Months 1-3 (6 sprints) - Core Editor + SpacetimeDB Foundation
- **Phase 2**: Months 4-6 (6 sprints) - Combat + Server Authority
- **Phase 3**: Months 7-9 (6 sprints) - Advanced Features + Scaling
- **Phase 4**: Months 10-12 (6 sprints) - Multiplayer Features (Future)

---

## Phase 1: Core Editor + SpacetimeDB Foundation

### Month 1 - Sprints 1-2: Foundation

```mermaid
%%{init: {'theme':'base'}}%%
kanban
  Backlog
    Sprint 1 Planning
    Sprint 2 Planning
  
  Sprint 1 (Week 1-2)
    Unity Project Setup[Unity 2022.3 LTS + URP]
    Install Rust Toolchain[rustc + cargo + wasm32 target]
    Install SpacetimeDB CLI[spacetime command]
    Create Git Repositories[client repo + server repo]
    Basic ScriptableObject Architecture[WorldData, ZoneVisualData]
    Initialize SpacetimeDB Module[Cargo project with spacetimedb dependency]
    Define Core Tables - Zone[Rust table definition]
    Define Core Tables - Player[Rust table definition]
    Test Local SpacetimeDB Server[spacetime start + publish]
    Install Material Maker[Set up folder structure, create first test graph - §9A]
    Install ArmorPaint[Paint first test NPC model using baked maps - §9B]

  Sprint 2 (Week 3-4)
    Basic Map Editor UI[Unity Editor Window - Zone Editor]
    Grid System Implementation[Tilemap Grid component]
    Tile Palette Setup[Ground, decoration, collision tiles]
    Import Test Assets[1 character, 5 props, basic textures]
    Create Zone Reducer[create_zone in Rust]
    Deploy Module to Local Server[spacetime publish --clear-database]
    SpacetimeDB C# SDK Integration[Unity package import]
    Test Client-Server Connection[Unity → localhost:3000]
    Village Ground Tile Sets[Grass, dirt path, stone cobble - §9A]
    Forest Floor Scatter Graph[Export grass/rock decoration set - §9C]
    Tree Canopy Graph[3 tree variants for Village zone - §9C]
  
  In Progress
    Documentation[Setup guide for dev environment]
  
  Done
    Roadmap Planning
    Design Document v2.0
```

### Month 2 - Sprints 3-4: Entity System + Real-Time Sync

```mermaid
%%{init: {'theme':'base'}}%%
kanban
  Backlog
    Sprint 3 Planning
    Sprint 4 Planning
  
  Sprint 3 (Week 5-6)
    Entity Palette Window[Unity Editor Window with thumbnails]
    EntityInstance Table[Rust table with zone_id, prefab_name, position]
    Drag-and-Drop Placement[Scene view entity placement]
    Entity Gizmos[Visual indicators in Scene view]
    Subscribe to EntityInstance Table[Unity C# - OnInsert callback]
    Spawn Entity Reducer[spawn_entity in Rust]
    Real-time Entity Spawning[Other clients see new entities]
    Player Table Expansion[Add movement_speed, zone_id]
    Smart Materials - Entity Props[Barrels, crates, walls via ArmorPaint - §9B]
  
  Sprint 4 (Week 7-8)
    Player Controller - Input[WASD input handling]
    Player Controller - Movement[Transform updates]
    Move Player Reducer[Server-side movement validation]
    Subscribe to Player Table[Unity C# - OnUpdate callback]
    Client-Side Prediction[Optimistic movement updates]
    Server Reconciliation[Correct invalid movements]
    NavMesh Setup[Basic pathfinding setup]
    Collision Detection[Zone boundaries, walls]
    Multi-Client Testing[2+ Unity clients, same zone]
  
  In Progress
    NPC Prefab Creation[3 NPC character models]
  
  Done
    Basic server-client sync working
```

### Month 3 - Sprints 5-6: Triggers and Multiplayer Playtest

```mermaid
%%{init: {'theme':'base'}}%%
kanban
  Backlog
    Sprint 5 Planning
    Sprint 6 Planning
  
  Sprint 5 (Week 9-10)
    Trigger Table[Rust table with trigger_type, zone_id, conditions]
    Visual Scripting UI - Triggers[Unity Visual Scripting integration]
    Condition System[has_item, quest_status, player_level conditions]
    Action System[spawn_entity, play_sound, give_item actions]
    Create Trigger Reducer[Rust reducer to register triggers]
    Execute Trigger Reducer[Check conditions, run actions]
    OnEnter Trigger Type[Zone-based collision detection]
    OnInteract Trigger Type[Player interaction key]
  
  Sprint 6 (Week 11-12)
    Quest Table[Basic quest structure in Rust]
    Item Table[Item definitions, inventory system]
    Inventory Reducer[add_item, remove_item, has_item]
    Simple Quest Flow[Talk to NPC → get item → return]
    NPC Interaction System[Click NPC, trigger dialogue]
    Quest UI - Objectives[Display active quest]
    Multi-Player Quest Testing[2-4 players complete quest together]
    Deploy to SpacetimeDB Cloud[Production deployment test]
  
  In Progress
    Village Zone Creation[64x64 map with 5 buildings]
  
  Done
    MILESTONE - Multiplayer Village Zone Complete
```

---

## Phase 2: Combat + Server Authority

### Month 4 - Sprints 7-8: Combat Foundation

```mermaid
%%{init: {'theme':'base'}}%%
kanban
  Backlog
    Sprint 7 Planning
    Sprint 8 Planning
  
  Sprint 7 (Week 13-14)
    Ability Table[ability_id, damage, cooldown, mana_cost, range]
    StatusEffect Table[burn, freeze, stun, poison]
    CombatLog Table[timestamp, attacker, target, damage]
    Use Ability Reducer[Server-authoritative combat logic]
    Damage Calculation[Validate range, cooldown, mana]
    Apply Damage Reducer[Update player health]
    Death Handling[Player death, respawn logic]
    Combat VFX - Projectiles[Fireball, arrow prefabs]
    Smart Materials - Enemy Prefabs[Worn/damaged look via ArmorPaint - §9B]
    Dungeon Tile Sets[Wet stone, carved rock, mossy floor - §9A]

  Sprint 8 (Week 15-16)
    Combat VFX - Impact Effects[Hit sparks, explosions]
    Combat VFX - Status Effects[Burn flames, freeze ice]
    Ability UI - Hotbar[1-5 ability slots]
    Ability UI - Cooldown Indicators[Visual cooldown timers]
    Health Bar UI[Player and enemy health bars]
    Damage Numbers[Floating combat text]
    Melee Attack Ability[Basic attack with no cooldown]
    Fireball Spell[Ranged projectile with burn DoT]
    Heal Spell[Self or ally target]
    ZoneForgePoolManager[Configure projectile and VFX pools - §8A]
    PooledProjectile + PooledVFX[Add to all combat prefabs - §8A]
  
  In Progress
    Enemy AI - Basic Behavior[Idle, Chase, Attack states]
  
  Done
    Server-authoritative combat working
```

### Month 5 - Sprints 9-10: Zone Stitching + Cloud Deployment

```mermaid
%%{init: {'theme':'base'}}%%
kanban
  Backlog
    Sprint 9 Planning
    Sprint 10 Planning
  
  Sprint 9 (Week 17-18)
    Portal Table[source_zone, dest_zone, spawn_point]
    Enter Zone Reducer[Zone transfer with validation]
    Portal Trigger Volumes[Unity collision triggers]
    Zone Loading UI[Loading screen]
    Scene Unloading[Cleanup previous zone]
    World Graph Editor Window[Node-based zone visualization]
    Create Portal UI[Connect zones in editor]
    Zone Subscription Management[Subscribe/unsubscribe on transfer]
    Dungeon Debris Scatter Graph[Cave zone decoration set - §9C]
  
  Sprint 10 (Week 19-20)
    Create 3 Zones[Village, Forest, Cave]
    Place Portals[Village ↔ Forest ↔ Cave]
    Test Zone Transfers[Single player transitions]
    Test Multi-Player Transfers[Players in different zones]
    SpacetimeDB Cloud Setup[Production account]
    CI/CD Pipeline[Automated deployment]
    Production Deployment[Deploy to cloud]
    Load Testing[10+ concurrent players]
  
  In Progress
    Forest Zone Design[Outdoor environment]
    Cave Zone Design[Dungeon environment]
  
  Done
    Cloud deployment successful
```

### Month 6 - Sprints 11-12: Systems Polish + Inventory

```mermaid
%%{init: {'theme':'base'}}%%
kanban
  Backlog
    Sprint 11 Planning
    Sprint 12 Planning
  
  Sprint 11 (Week 21-22)
    Inventory Table[player_id, item_id, quantity, slot]
    Equipment Table[weapon_slot, armor_slot, accessory_slots]
    Add Item Reducer[Server-side inventory management]
    Remove Item Reducer[Validation and stack handling]
    Equip Item Reducer[Apply stat bonuses]
    Inventory UI - Grid System[Drag-and-drop slots]
    Inventory UI - Item Tooltips[Hover information]
    Equipment UI - Character Sheet[Visual equipment slots]
  
  Sprint 12 (Week 23-24)
    Loot Table System[Enemy drop definitions]
    Drop Loot Reducer[Spawn loot on death]
    Pickup Item Reducer[Collision-based pickup]
    Trade System[Player-to-player trading]
    Health Bar Component[Reusable UI prefab]
    Minimap System[Zone overview]
    Quest Tracker UI[Active objectives display]
    Performance Optimization[Profiling and fixes]
    PooledEnemy Component[Add to all enemy prefabs before load testing - §8A]
  
  In Progress
    Boss Enemy Prefab[Cave boss model + animations]
  
  Done
    MILESTONE - Multiplayer Combat Demo Complete
```

---

## Phase 3: Advanced Features + Scaling

### Month 7 - Sprints 13-14: Advanced Triggers + Quests

```mermaid
%%{init: {'theme':'base'}}%%
kanban
  Backlog
    Sprint 13 Planning
    Sprint 14 Planning
  
  Sprint 13 (Week 25-26)
    Quest Table Expansion[objectives, rewards, prerequisites]
    QuestProgress Table[player_id, quest_id, objective_states]
    Start Quest Reducer[Initialize quest for player]
    Update Quest Objective Reducer[Track progress]
    Complete Quest Reducer[Give rewards]
    Quest Sharing[Party quest sync]
    Quest UI - Journal[Full quest list]
    Quest UI - Dialogue Tree[Branching conversations]
  
  Sprint 14 (Week 27-28)
    Advanced Conditions[TimeOfDay, WorldFlag, RandomChance]
    Advanced Actions[PlayCutscene, ChangeWeather, TeleportPlayer]
    Trigger Conditions - AND/OR Logic[Complex condition chains]
    Trigger Actions - Sequences[Ordered action execution]
    Timeline Integration[Unity Timeline for cutscenes]
    Cutscene Table[Store cutscene metadata]
    Play Cutscene Reducer[Trigger server-side events]
    Cutscene Camera Control[Cinemachine integration]
  
  In Progress
    Create 3 Quest Chains[Interconnected quests]
  
  Done
    Quest system fully functional
```

### Month 8 - Sprints 15-16: Interior Builder + Lighting

```mermaid
%%{init: {'theme':'base'}}%%
kanban
  Backlog
    Sprint 15 Planning
    Sprint 16 Planning
  
  Sprint 15 (Week 29-30)
    Interior Zone Type[Building interiors as separate zones]
    Furniture Placement Tool[Editor tool for interiors]
    Door Prefabs[Interactive doors with open/close]
    Window Prefabs[Light-casting windows]
    Door State Table[open/closed/locked state]
    Toggle Door Reducer[Server-side door control]
    Lighting Preset ScriptableObjects[Pre-configured lighting]
    Apply Lighting Preset[Runtime lighting updates]
    Interior Tile Sets[Wooden floor, plaster wall, stone fireplace - §9A]
    Smart Materials - Interior Props[Furniture, candles, bookshelves - §9B]
    Interior Clutter Scatter Graph[Tavern and building interiors - §9C]
  
  Sprint 16 (Week 31-32)
    Baked Lightmaps[Static lighting for interiors]
    Dynamic Lights - Torches[Flickering point lights]
    Dynamic Lights - Magic Orbs[Colored ambient lights]
    Light Cookie Setup[Window shadow patterns]
    Post-Processing Profiles[Per-zone atmosphere]
    Weather System Table[Rain, snow, fog states]
    Change Weather Reducer[Server-controlled weather]
    Ambient Sound System[Zone-specific audio]
  
  In Progress
    Tavern Interior[Fully furnished building]
  
  Done
    Interior zones with custom lighting
```

### Month 9 - Sprints 17-18: AI Enhancements + Production Readiness

```mermaid
%%{init: {'theme':'base'}}%%
kanban
  Backlog
    Sprint 17 Planning
    Sprint 18 Planning
  
  Sprint 17 (Week 33-34)
    Enemy AI - Melee Behavior[Rush and attack pattern]
    Enemy AI - Ranged Behavior[Kite and maintain distance]
    Enemy AI - Caster Behavior[Spell rotation and positioning]
    AI Pathfinding Reducer[Server-side NavMesh queries]
    AI State Table[Current behavior state per enemy]
    Update AI State Reducer[Behavior tree tick]
    Spawn Enemy Reducer[Zone-based spawning]
    Despawn Enemy Reducer[Cleanup and loot]
  
  Sprint 18 (Week 35-36)
    Performance Profiling[Client and server metrics]
    Database Query Optimization[Add indexes to hot tables]
    Reducer Optimization[Batch operations where possible]
    Network Bandwidth Optimization[Delta compression]
    Load Testing - 50 Players[Stress test infrastructure]
    Load Testing - 100 Players[Target concurrent users]
    Database Backup System[Automated backups]
    Server Monitoring Dashboard[Grafana + Prometheus]
    Error Tracking Setup[Sentry integration]
    Pool Size Tuning[Profile defaultCapacity/maxSize under 100-player load - §8A]
    LOD Distance Profiling[Decoration meshes during 100-player load test - §9C]
  
  In Progress
    Create 10 Unique Enemies[Different AI behaviors]
  
  Done
    MILESTONE - Production-Ready Vertical Slice Complete
```

---

## Phase 4: Multiplayer Features (Future Roadmap)

### Month 10 - Sprints 19-20: Party System

```mermaid
%%{init: {'theme':'base'}}%%
kanban
  Backlog
    Sprint 19 Planning
    Sprint 20 Planning
  
  Sprint 19 (Week 37-38)
    Party Table[party_id, leader_id, members list]
    PartyInvite Table[inviter, invitee, status]
    Create Party Reducer[Initialize new party]
    Invite to Party Reducer[Send invitation]
    Join Party Reducer[Accept invitation]
    Leave Party Reducer[Remove member]
    Party Chat System[Isolated chat channel]
    Party UI - Member List[Health bars, names]
  
  Sprint 20 (Week 39-40)
    Shared Quest Progress[Party quest tracking]
    Party Loot Distribution[Round-robin or need/greed]
    Party Experience Sharing[XP split among members]
    Party Teleport[Leader can teleport party]
    Party Finder UI[Browse available parties]
    Voice Chat Integration[Optional third-party service]
    Party Roles[Tank, healer, DPS designations]
  
  In Progress
    Party dungeons[Content for 4-player groups]
  
  Done
    Full party system implemented
```

### Month 11 - Sprints 21-22: PvP + Guild System

```mermaid
%%{init: {'theme':'base'}}%%
kanban
  Backlog
    Sprint 21 Planning
    Sprint 22 Planning
  
  Sprint 21 (Week 41-42)
    PvP Flag Table[player_id, pvp_enabled, cooldown]
    Toggle PvP Reducer[Enable/disable PvP mode]
    PvP Combat Validation[Only allow if both flagged]
    PvP Zone Table[Designated PvP areas]
    Duel System[1v1 consensual combat]
    Leaderboard Table[PvP rankings, kills, deaths]
    Update Leaderboard Reducer[Post-combat updates]
    PvP Rewards[Honor points, titles]
  
  Sprint 22 (Week 43-44)
    Guild Table[guild_id, name, leader, members]
    GuildInvite Table[Invitation management]
    Create Guild Reducer[Establish new guild]
    Guild Permissions Table[Officer ranks and rights]
    Guild Bank Table[Shared storage]
    Guild Hall Zone[Persistent guild space]
    Guild UI - Roster[Member list and roles]
    Guild Chat Channel[Dedicated communication]
  
  In Progress
    Guild progression system[Levels and perks]
  
  Done
    PvP and guilds functional
```

### Month 12 - Sprints 23-24: Economy + World Events

```mermaid
%%{init: {'theme':'base'}}%%
kanban
  Backlog
    Sprint 23 Planning
    Sprint 24 Planning
  
  Sprint 23 (Week 45-46)
    Trade Table[player trades, escrow system]
    AuctionHouse Table[listings, bids, buyouts]
    Create Listing Reducer[Post item for sale]
    Place Bid Reducer[Auction bidding]
    Buyout Item Reducer[Instant purchase]
    Currency System[Gold, gems, tokens]
    Crafting Table[Recipe definitions]
    Craft Item Reducer[Resource consumption, output]
    Merchant NPCs[Buy/sell interfaces]
  
  Sprint 24 (Week 47-48)
    WorldEvent Table[event_id, type, start_time, status]
    Trigger World Event Reducer[Server-wide events]
    World Boss Spawning[High-level raid content]
    Dynamic Events[Timed challenges]
    Seasonal Content Table[Holiday events]
    Event Rewards[Unique loot and achievements]
    Achievement System[Track player milestones]
    Final Polish[Bug fixes, UX improvements]
    Launch Preparation[Marketing, community setup]
  
  In Progress
    Beta Testing[Community playtest]
  
  Done
    MILESTONE - Full Multiplayer RPG Complete
```

---

## Sprint Velocity Tracking

```mermaid
%%{init: {'theme':'base'}}%%
graph LR
    A[Sprint Planning] --> B[Development 2 weeks]
    B --> C[Sprint Review]
    C --> D[Sprint Retrospective]
    D --> E[Next Sprint Planning]
    
    style A fill:#4CAF50
    style B fill:#2196F3
    style C fill:#FF9800
    style D fill:#9C27B0
    style E fill:#4CAF50
```

## Risk Board (Ongoing)

```mermaid
%%{init: {'theme':'base'}}%%
kanban
  High Priority Risks
    SpacetimeDB Learning Curve[Mitigate - Documentation, examples]
    Client-Server Desync[Mitigate - Server reconciliation]
    Network Latency[Mitigate - Client prediction]
    Cheating Exploits[Mitigate - Server authority]
  
  Medium Priority Risks
    Performance Issues[Monitor - Early profiling]
    Scope Creep[Control - Strict MVP]
    Cloud Costs[Track - Usage monitoring]
  
  Low Priority Risks
    Version Compatibility[Document - Lock versions]
    Data Loss[Backup - Commit log + backups]
  
  Resolved
    Technology Selection[Done - SpacetimeDB + Rust]
    Architecture Design[Done - Client-Server hybrid]
```

---

## Notes for Obsidian

- Each sprint is 2 weeks (10 working days)
- Use Obsidian's task plugin to track individual items: `- [ ] Task name`
- Link to related docs using `[[Document Name]]` syntax
- Tag sprints with `#sprint-1`, `#sprint-2`, etc.
- Use dataview plugin to query tasks by sprint or phase
- Add daily notes to track progress within sprints

### Example Dataview Query for Current Sprint

```dataview
TASK
WHERE contains(text, "#sprint-3")
GROUP BY file.link
```

### Sprint Board Template for Obsidian

Create a new note for each sprint:

```markdown
---
sprint: 3
phase: 1
start_date: 2026-04-01
end_date: 2026-04-14
tags: [sprint, phase-1, zoneforge]
---

# Sprint 3: Entity System + Real-Time Sync

## Sprint Goal
Implement real-time entity synchronization between Unity client and SpacetimeDB server.

## Tasks
- [ ] Entity Palette Window #sprint-3
- [ ] EntityInstance Table #sprint-3
- [ ] Drag-and-Drop Placement #sprint-3
...
