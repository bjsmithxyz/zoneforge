# Phase 5 Group 12 — Triggers, Quests & NPC Authoring

**Date:** 2026-03-26
**Status:** Approved
**Scope:** Server schema + reducers, client UI + managers, editor tools, quest content

---

## Overview

This spec covers the trigger system, quest system, NPC authoring, dialogue trees, a unified visual scripting editor, and a minimal item/inventory backbone pulled forward from Group 13. Three interconnected quest chains exercise all features.

### Key Design Decisions

- **NPCs are a top-level table** (like Enemy), not a sub-record of EntityInstance — avoids SpacetimeDB's no-join constraint and matches the existing Enemy pattern.
- **Hybrid trigger evaluation** — client detects spatial events (OnEnter via collider, OnInteract via click), server validates conditions and executes state-changing actions. Client handles presentation-only actions (play_sound) locally.
- **Branching dialogue trees** — stored as a graph of DialogueNode + DialogueChoice rows. Supports linear sequences (one choice per node) and branching (multiple choices with optional conditions).
- **Unified visual scripting panel** in the editor — one node canvas for triggers, quests, dialogue, and conditions/actions. Different node types share the same graph infrastructure.
- **Minimal item/inventory** pulled forward from Group 13 — just Item definitions + Inventory table + give/remove helpers. No equipment, loot tables, or drag-and-drop UI.

---

## 1. Server Schema

### New Enums (SpacetimeType)

```rust
pub enum NpcType { QuestGiver, Vendor, Dialogue }
pub enum TriggerType { OnEnter, OnInteract }
pub enum QuestStatus { NotStarted, Active, Completed, Failed }
pub enum ObjectiveType { KillEnemy, CollectItem, TalkToNpc, EnterZone }
```

### New Tables

#### NPC System

| Table | Key Fields | Notes |
|-------|-----------|-------|
| `NpcDefinition` | id (PK auto_inc), name, npc_type, default_dialogue_tree_id | NPC archetypes (like EnemyDefinition) |
| `Npc` | id (PK auto_inc), zone_id (btree), npc_def_id, position_x, position_y, name, dialogue_tree_id | Placed NPC instances (own table, like Enemy) |

#### Dialogue System

| Table | Key Fields | Notes |
|-------|-----------|-------|
| `DialogueNode` | id (PK auto_inc), tree_id (btree), npc_text, sort_order | One node in a dialogue graph |
| `DialogueChoice` | id (PK auto_inc), node_id (btree), choice_text, next_node_id (nullable — null = terminal), condition_json, action_json | Player response option; condition_json gates visibility; action_json fires on selection |

`tree_id` is a logical grouping ID (u64). The `create_dialogue_tree` reducer auto-assigns the next available tree_id by scanning existing DialogueNode rows for the current max tree_id + 1. Each NPC references a tree_id. If the editor passes tree_id=0, the server assigns a new one; otherwise it uses the provided ID (for updates/overwrites).

#### Quest System

| Table | Key Fields | Notes |
|-------|-----------|-------|
| `Quest` | id (PK auto_inc), name, description, prerequisite_quest_ids_json, reward_action_json | Quest definition; prerequisites as JSON array of quest IDs; reward_action_json executes on completion (e.g., give_item) |
| `QuestObjective` | id (PK auto_inc), quest_id (btree), description, objective_type, target_json, required_count, sort_order, loot_on_kill_json | Individual objective; target_json holds type-specific data (enemy_def_id, item_id, npc_id, zone_id); loot_on_kill_json (optional) specifies items to drop when a KillEnemy objective kill occurs (e.g., `{"item_id": 5, "count": 1}`) |
| `QuestProgress` | id (PK auto_inc), player_id (btree), quest_id, status | Per-player quest state |
| `ObjectiveProgress` | id (PK auto_inc), player_id (btree), objective_id, current_count | Per-player per-objective progress |

#### Trigger System

