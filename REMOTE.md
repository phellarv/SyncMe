# Remote Machine Synchronization Reference (Laptop & Workstation)

This document describes the multi-machine synchronization architecture powered by **SyncMe**. It defines the configured workspaces, directory and file mappings, `.smignore` exclusion patterns, workstation `config.toml` specifications, and the step-by-step onboarding procedure for remote workstations.

---

## 1. Overview & Purpose

SyncMe provides real-time, bidirectional file synchronization between primary development environments (such as **Laptop** and **Workstation**) via a centralized **SyncMe-Server** instance running at `https://sync.example.com`.

### Core Goals
- **Unified Shell & Terminal Environment:** Keep shell configurations (`fish`, `bashrc`), prompt settings (`starship.toml`), git preferences (`.gitconfig`), editor configs (`micro`), and terminal emulator configurations (`kitty`) identical across physical machines without manual git rebasing.
- **Autonomous Agent Parity:** Synchronize AI agent configurations and persistent state (`Claude`, `Gemini`) while isolating ephemeral execution data, database locks, and machine-specific tokens.
- **Symlink Preservation vs. Dereferencing:** Maintain repository-backed symlinks to local dotfile repositories (`replace_symlinks = false`) where symlinks must not be flattened, while allowing agent directories (`replace_symlinks = true`) to synchronize symlink contents into regular files when desired.
- **Conflict Avoidance:** Prevent write collisions, SQLite WAL corruption, and daemon deadlocks through fine-grained `.smignore` patterns and deterministic initial-sync conflict policies.

---

## 2. Synchronized Workspaces and Paths

SyncMe partitions files into logical **workspaces** on the server. Workspaces can represent entire recursive directory hierarchies (`[[directories]]`) or discrete individual configuration files (`[[files]]`).

### Configured Directory Workspaces

| Workspace Name | Local Path | `replace_symlinks` | Conflict Strategy | Description |
| :--- | :--- | :---: | :---: | :--- |
| `Claude` | `~/.claude` | `true` | `ask` | Claude agent runtime, custom prompt configurations, and settings. |
| `Gemini` | `~/.gemini` | `true` | `ask` | Gemini CLI configuration, skills, and agent operational files. |
| `fish` | `~/.config/fish` | `false` | `ask` | Fish shell functions, completions, and configuration. **Must preserve dotfiles symlinks.** |
| `bashrc` | `~/.config/bashrc` | `false` | `ask` | Modular bash helper scripts and aliases. |
| `kitty` | `~/.config/kitty` | `false` | `ask` | Kitty terminal configuration (`kitty.conf`, `colors-matugen.conf`). |

> [!IMPORTANT]
> **Preserving Symlinks (`replace_symlinks = false`):**
> For `fish`, `bashrc`, and `kitty`, `replace_symlinks` is set to `false`. These directories typically contain symbolic links pointing to a local dotfiles git repository (`~/dotfiles` or similar). Disabling symlink replacement prevents SyncMe from dereferencing and overwriting symlinks with static regular files.

### Configured Individual Files Workspace (`shell-files`)

Standalone configuration files across `~` and `~/.config` are grouped into the logical workspace `shell-files`. Metadata and state for this workspace are maintained centrally in `~/.local/share/SyncMe/state/shell-files.db`.

| Workspace Name | Local File Path | Remote Path | Conflict Strategy | Description |
| :--- | :--- | :--- | :---: | :--- |
| `shell-files` | `~/.bashrc` | `.bashrc` | `ask` | Primary interactive bash configuration. |
| `shell-files` | `~/.bash_profile` | `.bash_profile` | `ask` | Bash login shell configuration. |
| `shell-files` | `~/.config/starship.toml` | `starship.toml` | `ask` | Cross-shell Starship prompt configuration. |
| `shell-files` | `~/.gitconfig` | `.gitconfig` | `ask` | Git user aliases, formatting, and default preferences. |
| `shell-files` | `~/.config/micro/settings.json` | `settings.json` | `ask` | Micro terminal text editor user preferences. |

