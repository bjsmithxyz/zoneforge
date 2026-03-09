# Sprint 1 Task Checklist - Week 1-2

## Sprint Goal

Establish core development environment and basic Unity-SpacetimeDB integration

---

## Week 1: Environment Setup & Core Tables

### Day 1-2: Development Environment ✅

- [x] Install Unity Hub 6.3
- [x] Install Unity 2022.3 LTS with URP
- [ ] Install Rust toolchain (rustc, cargo)
- [ ] Install WebAssembly target (wasm32-unknown-unknown)
- [ ] Install SpacetimeDB CLI
- [ ] Create project directory structure
- [ ] Initialize Git repositories (client + server)
- [ ] Configure VS Code with extensions

### Day 3-4: SpacetimeDB Foundation

- [x] **Start SpacetimeDB local server**
    
    ```bash
    spacetime start
    ```
    
- [ ] **Define Core Tables** (server/src/lib.rs)
    
    - [ ] Player table (id, identity, name, position, health)
    - [ ] Zone table (id, name, grid_width, grid_height)
    - [ ] EntityInstance table (id, zone_id, prefab_name, position, type)
    
    ```rust
    // Note: do NOT add #[derive(SpacetimeType)] here — that is only for embedded types,
    // not tables. The #[table(...)] macro handles the necessary derivations.
    #[table(name = entity_instance, public)]
    pub struct EntityInstance {
        #[primary_key]
        #[auto_inc]
        pub id: u64,
        pub zone_id: u64,
        pub prefab_name: String,
        pub position_x: f32,
        pub position_y: f32,
        pub rotation: f32,
        pub entity_type: EntityType,
    }
    
    #[derive(SpacetimeType)]
    pub enum EntityType {
        StaticProp,
        Interactive,
        NPC,
        Enemy,
        Player,
        TriggerVolume,
    }
    ```
    
- [ ] **Define Core Reducers**
    
    - [ ] create_player
    - [ ] move_player
    - [ ] create_zone
    - [ ] spawn_entity (for placing entities in zones)
    
    ```rust
    #[reducer]
    pub fn spawn_entity(
        ctx: &ReducerContext,
        zone_id: u64,
        prefab_name: String,
        pos_x: f32,
        pos_y: f32,
        entity_type: EntityType
    ) {
        ctx.db.entity_instance().insert(EntityInstance {
            id: 0,
            zone_id,
            prefab_name,
            position_x: pos_x,
            position_y: pos_y,
            rotation: 0.0,
            entity_type,
        });
    }
    ```
    
- [ ] **Build and publish module**

    ```bash
    cd ~/Projects/ZoneForge/server
    spacetime build
    spacetime publish --server local zoneforge-server
    ```

    > **Note:** Use `spacetime build` instead of `cargo build` directly — it handles the WASM target and any SpacetimeDB-specific post-processing automatically. Add `--delete-data` to the publish command only when resetting after a breaking schema change.
    
- [ ] **Test with CLI**

    ```bash
    # Create a test zone
    spacetime call --server local zoneforge-server create_zone "Test Village" 64 64

    # Query zones
    spacetime sql --server local zoneforge-server "SELECT * FROM zone"

    # Create a test player
    spacetime call --server local zoneforge-server create_player "TestPlayer1"

    # Query players
    spacetime sql --server local zoneforge-server "SELECT * FROM player"
    ```
    

### Day 5: Unity Project Setup

- [ ] **Create Unity Project in Unity Hub**
    
    - Template: 3D (URP)
    - Name: ZoneForge
    - Location: ~/Projects/ZoneForge/client
    - Version: 2022.3.x LTS
- [ ] **Configure Unity Project Settings**
    
    - Edit → Project Settings → Graphics
        - Verify URP Renderer is set
    - Edit → Project Settings → Quality
        - Set default to "High"
    - File → Build Settings
        - Platform: Linux (or your target platform)
        - Switch Platform
