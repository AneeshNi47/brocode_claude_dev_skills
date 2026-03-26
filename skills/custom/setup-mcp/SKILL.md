---
name: setup-mcp
description: Use when the user wants to install, configure, or troubleshoot the Proxmox MCP server for Claude Code. Covers cloning the repo, creating API tokens, configuring .mcp.json, and verifying the connection.
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# Setup Proxmox MCP Server Skill

Install and configure the Proxmox VE MCP server so Claude Code can manage VMs, run commands, and monitor infrastructure.

**User provides:** $ARGUMENTS (Proxmox host IP, credentials, or troubleshooting context)

---

## Quick Reference: Full Setup

```bash
# 1. Clone the MCP server
git clone https://github.com/Markermav/ProxmoxMCP.git ~/ProxmoxMCP-advance
cd ~/ProxmoxMCP-advance

# 2. Create Python venv and install
uv venv
source .venv/bin/activate
uv pip install -e ".[dev]"

# 3. Create config
mkdir -p proxmox-config
cat > proxmox-config/config.json << 'EOF'
{
  "proxmox": {
    "host": "PROXMOX_HOST_IP",
    "port": 8006,
    "verify_ssl": false,
    "service": "PVE"
  },
  "auth": {
    "user": "root@pam",
    "token_name": "TOKEN_NAME",
    "token_value": "TOKEN_VALUE"
  },
  "logging": {
    "level": "INFO",
    "format": "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
  }
}
EOF

# 4. Register with Claude Code (project-level)
# Place .mcp.json in your project root — see Step 4 below

# 5. Restart Claude Code to pick up the MCP server
```

---

## Step-by-Step Guide

### Step 1: Prerequisites

**On your local machine (macOS/Linux):**
- Python 3.10+ installed
- `uv` package manager (recommended) or `pip`
- Claude Code CLI installed
- Network access to the Proxmox host (LAN or Tailscale)

**On the Proxmox host:**
- Proxmox VE 7.x or 8.x
- API token created (see Step 2)
- QEMU guest agent installed on VMs you want to manage (see Step 6)

### Step 2: Create Proxmox API Token

