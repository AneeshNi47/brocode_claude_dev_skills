---
name: deploy-frappe
description: Use when the user wants to deploy Frappe Framework, ERPNext, Frappe Press, or any Frappe app on a server or Proxmox VM. Covers full stack installation — system dependencies, MariaDB, Node.js, bench setup, Python 3.12 compatibility fixes, site creation, and networking.
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, Agent, mcp__proxmox__execute_vm_command, mcp__proxmox__change_vm_state, mcp__proxmox__get_vms
---

# Deploy Frappe App Skill

Deploy Frappe Framework, ERPNext, Press, or any custom Frappe app on a server. Works on bare metal, cloud VMs, or Proxmox VMs.

If deploying on a Proxmox VM, use the `/proxmox-vm` skill first to create the VM, then return here for software setup.

**User provides:** $ARGUMENTS (app name, branch, site name, or other context)

---

## Quick Reference: Full Command Sequence

For experienced users — copy-paste this entire block on a fresh Ubuntu 24.04 server. Detailed explanations follow in later sections.

```bash
# === AS ROOT ===

# 1. System dependencies
apt-get update -qq
apt-get install -y git python3-dev python3-pip python3-venv python3-setuptools \
  redis-server software-properties-common curl wget gnupg2 xvfb libfontconfig \
  wkhtmltopdf nginx supervisor pkg-config \
  mariadb-server mariadb-client libmysqlclient-dev

# 2. MariaDB config
cat > /etc/mysql/mariadb.conf.d/99-frappe.cnf << 'EOF'
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
[mysql]
default-character-set = utf8mb4
EOF
systemctl restart mariadb
mariadb -u root -e "ALTER USER root@localhost IDENTIFIED BY 'DB_PASSWORD'; FLUSH PRIVILEGES;"

# 3. Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs
npm install -g yarn

# 4. Frappe Bench CLI
pip3 install frappe-bench --break-system-packages

# 5. Create frappe user (if not exists)
useradd -m -s /bin/bash frappe
echo "frappe:FRAPPE_PASSWORD" | chpasswd
usermod -aG sudo frappe

# === AS FRAPPE USER ===
su - frappe

# 6. Init bench
bench init --frappe-branch version-15 frappe-bench
cd frappe-bench

# 7. Get app (replace with desired app)
bench get-app APP_NAME --branch BRANCH

# 8. Fix Python 3.12 dependency issues (BEFORE creating site)
env/bin/pip install "setuptools<81"
env/bin/pip install urllib3==1.26.20
env/bin/pip install stripe==5.5.0
env/bin/pip install "ansible-core>=2.15"

# 9. Create site
bench new-site SITE_NAME \
  --mariadb-root-password DB_PASSWORD \
  --admin-password ADMIN_PASSWORD \
  --install-app APP_NAME
bench use SITE_NAME
bench enable-scheduler

# 10. Start
bench start
```

---

## Step-by-Step with Explanations

### Step 1: System Dependencies

```bash
apt-get update -qq
apt-get install -y \
  git python3-dev python3-pip python3-venv python3-setuptools \
  redis-server software-properties-common \
  curl wget gnupg2 xvfb libfontconfig wkhtmltopdf \
  nginx supervisor pkg-config \
  mariadb-server mariadb-client libmysqlclient-dev
```

| Package | Why |
|---------|-----|
| `python3-dev`, `python3-pip`, `python3-venv` | Frappe runs on Python, bench creates a venv |
| `redis-server` | Caching, queues, real-time (socketio) |
| `mariadb-server`, `libmysqlclient-dev` | Database (Frappe requires MariaDB, not MySQL) |
| `wkhtmltopdf` | PDF generation (print formats, invoices) |
| `nginx` | Reverse proxy for production mode |
| `supervisor` | Process manager for production mode |
| `pkg-config` | Required by bench during `bench init` |

### Step 2: Configure MariaDB

Frappe requires `utf8mb4` character set:

```bash
cat > /etc/mysql/mariadb.conf.d/99-frappe.cnf << 'EOF'
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
[mysql]
default-character-set = utf8mb4
EOF

systemctl restart mariadb
mariadb -u root -e "ALTER USER root@localhost IDENTIFIED BY '<db-password>'; FLUSH PRIVILEGES;"
```

### Step 3: Install Node.js 20

Frappe v15 frontend and many apps (especially Press) require Node.js 20+:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs
npm install -g yarn
```

**Do NOT use Node.js 18** — Press and newer Frappe apps use `@vitejs/plugin-vue@6` which requires `^20.19.0 || >=22.12.0`.

### Step 4: Install Frappe Bench CLI

```bash
pip3 install frappe-bench --break-system-packages
```

### Step 5: Create Frappe User

Never run bench as root:

```bash
useradd -m -s /bin/bash frappe
echo "frappe:<password>" | chpasswd
usermod -aG sudo frappe
su - frappe
```

### Step 6: Initialize Bench

```bash
bench init --frappe-branch version-15 frappe-bench
cd frappe-bench
```

**CRITICAL**: Use `version-15`, NOT `develop`. The develop branch requires Python 3.14+ as of March 2026.

This creates:
```
frappe-bench/
├── apps/frappe/     # Frappe framework
├── env/             # Python virtual environment
├── sites/           # Sites directory
├── config/          # Procfile, supervisor/nginx configs
└── logs/
```

### Step 7: Get Your App

```bash
# ERPNext
bench get-app erpnext --branch version-15

# Frappe Press
bench get-app press --branch develop