- [ ] **Create Basic ScriptableObject Architecture**
    
    Create `Assets/Scripts/Data/WorldData.cs`:
    
    ```csharp
    using UnityEngine;
    
    [CreateAssetMenu(menuName = "ZoneForge/World Data")]
    public class WorldData : ScriptableObject
    {
        public string worldName;
        public ZoneVisualData[] zones;
    }
    ```
    
    Create `Assets/Scripts/Data/ZoneVisualData.cs`:
    
    ```csharp
    using UnityEngine;
    using UnityEngine.Tilemaps;
    
    [CreateAssetMenu(menuName = "ZoneForge/Zone Visual Data")]
    public class ZoneVisualData : ScriptableObject
    {
        public string zoneName;
        public int gridWidth = 64;
        public int gridHeight = 64;
        
        [Header("Visual Data")]
        public TileBase[] groundTiles;
        public TileBase[] decorationTiles;
        public Material skyboxMaterial;
    }
    ```
    
- [ ] **Import Test Assets**
    
    - 1 character model (can use Unity primitives for now)
    - 5 prop models (cubes, cylinders for testing)
    - Basic materials/textures
    
    Place in: `Assets/Art/Models/`
    

---

## Week 2: SpacetimeDB SDK Integration & Basic Map Editor

### Day 6-7: SpacetimeDB Unity SDK

