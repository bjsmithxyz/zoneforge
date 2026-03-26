# Phase 5 Group 12 — Triggers, Quests & NPC Authoring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the trigger system, quest system, NPC authoring, dialogue trees, item/inventory backbone, unified visual scripting editor, and 3 interconnected quest chains.

**Architecture:** Server-first — all new tables and reducers are added to `server/spacetimedb/src/lib.rs`. Client managers (NpcManager, TriggerManager) follow the existing EnemyManager singleton pattern. Editor panels follow the existing UIToolkit MonoBehaviour pattern with UXML templates. Conditions/actions are stored as JSON strings and parsed with `serde_json` in Rust reducers.

**Tech Stack:** Rust (SpacetimeDB 2.x WASM module), C# (Unity 2022.3 LTS, UIToolkit), SpacetimeDB C# SDK

**Design Spec:** `docs/superpowers/specs/2026-03-26-triggers-quests-npc-design.md`

---

## File Map

### Server (all in `server/spacetimedb/src/lib.rs`)
- Modify: `server/spacetimedb/src/lib.rs` — add 4 enums, 11 tables, ~18 reducers, condition/action engine
- Modify: `server/spacetimedb/Cargo.toml` — add `serde`, `serde_json` dependencies

### Client (new files under `client/Assets/Scripts/`)
- Modify: `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs` — add subscriptions + events
- Create: `client/Assets/Scripts/Npc/NpcManager.cs` — NPC singleton manager
- Create: `client/Assets/Scripts/Runtime/TriggerManager.cs` — trigger singleton manager
- Create: `client/Assets/Scripts/UI/DialogueUI.cs` — modal dialogue panel
- Create: `client/Assets/Scripts/UI/QuestUI.cs` — journal, tracker HUD, accept popup
- Create: `client/Assets/Scripts/UI/InventoryUI.cs` — simple item list

### Editor (new files under `editor/Assets/`)
- Modify: `editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs` — add subscriptions + events
- Modify: `editor/Assets/Scripts/Runtime/ToolbarController.cs` — add new panel toggles
- Modify: `editor/Assets/UI/ToolbarController.uxml` — add new buttons
- Create: `editor/Assets/Scripts/Runtime/NpcCreationPanel.cs`
- Create: `editor/Assets/UI/NpcCreationPanel.uxml`
- Create: `editor/Assets/Scripts/Runtime/NpcRenderer.cs` — spawns NPC capsules in editor
- Create: `editor/Assets/Scripts/Runtime/NpcPlacer.cs` — click-to-place NPC
- Create: `editor/Assets/Scripts/Runtime/ItemDefinitionPanel.cs`
- Create: `editor/Assets/UI/ItemDefinitionPanel.uxml`
- Create: `editor/Assets/Scripts/Runtime/VisualScriptingPanel.cs`
- Create: `editor/Assets/UI/VisualScriptingPanel.uxml`
- Create: `editor/Assets/Scripts/Runtime/NodeGraph/NodeGraphCanvas.cs` — pan/zoom canvas
- Create: `editor/Assets/Scripts/Runtime/NodeGraph/GraphNode.cs` — draggable node element
- Create: `editor/Assets/Scripts/Runtime/NodeGraph/GraphPort.cs` — typed port element
- Create: `editor/Assets/Scripts/Runtime/NodeGraph/GraphEdge.cs` — edge drawer element
- Create: `editor/Assets/Scripts/Runtime/NodeGraph/GraphValidator.cs` — validation engine
- Create: `editor/Assets/Scripts/Runtime/NodeGraph/QuestFileSerializer.cs` — .quest.json import/export

### Content
- Create: `editor/Assets/StreamingAssets/quests/welcome-to-the-village.quest.json`
- Create: `editor/Assets/StreamingAssets/quests/forest-patrol.quest.json`
- Create: `editor/Assets/StreamingAssets/quests/the-cave-below.quest.json`

---

## Task 1: Server — Enums, Item/Inventory, and NPC Tables + Reducers

**Files:**
- Modify: `server/spacetimedb/Cargo.toml`
- Modify: `server/spacetimedb/src/lib.rs`

### Steps

- [ ] **Step 1: Add serde dependencies to Cargo.toml**

Add `serde` and `serde_json` for JSON condition/action parsing:

```toml
[dependencies]
spacetimedb = "2.0"
log = "0.4"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

- [ ] **Step 2: Add new enums after existing enums (after line 30)**

Add after the `EnemyType` enum in `lib.rs`:

```rust
#[derive(SpacetimeType, Clone, Debug, PartialEq)]
pub enum NpcType {
    QuestGiver,
    Vendor,
    Dialogue,
}

#[derive(SpacetimeType, Clone, Debug, PartialEq)]
pub enum TriggerType {
    OnEnter,
    OnInteract,
}

#[derive(SpacetimeType, Clone, Debug, PartialEq)]
pub enum QuestStatus {
    NotStarted,
    Active,
    Completed,
    Failed,
}

#[derive(SpacetimeType, Clone, Debug, PartialEq)]
pub enum ObjectiveType {
    KillEnemy,
    CollectItem,
    TalkToNpc,
    EnterZone,
}
```

- [ ] **Step 3: Add Item and Inventory tables (after EntityInstance, ~line 1005)**

```rust
// ── Item & Inventory (minimal — pulled forward from Group 13) ──

#[table(accessor = item, public)]
pub struct Item {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub name: String,
    pub description: String,
    pub item_type: String,
    pub max_stack: u32,
}

#[table(accessor = inventory, public)]
pub struct Inventory {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    #[index(btree)]
    pub player_id: u64,
    pub item_id: u64,
    pub quantity: u32,
}
```

- [ ] **Step 4: Add NpcDefinition and Npc tables**

```rust
// ── NPC System ──

#[table(accessor = npc_def, public)]
pub struct NpcDefinition {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub name: String,
    pub npc_type: NpcType,
    pub default_dialogue_tree_id: u64,
}

#[table(accessor = npc, public)]
pub struct Npc {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    #[index(btree)]
    pub zone_id: u64,
    pub npc_def_id: u64,
    pub position_x: f32,
    pub position_y: f32,
    pub name: String,
    pub dialogue_tree_id: u64,
}
```

- [ ] **Step 5: Add Item/Inventory helper functions and reducers**

```rust
// ── Item & Inventory helpers (internal, not reducers) ──

fn give_item_to_player(
    ctx: &ReducerContext,
    player_id: u64,
    item_id: u64,
    count: u32,
) -> Result<(), String> {
    let item = ctx.db.item().id().find(&item_id)
        .ok_or_else(|| format!("Item {} not found", item_id))?;
    // Upsert: if player already has this item, increment quantity
    if let Some(existing) = ctx.db.inventory()
        .player_id()
        .filter(&player_id)
        .find(|inv| inv.item_id == item_id)
    {
        let new_qty = (existing.quantity + count).min(item.max_stack);
        ctx.db.inventory().id().update(Inventory {
            quantity: new_qty,
            ..existing
        });
    } else {
        ctx.db.inventory().insert(Inventory {
            id: 0,
            player_id,
            item_id,
            quantity: count.min(item.max_stack),
        });
    }
    log::info!("give_item: player={} item={} count={}", player_id, item_id, count);
    Ok(())
}

fn remove_item_from_player(
    ctx: &ReducerContext,
    player_id: u64,
    item_id: u64,
    count: u32,
) -> Result<(), String> {
    let existing = ctx.db.inventory()
        .player_id()
        .filter(&player_id)
        .find(|inv| inv.item_id == item_id)
        .ok_or_else(|| format!("Player {} does not have item {}", player_id, item_id))?;
    if existing.quantity < count {
        return Err(format!("Insufficient quantity ({}/{})", existing.quantity, count));
    }
    if existing.quantity == count {
        ctx.db.inventory().id().delete(&existing.id);
    } else {
        ctx.db.inventory().id().update(Inventory {
            quantity: existing.quantity - count,
            ..existing
        });
    }
    Ok(())
}

fn player_has_item(ctx: &ReducerContext, player_id: u64, item_id: u64, count: u32) -> bool {
    ctx.db.inventory()
        .player_id()
        .filter(&player_id)
        .find(|inv| inv.item_id == item_id && inv.quantity >= count)
        .is_some()
}

// ── Item definition reducers ──

#[reducer]
pub fn create_item_def(
    ctx: &ReducerContext,
    name: String,
    description: String,
    item_type: String,
    max_stack: u32,
) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized: admin only".to_string()); }
    if name.is_empty() || name.len() > 128 { return Err("Invalid item name".to_string()); }
    if description.len() > 512 { return Err("Description too long".to_string()); }
    if item_type.is_empty() || item_type.len() > 64 { return Err("Invalid item_type".to_string()); }
    if max_stack == 0 || max_stack > 9999 { return Err("max_stack out of range [1, 9999]".to_string()); }
    ctx.db.item().insert(Item { id: 0, name, description, item_type, max_stack });
    Ok(())
}

#[reducer]
pub fn delete_item_def(ctx: &ReducerContext, item_id: u64) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized: admin only".to_string()); }
    ctx.db.item().id().find(&item_id)
        .ok_or_else(|| format!("Item {} not found", item_id))?;
    ctx.db.item().id().delete(&item_id);
    Ok(())
}
```

- [ ] **Step 6: Add NPC reducers**

```rust
// ── NPC reducers ──

#[reducer]
pub fn create_npc_def(
    ctx: &ReducerContext,
    name: String,
    npc_type: NpcType,
) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized: admin only".to_string()); }
    if name.is_empty() || name.len() > 128 { return Err("Invalid NPC name".to_string()); }
    ctx.db.npc_def().insert(NpcDefinition {
        id: 0,
        name,
        npc_type,
        default_dialogue_tree_id: 0,
    });
    Ok(())
}

#[reducer]
pub fn delete_npc_def(ctx: &ReducerContext, def_id: u64) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized: admin only".to_string()); }
    ctx.db.npc_def().id().find(&def_id)
        .ok_or_else(|| format!("NpcDef {} not found", def_id))?;
    ctx.db.npc_def().id().delete(&def_id);
    Ok(())
}

#[reducer]
pub fn spawn_npc(
    ctx: &ReducerContext,
    zone_id: u64,
    npc_def_id: u64,
    x: f32,
    y: f32,
    dialogue_tree_id: u64,
) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized: admin only".to_string()); }
    let zone = ctx.db.zone().id().find(&zone_id)
        .ok_or_else(|| format!("Zone {} not found", zone_id))?;
    let def = ctx.db.npc_def().id().find(&npc_def_id)
        .ok_or_else(|| format!("NpcDef {} not found", npc_def_id))?;
    if !x.is_finite() || !y.is_finite()
        || x < 0.0 || x > zone.terrain_width as f32
        || y < 0.0 || y > zone.terrain_height as f32
    {
        return Err("NPC position out of zone bounds".to_string());
    }
    ctx.db.npc().insert(Npc {
        id: 0,
        zone_id,
        npc_def_id,
        position_x: x,
        position_y: y,
        name: def.name.clone(),
        dialogue_tree_id,
    });
    Ok(())
}

#[reducer]
pub fn despawn_npc(ctx: &ReducerContext, npc_id: u64) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized: admin only".to_string()); }
    ctx.db.npc().id().find(&npc_id)
        .ok_or_else(|| format!("Npc {} not found", npc_id))?;
    ctx.db.npc().id().delete(&npc_id);
    Ok(())
}

#[reducer]
pub fn interact_with_npc(ctx: &ReducerContext, npc_id: u64) -> Result<(), String> {
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or("Player not found")?;
    let npc = ctx.db.npc().id().find(&npc_id)
        .ok_or_else(|| format!("NPC {} not found", npc_id))?;
    if player.zone_id != npc.zone_id {
        return Err("NPC is not in your zone".to_string());
    }
    // Proximity check: must be within 3 units
    let d2 = dist_sq(player.position_x, player.position_y, npc.position_x, npc.position_y);
    if d2 > 9.0 {
        return Err("Too far from NPC".to_string());
    }
    // Auto-track TalkToNpc objectives
    track_objective(ctx, player.id, ObjectiveType::TalkToNpc, npc.npc_def_id);
    log::info!("interact_with_npc: player={} npc={}", player.id, npc_id);
    Ok(())
}
```

- [ ] **Step 7: Build to verify tables and reducers compile**

Run from `server/`:
```bash
spacetime build
```

Expected: Build succeeds (ignore `wasm-opt` warning). `track_objective` will cause a compile error since it's not defined yet — add a stub:

```rust
fn track_objective(_ctx: &ReducerContext, _player_id: u64, _obj_type: ObjectiveType, _target_id: u64) {
    // Stub — implemented in Task 4
}
```

- [ ] **Step 8: Commit**

```bash
git add server/spacetimedb/Cargo.toml server/spacetimedb/src/lib.rs
git commit -m "feat(server): add Item, Inventory, NPC tables and reducers"
```

---

## Task 2: Server — Dialogue Tables + Reducers

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

### Steps

- [ ] **Step 1: Add DialogueNode and DialogueChoice tables**

Add after the Npc table:

```rust
// ── Dialogue System ──

