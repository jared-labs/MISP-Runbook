# Deploy VMCTI01 — MISP Install & Base OS

Single-VM deployment of MISP on Ubuntu Server 24.04 **minimal**, running as `vmcti01` on Proxmox and reachable at `https://misp.vmcti01.lan/`.

> Guardrail: VMCTI01 is **only** for MISP and supporting services (MariaDB, Redis, web server, workers).
> Guardrail: All steps assume **Ubuntu Server 24.04 minimal** with a static IP on `vmbr0` in the 10.0.0.0/24 homelab range.

* * *

## Overview

This runbook covers:

- Proxmox VM creation for **VMCTI01**
- Ubuntu 24.04 minimal install + basic tooling
- Hostname / FQDN / `/etc/hosts` setup for `misp.vmcti01.lan`
- MISP installation via the official Ubuntu installer scripts
- Service model (Apache/HTTPD + Supervisor-managed `misp-workers`)
- Basic validation from a browser (HTTPS) and CLI checks
- Real-world quirks and gotchas encountered during first install

It **does not** cover:

- MISP UI configuration (org/users/feeds) — see “MISP Initial Configuration” runbook
- MISP → Cribl / Graylog enrichment — see “MISP → Cribl IOC Export & Enrichment” runbook

* * *

## Homelab Context

- Proxmox VE homelab cluster (“laptop stack”)
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

* * *

## VM Specs

### Recommended MISP sizing (small homelab)

For a small, single-user MISP in a homelab:

- vCPU: **4 vCPUs**
- RAM: **8 GB**
- Disk: **120 GB** (thin-provisioned; you can go larger if disk is cheap)
- NIC: 1 x virtio bridged to `vmbr0` (10.0.0.0/24)

You can run lighter (2 vCPUs / 4 GB RAM / 80 GB disk) for testing, but MISP will be happier with the above if you plan to:

- Enable multiple feeds
- Store larger event sets
- Run background workers reliably

### Proxmox CPU type

- CPU type: `host` (recommended)
  - Gives MISP / MariaDB / PHP the full feature set of the physical CPU.
- No special flags required (MISP doesn’t need AVX like some other apps).

* * *

## 0) Proxmox VM definition (before OS install)

Create the VM in the Proxmox UI:

- Node: your preferred Proxmox node
- VM ID: next available
- Name: `VMCTI01`

Disks / CPU / Memory:

- OS:
  - ISO: `ubuntu-24.04-live-server-amd64.iso` (or newer 24.04.x)
- System:
  - Machine: `q35` (or default)
  - BIOS: `OVMF (UEFI)` or `SeaBIOS` (either is fine — choose what you use elsewhere)
  - SCSI Controller: `VirtIO SCSI`
- Disks:
  - Bus/Device: `scsi0`
  - Size: `120G` (thin)
  - Cache: `Write back` (if you’re comfortable with battery-backed storage) or `Default`
- CPU:
  - Sockets: `1`
  - Cores: `4`
  - Type: `host`
- Memory:
  - `8192` MB RAM
  - Ballooning: enabled or disabled per your cluster standards
- Network:
  - Model: `VirtIO (paravirtualized)`
  - Bridge: `vmbr0`
  - VLAN tag: (set if you use VLANs, otherwise leave blank)

Attach the Ubuntu ISO and ensure the VM boots from CD first.

* * *

## 1) Ubuntu 24.04 minimal install on VMCTI01

Boot the VM and walk through the Ubuntu Server 24.04 installer:

- Language: `English` (or your preference)
- Keyboard: your layout
- Installation type: **Ubuntu Server (minimized)** or equivalent minimal profile
- Network:
  - Interface on `vmbr0` should be visible.
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

For additional snaps / server roles: skip (we’ll install manually later).

After install, reboot and log in via console or SSH:

- Console (Proxmox web UI) or:
  
    ssh admin@10.0.0.X

* * *

## 2) Base OS post-install setup

### 2.1 Update and upgrade

As your non-root user with sudo:

    sudo apt update
    sudo apt full-upgrade -y
    sudo reboot

Log back in after reboot.

### 2.2 Install basic tooling

Ubuntu minimal often omits some basics (like `nano`).

    sudo apt install -y \
        nano vim-tiny curl wget git htop unzip ca-certificates gnupg lsb-release

