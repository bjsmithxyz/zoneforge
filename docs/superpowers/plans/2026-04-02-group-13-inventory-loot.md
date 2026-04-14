# Group 13 — Inventory & Loot Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a complete inventory and loot system — item definitions, per-player inventory and equipment slots, world item drops from enemy deaths, pickup mechanic, client UI (grid + character sheet), and an editor loot-creation panel.

**Architecture:** Server-authoritative: all inventory mutations happen in reducers. Client subscribes to `item_def`, `inventory`, `equipment`, and `item_drop` tables and renders them reactively. Loot rolls happen inside `apply_damage_to_enemy` when an enemy dies; the resulting `ItemDrop` rows appear in the world for nearby players to pick up.

**Tech Stack:** SpacetimeDB 2.x Rust server module, Unity 2022.3 LTS C# (UIToolkit for UI), SpacetimeDB C# SDK autogen bindings.

---

## File Map

### Server — `server/spacetimedb/src/lib.rs`
Add at end of existing tables/reducers section (or as logically grouped):
- New custom types: `ItemType` enum, `Rarity` enum
- New tables: `ItemDefinition`, `Inventory`, `Equipment`, `ItemDrop`, `LootTable`
- New reducers: `create_item_def`, `delete_item_def`, `give_item`, `pickup_item`, `equip_item`, `unequip_item`, `create_loot_table`, `delete_loot_table`
- Modify: `apply_damage_to_enemy` — add loot-drop logic on enemy death

### Client — `client/Assets/Scripts/`
- **Modify:** `Runtime/SpacetimeDBManager.cs` — add events + subscriptions for 4 new tables
- **Create:** `UI/InventoryManager.cs` — caches inventory/equipment/item-drop state, bridges SpacetimeDBManager events to UI
- **Create:** `UI/InventoryUI.cs` — I key toggles inventory grid panel (UIToolkit), drag-and-drop, tooltips
- **Create:** `UI/EquipmentUI.cs` — C key toggles equipment character-sheet panel (UIToolkit)
- **Create:** `Zone/ItemPickupManager.cs` — scans active ItemDrop rows, shows "F to pick up" prompt, calls Reducer.PickupItem

### Editor — `editor/Assets/Scripts/`
- **Modify:** `Runtime/SpacetimeDBManager.cs` — add subscriptions + events for `item_def` and `loot_table`
- **Create:** `Runtime/LootCreationPanel.cs` — UIToolkit panel: create/delete ItemDefinitions and LootTable entries

---

## Task 1: Server — Custom Types + ItemDefinition Table + Admin Reducers

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Step 1: Add ItemType and Rarity enums**

Add these two types after the existing `StatusEffectType` enum (search for `pub enum StatusEffectType`). They need `#[derive(SpacetimeType, Clone, Debug, PartialEq)]`.

```rust
#[derive(SpacetimeType, Clone, Debug, PartialEq)]
pub enum ItemType {
    Weapon,
    Armor,
    Accessory,
    Consumable,
}

#[derive(SpacetimeType, Clone, Debug, PartialEq)]
pub enum Rarity {
    Common,
    Uncommon,
    Rare,
    Epic,
}
```

- [ ] **Step 2: Add ItemDefinition table**

Add after the two enums:

```rust
/// Shared item template. All players see all item definitions.
#[table(accessor = item_def, public)]
pub struct ItemDefinition {
    #[primary_key]
    #[auto_inc]
    pub id:           u64,
    pub name:         String,
    pub description:  String,
    pub item_type:    ItemType,
    pub rarity:       Rarity,
    pub icon_name:    String,   // colour-swatch name used by client UI
    pub damage_bonus: i32,      // weapon stat
    pub armor_bonus:  i32,      // armor stat
    pub healing:      i32,      // consumable stat
    pub value:        u32,      // gold sell value
}
```

- [ ] **Step 3: Add create_item_def and delete_item_def reducers**

Add at end of file:

```rust
/// Admin: create an item definition (shared template visible to all clients).
#[reducer]
pub fn create_item_def(
    ctx: &ReducerContext,
    name: String,
    description: String,
    item_type: ItemType,
    rarity: Rarity,
    icon_name: String,
    damage_bonus: i32,
    armor_bonus: i32,
    healing: i32,
    value: u32,
) -> Result<(), String> {
    if !is_admin(ctx) {
        return Err("Not authorized: admin only".to_string());
    }
    if name.is_empty() || name.len() > 64 {
        return Err("name must be 1–64 characters".to_string());
    }
    ctx.db.item_def().insert(ItemDefinition {
        id: 0,
        name,
        description,
        item_type,
        rarity,
        icon_name,
        damage_bonus,
        armor_bonus,
        healing,
        value,
    });
    Ok(())
}

/// Admin: delete an item definition. Does NOT remove it from inventories.
#[reducer]
pub fn delete_item_def(ctx: &ReducerContext, item_def_id: u64) -> Result<(), String> {
    if !is_admin(ctx) {
        return Err("Not authorized: admin only".to_string());
    }
    ctx.db.item_def().id().delete(&item_def_id);
    Ok(())
}
```

- [ ] **Step 4: Build and verify no compile errors**

```bash
cd server && spacetime build
```
Expected: `Finished release [optimized]` — zero errors. Fix any before continuing.

- [ ] **Step 5: Commit**

```bash
cd server && git add spacetimedb/src/lib.rs && git commit -m "feat(server): add ItemType/Rarity enums, ItemDefinition table, admin CRUD reducers"
```

---

## Task 2: Server — Inventory + Equipment Tables + Player Reducers

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Step 1: Add Inventory and Equipment tables**

Add after `ItemDefinition`:

```rust
/// One row per stack of items a player holds.
#[table(accessor = inventory, public)]
pub struct Inventory {
    #[primary_key]
    #[auto_inc]
    pub id:          u64,
    #[index(btree)]
    pub player_id:   u64,
    pub item_def_id: u64,
    pub quantity:    u32,
}

/// One row per player — tracks equipped items (one per equipment slot).
/// Row is created automatically when a player equips their first item.
#[table(accessor = equipment, public)]
pub struct Equipment {
    #[primary_key]
    pub player_id:    u64,  // same as Player.id
    pub weapon_id:    Option<u64>,    // ItemDefinition.id, or None
    pub armor_id:     Option<u64>,
    pub accessory_id: Option<u64>,
}
```

- [ ] **Step 2: Add add_to_inventory internal helper**

Add as a plain `fn` (not `#[reducer]`) before the new reducers:

```rust
/// Internal helper: add `quantity` of `item_def_id` to `player_id`'s inventory.
/// Stacks into an existing row if one exists, otherwise inserts a new row.
fn add_to_inventory(ctx: &ReducerContext, player_id: u64, item_def_id: u64, quantity: u32) {
    if let Some(existing) = ctx.db.inventory()
        .player_id()
        .filter(&player_id)
        .find(|row| row.item_def_id == item_def_id)
    {
        ctx.db.inventory().id().update(Inventory {
            quantity: existing.quantity + quantity,
            ..existing
        });
    } else {
        ctx.db.inventory().insert(Inventory {
            id: 0,
            player_id,
            item_def_id,
            quantity,
        });
    }
}

/// Internal helper: remove `quantity` of `item_def_id` from `player_id`'s inventory.
/// Returns true if successful. Deletes the row if quantity reaches 0.
fn remove_from_inventory(ctx: &ReducerContext, player_id: u64, item_def_id: u64, quantity: u32) -> bool {
    if let Some(row) = ctx.db.inventory()
        .player_id()
        .filter(&player_id)
        .find(|r| r.item_def_id == item_def_id)
    {
        if row.quantity < quantity {
            return false;
        }
        if row.quantity == quantity {
            ctx.db.inventory().id().delete(&row.id);
        } else {
            ctx.db.inventory().id().update(Inventory {
                quantity: row.quantity - quantity,
                ..row
            });
        }
        true
    } else {
        false
    }
}
```

- [ ] **Step 3: Add give_item, equip_item, unequip_item reducers**

