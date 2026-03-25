# AI NPC Interaction System — Design Spec

**Date:** 2026-03-25
**Status:** Design complete — no current implementation plans. Potential post-ship feature pending business model decisions.

---

## Context

ZoneForge currently has no NPC, quest, or dialogue system. Phase 5 Group 12 plans a traditional node-based dialogue + quest table system. This design describes a **runtime LLM integration layer** that would let each NPC respond dynamically to free-text player input while still being able to trigger structured game actions (start quest, give item, open shop, etc.).

The goal: NPCs that feel alive — players can ask anything and get an in-character response — while quest mechanics remain deterministic and server-validated.

This feature is intentionally scoped out of the current roadmap. Per-call LLM API costs require a resolved business model before committing to it. The design is preserved here for reference.

---

## Architecture: Middleware Gateway

SpacetimeDB WASM reducers cannot make external HTTP calls. All LLM communication goes through a **stateless Python gateway service** that sits between the Unity client and the Claude API.

```text
Unity Client
  → POST /chat { npc_id, player_identity, message }
AI Gateway (FastAPI)
  → Query SpacetimeDB HTTP API (NPC data, memory, quest state)
  → Call Claude API with tool use
  ← { dialogue_text, actions[] }
Unity Client
  → Renders dialogue text
  → Calls SpacetimeDB reducers for each action
```

---

## Section 1: SpacetimeDB Tables

### NpcDefinition

`NpcDefinition` is both template and instance — NPCs don't move, so one table suffices (analogous to `EnemyDefinition` but with embedded position).

```text
id: u32
name: String           // "Old Marta the innkeeper"
prefab_name: String
zone_id: u32
position_x: f32
position_y: f32
elevation: f32         // vertical placement (matches EntityInstance pattern)
backstory: String      // Full backstory markdown, ~500-2000 chars
fallback_lines: String // JSON array of canned fallback responses
```

### NpcMemory

Per-NPC, per-player running relationship memory. Written by the gateway after each exchange.

```text
id: u32
npc_id: u32
player_identity: Identity
memory_text: String    // Max 600 chars; gateway distills when it grows
```

**Unique index:** `(npc_id, player_identity)`

### NpcQuestLink

Which quests an NPC can initiate or advance.

```text
npc_id: u32
quest_id: u32
role: String           // "giver" | "advancer" | "completer"
```

### ConversationTurn

Recent exchange history for context continuity. Truncated to last 10 turns at query time.

```text
id: u64
npc_id: u32
player_identity: Identity
role: String           // "player" | "npc"
content: String
timestamp_us: u64
```

**Index on:** `(npc_id, player_identity)` for efficient recent-turns query

### New reducers required

- `create_or_update_npc(name, prefab_name, zone_id, x, y, elevation, backstory, fallback_lines)`
- `update_npc_memory(npc_id, player_identity, memory_text)`
- `log_conversation_turn(npc_id, player_identity, role, content)`
- `link_quest_to_npc(npc_id, quest_id, role)`
- Quest/item reducers from Phase 5 Group 12: `start_quest`, `complete_quest`, `give_item`

---

## Section 2: AI Gateway Service

**Language:** Python 3.11+ with FastAPI

**Location:** `gateway/` directory at umbrella repo root

**Stateless:** All persistent state lives in SpacetimeDB

### Endpoint

```text
POST /chat
{
  "npc_id": 7,
  "player_identity": "abc123...",
  "message": "Do you know who burned the mill?"
}

→ 200 OK
{
  "dialogue": "Aye, I saw smoke that night...",
  "actions": [{ "type": "start_quest", "quest_id": 12 }],
  "is_fallback": false
}
```

### Request pipeline

1. Query SpacetimeDB HTTP API: `NpcDefinition`, `NpcMemory`, `NpcQuestLink`, last 10 `ConversationTurn` rows, player active quest states
2. Build system prompt (see below)
3. Call Claude API (`claude-haiku-4-5` for cost efficiency, `claude-sonnet-4-6` for quality — configurable)
4. Parse response: extract dialogue text + tool calls
5. Write `ConversationTurn` rows + updated `NpcMemory` via SpacetimeDB reducer calls
6. Return `{ dialogue, actions }`

### Tool definitions (LLM function calling)

```python
tools = [
    {"name": "start_quest",    "input_schema": {"quest_id": "integer"}},
    {"name": "complete_quest", "input_schema": {"quest_id": "integer"}},
    {"name": "give_item",      "input_schema": {"item_id": "integer"}},
    {"name": "open_shop",      "input_schema": {}},
    {"name": "update_memory",  "input_schema": {"memory_text": "string"}},
]
```

### System prompt structure

```text
You are {npc_name}.

== Backstory ==
{npc.backstory}

== Your memory of this player ==
{memory_text | "You haven't met this player before."}

== Quests you can offer or advance ==
{NpcQuestLink entries with quest names and brief descriptions}

== Player's active/completed quests ==
{player quest state snapshot}

== Rules ==
- Stay in character at all times.
- Only trigger actions when it makes narrative sense.
- Keep responses under 3 sentences unless a long answer is warranted.
- Do not acknowledge that you are an AI.
- Use update_memory only when something significant happens worth remembering.
```

### Memory distillation

- Hard cap: `memory_text` max 600 chars
- When new memory would exceed cap: gateway makes a second cheap `claude-haiku` call to summarize existing memory to ≤300 chars, then appends new content
- LLM self-authors what to remember; gateway enforces the size budget