| Table | Key Fields | Notes |
|-------|-----------|-------|
| `Trigger` | id (PK auto_inc), zone_id (btree), trigger_type, position_x, position_y, radius, condition_json, action_json | World triggers; condition_json and action_json are serialized arrays |

#### Item & Inventory (Minimal — pulled from Group 13)

| Table | Key Fields | Notes |
|-------|-----------|-------|
| `Item` | id (PK auto_inc), name, description, item_type, max_stack | Item definitions only |
| `Inventory` | id (PK auto_inc), player_id (btree), item_id, quantity | Per-player inventory; one row per item stack |

### Condition & Action JSON Schemas

Conditions and actions are stored as JSON strings in trigger, dialogue choice, and quest objective fields. The server parses and evaluates them.

**Conditions:**
```json
[
  {"type": "quest_status", "quest_id": 1, "status": "Completed"},
  {"type": "has_item", "item_id": 5, "count": 2},
  {"type": "player_level", "min_level": 3}
]
```
All conditions in the array must be true (AND logic). Empty array = always true.

`player_level` is included in the schema for future use but will return true unconditionally until a leveling system exists.

**Actions:**
```json
[
  {"type": "give_item", "item_id": 3, "count": 1},
  {"type": "spawn_entity", "zone_id": 2, "prefab": "boss_goblin", "x": 10.0, "y": 15.0},
  {"type": "start_quest", "quest_id": 2},
  {"type": "play_sound", "sound_id": "quest_complete"}
]
```
Actions execute sequentially. `play_sound` is client-side only — the server includes it in the response but doesn't process it.

---

## 2. Server Reducers

### NPC Reducers