```rust
/// Admin: give an item directly to a player's inventory.
#[reducer]
pub fn give_item(
    ctx: &ReducerContext,
    player_id: u64,
    item_def_id: u64,
    quantity: u32,
) -> Result<(), String> {
    if !is_admin(ctx) {
        return Err("Not authorized: admin only".to_string());
    }
    ctx.db.player().id().find(&player_id)
        .ok_or("Player not found")?;
    ctx.db.item_def().id().find(&item_def_id)
        .ok_or("ItemDefinition not found")?;
    if quantity == 0 {
        return Err("quantity must be > 0".to_string());
    }
    add_to_inventory(ctx, player_id, item_def_id, quantity);
    Ok(())
}

/// Player: equip an item from inventory. Swaps back any previously equipped item.
#[reducer]
pub fn equip_item(ctx: &ReducerContext, item_def_id: u64) -> Result<(), String> {
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or("Player not found")?;
    let def = ctx.db.item_def().id().find(&item_def_id)
        .ok_or("ItemDefinition not found")?;

    // Must have it in inventory
    if !remove_from_inventory(ctx, player.id, item_def_id, 1) {
        return Err("Item not in inventory".to_string());
    }

    // Get or create equipment row
    let eq = ctx.db.equipment().player_id().find(&player.id)
        .unwrap_or(Equipment {
            player_id:    player.id,
            weapon_id:    None,
            armor_id:     None,
            accessory_id: None,
        });

    // Determine which slot to use
    let (old_id, new_eq) = match def.item_type {
        ItemType::Weapon => (eq.weapon_id, Equipment { weapon_id: Some(item_def_id), ..eq }),
        ItemType::Armor  => (eq.armor_id,  Equipment { armor_id:  Some(item_def_id), ..eq }),
        ItemType::Accessory => (eq.accessory_id, Equipment { accessory_id: Some(item_def_id), ..eq }),
        ItemType::Consumable => {
            // Put item back — consumables aren't equipped
            add_to_inventory(ctx, player.id, item_def_id, 1);
            return Err("Consumables cannot be equipped".to_string());
        }
    };

    // Swap: return the previously equipped item to inventory
    if let Some(prev_id) = old_id {
        add_to_inventory(ctx, player.id, prev_id, 1);
    }

    if ctx.db.equipment().player_id().find(&player.id).is_some() {
        ctx.db.equipment().player_id().update(new_eq);
    } else {
        ctx.db.equipment().insert(new_eq);
    }
    Ok(())
}

/// Player: unequip an item by slot name ("weapon", "armor", "accessory").
/// Returns it to inventory.
#[reducer]
pub fn unequip_item(ctx: &ReducerContext, slot: String) -> Result<(), String> {
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or("Player not found")?;
    let eq = ctx.db.equipment().player_id().find(&player.id)
        .ok_or("No equipment row found")?;

    let (item_id, new_eq) = match slot.as_str() {
        "weapon" => (
            eq.weapon_id.ok_or("Weapon slot is empty")?,
            Equipment { weapon_id: None, ..eq },
        ),
        "armor" => (
            eq.armor_id.ok_or("Armor slot is empty")?,
            Equipment { armor_id: None, ..eq },
        ),
        "accessory" => (
            eq.accessory_id.ok_or("Accessory slot is empty")?,
            Equipment { accessory_id: None, ..eq },
        ),
        _ => return Err(format!("Unknown slot '{}'. Use weapon/armor/accessory", slot)),
    };

    add_to_inventory(ctx, player.id, item_id, 1);
    ctx.db.equipment().player_id().update(new_eq);
    Ok(())
}
```

- [ ] **Step 4: Build**

```bash
cd server && spacetime build
```
Expected: zero errors.

- [ ] **Step 5: Commit**

```bash
cd server && git add spacetimedb/src/lib.rs && git commit -m "feat(server): add Inventory/Equipment tables and equip/give/unequip reducers"
```

---

## Task 3: Server — LootTable + ItemDrop + Loot-on-Death

**Files:**
- Modify: `server/spacetimedb/src/lib.rs`

- [ ] **Step 1: Add LootTable and ItemDrop tables**

Add after `Equipment`:

```rust
/// Defines what an enemy type can drop and at what chance.
#[table(accessor = loot_table, public)]
pub struct LootTable {
    #[primary_key]
    #[auto_inc]
    pub id:           u64,
    #[index(btree)]
    pub enemy_def_id: u64,
    pub item_def_id:  u64,
    pub drop_chance:  u32,   // 0–100 (percent)
    pub min_quantity: u32,
    pub max_quantity: u32,
}

/// A dropped item stack sitting in the world waiting to be picked up.
#[table(accessor = item_drop, public)]
pub struct ItemDrop {
    #[primary_key]
    #[auto_inc]
    pub id:          u64,
    #[index(btree)]
    pub zone_id:     u64,
    pub item_def_id: u64,
    pub quantity:    u32,
    pub pos_x:       f32,
    pub pos_y:       f32,
}
```

- [ ] **Step 2: Add create_loot_table and delete_loot_table reducers**

```rust
/// Admin: add a loot entry — enemy_def drops item_def at drop_chance%.
#[reducer]
pub fn create_loot_table(
    ctx: &ReducerContext,
    enemy_def_id: u64,
    item_def_id:  u64,
    drop_chance:  u32,
    min_quantity: u32,
    max_quantity: u32,
) -> Result<(), String> {
    if !is_admin(ctx) {
        return Err("Not authorized: admin only".to_string());
    }
    if drop_chance > 100 {
        return Err("drop_chance must be 0–100".to_string());
    }
    if min_quantity == 0 || max_quantity < min_quantity {
        return Err("min_quantity must be >= 1 and max_quantity >= min_quantity".to_string());
    }
    ctx.db.enemy_def().id().find(&enemy_def_id)
        .ok_or("EnemyDefinition not found")?;
    ctx.db.item_def().id().find(&item_def_id)
        .ok_or("ItemDefinition not found")?;
    ctx.db.loot_table().insert(LootTable {
        id: 0,
        enemy_def_id,
        item_def_id,
        drop_chance,
        min_quantity,
        max_quantity,
    });
    Ok(())
}

/// Admin: remove a loot table entry by id.
#[reducer]
pub fn delete_loot_table(ctx: &ReducerContext, loot_table_id: u64) -> Result<(), String> {
    if !is_admin(ctx) {
        return Err("Not authorized: admin only".to_string());
    }
    ctx.db.loot_table().id().delete(&loot_table_id);
    Ok(())
}
```

- [ ] **Step 3: Add pickup_item reducer**

```rust
/// Player: pick up an ItemDrop. Player must be in the same zone.
#[reducer]
pub fn pickup_item(ctx: &ReducerContext, item_drop_id: u64) -> Result<(), String> {
    let player = ctx.db.player().identity().find(ctx.sender())
        .ok_or("Player not found")?;
    if player.is_dead {
        return Err("Cannot pick up items while dead".to_string());
    }
    let drop = ctx.db.item_drop().id().find(&item_drop_id)
        .ok_or("ItemDrop not found")?;
    if drop.zone_id != player.zone_id {
        return Err("ItemDrop is not in your zone".to_string());
    }
    // Proximity check: 2 unit radius
    let dx = player.position_x - drop.pos_x;
    let dz = player.position_y - drop.pos_y;
    if dx * dx + dz * dz > 4.0 {
        return Err("Too far from item drop".to_string());
    }
    ctx.db.item_drop().id().delete(&item_drop_id);
    add_to_inventory(ctx, player.id, drop.item_def_id, drop.quantity);
    log::info!(
        "pickup_item: player={} picked up item_def={} x{}",
        player.id, drop.item_def_id, drop.quantity
    );
    Ok(())
}
```

- [ ] **Step 4: Add spawn_loot_drops helper and wire into apply_damage_to_enemy**

First add this helper function (before `apply_damage_to_enemy` in file):

```rust
/// Called when an enemy dies — consults LootTable and spawns ItemDrop rows.
fn spawn_loot_drops(ctx: &ReducerContext, enemy: &Enemy) {
    let entries: Vec<LootTable> = ctx.db.loot_table()
        .enemy_def_id()
        .filter(&enemy.enemy_def_id)
        .collect();

    for entry in entries {
        // Deterministic pseudo-random from enemy_id + timestamp
        let seed = enemy.id
            .wrapping_mul(0x9e3779b97f4a7c15)
            .wrapping_add(
                ctx.timestamp
                    .to_duration_since_unix_epoch()
                    .unwrap_or_default()
                    .as_micros() as u64,
            )
            .wrapping_add(entry.id);
        let roll = (seed % 101) as u32;
        if roll > entry.drop_chance {
            continue;
        }
        // Quantity in [min, max]
        let range = entry.max_quantity - entry.min_quantity + 1;
        let qty = entry.min_quantity + (seed.wrapping_mul(0x6c62272e07bb0142) % range as u64) as u32;
        // Scatter drop slightly around enemy position
        let offset_x = ((seed & 0xff) as f32 / 255.0 - 0.5) * 1.5;
        let offset_y = (((seed >> 8) & 0xff) as f32 / 255.0 - 0.5) * 1.5;
        ctx.db.item_drop().insert(ItemDrop {
            id:          0,
            zone_id:     enemy.zone_id,
            item_def_id: entry.item_def_id,
            quantity:    qty,
            pos_x:       enemy.position_x + offset_x,
            pos_y:       enemy.position_y + offset_y,
        });
    }
}
```

Then find the `if is_dead {` block inside `apply_damage_to_enemy` (around line 1376) and add a `spawn_loot_drops` call:

```rust
    if is_dead {
        log::info!("apply_damage_to_enemy: enemy={} killed by player={}", enemy_id, attacker_id);
        // Spawn loot drops
        let dead_enemy = ctx.db.enemy().id().find(&enemy_id).unwrap_or_else(|| ctx.db.enemy().id().find(&enemy_id).unwrap());
        // Re-fetch after update since we moved `enemy` into the update call
        if let Some(dead_enemy) = ctx.db.enemy().id().find(&enemy_id) {
            spawn_loot_drops(ctx, &dead_enemy);
        }
        // Schedule respawn if this enemy belongs to a spawn point
        if let Some(sp_id) = spawn_point_id {
```