### Rate limiting (cost control)

- Max 20 AI exchanges per player×NPC per hour (configurable via env var)
- Beyond limit: fallback path only

---

## Section 3: Failure Handling

Graceful degradation operates at three levels:

1. **Scripted interactions always work.** Traditional UI buttons ([Accept Quest], [Open Shop]) call SpacetimeDB reducers directly, bypassing the gateway entirely. The AI layer is additive — its failure never breaks core quest/shop interactions.

2. **LLM failure → fallback lines.** Gateway catches all exceptions (timeout, API error, rate limit). Returns a random `fallback_line` from `NpcDefinition` with `is_fallback: true`. Player sees an in-character deflection.

3. **Gateway unreachable.** `AiNpcClient` in Unity handles HTTP errors by displaying a fallback line from the locally-cached `NpcDefinition` subscription data.

---

## Section 4: Unity Client Integration

### New C# components

**`NpcInteractionController.cs`** (on NPC GameObject)

- Detects player proximity + interact key
- Opens `DialoguePanelUI` with bound `npcId`
- Routes player messages to `AiNpcClient`

**`AiNpcClient.cs`** (singleton service)

- `UnityWebRequest` POST to configured gateway URL
- Parses JSON response
- Fires `OnDialogueReceived(string text)` event
- Dispatches actions:

```csharp
switch (action.type) {
    case "start_quest":    Reducer.StartQuest(action.quest_id);    break;
    case "complete_quest": Reducer.CompleteQuest(action.quest_id); break;
    case "give_item":      Reducer.GiveItem(action.item_id);       break;
    case "open_shop":      ShopPanel.Open(npcId);                  break;
    // update_memory is handled server-side; client never sees it
}
```

**`DialoguePanelUI.cs`** (UXML + USS)

- Scroll view: NPC lines (left-aligned) + player lines (right-aligned)
- Text input field + Send button
- Typing indicator (animated dots) while awaiting response
- Follows existing editor USS/UXML patterns; Sort Order = 200 (above all existing panels)

### Gateway URL config

`AiNpcClient` reads the gateway URL from a `ScriptableObject` config asset (not hardcoded). Dev default: `http://localhost:8080`. Swappable per build target without code changes.

---

## Section 5: Editor Authoring Tools

A new **NPC Panel** in the standalone editor (runtime UI — no EditorWindow APIs).

```text
┌─────────────────────────────────────┐
│ NPC: [name field              ]     │
│ Prefab: [dropdown             ▼]    │
├─────────────────────────────────────┤
│ Backstory                           │
│ ┌─────────────────────────────────┐ │
│ │ Multi-line text field           │ │
│ │ (~500-2000 chars)               │ │
│ └─────────────────────────────────┘ │
├─────────────────────────────────────┤
│ Fallback Lines                      │
│ [Text field                      ]  │
│ [+ Add line]                        │
├─────────────────────────────────────┤
│ Linked Quests                       │
│ ● Missing Shipment (giver)  [×]     │
│ [Link quest ▼]                      │
├─────────────────────────────────────┤
│ [ Test Chat ]      [ Save NPC ]     │
└─────────────────────────────────────┘
```

- **Save NPC** → calls `create_or_update_npc` + `link_quest_to_npc` reducers
- **Test Chat** → opens inline `DialoguePanelUI` using the designer's identity as `player_identity`
- NPC position is stored in `NpcDefinition` directly (no `EntityInstance` row). Client subscribes to `NpcDefinition` and spawns NPC GameObjects from it, same way it subscribes to `Enemy` rows.

---

## Section 6: Deployment

### Dev loop addition

```bash
# Terminal 1 — SpacetimeDB (unchanged)
spacetime start

# Terminal 2 — AI gateway (new)
cd gateway && uvicorn main:app --port 8080 --reload

# Terminal 3 — server changes (unchanged)
cd server && spacetime build && spacetime publish --server local zoneforge-server
```

### Production

- Gateway is stateless → deploy as a systemd unit or Docker container alongside SpacetimeDB
- `ANTHROPIC_API_KEY` lives in the gateway environment only — never touches Unity builds
- SpacetimeDB HTTP API: `http://localhost:3000` (same host) or configured cloud URL

---

## Section 7: Recommended Phasing

This feature depends on Phase 5 Group 12 (Quests & NPCs) being complete first.

**Phase A — Foundation**

- `NpcDefinition`, `NpcMemory`, `NpcQuestLink`, `ConversationTurn` tables + reducers
- Basic `NpcInteractionController` + `DialoguePanelUI` (scripted-only, no AI yet)
- NPC Panel in editor (create/save NPCs, no test chat)

**Phase B — AI Integration**

- Gateway service with Claude tool use
- `AiNpcClient` HTTP integration
- Memory distillation
- Fallback handling
- Rate limiting

**Phase C — Polish**

- Test Chat in editor NPC Panel
- Streaming dialogue (typewriter effect via SSE)
- Per-build-target gateway URL config

---

## Verification

- Place an NPC via editor → fill backstory → save → confirm `NpcDefinition` row appears via `spacetime sql`
- Open dialogue in client → type message → confirm gateway logs show LLM call → confirm response renders
- Trigger a quest via conversation → confirm `start_quest` reducer fires → confirm quest appears in player state
- Confirm `NpcMemory` row written after exchange
- Kill gateway → confirm fallback line appears instead of error
- Confirm [Accept Quest] button works with gateway completely down