#[table(accessor = dialogue_node, public)]
pub struct DialogueNode {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    #[index(btree)]
    pub tree_id: u64,
    pub npc_text: String,
    pub sort_order: u32,
}

#[table(accessor = dialogue_choice, public)]
pub struct DialogueChoice {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    #[index(btree)]
    pub node_id: u64,
    pub choice_text: String,
    pub next_node_id: u64,  // 0 = terminal (no next node)
    pub condition_json: String,  // JSON array of conditions, empty string = always true
    pub action_json: String,     // JSON array of actions, empty string = no actions
}
```

- [ ] **Step 2: Add dialogue reducers**

```rust
// ── Dialogue reducers ──

#[reducer]
pub fn create_dialogue_tree(
    ctx: &ReducerContext,
    tree_id: u64,
    nodes_json: String,
    choices_json: String,
) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized: admin only".to_string()); }

    // Determine tree_id: if 0, auto-assign next available
    let actual_tree_id = if tree_id == 0 {
        let max_id = ctx.db.dialogue_node().iter()
            .map(|n| n.tree_id)
            .max()
            .unwrap_or(0);
        max_id + 1
    } else {
        // Delete existing tree data for overwrite
        let old_nodes: Vec<u64> = ctx.db.dialogue_node()
            .tree_id()
            .filter(&tree_id)
            .map(|n| n.id)
            .collect();
        for node_id in &old_nodes {
            // Delete choices for this node
            let choice_ids: Vec<u64> = ctx.db.dialogue_choice()
                .node_id()
                .filter(node_id)
                .map(|c| c.id)
                .collect();
            for cid in choice_ids {
                ctx.db.dialogue_choice().id().delete(&cid);
            }
            ctx.db.dialogue_node().id().delete(node_id);
        }
        tree_id
    };

    // Parse nodes JSON: [{"npc_text": "...", "sort_order": 0}, ...]
    let nodes: Vec<serde_json::Value> = serde_json::from_str(&nodes_json)
        .map_err(|e| format!("Invalid nodes_json: {}", e))?;

    // Insert nodes and track old_index → new_id mapping
    let mut node_id_map: Vec<u64> = Vec::new();
    for node_val in &nodes {
        let npc_text = node_val["npc_text"].as_str().unwrap_or("").to_string();
        let sort_order = node_val["sort_order"].as_u64().unwrap_or(0) as u32;
        if npc_text.is_empty() { return Err("Dialogue node has empty npc_text".to_string()); }
        let row = ctx.db.dialogue_node().insert(DialogueNode {
            id: 0,
            tree_id: actual_tree_id,
            npc_text,
            sort_order,
        });
        node_id_map.push(row.id);
    }

    // Parse choices JSON: [{"node_index": 0, "choice_text": "...", "next_node_index": 1, "condition_json": "", "action_json": ""}, ...]
    let choices: Vec<serde_json::Value> = serde_json::from_str(&choices_json)
        .map_err(|e| format!("Invalid choices_json: {}", e))?;

    for choice_val in &choices {
        let node_index = choice_val["node_index"].as_u64().unwrap_or(0) as usize;
        let choice_text = choice_val["choice_text"].as_str().unwrap_or("").to_string();
        let next_node_index = choice_val["next_node_index"].as_i64().unwrap_or(-1);
        let condition_json = choice_val["condition_json"].as_str().unwrap_or("").to_string();
        let action_json = choice_val["action_json"].as_str().unwrap_or("").to_string();

        if node_index >= node_id_map.len() {
            return Err(format!("Choice references invalid node_index {}", node_index));
        }
        let node_id = node_id_map[node_index];
        let next_node_id = if next_node_index < 0 || next_node_index as usize >= node_id_map.len() {
            0 // Terminal
        } else {
            node_id_map[next_node_index as usize]
        };

        ctx.db.dialogue_choice().insert(DialogueChoice {
            id: 0,
            node_id,
            choice_text,
            next_node_id,
            condition_json,
            action_json,
        });
    }

    log::info!("create_dialogue_tree: tree_id={} nodes={} choices={}", actual_tree_id, nodes.len(), choices.len());
    Ok(())
}

#[reducer]
pub fn delete_dialogue_tree(ctx: &ReducerContext, tree_id: u64) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized: admin only".to_string()); }
    let node_ids: Vec<u64> = ctx.db.dialogue_node()
        .tree_id()
        .filter(&tree_id)
        .map(|n| n.id)
        .collect();
    if node_ids.is_empty() {
        return Err(format!("No dialogue tree with id {}", tree_id));
    }
    for node_id in &node_ids {
        let choice_ids: Vec<u64> = ctx.db.dialogue_choice()
            .node_id()
            .filter(node_id)
            .map(|c| c.id)
            .collect();
        for cid in choice_ids {
            ctx.db.dialogue_choice().id().delete(&cid);
        }
        ctx.db.dialogue_node().id().delete(node_id);
    }
    log::info!("delete_dialogue_tree: tree_id={}", tree_id);
    Ok(())
}

#[reducer]
pub fn select_dialogue_choice(
    ctx: &ReducerContext,
    npc_id: u64,
    choice_id: u64,
) -> Result<(), String> {
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or("Player not found")?;
    let npc = ctx.db.npc().id().find(&npc_id)
        .ok_or_else(|| format!("NPC {} not found", npc_id))?;
    if player.zone_id != npc.zone_id {
        return Err("NPC is not in your zone".to_string());
    }

    let choice = ctx.db.dialogue_choice().id().find(&choice_id)
        .ok_or_else(|| format!("Choice {} not found", choice_id))?;

    // Validate choice belongs to this NPC's dialogue tree
    let node = ctx.db.dialogue_node().id().find(&choice.node_id)
        .ok_or("Dialogue node not found")?;
    if node.tree_id != npc.dialogue_tree_id {
        return Err("Choice does not belong to this NPC's dialogue tree".to_string());
    }

    // Evaluate conditions
    if !choice.condition_json.is_empty() {
        if !evaluate_conditions(ctx, player.id, &choice.condition_json)? {
            return Err("Conditions not met for this choice".to_string());
        }
    }

    // Execute actions
    if !choice.action_json.is_empty() {
        execute_actions(ctx, player.id, player.zone_id, &choice.action_json)?;
    }

    log::info!("select_dialogue_choice: player={} npc={} choice={}", player.id, npc_id, choice_id);
    Ok(())
}
```

- [ ] **Step 3: Build to verify**

```bash
cd server && spacetime build
```

Expected: Compile errors for `evaluate_conditions` and `execute_actions` — add stubs:

```rust
fn evaluate_conditions(_ctx: &ReducerContext, _player_id: u64, _json: &str) -> Result<bool, String> {
    Ok(true) // Stub — implemented in Task 3
}

fn execute_actions(_ctx: &ReducerContext, _player_id: u64, _zone_id: u64, _json: &str) -> Result<(), String> {
    Ok(()) // Stub — implemented in Task 3
}
```

- [ ] **Step 4: Commit**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add Dialogue tables and reducers"
```

---

## Task 3: Server — Condition/Action Engine + Quest Tables + Reducers

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

### Steps

- [ ] **Step 1: Add Quest tables**

```rust
// ── Quest System ──

#[table(accessor = quest, public)]
pub struct Quest {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub name: String,
    pub description: String,
    pub prerequisite_quest_ids_json: String,  // JSON array of u64 quest IDs
    pub reward_action_json: String,           // JSON array of actions
}

#[table(accessor = quest_objective, public)]
pub struct QuestObjective {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    #[index(btree)]
    pub quest_id: u64,
    pub description: String,
    pub objective_type: ObjectiveType,
    pub target_json: String,    // e.g., {"enemy_def_id": 5} or {"zone_id": 2}
    pub required_count: u32,
    pub sort_order: u32,
    pub loot_on_kill_json: String,  // e.g., {"item_id": 3, "count": 1} or empty
}

#[table(accessor = quest_progress, public)]
pub struct QuestProgress {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    #[index(btree)]
    pub player_id: u64,
    pub quest_id: u64,
    pub status: QuestStatus,
}

#[table(accessor = objective_progress, public)]
pub struct ObjectiveProgress {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    #[index(btree)]
    pub player_id: u64,
    pub objective_id: u64,
    pub current_count: u32,
}
```

- [ ] **Step 2: Implement the condition evaluation engine**

Replace the `evaluate_conditions` stub:

```rust
fn evaluate_conditions(ctx: &ReducerContext, player_id: u64, json: &str) -> Result<bool, String> {
    let conditions: Vec<serde_json::Value> = serde_json::from_str(json)
        .map_err(|e| format!("Invalid condition JSON: {}", e))?;

    for cond in &conditions {
        let ctype = cond["type"].as_str().unwrap_or("");
        match ctype {
            "quest_status" => {
                let quest_id = cond["quest_id"].as_u64().unwrap_or(0);
                let required_status = cond["status"].as_str().unwrap_or("Completed");
                let has_status = ctx.db.quest_progress()
                    .player_id()
                    .filter(&player_id)
                    .any(|qp| {
                        qp.quest_id == quest_id && match required_status {
                            "NotStarted" => qp.status == QuestStatus::NotStarted,
                            "Active" => qp.status == QuestStatus::Active,
                            "Completed" => qp.status == QuestStatus::Completed,
                            "Failed" => qp.status == QuestStatus::Failed,
                            _ => false,
                        }
                    });
                // If quest_status is NotStarted, also true if no progress row exists
                if !has_status {
                    if required_status == "NotStarted" {
                        let has_any = ctx.db.quest_progress()
                            .player_id()
                            .filter(&player_id)
                            .any(|qp| qp.quest_id == quest_id);
                        if has_any { return Ok(false); }
                        // No row = NotStarted, condition passes
                    } else {
                        return Ok(false);
                    }
                }
            }
            "has_item" => {
                let item_id = cond["item_id"].as_u64().unwrap_or(0);
                let count = cond["count"].as_u64().unwrap_or(1) as u32;
                if !player_has_item(ctx, player_id, item_id, count) {
                    return Ok(false);
                }
            }
            "player_level" => {
                // No leveling system yet — always passes
            }
            other => {
                log::warn!("Unknown condition type: {}", other);
                return Ok(false);
            }
        }
    }
    Ok(true)
}
```

- [ ] **Step 3: Implement the action execution engine**

Replace the `execute_actions` stub:

```rust
fn execute_actions(
    ctx: &ReducerContext,
    player_id: u64,
    zone_id: u64,
    json: &str,
) -> Result<(), String> {
    let actions: Vec<serde_json::Value> = serde_json::from_str(json)
        .map_err(|e| format!("Invalid action JSON: {}", e))?;

    for action in &actions {
        let atype = action["type"].as_str().unwrap_or("");
        match atype {
            "give_item" => {
                let item_id = action["item_id"].as_u64().unwrap_or(0);
                let count = action["count"].as_u64().unwrap_or(1) as u32;
                give_item_to_player(ctx, player_id, item_id, count)?;
            }
            "start_quest" => {
                let quest_id = action["quest_id"].as_u64().unwrap_or(0);
                start_quest_for_player(ctx, player_id, quest_id)?;
            }
            "spawn_entity" => {
                let sz = action["zone_id"].as_u64().unwrap_or(zone_id);
                let prefab = action["prefab"].as_str().unwrap_or("unknown").to_string();
                let x = action["x"].as_f64().unwrap_or(0.0) as f32;
                let y = action["y"].as_f64().unwrap_or(0.0) as f32;
                // Spawn as enemy if prefab starts with "boss_" or "enemy_", otherwise as entity
                if prefab.starts_with("boss_") || prefab.starts_with("enemy_") {
                    // Look up enemy def by prefab_name
                    if let Some(def) = ctx.db.enemy_def().iter().find(|d| d.prefab_name == prefab) {
                        ctx.db.enemy().insert(Enemy {
                            id: 0,
                            zone_id: sz,
                            spawn_point_id: None,
                            enemy_def_id: def.id,
                            position_x: x,
                            position_y: y,
                            home_x: x,
                            home_y: y,
                            health: def.max_health,
                            ai_state: AiState::Idle,
                            target_player_id: None,
                            last_attack_us: 0,
                            is_dead: false,
                        });
                    }
                } else {
                    ctx.db.entity_instance().insert(EntityInstance {
                        id: 0,
                        zone_id: sz,
                        prefab_name: prefab,
                        position_x: x,
                        position_y: y,
                        elevation: 0.0,
                        entity_type: "Prop".to_string(),
                    });
                }
            }
            "play_sound" => {
                // Client-side only — server ignores but doesn't error
                log::info!("play_sound action: {}", action["sound_id"].as_str().unwrap_or(""));
            }
            other => {
                log::warn!("Unknown action type: {}", other);
            }
        }
    }
    Ok(())
}
```