- `create_npc_def(name: String, npc_type: NpcType)` — create NPC archetype. Admin only.
- `delete_npc_def(id: u64)` — remove NPC definition. Admin only.
- `spawn_npc(zone_id: u64, npc_def_id: u64, x: f32, y: f32, dialogue_tree_id: u64)` — place NPC in world. Admin only. Validates zone exists and npc_def exists.
- `despawn_npc(id: u64)` — remove NPC instance. Admin only.
- `interact_with_npc(npc_id: u64)` — player initiates dialogue. Server validates NPC exists and player is in same zone. Returns first dialogue node via table state (client reads DialogueNode rows for the NPC's tree_id).

### Dialogue Reducers

- `create_dialogue_tree(tree_id: u64, nodes_json: String, choices_json: String)` — batch-create a full dialogue tree. Editor sends serialized arrays of nodes and choices. Admin only. Deletes any existing nodes/choices with the same tree_id first (full replace on save).
- `delete_dialogue_tree(tree_id: u64)` — remove all nodes and choices for a tree. Admin only.
- `select_dialogue_choice(npc_id: u64, choice_id: u64)` — player selects a dialogue option. Server validates: choice exists, choice belongs to the NPC's current tree, conditions met. Executes action_json if present. Updates any relevant quest objectives (TalkToNpc type).

### Quest Reducers

- `create_quest(name: String, description: String, objectives_json: String, prerequisite_quest_ids_json: String)` — create quest definition with objectives. Admin only. Parses objectives_json into QuestObjective rows.
- `delete_quest(id: u64)` — remove quest and all objectives. Admin only.
- `accept_quest(quest_id: u64)` — player starts quest. Validates prerequisites (all prerequisite quests have Completed status for this player). Creates QuestProgress (Active) and ObjectiveProgress rows (count=0).
- `complete_quest(quest_id: u64)` — player turns in quest. Validates all objectives met (current_count >= required_count). Sets status to Completed. Executes reward_action_json (e.g., give_item).
- `abandon_quest(quest_id: u64)` — player drops active quest. Deletes QuestProgress and ObjectiveProgress rows.

### Trigger Reducers

- `create_trigger(zone_id: u64, trigger_type: TriggerType, x: f32, y: f32, radius: f32, condition_json: String, action_json: String)` — editor places trigger. Admin only.
- `delete_trigger(id: u64)` — remove trigger. Admin only.
- `fire_trigger(trigger_id: u64)` — client reports trigger activation. Server validates: trigger exists, player is in correct zone, player position is within trigger radius, conditions met. Executes action_json.

### Item & Inventory Reducers

- `create_item_def(name: String, description: String, item_type: String, max_stack: u32)` — create item definition. Admin only.
- `delete_item_def(id: u64)` — remove item definition. Admin only.

Internal helpers (not reducers — called by action system):
- `give_item(ctx, player_id, item_id, count)` — add items to inventory. Upserts: if player already has the item, increment quantity (up to max_stack). Otherwise insert new Inventory row.
- `remove_item(ctx, player_id, item_id, count)` — remove items. Decrements quantity; deletes row if quantity hits 0. Returns error if insufficient.

### Objective Auto-Tracking

Integrated into existing and new reducers:

- **KillEnemy**: In the enemy death logic (when enemy health <= 0), check all active QuestProgress rows for the killing player. For each, check QuestObjective rows with type=KillEnemy and matching enemy_def_id. Increment ObjectiveProgress.current_count. If the objective has `loot_on_kill_json`, call `give_item` for the specified drops.
- **CollectItem**: After `give_item`, check active quests for CollectItem objectives matching the item_id. Update ObjectiveProgress.
- **TalkToNpc**: In `interact_with_npc`, check active quests for TalkToNpc objectives matching the npc_id. Update ObjectiveProgress.
- **EnterZone**: In `enter_zone`, check active quests for EnterZone objectives matching the destination zone_id. Update ObjectiveProgress.

---

## 3. Client Systems

### NpcManager (new singleton)

- Subscribes to `Npc` table (zone-filtered).
- Spawns NPC visuals: colored capsules (green for QuestGiver, yellow for Vendor, white for Dialogue) with floating name label.
- Quest-giver indicator: yellow "!" above NPCs that have available quests, "?" for quests ready to turn in.
- Click detection: raycast from camera, if NPC hit, call `interact_with_npc` reducer.
- Cleanup on zone change (same pattern as EnemyManager).

### DialogueUI (new)

- Modal UIToolkit panel (ScreenSpaceOverlay canvas or UIDocument).
- Top section: NPC name + portrait placeholder + current node's npc_text.
- Bottom section: list of DialogueChoice buttons for current node.
- Choices with unmet conditions are hidden (client evaluates condition_json against local QuestProgress/Inventory state).
- Clicking a choice calls `select_dialogue_choice` reducer, then advances to next_node_id. If next_node_id is null, close dialogue.
- Blocks player movement input while open.

### QuestUI (new)

- **Quest Journal** — toggled with J key. UIToolkit panel listing:
  - Active quests with objective checklist and progress counts.
  - Completed quests section (collapsed by default).
- **Quest Accept Popup** — appears when `start_quest` action fires. Shows quest name, description, objectives. Accept/Decline buttons. Accept calls `accept_quest` reducer.
- **Quest Tracker HUD** — small panel in top-right corner showing active quest objectives with progress (e.g., "Kill Goblins: 2/5"). Always visible, updates in real time via ObjectiveProgress callbacks.

### TriggerManager (new singleton)

- Subscribes to `Trigger` table (zone-filtered).
- **OnEnter**: creates invisible SphereCollider GameObjects at trigger positions. Player's collider enters → calls `fire_trigger` reducer. Debounced (won't re-fire for same trigger within 5 seconds).
- **OnInteract**: on player click, checks if clicked position is within any OnInteract trigger's radius. If so, calls `fire_trigger`.
- Handles `play_sound` action client-side (plays AudioClip from Resources). All other actions are server-side.
- Cleanup on zone change.

### InventoryUI (minimal, new)

- Simple UIToolkit panel toggled with I key.
- Vertical list: item name + quantity per row.
- Subscribes to `Inventory` table (player-filtered).
- No drag-and-drop, no grid, no equipment slots — deferred to Group 13.

