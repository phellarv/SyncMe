# SyncMe

SyncMe is a lightweight, real-time, bidirectional file synchronization system written in Python 3.12. Designed for developers and multi-workstation workflows, it provides directory and standalone dotfile synchronization, optimistic concurrency control (OCC), revision-based conflict resolution with interactive desktop alerts and visual diffing (Meld), Content-Addressable Storage (CAS) with monotonic per-workspace file versioning, automated safety archiving, and instant change propagation powered by Server-Sent Events (SSE).

---

## Architecture Overview

SyncMe is organized as a multi-repository workspace consisting of two primary components:

1. **SyncMe-Server**: Central synchronization and coordination server built with FastAPI, SQLite (WAL mode), Content-Addressable Storage (CAS), and a workspace-partitioned SSE broadcaster.
2. **SyncMe-Client**: Desktop daemon and CLI built with Click, Watchdog (filesystem watcher), and HTTPX. It supports concurrent multi-directory workspaces via `SyncEngine` and standalone individual file mappings (e.g. dotfiles) via `FileGroupEngine`, operating either as an interactive CLI or as an unattended `systemd` user service.

```
+---------------------------------------------------------------------------------------------------+
|                                           SyncMe System                                           |
|                                                                                                   |
|  +---------------------------------------------------------------------------------------------+  |
|  |                                        SyncMe-Client                                        |  |
|  |                                                                                             |  |
|  |  +-------------------------+     +--------------------------+     +----------------------+  |  |
|  |  |   Watchdog Observers    | --> |    SyncEngine (Dirs)     | <-> | Local SQLite (WAL)   |  |  |
|  |  |   (Debounced writes)    |     | FileGroupEngine (Files)  |     | & Safe Local Archive |  |  |
|  |  +-------------------------+     +--------------------------+     |                      |  |  |
|  |               |                               ^       |           | Dirs: .syncme/*.db   |  |  |
|  |               v                               |       |           | Files: state/*.db    |  |  |
|  |  +-------------------------+                  |       |           +----------------------+  |  |
|  |  |  Desktop Notifications  |                  |       |                                     |  |
|  |  |  (notify-send & Meld)   |                  |       |                                     |  |
|  |  +-------------------------+                  |       |                                     |  |
|  |                                               |       |                                     |  |
|  |  +-------------------------+                  |       |                                     |  |
|  |  | SSE Event Streams       | -----------------+       |                                     |  |
|  |  | (1 per active workspace)|                          |                                     |  |
|  |  +-------------------------+                          |                                     |  |
|  +-------------------------------------------------------|-------------------------------------+  |
|                                                          |                                        |
|                          HTTP REST & SSE Streams         |                                        |
|                          /api/v1/workspaces/{id}/...     |                                        |
|                          (Bearer Token Auth over HTTPS)  |                                        |
|                                                          v                                        |
|  +---------------------------------------------------------------------------------------------+  |
|  |                                        SyncMe-Server                                        |  |
|  |                                                                                             |  |
|  |  +-------------------------+     +--------------------------+     +----------------------+  |  |
|  |  |     FastAPI Router      | <-> |    Partitioned Store     | <-> | Server SQLite (WAL)  |  |  |
|  |  |   (REST Endpoints &     |     |  workspaces/{id}/        |     | Files & Revisions    |  |  |
|  |  |   ETag Concurrency)     |     |  objects/{id}/ (CAS/sha) |     | By (workspace, path) |  |  |
|  |  +-------------------------+     |  archive/{id}/           |     |  file_versions table |  |  |
|  |               |                  +--------------------------+     +----------------------+  |  |
|  |               v                                                                             |  |
|  |  +-------------------------+                                                                |  |
|  |  |     SSE Broadcaster     |                                                                |  |
|  |  | (Partitioned per-ws,    |                                                                |  |
|  |  |  echo-suppressed)       |                                                                |  |
|  |  +-------------------------+                                                                |  |
|  +---------------------------------------------------------------------------------------------+  |
+---------------------------------------------------------------------------------------------------+
```

### Core Architecture & Engines

- **Dual Reconciliation Engines:**
  - **`SyncEngine` (Directory Workspaces):** Recursively monitors entire directory hierarchies mapped via `[[directories]]`. Manages independent per-directory state in `<local_path>/.syncme/local_state.db` and local file safety archives in `<local_path>/.syncme/archive/`.
  - **`FileGroupEngine` (Standalone Dotfiles):** Watches discrete files scattered across `~` and `~/.config` mapped via `[[files]]` into a unified server workspace (such as `dotfiles` or `shell-files`). State is stored centrally in `~/.local/share/SyncMe/state/<workspace>.db`.