**Note:** The `enemy` struct is moved into the `update` call before this block, so we re-fetch the row. The `spawn_point_id` was already saved before the update, so the respawn scheduling code after is unaffected.

- [ ] **Step 5: Build**

```bash
cd server && spacetime build
```
Expected: zero errors.

- [ ] **Step 6: Commit**

```bash
cd server && git add spacetimedb/src/lib.rs && git commit -m "feat(server): add LootTable/ItemDrop tables, pickup_item reducer, loot-on-death drops"
```

---

## Task 4: Deploy + Regenerate Bindings

**Files:**
- Run commands, no file edits

- [ ] **Step 1: Publish with --delete-data (new tables added)**

```bash
cd server && spacetime publish --server local zoneforge-server --delete-data
```
Expected: `Publishing module...` then success. `--delete-data` is required because new tables were added.

**After this:** recreate zone id=1 via the editor before testing the client (gotcha from memory — the client hardcodes zone_id=1 on player create).

- [ ] **Step 2: Regenerate client bindings**

```bash
cd client && spacetime generate \
  --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```
Expected: new files appear in `client/Assets/Scripts/autogen/Tables/` — `ItemDef.g.cs`, `Inventory.g.cs`, `Equipment.g.cs`, `ItemDrop.g.cs`, `LootTable.g.cs` — plus new `Reducers/` entries.

- [ ] **Step 3: Regenerate editor bindings**

```bash
cd editor && spacetime generate \
  --lang csharp \
  --out-dir Assets/Scripts/autogen \
  --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
```

- [ ] **Step 4: Commit generated bindings**

```bash
cd client && git add Assets/Scripts/autogen && git commit -m "chore(client): regenerate autogen bindings — inventory/loot schema"
cd editor && git add Assets/Scripts/autogen && git commit -m "chore(editor): regenerate autogen bindings — inventory/loot schema"
```

---

## Task 5: Client — SpacetimeDBManager + InventoryManager

**Files:**
- Modify: `client/Assets/Scripts/Runtime/SpacetimeDBManager.cs`
- Create: `client/Assets/Scripts/UI/InventoryManager.cs`

- [ ] **Step 1: Add events and subscriptions to SpacetimeDBManager**

In `SpacetimeDBManager.cs`, add these event declarations after the existing `OnPortalDeleted` event (around line 38):

```csharp
    public static event Action<ItemDefinition> OnItemDefInserted;
    public static event Action<ItemDefinition> OnItemDefDeleted;
    public static event Action<Inventory> OnInventoryInserted;
    public static event Action<Inventory, Inventory> OnInventoryUpdated;
    public static event Action<Inventory> OnInventoryDeleted;
    public static event Action<Equipment> OnEquipmentInserted;
    public static event Action<Equipment, Equipment> OnEquipmentUpdated;
    public static event Action<ItemDrop> OnItemDropInserted;
    public static event Action<ItemDrop> OnItemDropDeleted;
```

In `OnConnect`, add these lines to the `Subscribe` array (after the portal lines):

```csharp
                "SELECT * FROM item_def",
                "SELECT * FROM inventory",
                "SELECT * FROM equipment",
                $"SELECT * FROM item_drop WHERE zone_id = {CurrentZoneId}",
```

In `OnSubscriptionApplied`, add these registrations after the Portal callbacks (before `IsSubscribed = true`):

```csharp
        Conn.Db.ItemDef.OnInsert += (eventCtx, def) => OnItemDefInserted?.Invoke(def);
        Conn.Db.ItemDef.OnDelete += (eventCtx, def) => OnItemDefDeleted?.Invoke(def);
        Conn.Db.Inventory.OnInsert += (eventCtx, inv) => OnInventoryInserted?.Invoke(inv);
        Conn.Db.Inventory.OnUpdate += (eventCtx, oldInv, newInv) => OnInventoryUpdated?.Invoke(oldInv, newInv);
        Conn.Db.Inventory.OnDelete += (eventCtx, inv) => OnInventoryDeleted?.Invoke(inv);
        Conn.Db.Equipment.OnInsert += (eventCtx, eq) => OnEquipmentInserted?.Invoke(eq);
        Conn.Db.Equipment.OnUpdate += (eventCtx, oldEq, newEq) => OnEquipmentUpdated?.Invoke(oldEq, newEq);
        Conn.Db.ItemDrop.OnInsert += (eventCtx, drop) => OnItemDropInserted?.Invoke(drop);
        Conn.Db.ItemDrop.OnDelete += (eventCtx, drop) => OnItemDropDeleted?.Invoke(drop);
```

Also add item_drop subscription to `ReconnectForNewZone` so it re-filters on zone change. The new `OnConnect` callback already does this dynamically via `CurrentZoneId`, so `ReconnectForNewZone` is automatically correct.

- [ ] **Step 2: Create InventoryManager.cs**

Create `client/Assets/Scripts/UI/InventoryManager.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;
using SpacetimeDB.Types;

/// <summary>
/// Scene singleton. Caches the local player's inventory, equipment, and nearby
/// item drops. Fires events that InventoryUI and EquipmentUI listen to.
/// Follows EnemyManager pattern.
/// </summary>
public class InventoryManager : MonoBehaviour
{
    public static InventoryManager Instance { get; private set; }

    // Local player's inventory rows keyed by Inventory.id
    public readonly Dictionary<ulong, Inventory> Inventory = new();
    // Local player's equipment row (null until first equip)
    public Equipment Equipment;
    // All item definitions keyed by id
    public readonly Dictionary<ulong, ItemDefinition> ItemDefs = new();
    // Active drops in current zone keyed by ItemDrop.id
    public readonly Dictionary<ulong, ItemDrop> ItemDrops = new();

    public static event System.Action OnInventoryChanged;
    public static event System.Action OnEquipmentChanged;

    void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);

        SpacetimeDBManager.OnConnected         += OnConnected;
        SpacetimeDBManager.OnItemDefInserted   += OnItemDefInserted;
        SpacetimeDBManager.OnItemDefDeleted    += OnItemDefDeleted;
        SpacetimeDBManager.OnInventoryInserted += OnInventoryRowInserted;
        SpacetimeDBManager.OnInventoryUpdated  += OnInventoryRowUpdated;
        SpacetimeDBManager.OnInventoryDeleted  += OnInventoryRowDeleted;
        SpacetimeDBManager.OnEquipmentInserted += OnEquipmentRowInserted;
        SpacetimeDBManager.OnEquipmentUpdated  += OnEquipmentRowUpdated;
        SpacetimeDBManager.OnItemDropInserted  += OnItemDropInserted;
        SpacetimeDBManager.OnItemDropDeleted   += OnItemDropDeleted;
        SpacetimeDBManager.OnZoneChanged       += OnZoneChanged;
    }

    void OnDestroy()
    {
        SpacetimeDBManager.OnConnected         -= OnConnected;
        SpacetimeDBManager.OnItemDefInserted   -= OnItemDefInserted;
        SpacetimeDBManager.OnItemDefDeleted    -= OnItemDefDeleted;
        SpacetimeDBManager.OnInventoryInserted -= OnInventoryRowInserted;
        SpacetimeDBManager.OnInventoryUpdated  -= OnInventoryRowUpdated;
        SpacetimeDBManager.OnInventoryDeleted  -= OnInventoryRowDeleted;
        SpacetimeDBManager.OnEquipmentInserted -= OnEquipmentRowInserted;
        SpacetimeDBManager.OnEquipmentUpdated  -= OnEquipmentRowUpdated;
        SpacetimeDBManager.OnItemDropInserted  -= OnItemDropInserted;
        SpacetimeDBManager.OnItemDropDeleted   -= OnItemDropDeleted;
        SpacetimeDBManager.OnZoneChanged       -= OnZoneChanged;
    }

    void OnConnected()
    {
        // Backfill item defs
        foreach (var def in SpacetimeDBManager.Conn.Db.ItemDef.Iter())
            ItemDefs[def.Id] = def;

        // Backfill local player's inventory
        var localPlayerId = GetLocalPlayerId();
        if (localPlayerId == 0) return;

        foreach (var inv in SpacetimeDBManager.Conn.Db.Inventory.Iter())
            if (inv.PlayerId == localPlayerId) Inventory[inv.Id] = inv;

        var eq = SpacetimeDBManager.Conn.Db.Equipment.FilterByPlayerId(localPlayerId).FirstOrDefault();
        if (eq != null) Equipment = eq;

        foreach (var drop in SpacetimeDBManager.Conn.Db.ItemDrop.Iter())
            ItemDrops[drop.Id] = drop;

        OnInventoryChanged?.Invoke();
        OnEquipmentChanged?.Invoke();
    }

    static ulong GetLocalPlayerId()
    {
        foreach (var p in SpacetimeDBManager.Conn.Db.Player.Iter())
            if (p.Identity == SpacetimeDBManager.LocalIdentity) return p.Id;
        return 0;
    }

    void OnItemDefInserted(ItemDefinition def) => ItemDefs[def.Id] = def;
    void OnItemDefDeleted(ItemDefinition def)  => ItemDefs.Remove(def.Id);

    void OnInventoryRowInserted(Inventory row)
    {
        if (row.PlayerId != GetLocalPlayerId()) return;
        Inventory[row.Id] = row;
        OnInventoryChanged?.Invoke();
    }

    void OnInventoryRowUpdated(Inventory oldRow, Inventory newRow)
    {
        if (newRow.PlayerId != GetLocalPlayerId()) return;
        Inventory[newRow.Id] = newRow;
        OnInventoryChanged?.Invoke();
    }

    void OnInventoryRowDeleted(Inventory row)
    {
        if (row.PlayerId != GetLocalPlayerId()) return;
        Inventory.Remove(row.Id);
        OnInventoryChanged?.Invoke();
    }

    void OnEquipmentRowInserted(Equipment eq)
    {
        if (eq.PlayerId != GetLocalPlayerId()) return;
        Equipment = eq;
        OnEquipmentChanged?.Invoke();
    }

    void OnEquipmentRowUpdated(Equipment oldEq, Equipment newEq)
    {
        if (newEq.PlayerId != GetLocalPlayerId()) return;
        Equipment = newEq;
        OnEquipmentChanged?.Invoke();
    }

    void OnItemDropInserted(ItemDrop drop)
    {
        ItemDrops[drop.Id] = drop;
    }

    void OnItemDropDeleted(ItemDrop drop)
    {
        ItemDrops.Remove(drop.Id);
    }

    void OnZoneChanged(ulong _oldZoneId)
    {
        // Zone transfer reconnects with new zone filter — clear drops for old zone
        ItemDrops.Clear();
    }
}
```

