---
name: proxmox-vm
description: Use when the user wants to create, update, delete, or manage VMs on Proxmox. Covers cloud-init VM provisioning, OS setup, guest agent installation, networking, and software deployment — all without manual console interaction.
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, Agent, mcp__proxmox__get_nodes, mcp__proxmox__get_node_status, mcp__proxmox__get_cluster_status, mcp__proxmox__get_vms, mcp__proxmox__get_storage, mcp__proxmox__change_vm_state, mcp__proxmox__create_vm, mcp__proxmox__clone_vm, mcp__proxmox__execute_vm_command, mcp__proxmox__list_isos
---

# Proxmox VM Management Skill

You are managing virtual machines on a Proxmox VE host. Follow these procedures and constraints exactly.

## Environment

- **Proxmox Host**: `<PROXMOX_HOST_IP>:<PROXMOX_PORT>` (replace with your Proxmox IP and port)
- **SSH Access**: `ssh root@<PROXMOX_HOST_IP>` (ensure SSH access is configured)
- **Storage**: `local-lvm` (lvmthin), `local` (dir for ISOs/cloud images)
- **Node Specs**: Update with your node's CPU cores and RAM
- **Network Bridges**: `vmbr0` (primary LAN), `vmbr1` (secondary network, optional)

## Available Cloud Images & ISOs

Check with `mcp__proxmox__list_isos(node="<NODE_NAME>")`. Common images:
- `local:iso/ubuntu-24.04-cloud.img` — Ubuntu 24.04 cloud image (preferred for automated installs)
- `local:iso/ubuntu-24.04.3-live-server-amd64.iso` — Ubuntu 24.04 server ISO
- `local:iso/ubuntu-22.04.5-live-server-amd64.iso` — Ubuntu 22.04 server ISO

---

## Procedure 1: Create a VM (Cloud-Init — Preferred)

Cloud images boot with OS pre-installed. No manual console interaction needed.

### Step 1: Create and configure via SSH to Proxmox host

```bash
ssh -o StrictHostKeyChecking=no root@<PROXMOX_HOST_IP> bash -s << 'SCRIPT'
VMID=<next-available-id>
VM_NAME="<name>"
CORES=<cores>
MEMORY=<memory-in-mb>
DISK_SIZE="<size>G"
CI_USER="<username>"
CI_PASSWORD="<password>"

qm create $VMID --name $VM_NAME --cores $CORES --memory $MEMORY \
  --net0 virtio,bridge=vmbr0 --agent enabled=1 --ostype l26

qm importdisk $VMID /var/lib/vz/template/iso/ubuntu-24.04-cloud.img local-lvm

qm set $VMID --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-${VMID}-disk-0
qm resize $VMID scsi0 $DISK_SIZE
qm set $VMID --ide2 local-lvm:cloudinit
qm set $VMID --boot order=scsi0
qm set $VMID --ciuser $CI_USER
qm set $VMID --cipassword $CI_PASSWORD
qm set $VMID --ipconfig0 ip=dhcp
qm set $VMID --serial0 socket --vga serial0
qm set $VMID --cpu host

echo "=== VM $VMID config ==="
qm config $VMID
SCRIPT
```

**CRITICAL**: Always set `--cpu host`. Default `kvm64` lacks CPU instructions needed by NumPy and other modern packages.

### Step 2: Start the VM

```
mcp__proxmox__change_vm_state(node="<NODE_NAME>", vmid="<vmid>", action="start")
```

### Step 3: Wait for boot and find IP

Wait ~30 seconds, then read the serial console from the Proxmox host:

```bash
ssh -o StrictHostKeyChecking=no root@<PROXMOX_HOST_IP> "python3 -c \"
import socket, time
s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect('/var/run/qemu-server/<VMID>.serial0')
s.setblocking(False)
s.send(b'\n')
time.sleep(2)
try:
    data = s.recv(4096)
    print(data.decode(errors='replace'))
except: pass
s.close()
\""
```