- [ ] **Step 4: Add quest helper and reducers**

```rust
// ── Quest helpers ──

fn start_quest_for_player(
    ctx: &ReducerContext,
    player_id: u64,
    quest_id: u64,
) -> Result<(), String> {
    let quest = ctx.db.quest().id().find(&quest_id)
        .ok_or_else(|| format!("Quest {} not found", quest_id))?;

    // Check not already started
    if ctx.db.quest_progress()
        .player_id()
        .filter(&player_id)
        .any(|qp| qp.quest_id == quest_id)
    {
        return Err("Quest already started or completed".to_string());
    }

    // Check prerequisites
    if !quest.prerequisite_quest_ids_json.is_empty() {
        let prereqs: Vec<u64> = serde_json::from_str(&quest.prerequisite_quest_ids_json)
            .map_err(|e| format!("Invalid prerequisite JSON: {}", e))?;
        for prereq_id in &prereqs {
            let completed = ctx.db.quest_progress()
                .player_id()
                .filter(&player_id)
                .any(|qp| qp.quest_id == *prereq_id && qp.status == QuestStatus::Completed);
            if !completed {
                return Err(format!("Prerequisite quest {} not completed", prereq_id));
            }
        }
    }

    // Create progress rows
    ctx.db.quest_progress().insert(QuestProgress {
        id: 0,
        player_id,
        quest_id,
        status: QuestStatus::Active,
    });

    let objectives: Vec<QuestObjective> = ctx.db.quest_objective()
        .quest_id()
        .filter(&quest_id)
        .collect();
    for obj in &objectives {
        ctx.db.objective_progress().insert(ObjectiveProgress {
            id: 0,
            player_id,
            objective_id: obj.id,
            current_count: 0,
        });
    }

    log::info!("start_quest: player={} quest={}", player_id, quest_id);
    Ok(())
}

// ── Quest reducers ──

#[reducer]
pub fn create_quest(
    ctx: &ReducerContext,
    name: String,
    description: String,
    objectives_json: String,
    prerequisite_quest_ids_json: String,
    reward_action_json: String,
) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized: admin only".to_string()); }
    if name.is_empty() || name.len() > 128 { return Err("Invalid quest name".to_string()); }
    if description.len() > 1024 { return Err("Description too long".to_string()); }

    let quest_row = ctx.db.quest().insert(Quest {
        id: 0,
        name: name.clone(),
        description,
        prerequisite_quest_ids_json,
        reward_action_json,
    });

    // Parse objectives: [{"description": "...", "objective_type": "KillEnemy", "target_json": "{...}", "required_count": 3, "sort_order": 0, "loot_on_kill_json": ""}, ...]
    let objectives: Vec<serde_json::Value> = serde_json::from_str(&objectives_json)
        .map_err(|e| format!("Invalid objectives_json: {}", e))?;

    for obj_val in &objectives {
        let desc = obj_val["description"].as_str().unwrap_or("").to_string();
        let obj_type_str = obj_val["objective_type"].as_str().unwrap_or("");
        let objective_type = match obj_type_str {
            "KillEnemy" => ObjectiveType::KillEnemy,
            "CollectItem" => ObjectiveType::CollectItem,
            "TalkToNpc" => ObjectiveType::TalkToNpc,
            "EnterZone" => ObjectiveType::EnterZone,
            other => return Err(format!("Unknown objective type: {}", other)),
        };
        let target_json = obj_val["target_json"].as_str().unwrap_or("{}").to_string();
        let required_count = obj_val["required_count"].as_u64().unwrap_or(1) as u32;
        let sort_order = obj_val["sort_order"].as_u64().unwrap_or(0) as u32;
        let loot_on_kill_json = obj_val["loot_on_kill_json"].as_str().unwrap_or("").to_string();

        ctx.db.quest_objective().insert(QuestObjective {
            id: 0,
            quest_id: quest_row.id,
            description: desc,
            objective_type,
            target_json,
            required_count,
            sort_order,
            loot_on_kill_json,
        });
    }

    log::info!("create_quest: name='{}' id={} objectives={}", name, quest_row.id, objectives.len());
    Ok(())
}

#[reducer]
pub fn delete_quest(ctx: &ReducerContext, quest_id: u64) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized: admin only".to_string()); }
    ctx.db.quest().id().find(&quest_id)
        .ok_or_else(|| format!("Quest {} not found", quest_id))?;
    // Delete all objectives
    let obj_ids: Vec<u64> = ctx.db.quest_objective()
        .quest_id()
        .filter(&quest_id)
        .map(|o| o.id)
        .collect();
    for oid in obj_ids {
        ctx.db.quest_objective().id().delete(&oid);
    }
    ctx.db.quest().id().delete(&quest_id);
    Ok(())
}

#[reducer]
pub fn accept_quest(ctx: &ReducerContext, quest_id: u64) -> Result<(), String> {
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or("Player not found")?;
    start_quest_for_player(ctx, player.id, quest_id)
}

#[reducer]
pub fn complete_quest(ctx: &ReducerContext, quest_id: u64) -> Result<(), String> {
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or("Player not found")?;

    let qp = ctx.db.quest_progress()
        .player_id()
        .filter(&player.id)
        .find(|qp| qp.quest_id == quest_id && qp.status == QuestStatus::Active)
        .ok_or("Quest not active")?;

    // Verify all objectives met
    let objectives: Vec<QuestObjective> = ctx.db.quest_objective()
        .quest_id()
        .filter(&quest_id)
        .collect();
    for obj in &objectives {
        let progress = ctx.db.objective_progress()
            .player_id()
            .filter(&player.id)
            .find(|op| op.objective_id == obj.id);
        let count = progress.map(|p| p.current_count).unwrap_or(0);
        if count < obj.required_count {
            return Err(format!("Objective '{}' not complete ({}/{})", obj.description, count, obj.required_count));
        }
    }

    // Mark completed
    ctx.db.quest_progress().id().update(QuestProgress {
        status: QuestStatus::Completed,
        ..qp
    });

    // Execute rewards
    let quest = ctx.db.quest().id().find(&quest_id).unwrap();
    if !quest.reward_action_json.is_empty() {
        execute_actions(ctx, player.id, player.zone_id, &quest.reward_action_json)?;
    }

    log::info!("complete_quest: player={} quest={}", player.id, quest_id);
    Ok(())
}

#[reducer]
pub fn abandon_quest(ctx: &ReducerContext, quest_id: u64) -> Result<(), String> {
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or("Player not found")?;

    let qp = ctx.db.quest_progress()
        .player_id()
        .filter(&player.id)
        .find(|qp| qp.quest_id == quest_id && qp.status == QuestStatus::Active)
        .ok_or("Quest not active")?;

    // Delete objective progress rows
    let objectives: Vec<QuestObjective> = ctx.db.quest_objective()
        .quest_id()
        .filter(&quest_id)
        .collect();
    for obj in &objectives {
        if let Some(op) = ctx.db.objective_progress()
            .player_id()
            .filter(&player.id)
            .find(|op| op.objective_id == obj.id)
        {
            ctx.db.objective_progress().id().delete(&op.id);
        }
    }

    ctx.db.quest_progress().id().delete(&qp.id);
    log::info!("abandon_quest: player={} quest={}", player.id, quest_id);
    Ok(())
}
```

- [ ] **Step 5: Build to verify**

```bash
cd server && spacetime build
```

Expected: Build succeeds.

- [ ] **Step 6: Commit**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add Quest tables, condition/action engine, quest reducers"
```

---

## Task 4: Server — Trigger Tables, Reducers, and Objective Auto-Tracking

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

### Steps

- [ ] **Step 1: Add Trigger table**

```rust
// ── Trigger System ──

#[table(accessor = trigger, public)]
pub struct Trigger {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    #[index(btree)]
    pub zone_id: u64,
    pub trigger_type: TriggerType,
    pub position_x: f32,
    pub position_y: f32,
    pub radius: f32,
    pub condition_json: String,
    pub action_json: String,
}
```

- [ ] **Step 2: Add trigger reducers**

```rust
// ── Trigger reducers ──

#[reducer]
pub fn create_trigger(
    ctx: &ReducerContext,
    zone_id: u64,
    trigger_type: TriggerType,
    x: f32,
    y: f32,
    radius: f32,
    condition_json: String,
    action_json: String,
) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized: admin only".to_string()); }
    let zone = ctx.db.zone().id().find(&zone_id)
        .ok_or_else(|| format!("Zone {} not found", zone_id))?;
    if !x.is_finite() || !y.is_finite() || !radius.is_finite() {
        return Err("Non-finite position/radius values".to_string());
    }
    if x < 0.0 || x > zone.terrain_width as f32 || y < 0.0 || y > zone.terrain_height as f32 {
        return Err("Trigger position out of zone bounds".to_string());
    }
    if radius <= 0.0 || radius > 100.0 {
        return Err("Radius out of range (0, 100]".to_string());
    }
    ctx.db.trigger().insert(Trigger {
        id: 0,
        zone_id,
        trigger_type,
        position_x: x,
        position_y: y,
        radius,
        condition_json,
        action_json,
    });
    Ok(())
}

#[reducer]
pub fn delete_trigger(ctx: &ReducerContext, trigger_id: u64) -> Result<(), String> {
    if !is_admin(ctx) { return Err("Not authorized: admin only".to_string()); }
    ctx.db.trigger().id().find(&trigger_id)
        .ok_or_else(|| format!("Trigger {} not found", trigger_id))?;
    ctx.db.trigger().id().delete(&trigger_id);
    Ok(())
}

#[reducer]
pub fn fire_trigger(ctx: &ReducerContext, trigger_id: u64) -> Result<(), String> {
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or("Player not found")?;
    let trigger = ctx.db.trigger().id().find(&trigger_id)
        .ok_or_else(|| format!("Trigger {} not found", trigger_id))?;

    // Validate zone
    if player.zone_id != trigger.zone_id {
        return Err("Trigger is not in your zone".to_string());
    }

    // Validate proximity
    let d2 = dist_sq(player.position_x, player.position_y, trigger.position_x, trigger.position_y);
    if d2 > trigger.radius * trigger.radius {
        return Err("Too far from trigger".to_string());
    }

    // Evaluate conditions
    if !trigger.condition_json.is_empty() {
        if !evaluate_conditions(ctx, player.id, &trigger.condition_json)? {
            return Err("Trigger conditions not met".to_string());
        }
    }

    // Execute actions
    if !trigger.action_json.is_empty() {
        execute_actions(ctx, player.id, player.zone_id, &trigger.action_json)?;
    }

    log::info!("fire_trigger: player={} trigger={}", player.id, trigger_id);
    Ok(())
}
```

- [ ] **Step 3: Implement objective auto-tracking**

Replace the `track_objective` stub with the real implementation:

```rust
fn track_objective(ctx: &ReducerContext, player_id: u64, obj_type: ObjectiveType, target_id: u64) {
    // Find all active quests for this player
    let active_quests: Vec<QuestProgress> = ctx.db.quest_progress()
        .player_id()
        .filter(&player_id)
        .filter(|qp| qp.status == QuestStatus::Active)
        .collect();

    for qp in &active_quests {
        let objectives: Vec<QuestObjective> = ctx.db.quest_objective()
            .quest_id()
            .filter(&qp.quest_id)
            .collect();

        for obj in &objectives {
            if obj.objective_type != obj_type { continue; }

            // Check if target matches
            let target_matches = match obj_type {
                ObjectiveType::KillEnemy => {
                    let target: serde_json::Value = serde_json::from_str(&obj.target_json).unwrap_or_default();
                    target["enemy_def_id"].as_u64().unwrap_or(0) == target_id
                }
                ObjectiveType::CollectItem => {
                    let target: serde_json::Value = serde_json::from_str(&obj.target_json).unwrap_or_default();
                    target["item_id"].as_u64().unwrap_or(0) == target_id
                }
                ObjectiveType::TalkToNpc => {
                    let target: serde_json::Value = serde_json::from_str(&obj.target_json).unwrap_or_default();
                    target["npc_def_id"].as_u64().unwrap_or(0) == target_id
                }
                ObjectiveType::EnterZone => {
                    let target: serde_json::Value = serde_json::from_str(&obj.target_json).unwrap_or_default();
                    target["zone_id"].as_u64().unwrap_or(0) == target_id
                }
            };

            if !target_matches { continue; }

            // Increment progress
            if let Some(op) = ctx.db.objective_progress()
                .player_id()
                .filter(&player_id)
                .find(|op| op.objective_id == obj.id)
            {
                if op.current_count < obj.required_count {
                    ctx.db.objective_progress().id().update(ObjectiveProgress {
                        current_count: op.current_count + 1,
                        ..op
                    });
                    log::info!("track_objective: player={} obj={} count={}/{}", player_id, obj.id, op.current_count + 1, obj.required_count);

                    // KillEnemy loot drops
                    if obj_type == ObjectiveType::KillEnemy && !obj.loot_on_kill_json.is_empty() {
                        if let Ok(loot) = serde_json::from_str::<serde_json::Value>(&obj.loot_on_kill_json) {
                            let item_id = loot["item_id"].as_u64().unwrap_or(0);
                            let count = loot["count"].as_u64().unwrap_or(1) as u32;
                            if item_id > 0 {
                                let _ = give_item_to_player(ctx, player_id, item_id, count);
                            }
                        }
                    }
                }
            }
        }
    }
}
```

- [ ] **Step 4: Hook objective tracking into existing reducers**

Add to `apply_damage_to_enemy` (after the `is_dead` block that schedules respawn, ~line 1376-1390):

```rust
    if is_dead {
        log::info!("apply_damage_to_enemy: enemy={} killed by player={}", enemy_id, attacker_id);
        // Track KillEnemy objectives for the killing player
        track_objective(ctx, attacker_id, ObjectiveType::KillEnemy, enemy.enemy_def_id);
        // Schedule respawn... (existing code)
    }