- [ ] **Step 3: Add InventoryManager to SampleScene**

Open Unity, add an empty GameObject named `InventoryManager` to SampleScene and attach the `InventoryManager` component. (Position irrelevant — uses `DontDestroyOnLoad`.)

- [ ] **Step 4: Commit**

```bash
cd client && git add Assets/Scripts/Runtime/SpacetimeDBManager.cs Assets/Scripts/UI/InventoryManager.cs && git commit -m "feat(client): add inventory/equipment/item-drop subscriptions and InventoryManager"
```

---

## Task 6: Client — InventoryUI (I key, grid, drag-and-drop, tooltips)

**Files:**
- Create: `client/Assets/Scripts/UI/InventoryUI.cs`

- [ ] **Step 1: Create InventoryUI.cs**

Create `client/Assets/Scripts/UI/InventoryUI.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UIElements;
using SpacetimeDB.Types;

/// <summary>
/// Inventory panel. Toggle with I key.
/// 20-slot grid (4 columns × 5 rows) showing the local player's items.
/// Supports drag-and-drop between slots and hover tooltips.
/// Requires a UIDocument on the same GameObject.
/// </summary>
[RequireComponent(typeof(UIDocument))]
public class InventoryUI : MonoBehaviour
{
    public static InventoryUI Instance { get; private set; }

    private UIDocument _doc;
    private VisualElement _root;
    private VisualElement _panel;
    private VisualElement _grid;
    private VisualElement _tooltip;
    private Label         _tooltipName;
    private Label         _tooltipDesc;
    private Label         _tooltipStats;

    // 20 slots
    private const int SlotCount = 20;
    private readonly VisualElement[] _slots  = new VisualElement[SlotCount];
    private readonly ulong[]         _slotItemDefIds = new ulong[SlotCount];

    // Drag state
    private VisualElement _dragGhost;
    private int           _dragSourceSlot = -1;

    void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;
    }

    void OnEnable()
    {
        _doc  = GetComponent<UIDocument>();
        _root = _doc.rootVisualElement;
        _root.pickingMode = PickingMode.Ignore;
        BuildUI();
        _panel.style.display = DisplayStyle.None;

        InventoryManager.OnInventoryChanged += Refresh;
    }

    void OnDisable()
    {
        InventoryManager.OnInventoryChanged -= Refresh;
    }

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.I))
            TogglePanel();
    }

    void TogglePanel()
    {
        bool visible = _panel.style.display == DisplayStyle.Flex;
        _panel.style.display = visible ? DisplayStyle.None : DisplayStyle.Flex;
        if (!visible) Refresh();
    }

    void BuildUI()
    {
        _panel = new VisualElement();
        _panel.style.position         = Position.Absolute;
        _panel.style.right            = 10;
        _panel.style.top              = 10;
        _panel.style.width            = 220;
        _panel.style.backgroundColor  = new StyleColor(new Color(0.1f, 0.1f, 0.15f, 0.92f));
        _panel.style.borderTopLeftRadius     = 6;
        _panel.style.borderTopRightRadius    = 6;
        _panel.style.borderBottomLeftRadius  = 6;
        _panel.style.borderBottomRightRadius = 6;
        _panel.style.paddingTop    = 8;
        _panel.style.paddingBottom = 8;
        _panel.style.paddingLeft   = 8;
        _panel.style.paddingRight  = 8;

        var title = new Label("Inventory");
        title.style.color          = Color.white;
        title.style.unityFontStyleAndWeight = FontStyle.Bold;
        title.style.marginBottom   = 6;
        _panel.Add(title);

        _grid = new VisualElement();
        _grid.style.flexDirection = FlexDirection.Row;
        _grid.style.flexWrap      = Wrap.Wrap;
        _grid.style.width         = 200;
        _panel.Add(_grid);

        for (int i = 0; i < SlotCount; i++)
        {
            var slot = BuildSlot(i);
            _slots[i] = slot;
            _grid.Add(slot);
        }

        // Tooltip (initially hidden, shown on hover)
        _tooltip = new VisualElement();
        _tooltip.style.position         = Position.Absolute;
        _tooltip.style.backgroundColor  = new StyleColor(new Color(0.05f, 0.05f, 0.1f, 0.95f));
        _tooltip.style.borderTopLeftRadius     = 4;
        _tooltip.style.borderTopRightRadius    = 4;
        _tooltip.style.borderBottomLeftRadius  = 4;
        _tooltip.style.borderBottomRightRadius = 4;
        _tooltip.style.paddingTop    = 6;
        _tooltip.style.paddingBottom = 6;
        _tooltip.style.paddingLeft   = 8;
        _tooltip.style.paddingRight  = 8;
        _tooltip.style.minWidth      = 140;
        _tooltip.style.maxWidth      = 220;
        _tooltip.pickingMode         = PickingMode.Ignore;
        _tooltip.style.display       = DisplayStyle.None;

        _tooltipName  = new Label();
        _tooltipName.style.color = Color.white;
        _tooltipName.style.unityFontStyleAndWeight = FontStyle.Bold;
        _tooltipDesc  = new Label();
        _tooltipDesc.style.color = new Color(0.8f, 0.8f, 0.8f);
        _tooltipDesc.style.whiteSpace = WhiteSpace.Normal;
        _tooltipStats = new Label();
        _tooltipStats.style.color = new Color(0.5f, 0.9f, 0.5f);
        _tooltip.Add(_tooltipName);
        _tooltip.Add(_tooltipDesc);
        _tooltip.Add(_tooltipStats);

        _root.Add(_panel);
        _root.Add(_tooltip);
    }

    VisualElement BuildSlot(int index)
    {
        var slot = new VisualElement();
        slot.style.width   = 44;
        slot.style.height  = 44;
        slot.style.margin  = 2;
        slot.style.borderTopLeftRadius     = 3;
        slot.style.borderTopRightRadius    = 3;
        slot.style.borderBottomLeftRadius  = 3;
        slot.style.borderBottomRightRadius = 3;
        slot.style.backgroundColor = new StyleColor(new Color(0.2f, 0.2f, 0.25f, 1f));
        slot.userData = index;

        var qty = new Label("");
        qty.name = "qty";
        qty.style.position       = Position.Absolute;
        qty.style.bottom         = 2;
        qty.style.right          = 4;
        qty.style.color          = Color.white;
        qty.style.fontSize       = 11;
        qty.pickingMode          = PickingMode.Ignore;
        slot.Add(qty);

        // Hover: show tooltip
        slot.RegisterCallback<MouseEnterEvent>(evt => ShowTooltip(index, slot));
        slot.RegisterCallback<MouseLeaveEvent>(evt => _tooltip.style.display = DisplayStyle.None);
        slot.RegisterCallback<MouseMoveEvent>(evt => PositionTooltip(evt.mousePosition));

        // Drag-and-drop: pointer events
        int capturedIndex = index; // closure capture
        slot.RegisterCallback<PointerDownEvent>(evt => {
            if (_slotItemDefIds[capturedIndex] == 0) return;
            BeginDrag(capturedIndex, evt);
            evt.StopPropagation();
        });

        return slot;
    }

    void Refresh()
    {
        if (InventoryManager.Instance == null) return;
        // Clear all slots
        for (int i = 0; i < SlotCount; i++)
        {
            _slotItemDefIds[i] = 0;
            _slots[i].style.backgroundColor = new StyleColor(new Color(0.2f, 0.2f, 0.25f, 1f));
            (_slots[i].Q<Label>("qty")).text = "";
        }

        int slotIndex = 0;
        foreach (var kv in InventoryManager.Instance.Inventory)
        {
            if (slotIndex >= SlotCount) break;
            var inv = kv.Value;
            _slotItemDefIds[slotIndex] = inv.ItemDefId;

            if (InventoryManager.Instance.ItemDefs.TryGetValue(inv.ItemDefId, out var def))
                _slots[slotIndex].style.backgroundColor = new StyleColor(GetRarityColor(def.Rarity));

            var qtyLabel = _slots[slotIndex].Q<Label>("qty");
            qtyLabel.text = inv.Quantity > 1 ? inv.Quantity.ToString() : "";
            slotIndex++;
        }
    }

    // ── Drag-and-drop ────────────────────────────────────────────────────────

    void BeginDrag(int slotIndex, PointerDownEvent evt)
    {
        _dragSourceSlot = slotIndex;

        _dragGhost = new VisualElement();
        _dragGhost.style.width           = 40;
        _dragGhost.style.height          = 40;
        _dragGhost.style.position        = Position.Absolute;
        _dragGhost.style.backgroundColor = _slots[slotIndex].style.backgroundColor;
        _dragGhost.style.opacity         = 0.75f;
        _dragGhost.style.left            = evt.position.x - 20;
        _dragGhost.style.top             = evt.position.y - 20;
        _dragGhost.pickingMode           = PickingMode.Ignore;
        _root.Add(_dragGhost);

        _root.CapturePointer(evt.pointerId);
        _root.RegisterCallback<PointerMoveEvent>(OnDragMove);
        _root.RegisterCallback<PointerUpEvent>(OnDragRelease);
    }

    void OnDragMove(PointerMoveEvent evt)
    {
        if (_dragGhost == null) return;
        _dragGhost.style.left = evt.position.x - 20;
        _dragGhost.style.top  = evt.position.y - 20;
    }

    void OnDragRelease(PointerUpEvent evt)
    {
        if (_dragGhost == null) return;
        _root.ReleasePointer(evt.pointerId);
        _root.UnregisterCallback<PointerMoveEvent>(OnDragMove);
        _root.UnregisterCallback<PointerUpEvent>(OnDragRelease);

        // Find which slot is under the release point
        int targetSlot = FindSlotIndexAt(evt.position);
        if (targetSlot >= 0 && targetSlot != _dragSourceSlot)
            SwapSlots(_dragSourceSlot, targetSlot);

        _root.Remove(_dragGhost);
        _dragGhost      = null;
        _dragSourceSlot = -1;
    }

    int FindSlotIndexAt(Vector2 screenPos)
    {
        for (int i = 0; i < SlotCount; i++)
        {
            var bounds = _slots[i].worldBound;
            if (bounds.Contains(screenPos)) return i;
        }
        return -1;
    }

    /// Swap is purely visual (client-side slot reordering — no server call needed).
    void SwapSlots(int a, int b)
    {
        ulong tmpId  = _slotItemDefIds[a];
        StyleColor tmpColor = _slots[a].style.backgroundColor;

        _slotItemDefIds[a] = _slotItemDefIds[b];
        _slots[a].style.backgroundColor = _slots[b].style.backgroundColor;
        (_slots[a].Q<Label>("qty")).text = (_slots[b].Q<Label>("qty")).text;

        _slotItemDefIds[b] = tmpId;
        _slots[b].style.backgroundColor = tmpColor;
        // qty labels
        string aqty = (_slots[a].Q<Label>("qty")).text;
        (_slots[a].Q<Label>("qty")).text = (_slots[b].Q<Label>("qty")).text;
        (_slots[b].Q<Label>("qty")).text = aqty;
    }

    // ── Tooltip ──────────────────────────────────────────────────────────────

    void ShowTooltip(int slotIndex, VisualElement slot)
    {
        ulong defId = _slotItemDefIds[slotIndex];
        if (defId == 0) { _tooltip.style.display = DisplayStyle.None; return; }
        if (!InventoryManager.Instance.ItemDefs.TryGetValue(defId, out var def))
        { _tooltip.style.display = DisplayStyle.None; return; }

        _tooltipName.text = $"{def.Name}  [{def.Rarity}]";
        _tooltipDesc.text = def.Description;
        _tooltipStats.text = BuildStatLine(def);
        _tooltip.style.display = DisplayStyle.Flex;
    }

    void PositionTooltip(Vector2 mousePos)
    {
        if (_tooltip.style.display != DisplayStyle.Flex) return;
        _tooltip.style.left = mousePos.x + 12;
        _tooltip.style.top  = mousePos.y + 4;
    }

    string BuildStatLine(ItemDefinition def)
    {
        var parts = new System.Text.StringBuilder();
        if (def.DamageBonus != 0) parts.Append($"+{def.DamageBonus} DMG  ");
        if (def.ArmorBonus  != 0) parts.Append($"+{def.ArmorBonus} ARM  ");
        if (def.Healing     != 0) parts.Append($"+{def.Healing} HP  ");
        if (def.Value       > 0)  parts.Append($"{def.Value}g");
        return parts.ToString();
    }

    static Color GetRarityColor(Rarity rarity) => rarity switch
    {
        Rarity.Common   => new Color(0.35f, 0.35f, 0.4f),
        Rarity.Uncommon => new Color(0.1f,  0.55f, 0.1f),
        Rarity.Rare     => new Color(0.1f,  0.3f,  0.8f),
        Rarity.Epic     => new Color(0.6f,  0.1f,  0.8f),
        _               => new Color(0.35f, 0.35f, 0.4f),
    };
}
```