1. Open the Proxmox web UI: `https://<proxmox-ip>:8006`
2. Go to **Datacenter** → **Permissions** → **API Tokens**
3. Click **Add**:
   - **User**: `root@pam` (or another user with sufficient permissions)
   - **Token ID**: `mcp_proxmox` (or any name)
   - **Privilege Separation**: **Uncheck** this (token inherits user's permissions)
4. Click **Add** and **copy the token value** — it's only shown once

**Required permissions** (if using a non-root user):
- `VM.Audit` — list and view VMs
- `VM.Console` — execute commands via guest agent
- `VM.PowerMgmt` — start/stop/reboot VMs
- `VM.Allocate` — create/delete VMs
- `VM.Config.Disk`, `VM.Config.CPU`, `VM.Config.Memory`, `VM.Config.Network` — modify VM settings
- `Sys.Audit` — view node/cluster status
- `Datastore.Allocate`, `Datastore.AllocateSpace` — manage disks and storage

### Step 3: Clone and Install the MCP Server

```bash
# Clone the repo
git clone https://github.com/Markermav/ProxmoxMCP.git ~/ProxmoxMCP-advance
cd ~/ProxmoxMCP-advance

# Create Python virtual environment
uv venv
source .venv/bin/activate

# Install the package
uv pip install -e ".[dev]"
```

**Alternative with pip (if uv is not installed):**
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
```

### Step 4: Create Configuration File

```bash
mkdir -p proxmox-config
```

Create `proxmox-config/config.json`:

```json
{
  "proxmox": {
    "host": "<PROXMOX_HOST_IP>",
    "port": 8006,
    "verify_ssl": false,
    "service": "PVE"
  },
  "auth": {
    "user": "root@pam",
    "token_name": "<TOKEN_NAME>",
    "token_value": "<TOKEN_VALUE>"
  },
  "logging": {
    "level": "INFO",
    "format": "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
  }
}
```

Replace:
- `<PROXMOX_HOST_IP>` — your Proxmox host's IP (Tailscale IP recommended for remote access)
- `<TOKEN_NAME>` — the token ID from Step 2 (e.g., `mcp_proxmox`)
- `<TOKEN_VALUE>` — the token value from Step 2

### Step 5: Register with Claude Code

Create `.mcp.json` in your project root (or any working directory where you run Claude Code):

```json
{
  "mcpServers": {
    "proxmox": {
      "command": "<HOME>/ProxmoxMCP-advance/.venv/bin/python",
      "args": ["-m", "proxmox_mcp.server"],
      "cwd": "<HOME>/ProxmoxMCP-advance",
      "env": {
        "PYTHONPATH": "<HOME>/ProxmoxMCP-advance/src",
        "PROXMOX_MCP_CONFIG": "<HOME>/ProxmoxMCP-advance/proxmox-config/config.json",
        "PROXMOX_HOST": "<PROXMOX_HOST_IP>",
        "PROXMOX_USER": "root@pam",
        "PROXMOX_TOKEN_NAME": "<TOKEN_NAME>",
        "PROXMOX_TOKEN_VALUE": "<TOKEN_VALUE>",
        "PROXMOX_PORT": "8006",
        "PROXMOX_VERIFY_SSL": "false",
        "PROXMOX_SERVICE": "PVE"
      }
    }
  }
}
```

Replace all `<HOME>` with the actual home directory path (e.g., `/Users/username` on macOS, `/home/username` on Linux).

**Scope options:**
- **Project-level** (`.mcp.json` in project root): Only available in that project. Shared via git.
- **User-level** (via CLI): Available in all projects for this user.

To register at user level instead:
```bash
claude mcp add --scope user proxmox \
  ~/ProxmoxMCP-advance/.venv/bin/python \
  -- -m proxmox_mcp.server
```

### Step 6: Set Up QEMU Guest Agent on VMs

The `execute_vm_command` tool requires the QEMU guest agent running inside each VM.

**In Proxmox UI:**
1. Select the VM → **Options** → **QEMU Guest Agent** → **Enable**
2. **Fully shut down** the VM (not reboot), then start it again

**Inside the VM:**
```bash
sudo apt-get update
sudo apt-get install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

**Verify from Proxmox host:**
```bash
qm agent <vmid> ping
```

No output = success.

### Step 7: Verify the Setup

Restart Claude Code (exit and reopen), then test:

1. Ask Claude: "List all VMs on Proxmox"
   - Should use `mcp__proxmox__get_vms`

2. Ask Claude: "What's the status of node pve01?"
   - Should use `mcp__proxmox__get_node_status`

3. Ask Claude: "Run `uname -a` on VM 101"
   - Should use `mcp__proxmox__execute_vm_command`

---

## Available MCP Tools After Setup

| Tool | Parameters | Description |
|------|-----------|-------------|
| `get_nodes` | — | List cluster nodes |
| `get_node_status` | `node` | Detailed node info (CPU, RAM, uptime) |
| `get_cluster_status` | — | Cluster health |
| `get_vms` | — | List all VMs with status and resources |
| `get_storage` | — | List storage pools |
| `list_isos` | `node` | List available ISO images |
| `change_vm_state` | `node, vmid, action` | start/stop/shutdown/reboot/reset/suspend/resume/pause/hibernate |
| `create_vm` | `node, name, iso, cores?, memory?, storage?` | Create VM from ISO |
| `clone_vm` | `node, vmid, name, cores?, memory?, storage?` | Clone an existing VM |
| `execute_vm_command` | `node, vmid, command` | Run shell command inside VM via guest agent |

---

## Tool Limitations

### execute_vm_command constraints
- **No `&&` or `||` chaining** — only the first command runs
- **No pipes (`|`)** — `ls | grep foo` won't work
- **No glob expansion** — `rm *.log` won't expand
- **~5 second timeout** — long commands must use `nohup ... &`
- **Runs as root** — all commands execute with root privileges
- **No PATH** — wrap commands in `/bin/bash -c '...'` if needed
- **Max 2-3 parallel calls** — more causes API timeouts

### Workaround for long commands
```
execute_vm_command: nohup apt-get install -y nginx > /tmp/install.log 2>&1 &
# Wait, then check:
execute_vm_command: tail -10 /tmp/install.log
```

### Workaround for multi-step scripts
```
execute_vm_command: cat > /tmp/setup.sh << 'SCRIPT'
#!/bin/bash
command1
command2
command3
SCRIPT

execute_vm_command: chmod +x /tmp/setup.sh
execute_vm_command: nohup /tmp/setup.sh > /tmp/setup.log 2>&1 &
```

---

## Troubleshooting

### MCP server not appearing in Claude Code

1. Check `.mcp.json` is in the correct directory (project root or `~/.claude/`)
2. Verify the Python path exists: `ls <HOME>/ProxmoxMCP-advance/.venv/bin/python`
3. Restart Claude Code completely (exit and reopen)
4. Check Claude Code MCP status: run `claude mcp list`

### "Connection refused" or timeout errors

1. Verify Proxmox host is reachable: `ping <PROXMOX_HOST_IP>`
2. Check API token is valid:
   ```bash
   curl -k -H "Authorization: PVEAPIToken=root@pam!mcp_proxmox=<TOKEN_VALUE>" \
     https://<PROXMOX_HOST_IP>:8006/api2/json/nodes
   ```
3. Ensure port 8006 is not blocked by firewall

### "QEMU guest agent is not running"

1. Check guest agent is enabled in Proxmox UI (VM → Options)
2. Check it's installed inside the VM: `systemctl status qemu-guest-agent`
3. If just enabled in UI: **full shutdown + start** required (not reboot)

### execute_vm_command returns empty or errors

- Command may have timed out — use `nohup` approach
- Check if the command exists in the VM's PATH
- Try wrapping in `/bin/bash -c '<command>'`

### SSL verification errors

Set `PROXMOX_VERIFY_SSL` to `false` in `.mcp.json` env vars. Self-signed certs are common on Proxmox.

---

## Project Structure Reference

```
~/ProxmoxMCP-advance/
├── .venv/                          # Python virtual environment
├── src/proxmox_mcp/
│   ├── server.py                   # Main MCP server entry point
│   ├── config/                     # Config loader + models
│   ├── core/                       # Proxmox API client
│   ├── formatting/                 # Output formatters
│   ├── tools/                      # Tool implementations
│   │   ├── vm.py                   # VM management (create, clone, state)
│   │   └── console/                # VM command execution
│   └── utils/                      # Utilities
├── proxmox-config/
│   └── config.json                 # Credentials and connection config
├── pyproject.toml                  # Package definition
└── tests/                          # Test suite
```

---

## Security Notes

- **Never commit `.mcp.json` with real tokens to a public repo** — add it to `.gitignore`
- API tokens have the same permissions as the user they belong to — use a limited user if possible
- `execute_vm_command` runs as root inside VMs — be careful with destructive commands
- Proxmox web UI and API default to self-signed SSL — `verify_ssl: false` is expected but understand the risk on untrusted networks