- [ ] **Install SpacetimeDB Unity SDK**
    
    Option 1 - Unity Package Manager:
    
    ```
    Window → Package Manager → + → Add package from git URL
    → https://github.com/clockworklabs/com.clockworklabs.spacetimedbsdk.git
    ```
    
    Option 2 - Manual (if git URL doesn't work):
    
    - Download SDK from: https://github.com/clockworklabs/SpacetimeDB-Unity-SDK
    - Place in `Assets/Plugins/SpacetimeDB/`
- [ ] **Create SpacetimeDB Connection Manager**

    Create `Assets/Scripts/Network/SpacetimeDBManager.cs`:

    ```csharp
    using System;
    using UnityEngine;
    using SpacetimeDB;

    public class SpacetimeDBManager : MonoBehaviour
    {
        public static SpacetimeDBManager Instance;

        [SerializeField] private string serverUrl = "http://localhost:3000";
        [SerializeField] private string moduleName = "zoneforge-server";

        private DbConnection _conn;

        void Awake()
        {
            if (Instance == null)
            {
                Instance = this;
                DontDestroyOnLoad(gameObject);
            }
            else
            {
                Destroy(gameObject);
            }
        }

        void Start()
        {
            ConnectToServer();
        }

        void Update()
        {
            _conn?.FrameTick();
        }

        void ConnectToServer()
        {
            Debug.Log($"Connecting to {serverUrl}...");

            _conn = DbConnection.Builder()
                .WithUri(serverUrl)
                .WithModuleName(moduleName)
                .OnConnect(OnConnect)
                .OnConnectError(OnConnectError)
                .OnDisconnect(OnDisconnect)
                .Build();
        }

        void OnConnect(DbConnection conn, Identity identity, string token)
        {
            Debug.Log($"Connected! Identity: {identity}");

            conn.SubscriptionBuilder()
                .OnApplied(OnSubscriptionApplied)
                .Subscribe(new[] { "SELECT * FROM player", "SELECT * FROM zone" });
        }

        void OnSubscriptionApplied(SubscriptionEventContext ctx)
        {
            Debug.Log("Subscribed to tables");

            // Table event callbacks are registered after subscription is confirmed.
            // Player, Zone, and their event types are generated by `spacetime generate`.
            ctx.Db.Player.OnInsert += OnPlayerInserted;
            ctx.Db.Player.OnUpdate += OnPlayerUpdated;
            ctx.Db.Zone.OnInsert += OnZoneInserted;
        }

        void OnConnectError(Exception e)
        {
            Debug.LogError($"Connection error: {e.Message}");
        }

        void OnDisconnect(DbConnection conn, Exception e)
        {
            if (e != null)
                Debug.LogWarning($"Disconnected with error: {e.Message}");
            else
                Debug.Log("Disconnected cleanly");
        }

        void OnPlayerInserted(EventContext ctx, Player player)
        {
            Debug.Log($"Player joined: {player.name} at ({player.position_x}, {player.position_y})");
        }

        void OnPlayerUpdated(EventContext ctx, Player oldPlayer, Player newPlayer)
        {
            Debug.Log($"Player {newPlayer.name} moved to ({newPlayer.position_x}, {newPlayer.position_y})");
        }

        void OnZoneInserted(EventContext ctx, Zone zone)
        {
            Debug.Log($"Zone created: {zone.name} ({zone.grid_width}x{zone.grid_height})");
        }

        public void CreatePlayer(string playerName)
        {
            if (_conn != null)
                Reducer.CreatePlayer(_conn, playerName);
        }

        public void MovePlayer(float x, float y)
        {
            if (_conn != null)
                Reducer.MovePlayer(_conn, x, y);
        }
    }
    ```
    
- [ ] **Test Connection**

    - Create empty scene
    - Add GameObject with SpacetimeDBManager component
    - Verify server URL is `http://localhost:3000` and module name is `zoneforge-server` (defaults are set in the script)
    - Press Play and check Console for "Connected!" message

### Day 8-9: Basic Map Editor UI

- [ ] **Create Zone Editor Window**
    
    Create `Assets/Editor/ZoneEditorWindow.cs`:
    
    ```csharp
    using UnityEngine;
    using UnityEditor;
    using UnityEditor.SceneManagement;

    public class ZoneEditorWindow : EditorWindow
    {
        private string zoneName = "New Zone";
        private int gridWidth = 64;
        private int gridHeight = 64;
        
        [MenuItem("Tools/ZoneForge/Zone Editor")]
        public static void ShowWindow()
        {
            GetWindow<ZoneEditorWindow>("Zone Editor");
        }
        
        void OnGUI()
        {
            GUILayout.Label("Zone Settings", EditorStyles.boldLabel);
            
            zoneName = EditorGUILayout.TextField("Zone Name", zoneName);
            gridWidth = EditorGUILayout.IntField("Grid Width", gridWidth);
            gridHeight = EditorGUILayout.IntField("Grid Height", gridHeight);
            
            if (GUILayout.Button("Create Zone"))
            {
                CreateZone();
            }
        }
        
        void CreateZone()
        {
            // Create new scene
            var scene = EditorSceneManager.NewScene(NewSceneSetup.EmptyScene, NewSceneMode.Single);
            
            // Create Grid GameObject
            GameObject gridObject = new GameObject(zoneName);
            Grid grid = gridObject.AddComponent<Grid>();
            grid.cellSize = new Vector3(1, 1, 0);
            
            // Create Tilemap for ground layer
            GameObject tilemapObject = new GameObject("Ground");
            tilemapObject.transform.parent = gridObject.transform;
            Tilemap tilemap = tilemapObject.AddComponent<Tilemap>();
            TilemapRenderer tilemapRenderer = tilemapObject.AddComponent<TilemapRenderer>();
            
            // Create ZoneData component
            ZoneController zoneController = gridObject.AddComponent<ZoneController>();
            zoneController.zoneName = zoneName;
            zoneController.gridWidth = gridWidth;
            zoneController.gridHeight = gridHeight;
            
            // Save scene
            string path = $"Assets/Scenes/Zones/{zoneName}.unity";
            System.IO.Directory.CreateDirectory("Assets/Scenes/Zones");
            EditorSceneManager.SaveScene(scene, path);
            
            Debug.Log($"Zone created: {zoneName} at {path}");
        }
    }
    ```
    
- [ ] **Create ZoneController Component**
    
    Create `Assets/Scripts/Zone/ZoneController.cs`:
    
    ```csharp
    using UnityEngine;
    
    public class ZoneController : MonoBehaviour
    {
        public string zoneName;
        public int gridWidth = 64;
        public int gridHeight = 64;
        
        public ulong zoneId; // SpacetimeDB zone ID
        
        void Start()
        {
            // Register zone with SpacetimeDB if not already registered
            RegisterZone();
        }
        
        void RegisterZone()
        {
            if (SpacetimeDBManager.Instance != null)
            {
                // Call create_zone reducer
                // Note: You'll need to add this method to SpacetimeDBManager
                Debug.Log($"Registering zone: {zoneName}");
            }
        }
    }
    ```
    

### Day 10: Grid System & Tile Palette

- [ ] **Set up Tilemap Grid**
    
    - Window → 2D → Tile Palette
    - Create new palette: "ZoneForge Tiles"
    - Create folder: `Assets/Art/Tiles/`
- [ ] **Create Basic Tiles**
    
    - Create test tiles (can use colored sprites for now):
        - Grass tile (green)
        - Dirt tile (brown)
        - Stone tile (gray)
    - Add to Tile Palette
- [ ] **Test Tile Painting**
    
    - Open Zone Editor: Tools → ZoneForge → Zone Editor
    - Create test zone (32x32 for quick testing)
    - Use Tile Palette to paint ground layer
    - Verify tiles appear in Scene view

---

## Sprint 1 Acceptance Criteria

### Must Have ✅

- [ ] Local SpacetimeDB server running successfully
- [ ] Server module compiles and publishes without errors
- [ ] Core tables defined: Player, Zone, EntityInstance
- [ ] Core reducers working: create_player, move_player, create_zone
- [ ] Unity project created with URP
- [ ] SpacetimeDB SDK integrated and connecting to local server
- [ ] Basic Zone Editor window functional
- [ ] Can create zones with Grid and Tilemap
- [ ] Can paint tiles on ground layer

### Nice to Have 🎯

- [ ] Multiple tile layers (ground, decoration)
- [ ] Zone persistence (save/load)
- [ ] Real-time multi-user zone editing
- [ ] Entity placement basics

---

## Testing Checklist

- [ ] **Server Tests**

    ```bash
    # Test zone creation
    spacetime call --server local zoneforge-server create_zone "Village" 64 64

    # Test player creation
    spacetime call --server local zoneforge-server create_player "Alice"
    spacetime call --server local zoneforge-server create_player "Bob"

    # Test player movement
    spacetime call --server local zoneforge-server move_player 10.5 20.3

    # Query all data
    spacetime sql --server local zoneforge-server "SELECT * FROM zone"
    spacetime sql --server local zoneforge-server "SELECT * FROM player"
    ```
    
- [ ] **Unity Tests**
    
    - Press Play, verify connection message in Console
    - Check that Player/Zone insert callbacks fire
    - Create zone via editor, verify it appears in scene
    - Paint tiles, verify they render correctly

---

## Common Issues & Solutions

### SpacetimeDB won't start

```bash
# Check if port 3000 is in use
sudo lsof -i :3000

# Kill process if needed
kill -9 <PID>

# Or start on different port
spacetime start --listen-addr 127.0.0.1:3001
```

### Unity SDK not found

- Verify Package Manager shows SDK installed
- Check for errors in Console
- Try reimporting SDK package
- Ensure SpacetimeDB server is running before Unity connects

### Rust build fails

```bash
# Clean build
cargo clean

# Rebuild via SpacetimeDB CLI (handles WASM target automatically)
spacetime build

# Check for missing dependencies
cargo check
```

### Tiles not appearing

- Verify Tilemap Renderer is present
- Check Tilemap Renderer sorting layer
- Ensure camera can see the tiles (orthographic camera)
- Check Grid cell size matches tile size

---

## End of Sprint 1

**Sprint Review Questions:**

1. Can you create a zone in the editor? YES / NO
2. Can tiles be painted on the grid? YES / NO
3. Does Unity connect to SpacetimeDB? YES / NO
4. Are core tables and reducers working? YES / NO
5. Can you query data from the server? YES / NO

**Sprint Retrospective:**

- What went well?
- What could be improved?
- Action items for Sprint 2?

---

**Ready for Sprint 2!** 🚀 Next up: Entity Palette, real-time entity spawning, and multi-client testing!