- **Content-Addressable Storage (CAS) & Monotonic Versioning:** Every file revision uploaded to the server is indexed by its SHA-256 digest into `<STORAGE_DIR>/objects/{workspace_id}/{sha256[:2]}/{sha256[2:]}`. Transactional metadata is recorded in the `file_versions` SQLite table, providing immutable point-in-time snapshots, SHA-256 deduplication, and zero-loss history inspection and rollbacks (`history` and `restore`).
- **Real-Time Notification & Polling Fallback:** Local filesystem changes are monitored using [Watchdog](https://github.com/gorakhargosh/watchdog) and batched over a debouncing window (`debounce_seconds`, default: `1.0s`). In addition, periodic background polling (`poll_interval_seconds`, default: `300s`) runs as a fail-safe fallback to ensure eventual consistency across network disconnects or suspend/resume cycles.
- **Server-Sent Events (SSE):** Each workspace maintains an event channel (`/api/v1/workspaces/{workspace_id}/events`). The server broadcasts `file_updated` and `file_deleted` events in real time. Originating clients are automatically echo-suppressed via `origin_client_id` (matched against the authenticated client identity resolved from the API key). Reconnections employ exponential backoff (from 1s up to 60s) followed by a catch-up reconciliation pass.
- **Optimistic Concurrency Control (OCC):** All modifications are guarded by revision counters and HTTP `ETag` / `If-Match` headers. The server atomically applies updates using temporary staging files and `os.replace` with `fsync`, rejecting out-of-sequence mutations with HTTP `409 Conflict`.
- **Interactive Conflict Diffing & Status Integration:** When concurrent mutations collide, the client preserves the local copy as `<stem>.conflict.<timestamp><suffix>` and downloads the server version. It triggers critical desktop notifications (`notify-send`) with one-click actions for Meld visual diffing, provides an inline terminal diffing wizard (`syncme-client conflicts resolve`), and exposes structured status outputs for Waybar (`syncme-client conflicts count --format waybar`).
- **Automated Safe Archival:** Deletions and overwrites on both client and server automatically preserve previous file states in timestamped archive directories (`.syncme/archive/` locally, `<STORAGE_DIR>/archive/{workspace_id}/` on the server) before modifying disk files.
- **Resilient Systemd Daemon:** Designed to run 24/7 as a systemd user service (`syncme-client.service`). The unit configures `RestartPreventExitStatus=78` to prevent restart thrashing if an API key is revoked (HTTP 401), and supports unattended background execution across logouts via user lingering.

---

## Configuration Guide (`~/.config/SyncMe/config.toml`)

The client configuration file is located at `~/.config/SyncMe/config.toml` by default, or loaded from the path specified by the `SYNCME_CONFIG_PATH` environment variable.

### Security Constraints & Best Practices

> [!IMPORTANT]
> **Strict File Permissions (`chmod 600`):**
> Because `config.toml` contains your authentication token, the file must be secured against other local users. On startup, `syncme-client` verifies permissions and prints a warning to `stderr` if group or world access bits are detected:
> ```bash
> chmod 600 ~/.config/SyncMe/config.toml
> ```

> [!IMPORTANT]
> **HTTPS Enforced:**
> SyncMe requires HTTPS for all remote connections. Plain `http://` URLs are strictly rejected by the client unless connecting to loopback interfaces (`localhost`, `127.0.0.1`, or `::1`). Server URLs must not contain embedded basic-auth userinfo (`user:pass@host`).

### Per-Client Token Generation

SyncMe uses individual, per-machine API keys instead of a shared global secret. The client will refuse to start if `api_key` is empty, missing, or left as the placeholder `"default_token"` (`ConfigError`). API keys must consist of printable ASCII characters (`[\x21-\x7e]+`) without whitespace or control characters.

Generate an API key for each workstation directly on the server:

```bash
# Via Docker Compose:
docker compose run --rm keys add <client_name>

# Or directly via Python module on the host:
python -m app.keys add <client_name>
```

Copy the returned key and paste it directly into your client `config.toml`.

### Realistic Configuration Sample

Here is a recommended configuration illustrating both recursive directory synchronization (`[[directories]]`) and standalone dotfile synchronization (`[[files]]`):

```toml
[server]
url = "https://sync.example.com"
api_key = "PASTE_GENERATED_KEY_HERE"  # Generated via: docker compose run --rm keys add laptop
client_id = "laptop-client"
debounce_seconds = 1.0
poll_interval_seconds = 300
request_timeout = 300.0

# ------------------------------------------------------------------------------
# Directory Workspaces (SyncEngine)
# ------------------------------------------------------------------------------
# Entire directory hierarchies mapped to distinct server workspaces.
# Each workspace maintains its own isolated database in <local_path>/.syncme/

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

[[directories]]
name = "fish"
local_path = "~/.config/fish"
conflict_strategy = "ask"
replace_symlinks = false   # Preserve symlinks to dotfiles repository

[[directories]]
name = "kitty"
local_path = "~/.config/kitty"
conflict_strategy = "ask"
replace_symlinks = false   # Preserve symlinks to dotfiles repository

# ------------------------------------------------------------------------------
# Discrete File Mappings (FileGroupEngine)
# ------------------------------------------------------------------------------
# Standalone files scattered across the system grouped into a single workspace.
# State is tracked centrally in ~/.local/share/SyncMe/state/<workspace>.db

[[files]]
local_path = "~/.bashrc"
workspace = "shell-files"
remote_path = ".bashrc"
conflict_strategy = "ask"
replace_symlinks = false

[[files]]
local_path = "~/.gitconfig"
workspace = "shell-files"
remote_path = ".gitconfig"
conflict_strategy = "ask"
replace_symlinks = false

[[files]]
local_path = "~/.config/starship.toml"
workspace = "shell-files"
remote_path = "starship.toml"
conflict_strategy = "ask"
replace_symlinks = false

[[files]]
local_path = "~/.config/micro/settings.json"
workspace = "shell-files"
remote_path = "settings.json"
conflict_strategy = "ask"
replace_symlinks = false
```

### Configuration Schema

#### `[server]` Section

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `url` | `string` | `"http://localhost:8000"` | Base URL of the server. Must use `https://` (plain `http://` permitted only for `localhost`, `127.0.0.1`, `::1`). |
| `api_key` | `string` | `""` | Per-client authentication key. Empty and `"default_token"` are strictly rejected. |
| `client_id` | `string` | `"default_client"` | Client identifier (1-64 characters matching `[A-Za-z0-9._-]`). Sent in `X-Client-ID` header. |
| `debounce_seconds` | `float` | `1.0` | Inactivity window (seconds) to debounce rapid filesystem write events before uploading. |
| `poll_interval_seconds` | `integer` | `300` | Periodic fallback polling interval (seconds) to verify synchronization state. |
| `request_timeout` | `float` | `300.0` | HTTP request timeout (seconds) for API operations (connect, read, write). |

#### `[[directories]]` Section

Configures recursive directory monitoring managed by `SyncEngine`:

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `name` | `string` | `"default"` | Workspace identifier on the server (`workspace_id`). |
| `local_path` | `path` | `"~/.syncme_data"` | Local directory path (supports `~` home directory expansion). |
| `conflict_strategy` | `string` | `"ask"` | Conflict resolution strategy: `ask`, `server`, `local`, or `both`. |
| `replace_symlinks` | `boolean` | `false` | If `true`, symlinks are resolved and replaced with actual file contents. If `false`, symlinks are preserved (essential when directories point into a dotfiles git repository). |

#### `[[files]]` Section

Configures individual file synchronization managed by `FileGroupEngine`:

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `local_path` | `path` | *(Required)* | Path to the individual file (supports `~` expansion). |
| `workspace` | `string` | `"files"` | The shared logical workspace identifier on the server. |
| `remote_path` | `string` | `local_path.name` | Target relative file path within the remote workspace. |
| `conflict_strategy` | `string` | `"ask"` | Conflict resolution strategy: `ask`, `server`, `local`, or `both`. |
| `replace_symlinks` | `boolean` | `false` | Whether to replace local symlinks with remote file contents. |

---

## Client Environment Overrides & Ignore Rules

### Supported Client Environment Variables

The client supports direct environment variable overrides for rapid testing, automated container runs, or headless scripts:

| Environment Variable | Overrides | Description |
| :--- | :--- | :--- |
| `SYNCME_CONFIG_PATH` | Path to `config.toml` | Custom path to the client configuration file (default: `~/.config/SyncMe/config.toml`). |
| `SYNCME_SERVER_URL` | `[server].url` | Base URL of the SyncMe server (e.g. `https://sync.example.com`). |
| `SYNCME_API_KEY` | `[server].api_key` | Authentication Bearer token. |
| `SYNCME_CLIENT_ID` | `[server].client_id` | Unique client identifier sent in `X-Client-ID`. |
| `SYNCME_SYNC_DIR` | `[[directories]][0].local_path` | Overrides the local directory path of the primary directory workspace. |
| `SYNCME_REQUEST_TIMEOUT` | `[server].request_timeout` | HTTP request timeout in seconds (default: `300.0`). |

### Ignore System (`.smignore`)

SyncMe-Client supports fine-grained exclusion rules for directory workspaces (`[[directories]]`) via `.smignore` files placed in the workspace root:

- **Syntax & Directives:**
  - Standard globbing and gitignore-style patterns powered by `pathspec` (e.g. `*.pyc`, `cache/`, `node_modules/`).
  - Comment support: Full-line comments starting with `#` or `//`, as well as inline comments (`pattern # comment` or `pattern // comment`).
  - Case-insensitivity: Prefix a pattern line with `(?i)` for case-insensitive matching (e.g. `(?i)*.tmp`).
  - Directory filtering: Prefix a pattern line with `(?d)` for directory-only targeting.
- **Built-in Default Ignores:**
  The client enforces permanent internal ignore patterns that are always active regardless of `.smignore`:
  ```
  .smignore
  .syncme/
  .git/
  *.conflict.*
  .syncme-tmp-*
  .syncme-upload-*
  ```
  > [!WARNING]
  > Because `*.conflict.*` is a built-in ignore pattern, do not name regular files with `.conflict.` in their stem (e.g. `config.conflict.json`), as they will be permanently ignored by the synchronization engine.
- **`.gitignore` Ignored:**
  > [!NOTE]
  > SyncMe deliberately **does not** read or evaluate `.gitignore` files. Only explicit rules defined in `.smignore` and the built-in default ignore list govern synchronization behavior.

---

## Server Administration & Operational Limits

### Server Key Management CLI

SyncMe uses per-client API keys stored securely in `<DATA_DIR>/keys.toml` (restricted to permissions `chmod 600`). Keys are managed via the server key management tool:

```bash
# Via Docker Compose:
docker compose run --rm keys add <client_name>
docker compose run --rm keys list
docker compose run --rm keys revoke <client_name>

# Directly via Python on the host:
python -m app.keys add <client_name>
python -m app.keys list
python -m app.keys revoke <client_name>
```

- **Naming Rules:** Client names must match `[a-z0-9][a-z0-9-]{0,31}` (1–32 characters of lowercase letters, digits, and hyphens; cannot start with a hyphen).
- **Key Generation:** Generates 32-byte cryptographically secure URL-safe tokens (`secrets.token_urlsafe(32)`).
- **Concurrency & Locking:** All file modifications are synchronized with exclusive process locks (`fcntl.flock` on `.keys.lock`).
- **Last Key Protection:** `keys revoke` contains an explicit safety guard: it strictly refuses to revoke the last remaining client key to prevent locking out all clients from the server.

### Server Status & Client Monitoring CLI

SyncMe-Server tracks client telemetry and workspace states in SQLite (`client_state` table). You can inspect connected clients, synchronization lag across workspaces, and individual client activity using the server CLI:

```bash
# Via Docker Compose (opt-in tools profile):
docker compose run --rm cli clients
docker compose run --rm cli status
docker compose run --rm cli client status <client_name>

# Directly inside the running server container:
docker compose exec server python -m app.cli clients
docker compose exec server python -m app.cli status
docker compose exec server python -m app.cli client status <client_name>

# Directly via Python on the host:
python -m app.cli clients
python -m app.cli status
python -m app.cli client status <client_name>
```

- **`clients`**: Lists registered clients from `keys.toml` alongside connection status (`Online`, `Idle`, `Offline`, `Never connected`), last seen timestamp, IP address, and active workspaces.
- **`status`**: Shows a per-workspace overview of the latest revision, active file count, total storage size, last modification timestamp, the client responsible for the latest change, and per-client synchronization lag.
- **`client status <name>`**: Shows details for a specific client, its sync lag per workspace, and its last 10 modified or deleted files.

### Server Storage Layout

Server data is organized into three distinct subdirectories under `<STORAGE_DIR>` (defaulting to `<DATA_DIR>/files`):

```
<STORAGE_DIR>/
├── workspaces/
│   └── {workspace_id}/               # Active workspace files on disk
├── objects/
│   └── {workspace_id}/
│       └── {sha256[:2]}/{sha256[2:]}  # Content-Addressable Storage (CAS) payloads
└── archive/
    └── {workspace_id}/               # Safe disk archives of deleted/overwritten files
```

### Server Environment Variables & Limits

| Environment Variable | Default | Description |
| :--- | :--- | :--- |
| `DATA_DIR` | `./data` | Base directory for database files, keys, and storage. |
| `STORAGE_DIR` / `FILES_DIR` | `<DATA_DIR>/files` | Root directory for workspace content, CAS objects, and archives. |
| `ARCHIVE_DIR` | `<STORAGE_DIR>/archive` | Custom root directory for server-side historical deleted/overwritten file archives. |
| `DB_PATH` | `<DATA_DIR>/server_state.db` | Path to the server SQLite database file (WAL mode). |
| `SYNCME_KEYS_FILE` | `<DATA_DIR>/keys.toml` | Path to the client API keys file. |
| `HOST` | `0.0.0.0` | Bind host interface. |
| `PORT` | `8000` | Bind port. |
| `MAX_UPLOAD_BYTES` | `268435456` (256 MiB) | Maximum allowed file upload payload. Exceeding triggers HTTP 413. |

- **Payload Upload Cap:** Maximum upload size is strictly capped at 256 MiB (`MAX_UPLOAD_BYTES`). Any payload exceeding this size is rejected with HTTP `413 Content Too Large` before disk persistence.
- **Concurrent SSE Stream Limits:** To prevent resource exhaustion, each authenticated client identity is limited to a maximum of 10 concurrent active SSE event streams (`MAX_SUBSCRIBERS_PER_CLIENT`). Attempting to open more streams triggers HTTP `429 Too Many Requests`.
- **Path Length Limits:** Path components are limited to 230 bytes UTF-8 (reserving space for `_<timestamp>` archive suffixes), and total relative paths are capped at 3,500 bytes. Workspace identifiers are restricted to 1–64 characters matching `[A-Za-z0-9][A-Za-z0-9._-]{0,63}` (`"objects"` and `"archive"` are reserved names).

---

## CLI Command Reference

The `syncme-client` binary provides a comprehensive command-line suite:

### Workspace Initialization & Lifecycle

#### `syncme-client init`
```bash
syncme-client init [-d|--directory <name>] [--conflict-strategy ask|server|local|both]
```
Initializes SyncMe workspaces and executes an initial synchronization pass.
- Validates server connectivity and API key permissions.
- Scans local files and compares them against server revision state.
- Automatically prompts if a workspace is already initialized.
- Optional `--conflict-strategy` overrides configured conflict behavior (`ask`, `server`, `local`, `both`).

#### `syncme-client run`
```bash
syncme-client run [-d|--directory <name>]
```
Starts the long-lived client daemon:
- Launches Watchdog filesystem watchers with event debouncing (`debounce_seconds`).
- Establishes persistent SSE event connections to `/api/v1/workspaces/{workspace_id}/events`.
- Runs periodic background reconciliation passes (`poll_interval_seconds`).
- Manages automatic reconnection with exponential backoff (1s to 60s) upon network interruption.

#### `syncme-client sync`
```bash
syncme-client sync [-d|--directory <name>]
```
Executes a synchronous one-shot bidirectional synchronization pass across all (or the specified) workspaces and exits. Blocks until completion. Ideal for scheduled cron jobs, system maintenance scripts, or manual reconciliation checks.

#### `syncme-client status`
```bash
syncme-client status [-d|--directory <name>]
```
Displays current tracking metrics, reporting the number of active files registered in the local SQLite state database.

### Archival & Recovery

#### `syncme-client archive list`
```bash
syncme-client archive list [-d|--directory <name>]
```
Lists all files currently preserved in the local safety archive directory (`.syncme/archive/`), displaying archive timestamped filenames and byte sizes.

#### `syncme-client archive restore`
```bash
syncme-client archive restore <archive_name> [target_path] [-d|--directory <name>]
```
Restores an archived file back to its target local path. If `target_path` is omitted, the path is automatically decoded from the archive filename.

### Conflict Management

#### `syncme-client conflicts list`
```bash
syncme-client conflicts list
```
Scans all local workspaces and outputs every pending, unresolved file conflict with its unique numeric ID and relative path.

#### `syncme-client conflicts count`
```bash
syncme-client conflicts count [--format plain|waybar|json]
```
Reports the total count of unresolved conflicts across all workspaces. Supports three output formats:
- `plain`: Raw integer (e.g. `0` or `3`).
- `waybar`: Formatted JSON payload tailored for status bars like Waybar:
  - If conflicts exist: `{"text": "⚠️ 3", "tooltip": "3 conflicts pending", "class": "warning"}`
  - If synced: `{"text": "✔", "tooltip": "All synced", "class": "synced"}`
- `json`: Structured JSON containing the count and full array of pending conflict records.

#### `syncme-client conflicts resolve`
```bash
syncme-client conflicts resolve
```
Interactive terminal resolution wizard. Iterates through pending conflicts, renders colorized unified diffs (`format_inline_diff`), and prompts for resolution:
- `[s]`: Server version wins (local conflict file is deleted).
- `[l]`: Local version wins (local conflict copy overwrites server version and uploads).
- `[n]`: Skip to next conflict.
- `[q]`: Quit resolution wizard.

### File History & Revisions

#### `syncme-client history`
```bash
syncme-client history <file_path> [-l|--limit <int>] [-d|--directory <name>]
```
Retrieves and displays the complete version history of a file from the server's `file_versions` table. Formatted as an aligned table showing revision index, ISO timestamp, size, originating client ID, and deletion tombstones.

#### `syncme-client restore`
```bash
syncme-client restore <file_path> -r|--revision <int> [-d|--directory <name>]
```
Rolls back a file to a specified historical revision. The server fetches the corresponding snapshot from CAS storage, stages it atomically, commits a new revision entry, and broadcasts an update event so all connected clients sync the restored version immediately.

---

## Conflict Resolution, Diffing & Status Bar Integration

### The Conflict Lifecycle

```
[Local Mutation]           [Remote SSE Event / Polling]
       |                                |
       +---------------+----------------+
                       |
             [Reconciliation Pass]
                       |
                {Hash Mismatch?}
                 /            \
           (Yes)/              \(No)
               v                v
    1. Download & verify       [In Sync]
       remote file
    2. Preserve local as:
       <stem>.conflict.<ts><suffix>
    3. Register conflict in
       local SQLite state
               |
               +----------------+
               |                |
               v                v
        [Desktop Alert]   [Status Bar / Waybar]
        (notify-send)     (conflicts count --format waybar)
```

1. **Automatic Preservation:** When local and remote versions diverge concurrently, SyncMe downloads and validates the server file first into a temporary staging file. It then preserves the local file as `<stem>.conflict.<timestamp><suffix>` and replaces the active file with the server version.
2. **Desktop Alert (`notify-send`):** The daemon dispatches a critical notification via `notify-send`:
   ```bash
   notify-send -u critical -a SyncMe -i dialog-warning "SyncMe Conflict: config.json" \
     "Conflict in workspace 'default': config.json\nServer and local copies have diverged." \
     -A "meld=🔍 Resolve with Meld" \
     -A "local=💾 Keep my local version" \
     -A "server=☁️ Keep server version"
   ```
   - Clicking **🔍 Resolve with Meld** opens Meld displaying a side-by-side visual diff. When Meld exits with code `0`, the merged file is automatically uploaded to the server and the `.conflict` artifact is removed.
     > [!WARNING]
     > Meld exits with return code `0` even when closed without saving. Always remember to save changes within Meld before closing the window, or the unmodified file will be synchronized and the `.conflict` copy will be removed.
   - Clicking **💾 Keep my local version** copies the conflict version over the active file and uploads it.
   - Clicking **☁️ Keep server version** discards the local conflict artifact.
3. **Interactive Terminal Wizard (`conflicts resolve`):**
   The terminal wizard uses `format_inline_diff` to render an inline comparison:
   - **Safety Cap:** Files exceeding 1 MiB display `(File too large for inline diff > 1MB)` to prevent terminal lockup.
   - **Binary Detection:** Detects null bytes in initial 1 KiB chunks and renders `(Binary files differ - text diff unavailable)`.
   - **Sanitization:** Sanitizes control characters and escape sequences to protect the terminal.
   - **Colorized Snippet:** Renders unified diff chunks with ANSI colors (green additions, red deletions, cyan headers), capped at 60 lines.

### Waybar / Status Bar Integration

Integrate conflict alerts directly into your Waybar status bar configuration (`~/.config/waybar/modules.json`):

```jsonc
"custom/syncme": {
  "exec": "syncme-client conflicts count --format waybar",
  "return-type": "json",
  "interval": 15,
  "format": "{}",
  "on-click": "kitty --class dotfiles-floating -e syncme-client conflicts resolve",
  "tooltip": true
}
```

Add accompanying CSS styling to `~/.config/waybar/style.css`:

```css
#custom-syncme.warning {
  color: #f38ba8;
  font-weight: bold;
}

#custom-syncme.synced {
  color: #a6e3a1;
}
```

> [!TIP]
> Setting `"on-click": "kitty --class dotfiles-floating -e syncme-client conflicts resolve"` leverages existing Hyprland floating window rules (`float = true`, `center = true`) for seamless in-terminal conflict resolution without window-tiling disruption.

---

## File Versioning & Content-Addressable Storage (CAS)

SyncMe-Server implements Git-like Content-Addressable Storage for point-in-time file versioning and deduplication.

### Server CAS Layout

File contents are stored by their SHA-256 hash using 2-character hex prefix sharding:

```
<STORAGE_DIR>/
└── objects/
    └── {workspace_id}/
        ├── 0a/
        │   └── 3f8b1c4d9e2... (CAS object payload)
        ├── 7c/
        │   └── d51a8e9f2b3...
        └── ff/
            └── 4a12bc90e31...
```

- When files are uploaded or restored, their contents are hashed and written to a temporary staging file (`cas.*`), flushed with `fsync`, and atomically moved into `objects/{workspace_id}/{sha256[:2]}/{sha256[2:]}`.
- If multiple revisions across time share the identical SHA-256 content, only one copy is retained on disk within that workspace.

### Transactional Metadata (`file_versions`)

Every file revision is recorded in SQLite within an atomic `BEGIN IMMEDIATE` transaction:

```sql
CREATE TABLE file_versions (
    workspace_id TEXT NOT NULL,
    path TEXT NOT NULL,
    revision INTEGER NOT NULL,
    sha256 TEXT,
    size INTEGER NOT NULL,
    mtime REAL NOT NULL,
    client_name TEXT,
    created_at REAL NOT NULL,
    is_deleted BOOLEAN NOT NULL DEFAULT 0,
    PRIMARY KEY (workspace_id, path, revision)
);

CREATE INDEX idx_file_versions_path 
ON file_versions (workspace_id, path, revision DESC);
```

- **Monotonic Revisioning:** Each workspace maintains an incrementing revision counter in `metadata` (`current_revision:{workspace_id}`). Revisions are monotonically incremented across the entire workspace for each mutation.
- **Deletions / Tombstones:** Deleting a file records a tombstone row with `is_deleted = 1`, `sha256 = NULL`, and `size = 0`.
- **Client Attribution:** The authenticated `client_name` is permanently associated with the revision entry.

### Background Startup Indexing & Cleanup

During server startup (`lifespan` handler):
1. **Orphaned Upload Sweeping:** `storage.sweep_stale_temp_files()` cleans up orphaned upload staging files (`.syncme-upload-*`) older than 1 hour (`STALE_TEMP_SECONDS = 3600`).
2. **Background Indexing:** `db.index_existing_files()` runs asynchronously in a worker thread:
   - Scans existing files on disk in `workspaces/`.
   - Populates missing CAS objects and seeds baseline `file_versions` records (`client_name = "baseline"`).
   - Updates files modified offline without bumping revisions if content SHA-256 matches, or increments revisions if contents changed (`client_name = "index"`).

### History Inspection & Rollback Workflow

```bash
# 1. Inspect version history for a file:
syncme-client history configs/settings.json

# Output:
# History for configs/settings.json:
# Revision   | Timestamp                 | Size       | Client               | Deleted   
# -------------------------------------------------------------------------------------
# 34         | 2026-10-09 18:30:12       | 1042       | laptop-client        | No        
# 25         | 2026-10-09 17:15:00       | 980        | workstation-office   | No        
# 18         | 2026-10-09 12:00:22       | 0          | workstation-office   | Yes       
# 10         | 2026-10-08 09:41:10       | 850        | laptop-client        | No        

# 2. Roll back file to revision 10:
syncme-client restore configs/settings.json -r 10
```

Because revision numbers increment monotonically across the entire workspace, revisions for an individual file reflect the workspace's revision state when each modification occurred. In the example above, rolling back to revision 10 copies the CAS object from revision 10 into the active workspace, issues a new revision (e.g. revision 35), and broadcasts an SSE notification to propagate the restored state across all clients.

---

## Server REST API Overview

SyncMe-Server exposes an authenticated REST API. All endpoints under `/api/v1/workspaces/{workspace_id}/` require HTTP Bearer authentication via the `Authorization: Bearer <api_key>` header:

| Method | Endpoint | Description | Headers & Parameters | Status Codes |
| :--- | :--- | :--- | :--- | :--- |
| `GET` | `/health`<br>`/api/v1/health` | Unauthenticated health check probe; verifies database connectivity | None | `200`, `503` |
| `GET` | `/api/v1/workspaces/{workspace_id}/events` | Real-time SSE event stream for workspace mutations | Requires Bearer Auth | `200`, `401`, `429` |
| `GET` | `/api/v1/workspaces/{workspace_id}/sync/state` | Query workspace file metadata and highest revision | Query: `since_revision=<int>` | `200`, `400`, `401` |
| `GET` | `/api/v1/workspaces/{workspace_id}/files/{path}` | Download active file payload | Returns `ETag: "<rev>"` | `200`, `401`, `404` |
| `PUT` | `/api/v1/workspaces/{workspace_id}/files/{path}` | Upload or update file content (OCC revision guarded) | `If-Match: "<rev>"`, Body: bytes | `200`, `401`, `409`, `413` |
| `DELETE`| `/api/v1/workspaces/{workspace_id}/files/{path}` | Mark file deleted and archive previous disk copy | `If-Match: "<rev>"` | `200`, `401`, `404`, `409` |
| `GET` | `/api/v1/workspaces/{workspace_id}/history/{path}` | Retrieve chronological revision history from `file_versions` | Query: `limit=<int>` (default: 50) | `200`, `400`, `401` |
| `GET` | `/api/v1/workspaces/{workspace_id}/revisions/{rev}/{path}` | Download historical revision content directly from CAS | Returns `ETag: "<sha256>"` | `200`, `401`, `404` |
| `POST`| `/api/v1/workspaces/{workspace_id}/history/{path}/restore` | Roll back file to a specified historical CAS revision | JSON: `{"revision": <rev>, "expected_revision": <opt>}` | `200`, `401`, `404`, `409` |
| `GET` | `/api/v1/workspaces/{workspace_id}/archive` | List safety archived disk files for workspace | None | `200`, `401` |
| `POST`| `/api/v1/workspaces/{workspace_id}/archive/restore` | Restore file from server-side safety archive directory | JSON: `{"archive_filename": "...", "target_path": "..."}` | `200`, `401`, `404`, `409` |

---

## Running as a Systemd User Service

SyncMe-Client includes a native systemd user service unit (`systemd/syncme-client.service`) for unattended background execution:

```bash
# 1. Install user service definition:
mkdir -p ~/.config/systemd/user
ln -sf /path/to/SyncMe/SyncMe-Client/systemd/syncme-client.service ~/.config/systemd/user/syncme-client.service

# 2. Reload systemd and enable service:
systemctl --user daemon-reload
systemctl --user enable --now syncme-client.service

# 3. Enable user lingering so daemon persists after logout:
loginctl enable-linger $USER
```

### Operational Commands & Troubleshooting

```bash
# View live daemon logs:
journalctl --user -u syncme-client.service -f

# Check service status:
systemctl --user status syncme-client.service

# Restart daemon:
systemctl --user restart syncme-client.service
```

### Exit Code 78 & Restart Prevention

The systemd service unit defines:

```ini
Restart=on-failure
RestartSec=5
RestartPreventExitStatus=78
```

Exit code `78` (`EX_CONFIG`) is raised when the server returns HTTP `401 Unauthorized` (indicating the client's API key is missing or has been revoked). `RestartPreventExitStatus=78` instructs systemd not to restart the daemon in a tight loop when authentication fails, preventing log saturation and CPU thrashing.

---

## Workspace Layout & Documentation Index

| Component | Path | Comprehensive Guide | Cheat Sheet / Quickstart |
| :--- | :--- | :--- | :--- |
| **SyncMe-Server** | [`./SyncMe-Server`](./SyncMe-Server) | [Server README](./SyncMe-Server/README.md) | [Server TL;DR](./SyncMe-Server/TLDR.md) |
| **SyncMe-Client** | [`./SyncMe-Client`](./SyncMe-Client) | [Client README](./SyncMe-Client/README.md) | [Client TL;DR](./SyncMe-Client/TLDR.md) |
| **Multi-Machine Blueprint** | [`./REMOTE.md`](./REMOTE.md) | [Remote Machine Setup & Reference](./REMOTE.md) | Workstation/Laptop sync, dotfiles, agents, and `.smignore` |

---

## Multi-Repository Workspace Management (`mr`)

This project uses [myrepos (`mr`)](https://myrepos.branchable.com/) configured via `.mrconfig` to coordinate git operations across the root repository and sub-repositories (`SyncMe-Server` and `SyncMe-Client`).

### Setting Up Workspace

```bash
# Register current workspace root with mr:
mr register

# Update all repositories concurrently:
mr update

# Inspect git status across root and sub-repos:
mr status
```

### Common `mr` Commands

- `mr up` / `mr update`: Pull remote branches across root, `SyncMe-Server`, and `SyncMe-Client`.
- `mr status`: Display unstaged changes and status across all repositories.
- `mr diff`: Show unified diff across the entire workspace.
- `mr run git <command>`: Execute an arbitrary git command across each repository (e.g. `mr run git checkout -b feat/my-feature`).