- [ ] **Step 2: Add InventoryUI to SampleScene**

In Unity:
1. Create an empty GameObject named `InventoryUI` in SampleScene.
2. Add a `UIDocument` component. Set **Sort Order = 50**.
3. Add the `InventoryUI` component.
4. No UXML needed — UI is built entirely in code.

- [ ] **Step 3: Play-test inventory UI**

Enter Play mode. Press I — the inventory panel should appear in the top-right corner. It will be empty until items are added via `give_item` admin call.

- [ ] **Step 4: Commit**

```bash
cd client && git add Assets/Scripts/UI/InventoryUI.cs && git commit -m "feat(client): add InventoryUI — grid panel, drag-and-drop, tooltips (I key)"
```

---

## Task 7: Client — EquipmentUI (C key, character sheet)

**Files:**
- Create: `client/Assets/Scripts/UI/EquipmentUI.cs`

- [ ] **Step 1: Create EquipmentUI.cs**

Create `client/Assets/Scripts/UI/EquipmentUI.cs`:

```csharp
using UnityEngine;
using UnityEngine.UIElements;
using SpacetimeDB.Types;

/// <summary>
/// Equipment character sheet. Toggle with C key.
/// Shows three equipment slots (Weapon, Armor, Accessory).
/// Click a filled slot → calls unequip_item reducer.
/// </summary>
[RequireComponent(typeof(UIDocument))]
public class EquipmentUI : MonoBehaviour
{
    private UIDocument   _doc;
    private VisualElement _root;
    private VisualElement _panel;
    private VisualElement _weaponSlot;
    private VisualElement _armorSlot;
    private VisualElement _accessorySlot;
    private Label        _weaponLabel;
    private Label        _armorLabel;
    private Label        _accessoryLabel;

    void OnEnable()
    {
        _doc  = GetComponent<UIDocument>();
        _root = _doc.rootVisualElement;
        _root.pickingMode = PickingMode.Ignore;
        BuildUI();
        _panel.style.display = DisplayStyle.None;

        InventoryManager.OnEquipmentChanged += Refresh;
        InventoryManager.OnInventoryChanged += Refresh; // item defs may load after
    }

    void OnDisable()
    {
        InventoryManager.OnEquipmentChanged -= Refresh;
        InventoryManager.OnInventoryChanged -= Refresh;
    }

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.C))
            TogglePanel();
    }

    void TogglePanel()
    {
        bool visible = _panel.style.display == DisplayStyle.Flex;
        _panel.style.display = visible ? DisplayStyle.None : DisplayStyle.Flex;
        if (!visible) Refresh();
    }

    void BuildUI()
    {
        _panel = new VisualElement();
        _panel.style.position         = Position.Absolute;
        _panel.style.right            = 240;  // left of inventory panel
        _panel.style.top              = 10;
        _panel.style.width            = 180;
        _panel.style.backgroundColor  = new StyleColor(new Color(0.1f, 0.1f, 0.15f, 0.92f));
        _panel.style.borderTopLeftRadius     = 6;
        _panel.style.borderTopRightRadius    = 6;
        _panel.style.borderBottomLeftRadius  = 6;
        _panel.style.borderBottomRightRadius = 6;
        _panel.style.paddingTop    = 8;
        _panel.style.paddingBottom = 8;
        _panel.style.paddingLeft   = 8;
        _panel.style.paddingRight  = 8;

        var title = new Label("Equipment");
        title.style.color = Color.white;
        title.style.unityFontStyleAndWeight = FontStyle.Bold;
        title.style.marginBottom = 8;
        _panel.Add(title);

        (_weaponSlot,    _weaponLabel)    = AddSlot("Weapon");
        (_armorSlot,     _armorLabel)     = AddSlot("Armor");
        (_accessorySlot, _accessoryLabel) = AddSlot("Accessory");

        _weaponSlot.RegisterCallback<ClickEvent>(_ => TryUnequip("weapon"));
        _armorSlot.RegisterCallback<ClickEvent>(_ => TryUnequip("armor"));
        _accessorySlot.RegisterCallback<ClickEvent>(_ => TryUnequip("accessory"));

        _root.Add(_panel);
    }

    (VisualElement slot, Label label) AddSlot(string slotName)
    {
        var row = new VisualElement();
        row.style.flexDirection  = FlexDirection.Row;
        row.style.alignItems     = Align.Center;
        row.style.marginBottom   = 6;

        var headerLabel = new Label(slotName);
        headerLabel.style.color  = new Color(0.7f, 0.7f, 0.7f);
        headerLabel.style.width  = 70;

        var slot = new VisualElement();
        slot.style.width   = 44;
        slot.style.height  = 44;
        slot.style.borderTopLeftRadius     = 3;
        slot.style.borderTopRightRadius    = 3;
        slot.style.borderBottomLeftRadius  = 3;
        slot.style.borderBottomRightRadius = 3;
        slot.style.backgroundColor = new StyleColor(new Color(0.2f, 0.2f, 0.25f));
        slot.style.justifyContent  = Justify.Center;
        slot.style.alignItems      = Align.Center;

        var label = new Label("—");
        label.style.color    = Color.white;
        label.style.fontSize = 10;
        label.pickingMode    = PickingMode.Ignore;
        slot.Add(label);

        row.Add(headerLabel);
        row.Add(slot);
        _panel.Add(row);
        return (slot, label);
    }

    void Refresh()
    {
        if (InventoryManager.Instance == null) return;
        var eq = InventoryManager.Instance.Equipment;
        RefreshSlot(_weaponSlot,    _weaponLabel,    eq?.WeaponId);
        RefreshSlot(_armorSlot,     _armorLabel,     eq?.ArmorId);
        RefreshSlot(_accessorySlot, _accessoryLabel, eq?.AccessoryId);
    }

    void RefreshSlot(VisualElement slot, Label label, ulong? itemDefId)
    {
        if (itemDefId == null || itemDefId == 0)
        {
            slot.style.backgroundColor = new StyleColor(new Color(0.2f, 0.2f, 0.25f));
            label.text = "—";
            return;
        }
        if (InventoryManager.Instance.ItemDefs.TryGetValue(itemDefId.Value, out var def))
        {
            slot.style.backgroundColor = new StyleColor(GetRarityColor(def.Rarity));
            label.text = def.Name.Length > 10 ? def.Name[..10] + "…" : def.Name;
        }
    }

    void TryUnequip(string slot)
    {
        if (!SpacetimeDBManager.IsSubscribed) return;
        Reducer.UnequipItem(SpacetimeDBManager.Conn, slot);
    }

    static Color GetRarityColor(Rarity rarity) => rarity switch
    {
        Rarity.Common   => new Color(0.35f, 0.35f, 0.4f),
        Rarity.Uncommon => new Color(0.1f,  0.55f, 0.1f),
        Rarity.Rare     => new Color(0.1f,  0.3f,  0.8f),
        Rarity.Epic     => new Color(0.6f,  0.1f,  0.8f),
        _               => new Color(0.35f, 0.35f, 0.4f),
    };
}
```