If the VM shows a login prompt, log in via serial console to get the IP:

```bash
ssh -o StrictHostKeyChecking=no root@<PROXMOX_HOST_IP> "python3 -c \"
import socket, time
s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect('/var/run/qemu-server/<VMID>.serial0')
s.setblocking(False)
def send_and_read(cmd, wait=3):
    s.send((cmd + '\n').encode())
    time.sleep(wait)
    data = b''
    try:
        while True:
            chunk = s.recv(4096)
            if not chunk: break
            data += chunk
    except: pass
    return data.decode(errors='replace')
print(send_and_read('<username>', 2))
print(send_and_read('<password>', 3))
print(send_and_read('ip addr show', 3))
s.close()
\""
```

### Step 4: Install QEMU guest agent

The guest agent is NOT pre-installed on cloud images. Install via serial console:

```bash
ssh -o StrictHostKeyChecking=no root@<PROXMOX_HOST_IP> "python3 << 'PYEOF'
import socket, time
s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect('/var/run/qemu-server/<VMID>.serial0')
s.setblocking(False)
def send_and_read(cmd, wait=5):
    s.send((cmd + '\n').encode())
    time.sleep(wait)
    data = b''
    try:
        while True:
            chunk = s.recv(8192)
            if not chunk: break
            data += chunk
    except: pass
    return data.decode(errors='replace')
print(send_and_read('sudo apt-get update -qq', 30))
print(send_and_read('sudo apt-get install -y qemu-guest-agent', 30))
print(send_and_read('sudo systemctl enable --now qemu-guest-agent', 5))
s.close()
PYEOF"
```

Wait ~10 seconds, then verify:

```bash
ssh -o StrictHostKeyChecking=no root@<PROXMOX_HOST_IP> "qm agent <VMID> ping"
```

No output = success. Once the agent is running, use `mcp__proxmox__execute_vm_command` for all further commands.

### Step 5: Configure networking (optional static IP)

For a secondary subnet, add a second NIC in Proxmox first, then configure in the VM:

```
mcp__proxmox__execute_vm_command(node="<NODE_NAME>", vmid="<vmid>",
  command="cat > /etc/netplan/60-secondary.yaml << 'EOF'\nnetwork:\n  version: 2\n  ethernets:\n    ens19:\n      addresses:\n        - <SECONDARY_IP>/<CIDR>\nEOF")
```

Then: `netplan apply`

### Step 6: Install Tailscale (optional)

```
execute_vm_command: curl -fsSL https://tailscale.com/install.sh -o /tmp/tailscale.sh
execute_vm_command: bash /tmp/tailscale.sh
execute_vm_command: tailscale up
```

The last command returns an auth URL. Share it with the user to authenticate.

---

## Procedure 2: Create a VM (ISO-based)

Use the MCP `create_vm` tool. **Note**: This requires manual OS installation via the Proxmox console — avoid when possible, prefer cloud-init.

```
mcp__proxmox__create_vm(
  node="<NODE_NAME>",
  name="<name>",
  iso="local:iso/ubuntu-24.04.3-live-server-amd64.iso",
  cores=4,
  memory=8192,
  storage="local-lvm"
)
```

After creation, set CPU to host:
```bash
ssh root@<PROXMOX_HOST_IP> "qm set <vmid> --cpu host"
```

Tell the user to complete OS installation via the Proxmox web console at `https://<PROXMOX_HOST_IP>:<PROXMOX_PORT>`.

---

## Procedure 3: Update a VM

### Change VM specs (requires shutdown for CPU/memory changes)

```bash
ssh root@<PROXMOX_HOST_IP> "qm set <vmid> --cores <n> --memory <mb>"
# If changing CPU or memory, shutdown and restart:
ssh root@<PROXMOX_HOST_IP> "qm shutdown <vmid> && sleep 15 && qm start <vmid>"
```

