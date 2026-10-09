# SyncMe TL;DR

Quickstart cheat sheet for the SyncMe synchronization system (Server & Client).

---

## 1. Server Quickstart

```bash
cd SyncMe-Server
cp .env.example .env

# 1. Create client key directory (default uid:gid 1000:1000)
sudo install -d -o 1000 -g 1000 -m 700 keys

# 2. Generate an API key for your client (required before server start)
docker compose run --rm keys add laptop

# 3. Start the server daemon
docker compose up -d --build
```
Default server URL: `http://localhost:8000` (or configured via reverse proxy / HTTPS).

---

## 2. Client Quickstart

```bash
cd SyncMe-Client

# 1. Install client CLI via uv (or pip)
uv tool install --editable .

# 2. Configure client (~/.config/SyncMe/config.toml)
mkdir -p ~/.config/SyncMe
cat <<'EOF' > ~/.config/SyncMe/config.toml
[server]
url = "https://sync.example.com"
api_key = "<PASTE_KEY_FROM_SERVER_KEYS_ADD>"
client_id = "laptop-client"

[[directories]]
name = "Claude"
local_path = "~/.claude"
conflict_strategy = "ask"
replace_symlinks = true

[[files]]
workspace = "shell-files"
local_path = "~/.config/fish/config.fish"
EOF

# 3. Initialize workspaces
syncme-client init

# 4. Run synchronization daemon
syncme-client run
```

---

## 3. Client Command Reference (`syncme-client`)

| Command | Description |
| :--- | :--- |
| `syncme-client init [-d <ws>] [--conflict-strategy ask\|server\|local\|both]` | First-time reconciliation between local disk and server. |
| `syncme-client run [-d <ws>]` | Long-running daemon with filesystem watcher and SSE stream. |
| `syncme-client sync [-d <ws>]` | One-off synchronization pass (reconcile then exit). |
| `syncme-client status [-d <ws>]` | Print tracked file count and sync status. |
| `syncme-client history <file_path> [-l <limit>]` | Show revision history for a file (`Revision`, `Size`, `Client`, `Deleted`). |
| `syncme-client restore <file_path> -r <rev>` | Roll back a file to an earlier Content-Addressable Storage revision. |
| `syncme-client conflicts list` | List pending conflict files (`*.conflict.*`). |
| `syncme-client conflicts count [--format plain\|waybar\|json]` | Return number of unresolved conflicts (for Waybar / scripts). |
| `syncme-client conflicts resolve` | Interactive inline terminal diff wizard (keep local, keep server, or manual edit). |
| `syncme-client archive list [-d <ws>]` | List local safety-archived deleted files. |
| `syncme-client archive restore <name> [target]` | Restore an archived file from `.syncme/archive/`. |

---

## 4. Server Administration Reference

All commands run via Docker Compose in `SyncMe-Server/` or directly via Python on the host:

### Key Management (`app.keys`)
```bash
# Add a new client key (copy printed key immediately):
docker compose run --rm keys add <client_name>

# List active clients (names and creation dates, never keys):
docker compose run --rm keys list

# Revoke a client key (strictly refuses to revoke the last remaining key):
docker compose run --rm keys revoke <client_name>
```

### Status & Client Inspection (`app.cli`)
```bash
# List clients and connection status (Online, Idle, Offline, Never connected):
docker compose run --rm cli clients

# Global status and synchronization lag across all workspaces:
docker compose run --rm cli status

# Detailed status, sync lag, and recent file activity for a client:
docker compose run --rm cli client status <client_name>
```

*(Direct execution inside running server container: `docker compose exec server python -m app.cli <cmd>`)*

---

## 5. Systemd Daemon Service (Client)

```bash
# Enable lingering so daemon runs when logged out:
loginctl enable-linger $USER

# Install and start user systemd service:
mkdir -p ~/.config/systemd/user
ln -sf /path/to/SyncMe/SyncMe-Client/systemd/syncme-client.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now syncme-client.service

# View live logs:
journalctl --user -u syncme-client.service -f
```