- [ ] **Step 2: Add EquipmentUI to SampleScene**

In Unity:
1. Create empty GO named `EquipmentUI`.
2. Add `UIDocument` (Sort Order = 50).
3. Add `EquipmentUI` component.

- [ ] **Step 3: Commit**

```bash
cd client && git add Assets/Scripts/UI/EquipmentUI.cs && git commit -m "feat(client): add EquipmentUI — character sheet with equip slots (C key)"
```

---

## Task 8: Client — ItemPickupManager (world drops)

**Files:**
- Create: `client/Assets/Scripts/Zone/ItemPickupManager.cs`

- [ ] **Step 1: Create ItemPickupManager.cs**

Create `client/Assets/Scripts/Zone/ItemPickupManager.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;
using SpacetimeDB.Types;

/// <summary>
/// Each frame finds the nearest ItemDrop within 1.5 units of the local player.
/// Shows an on-screen "F — Pick Up [item name]" prompt.
/// Pressing F calls Reducer.PickupItem.
/// </summary>
public class ItemPickupManager : MonoBehaviour
{
    public static ItemPickupManager Instance { get; private set; }

    private const float PickupRadius = 1.5f;

    // Prompt label rendered via a small screen-space GUI
    private ulong  _nearestDropId;
    private string _nearestDropName;
    private bool   _promptVisible;

    void Awake()
    {
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);
    }

    void Update()
    {
        _promptVisible = false;
        _nearestDropId = 0;

        if (!SpacetimeDBManager.IsSubscribed || InventoryManager.Instance == null) return;

        // Find local player position
        Vector3 playerPos = Vector3.zero;
        bool foundPlayer = false;
        foreach (var p in SpacetimeDBManager.Conn.Db.Player.Iter())
        {
            if (p.Identity == SpacetimeDBManager.LocalIdentity)
            {
                playerPos  = new Vector3(p.PositionX, 0, p.PositionY);
                foundPlayer = true;
                break;
            }
        }
        if (!foundPlayer) return;

        float bestDistSq = PickupRadius * PickupRadius;
        foreach (var kv in InventoryManager.Instance.ItemDrops)
        {
            var drop = kv.Value;
            var dropPos = new Vector3(drop.PosX, 0, drop.PosY);
            float distSq = (playerPos - dropPos).sqrMagnitude;
            if (distSq < bestDistSq)
            {
                bestDistSq   = distSq;
                _nearestDropId = drop.Id;
                InventoryManager.Instance.ItemDefs.TryGetValue(drop.ItemDefId, out var def);
                _nearestDropName = def != null
                    ? $"{def.Name} x{drop.Quantity}"
                    : $"Item x{drop.Quantity}";
                _promptVisible = true;
            }
        }

        if (_promptVisible && Input.GetKeyDown(KeyCode.F))
        {
            Reducer.PickupItem(SpacetimeDBManager.Conn, _nearestDropId);
            _promptVisible = false;
        }
    }

    void OnGUI()
    {
        if (!_promptVisible) return;
        var style = new GUIStyle(GUI.skin.box)
        {
            fontSize  = 16,
            alignment = TextAnchor.MiddleCenter,
        };
        style.normal.textColor = Color.white;
        float w = 280, h = 32;
        GUI.Box(new Rect(Screen.width / 2f - w / 2f, Screen.height - 80, w, h),
            $"F — Pick Up {_nearestDropName}", style);
    }
}
```

- [ ] **Step 2: Add ItemPickupManager to SampleScene**

In Unity, add an empty GO named `ItemPickupManager` to SampleScene and attach `ItemPickupManager`.

- [ ] **Step 3: Commit**

```bash
cd client && git add Assets/Scripts/Zone/ItemPickupManager.cs && git commit -m "feat(client): add ItemPickupManager — proximity pickup with F key prompt"
```

---

## Task 9: Editor — SpacetimeDBManager + LootCreationPanel

**Files:**
- Modify: `editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs`
- Create: `editor/Assets/Scripts/Runtime/LootCreationPanel.cs`

- [ ] **Step 1: Add item_def and loot_table subscriptions to editor SpacetimeDBManager**

In `editor/Assets/Scripts/Runtime/SpacetimeDBManager.cs`:

Add these events after the existing `OnPortalDeleted` event declaration:

```csharp
    public static event Action<ItemDefinition> OnItemDefInserted;
    public static event Action<ItemDefinition> OnItemDefDeleted;
    public static event Action<LootTable> OnLootTableInserted;
    public static event Action<LootTable> OnLootTableDeleted;
```

In `OnConnect`, add to the Subscribe array:

```csharp
                "SELECT * FROM item_def",
                "SELECT * FROM loot_table",
                "SELECT * FROM enemy_def",
```

In `OnSubscriptionApplied`, add before the closing brace:

```csharp
        Conn.Db.ItemDef.OnInsert += (eventCtx, def) => OnItemDefInserted?.Invoke(def);
        Conn.Db.ItemDef.OnDelete += (eventCtx, def) => OnItemDefDeleted?.Invoke(def);
        Conn.Db.LootTable.OnInsert += (eventCtx, lt) => OnLootTableInserted?.Invoke(lt);
        Conn.Db.LootTable.OnDelete += (eventCtx, lt) => OnLootTableDeleted?.Invoke(lt);
```

- [ ] **Step 2: Create LootCreationPanel.cs**