### Resize disk (online, no shutdown needed)

```bash
ssh root@<PROXMOX_HOST_IP> "qm resize <vmid> scsi0 +10G"  # Add 10GB
```

### Add a network interface

```bash
ssh root@<PROXMOX_HOST_IP> "qm set <vmid> --net1 virtio,bridge=vmbr1"
```

### Change VM state

```
mcp__proxmox__change_vm_state(node="<NODE_NAME>", vmid="<vmid>", action="start|stop|shutdown|reboot|reset|suspend|resume|pause|hibernate")
```

- `shutdown` = graceful (ACPI signal), `stop` = force kill
- Prefer `shutdown` over `stop`

---

## Procedure 4: Delete a VM

### Step 1: Stop the VM if running

```
mcp__proxmox__change_vm_state(node="<NODE_NAME>", vmid="<vmid>", action="stop")
```

### Step 2: Destroy the VM (removes disks too)

```bash
ssh root@<PROXMOX_HOST_IP> "qm destroy <vmid> --purge"
```

The `--purge` flag removes the VM from backup jobs and replication configs.

**Always confirm with the user before deleting a VM.**

---

## Procedure 5: Run Commands Inside a VM

### Short commands (< 5 seconds)

Use the MCP tool directly:

```
mcp__proxmox__execute_vm_command(node="<NODE_NAME>", vmid="<vmid>", command="<command>")
```

### Long-running commands (> 5 seconds)

The MCP tool has a 5-second timeout. For long commands:

**Option A: nohup via MCP tool**
```
execute_vm_command: nohup <command> > /tmp/<logfile>.log 2>&1 &
```
Then check progress:
```
execute_vm_command: tail -10 /tmp/<logfile>.log
```

**Option B: qm guest exec via SSH (30s timeout)**
```bash
ssh root@<PROXMOX_HOST_IP> "qm guest exec <vmid> -- /bin/bash -c '<command>'"
```

**Option C: Write a script, then execute it**
```
execute_vm_command: cat > /tmp/setup.sh << 'SCRIPT'
#!/bin/bash
<commands here>
SCRIPT

execute_vm_command: chmod +x /tmp/setup.sh
execute_vm_command: nohup /tmp/setup.sh > /tmp/setup.log 2>&1 &
```

---

## Procedure 6: Clone a VM

```
mcp__proxmox__clone_vm(
  node="<NODE_NAME>",
  vmid="<source-vmid>",
  name="<new-name>",
  cores=0,      # 0 = same as source
  memory=0,     # 0 = same as source
  storage="local-lvm"
)
```

---

## Critical Constraints

### execute_vm_command limitations
- **No `&&` chaining** — commands with `&&` or `||` will fail or only run the first part
- **No pipes (`|`)** — `ls | grep foo` won't work
- **No glob expansion** — `rm *.log` won't expand; use explicit paths
- **5-second timeout** — anything longer must use nohup or script approach
- **No PATH** — guest agent runs with minimal PATH; use `/bin/bash -c '...'` wrapper
- **Max 2-3 parallel calls** — more causes timeouts on the MCP server
- **Runs as root** — all commands execute as root in the VM

### apt-get lock handling
After running `apt-get install` via nohup, always check the lock before running another:
```
execute_vm_command: fuser /var/lib/dpkg/lock-frontend
```
If a PID is returned, wait. If empty, safe to proceed.

### Cloud image first-boot
- Guest agent is NOT installed — must install via serial console
- SSH may need password auth enabled: `sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config`
- Cloud-init runs on first boot only — subsequent config changes require `cloud-init clean && reboot`

---

## Current VM Inventory

Always verify with `mcp__proxmox__get_vms()` before making changes. Update this table with your own VMs:

| VMID | Name | IP (LAN) | IP (Tailscale) | Purpose |
|------|------|----------|----------------|---------|
| — | — | — | — | — |