- `nano` / `vim-tiny`: quick editing
- `curl` / `wget`: grabbing scripts & debugging HTTP
- `git`: required for MISP repo
- `htop`: basic resource view

### 2.3 Set/verify hostname

Installer should already have set `vmcti01`, but confirm:

    hostnamectl

If needed, set explicitly:

    sudo hostnamectl set-hostname vmcti01

Log out and back in to see the updated prompt.

* * *

## 3) Hostname, FQDN & `/etc/hosts` quirks

MISP expects a proper FQDN; we want `misp.vmcti01.lan`.

### 3.1 Configure local FQDN on the VM

Edit `/etc/hosts`:

    sudo nano /etc/hosts

Add an entry for your static IP + FQDN + short hostname, similar to:

    127.0.0.1       localhost
    10.0.0.X        misp.vmcti01.lan vmcti01

Leave any other existing lines (like `::1`) as-is.

Verify DNS resolution **inside** the VM:

    getent hosts misp.vmcti01.lan
    hostname
    hostname -f

You want:

- `hostname` → `vmcti01`
- `hostname -f` → `misp.vmcti01.lan`

### 3.2 Client-side DNS / browser behavior

**Quirk we hit:** from the **workstation** (laptop/desktop), going to `https://misp.vmcti01.lan/` initially failed with:

- `DNS_PROBE_FINISHED_NXDOMAIN`

Because **nothing** in the homelab DNS knew about `misp.vmcti01.lan`.

You have two options on the **client** side:

1. **Use the IP directly**

   - Visit `https://10.0.0.X/` in your browser.
   - You’ll get a certificate warning (self-signed) — accept/continue.
   - The URL won’t show `misp.vmcti01.lan`, but it works.

2. **Add a DNS/hosts entry for `misp.vmcti01.lan`**

   - Option A: Add a record in your homelab DNS (Pi-hole, router, etc.):
     - `A misp.vmcti01.lan → 10.0.0.X`
   - Option B: Add a local hosts file entry **on your workstation**, e.g.:

       10.0.0.X    misp.vmcti01.lan

   - After this, `https://misp.vmcti01.lan/` should resolve & load (still with a cert warning).

> **Gotcha:** If you forget this step, the VM itself will resolve `misp.vmcti01.lan` just fine (due to `/etc/hosts`), but external browsers will fail with NXDOMAIN.

* * *

## 4) MISP Install on Ubuntu 24.04

This section uses the official MISP installation scripts for Ubuntu.

> Guardrail: Run all MISP setup as `root` or via `sudo` from your admin user.
> Guardrail: These steps assume a **fresh** Ubuntu 24.04 minimal server with no other webapps.

### 4.1 Install core dependencies

    sudo apt update
    sudo apt install -y \
        apache2 mariadb-server mariadb-client \
        redis-server \
        python3 python3-venv python3-pip \
        libapache2-mod-php php php-cli php-mysql php-redis php-xml php-mbstring php-zip php-gd php-curl \
        git gnupg make gcc g++ \
        supervisor

Enable/verify services:

    sudo systemctl enable --now apache2
    sudo systemctl enable --now mariadb
    sudo systemctl enable --now redis-server
    sudo systemctl enable --now supervisor

Quick status checks:

    sudo systemctl status apache2
    sudo systemctl status mariadb
    sudo systemctl status redis-server
    sudo systemctl status supervisor

### 4.2 Secure MariaDB (basic hardening)

    sudo mysql_secure_installation

Recommended answers:

- Switch to unix_socket auth? → accept default (often `Y`)
- Change root password? → yes; set a strong password and write it down
- Remove anonymous users? → `Y`
- Disallow root login remotely? → `Y`
- Remove test database? → `Y`
- Reload privilege tables? → `Y`

### 4.3 Clone MISP into /var/www

    sudo mkdir -p /var/www
    sudo chown root:root /var/www

    cd /var/www
    sudo git clone https://github.com/MISP/MISP.git

    cd /var/www/MISP
    git status
    git branch

The working directory for MISP should now be `/var/www/MISP`.

### 4.4 Run the official MISP installer for Ubuntu

The MISP repo includes installer scripts under `INSTALL/`.