Create `editor/Assets/Scripts/Runtime/LootCreationPanel.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UIElements;
using SpacetimeDB.Types;

/// <summary>
/// Editor panel for managing item definitions and loot tables.
/// Toggle with L key. Allows creating/deleting ItemDefinitions and
/// assigning loot drops to enemy types.
/// Admin-only actions — server will reject non-admin callers.
/// </summary>
[RequireComponent(typeof(UIDocument))]
public class LootCreationPanel : MonoBehaviour
{
    private UIDocument    _doc;
    private VisualElement _root;
    private VisualElement _panel;
    private bool          _isVisible;

    // Item def creation fields
    private TextField   _nameField;
    private TextField   _descField;
    private EnumField   _typeField;
    private EnumField   _rarityField;
    private IntegerField _damageField;
    private IntegerField _armorField;
    private IntegerField _healingField;
    private IntegerField _valueField;
    private ScrollView  _itemDefList;

    // Loot table creation fields
    private DropdownField _enemyDefDropdown;
    private DropdownField _itemDefDropdown;
    private IntegerField  _chanceField;
    private IntegerField  _minQtyField;
    private IntegerField  _maxQtyField;
    private ScrollView    _lootTableList;

    // Cached data
    private readonly List<EnemyDefinition> _enemyDefs = new();
    private readonly List<ItemDefinition>  _itemDefs  = new();

    void OnEnable()
    {
        _doc  = GetComponent<UIDocument>();
        _root = _doc.rootVisualElement;
        _root.pickingMode = PickingMode.Ignore;
        BuildUI();
        _panel.style.display = DisplayStyle.None;

        SpacetimeDBManager.OnConnected         += OnConnected;
        SpacetimeDBManager.OnItemDefInserted   += _ => RefreshItemDefList();
        SpacetimeDBManager.OnItemDefDeleted    += _ => RefreshItemDefList();
        SpacetimeDBManager.OnLootTableInserted += _ => RefreshLootTableList();
        SpacetimeDBManager.OnLootTableDeleted  += _ => RefreshLootTableList();
    }

    void OnDisable()
    {
        SpacetimeDBManager.OnConnected         -= OnConnected;
    }

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.L)) TogglePanel();
    }

    void TogglePanel()
    {
        _isVisible = !_isVisible;
        _panel.style.display = _isVisible ? DisplayStyle.Flex : DisplayStyle.None;
        if (_isVisible) { RefreshItemDefList(); RefreshLootTableList(); RefreshDropdowns(); }
    }

    void OnConnected()
    {
        // Backfill
        _enemyDefs.Clear(); _itemDefs.Clear();
        foreach (var d in SpacetimeDBManager.Conn.Db.EnemyDef.Iter()) _enemyDefs.Add(d);
        foreach (var d in SpacetimeDBManager.Conn.Db.ItemDef.Iter())  _itemDefs.Add(d);
        if (_isVisible) { RefreshDropdowns(); RefreshItemDefList(); RefreshLootTableList(); }
    }

    void BuildUI()
    {
        _panel = new VisualElement();
        _panel.style.position        = Position.Absolute;
        _panel.style.left            = 10;
        _panel.style.bottom          = 10;
        _panel.style.width           = 300;
        _panel.style.maxHeight       = 500;
        _panel.style.backgroundColor = new StyleColor(new Color(0.08f, 0.08f, 0.12f, 0.93f));
        _panel.style.borderTopLeftRadius     = 6;
        _panel.style.borderTopRightRadius    = 6;
        _panel.style.borderBottomLeftRadius  = 6;
        _panel.style.borderBottomRightRadius = 6;
        _panel.style.paddingTop    = 8;
        _panel.style.paddingBottom = 8;
        _panel.style.paddingLeft   = 8;
        _panel.style.paddingRight  = 8;
        _panel.style.overflow      = Overflow.Hidden;

        var scrollOuter = new ScrollView(ScrollViewMode.Vertical);
        scrollOuter.style.flexGrow = 1;

        // ── Item Definitions section ─────────────────────────────────────────
        var itemSection = MakeSectionLabel("Item Definitions");
        scrollOuter.Add(itemSection);

        _nameField   = MakeTextField("Name");
        _descField   = MakeTextField("Description");
        _typeField   = new EnumField("Type", ItemType.Weapon);   StyleField(_typeField);
        _rarityField = new EnumField("Rarity", Rarity.Common);   StyleField(_rarityField);
        _damageField  = new IntegerField("Damage Bonus");  StyleField(_damageField);
        _armorField   = new IntegerField("Armor Bonus");   StyleField(_armorField);
        _healingField = new IntegerField("Healing");       StyleField(_healingField);
        _valueField   = new IntegerField("Value (g)");     StyleField(_valueField);

        scrollOuter.Add(_nameField);
        scrollOuter.Add(_descField);
        scrollOuter.Add(_typeField);
        scrollOuter.Add(_rarityField);
        scrollOuter.Add(_damageField);
        scrollOuter.Add(_armorField);
        scrollOuter.Add(_healingField);
        scrollOuter.Add(_valueField);

        var createItemBtn = new Button(CreateItemDef) { text = "Create Item" };
        StyleButton(createItemBtn, new Color(0.1f, 0.5f, 0.1f));
        scrollOuter.Add(createItemBtn);

        _itemDefList = new ScrollView(ScrollViewMode.Vertical);
        _itemDefList.style.maxHeight = 100;
        _itemDefList.style.marginTop = 4;
        scrollOuter.Add(_itemDefList);

        // Separator
        scrollOuter.Add(MakeSeparator());

        // ── Loot Tables section ──────────────────────────────────────────────
        scrollOuter.Add(MakeSectionLabel("Loot Tables"));

        _enemyDefDropdown = new DropdownField("Enemy Type", new List<string>{"(none)"}, 0);
        StyleField(_enemyDefDropdown);
        _itemDefDropdown  = new DropdownField("Item",       new List<string>{"(none)"}, 0);
        StyleField(_itemDefDropdown);
        _chanceField  = new IntegerField("Drop Chance %");  StyleField(_chanceField);
        _minQtyField  = new IntegerField("Min Quantity");   StyleField(_minQtyField);
        _maxQtyField  = new IntegerField("Max Quantity");   StyleField(_maxQtyField);
        _chanceField.value = 50; _minQtyField.value = 1; _maxQtyField.value = 1;

        scrollOuter.Add(_enemyDefDropdown);
        scrollOuter.Add(_itemDefDropdown);
        scrollOuter.Add(_chanceField);
        scrollOuter.Add(_minQtyField);
        scrollOuter.Add(_maxQtyField);

        var createLootBtn = new Button(CreateLootEntry) { text = "Add Loot Entry" };
        StyleButton(createLootBtn, new Color(0.1f, 0.3f, 0.6f));
        scrollOuter.Add(createLootBtn);

        _lootTableList = new ScrollView(ScrollViewMode.Vertical);
        _lootTableList.style.maxHeight = 100;
        _lootTableList.style.marginTop = 4;
        scrollOuter.Add(_lootTableList);

        _panel.Add(scrollOuter);
        _root.Add(_panel);
    }

    // ── Actions ──────────────────────────────────────────────────────────────

    void CreateItemDef()
    {
        if (!SpacetimeDBManager.IsSubscribed) return;
        string name = _nameField.value.Trim();
        if (string.IsNullOrEmpty(name)) { Debug.LogWarning("[LootPanel] Name required"); return; }

        Reducer.CreateItemDef(
            SpacetimeDBManager.Conn,
            name,
            _descField.value,
            (ItemType)_typeField.value,
            (Rarity)_rarityField.value,
            name.ToLower().Replace(" ", "_"),  // icon_name
            _damageField.value,
            _armorField.value,
            _healingField.value,
            (uint)Mathf.Max(0, _valueField.value)
        );
    }

    void CreateLootEntry()
    {
        if (!SpacetimeDBManager.IsSubscribed) return;
        int enemyIdx = _enemyDefDropdown.index;
        int itemIdx  = _itemDefDropdown.index;
        if (enemyIdx <= 0 || itemIdx <= 0 || _enemyDefs.Count == 0 || _itemDefs.Count == 0)
        { Debug.LogWarning("[LootPanel] Select enemy and item"); return; }

        var enemyDef = _enemyDefs[enemyIdx - 1]; // -1 for "(none)" at index 0
        var itemDef  = _itemDefs[itemIdx  - 1];
        int chance = Mathf.Clamp(_chanceField.value, 0, 100);
        uint minQ  = (uint)Mathf.Max(1, _minQtyField.value);
        uint maxQ  = (uint)Mathf.Max(minQ, _maxQtyField.value);

        Reducer.CreateLootTable(
            SpacetimeDBManager.Conn,
            enemyDef.Id,
            itemDef.Id,
            (uint)chance,
            minQ,
            maxQ
        );
    }

    // ── List refreshes ────────────────────────────────────────────────────────

    void RefreshItemDefList()
    {
        _itemDefList.Clear();
        _itemDefs.Clear();
        foreach (var def in SpacetimeDBManager.Conn.Db.ItemDef.Iter())
            _itemDefs.Add(def);

        foreach (var def in _itemDefs)
        {
            var row = new VisualElement();
            row.style.flexDirection = FlexDirection.Row;
            row.style.marginBottom  = 2;

            var lbl = new Label($"{def.Name}  [{def.Rarity}]");
            lbl.style.color    = Color.white;
            lbl.style.flexGrow = 1;

            var delBtn = new Button() { text = "✕" };
            delBtn.style.width      = 24;
            delBtn.style.color      = Color.red;
            ulong capturedId = def.Id;
            delBtn.clicked += () => {
                if (SpacetimeDBManager.IsSubscribed)
                    Reducer.DeleteItemDef(SpacetimeDBManager.Conn, capturedId);
            };

            row.Add(lbl); row.Add(delBtn);
            _itemDefList.Add(row);
        }
        RefreshDropdowns();
    }

    void RefreshLootTableList()
    {
        _lootTableList.Clear();
        foreach (var lt in SpacetimeDBManager.Conn.Db.LootTable.Iter())
        {
            _enemyDefs.TryFind(e => e.Id == lt.EnemyDefId, out var eDef);
            _itemDefs.TryFind(i => i.Id  == lt.ItemDefId,  out var iDef);
            string label = $"{eDef?.Name ?? "?"} → {iDef?.Name ?? "?"} {lt.DropChance}%";

            var row = new VisualElement();
            row.style.flexDirection = FlexDirection.Row;
            row.style.marginBottom  = 2;

            var lbl = new Label(label);
            lbl.style.color    = Color.white;
            lbl.style.flexGrow = 1;

            var delBtn = new Button() { text = "✕" };
            delBtn.style.color = Color.red;
            delBtn.style.width = 24;
            ulong capturedId = lt.Id;
            delBtn.clicked += () => {
                if (SpacetimeDBManager.IsSubscribed)
                    Reducer.DeleteLootTable(SpacetimeDBManager.Conn, capturedId);
            };

            row.Add(lbl); row.Add(delBtn);
            _lootTableList.Add(row);
        }
    }

    void RefreshDropdowns()
    {
        var enemyChoices = new List<string> { "(none)" };
        foreach (var d in _enemyDefs) enemyChoices.Add(d.Name);
        _enemyDefDropdown.choices = enemyChoices;
        if (_enemyDefDropdown.index >= enemyChoices.Count) _enemyDefDropdown.index = 0;

        var itemChoices = new List<string> { "(none)" };
        foreach (var d in _itemDefs) itemChoices.Add($"{d.Name} [{d.Rarity}]");
        _itemDefDropdown.choices = itemChoices;
        if (_itemDefDropdown.index >= itemChoices.Count) _itemDefDropdown.index = 0;
    }

    // ── UI helpers ────────────────────────────────────────────────────────────

    TextField MakeTextField(string label)
    {
        var f = new TextField(label);
        StyleField(f);
        return f;
    }

    static void StyleField(VisualElement f)
    {
        f.style.marginBottom = 4;
    }

    static VisualElement MakeSeparator()
    {
        var sep = new VisualElement();
        sep.style.height          = 1;
        sep.style.backgroundColor = new StyleColor(new Color(0.3f, 0.3f, 0.3f));
        sep.style.marginTop       = 8;
        sep.style.marginBottom    = 8;
        return sep;
    }

    static Label MakeSectionLabel(string text)
    {
        var lbl = new Label(text);
        lbl.style.color = Color.white;
        lbl.style.unityFontStyleAndWeight = FontStyle.Bold;
        lbl.style.marginBottom = 4;
        return lbl;
    }

    static void StyleButton(Button btn, Color color)
    {
        btn.style.backgroundColor = new StyleColor(color);
        btn.style.color           = Color.white;
        btn.style.marginTop       = 2;
        btn.style.marginBottom    = 6;
    }
}

// Extension helper (not in standard Unity API)
static class ListExtensions
{
    public static bool TryFind<T>(this List<T> list, System.Predicate<T> match, out T result)
    {
        int idx = list.FindIndex(match);
        if (idx >= 0) { result = list[idx]; return true; }
        result = default; return false;
    }
}
```