---

## 3. Ignore Rules (`.smignore`)

Files matching `.smignore` patterns are filtered locally by `pathspec` and will not be tracked, hashed, uploaded, or deleted across machines.

### 3.1 `~/.gemini/.smignore`

Save this file as `~/.gemini/.smignore`:

```gitignore
# ==========================================
# SyncMe Ignore Rules for ~/.gemini
# ==========================================

# 1. SQLite Databases, WAL and Lock Files
# Active SQLite databases must never be synced; leads to WAL locks and write conflicts
*.db
*.db-wal
*.db-shm
conversation_summaries.db*
conversations/

# 2. Ephemeral Logs & Crash Dumps
*.log
log/
crashes/

# 3. Heavy Session State & Agent Transcripts
# brain/ contains full session execution logs, transcripts, and artifacts (~100MB+)
brain/
annotations/
html_artifacts/

# 4. Active Sessions, IPC, Locks & Presence
# Prevents false active session detection or socket/PID contention between machines
presence/

# 5. Machine-Specific State & Identifiers
installation_id
jetski_state.pbtxt
state.json

# 6. Caches & Temporary Artifacts
cache/
implicit/
mcp/

# 7. Binaries & Bundled Built-ins
# Downloaded binary runtimes and version-specific tools
bin/
builtin/

# 8. Interactive CLI History
# Simultaneous CLI sessions across machines will collide and generate conflict files
history.jsonl

# 9. Symlinks Managed via Git (dotfiles)
# GEMINI.md, agents/, and skills/ are symlinks to a local dotfiles repo (tracked in Git).
# Ignoring them prevents SyncMe from dereferencing or replacing symlinks with regular files.
# config/GEMINI.md
# config/agents/
# config/skills/
```

### 3.2 Recommended `~/.claude/.smignore`

Save this file as `~/.claude/.smignore`:

```gitignore
# ==========================================
# SyncMe Ignore Rules for ~/.claude
# ==========================================

# 1. Python Virtual Environments & Packages
# Prevents syncing arch/OS-specific python binaries and high file counts
*venv*/
venv/
.venv/
site-packages/
__pycache__/
*.pyc

# 2. SQLite Databases, WAL and Locks
*.db
*.db-wal
*.db-shm
*.sqlite
*.sqlite3

# 3. Ephemeral Logs & Debug Traces
*.log
logs/
crash/
debug/

# 4. Session Transcripts, Cache & Temporary State
cache/
sessions/
transcripts/
*.tmp
*.sock
*.pid
```

---

## 4. Workstation Configuration Reference

On the remote workstation, create `~/.config/SyncMe/config.toml` (or deploy via your configuration manager). Set `client_id = "Workstation"` to ensure mutation echo-suppression over SSE.

```toml
[server]
url = "https://sync.example.com"
api_key = "<YOUR_SERVER_API_KEY>"
client_id = "Workstation"
debounce_seconds = 1.0
poll_interval_seconds = 300
request_timeout = 600

# ==============================================================================
# Directory Workspaces (each directory maps to an isolated server workspace)
# ==============================================================================

[[directories]]
name = "Claude"
local_path = "~/.claude"
conflict_strategy = "ask"
replace_symlinks = true

[[directories]]
name = "Gemini"
local_path = "~/.gemini"
conflict_strategy = "ask"
replace_symlinks = true

# ==============================================================================
# Shell Workspaces
# ==============================================================================

# Fish configuration (preserves symlinks to dotfiles repository)
[[directories]]
name = "fish"
local_path = "~/.config/fish"
conflict_strategy = "ask"
replace_symlinks = false

# Modular Bash scripts
[[directories]]
name = "bashrc"
local_path = "~/.config/bashrc"
conflict_strategy = "ask"
replace_symlinks = false

# Standalone configuration files
[[files]]
workspace = "shell-files"
local_path = "~/.bashrc"
conflict_strategy = "ask"

[[files]]
workspace = "shell-files"
local_path = "~/.bash_profile"
conflict_strategy = "ask"

[[files]]
workspace = "shell-files"
local_path = "~/.config/starship.toml"
conflict_strategy = "ask"

[[files]]
workspace = "shell-files"
local_path = "~/.gitconfig"
conflict_strategy = "ask"

[[files]]
workspace = "shell-files"
local_path = "~/.config/micro/settings.json"
conflict_strategy = "ask"

# Kitty terminal configuration
[[directories]]
name = "kitty"
local_path = "~/.config/kitty"
conflict_strategy = "ask"
replace_symlinks = false
```

