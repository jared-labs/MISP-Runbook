# VMCTI01 — MISP Install & Base OS

> Single-VM deployment of MISP on Ubuntu Server 24.04 **minimal**, running as `vmcti01` on Proxmox and reachable at `https://misp.vmcti01.lan/`.

---

## At a Glance

| Field | Value |
|-------|-------|
| VM/CT Name | `VMCTI01` |
| Host Node | Preferred Proxmox node |
| IP Address | `10.0.0.X/24` (static, on `vmbr0`) |
| OS | Ubuntu Server 24.04 LTS (minimal) |
| vCPU / RAM / Disk | 4 / 8 GB / 120 GB (thin) |
| CPU Type | `host` |
| Key Ports | `443` (HTTPS), `80` (HTTP), `22` (SSH) |
| Credentials | Default admin: `admin@admin.test` / `admin` — change immediately |
| FQDN | `misp.vmcti01.lan` |
| Depends On | Proxmox, homelab DNS (optional) |
| Worker Manager | Supervisor (not systemd) |

---

## Prerequisites

- [ ] Proxmox node with available resources (4 vCPU / 8 GB RAM / 120 GB disk)
- [ ] Network: static IP reserved in 10.0.0.0/24 on `vmbr0`
- [ ] ISO uploaded: `ubuntu-24.04-live-server-amd64.iso`
- [ ] Upstream dependencies running: Proxmox VE

---

## Scope

This runbook covers:

- Proxmox VM creation for **VMCTI01**
- Ubuntu 24.04 minimal install + basic tooling
- Hostname / FQDN / `/etc/hosts` setup for `misp.vmcti01.lan`
- MISP installation via the official Ubuntu installer scripts
- Service model (Apache/HTTPD + Supervisor-managed `misp-workers`)
- Basic validation from a browser (HTTPS) and CLI checks
- Real-world quirks and gotchas encountered during first install

It **does not** cover:

- MISP UI configuration (org/users/feeds) — see "MISP Initial Configuration" runbook
- MISP → Cribl / Graylog enrichment — see "MISP → Cribl IOC Export & Enrichment" runbook

---

## Homelab Context

- Proxmox VE homelab cluster ("laptop stack")
- All service VMs use **Ubuntu Server 24.04 minimal**
- Existing stack:
  - `vmgray01` — Graylog / OpenSearch / MongoDB
  - `vmcrib01` — Cribl Stream
  - `vmmon01` — Prometheus / Grafana
  - `vmvuln01` — Greenbone / OpenVAS
  - `vmvlt01` — Vaultwarden
- New VM in this runbook:
  - Proxmox VM name: **VMCTI01**
  - Hostname inside VM: **vmcti01**
  - Intended FQDN: **misp.vmcti01.lan**
  - Static IP: `10.0.0.X/24` on `vmbr0` (pick an unused IP in 10.0.0.0/24)

---

## 0 — Proxmox VM Definition (before OS install)

Create the VM in the Proxmox UI:

- Node: your preferred Proxmox node
- VM ID: next available
- Name: `VMCTI01`

Disks / CPU / Memory:

- **OS:**
  - ISO: `ubuntu-24.04-live-server-amd64.iso` (or newer 24.04.x)
- **System:**
  - Machine: `q35` (or default)
  - BIOS: `OVMF (UEFI)` or `SeaBIOS` (either is fine — choose what you use elsewhere)
  - SCSI Controller: `VirtIO SCSI`
- **Disks:**
  - Bus/Device: `scsi0`
  - Size: `120G` (thin)
  - Cache: `Write back` (if comfortable with battery-backed storage) or `Default`
- **CPU:**
  - Sockets: `1`
  - Cores: `4`
  - Type: `host`
- **Memory:**
  - `8192` MB RAM
  - Ballooning: enabled or disabled per your cluster standards
- **Network:**
  - Model: `VirtIO (paravirtualized)`
  - Bridge: `vmbr0`
  - VLAN tag: (set if you use VLANs, otherwise leave blank)

Attach the Ubuntu ISO and ensure the VM boots from CD first.