```

Add to `enter_zone` (just before the `Ok(())` return in both forward and reverse branches):

```rust
            // Track EnterZone objectives
            track_objective(ctx, player.id, ObjectiveType::EnterZone, dest_zone_id);
```

Add to `give_item_to_player` (at the end, before `Ok(())`):

```rust
    // Track CollectItem objectives
    track_objective(ctx, player_id, ObjectiveType::CollectItem, item_id);
```

Note: `track_objective` calling `give_item_to_player` (for loot drops) which calls `track_objective` again could recurse — but only if a CollectItem objective has loot_on_kill_json, which doesn't make sense. The recursion terminates because CollectItem objectives never have loot_on_kill_json set. Add a guard just in case:

In `track_objective`, add a depth parameter or use a static flag. Simpler: move the `track_objective` call for CollectItem into `give_item_to_player` but skip calling it from within `track_objective`'s own loot drop path. The cleanest approach: make `give_item_to_player` take an optional `track: bool` parameter:

```rust
fn give_item_to_player(
    ctx: &ReducerContext,
    player_id: u64,
    item_id: u64,
    count: u32,
    track: bool,  // false when called from within track_objective to prevent recursion
) -> Result<(), String> {
    // ... existing insert/update logic ...
    if track {
        track_objective(ctx, player_id, ObjectiveType::CollectItem, item_id);
    }
    Ok(())
}
```

Update all call sites to pass `track: true` (from `execute_actions`) or `track: false` (from `track_objective` loot drops).

- [ ] **Step 5: Build to verify**

```bash
cd server && spacetime build
```

Expected: Build succeeds.

- [ ] **Step 6: Commit**

```bash
git add server/spacetimedb/src/lib.rs
git commit -m "feat(server): add Trigger table, objective auto-tracking, condition/action engine"
```

---

## Task 5: Deploy Server + Regenerate Bindings

**Files:**
- No code changes — build, publish, generate

### Steps

- [ ] **Step 1: Build the server module**

```bash
cd server && spacetime build
```

Expected: Build succeeds.

- [ ] **Step 2: Publish to local SpacetimeDB**

This is a breaking schema change (new tables), so use `--delete-data`:

```bash
spacetime publish --server local zoneforge-server --delete-data -y
```

Expected: Module published. All existing data wiped.

- [ ] **Step 3: Regenerate client bindings**

```bash
cd client && spacetime generate --lang csharp --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

Expected: New autogen files for all new tables (Npc, NpcDefinition, DialogueNode, DialogueChoice, Quest, QuestObjective, QuestProgress, ObjectiveProgress, Trigger, Item, Inventory) and reducers.

- [ ] **Step 4: Regenerate editor bindings**

```bash
cd editor && spacetime generate --lang csharp --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

- [ ] **Step 5: Verify bindings generated correctly**

```bash
ls client/Assets/Scripts/autogen/Types/ | grep -i -E "npc|quest|trigger|item|inventory|dialogue"
ls editor/Assets/Scripts/autogen/Types/ | grep -i -E "npc|quest|trigger|item|inventory|dialogue"
```

Expected: Files like `Npc.g.cs`, `NpcDefinition.g.cs`, `Quest.g.cs`, `QuestObjective.g.cs`, `QuestProgress.g.cs`, `ObjectiveProgress.g.cs`, `Trigger.g.cs`, `Item.g.cs`, `Inventory.g.cs`, `DialogueNode.g.cs`, `DialogueChoice.g.cs`.

- [ ] **Step 6: Recreate default zone (required after --delete-data)**

```bash
spacetime call --server local zoneforge-server create_zone '["Village", 64, 64, 2.0]'
```

- [ ] **Step 7: Commit autogen files**

```bash
git add client/Assets/Scripts/autogen/ editor/Assets/Scripts/autogen/
git commit -m "chore: regenerate client and editor bindings for Phase 5 tables"
```

---

## Task 6: Client — SpacetimeDBManager Updates + NpcManager + TriggerManager

**Files:**
- Modify: `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs`
- Create: `client/Assets/Scripts/Npc/NpcManager.cs`
- Create: `client/Assets/Scripts/Runtime/TriggerManager.cs`

### Steps

- [ ] **Step 1: Add new events and subscriptions to client SpacetimeDBManager**

Add new static events alongside existing ones:

```csharp
// NPC events
public static event Action<Npc> OnNpcInserted;
public static event Action<Npc, Npc> OnNpcUpdated;
public static event Action<Npc> OnNpcDeleted;
public static event Action<NpcDefinition> OnNpcDefInserted;

// Quest events
public static event Action<QuestProgress> OnQuestProgressInserted;
public static event Action<QuestProgress, QuestProgress> OnQuestProgressUpdated;
public static event Action<QuestProgress> OnQuestProgressDeleted;
public static event Action<ObjectiveProgress> OnObjectiveProgressInserted;
public static event Action<ObjectiveProgress, ObjectiveProgress> OnObjectiveProgressUpdated;
public static event Action<ObjectiveProgress> OnObjectiveProgressDeleted;
public static event Action<Quest> OnQuestInserted;
public static event Action<QuestObjective> OnQuestObjectiveInserted;

// Trigger events
public static event Action<Trigger> OnTriggerInserted;
public static event Action<Trigger> OnTriggerDeleted;

// Dialogue events
public static event Action<DialogueNode> OnDialogueNodeInserted;
public static event Action<DialogueChoice> OnDialogueChoiceInserted;

// Item/Inventory events
public static event Action<Item> OnItemInserted;
public static event Action<Inventory> OnInventoryInserted;
public static event Action<Inventory, Inventory> OnInventoryUpdated;
public static event Action<Inventory> OnInventoryDeleted;
```

Add to the subscription query array in `OnConnect`:

```csharp
// Light tables (unfiltered) — add these:
"SELECT * FROM npc_def",
"SELECT * FROM quest",
"SELECT * FROM quest_objective",
"SELECT * FROM item",
"SELECT * FROM dialogue_node",
"SELECT * FROM dialogue_choice",

// Zone-filtered — add these:
$"SELECT * FROM npc WHERE zone_id = {CurrentZoneId}",
$"SELECT * FROM trigger WHERE zone_id = {CurrentZoneId}",

