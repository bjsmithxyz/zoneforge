# ZoneForge Development Setup Guide - Kubuntu

## Environment Information

- **OS**: Kubuntu (Ubuntu-based)
- **Unity Hub**: 6.3
- **IDE**: VS Code
- **Project**: ZoneForge - Multiplayer Isometric RPG Editor

---

## Table of Contents

1. [Prerequisites Check](#prerequisites-check)
2. [Unity Setup](#unity-setup)
3. [Rust Toolchain Setup](#rust-toolchain-setup)
4. [SpacetimeDB Installation](#spacetimedb-installation)
5. [VS Code Configuration](#vs-code-configuration)
6. [Project Initialization](#project-initialization)
7. [Verification Tests](#verification-tests)
8. [Next Steps](#next-steps)

---

## 1. Prerequisites Check

### System Requirements

```bash
# Check your system
uname -a                    # Verify Linux kernel
lsb_release -a             # Ubuntu version (should be 20.04+)
free -h                    # RAM (minimum 8GB recommended)
df -h                      # Disk space (minimum 50GB free)
```

### Required Software

- **Git**: Version control
- **Git LFS**: Large file storage for Unity assets
- **curl/wget**: For downloading tools
- **Build essentials**: C/C++ compiler toolchain

### Install Prerequisites

```bash
# Update package lists
sudo apt update

# Install essential tools
sudo apt install -y git git-lfs curl wget build-essential pkg-config libssl-dev

# Configure Git LFS
git lfs install

# Verify installations
git --version          # Should be 2.x+
git lfs version        # Should show LFS version
gcc --version          # Should be 9.x+
```

---

## 2. Unity Setup

### Install Unity Editor via Unity Hub

1. **Open Unity Hub 6.3** (already installed)
2. **Install Unity 2022.3 LTS**:

    ```text
    Hub → Installs → Install Editor → Select 2022.3.x LTS
    ```

3. **Select Required Modules**:
    - ✅ Linux Build Support (Mono)
    - ✅ WebGL Build Support
    - ✅ Windows Build Support (Mono) - optional, for cross-platform
    - ✅ Documentation
    - ✅ Language packs (English)
4. **Wait for Installation** (~5-10 GB download)

### Create Unity Project

1. **In Unity Hub**:

    ```text
    Projects → New Project
    ```

2. **Project Settings**:
    - **Template**: 3D (URP) - Universal Render Pipeline
    - **Project Name**: `ZoneForge`
    - **Location**: `~/Projects/ZoneForge/client`
    - **Unity Version**: 2022.3.x LTS
3. **Click "Create Project"**

### Initial Unity Configuration

Once Unity opens:

1. **Verify URP Setup**:

    ```text
    Edit → Project Settings → Graphics
    → Scriptable Render Pipeline Settings (should show URP asset)
    ```

2. **Set Build Target**:

    ```text
    File → Build Settings → Platform → Linux
    → Switch Platform
    ```

3. **Configure Quality Settings**:

    ```text
    Edit → Project Settings → Quality
    → Set default quality level to "High"
    ```

### Initialize Client Git Repository

```bash
cd ~/Projects/ZoneForge/client
git init
git lfs track "*.fbx" "*.png" "*.wav" "*.prefab"
git add .
git commit -m "Initial Unity project setup"
```

---

## 3. Rust Toolchain Setup

### Install Rust via rustup

```bash
# Download and install rustup (Rust installer)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# During installation, select:
# 1) Proceed with installation (default)

# Add Rust to current shell path
source "$HOME/.cargo/env"

# Verify installation
rustc --version      # Should show 1.70+
cargo --version      # Should show 1.70+

# Add to .bashrc or .zshrc for persistence
echo 'source "$HOME/.cargo/env"' >> ~/.bashrc
```

### Install WebAssembly Target

SpacetimeDB modules compile to WebAssembly:

```bash
# Add wasm32-unknown-unknown target
rustup target add wasm32-unknown-unknown

# Verify target installation
rustup target list --installed | grep wasm32
```

### Install Development Tools

```bash
# Install rust-analyzer (LSP for VS Code)
rustup component add rust-analyzer

# Install rustfmt (code formatter)
rustup component add rustfmt

# Install clippy (linter)
rustup component add clippy

# Verify components
rustup component list --installed
```

---

## 4. SpacetimeDB Installation

### Install SpacetimeDB CLI

```bash
# Download and install SpacetimeDB CLI
curl -fsSL https://install.spacetimedb.com | bash

# The installer will:
# 1. Download the 'spacetime' CLI tool
# 2. Install it to ~/.spacetime/bin
# 3. Add it to your PATH

# Reload shell configuration
source ~/.bashrc

# Verify installation
spacetime version

# Expected output:
# spacetime-cli X.X.X
```

### Alternative Manual Installation

If automatic install fails:

```bash
# Create directory
mkdir -p ~/.spacetime/bin

# Download latest release (replace X.X.X with latest version)
wget https://github.com/clockworklabs/SpacetimeDB/releases/latest/download/spacetime-linux-amd64 \
  -O ~/.spacetime/bin/spacetime

# Make executable
chmod +x ~/.spacetime/bin/spacetime

# Add to PATH in ~/.bashrc
echo 'export PATH="$HOME/.spacetime/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Verify
spacetime version
```

### Start Local SpacetimeDB Server

```bash
# Start the database server (first time)
spacetime start

# This will:
# - Download the SpacetimeDB server binary
# - Start the server on http://localhost:3000
# - Initialize local database storage

# Expected output:
# Starting SpacetimeDB...
# Server running at http://localhost:3000
```

### Test SpacetimeDB Server

```bash
# In a new terminal, check server status
curl http://localhost:3000/database/ping

# Expected: {"status":"ok"}

# List databases (should be empty initially)
spacetime server list
```

---

## 5. VS Code Configuration

### Install VS Code Extensions

Open VS Code and install these extensions:

#### For Unity (C#):

> **Prerequisite**: The C# extension requires the .NET SDK. Install it first:
> ```bash
> sudo apt update && sudo apt install dotnet-sdk-8.0
> ```

```bash
code --install-extension ms-dotnettools.csharp
code --install-extension ms-dotnettools.csdevkit
code --install-extension Unity.unity-debug
```

Or manually in VS Code (`Ctrl+Shift+X`):

1. **C#** (ms-dotnettools.csharp) - Core C# language support, IntelliSense, and syntax highlighting
2. **C# Dev Kit** (ms-dotnettools.csdevkit) - Enhanced C# tooling including solution explorer and test runner
3. **Unity** (Unity.unity-debug) - Unity debugger, shader support, and Unity-specific code hints

#### For Rust:

```bash
code --install-extension rust-lang.rust-analyzer
code --install-extension vadimcn.vscode-lldb
code --install-extension fill-labs.dependi
```

Or manually (`Ctrl+Shift+X`):

1. **rust-analyzer** (rust-lang.rust-analyzer) - Language server providing IntelliSense, inline errors, and refactoring for Rust
2. **CodeLLDB** (vadimcn.vscode-lldb) - Native debugger for stepping through Rust code with breakpoints
3. **Dependi** (fill-labs.dependi) - Displays latest crate versions inline in `Cargo.toml` and highlights outdated dependencies

#### General Tools:

```bash
code --install-extension eamodio.gitlens
code --install-extension ms-vscode.live-server
code --install-extension PKief.material-icon-theme
```

Or manually (`Ctrl+Shift+X`):

1. **GitLens** (eamodio.gitlens) - Inline git blame, commit history, and diff views directly in the editor
2. **Live Server** (ms-vscode.live-server) - Local dev server with hot reload, useful for previewing any web-based tooling
3. **Material Icon Theme** (PKief.material-icon-theme) - File icons that distinguish Unity, Rust, TOML, and other project files at a glance

### Configure VS Code for Unity

1. **Open Unity Project in VS Code**:

    ```bash
    cd ~/Projects/ZoneForge/client
    code .
    ```

2. **Set VS Code as Unity's External Script Editor**:

    ```text
    Edit → Preferences → External Tools
    → External Script Editor → Browse → /usr/bin/code
    ```

    This ensures double-clicking a script in Unity opens it in VS Code instead of the default editor.

3. **Generate VS Code Project Files**:

    ```text
    Assets → Open C# Project
    ```

    This tells Unity to generate the `.csproj` and `.sln` files that the C# extension needs for IntelliSense and build support. Run this once after setting the external editor, and again after adding new Assembly Definition files.

4. **Review the auto-generated `.vscode/` files**:

    Unity generates `.vscode/settings.json`, `extensions.json`, and `launch.json` automatically when you run "Assets → Open C# Project". Do not overwrite these — the `settings.json` contains a `files.exclude` block that hides binary assets (textures, meshes, audio, etc.) from the VS Code file explorer so only code files are visible, and a `dotnet.defaultSolution` entry pointing at the correct `.sln` file.

    Add the following editor settings **inside** the existing `settings.json`, alongside the generated content:

    ```json
    "editor.formatOnSave": true,
    "[csharp]": {
      "editor.defaultFormatter": "ms-dotnettools.csharp"
    }
    ```

    What each setting does:
    - `editor.formatOnSave` — auto-formats C# files on save using the C# extension's built-in formatter
    - `editor.defaultFormatter` — scopes C# formatting to the C# extension only, so other file types use their own formatters

---

## 6. Project Initialization

### Create Project Directory Structure

```bash
# Create root project directory
mkdir -p ~/Projects/ZoneForge
cd ~/Projects/ZoneForge

# The structure will be:
# ZoneForge/
# ├── client/          (Unity project)
# ├── editor/          (Unity project)
# └── server/          (SpacetimeDB Rust module)
```

### Initialize Server (SpacetimeDB) Project

```bash
cd ~/Projects/ZoneForge

# Initialize a SpacetimeDB Rust module — this scaffolds the project correctly,
# including .cargo/config.toml with the wasm32-unknown-unknown target configured.
spacetime init --lang rust server
```

The command is interactive and will prompt twice:

| Prompt | What to enter |
| --- | --- |
| **Project path** | Press Enter to accept the default (`./server`) |
| **Database name** | `zoneforge-server` |

> **Database name format:** SpacetimeDB database names must be lowercase with dashes only — underscores are not allowed. Use `zoneforge-server`, not `zoneforge_server`.

```bash
# After the prompts complete:
cd server

# This creates:
# server/
# ├── .cargo/
# │   └── config.toml   (sets default build target to wasm32-unknown-unknown)
# ├── Cargo.toml
# └── src/
#     └── lib.rs
```

### Initialize Server Git Repository

Initialize git now, before editing any files, so all subsequent changes are tracked from the start.

```bash
cd ~/Projects/ZoneForge/server

git init

# Create .gitignore before the initial commit so it is included from the start.
cat > .gitignore << 'EOF'
/target/
**/*.rs.bk
Cargo.lock
EOF

git add .
git commit -m "Initial SpacetimeDB module setup"
```

### Configure VS Code for Rust

1. **Create the VS Code settings directory**:

    ```bash
    mkdir -p ~/Projects/ZoneForge/server/.vscode
    ```

2. **Open the server project in VS Code**:

    ```bash
    cd ~/Projects/ZoneForge/server
    code .
    ```

3. **Create `.vscode/settings.json`** with the following content:

    ```json
    {
      "rust-analyzer.linkedProjects": ["./Cargo.toml"],
      "rust-analyzer.check.command": "clippy",
      "rust-analyzer.cargo.features": "all",
      "editor.formatOnSave": true,
      "[rust]": {
        "editor.defaultFormatter": "rust-lang.rust-analyzer"
      }
    }
    ```

    What each setting does:
    - `rust-analyzer.linkedProjects` — explicitly points rust-analyzer to the `Cargo.toml` in the `server/` root; without this, rust-analyzer cannot discover the workspace and will show a red error icon
    - `rust-analyzer.check.command: "clippy"` — runs Clippy (the Rust linter) on every save instead of just `cargo check`, catching more issues
    - `rust-analyzer.cargo.features: "all"` — analyses code with all Cargo feature flags enabled so nothing is hidden
    - `editor.formatOnSave` — auto-formats `.rs` files using rustfmt on every save
    - `editor.defaultFormatter` — scopes the Rust formatter to Rust files only

4. **Verify rust-analyzer is active**:

    - The rust-analyzer status icon in the bottom status bar should show as loading, then turn green
    - Open `src/lib.rs` — hover over any type or function and IntelliSense tooltips should appear
    - Save the file — Clippy should run and any warnings will appear in the Problems panel (`Ctrl+Shift+M`)

### Configure Cargo.toml for SpacetimeDB

Edit `~/Projects/ZoneForge/server/Cargo.toml`:

```toml
[package]
name = "zoneforge_server"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
spacetimedb = "2.0"
log = "0.4"

[profile.release]
opt-level = 3
lto = true
```

### Create Basic SpacetimeDB Module

Edit `~/Projects/ZoneForge/server/src/lib.rs`:

```rust
use spacetimedb::{table, reducer, ReducerContext, Identity, Table};

// Define a simple Player table.
// Note: #[table(...)] is the table attribute — do NOT also add #[derive(SpacetimeType)].
// SpacetimeType is only for custom embedded types used as fields inside table rows.
#[table(accessor = player, public)]
pub struct Player {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    // #[unique] creates an index so reducers can look up players by identity.
    #[unique]
    pub identity: Identity,
    pub name: String,
    pub zone_id: u64,
    pub position_x: f32,
    pub position_y: f32,
    pub health: i32,
    pub max_health: i32,
}

// Define a Zone table
#[table(accessor = zone, public)]
pub struct Zone {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub name: String,
    pub grid_width: u32,
    pub grid_height: u32,
}

// Reducer to create a new player
#[reducer]
pub fn create_player(ctx: &ReducerContext, name: String) {
    let player = Player {
        id: 0, // auto_inc will assign
        identity: ctx.sender(),
        name,
        zone_id: 1, // Default starting zone
        position_x: 0.0,
        position_y: 0.0,
        health: 100,
        max_health: 100,
    };

    ctx.db.player().insert(player);
    log::info!("Player created: {}", ctx.sender());
}

// Reducer to move a player
#[reducer]
pub fn move_player(ctx: &ReducerContext, new_x: f32, new_y: f32) {
    let player_identity = ctx.sender();

    // .identity() works here because the field is marked #[unique] above.
    if let Some(player) = ctx.db.player().identity().find(player_identity) {
        ctx.db.player().id().update(Player {
            position_x: new_x,
            position_y: new_y,
            ..player
        });
        log::info!("Player moved to ({}, {})", new_x, new_y);
    }
}

// Reducer to create a zone
#[reducer]
pub fn create_zone(ctx: &ReducerContext, name: String, width: u32, height: u32) {
    let zone = Zone {
        id: 0, // auto_inc
        name: name.clone(),
        grid_width: width,
        grid_height: height,
    };

    ctx.db.zone().insert(zone);
    log::info!("Zone created: {}", name);
}
```

### Build the SpacetimeDB Module

```bash
cd ~/Projects/ZoneForge/server

# Build the module (compiles to WebAssembly)
# spacetime build handles the WASM target and SpacetimeDB-specific steps automatically.
spacetime build

# Expected output:
# Compiling zoneforge_server v0.1.0
# Finished `release` profile [optimized] target(s) in Xs
# Optimising module with wasm-opt...
# Could not find wasm-opt to optimise the module.
# ...
# Build finished successfully.
```

> **`wasm-opt` warning:** The "Could not find wasm-opt" message is non-critical. The build succeeds without it — `wasm-opt` only applies post-build size and performance optimizations to the WASM binary. It can be safely ignored during development. For production, install it from [github.com/WebAssembly/binaryen/releases](https://github.com/WebAssembly/binaryen/releases).

### Publish Module to Local SpacetimeDB Server

```bash
# Make sure SpacetimeDB server is running
# (In another terminal: spacetime start)

# First publish — no --delete-data needed as there is nothing to clear yet.
# --server local targets http://127.0.0.1:3000 (the local SpacetimeDB instance).
spacetime publish --server local zoneforge-server

# Expected output:
# Publishing module ... to database 'zoneforge-server'
# Uploading to local => http://127.0.0.1:3000
# Build finished successfully.
```

> **Republishing after schema changes:** Add `--delete-data` to clear existing data when the schema has changed incompatibly. This will prompt for confirmation before destroying data — that is expected behavior.
>
> ```bash
> spacetime publish --server local zoneforge-server --delete-data
> ```

---

## 7. Verification Tests

### Test 1: Rust Compilation

```bash
cd ~/Projects/ZoneForge/server

# Run cargo check
cargo check

# Should output: Finished dev [unoptimized + debuginfo]

# Run clippy (linter)
cargo clippy

# Should not show errors
```

### Test 2: SpacetimeDB Module

```bash
# Verify module is published
spacetime server list

# Should show:
# zoneforge-server - http://localhost:3000

# Call a reducer using CLI
spacetime call --server local zoneforge-server create_zone "Test Village" 64 64

# Expected output:
# WARNING: This command is UNSTABLE and subject to breaking changes.
# (no error = success)

# Query the zone table
spacetime sql --server local zoneforge-server "SELECT * FROM zone"

# Expected output:
# WARNING: This command is UNSTABLE and subject to breaking changes.
#
#  id | name           | grid_width | grid_height
# ----+----------------+------------+-------------
#  1  | "Test Village" | 64         | 64
```

### Test 3: Unity Project

1. **Open Unity Project**:

    ```bash
    # Unity Hub → Projects → Open → ~/Projects/ZoneForge/client
    ```

2. **Verify URP**:

    - Create a new scene
    - Add a Cube (GameObject → 3D Object → Cube)
    - Should render with URP lighting

3. **Test Script Compilation**:

    - Create a test script:

        ```text
        Assets → Create → C# Script
        ```

        Name it `TestScript` when prompted (press Enter to confirm).

    - Double-click `TestScript` in the Project window to open it in VS Code, then replace the contents with:

        ```csharp
        using UnityEngine;

        public class TestScript : MonoBehaviour
        {
            void Start()
            {
                Debug.Log("ZoneForge Unity project is working!");
            }
        }
        ```

        Save the file (`Ctrl+S`) and switch back to Unity — wait a moment for Unity to recompile (progress bar in the bottom-right corner).

    - **Attach the script to an object**:

        1. In the **Hierarchy** panel (left side), select the Cube created in step 2
        2. In the **Inspector** panel (right side), click **Add Component** at the bottom
        3. Type `TestScript` in the search box and select it from the list

        The script is now attached — you will see it listed as a component on the Cube in the Inspector.

    - **Press Play** using the Play button (▶) at the top-centre of the Unity editor

    - Open the **Console** panel (`Window → General → Console` if not already visible) — you should see:

        ```text
        ZoneForge Unity project is working!
        UnityEngine.Debug:Log (object)
        ```

    - Press the Play button again (or `Ctrl+P`) to exit Play mode
        

### Test 4: VS Code Integration

```bash
# Open server project in VS Code
cd ~/Projects/ZoneForge/server
code .

# rust-analyzer should activate (bottom right status bar)
# Open lib.rs and hover over types - should show documentation

# Open client project in VS Code
cd ~/Projects/ZoneForge/client
code .

# C# extension should activate
# Open any C# script - should have IntelliSense
```

---

## 8. Next Steps

### Sprint 1 Tasks (Week 1-2)

Now that your environment is set up, you're ready to start Sprint 1:

#### Completed ✅:

- [x] Unity Project Setup
- [x] Install Rust Toolchain
- [x] Install SpacetimeDB CLI
- [x] Create Git Repositories
- [x] Initialize SpacetimeDB Module
- [x] Define Core Tables - Zone
- [x] Define Core Tables - Player
- [x] Test Local SpacetimeDB Server

#### Next Tasks 📋:

0. **Cleanup Test Assets**

    The Cube and `TestScript` created during Section 7 testing are scaffolding only and should be removed before continuing:

    - In the **Hierarchy** panel: right-click `Cube` → **Delete**
    - In the **Project** panel: navigate to `Assets/` → right-click `TestScript.cs` → **Delete** → confirm

1. **Unity: Create ScriptableObject Architecture**

    ScriptableObjects are Unity data containers that live as assets in the project, independent of any scene. Create a dedicated subfolder to keep data-definition scripts separate from gameplay logic:

    ```text
    Assets/Scripts/Data/     ← create this folder first
    ```

    Then create the scripts inside it:

    ```text
    Right-click Assets/Scripts/Data/ → Create → C# Script → WorldData.cs
    Right-click Assets/Scripts/Data/ → Create → C# Script → ZoneVisualData.cs
    ```

2. **Install Material Maker**

    Material Maker is a node-based procedural texture authoring tool. Install it via Flatpak (recommended on Linux) or download directly:

    ```bash
    # Option A: Flatpak (recommended)
    flatpak install flathub org.materialmaker.MaterialMaker

    # Option B: Manual download
    # Visit https://www.materialmaker.org/ → download the archive → extract and run the binary directly (no installer required)
    ```

    Once installed, create the following folder structure inside your Unity project to keep generated assets organised:

    ```text
    Assets/Art/Materials/
    ├── Graphs/       ← Material Maker .mmg graph source files
    ├── Exports/      ← Exported texture maps (albedo, normal, roughness, etc.)
    └── Parameters/   ← Saved parameter presets
    ```

    - Open Material Maker and build a simple test graph (e.g. a basic stone texture) and export it to `Assets/Art/Materials/Exports/` to verify the full workflow end-to-end

3. **Install ArmorPaint**

    ```bash
    # Option A: Pre-built binary (~$19)
    # Download from https://armorpaint.org → run executable

    # Option B: Build from source (free)
    git clone --recursive https://github.com/armory3d/armorpaint
    cd armorpaint
    # Follow README build instructions (requires Kha framework)
    ```

    - Paint a test NPC model using baked maps to verify the workflow

4. **Unity: Install SpacetimeDB C# SDK**

    ```text
    Window → Package Manager → Add package from git URL
    → https://github.com/clockworklabs/com.clockworklabs.spacetimedbsdk.git
    ```

5. **Test Client-Server Connection**

    This step verifies that Unity can talk to the local SpacetimeDB server end-to-end: connecting, subscribing to a table, and calling a reducer.

    **Prerequisites:**
    - Step 4 (SpacetimeDB C# SDK) must be installed
    - SpacetimeDB server must be running (`spacetime start` in a terminal)
    - The `zoneforge-server` module must be published (`spacetime publish --server local zoneforge-server`)

    **A — Generate client bindings**

    SpacetimeDB generates typed C# classes from your server module (Player, Zone, Reducer, etc.).

    Open a terminal in your **Unity project root** — the folder that directly contains `Assets/`, `Packages/`, and `ProjectSettings/`. Do **not** run this from `server/`; the `--out-dir` path is relative to wherever you run the command.

    ```bash
    cd /path/to/ZoneForge/client   # adjust to your actual Unity project path
    spacetime generate --lang csharp --out-dir Assets/Scripts/autogen --bin-path ../server/spacetimedb/target/wasm32-unknown-unknown/release/zoneforge_server.wasm
    ```

    This reads from the compiled WASM binary produced by `spacetime build`. If you have not built recently, run `spacetime build` from `server/` first.

    Unity will auto-import the generated files from `Assets/Scripts/autogen/`. Wait for the import spinner to finish before continuing.

    **B — Create the connection script**

    Create `Assets/Scripts/ConnectionTest.cs` and replace its contents with the following:

    ```csharp
    using System;
    using UnityEngine;
    using SpacetimeDB;

    public class ConnectionTest : MonoBehaviour
    {
        private DbConnection _conn;

        void Start()
        {
            Debug.Log("Connecting to SpacetimeDB...");

            _conn = DbConnection.Builder()
                .WithUri("http://localhost:3000")
                .WithModuleName("zoneforge-server")
                .OnConnect(OnConnect)
                .OnConnectError(OnConnectError)
                .OnDisconnect(OnDisconnect)
                .Build();
        }

        void Update()
        {
            _conn?.FrameTick();
        }

        void OnConnect(DbConnection conn, Identity identity, string token)
        {
            Debug.Log($"Connected! Identity: {identity}");

            conn.SubscriptionBuilder()
                .OnApplied(OnSubscriptionApplied)
                .Subscribe("SELECT * FROM player");
        }

        void OnSubscriptionApplied(SubscriptionEventContext ctx)
        {
            Debug.Log("Subscription applied — calling create_player reducer");
            Reducer.CreatePlayer(ctx, "TestPlayer");
        }

        void OnConnectError(Exception e)
        {
            Debug.LogError($"Connection failed: {e.Message}");
        }

        void OnDisconnect(DbConnection conn, Exception e)
        {
            if (e != null)
                Debug.LogWarning($"Disconnected with error: {e.Message}");
            else
                Debug.Log("Disconnected cleanly");
        }
    }
    ```

    > **Note:** `Reducer.CreatePlayer` and the `Identity` type come from the generated autogen files. If they show as unresolved, check that the autogen import completed and try `Assets → Reimport All`.

    **C — Attach and run**

    1. In the **Hierarchy** panel: `GameObject → Create Empty` — name it `ConnectionTest`
    2. Select it → In the **Inspector**: **Add Component** → search `ConnectionTest` → click to add
    3. Ensure `spacetime start` is running in a terminal
    4. Press **Play** (▶ at the top centre of the Unity editor)
    5. Open `Window → General → Console`

    **Expected console output:**

    ```text
    Connecting to SpacetimeDB...
    Connected! Identity: <your identity hash>
    Subscription applied — calling create_player reducer
    ```

    **D — Verify in the database**

    In a second terminal, query the player table to confirm the row was inserted:

    ```bash
    spacetime sql --server local zoneforge-server "SELECT * FROM player"
    ```

    Expected output:

    ```text
    +----+------------------+------------+---------+------------+------------+--------+------------+
    | id | identity         | name       | zone_id | position_x | position_y | health | max_health |
    +----+------------------+------------+---------+------------+------------+--------+------------+
    | 1  | <identity hash>  | TestPlayer | 1       | 0          | 0          | 100    | 100        |
    +----+------------------+------------+---------+------------+------------+--------+------------+
    ```

    If you see the row, client-server connectivity is confirmed. Press **Stop** (■) in Unity to exit Play mode.

6. **Open the ZoneForge Editor**

    World building (zone creation, tile painting, entity placement) is done in the **standalone `zoneforge-editor` application** — not in the game client.

    Open the editor project in Unity Hub:

    ```text
    Unity Hub → Projects → Add → navigate to editor/
    ```

    Then open it and press **Play** to launch the editor in Play mode. It will connect to your local SpacetimeDB instance automatically.

    > See [Editor_Dev.md](Editor_Dev.md) for the full editor development workflow.

### Recommended Learning Resources

#### SpacetimeDB:

- Official Docs: https://spacetimedb.com/docs
- Getting Started: https://spacetimedb.com/docs/getting-started
- Unity SDK Guide: https://spacetimedb.com/docs/sdks/unity
- Discord Community: https://spacetimedb.com/community

#### Rust:

- The Rust Book: https://doc.rust-lang.org/book/
- Rust by Example: https://doc.rust-lang.org/rust-by-example/
- Cargo Book: https://doc.rust-lang.org/cargo/

#### Unity:

- Unity Manual: https://docs.unity3d.com/Manual/
- URP Guide: https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@latest

---

## Troubleshooting

### Issue: "spacetime: command not found"

```bash
# Check if installed
ls ~/.spacetime/bin/spacetime

# If exists, add to PATH manually
echo 'export PATH="$HOME/.spacetime/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Issue: Rust build fails with "linker not found"

```bash
# Install build essentials
sudo apt install build-essential pkg-config libssl-dev
```

### Issue: Unity Hub won't install Unity 2022.3

```bash
# Update Unity Hub
# Download latest from: https://unity.com/download

# Or use manual Unity Editor install
# Download from: https://unity.com/releases/editor/archive
```

### Issue: SpacetimeDB server won't start

```bash
# Check if port 3000 is in use
sudo lsof -i :3000

# If occupied, kill the process or use different port
spacetime start --listen-addr 127.0.0.1:3001
```

### Issue: VS Code rust-analyzer not working

```bash
# Reinstall rust-analyzer
rustup component remove rust-analyzer
rustup component add rust-analyzer

# Restart VS Code
# Press Ctrl+Shift+P → "Developer: Reload Window"
```

### Issue: ".NET SDK cannot be located" in C# extension

The C# extension requires the .NET SDK to be on the system PATH. Install it via apt:

```bash
sudo apt update && sudo apt install dotnet-sdk-8.0
dotnet --version    # Should output 8.x.x
```

Then reload VS Code (`Ctrl+Shift+P` → "Developer: Reload Window"). Unity 2022.3 targets .NET Standard 2.1 — use SDK 8.0, not 10.x, for maximum compatibility with the C# tooling.

---

## Development Workflow Summary

### Daily Development Loop:

1. **Start SpacetimeDB Server**:
    
    ```bash
    spacetime start
    ```
    
2. **Work on Server Code**:

    ```bash
    cd ~/Projects/ZoneForge/server
    code .
    # Make changes to src/lib.rs
    spacetime build
    spacetime publish --server local zoneforge-server
    ```

    > **Warning**: Add `--delete-data` only when you need to reset all data (e.g. after a breaking schema change). It wipes every table in the database and cannot be undone.
    
3. **Work on Client Code**:
    
    ```bash
    cd ~/Projects/ZoneForge/client
    code .
    # Open Unity Hub → Open ZoneForge project
    # Make changes in Unity and VS Code
    ```
    
4. **Test Changes**:
    
    - Play mode in Unity
    - Check SpacetimeDB logs
    - Use `spacetime sql` to query database
5. **Commit Progress**:
    
    ```bash
    git add .
    git commit -m "Description of changes"
    ```
    

---

## Congratulations! 🎉

Your ZoneForge development environment is now fully configured and ready for Sprint 1 development.

**You have:**

- ✅ Unity 2022.3 LTS with URP
- ✅ Rust toolchain with wasm32 target
- ✅ SpacetimeDB CLI and local server
- ✅ VS Code with C# and Rust extensions
- ✅ Basic SpacetimeDB module compiled and published
- ✅ Git repositories initialized

**Start building your multiplayer RPG editor!**