> **Note:** CPU type `host` gives MISP / MariaDB / PHP the full feature set of the physical CPU. No special flags required (MISP doesn't need AVX).

> **Note:** You can run lighter (2 vCPUs / 4 GB RAM / 80 GB disk) for testing, but MISP will be happier with the recommended specs if you plan to enable multiple feeds, store larger event sets, or run background workers reliably.

---

## 1 — Ubuntu 24.04 Minimal Install

Boot the VM and walk through the Ubuntu Server 24.04 installer:

- Language: `English` (or your preference)
- Keyboard: your layout
- Installation type: **Ubuntu Server (minimized)** or equivalent minimal profile
- Network:
  - Interface on `vmbr0` should be visible
  - Configure **static IP**:
    - Address: `10.0.0.X/24`
    - Gateway: `10.0.0.1` (or your router)
    - DNS: your homelab DNS (e.g., `10.0.0.1` or Pi-hole)
- Proxy: leave blank unless you use one
- Mirror: default Ubuntu mirror is usually fine
- Filesystem:
  - Use entire disk
  - Default partitioning is fine for a small node
- Profile:
  - Username: e.g. `admin`
  - Server name: `vmcti01`
- SSH:
  - **Enable OpenSSH server** in the installer (recommended)
  - Optionally import SSH keys from GitHub/Launchpad or use a password
- Additional snaps / server roles: skip (we'll install manually later)

After install, reboot and log in via console or SSH:

```bash
ssh admin@10.0.0.X
```

---

## 2 — Base OS Post-Install Setup

### 2.1 Update and upgrade

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

Log back in after reboot.

### 2.2 Install basic tooling

Ubuntu minimal often omits some basics (like `nano`).

```bash
sudo apt install -y \
    nano vim-tiny curl wget git htop unzip ca-certificates gnupg lsb-release
```

- `nano` / `vim-tiny`: quick editing
- `curl` / `wget`: grabbing scripts & debugging HTTP
- `git`: required for MISP repo
- `htop`: basic resource view

### 2.3 Set/verify hostname

Installer should already have set `vmcti01`, but confirm:

```bash
hostnamectl
```

If needed, set explicitly:

```bash
sudo hostnamectl set-hostname vmcti01
```

Log out and back in to see the updated prompt.

---

## 3 — Hostname, FQDN & `/etc/hosts`

MISP expects a proper FQDN; we want `misp.vmcti01.lan`.

### 3.1 Configure local FQDN on the VM

Edit `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Add an entry for your static IP + FQDN + short hostname:

```text
127.0.0.1       localhost
10.0.0.X        misp.vmcti01.lan vmcti01
```

Leave any other existing lines (like `::1`) as-is.

Verify DNS resolution **inside** the VM:

```bash
getent hosts misp.vmcti01.lan
hostname
hostname -f
```

Expected results:

- `hostname` → `vmcti01`
- `hostname -f` → `misp.vmcti01.lan`

### 3.2 Client-side DNS / browser behavior

From the **workstation** (laptop/desktop), `https://misp.vmcti01.lan/` will fail with `DNS_PROBE_FINISHED_NXDOMAIN` unless your homelab DNS knows about it.

Two options on the **client** side:

1. **Use the IP directly**
   - Visit `https://10.0.0.X/` in your browser
   - You'll get a certificate warning (self-signed) — accept/continue

2. **Add a DNS/hosts entry for `misp.vmcti01.lan`**
   - Option A: Add a record in your homelab DNS (Pi-hole, router, etc.):
     - `A misp.vmcti01.lan → 10.0.0.X`
   - Option B: Add a local hosts file entry **on your workstation**:

     ```text
     10.0.0.X    misp.vmcti01.lan
     ```

> **Gotcha:** The VM itself resolves `misp.vmcti01.lan` fine (due to `/etc/hosts`), but external browsers will fail with NXDOMAIN if you skip this step.

---

## 4 — MISP Installation

> Guardrail: Run all MISP setup as `root` or via `sudo` from your admin user.
> Guardrail: These steps assume a **fresh** Ubuntu 24.04 minimal server with no other webapps.

### 4.1 Install core dependencies

```bash
sudo apt update
sudo apt install -y \
    apache2 mariadb-server mariadb-client \
    redis-server \
    python3 python3-venv python3-pip \
    libapache2-mod-php php php-cli php-mysql php-redis php-xml php-mbstring php-zip php-gd php-curl \
    git gnupg make gcc g++ \
    supervisor
```

Enable and verify services:

```bash
sudo systemctl enable --now apache2
sudo systemctl enable --now mariadb
sudo systemctl enable --now redis-server
sudo systemctl enable --now supervisor
```

Quick status checks:

```bash
sudo systemctl status apache2
sudo systemctl status mariadb
sudo systemctl status redis-server
sudo systemctl status supervisor
```

### 4.2 Secure MariaDB (basic hardening)

```bash
sudo mysql_secure_installation
```

Recommended answers:

- Switch to unix_socket auth? → accept default (often `Y`)
- Change root password? → yes; set a strong password and write it down
- Remove anonymous users? → `Y`
- Disallow root login remotely? → `Y`
- Remove test database? → `Y`
- Reload privilege tables? → `Y`

### 4.3 Clone MISP into /var/www

```bash
sudo mkdir -p /var/www
sudo chown root:root /var/www

cd /var/www
sudo git clone https://github.com/MISP/MISP.git

cd /var/www/MISP
git status
git branch
```

The working directory for MISP should now be `/var/www/MISP`.

### 4.4 Run the official MISP installer for Ubuntu

The MISP repo includes installer scripts under `INSTALL/`.

```bash
cd /var/www/MISP
ls INSTALL
```

Run the installer wrapper script:

```bash
cd /var/www/MISP/INSTALL
sudo bash INSTALL.sh
```

Typical flow:

- Script starts and presents a menu of distributions / flavors
- Choose the entry for **Ubuntu 24.04** or the latest supported Ubuntu version
- The script will:
  - Install additional required packages
  - Create the MISP database and user in MariaDB
  - Configure Apache vhost for MISP
  - Set up PHP configuration
  - Install Python modules & venvs
  - Configure `supervisor` for MISP background workers
  - Set up a self-signed HTTPS cert (if not already present)

> **Note:** This step takes a while. On slower hardware (laptops in the homelab), some steps feel idle but are just compiling/installing. If it truly fails, scroll back to find the exact command that errored and re-run after fixing.

### 4.5 Key file locations after install

| Path | Purpose |
|------|---------|
| `/var/www/MISP` | MISP codebase |
| `/etc/apache2/sites-available/misp.conf` | Apache virtual host config |
| `/var/www/MISP/app/Config/config.php` | MISP app configuration |
| `/var/www/MISP/app/Config/database.php` | MISP database configuration |
| `/etc/supervisor/conf.d/misp-workers.conf` | Supervisor worker config |
| `/var/www/MISP/app/tmp/logs/` | MISP application logs |
| `/var/log/apache2/` | Apache/MISP vhost logs |

### 4.6 Enable the MISP Apache site & HTTPS

If the installer didn't already enable the site:

```bash
sudo a2ensite misp.conf
sudo a2enmod ssl rewrite headers
sudo systemctl reload apache2
```

Confirm Apache is listening on port 443:

```bash
sudo ss -tulpn | grep apache2
```

You should see a `:443` listener. At this point, `https://10.0.0.X/` (or your FQDN) should reach Apache.

---

## 5 — Services & Workers (Supervisor Model)

### 5.1 No `misp-workers.service` systemd unit

There is **no** `misp-workers.service` systemd unit after install.

```bash
sudo systemctl restart misp-workers
# Fails with: Unit misp-workers.service not found.
```

Background jobs (feed pulls, email, cache, scheduler) are managed by **Supervisor**, not systemd directly.

### 5.2 Managing Supervisor

Basic Supervisor checks:

```bash
sudo systemctl status supervisor
```

You should see `Active: active (running)`. If not:

```bash
sudo systemctl enable --now supervisor
```

Check the known programs:

```bash
sudo supervisorctl status
```

A healthy MISP worker list:

```text
misp-workers:cache       RUNNING   pid 1234, uptime 0:05:12
misp-workers:default     RUNNING   pid 1235, uptime 0:05:12
misp-workers:email       RUNNING   pid 1236, uptime 0:05:12
misp-workers:prio        RUNNING   pid 1237, uptime 0:05:12
misp-workers:scheduler   RUNNING   pid 1238, uptime 0:05:12
misp-workers:update      RUNNING   pid 1239, uptime 0:05:12
```

If Supervisor was just configured or you edited its conf:

```bash
sudo supervisorctl reread
sudo supervisorctl update
```

To restart all MISP workers:

```bash
sudo supervisorctl restart misp-workers:*
```

### 5.3 Ensuring workers start on reboot

Supervisor handles MISP worker lifecycle as long as supervisor itself is enabled at boot:

```bash
sudo systemctl is-enabled supervisor
```

If it is not:

```bash
sudo systemctl enable supervisor
```

After a reboot:

```bash
sudo reboot
# then...
sudo supervisorctl status
```

All `misp-workers:*` entries should be `RUNNING`.

---

## 6 — Validation

### 6.1 Confirm Apache & HTTPS

```bash
sudo systemctl status apache2
```

You want `Active: active (running)`. If needed:

```bash
sudo systemctl restart apache2
```

From a client machine, browse to either:

- `https://10.0.0.X/`
- `https://misp.vmcti01.lan/` (if DNS/hosts is configured)

You will see a browser warning about a self-signed certificate — continue anyway.

You should see the MISP login page.

### 6.2 First login as default admin

Default MISP admin credentials (unless changed by installer):

- Username (email): `admin@admin.test`
- Password: `admin`

**Immediately after login:**

- Go to the user profile
- Change the admin password to a strong, unique password
- Optionally change the admin email address

### 6.3 Check background workers from the UI

- Log in as admin
- Navigate to **Administration** → **Server Settings & Maintenance** → Workers tab
- Confirm workers (default, cache, email, prio, scheduler, update) all show as **running**

If some are stopped, from the CLI:

```bash
sudo supervisorctl status
sudo supervisorctl restart misp-workers:default
sudo supervisorctl restart misp-workers:scheduler
# Or restart all:
sudo supervisorctl restart misp-workers:*
```

### 6.4 Quick feed test (sanity only)

In MISP, go to **Server Settings & Maintenance** → **Diagnostics**. Confirm that basic checks (PHP, DB, Redis, workers) are green or mostly green.

### 6.5 CLI Validation Checklist

- [ ] Network / hostname resolves correctly
- [ ] Apache listening on 443 and 80
- [ ] MariaDB and Redis running
- [ ] Supervisor running with all `misp-workers:*` in RUNNING state
- [ ] MISP config files exist at `/var/www/MISP/app/Config/`
- [ ] Web UI accessible with MISP login page
- [ ] Logs are clean

```bash
# Network / hostname
ip addr show
hostname
hostname -f
getent hosts misp.vmcti01.lan

# Core services listening
sudo ss -tulpn | grep -E '443|80'
sudo systemctl status apache2
sudo systemctl status mariadb
sudo systemctl status redis-server

# MISP directory & permissions
ls -ld /var/www/MISP
ls /var/www/MISP/app/Config

# Supervisor & workers
sudo systemctl status supervisor
sudo supervisorctl status

# Logs (if issues)
sudo tail -n 100 /var/log/apache2/error.log
sudo tail -n 100 /var/www/MISP/app/tmp/logs/error.log
```

---

## 7 — Post-Install

- [ ] Change default admin credentials (`admin@admin.test` / `admin`)
- [ ] Configure DNS/hosts entry for `misp.vmcti01.lan` on workstations
- [ ] Replace self-signed cert with internal CA or Let's Encrypt (optional)
- [ ] Configure log forwarding to Graylog/Cribl
- [ ] Proceed to "MISP Initial Configuration" runbook for org/users/feeds setup

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `systemctl restart misp-workers` fails with "Unit not found" | MISP workers use Supervisor, not systemd | Use `sudo supervisorctl restart misp-workers:*` |
| Workers not running after reboot | Supervisor not enabled at boot | `sudo systemctl enable --now supervisor` |
| `DNS_PROBE_FINISHED_NXDOMAIN` in browser | Workstation doesn't know `misp.vmcti01.lan` | Add DNS A record or hosts entry on client; or use `https://10.0.0.X/` |
| Browser shows invalid certificate warning | Apache uses self-signed cert out-of-the-box | Accept the warning or replace with proper cert |
| MISP install script appears stuck | Compiling/installing on slow hardware | Wait; if truly failed, scroll back for the error and re-run after fixing |
| `hostname -f` doesn't return FQDN | `/etc/hosts` missing FQDN entry | Add `10.0.0.X misp.vmcti01.lan vmcti01` to `/etc/hosts` |
| Web UI loads but background tasks do nothing | Supervisor workers not running | `sudo supervisorctl status` then `restart misp-workers:*` |
| Apache not listening on 443 | Site/modules not enabled | `sudo a2ensite misp.conf && sudo a2enmod ssl rewrite headers && sudo systemctl reload apache2` |

---

## Quick Reference

```bash
# Check all MISP workers
sudo supervisorctl status

# Restart all MISP workers
sudo supervisorctl restart misp-workers:*

# Reload supervisor config after edits
sudo supervisorctl reread
sudo supervisorctl update

# Restart Apache
sudo systemctl restart apache2

# View MISP application logs
sudo tail -f /var/www/MISP/app/tmp/logs/error.log

# View Apache logs
sudo tail -f /var/log/apache2/error.log

# Check what's listening on HTTPS
sudo ss -tulpn | grep 443
```

---

## Quirks & Gotchas

- **No `misp-workers.service` unit:** `sudo systemctl restart misp-workers` fails with "Unit not found". All MISP workers are managed via **Supervisor** using `sudo supervisorctl status` / `restart`.
- **Supervisor must be enabled for workers to survive reboot:** MISP installer wires workers into Supervisor, but you still need `sudo systemctl enable --now supervisor`. Without this, workers won't run after reboot even though the web UI loads.
- **Browser FQDN vs. DNS (NXDOMAIN):** Inside `vmcti01`, `/etc/hosts` resolves `misp.vmcti01.lan` fine. On your workstation, it fails unless you add a DNS/hosts entry or use the IP directly.
- **Self-signed HTTPS certificate warnings:** Apache uses a self-signed cert; browsers will complain. Replace with internal CA or Let's Encrypt later.
- **Install script takes time & can look "stuck":** The MISP `INSTALL.sh` does a lot (packages, DB, virtual hosts, workers). On slower hardware some steps feel idle but are compiling/installing. If it truly fails, scroll back for the error.

---

*Last updated: 2025-XX-XX*