// Player-filtered — add these:
$"SELECT * FROM quest_progress WHERE player_id = {{LOCAL_PLAYER_ID}}",
$"SELECT * FROM objective_progress WHERE player_id = {{LOCAL_PLAYER_ID}}",
$"SELECT * FROM inventory WHERE player_id = {{LOCAL_PLAYER_ID}}",
```

Note: Player-filtered queries need the local player ID. Since `create_player` is called before subscriptions are fully set up, the player row may not exist yet. Use a two-phase subscription approach: subscribe to player-independent tables first, then subscribe to player-specific tables after the local player row is known. Alternatively, subscribe to all quest_progress/objective_progress/inventory rows (they're small tables) and filter client-side.

**Recommended approach:** Subscribe to all rows of `quest_progress`, `objective_progress`, and `inventory` (unfiltered) since these tables are small. Filter by local player ID on the client side when rendering UI.

Register event handlers in `OnSubscriptionApplied`:

```csharp
Conn.Db.Npc.OnInsert += (ctx, npc) => OnNpcInserted?.Invoke(npc);
Conn.Db.Npc.OnUpdate += (ctx, old, n) => OnNpcUpdated?.Invoke(old, n);
Conn.Db.Npc.OnDelete += (ctx, npc) => OnNpcDeleted?.Invoke(npc);
Conn.Db.NpcDef.OnInsert += (ctx, def) => OnNpcDefInserted?.Invoke(def);
Conn.Db.QuestProgress.OnInsert += (ctx, qp) => OnQuestProgressInserted?.Invoke(qp);
Conn.Db.QuestProgress.OnUpdate += (ctx, old, n) => OnQuestProgressUpdated?.Invoke(old, n);
Conn.Db.QuestProgress.OnDelete += (ctx, qp) => OnQuestProgressDeleted?.Invoke(qp);
Conn.Db.ObjectiveProgress.OnInsert += (ctx, op) => OnObjectiveProgressInserted?.Invoke(op);
Conn.Db.ObjectiveProgress.OnUpdate += (ctx, old, n) => OnObjectiveProgressUpdated?.Invoke(old, n);
Conn.Db.ObjectiveProgress.OnDelete += (ctx, op) => OnObjectiveProgressDeleted?.Invoke(op);
Conn.Db.Quest.OnInsert += (ctx, q) => OnQuestInserted?.Invoke(q);
Conn.Db.QuestObjective.OnInsert += (ctx, qo) => OnQuestObjectiveInserted?.Invoke(qo);
Conn.Db.Trigger.OnInsert += (ctx, t) => OnTriggerInserted?.Invoke(t);
Conn.Db.Trigger.OnDelete += (ctx, t) => OnTriggerDeleted?.Invoke(t);
Conn.Db.DialogueNode.OnInsert += (ctx, dn) => OnDialogueNodeInserted?.Invoke(dn);
Conn.Db.DialogueChoice.OnInsert += (ctx, dc) => OnDialogueChoiceInserted?.Invoke(dc);
Conn.Db.Item.OnInsert += (ctx, i) => OnItemInserted?.Invoke(i);
Conn.Db.Inventory.OnInsert += (ctx, inv) => OnInventoryInserted?.Invoke(inv);
Conn.Db.Inventory.OnUpdate += (ctx, old, n) => OnInventoryUpdated?.Invoke(old, n);
Conn.Db.Inventory.OnDelete += (ctx, inv) => OnInventoryDeleted?.Invoke(inv);
```

- [ ] **Step 2: Create NpcManager.cs**

Follow EnemyManager pattern exactly. Create `client/Assets/Scripts/Npc/NpcManager.cs`:

- Singleton with DontDestroyOnLoad
- `Dictionary<ulong, GameObject> _npcs` for tracking spawned NPC GameObjects
- Subscribe to `OnNpcInserted`, `OnNpcUpdated`, `OnNpcDeleted`, `OnConnected`
- Backfill on `OnConnected` with `Conn.Db.Npc.Iter()`
- `SpawnNpc(Npc npc)`: create capsule primitive, apply URP Lit material, set color by NpcType (Green=QuestGiver, Yellow=Vendor, White=Dialogue), add floating name label (reuse PlayerHealthBar pattern), disable CapsuleCollider for physics but add a new one as trigger for click detection
- Quest indicator: check if NPC's definition is QuestGiver, and if any quests reference this NPC, show "!" or "?" indicator
- `OnNpcDeleted`: Destroy and remove from dictionary
- Clean up on zone change via `OnZoneChanged` event

- [ ] **Step 3: Create TriggerManager.cs**

Create `client/Assets/Scripts/Runtime/TriggerManager.cs`:

- Singleton with DontDestroyOnLoad
- `Dictionary<ulong, GameObject> _triggerZones` for tracking trigger collider GameObjects
- Subscribe to `OnTriggerInserted`, `OnTriggerDeleted`, `OnConnected`, `OnZoneChanged`
- `SpawnTriggerZone(Trigger trigger)`: create empty GameObject with SphereCollider (isTrigger=true, radius from trigger), place at (position_x, 0, position_y)
- For OnEnter triggers: attach a `TriggerZoneDetector` MonoBehaviour component that implements `OnTriggerEnter(Collider other)` — if `other` is the local player, call `Conn.Reducers.FireTrigger(triggerId)`. Include a 5-second debounce per trigger ID using a `Dictionary<ulong, float> _lastFired`.
- For OnInteract triggers: on player click (integrate with CombatInputHandler or a separate InputManager), check if clicked world position falls within any OnInteract trigger's radius. If so, call `FireTrigger`.
- On `play_sound` action result: the server doesn't communicate play_sound back to the client since it's a no-op server-side. Handle this by having the client check trigger's action_json locally and play sounds immediately when firing a trigger, before calling the reducer.
- Clean up on zone change

- [ ] **Step 4: Commit**

```bash
git add client/Assets/Scripts/
git commit -m "feat(client): add NpcManager, TriggerManager, and SpacetimeDB subscription updates"
```

---

## Task 7: Client — DialogueUI, QuestUI, InventoryUI

**Files:**
- Create: `client/Assets/Scripts/UI/DialogueUI.cs`
- Create: `client/Assets/Scripts/UI/QuestUI.cs`
- Create: `client/Assets/Scripts/UI/InventoryUI.cs`

### Steps

- [ ] **Step 1: Create DialogueUI.cs**

Create `client/Assets/Scripts/UI/DialogueUI.cs`:

- Programmatic Canvas (ScreenSpaceOverlay, sortingOrder=20) with dark semi-transparent background panel
- Anchored bottom-center, taking ~60% width and ~30% height
- Top section: NPC name Label (bold, white, 18pt) + npc_text Label (white, 14pt, word-wrap)
- Bottom section: vertical list of choice Buttons
- Static `Open(ulong npcId)` method: reads NPC's dialogue_tree_id from `Conn.Db.Npc`, finds first DialogueNode (lowest sort_order) from `Conn.Db.DialogueNode` filtered by tree_id, displays node
- `ShowNode(ulong nodeId)`: clears choice buttons, sets npc_text, iterates `Conn.Db.DialogueChoice` filtered by node_id. For each choice: evaluate condition_json client-side (read QuestProgress/Inventory from local cache) — hide choices with unmet conditions. Add Button with choice_text, onClick calls `Conn.Reducers.SelectDialogueChoice(npcId, choiceId)` then advances to next_node_id. If next_node_id == 0, close panel.
- `Close()`: destroys canvas, re-enables player input
- While open: block PlayerController movement input (set a static `IsDialogueOpen` flag)
- NpcManager calls `DialogueUI.Open(npcId)` when player clicks an NPC

- [ ] **Step 2: Create QuestUI.cs**

Create `client/Assets/Scripts/UI/QuestUI.cs`:

Three sub-components, all managed by a single MonoBehaviour:

**Quest Journal** (toggled with J key):
- Programmatic Canvas (ScreenSpaceOverlay, sortingOrder=15), centered panel ~50% width, ~70% height
- ScrollView with sections: "Active Quests" and "Completed Quests" (Foldout-style)
- Each quest entry: quest name (bold) + description + objective checklist with progress (e.g., "☐ Kill Goblins: 2/5")
- Reads from `Conn.Db.Quest.Iter()`, `Conn.Db.QuestProgress.Iter()`, `Conn.Db.ObjectiveProgress.Iter()`
- Filter by local player ID
- Rebuild on `OnQuestProgressInserted/Updated/Deleted` and `OnObjectiveProgressInserted/Updated`

**Quest Accept Popup** (shown when start_quest action fires):
- Small centered panel with quest name, description, objective list preview
- Accept and Decline buttons
- Accept calls `Conn.Reducers.AcceptQuest(questId)`
- Listen for QuestProgress insert with status=Active to detect quest start actions
- Alternatively, the client can detect start_quest in trigger/dialogue action_json and show the popup directly

**Quest Tracker HUD** (always visible):
- Small panel top-right, ~200px wide
- Lists active quest objectives with progress: "Kill Goblins: 2/5"
- Updates in real-time via ObjectiveProgress callbacks
- No toggle — always on screen when there are active quests

- [ ] **Step 3: Create InventoryUI.cs**

Create `client/Assets/Scripts/UI/InventoryUI.cs`:

- Toggle with I key
- Programmatic Canvas (ScreenSpaceOverlay, sortingOrder=15), right side panel ~250px wide
- Vertical ScrollView listing each inventory row: Item name (from Item table lookup) + "×quantity"
- Subscribes to `OnInventoryInserted/Updated/Deleted`
- Refreshes list on any inventory change
- Filter by local player ID

- [ ] **Step 4: Commit**

```bash
git add client/Assets/Scripts/UI/
git commit -m "feat(client): add DialogueUI, QuestUI, and InventoryUI"
```

---

## Task 8: Editor — NPC Creation Panel + Item Definition Panel

**Files:**
- Modify: `editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs`
- Modify: `editor/Assets/Scripts/Runtime/ToolbarController.cs`
- Modify: `editor/Assets/UI/ToolbarController.uxml`
- Create: `editor/Assets/Scripts/Runtime/NpcCreationPanel.cs`
- Create: `editor/Assets/UI/NpcCreationPanel.uxml`
- Create: `editor/Assets/Scripts/Runtime/NpcRenderer.cs`
- Create: `editor/Assets/Scripts/Runtime/NpcPlacer.cs`
- Create: `editor/Assets/Scripts/Runtime/ItemDefinitionPanel.cs`
- Create: `editor/Assets/UI/ItemDefinitionPanel.uxml`

### Steps

- [ ] **Step 1: Update editor SpacetimeDBManager with new subscriptions**

Add to the subscription query array in `OnConnect`:

```csharp
"SELECT * FROM npc_def",
"SELECT * FROM npc",
"SELECT * FROM item",
"SELECT * FROM quest",
"SELECT * FROM quest_objective",
"SELECT * FROM dialogue_node",
"SELECT * FROM dialogue_choice",
"SELECT * FROM trigger",
"SELECT * FROM quest_progress",
"SELECT * FROM objective_progress",
"SELECT * FROM inventory",
```

Add corresponding static events and register handlers in `OnSubscriptionApplied`.

- [ ] **Step 2: Create NpcCreationPanel.uxml**

Follow EntityPalettePanel.uxml pattern:

```xml
<ui:UXML xmlns:ui="UnityEngine.UIElements">
    <ui:VisualElement name="panel" class="panel">
        <ui:VisualElement name="panel-header" class="panel-header">
            <ui:Label text="NPCs" class="panel-title"/>
            <ui:Button name="collapse-btn" text="▾" class="collapse-btn"/>
        </ui:VisualElement>
        <ui:VisualElement name="panel-body">
            <ui:Label text="Create NPC Definition" class="section-label"/>
            <ui:TextField name="npc-name-field" label="Name"/>
            <ui:DropdownField name="npc-type-dropdown" label="Type" choices="QuestGiver,Vendor,Dialogue"/>
            <ui:Button name="btn-create-def" text="Create Definition"/>
            <ui:Label text="Definitions" class="section-label"/>
            <ui:ScrollView name="def-list" class="entity-scroll"/>
            <ui:Label text="Placed NPCs" class="section-label"/>
            <ui:ScrollView name="npc-list" class="entity-scroll"/>
        </ui:VisualElement>
    </ui:VisualElement>
</ui:UXML>
```

- [ ] **Step 3: Create NpcCreationPanel.cs**

Follow EntityPalettePanel.cs pattern:

- `[RequireComponent(typeof(UIDocument))]` MonoBehaviour
- Query UXML elements in `OnEnable()`
- "Create Definition" button → calls `Conn.Reducers.CreateNpcDef(name, npcType)`
- Definition list: populated from `Conn.Db.NpcDef.Iter()`, each row has name + delete button
- Placed NPC list: populated from `Conn.Db.Npc.Iter()` filtered by active zone
- Selected definition stored for placement: `NpcPlacer.SelectedDefId = def.Id`
- `SetVisible(bool)` for ToolbarController integration
- Subscribe to `OnNpcDefInserted` and `OnNpcInserted/Deleted` to refresh lists

- [ ] **Step 4: Create NpcPlacer.cs**

Follow EntityPlacer.cs pattern:

- `public static ulong? SelectedDefId`
- In `Update()`, if `SelectedDefId != null` and left mouse clicked and not over UI (UIHoverTracker):
  - Raycast from camera to terrain MeshCollider
  - Call `Conn.Reducers.SpawnNpc(zoneId, defId, x, z, dialogueTreeId: 0)`

- [ ] **Step 5: Create NpcRenderer.cs**

Follow EntityRenderer.cs pattern:

- Subscribe to NPC insert/delete events
- Spawn colored capsules (green/yellow/white by type) with floating name labels
- Track in `Dictionary<ulong, GameObject>`
- Clear on zone change

- [ ] **Step 6: Create ItemDefinitionPanel**

Create `editor/Assets/UI/ItemDefinitionPanel.uxml` and `editor/Assets/Scripts/Runtime/ItemDefinitionPanel.cs`:

- Simple form: name TextField, description TextField, item_type TextField, max_stack IntegerField
- Create button → calls `Conn.Reducers.CreateItemDef(name, description, itemType, maxStack)`
- List of existing items from `Conn.Db.Item.Iter()` with delete buttons
- Follow same panel pattern (UIDocument, collapsible, SetVisible)

- [ ] **Step 7: Update ToolbarController**

Add `[SerializeField]` references for new panels: `NpcCreationPanel _npcPanel`, `ItemDefinitionPanel _itemPanel`

Add toggle buttons in UXML: "NPCs" and "Items" buttons

Add toggle methods and refresh logic following existing pattern.

- [ ] **Step 8: Commit**

```bash
git add editor/Assets/
git commit -m "feat(editor): add NPC Creation Panel, NPC Placer/Renderer, and Item Definition Panel"
```

---

## Task 9: Editor — Visual Scripting Panel (Node Graph Infrastructure)

**Files:**
- Create: `editor/Assets/Scripts/Runtime/NodeGraph/NodeGraphCanvas.cs`
- Create: `editor/Assets/Scripts/Runtime/NodeGraph/GraphNode.cs`
- Create: `editor/Assets/Scripts/Runtime/NodeGraph/GraphPort.cs`
- Create: `editor/Assets/Scripts/Runtime/NodeGraph/GraphEdge.cs`
- Create: `editor/Assets/Scripts/Runtime/VisualScriptingPanel.cs`
- Create: `editor/Assets/UI/VisualScriptingPanel.uxml`

### Steps

- [ ] **Step 1: Create NodeGraphCanvas.cs**

`editor/Assets/Scripts/Runtime/NodeGraph/NodeGraphCanvas.cs`:

A UIToolkit VisualElement subclass providing:
- Pan: middle-mouse-drag translates all children via `transform.position`
- Zoom: scroll wheel scales via `transform.scale` (clamp 0.25–2.0)
- Tracks all `GraphNode` children, all `GraphEdge` connections
- `AddNode(GraphNode node)`, `RemoveNode(GraphNode node)`, `AddEdge(GraphEdge edge)`, `RemoveEdge(GraphEdge edge)`
- Selection: click on node to select (yellow border), click canvas to deselect, Delete key removes selected
- `generateVisualContent` draws a subtle grid background

Key implementation:
```csharp
public class NodeGraphCanvas : VisualElement
{
    readonly VisualElement _contentContainer;
    readonly GraphEdgeDrawer _edgeDrawer;
    Vector2 _panOffset;
    float _zoom = 1f;
    GraphNode _selectedNode;
    readonly List<GraphNode> _nodes = new();
    readonly List<GraphEdge> _edges = new();

    public NodeGraphCanvas()
    {
        style.flexGrow = 1;
        style.overflow = Overflow.Hidden;

        _contentContainer = new VisualElement { name = "graph-content" };
        _contentContainer.style.position = Position.Absolute;
        _contentContainer.style.left = 0; _contentContainer.style.top = 0;
        _contentContainer.style.right = 0; _contentContainer.style.bottom = 0;
        Add(_contentContainer);

        _edgeDrawer = new GraphEdgeDrawer(this);
        _contentContainer.Add(_edgeDrawer);

        RegisterCallback<WheelEvent>(OnWheel);
        RegisterCallback<PointerDownEvent>(OnPointerDown);
        RegisterCallback<PointerMoveEvent>(OnPointerMove);
        RegisterCallback<PointerUpEvent>(OnPointerUp);
        RegisterCallback<KeyDownEvent>(OnKeyDown);
        focusable = true;
    }
    // ... pan/zoom/selection implementation
}
```

- [ ] **Step 2: Create GraphNode.cs**

`editor/Assets/Scripts/Runtime/NodeGraph/GraphNode.cs`:

A VisualElement representing a node:
- Title bar (Label with node type name + color coding)
- Body with inline field editors (TextField, DropdownField, IntegerField)
- Input/output ports as `GraphPort` child elements
- Draggable via pointer events on title bar
- Selection state: `.node--selected` class toggles yellow border
- `public string NodeType` (Dialogue, Choice, Quest, Objective, Trigger, Condition, Action)
- `public Dictionary<string, object> Data` — field values
- `public List<GraphPort> InputPorts`, `public List<GraphPort> OutputPorts`

```csharp
public class GraphNode : VisualElement
{
    public string NodeType { get; private set; }
    public readonly Dictionary<string, object> Data = new();
    public readonly List<GraphPort> InputPorts = new();
    public readonly List<GraphPort> OutputPorts = new();
    VisualElement _titleBar;
    VisualElement _fieldContainer;
    bool _isDragging;
    Vector2 _dragStart;