From `/var/www/MISP`:

    cd /var/www/MISP
    ls INSTALL

You should see various install scripts, including those for Debian/Ubuntu.

Run the installer wrapper script and choose the Ubuntu profile:

    cd /var/www/MISP/INSTALL
    sudo bash INSTALL.sh

Typical flow:

- Script starts and presents a menu of distributions / flavors.
- Choose the entry for **Ubuntu 24.04** or the latest supported Ubuntu version.
- The script will:
  - Install additional required packages
  - Create the MISP database and user in MariaDB
  - Configure Apache vhost for MISP
  - Set up PHP configuration
  - Install Python modules & venvs
  - Configure `supervisor` for MISP background workers
  - Set up a self-signed HTTPS cert (if not already present)

**This step takes a while.** Watch for errors; if something fails, scroll up and re-run after fixing the issue (common causes: network hiccups, mirror issues).

### 4.5 Where things end up on disk

After a successful `INSTALL.sh` run, key locations should be:

- MISP codebase:

    /var/www/MISP

- Apache virtual host config (name may vary):

    /etc/apache2/sites-available/misp.conf

- MISP config files (app & database):

    /var/www/MISP/app/Config/config.php
    /var/www/MISP/app/Config/database.php

- MISP worker supervisor config (name may vary):

    /etc/supervisor/conf.d/misp-workers.conf

- Logs:

    /var/www/MISP/app/tmp/logs/
    /var/log/apache2/ (MISP vhost logs)

### 4.6 Enable the MISP Apache site & HTTPS

If the installer didn’t already enable the site:

    sudo a2ensite misp.conf
    sudo a2enmod ssl rewrite headers
    sudo systemctl reload apache2

Confirm Apache is listening on port 443:

    sudo ss -tulpn | grep apache2

You should see a `:443` listener.

At this point, `https://10.0.0.X/` (or your FQDN) should reach Apache.

* * *

## 5) Services & Workers (Supervisor model)

### 5.1 No `misp-workers.service` systemd unit

**Important quirk:** There is **no** `misp-workers.service` systemd unit after install.

Attempts like:

    sudo systemctl restart misp-workers

Will fail with:

    Failed to restart misp-workers.service: Unit misp-workers.service not found.

Background jobs (e.g., feed pulls, email, cache, scheduler) are managed by **Supervisor**, not systemd directly.

### 5.2 Managing Supervisor

Basic Supervisor checks:

    sudo systemctl status supervisor

You should see:

    Active: active (running)

If it’s not running:

    sudo systemctl enable --now supervisor

Check the known programs:

    sudo supervisorctl status

A healthy MISP worker list typically includes items like:

    misp-workers:cache       RUNNING   pid 1234, uptime 0:05:12
    misp-workers:default     RUNNING   pid 1235, uptime 0:05:12
    misp-workers:email       RUNNING   pid 1236, uptime 0:05:12
    misp-workers:prio        RUNNING   pid 1237, uptime 0:05:12
    misp-workers:scheduler   RUNNING   pid 1238, uptime 0:05:12
    misp-workers:update      RUNNING   pid 1239, uptime 0:05:12

If Supervisor was just configured or you edited its conf:

    sudo supervisorctl reread
    sudo supervisorctl update

To restart all MISP workers:

    sudo supervisorctl restart misp-workers:*

### 5.3 Ensuring workers start on reboot

Supervisor handles MISP worker lifecycle as long as supervisor itself is enabled at boot:

    sudo systemctl is-enabled supervisor

If it is not:

    sudo systemctl enable supervisor

After a reboot:

    sudo reboot

Then:

    sudo supervisorctl status

All `misp-workers:*` entries should be `RUNNING`.

> **Gotcha:** If you forget to enable `supervisor`, MISP may load in the browser but background tasks (feeds, emails, scheduled jobs) will silently do nothing until you manually start `supervisor`.

* * *

## 6) Basic Web UI & Health Checks

### 6.1 Confirm Apache & HTTPS

Check Apache service:

    sudo systemctl status apache2

You want:

- `Active: active (running)`

If needed:

    sudo systemctl restart apache2

From a client machine, browse to either:

- `https://10.0.0.X/`
- `https://misp.vmcti01.lan/` (if DNS/hosts is configured)

You will likely see:

- A browser warning about an invalid / self-signed certificate.
- Continue anyway (advanced → accept risk / proceed).

You should see the MISP login page.

### 6.2 First login as default admin

Default MISP admin credentials (unless changed by installer):

- Username (email): `admin@admin.test`
- Password: `admin`

Log in, then **immediately**:

- Go to the user profile.
- Change the admin password to a strong, unique password.
- Optionally change the admin email address.

### 6.3 Check background workers from the UI

Within MISP:

- Log in as admin.
- Navigate to **Administration** → **Server Settings & Maintenance** → Workers tab (name may vary).
- Confirm workers (default, cache, email, prio, scheduler, update) all show as **running**.

If some are stopped:

- On the CLI, run:

    sudo supervisorctl status

- Restart as needed:

    sudo supervisorctl restart misp-workers:default
    sudo supervisorctl restart misp-workers:scheduler

Or restart them all:

    sudo supervisorctl restart misp-workers:*

### 6.4 Quick feed test (sanity only)

You don’t need full feed configuration here (that’s for the Initial Configuration runbook), but you can quickly verify that:

- MISP can reach the internet (for feeds / updates).
- Workers process tasks.

Example quick check:

- In MISP, go to **Server Settings & Maintenance** → **Diagnostics**.
- Confirm that basic checks (PHP, DB, Redis, workers) are green or mostly green.

* * *

## 7) Validation Checklist (CLI)

After installation, run through this quick CLI checklist on `vmcti01`:

1. **Network / hostname**

    ip addr show
    hostname
    hostname -f
    getent hosts misp.vmcti01.lan

2. **Core services listening**

    sudo ss -tulpn | grep -E '443|80'
    sudo systemctl status apache2
    sudo systemctl status mariadb
    sudo systemctl status redis-server

3. **MISP directory & permissions**

    ls -ld /var/www/MISP
    ls /var/www/MISP/app/Config

   Confirm `config.php` and `database.php` exist (exact names/paths may vary slightly but should match the MISP docs).

4. **Supervisor & workers**

    sudo systemctl status supervisor
    sudo supervisorctl status

   Verify all `misp-workers:*` entries show `RUNNING`.

5. **Logs (if issues)**

    sudo tail -n 100 /var/log/apache2/error.log
    sudo tail -n 100 /var/www/MISP/app/tmp/logs/error.log

* * *

## 8) Quirks & Gotchas

Explicitly capturing the weirdness and pitfalls from initial install:

- **No `misp-workers.service` unit**
  - There is **no** `misp-workers.service` systemd unit.
  - `sudo systemctl restart misp-workers` fails with `Unit not found`.
  - All MISP workers are managed via **Supervisor**, using `sudo supervisorctl status` / `restart`.

- **Supervisor must be enabled for workers to survive reboot**
  - MISP installer wires workers into Supervisor, but you still need:
    
        sudo systemctl enable --now supervisor

  - Without this, workers won’t run after reboot even though the web UI loads.

- **Browser FQDN vs. DNS (NXDOMAIN)**
  - Inside `vmcti01`, `/etc/hosts` makes `misp.vmcti01.lan` resolve fine.
  - On your workstation, `https://misp.vmcti01.lan/` initially failed with `DNS_PROBE_FINISHED_NXDOMAIN`.
  - Fix:
    - Use `https://10.0.0.X/` **or**
    - Add DNS/hosts entry: `misp.vmcti01.lan → 10.0.0.X`.

- **Self-signed HTTPS certificate warnings**
  - Apache uses a self-signed cert out-of-the-box.
  - Browsers complain (invalid certificate) — you must explicitly proceed.
  - You can later replace this with a proper cert (internal CA, Let’s Encrypt, etc.)

- **Install script takes time & can look “stuck”**
  - The MISP `INSTALL.sh` for Ubuntu does a lot:
    - Packages, DB, virtual hosts, workers, etc.
  - On slower hardware (laptops in the homelab), some steps feel idle but are just compiling / installing.
  - If it truly fails, scroll back to find the exact command that errored and re-run after fixing (e.g., apt mirror issues).

With these quirks handled, `VMCTI01` should provide a stable, HTTPS-accessible MISP instance ready for initial configuration and later integration with Cribl / Graylog.