### Subscription Changes

Add to light tables (unfiltered): `npc_def`, `quest`, `quest_objective`, `item`, `dialogue_node`, `dialogue_choice`

Add to zone-filtered: `npc`, `trigger`

Add to player-filtered (new filter category): `quest_progress`, `objective_progress`, `inventory`

---

## 4. Editor Tools

### NPC Creation Panel (new UIToolkit panel)

- Follows existing panel patterns: MonoBehaviour + UIDocument + USS stylesheet, collapsible, PickingMode.Ignore on wrapper.
- **Definition section**: form with name TextField, npc_type dropdown (QuestGiver/Vendor/Dialogue). Create button calls `create_npc_def` reducer. List of existing definitions with delete buttons.
- **Placement section**: select an NPC definition, click terrain to place (same EntityPlacer raycast pattern). Calls `spawn_npc` reducer.
- NPC instances rendered as colored capsules in the editor scene (green/yellow/white by type) with floating name labels.
- Managed by ToolbarController alongside existing panels.

### Visual Scripting Panel (new UIToolkit panel)

**Generic Node Graph Infrastructure:**
- Canvas: pannable, zoomable VisualElement container. Pan via middle-mouse drag, zoom via scroll wheel.
- Nodes: draggable VisualElements with title bar, typed input/output ports, and inline field editors.
- Edges: drawn via custom `generateVisualContent` callback (same approach as WorldGraphPanel's EdgeDrawer). Edges connect output port → input port.
- Selection: click to select node, drag to multi-select. Delete key removes selected nodes/edges.
- Serialization: graph state serialized to JSON for batch reducer calls.

**Node Types:**

| Node Type | Ports (In → Out) | Inline Fields |
|-----------|-------------------|---------------|
| Dialogue | prev_node → choices[] | npc_text (multiline) |
| Choice | parent_dialogue → next_node | choice_text, condition dropdown, action dropdown |
| Quest | — → objectives[] | name, description, prerequisite quest picker |
| Objective | parent_quest → — | type dropdown, target picker, required_count, reward action |
| Trigger | — → actions[] | zone, position (x/y), radius, type dropdown, condition |
| Condition | — → (attaches to trigger/choice) | type dropdown, parameter fields |
| Action | (from trigger/choice/objective) → — | type dropdown, parameter fields |

**Workflow:**
1. Sidebar lists all dialogue trees / quests / triggers. Click to load onto canvas, or create new.
2. Right-click canvas or drag from node toolbox to add nodes.
3. Drag from output port to input port to create edges.
4. Edit fields inline on the node or in a properties panel.
5. Save button serializes the graph and calls appropriate batch reducers (`create_dialogue_tree`, `create_quest`, `create_trigger`).
6. Unsaved changes indicator (asterisk in panel title).

**Import/Export:**
- **Import from File**: button opens a file picker. Loads a `.quest.json` file onto the canvas. The file format uses human-readable names (e.g., `"forest-ranger"`) instead of numeric IDs — resolved to database IDs on import by looking up NpcDefinition, Item, Quest, and EnemyDefinition tables by name. Missing references flagged as validation errors.
- **Export to File**: button serializes the current canvas graph to a `.quest.json` file. Resolves numeric IDs back to human-readable names for readability and portability.
- **Bulk import**: a "Scan Directory" option imports all `.quest.json` files from a chosen directory, resolving dependencies in topological order (prerequisites before dependents).
- Canonical data lives in SpacetimeDB tables. Files are an authoring convenience and version control format.

Quest file format example:
```json
{
  "name": "Forest Patrol",
  "description": "Clear the goblins from the forest.",
  "prerequisites": ["welcome-to-the-village"],
  "reward": [{"type": "give_item", "item": "cave-key", "count": 1}],
  "dialogue": {
    "npc": "forest-ranger",
    "nodes": [
      {"id": "start", "text": "Halt! The forest is dangerous.", "choices": [
        {"text": "The Elder sent me.", "condition": [{"type": "has_item", "item": "village-map"}], "next": "recognized"},
        {"text": "I can handle myself.", "next": "prove-it"}
      ]},
      {"id": "recognized", "text": "Ah, you carry the Elder's map.", "choices": [
        {"text": "What do you need?", "next": "task"}
      ]},
      {"id": "task", "text": "Kill 3 goblins and bring back 2 teeth.", "choices": [
        {"text": "I'll return with the teeth.", "action": [{"type": "start_quest"}]}
      ]}
    ]
  },
  "objectives": [
    {"description": "Kill Goblins", "type": "KillEnemy", "target": "goblin", "count": 3,
     "loot_on_kill": {"item": "goblin-tooth", "count": 1}},
    {"description": "Collect Goblin Teeth", "type": "CollectItem", "target": "goblin-tooth", "count": 2}
  ]
}
```

**Validation & Error Highlighting:**

Live validation runs on load, on edit (debounced), and on save. Nodes and edges are color-coded:

| Level | Visual | Meaning | Examples |
|-------|--------|---------|---------|
| Error (red) | Red border + red edges | Broken, will not function | Referenced NPC/item/quest doesn't exist in database, orphaned node (no connections), missing required field (empty text, no target), prerequisite quest doesn't exist |
| Warning (yellow) | Yellow border | Suspicious but functional | Terminal dialogue node (no choices — intentional?), objective with count=0, unreachable nodes in graph |
| Valid | Normal border | No issues | — |

Validation checks:
- **Dangling references**: NPC name/ID not found in NpcDefinition, item not in Item table, prerequisite quest not in Quest table, enemy_def not in EnemyDefinition
- **Graph integrity**: orphaned nodes not reachable from root, dialogue choices pointing to nonexistent node IDs, cycles with no exit path (infinite loop)
- **Missing fields**: quest with no objectives, objective with no target, trigger with no action, dialogue node with empty text
- **Cross-quest**: prerequisite dependency cycle (A requires B requires A)

Error panel: collapsible list at the bottom of the Visual Scripting Panel. Each entry shows the error/warning message. Clicking an entry selects and pans to the problematic node. Errors block save with a summary dialog. Warnings allow save with confirmation.

### Item Definition Panel (minimal, added to editor)

- Simple form panel: name, description, item_type (TextField), max_stack (IntegerField).
- List of existing items with delete buttons.
- Calls `create_item_def` / `delete_item_def` reducers.
- Standalone panel, managed by ToolbarController alongside other panels.

---

## 5. Quest Content — 3 Interconnected Chains

### Chain 1: "Welcome to the Village" (Village zone)

**NPC:** Village Elder (QuestGiver), positioned near player spawn point.

**Dialogue tree:**
- Node 1: "Welcome, traveler! Our village has been troubled of late."
  - Choice A: "Tell me about the village." → Node 2
  - Choice B: "I'm ready to help." → Node 3
- Node 2: "We've lived here for generations, but creatures from the forest have grown bold..." → Node 3
- Node 3: "Speak with the Guard at the village gate, then venture into the Forest to see for yourself."
  - Choice: "I'll do it." → (terminal, triggers `start_quest`)

**Objectives:**
1. Talk to the Guard NPC (TalkToNpc, target: Guard npc_id)
2. Enter the Forest zone (EnterZone, target: Forest zone_id)

**Reward:** "Village Map" item via `give_item` action on quest completion.

**Systems tested:** basic quest flow, dialogue branching, TalkToNpc objective, EnterZone objective, give_item action.

---

### Chain 2: "Forest Patrol" (Forest zone)

**Prerequisite:** Chain 1 completed.

**NPC:** Forest Ranger (QuestGiver), positioned at forest entrance.

**Dialogue tree:**
- Node 1: "Halt! The forest is dangerous."
  - Choice A: "The Elder sent me." (condition: has_item Village Map) → Node 2
  - Choice B: "I can handle myself." → Node 3
- Node 2: "Ah, you carry the Elder's map. Good — we need help with the goblin problem."
  - Choice: "What do you need?" → Node 4
- Node 3: "Brave words. Prove them — clear out some goblins."
  - Choice: "Consider it done." → Node 4
- Node 4: "Kill 3 goblins and bring back 2 of their teeth as proof."
  - Choice: "I'll return with the teeth." → (terminal, triggers `start_quest`)

**Objectives:**
1. Kill 3 Goblins (KillEnemy, target: Goblin enemy_def_id, count: 3)
2. Collect 2 Goblin Teeth (CollectItem, target: Goblin Tooth item_id, count: 2)

**Goblin Tooth drops:** Integrated into enemy death logic. When a goblin dies and the killing player has Chain 2 active, the server runs `give_item(goblin_tooth, 1)` directly (checked alongside KillEnemy objective tracking). No trigger needed — the drop is a side effect of the kill, not a spatial event.

**Reward:** "Cave Key" item via `give_item` action on quest completion.

**Systems tested:** prerequisite quests, has_item condition in dialogue, KillEnemy objective, CollectItem objective, item drops.

---

### Chain 3: "The Cave Below" (Cave zone)

**Prerequisite:** Chain 2 completed.

**Entry:** No NPC — triggered by OnEnter at cave entrance.

**OnEnter trigger at cave entrance:**
- Condition: `has_item(Cave Key)` AND `quest_status(chain3, NotStarted)`
- Action: `start_quest(chain3)`

**Objectives:**
1. Explore 2 cave chambers (EnterZone — two OnEnter triggers at key positions within the cave, each increments the objective)
2. Defeat the Cave Boss (KillEnemy, target: Cave Boss enemy_def_id, count: 1)

**Boss spawn:** When objective 1 completes (2/2 chambers explored), an action fires `spawn_entity` to place a boss enemy in the final chamber.

**Reward:** "Hero's Medal" item via `give_item` action on quest completion.

**Completion trigger:** OnEnter at cave exit with condition `quest_status(chain3, Completed)` → action `play_sound(quest_complete)`. (Could also trigger a congratulatory dialogue via a placed NPC or a special trigger-attached dialogue.)

**Systems tested:** trigger-initiated quests (no NPC), has_item condition, spawn_entity action, cross-objective dependencies (boss spawns after exploration), play_sound action.

---

## 6. PROGRESS.md Updates

Group 12 tasks updated to reflect this design:

- [x] `Trigger` table (trigger_type, zone_id, conditions, actions) → covered above
- [x] `OnEnter` and `OnInteract` trigger types → TriggerType enum
- [x] Condition system (has_item, quest_status, player_level) → condition_json
- [x] Action system (spawn_entity, play_sound, give_item) → action_json + start_quest
- [x] Visual Scripting UI for triggers → unified Visual Scripting Panel (covers triggers + quests + dialogue)
- [x] `Quest` table (objectives, rewards, prerequisites) → Quest + QuestObjective tables
- [x] `QuestProgress` table (player_id, quest_id, objective_states) → QuestProgress + ObjectiveProgress tables
- [x] Quest reducers: start, update_objective, complete → accept_quest, complete_quest, abandon_quest + auto-tracking
- [x] Quest UI — journal and dialogue tree → QuestUI (journal + tracker + accept popup) + DialogueUI
- [x] 3 interconnected quest chains → Village → Forest → Cave
- [x] Editor tool: NPC creation panel → NPC Creation Panel
- [x] Editor tool: Quest designer panel → unified Visual Scripting Panel

**Added from Group 13 (minimal):**
- Item table (definitions only)
- Inventory table (player_id, item_id, quantity)
- Item Definition Panel in editor
- Basic InventoryUI in client (list view)