    public GraphNode(string nodeType, Vector2 position)
    {
        NodeType = nodeType;
        style.position = Position.Absolute;
        style.left = position.x;
        style.top = position.y;
        AddToClassList("graph-node");

        _titleBar = new VisualElement();
        _titleBar.AddToClassList("node-title");
        _titleBar.Add(new Label(nodeType));
        Add(_titleBar);

        _fieldContainer = new VisualElement();
        _fieldContainer.AddToClassList("node-fields");
        Add(_fieldContainer);

        BuildFieldsForType(nodeType);

        _titleBar.RegisterCallback<PointerDownEvent>(OnTitlePointerDown);
        RegisterCallback<PointerMoveEvent>(OnPointerMove);
        RegisterCallback<PointerUpEvent>(OnPointerUp);
    }
    // ... field building + drag implementation
}
```

- [ ] **Step 3: Create GraphPort.cs**

`editor/Assets/Scripts/Runtime/NodeGraph/GraphPort.cs`:

Small circle element (12x12) on node edge:
- `Direction` (Input/Output)
- `PortType` (string — "dialogue", "choice", "quest", "objective", "trigger", "condition", "action")
- `ConnectedEdge` reference
- Pointer down on output port starts edge drag; pointer up on input port completes connection

```csharp
public class GraphPort : VisualElement
{
    public enum Direction { Input, Output }
    public Direction Dir { get; }
    public string PortType { get; }
    public GraphNode ParentNode { get; }
    public GraphEdge ConnectedEdge { get; set; }

    public GraphPort(Direction dir, string portType, GraphNode parentNode)
    {
        Dir = dir;
        PortType = portType;
        ParentNode = parentNode;
        AddToClassList("graph-port");
        AddToClassList(dir == Direction.Input ? "port-input" : "port-output");
        style.width = 12; style.height = 12;
        style.borderTopLeftRadius = 6; style.borderTopRightRadius = 6;
        style.borderBottomLeftRadius = 6; style.borderBottomRightRadius = 6;
        style.backgroundColor = new Color(0.5f, 0.8f, 1f);
    }
}
```

- [ ] **Step 4: Create GraphEdge.cs (edge drawer)**

`editor/Assets/Scripts/Runtime/NodeGraph/GraphEdge.cs`:

```csharp
public class GraphEdge
{
    public GraphPort From { get; set; }
    public GraphPort To { get; set; }
}

public class GraphEdgeDrawer : VisualElement
{
    readonly NodeGraphCanvas _canvas;

    public GraphEdgeDrawer(NodeGraphCanvas canvas)
    {
        _canvas = canvas;
        generateVisualContent += DrawEdges;
        pickingMode = PickingMode.Ignore;
        style.position = Position.Absolute;
        style.left = 0; style.top = 0; style.right = 0; style.bottom = 0;
    }

    void DrawEdges(MeshGenerationContext ctx)
    {
        var painter = ctx.painter2D;
        foreach (var edge in _canvas.Edges)
        {
            var fromCenter = GetPortWorldPos(edge.From);
            var toCenter = GetPortWorldPos(edge.To);
            painter.strokeColor = new Color(0.6f, 0.8f, 1f, 0.8f);
            painter.lineWidth = 2f;
            painter.BeginPath();
            painter.MoveTo(fromCenter);
            // Bezier curve for nicer edges
            float dx = Mathf.Abs(toCenter.x - fromCenter.x) * 0.5f;
            painter.BezierCurveTo(
                new Vector2(fromCenter.x + dx, fromCenter.y),
                new Vector2(toCenter.x - dx, toCenter.y),
                toCenter
            );
            painter.Stroke();
        }
    }

    Vector2 GetPortWorldPos(GraphPort port)
    {
        var r = port.worldBound;
        var canvasR = worldBound;
        return new Vector2(r.center.x - canvasR.x, r.center.y - canvasR.y);
    }
}
```

- [ ] **Step 5: Create VisualScriptingPanel.uxml**

```xml
<ui:UXML xmlns:ui="UnityEngine.UIElements">
    <ui:VisualElement name="vs-root" class="panel-wrapper">
        <ui:VisualElement name="vs-panel" class="vs-panel">
            <ui:VisualElement name="panel-header" class="panel-header">
                <ui:Label name="panel-title" text="Visual Scripting" class="panel-title"/>
                <ui:Button name="collapse-btn" text="▾" class="collapse-btn"/>
            </ui:VisualElement>
            <ui:VisualElement name="panel-body" class="vs-body">
                <ui:VisualElement name="sidebar" class="vs-sidebar">
                    <ui:DropdownField name="graph-type" label="Type" choices="Dialogue,Quest,Trigger"/>
                    <ui:ScrollView name="graph-list" class="vs-graph-list"/>
                    <ui:Button name="btn-new" text="+ New"/>
                    <ui:Button name="btn-save" text="Save"/>
                    <ui:Button name="btn-import" text="Import File"/>
                    <ui:Button name="btn-export" text="Export File"/>
                </ui:VisualElement>
                <ui:VisualElement name="canvas-container" class="vs-canvas-container"/>
                <ui:ScrollView name="error-panel" class="vs-error-panel"/>
            </ui:VisualElement>
        </ui:VisualElement>
    </ui:VisualElement>
</ui:UXML>
```

- [ ] **Step 6: Create VisualScriptingPanel.cs (scaffold)**

`editor/Assets/Scripts/Runtime/VisualScriptingPanel.cs`:

- MonoBehaviour with `[RequireComponent(typeof(UIDocument))]`
- OnEnable: query UXML elements, create `NodeGraphCanvas` and add it to `canvas-container`
- Sidebar: populate graph-list from database (dialogue trees, quests, triggers)
- "New" button: create empty graph on canvas
- "Save" button: serialize graph to JSON and call reducers (implemented in Task 10)
- Node toolbox: right-click on canvas shows context menu with node types to add
- SetVisible(bool) for ToolbarController
- Wire up to ToolbarController (add "Scripting" toggle button)

- [ ] **Step 7: Commit**

```bash
git add editor/Assets/
git commit -m "feat(editor): add Visual Scripting Panel with node graph infrastructure"
```

---

## Task 10: Editor — Visual Scripting Panel (Node Types + Serialization)

**Files:**
- Modify: `editor/Assets/Scripts/Runtime/NodeGraph/GraphNode.cs`
- Modify: `editor/Assets/Scripts/Runtime/VisualScriptingPanel.cs`

### Steps

- [ ] **Step 1: Implement BuildFieldsForType in GraphNode**

Each node type gets specific inline editors:

```csharp
void BuildFieldsForType(string nodeType)
{
    switch (nodeType)
    {
        case "Dialogue":
            AddPort(GraphPort.Direction.Input, "dialogue");    // from previous choice
            AddPort(GraphPort.Direction.Output, "dialogue");   // to choices
            AddTextField("npc_text", "NPC Text", multiline: true);
            AddIntField("sort_order", "Order", 0);
            break;
        case "Choice":
            AddPort(GraphPort.Direction.Input, "dialogue");    // from parent dialogue
            AddPort(GraphPort.Direction.Output, "dialogue");   // to next node
            AddTextField("choice_text", "Choice Text");
            AddTextField("condition_json", "Condition (JSON)");
            AddTextField("action_json", "Action (JSON)");
            break;
        case "Quest":
            AddPort(GraphPort.Direction.Output, "quest");      // to objectives
            AddTextField("name", "Quest Name");
            AddTextField("description", "Description", multiline: true);
            AddTextField("prerequisites", "Prerequisites (JSON)");
            AddTextField("reward_action_json", "Reward (JSON)");
            break;
        case "Objective":
            AddPort(GraphPort.Direction.Input, "quest");       // from quest
            AddDropdown("objective_type", "Type", new[]{"KillEnemy","CollectItem","TalkToNpc","EnterZone"});
            AddTextField("description", "Description");
            AddTextField("target_json", "Target (JSON)");
            AddIntField("required_count", "Count", 1);
            AddTextField("loot_on_kill_json", "Loot on Kill (JSON)");
            break;
        case "Trigger":
            AddPort(GraphPort.Direction.Output, "trigger");    // to actions
            AddDropdown("trigger_type", "Type", new[]{"OnEnter","OnInteract"});
            AddTextField("position", "Position (x,y)");
            AddFloatField("radius", "Radius", 3f);
            AddTextField("condition_json", "Condition (JSON)");
            AddTextField("action_json", "Action (JSON)");
            break;
    }
}
```

Helper methods `AddTextField`, `AddIntField`, `AddDropdown`, `AddPort` create UIElements and store values in `Data` dictionary.

- [ ] **Step 2: Implement Save — serialize graph to reducer calls**

In `VisualScriptingPanel.cs`:

```csharp
void OnSaveClicked()
{
    string graphType = _graphTypeDropdown.value;
    switch (graphType)
    {
        case "Dialogue": SaveDialogueTree(); break;
        case "Quest": SaveQuest(); break;
        case "Trigger": SaveTriggers(); break;
    }
}

void SaveDialogueTree()
{
    // Collect all Dialogue and Choice nodes
    var dialogueNodes = _canvas.Nodes.Where(n => n.NodeType == "Dialogue").ToList();
    var choiceNodes = _canvas.Nodes.Where(n => n.NodeType == "Choice").ToList();

    // Build nodes_json: array of {npc_text, sort_order}
    var nodesJson = new List<object>();
    var nodeIndexMap = new Dictionary<GraphNode, int>();
    for (int i = 0; i < dialogueNodes.Count; i++)
    {
        nodeIndexMap[dialogueNodes[i]] = i;
        nodesJson.Add(new { npc_text = dialogueNodes[i].Data["npc_text"], sort_order = i });
    }

    // Build choices_json: array of {node_index, choice_text, next_node_index, condition_json, action_json}
    var choicesJson = new List<object>();
    foreach (var choice in choiceNodes)
    {
        // Find parent dialogue node (input edge)
        int nodeIndex = FindConnectedNodeIndex(choice, GraphPort.Direction.Input, nodeIndexMap);
        // Find next dialogue node (output edge)
        int nextIndex = FindConnectedNodeIndex(choice, GraphPort.Direction.Output, nodeIndexMap);

        choicesJson.Add(new {
            node_index = nodeIndex,
            choice_text = choice.Data.GetValueOrDefault("choice_text", ""),
            next_node_index = nextIndex,
            condition_json = choice.Data.GetValueOrDefault("condition_json", ""),
            action_json = choice.Data.GetValueOrDefault("action_json", ""),
        });
    }

    string nodesStr = JsonUtility.ToJson(nodesJson);  // Use Newtonsoft or manual serialization
    string choicesStr = JsonUtility.ToJson(choicesJson);

    SpacetimeDBManager.Conn.Reducers.CreateDialogueTree(_currentTreeId, nodesStr, choicesStr);
}
```

Similar implementations for `SaveQuest()` and `SaveTriggers()`.

- [ ] **Step 3: Implement Load — populate canvas from database**

When user clicks a quest/dialogue/trigger in the sidebar:

```csharp
void LoadDialogueTree(ulong treeId)
{
    _canvas.Clear();
    _currentTreeId = treeId;

    var nodes = SpacetimeDBManager.Conn.Db.DialogueNode.Iter()
        .Where(n => n.TreeId == treeId)
        .OrderBy(n => n.SortOrder)
        .ToList();

    var nodeMap = new Dictionary<ulong, GraphNode>();
    float y = 50f;
    foreach (var node in nodes)
    {
        var gn = new GraphNode("Dialogue", new Vector2(100f, y));
        gn.Data["npc_text"] = node.NpcText;
        gn.Data["sort_order"] = (int)node.SortOrder;
        gn.Data["_db_id"] = node.Id;
        _canvas.AddNode(gn);
        nodeMap[node.Id] = gn;
        y += 120f;

        // Load choices for this node
        var choices = SpacetimeDBManager.Conn.Db.DialogueChoice.Iter()
            .Where(c => c.NodeId == node.Id)
            .ToList();
        float cx = 350f;
        foreach (var choice in choices)
        {
            var cn = new GraphNode("Choice", new Vector2(cx, y - 60f));
            cn.Data["choice_text"] = choice.ChoiceText;
            cn.Data["condition_json"] = choice.ConditionJson;
            cn.Data["action_json"] = choice.ActionJson;
            cn.Data["_next_node_id"] = choice.NextNodeId;
            _canvas.AddNode(cn);
            // Create edge from dialogue → choice
            _canvas.AddEdge(new GraphEdge { From = gn.OutputPorts[0], To = cn.InputPorts[0] });
            cx += 200f;
        }
    }

    // Create edges from choices to their next dialogue nodes
    foreach (var gn in _canvas.Nodes.Where(n => n.NodeType == "Choice"))
    {
        ulong nextId = (ulong)gn.Data.GetValueOrDefault("_next_node_id", 0UL);
        if (nextId > 0 && nodeMap.TryGetValue(nextId, out var nextNode))
        {
            _canvas.AddEdge(new GraphEdge { From = gn.OutputPorts[0], To = nextNode.InputPorts[0] });
        }
    }

    _edgeDrawer.MarkDirtyRepaint();
}
```

Similar for `LoadQuest(questId)` and `LoadTriggers(zoneId)`.

- [ ] **Step 4: Commit**

```bash
git add editor/Assets/Scripts/
git commit -m "feat(editor): add node types, save/load for Visual Scripting Panel"
```

---

## Task 11: Editor — Visual Scripting Panel (Validation + Error Highlighting)

**Files:**
- Create: `editor/Assets/Scripts/Runtime/NodeGraph/GraphValidator.cs`
- Modify: `editor/Assets/Scripts/Runtime/VisualScriptingPanel.cs`

### Steps

- [ ] **Step 1: Create GraphValidator.cs**

`editor/Assets/Scripts/Runtime/NodeGraph/GraphValidator.cs`:

```csharp
public enum ValidationLevel { Error, Warning, Valid }