---

## 5. Workstation Onboarding Guide (Fish Shell)

Follow this step-by-step procedure to bootstrap a new workstation.

### Step 1: Ensure Prerequisites & Client Installation

Ensure Python 3.12+ and `syncme-client` are installed and available in `$PATH`:

```fish
# Clone the repository workspace if not already present
mkdir -p ~/projects
cd ~/projects
git clone https://github.com/<your-username>/SyncMe.git
cd SyncMe/SyncMe-Client

# Install syncme-client tool via uv (or pip)
uv tool install --editable .
# Verify executable
which syncme-client
```

### Step 2: Create Directories and Configuration File

Create the required configuration directories and place the workstation config:

```fish
mkdir -p ~/.config/SyncMe
mkdir -p ~/.config/fish ~/.config/bashrc ~/.config/kitty ~/.config/micro
mkdir -p ~/.gemini ~/.claude

# Edit or populate ~/.config/SyncMe/config.toml (reference Section 4)
micro ~/.config/SyncMe/config.toml
```

### Step 3: Populate `.smignore` Files

Place the required ignore rules before running the initial sync to avoid synchronizing unwanted files or machine state:

```fish
# Copy or create ~/.gemini/.smignore
micro ~/.gemini/.smignore

# Copy or create ~/.claude/.smignore
micro ~/.claude/.smignore
```

### Step 4: Perform Initial Synchronization

> [!TIP]
> **Conflict Strategy on New Machines:**
> In `syncme-client`, the CLI option for conflict resolution is `--conflict-strategy [ask|server|local|both]`.
> When onboarding a new machine, use `--conflict-strategy server` so the server's established versions win any discrepancies, archiving existing skeleton local files and downloading the remote state.
> The `-d` / `--directory` parameter targets any workspace by name, including the `shell-files` file group.

Execute the initial reconciliation for each workspace:

```fish
# Initialize agent workspaces
syncme-client init -d Claude --conflict-strategy server
syncme-client init -d Gemini --conflict-strategy server

# Initialize shell directories
syncme-client init -d fish --conflict-strategy server
syncme-client init -d bashrc --conflict-strategy server
syncme-client init -d kitty --conflict-strategy server

# Initialize standalone shell files workspace
syncme-client init -d shell-files --conflict-strategy server

# (Alternative) Initialize all configured workspaces at once:
# syncme-client init --conflict-strategy server
```

Check synchronization status across all workspaces:

```fish
syncme-client status
```

### Step 5: Configure and Enable Systemd User Service

To keep `syncme-client run` active in the background and survive reboots:

```fish
# Create user systemd directory
mkdir -p ~/.config/systemd/user

# Link the unit file from the repository
ln -sf ~/projects/SyncMe/SyncMe-Client/systemd/syncme-client.service ~/.config/systemd/user/syncme-client.service

# Reload systemd user daemon
systemctl --user daemon-reload

# Enable and start the service immediately
systemctl --user enable --now syncme-client.service

# Verify service status
systemctl --user status syncme-client.service

# Enable lingering so daemon runs when logged out
loginctl enable-linger $USER
```

### Step 6: Monitor Daemon Logs

```fish
# Stream live synchronization events and status
journalctl --user -u syncme-client.service -f
```