- [ ] **Step 2: Add LootCreationPanel to editor SampleScene**

In Unity (editor project):
1. Create empty GO named `LootCreationPanel`.
2. Add `UIDocument` (Sort Order = 60).
3. Add `LootCreationPanel` component.

- [ ] **Step 3: Commit**

```bash
cd editor && git add Assets/Scripts/Runtime/SpacetimeDBManager.cs Assets/Scripts/Runtime/LootCreationPanel.cs && git commit -m "feat(editor): add LootCreationPanel — item definition and loot table authoring (L key)"
```

---

## Task 10: Integration Test + PROGRESS.md Update

- [ ] **Step 1: Smoke test — create items and verify inventory**

1. Open editor, press L. Create an item: Name "Iron Sword", Type Weapon, Rarity Common, Damage Bonus 5.
2. Open `spacetime sql zoneforge-server --server local "SELECT * FROM item_def"` — should show 1 row.
3. In client, run `spacetime call zoneforge-server give_item --server local '{"player_id":1,"item_def_id":1,"quantity":1}'` (use actual player_id from SQL).
4. Press I in client — item should appear in slot 1 with a green (common) background.

- [ ] **Step 2: Test equip flow**

1. With an item in inventory (step 1), open Unity console — should show no errors.
2. Right-click the item slot in inventory and press enter or implement a quick "equip" button: for now, call the reducer directly from spacetime CLI:
   `spacetime call zoneforge-server equip_item --server local '{"item_def_id":1}'`
   (Or wire a double-click to call `Reducer.EquipItem` — see optional enhancement below.)
3. Press C — equipment panel should show "Iron Sword" in Weapon slot.
4. Click the Weapon slot in equipment panel — `unequip_item` is called, item returns to inventory.

- [ ] **Step 3: Test loot-on-death**

1. In editor, press L. Add loot entry: Skeleton → Iron Sword, 100% chance, qty 1.
2. In client, kill a Skeleton enemy (use abilities). Verify an `ItemDrop` row appears in the world.
3. Walk near the drop — "F — Pick Up Iron Sword x1" prompt should appear.
4. Press F — item moves to inventory.

- [ ] **Step 4: Optional — wire equip to double-click in InventoryUI**

In `InventoryUI.cs`, in the `BuildSlot` method, add after the pointer-down registration:

```csharp
        // Double-click equips item
        slot.RegisterCallback<ClickEvent>(evt => {
            if (evt.clickCount == 2 && _slotItemDefIds[capturedIndex] != 0)
            {
                Reducer.EquipItem(SpacetimeDBManager.Conn, _slotItemDefIds[capturedIndex]);
            }
        });
```

- [ ] **Step 5: Update PROGRESS.md**

Edit `docs/design/PROGRESS.md`. Replace the Group 13 checklist with all boxes checked and update the "Current Focus" header:

```markdown
## Current Focus

Phase 5 Group 13 complete. Inventory, equipment, loot-on-death, and editor loot authoring all implemented.

Completed phases: 1 · 2 · 3 · 4 (local) · 5 Group 12 · 5 Group 13
```

Update each Group 13 item to `[x]`:

```markdown
### Group 13 — Inventory & Loot

- [x] `Item` table (item definitions) — done in Group 12 *(actually implemented in Group 13)*
- [x] `Inventory` table (player_id, item_def_id, quantity)
- [x] `Equipment` table (weapon, armor, accessory slots)
- [x] Inventory, equip, and loot reducers
- [x] Inventory grid UI with drag-and-drop and tooltips
- [x] Equipment character sheet UI
- [x] Loot table system and drop-on-death
- [x] Pickup item reducer (collision-based)
- [x] **Editor tool:** Loot creation panel — define item templates and loot tables
```

- [ ] **Step 6: Commit everything**

```bash
cd /home/beek/zoneforge
git add docs/design/PROGRESS.md && git commit -m "docs: mark Phase 5 Group 13 complete in PROGRESS.md"
# Commit submodule pointer bumps in umbrella
git add client editor server && git commit -m "feat: Phase 5 Group 13 — Inventory, Equipment & Loot"
```

---

## Self-Review

**Spec coverage check:**
- `Equipment` table ✓ Task 2
- Inventory, equip, loot reducers ✓ Tasks 2–3
- Inventory grid UI + drag-and-drop + tooltips ✓ Task 6
- Equipment character sheet UI ✓ Task 7
- Loot table system and drop-on-death ✓ Tasks 3, 8
- Pickup item reducer (collision-based) ✓ Task 3 (2-unit radius server check + F-key client)
- Editor loot creation panel ✓ Task 9

**Type consistency check:**
- `ItemType` enum: defined Task 1, used in `equip_item` (Task 2), `LootCreationPanel` (Task 9) ✓
- `Rarity` enum: defined Task 1, used in `InventoryUI.GetRarityColor`, `EquipmentUI.GetRarityColor`, `LootCreationPanel` ✓
- `InventoryManager.Equipment` field: `Equipment` type (nullable), read in `EquipmentUI.Refresh` as `eq?.WeaponId` ✓
- `Reducer.EquipItem` / `Reducer.UnequipItem` / `Reducer.PickupItem`: generated from `equip_item`, `unequip_item`, `pickup_item` reducers ✓
- `SpacetimeDBManager.OnZoneChanged` fires with OLD zone id (ulong) — `ItemPickupManager` doesn't use it directly, `InventoryManager.OnZoneChanged` receives it correctly ✓

**Potential issue:** `List<T>.TryFind` extension in `LootCreationPanel.cs` is defined as a static class `ListExtensions` at the bottom of the same file. This will compile fine in Unity but watch for namespace conflicts if another file defines the same extension. If there's a conflict, move `ListExtensions` to a dedicated `ListExtensions.cs` file.