public struct ValidationResult
{
    public ValidationLevel Level;
    public string Message;
    public GraphNode Node;  // null for graph-level issues
}

public static class GraphValidator
{
    public static List<ValidationResult> Validate(NodeGraphCanvas canvas)
    {
        var results = new List<ValidationResult>();
        ValidateDanglingReferences(canvas, results);
        ValidateGraphIntegrity(canvas, results);
        ValidateMissingFields(canvas, results);
        ValidatePrerequisiteCycles(canvas, results);
        return results;
    }

    static void ValidateDanglingReferences(NodeGraphCanvas canvas, List<ValidationResult> results)
    {
        foreach (var node in canvas.Nodes)
        {
            switch (node.NodeType)
            {
                case "Objective":
                    // Check target references exist in database
                    string objType = node.Data.GetValueOrDefault("objective_type", "") as string;
                    string targetJson = node.Data.GetValueOrDefault("target_json", "{}") as string;
                    // Parse and validate against Conn.Db tables
                    if (objType == "KillEnemy")
                    {
                        // Check enemy_def_id exists
                        // ... parse targetJson, look up in SpacetimeDBManager.Conn.Db.EnemyDef
                    }
                    break;
                // Similar checks for other node types referencing NPCs, items, quests
            }
        }
    }

    static void ValidateGraphIntegrity(NodeGraphCanvas canvas, List<ValidationResult> results)
    {
        // Find orphaned nodes (no connections)
        foreach (var node in canvas.Nodes)
        {
            bool hasAnyEdge = canvas.Edges.Any(e => e.From.ParentNode == node || e.To.ParentNode == node);
            if (!hasAnyEdge && canvas.Nodes.Count > 1)
            {
                results.Add(new ValidationResult {
                    Level = ValidationLevel.Warning,
                    Message = $"{node.NodeType} node is disconnected",
                    Node = node,
                });
            }
        }
        // Check dialogue cycles without exit
        // ... traverse from root, detect cycles, ensure at least one path reaches a terminal
    }

    static void ValidateMissingFields(NodeGraphCanvas canvas, List<ValidationResult> results)
    {
        foreach (var node in canvas.Nodes)
        {
            switch (node.NodeType)
            {
                case "Dialogue":
                    if (string.IsNullOrWhiteSpace(node.Data.GetValueOrDefault("npc_text", "") as string))
                        results.Add(new ValidationResult { Level = ValidationLevel.Error, Message = "Dialogue node has empty text", Node = node });
                    break;
                case "Quest":
                    if (string.IsNullOrWhiteSpace(node.Data.GetValueOrDefault("name", "") as string))
                        results.Add(new ValidationResult { Level = ValidationLevel.Error, Message = "Quest has no name", Node = node });
                    bool hasObjective = canvas.Edges.Any(e => e.From.ParentNode == node && canvas.Nodes.Any(n => n == e.To.ParentNode && n.NodeType == "Objective"));
                    if (!hasObjective)
                        results.Add(new ValidationResult { Level = ValidationLevel.Error, Message = "Quest has no objectives", Node = node });
                    break;
                // ... similar for other types
            }
        }
    }

