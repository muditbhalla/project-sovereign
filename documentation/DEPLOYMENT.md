# PROJECT SOVEREIGN — FULL DEPLOYMENT GUIDE

> **6-Phase Bare-Metal Provisioning Walkthrough**
> From USB ISO flash to a live, operational cryptographic mesh network.
> Execute every command in exact sequence.

| Field | Value |
|-------|-------|
| **Target Host** | Apple MacBook Pro (Mid-2012) — bare-metal Ubuntu Server |
| **Host OS** | Ubuntu Server 26.04 LTS |
| **Architecture** | x86\_64 |
| **Docker Engine** | Docker CE (official repository install) |
| **Mesh Software** | Headscale `v0.23.0-alpha5` (containerized) |

---

## Table of Contents

- [Phase 1 — Bare-Metal Environment Provisioning](#phase-1--bare-metal-environment-provisioning)
- [Phase 2 — Core OS Dependencies & Docker Engine](#phase-2--core-os-dependencies--docker-engine)
- [Phase 3 — Repository Layout & Security Hardening](#phase-3--repository-layout--security-hardening)
- [Phase 4 — Overlay Mesh Network Configuration](#phase-4--overlay-mesh-network-configuration)
- [Phase 5 — Secure Network Convergence & Client Registration](#phase-5--secure-network-convergence--client-registration)
- [Phase 6 — Git Remote Management & First Push](#phase-6--git-remote-management--first-push)

---

## Phase 1 — Bare-Metal Environment Provisioning

### Step 1.1 — Create the Bootable USB Installer

*Executed on: external macOS machine*

```bash
# List all connected storage drives to identify the USB device
diskutil list

# Unmount the target USB drive (replace X with the correct disk number)
diskutil unmountDisk /dev/diskX

# Flash the Ubuntu Server LTS ISO directly onto the USB device
# ⚠️ Double-check /dev/rdiskX — this will permanently overwrite that drive
sudo dd if=ubuntu-server.iso of=/dev/rdiskX bs=1m status=progress
```

> Eject the USB, insert it into the MacBook Pro, and boot while holding **Option** key to select the USB boot volume.

---

### Step 1.2 — Post-Installation Lid & Sleep Configuration

*Executed on: MacBook Pro server — first boot into Ubuntu*

The MacBook's default ACPI configuration will suspend the OS when the lid is closed. For a headless server, this must be permanently disabled.

```bash
# Edit the systemd login daemon configuration file
sudo nano /etc/systemd/logind.conf
```

Add or modify the following three lines inside `logind.conf`:

```ini
# Prevent OS from sleeping when lid is closed in any power state
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

Apply the change without rebooting:

```bash
# Restart the login daemon to apply the new hardware profile immediately
sudo systemctl restart systemd-logind.service
```

> **Result:** The server will now remain fully active with the lid partially closed or at a 15-degree angle for passive airflow.

---

## Phase 2 — Core OS Dependencies & Docker Engine

### Step 2.1 — Update and Harden the Host OS

```bash
# Synchronize package repository indexes and upgrade all installed packages
sudo apt update && sudo apt upgrade -y

# Install core dependencies
sudo apt install -y curl git secure-delete iptables build-essential
```

---

### Step 2.2 — Install Docker CE from the Official Repository

> Do **not** use the `apt` default `docker.io` package — it is outdated. Use the official Docker repository.

```bash
# Step A — Add Docker's official GPG signing key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Step B — Register the official Docker apt repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Step C — Install Docker CE, CLI, containerd, and Compose plugin
sudo apt update
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

---

### Step 2.3 — Fix Docker Socket Permissions (Resolves Incident C2)

> Without this step, all `docker` commands require `sudo`.
> This adds the active user to the `docker` group for passwordless Docker access.

```bash
# Create the docker group if it doesn't already exist
sudo groupadd docker

# Add the active non-root user to the docker management group
sudo usermod -aG docker $USER

# Apply the new group membership in the current shell session without logging out
newgrp docker

# Verify — this should succeed without sudo
docker ps
```

---

## Phase 3 — Repository Layout & Security Hardening

### Step 3.1 — Create the Deployment Directory Tree

```bash
# Create all required service subdirectories in one command
mkdir -p ~/sovereign/{headscale/config,headscale/data,seafile/db,seafile/data,immich/db,immich/library,immich/model-cache}

# Navigate to the project root
cd ~/sovereign

# Initialize a local Git repository
git init
```

---

### Step 3.2 — Security Boundary (`.gitignore`)

Create the `.gitignore` file in the repository root:

```bash
nano ~/sovereign/.gitignore
```

Paste the following content:

```gitignore
# ==============================================================================
# PROJECT SOVEREIGN — CRYPTOGRAPHIC DEFENSE BOUNDARY
# ==============================================================================

# 1. GLOBAL CRYPTOGRAPHIC KEY PROTECTION
**/*.key
**/*.pem
**/*.crt
**/*.csr
**/*.p12
**/*.pub
**/id_rsa*

# Headscale-specific control plane keys (explicit paths)
config/headscale/private_key.pem
config/headscale/noise_private_key.pem

# 2. LOCAL DATABASE & RUNTIME STATE ISOLATION
config/headscale/db.sqlite
config/headscale/db.sqlite-journal
data/
sovereign/
postgres/
seafile/
immich/

# 3. ENVIRONMENT VARIABLES & SECRETS
.env
.env.*
!.env.example

# 4. OPERATIONAL LOGS
*.log
logs/
**/logs/
```

---

### Step 3.3 — Write the Master Deployment Manifest (`docker-compose.yml`)

```bash
nano ~/sovereign/docker-compose.yml
```

See [`docker-compose.yml`](../docker-compose.yml) in the repository root for the full contents.

---

## Phase 4 — Overlay Mesh Network Configuration

### Step 4.1 — Create Configuration Files

```bash
# Create the empty ACL policy file
touch ~/sovereign/headscale/config/acls.json

# Open the main control plane configuration
nano ~/sovereign/headscale/config/config.yaml
```

See [`config/headscale/config.yaml`](../config/headscale/config.yaml) for the full configuration to paste.

---

### Step 4.2 — Start the Full Infrastructure Stack

```bash
# Navigate to the project root containing docker-compose.yml
cd ~/sovereign

# Start all services in detached (background) mode
docker compose up -d

# Verify all containers are running and healthy
docker compose ps
```

Expected output — all services should show `running`:

```
NAME                      IMAGE                                   STATUS
sovereign-headscale       headscale/headscale:latest              running
sovereign-seafile-db      postgres:15                             running
sovereign-memcached       memcached:1.6                           running
sovereign-seafile-app     seafileltd/seafile-mc:latest            running
sovereign-immich-db       tensorchord/pgvecto-rs:pg14-v0.2.0     running
sovereign-immich-redis    redis:6.2-alpine                        running
sovereign-immich-server   ghcr.io/immich-app/immich-server:release    running
sovereign-immich-ml       ghcr.io/immich-app/immich-machine-learning:release  running
```

---

## Phase 5 — Secure Network Convergence & Client Registration

### Step 5.1 — Provision a User Namespace on the Server

*Executed on: Linux Server Terminal (`muditdatabase`)*

```bash
# Create the administration tenant namespace
docker exec -it sovereign-headscale headscale users create admin-mb

# Confirm it was created
docker exec -it sovereign-headscale headscale users list
```

---

### Step 5.2 — Purge macOS Client State

*Executed on: MacBook Air M4 Terminal*

```bash
# Terminate any dead or conflicting commercial session variables
# (Resolves Incidents B4 — stdout interception, B5 — multi-profile lockout)
/Applications/Tailscale.app/Contents/MacOS/Tailscale logout

# Reset cached state and initialize against the server
# (Resolves Incidents B1 — GUI freeze, B2 — cache conflict, B3 — force-reauth)
/Applications/Tailscale.app/Contents/MacOS/Tailscale up \
  --login-server=http://192.168.1.50:8080 \
  --force-reauth \
  --reset
```

> A browser window will open with a registration URL.
> Copy the `nodekey:` hex string from the URL bar.

---

### Step 5.3 — Cryptographic Handshake Injection

*Executed on: Linux Server Terminal (`muditdatabase`)*

> ⚠️ **Critical:** Change the prefix `nodekey:` → `mkey:` before running.
> (Resolves Incident D1 — schema drift between Tailscale client and Headscale alpha build)

```bash
# Register the client into the private mesh
# ⚠️ Replace nodekey: with mkey: in the key string
docker exec -it sovereign-headscale headscale nodes register \
  --user admin-mb \
  --key mkey:818b2f57e96289b1d82bc7ffef8f59800cef4597d9b3aa4f3eea3bdd81a7867
```

---

### Step 5.4 — Convergence Validation Audit

*Executed on: Linux Server Terminal (`muditdatabase`)*

```bash
# Verify the cluster topology — both nodes should show Online
docker exec -it sovereign-headscale headscale nodes list
```

> Switch to the `admin-mb` profile in the macOS Tailscale menu bar app.
> Wait 5–15 seconds for asynchronous daemon convergence (Incident B6 — normal behavior).

---

## Phase 6 — Git Remote Management & First Push

### Step 6.1 — Stage and Commit All Files

```bash
# Navigate to repository root
cd ~/sovereign

# Stage all Infrastructure as Code files
git add .

# Create the initial commit
git commit -m "feat: initial commit of Project Sovereign IaC and documentation"
```

---

### Step 6.2 — Push to GitHub

```bash
# Set the primary branch to 'main'
git branch -M main

# Register your GitHub repository as the remote origin
# Replace with your actual GitHub username
git remote add origin https://github.com/muditbhalla/project-sovereign.git

# Push to GitHub using your Personal Access Token (PAT)
# When prompted for password, enter your PAT — not your account password
git push -u origin main
```

> Generate a Personal Access Token at: **GitHub → Settings → Developer Settings → Personal Access Tokens → Tokens (classic)** — scope: `repo`

---

## Verification Checklist

After completing all 6 phases, confirm the following:

- [ ] `docker compose ps` — all 8 containers show `running`
- [ ] `headscale nodes list` — both nodes show `Online`
- [ ] SSH to `100.64.0.3` from the MacBook Air succeeds
- [ ] Seafile accessible at `http://100.64.0.2:8000`
- [ ] Immich accessible at `http://100.64.0.2:2283`
- [ ] GitHub repository shows all files committed
- [ ] `.gitignore` is preventing `*.key`, `*.pem`, `.env` from being tracked

---

*Project Sovereign — Deployment Guide | Mudit Bhalla | May 2026*