# Any custom app
bench get-app https://github.com/org/app-name --branch main
```

### Step 8: Fix Python 3.12 Dependency Issues

**Run these BEFORE creating a site or installing apps.** These are known incompatibilities with Python 3.12 (Ubuntu 24.04):

```bash
# Fix 1: setuptools >= 82 removed pkg_resources
# Symptom: ModuleNotFoundError: No module named 'pkg_resources'
env/bin/pip install "setuptools<81"

# Fix 2: urllib3 v2 breaks python-telegram-bot
# Symptom: ModuleNotFoundError: No module named 'urllib3.contrib.appengine'
env/bin/pip install urllib3==1.26.20

# Fix 3: stripe 2.56 has broken six.moves on Python 3.12
# Symptom: ModuleNotFoundError: No module named 'stripe.six.moves'
# Only needed if installing Press or apps that depend on stripe
env/bin/pip install stripe==5.5.0

# Fix 4: ansible 3.x uses six.moves (broken on Python 3.12)
# Symptom: ModuleNotFoundError: No module named 'ansible.module_utils.six.moves'
# Only needed if installing Press
env/bin/pip install "ansible-core>=2.15"
```

**Verify all fixes:**
```bash
env/bin/python -c "import pkg_resources; print('pkg_resources OK')"
env/bin/python -c "import urllib3; print('urllib3', urllib3.__version__)"
env/bin/python -c "import stripe; print('stripe OK')"
env/bin/python -c "import ansible; print('ansible OK')"
```

### Step 9: Create Site and Install App

```bash
bench new-site <site-name> \
  --mariadb-root-password <db-password> \
  --admin-password <admin-password> \
  --install-app <app-name>

bench use <site-name>
bench enable-scheduler
```

If `install-app` fails with `DuplicateEntryError` (from a previous partial install):

```bash
bench --site <site-name> console
```
```python
frappe.db.sql("DELETE FROM `tabModule Def` WHERE app_name='<app>'")
frappe.db.sql("DELETE FROM `tabInstalled Application` WHERE app_name='<app>'")
frappe.db.commit()
exit()
```

Then retry `bench --site <site-name> install-app <app-name>`.

If too corrupted, drop and recreate:
```bash
bench drop-site <site-name> --mariadb-root-password <db-password> --force
# Then rerun bench new-site...
```

### Step 10: Start the Server

**Development mode:**
```bash
bench start
# Accessible at http://<server-ip>:8000
```

**Production mode (systemd + nginx + supervisor):**
```bash
# As root:
sudo bench setup production frappe
# Accessible at http://<server-ip> (port 80)
```

---

## Deploying on a Proxmox VM via Guest Agent

When running commands via `mcp__proxmox__execute_vm_command`, adapt as follows:

### Short commands (< 5 seconds) — use MCP directly
```
execute_vm_command(node="pve01", vmid="<vmid>", command="<command>")
```

### Long commands (apt install, bench init, etc.) — use nohup
```
execute_vm_command: nohup <command> > /tmp/<logfile>.log 2>&1 &
```

Check progress:
```
execute_vm_command: tail -10 /tmp/<logfile>.log
```

Check if apt lock is free:
```
execute_vm_command: fuser /var/lib/dpkg/lock-frontend
```

### Multi-step operations — write a script
```
execute_vm_command: cat > /tmp/setup.sh << 'SCRIPT'
#!/bin/bash
apt-get update -qq
apt-get install -y git python3-dev python3-pip ...
SCRIPT

execute_vm_command: chmod +x /tmp/setup.sh
execute_vm_command: nohup /tmp/setup.sh > /tmp/setup.log 2>&1 &
```

### Commands as frappe user
```
execute_vm_command: su - frappe -c "cd frappe-bench && bench start"
```

---

## Common App Recipes

### ERPNext
```bash
bench get-app erpnext --branch version-15
bench new-site erp.local --mariadb-root-password <pw> --admin-password <pw> --install-app erpnext
```

### Frappe Press
```bash
bench get-app press --branch develop
# Apply ALL 4 Python fixes from Step 8
bench new-site press.local --mariadb-root-password <pw> --admin-password <pw> --install-app press
```

### HRMS
```bash
bench get-app hrms --branch version-15
bench new-site hr.local --mariadb-root-password <pw> --admin-password <pw> --install-app hrms
```

### Multiple apps on one site
```bash
bench new-site mysite.local --mariadb-root-password <pw> --admin-password <pw>
bench --site mysite.local install-app erpnext
bench --site mysite.local install-app hrms
```

---

## Version Compatibility Matrix

| Component | Tested Version | Notes |
|-----------|---------------|-------|
| Ubuntu | 24.04 LTS | Python 3.12 — needs dependency fixes |
| Ubuntu | 22.04 LTS | Python 3.10 — fewer issues |
| Python | 3.12.3 | Requires setuptools, urllib3, stripe, ansible fixes |
| Node.js | 20.20.1 | Required for Press; 18.x works for ERPNext only |
| MariaDB | 10.11.14 | Frappe warns >10.8 is untested, but works fine |
| Redis | 7.0.15 | No issues |
| Frappe | version-15 | Stable; develop needs Python 3.14+ |
| frappe-bench | 5.29.1 | Latest as of March 2026 |

---

## Proxmox-Specific Notes

- **CPU type must be `host`** — default `kvm64` causes NumPy `RuntimeError: X86_V2` failure
- **Cloud images preferred** — fully automated OS setup, no console interaction
- **Guest agent not pre-installed** on cloud images — install via serial console first
- **After changing CPU type**: must do full shutdown + start (not reboot)