    static void ValidatePrerequisiteCycles(NodeGraphCanvas canvas, List<ValidationResult> results)
    {
        // Build dependency graph from quest prerequisites, detect cycles via DFS
        // This checks across all quests in the database, not just the current canvas
    }
}
```

- [ ] **Step 2: Apply validation results to node visuals**

In `VisualScriptingPanel.cs`, add a `RunValidation()` method called on edit (debounced 500ms) and on save:

```csharp
void RunValidation()
{
    var results = GraphValidator.Validate(_canvas);

    // Reset all node styles
    foreach (var node in _canvas.Nodes)
    {
        node.RemoveFromClassList("node--error");
        node.RemoveFromClassList("node--warning");
    }

    // Apply error/warning classes
    foreach (var r in results)
    {
        if (r.Node != null)
        {
            r.Node.AddToClassList(r.Level == ValidationLevel.Error ? "node--error" : "node--warning");
        }
    }

    // Populate error panel
    _errorPanel.Clear();
    foreach (var r in results)
    {
        var row = new Label($"[{r.Level}] {r.Message}");
        row.AddToClassList(r.Level == ValidationLevel.Error ? "error-row" : "warning-row");
        var captured = r;
        row.RegisterCallback<ClickEvent>(_ => {
            if (captured.Node != null) _canvas.SelectAndPanTo(captured.Node);
        });
        _errorPanel.Add(row);
    }

    _hasErrors = results.Any(r => r.Level == ValidationLevel.Error);
    _hasWarnings = results.Any(r => r.Level == ValidationLevel.Warning);
}
```

CSS classes for validation:
```
.node--error { border-color: #ff4444; border-width: 2px; }
.node--warning { border-color: #ffaa00; border-width: 2px; }
.error-row { color: #ff4444; }
.warning-row { color: #ffaa00; }
```

On save: if `_hasErrors`, show dialog "Cannot save — fix errors first". If `_hasWarnings`, show "Save with warnings?"

- [ ] **Step 3: Commit**

```bash
git add editor/Assets/Scripts/Runtime/NodeGraph/GraphValidator.cs editor/Assets/Scripts/Runtime/VisualScriptingPanel.cs
git commit -m "feat(editor): add validation and error highlighting to Visual Scripting Panel"
```

---

## Task 12: Editor — Visual Scripting Panel (Import/Export)

**Files:**
- Create: `editor/Assets/Scripts/Runtime/NodeGraph/QuestFileSerializer.cs`
- Modify: `editor/Assets/Scripts/Runtime/VisualScriptingPanel.cs`

### Steps

- [ ] **Step 1: Create QuestFileSerializer.cs**

`editor/Assets/Scripts/Runtime/NodeGraph/QuestFileSerializer.cs`:

Handles bidirectional conversion between `.quest.json` files and the node graph canvas.

```csharp
public static class QuestFileSerializer
{
    // ── Export ──
    public static string ExportToJson(NodeGraphCanvas canvas)
    {
        // Find Quest node
        var questNode = canvas.Nodes.FirstOrDefault(n => n.NodeType == "Quest");
        if (questNode == null) return "{}";

        // Resolve numeric IDs to human-readable names
        var result = new Dictionary<string, object>
        {
            ["name"] = questNode.Data.GetValueOrDefault("name", ""),
            ["description"] = questNode.Data.GetValueOrDefault("description", ""),
            ["prerequisites"] = ResolveQuestIdsToNames(questNode.Data.GetValueOrDefault("prerequisites", "[]") as string),
            ["reward"] = questNode.Data.GetValueOrDefault("reward_action_json", "[]"),
        };

        // Export dialogue tree (connected to quest via NPC)
        var dialogueNodes = canvas.Nodes.Where(n => n.NodeType == "Dialogue").ToList();
        if (dialogueNodes.Any())
        {
            var dialogue = new Dictionary<string, object>();
            var nodes = new List<object>();
            foreach (var dn in dialogueNodes)
            {
                var nodeEntry = new Dictionary<string, object>
                {
                    ["id"] = dn.Data.GetValueOrDefault("_local_id", dn.GetHashCode().ToString()),
                    ["text"] = dn.Data.GetValueOrDefault("npc_text", ""),
                };
                // Gather choices
                var choices = GetConnectedChoices(canvas, dn);
                nodeEntry["choices"] = choices.Select(c => new Dictionary<string, object>
                {
                    ["text"] = c.Data.GetValueOrDefault("choice_text", ""),
                    ["condition"] = ParseJsonOrEmpty(c.Data.GetValueOrDefault("condition_json", "") as string),
                    ["action"] = ParseJsonOrEmpty(c.Data.GetValueOrDefault("action_json", "") as string),
                    ["next"] = FindNextNodeId(canvas, c),
                }).ToList();
                nodes.Add(nodeEntry);
            }
            dialogue["nodes"] = nodes;
            result["dialogue"] = dialogue;
        }

        // Export objectives
        var objectives = canvas.Nodes.Where(n => n.NodeType == "Objective")
            .Select(o => new Dictionary<string, object>
            {
                ["description"] = o.Data.GetValueOrDefault("description", ""),
                ["type"] = o.Data.GetValueOrDefault("objective_type", ""),
                ["target"] = ResolveTargetToName(o),
                ["count"] = o.Data.GetValueOrDefault("required_count", 1),
                ["loot_on_kill"] = ParseJsonOrEmpty(o.Data.GetValueOrDefault("loot_on_kill_json", "") as string),
            }).ToList();
        result["objectives"] = objectives;

        return JsonConvert.SerializeObject(result, Formatting.Indented);
        // Note: Unity doesn't include Newtonsoft by default. Use JsonUtility or add Newtonsoft package.
        // Alternatively, use a simple manual JSON builder.
    }

    // ── Import ──
    public static void ImportFromJson(string json, NodeGraphCanvas canvas)
    {
        canvas.Clear();
        var data = JsonConvert.DeserializeObject<Dictionary<string, object>>(json);

        // Create Quest node
        var questNode = new GraphNode("Quest", new Vector2(50, 50));
        questNode.Data["name"] = data.GetValueOrDefault("name", "");
        questNode.Data["description"] = data.GetValueOrDefault("description", "");
        questNode.Data["prerequisites"] = ResolveNamesToQuestIds(data.GetValueOrDefault("prerequisites", "[]"));
        questNode.Data["reward_action_json"] = data.GetValueOrDefault("reward", "[]");
        canvas.AddNode(questNode);

        // Create Objective nodes
        float objY = 200f;
        if (data.ContainsKey("objectives"))
        {
            var objectives = data["objectives"] as List<object>;
            foreach (var obj in objectives)
            {
                var objDict = obj as Dictionary<string, object>;
                var objNode = new GraphNode("Objective", new Vector2(300, objY));
                objNode.Data["description"] = objDict.GetValueOrDefault("description", "");
                objNode.Data["objective_type"] = objDict.GetValueOrDefault("type", "");
                objNode.Data["target_json"] = ResolveNameToTargetJson(objDict);
                objNode.Data["required_count"] = Convert.ToInt32(objDict.GetValueOrDefault("count", 1));
                canvas.AddNode(objNode);
                canvas.AddEdge(new GraphEdge { From = questNode.OutputPorts[0], To = objNode.InputPorts[0] });
                objY += 100f;
            }
        }

        // Create Dialogue nodes
        if (data.ContainsKey("dialogue"))
        {
            // ... similar logic to create Dialogue and Choice nodes from the dialogue tree
        }

        canvas.RefreshEdges();
    }

    // ── Name resolution helpers ──
    static List<string> ResolveQuestIdsToNames(string prerequisitesJson)
    {
        // Parse JSON array of quest IDs, look up Quest.Name in Conn.Db.Quest
        // Return list of human-readable names
    }

    static string ResolveNamesToQuestIds(object prerequisites)
    {
        // Look up quest names in Conn.Db.Quest, return JSON array of IDs
    }

    static string ResolveTargetToName(GraphNode objectiveNode)
    {
        // Resolve enemy_def_id → EnemyDef.Name, item_id → Item.Name, etc.
    }
}
```

- [ ] **Step 2: Wire import/export buttons in VisualScriptingPanel**

```csharp
void OnImportClicked()
{
    // Unity's EditorUtility.OpenFilePanel is editor-only.
    // For runtime app, use a simple path input or SFB (StandaloneFileBrowser).
    // Simplest approach: read from StreamingAssets/quests/ directory
    string path = System.IO.Path.Combine(Application.streamingAssetsPath, "quests");
    // Show file list from that directory
    var files = System.IO.Directory.GetFiles(path, "*.quest.json");
    // Show selection UI or import all
    foreach (var file in files)
    {
        string json = System.IO.File.ReadAllText(file);
        QuestFileSerializer.ImportFromJson(json, _canvas);
    }
    RunValidation();
}

void OnExportClicked()
{
    string json = QuestFileSerializer.ExportToJson(_canvas);
    string fileName = _canvas.Nodes
        .FirstOrDefault(n => n.NodeType == "Quest")?.Data.GetValueOrDefault("name", "untitled") as string;
    string safeName = fileName.ToLower().Replace(" ", "-");
    string path = System.IO.Path.Combine(Application.streamingAssetsPath, "quests", $"{safeName}.quest.json");
    System.IO.Directory.CreateDirectory(System.IO.Path.GetDirectoryName(path));
    System.IO.File.WriteAllText(path, json);
    Debug.Log($"Exported to: {path}");
}
```

- [ ] **Step 3: Commit**

```bash
git add editor/Assets/Scripts/Runtime/NodeGraph/QuestFileSerializer.cs editor/Assets/Scripts/Runtime/VisualScriptingPanel.cs
git commit -m "feat(editor): add import/export for .quest.json files"
```

---

## Task 13: Quest Content — 3 Chain Files + Server Seeding

**Files:**
- Create: `editor/Assets/StreamingAssets/quests/welcome-to-the-village.quest.json`
- Create: `editor/Assets/StreamingAssets/quests/forest-patrol.quest.json`
- Create: `editor/Assets/StreamingAssets/quests/the-cave-below.quest.json`

### Steps

- [ ] **Step 1: Create quest data via CLI or editor**

Since the import system uses human-readable names that map to database IDs, and those IDs don't exist until the data is created, the easiest approach is to seed data via `spacetime call` first, then create the quest files that reference it.

First, create the required NPC definitions, item definitions, and enemy definitions:

```bash
# NPC definitions
spacetime call --server local zoneforge-server create_npc_def '["Village Elder", {"QuestGiver":{}}]'
spacetime call --server local zoneforge-server create_npc_def '["Guard", {"Dialogue":{}}]'
spacetime call --server local zoneforge-server create_npc_def '["Forest Ranger", {"QuestGiver":{}}]'

# Item definitions
spacetime call --server local zoneforge-server create_item_def '["Village Map", "A map of the village and surrounding areas", "Quest", 1]'
spacetime call --server local zoneforge-server create_item_def '["Goblin Tooth", "A sharp tooth from a goblin", "Material", 99]'
spacetime call --server local zoneforge-server create_item_def '["Cave Key", "An ornate key that opens the cave entrance", "Quest", 1]'
spacetime call --server local zoneforge-server create_item_def '["Heros Medal", "A medal of honor for saving the village", "Quest", 1]'
```

- [ ] **Step 2: Create zones and place NPCs**

```bash
# Create zones (Village already exists from Task 5 Step 6)
spacetime call --server local zoneforge-server create_zone '["Forest", 64, 64, 1.0]'
spacetime call --server local zoneforge-server create_zone '["Cave", 32, 32, 0.0]'

# Check zone IDs
spacetime sql --server local zoneforge-server "SELECT id, name FROM zone"

# Place NPCs (adjust zone_id and positions based on actual IDs)
# spawn_npc(zone_id, npc_def_id, x, y, dialogue_tree_id)
spacetime call --server local zoneforge-server spawn_npc '[VILLAGE_ZONE_ID, ELDER_DEF_ID, 32.0, 30.0, 0]'
spacetime call --server local zoneforge-server spawn_npc '[VILLAGE_ZONE_ID, GUARD_DEF_ID, 32.0, 50.0, 0]'
spacetime call --server local zoneforge-server spawn_npc '[FOREST_ZONE_ID, RANGER_DEF_ID, 5.0, 32.0, 0]'
```

Replace placeholder IDs with actual auto-increment IDs from the database.

- [ ] **Step 3: Create dialogue trees**

Create dialogue trees for Village Elder and Forest Ranger using `spacetime call`:

```bash
# Village Elder dialogue tree
spacetime call --server local zoneforge-server create_dialogue_tree '[0, "[{\"npc_text\":\"Welcome, traveler! Our village has been troubled of late.\",\"sort_order\":0},{\"npc_text\":\"We have lived here for generations, but creatures from the forest have grown bold...\",\"sort_order\":1},{\"npc_text\":\"Speak with the Guard at the village gate, then venture into the Forest to see for yourself.\",\"sort_order\":2}]", "[{\"node_index\":0,\"choice_text\":\"Tell me about the village.\",\"next_node_index\":1,\"condition_json\":\"\",\"action_json\":\"\"},{\"node_index\":0,\"choice_text\":\"I am ready to help.\",\"next_node_index\":2,\"condition_json\":\"\",\"action_json\":\"\"},{\"node_index\":1,\"choice_text\":\"I see. What can I do?\",\"next_node_index\":2,\"condition_json\":\"\",\"action_json\":\"\"},{\"node_index\":2,\"choice_text\":\"I will do it.\",\"next_node_index\":-1,\"condition_json\":\"\",\"action_json\":\"[{\\\"type\\\":\\\"start_quest\\\",\\\"quest_id\\\":QUEST1_ID}]\"}]"]'
```

This is complex via CLI. An alternative is to build a Rust `init` seeder or use the editor's import system once it's working. **Recommended: create the quest .json files first, then import them via the editor.**

- [ ] **Step 4: Create welcome-to-the-village.quest.json**

```json
{
  "name": "Welcome to the Village",
  "description": "The Village Elder has asked you to speak with the Guard and explore the Forest.",
  "prerequisites": [],
  "reward": [{"type": "give_item", "item": "Village Map", "count": 1}],
  "dialogue": {
    "npc": "Village Elder",
    "nodes": [
      {"id": "welcome", "text": "Welcome, traveler! Our village has been troubled of late.", "choices": [
        {"text": "Tell me about the village.", "next": "lore"},
        {"text": "I'm ready to help.", "next": "task"}
      ]},
      {"id": "lore", "text": "We've lived here for generations, but creatures from the forest have grown bold...", "choices": [
        {"text": "I see. What can I do?", "next": "task"}
      ]},
      {"id": "task", "text": "Speak with the Guard at the village gate, then venture into the Forest to see for yourself.", "choices": [
        {"text": "I'll do it.", "action": [{"type": "start_quest"}]}
      ]}
    ]
  },
  "objectives": [
    {"description": "Talk to the Guard", "type": "TalkToNpc", "target": "Guard", "count": 1},
    {"description": "Enter the Forest", "type": "EnterZone", "target": "Forest", "count": 1}
  ]
}
```

- [ ] **Step 5: Create forest-patrol.quest.json**

```json
{
  "name": "Forest Patrol",
  "description": "The Forest Ranger needs help dealing with the goblin threat.",
  "prerequisites": ["Welcome to the Village"],
  "reward": [{"type": "give_item", "item": "Cave Key", "count": 1}],
  "dialogue": {
    "npc": "Forest Ranger",
    "nodes": [
      {"id": "halt", "text": "Halt! The forest is dangerous.", "choices": [
        {"text": "The Elder sent me.", "condition": [{"type": "has_item", "item": "Village Map"}], "next": "recognized"},
        {"text": "I can handle myself.", "next": "prove"}
      ]},
      {"id": "recognized", "text": "Ah, you carry the Elder's map. Good — we need help with the goblin problem.", "choices": [
        {"text": "What do you need?", "next": "task"}
      ]},
      {"id": "prove", "text": "Brave words. Prove them — clear out some goblins.", "choices": [
        {"text": "Consider it done.", "next": "task"}
      ]},
      {"id": "task", "text": "Kill 3 goblins and bring back 2 of their teeth as proof.", "choices": [
        {"text": "I'll return with the teeth.", "action": [{"type": "start_quest"}]}
      ]}
    ]
  },
  "objectives": [
    {"description": "Kill Goblins", "type": "KillEnemy", "target": "Goblin", "count": 3,
     "loot_on_kill": {"item": "Goblin Tooth", "count": 1}},
    {"description": "Collect Goblin Teeth", "type": "CollectItem", "target": "Goblin Tooth", "count": 2}
  ]
}
```

- [ ] **Step 6: Create the-cave-below.quest.json**

```json
{
  "name": "The Cave Below",
  "description": "Something stirs in the depths of the cave. Use the Cave Key to enter and investigate.",
  "prerequisites": ["Forest Patrol"],
  "reward": [{"type": "give_item", "item": "Heros Medal", "count": 1}],
  "triggers": [
    {
      "type": "OnEnter",
      "zone": "Cave",
      "position": [16.0, 2.0],
      "radius": 3.0,
      "condition": [{"type": "has_item", "item": "Cave Key"}, {"type": "quest_status", "quest": "The Cave Below", "status": "NotStarted"}],
      "action": [{"type": "start_quest"}]
    },
    {
      "type": "OnEnter",
      "zone": "Cave",
      "position": [16.0, 28.0],
      "radius": 3.0,
      "condition": [{"type": "quest_status", "quest": "The Cave Below", "status": "Completed"}],
      "action": [{"type": "play_sound", "sound_id": "quest_complete"}]
    }
  ],
  "objectives": [
    {"description": "Explore Cave Chambers", "type": "EnterZone", "target": "Cave", "count": 2},
    {"description": "Defeat the Cave Boss", "type": "KillEnemy", "target": "Cave Boss", "count": 1}
  ]
}
```

- [ ] **Step 7: Create StreamingAssets/quests directory and commit**

```bash
mkdir -p editor/Assets/StreamingAssets/quests
# Move/create the .quest.json files in that directory
git add editor/Assets/StreamingAssets/quests/
git commit -m "content: add 3 quest chain files (Village, Forest, Cave)"
```

---

## Task 14: Integration Testing + Polish

**Files:**
- Various fixes across all modified files

### Steps

- [ ] **Step 1: Verify server builds and publishes**

```bash
cd server && spacetime build && spacetime publish --server local zoneforge-server --delete-data -y
```

- [ ] **Step 2: Regenerate bindings for both client and editor**

```bash
cd client && spacetime generate --lang csharp --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
cd editor && spacetime generate --lang csharp --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

- [ ] **Step 3: Recreate zones and seed data**

```bash
spacetime call --server local zoneforge-server create_zone '["Village", 64, 64, 2.0]'
spacetime call --server local zoneforge-server create_zone '["Forest", 64, 64, 1.0]'
spacetime call --server local zoneforge-server create_zone '["Cave", 32, 32, 0.0]'
```

- [ ] **Step 4: Open Unity editor project and verify**

1. Open `editor/` in Unity
2. Verify all scripts compile (no C# errors in Console)
3. Enter Play mode, verify SpacetimeDB connection
4. Test NPC Creation Panel: create NPC def, place in zone
5. Test Item Definition Panel: create items
6. Test Visual Scripting Panel: create dialogue tree, save, reload

- [ ] **Step 5: Open Unity client project and verify**

1. Open `client/` in Unity
2. Verify all scripts compile
3. Enter Play mode, verify connection
4. Check NpcManager spawns NPC capsules
5. Click NPC → DialogueUI opens
6. Test quest accept/complete flow
7. Check TriggerManager creates colliders for OnEnter triggers
8. Verify InventoryUI shows items after quest rewards

- [ ] **Step 6: Fix any issues found during testing**

Address compile errors, runtime exceptions, subscription issues, UI layout problems.

- [ ] **Step 7: Update PROGRESS.md**

Mark all Group 12 items as complete. Note the minimal Item/Inventory items pulled forward from Group 13.

- [ ] **Step 8: Commit everything**

```bash
git add -A
git commit -m "feat: Phase 5 Group 12 complete — triggers, quests, NPCs, visual scripting"
```
