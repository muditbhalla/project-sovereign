# 📋 PROJECT SOVEREIGN — COMPLETE TROUBLESHOOTING & INCIDENT LOGBOOK

> **Full Lifecycle Audit:** Containerized Headscale Orchestration Plane Integration
> This document records every error, root cause, and verified resolution encountered across the
> complete deployment lifecycle of Project Sovereign.

| Field | Value |
|-------|-------|
| **Log Date** | May 29, 2026 |
| **System Operator** | `muditbhalla2008` |
| **Client Node** | MacBook Air M4 — `mb-macbook-air` |
| **Server Node** | Linux Database Host — `muditdatabase` |
| **Client Software** | Tailscale macOS App |
| **Server Software** | Headscale `v0.23.0-alpha5` (Docker: `sovereign-headscale`) |
| **Final Status** | ✅ **FULLY OPERATIONAL — VERIFIED MESH** |

---

## 📑 Table of Contents

- [Part A — Architectural & Hardware Defects](#part-a--architectural--hardware-defects)
- [Part B — macOS Client / Tailscale Incidents](#part-b--macos-client--tailscale-incidents)
- [Part C — Linux Server / Docker Incidents](#part-c--linux-server--docker-incidents)
- [Part D — Cryptographic & Schema Incidents](#part-d--cryptographic--schema-incidents)
- [Part E — Remote Access & Connectivity Incidents](#part-e--remote-access--connectivity-incidents)
- [Quick Error Reference Table](#quick-error-reference-table)

---

## The Core Problem: Architecture Deadlock Paradox

Before any incident log, understanding the **root architectural problem** is essential:

```
┌─────────────────────────────────────────────────────────────┐
│              COORDINATION DEADLOCK PARADOX                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Client needs to reach  →  http://100.64.0.3:8080          │
│  to build the tunnel    →  which lives INSIDE the tunnel    │
│                                                             │
│  ❌ Result: Infinite handshake loop → GUI freeze → timeout  │
│                                                             │
│  ✅ Fix: Use the REAL LAN IP (192.168.1.50:8080) to         │
│          bootstrap the tunnel, then use 100.64.x.x         │
└─────────────────────────────────────────────────────────────┘
```

---

## Part A — Architectural & Hardware Defects

These issues were identified and resolved at the infrastructure design and physical hardware layer.

| # | Defect | Layer | Root Cause | Resolution |
|---|--------|-------|-----------|------------|
| **A1** | **Embedded DERP Server Crash** | Network Control Plane | `config.yaml` had `derp.server.enabled: true` while operating on raw HTTP. Headscale requires TLS for its embedded relay. | Set `enabled: false`. Headscale falls back gracefully to public STUN relay infrastructure. |
| **A2** | **Data Persistence Deletion Vulnerability** | Application Storage Tier | Stateful containers without explicit volume mappings write to ephemeral storage — all data lost on restart. | Declared explicit bind-mount volumes on every stateful container (PostgreSQL, Headscale DB, Immich library). |
| **A3** | **Global Database Discovery Risk** | System Firewall | Default `-p 5432:5432` binding exposed databases on all interfaces (`0.0.0.0`), including WAN — visible to Shodan/Censys scanners. | Refactored all port bindings to prefix with the host mesh IP: `-p 100.64.0.2:5432:5432`. |
| **A4** | **Battery Physical Degradation Hazard** | Hardware Layer | Li-ion battery held at 100% charge under 24/7 thermal load undergoes accelerated electrochemical degradation and dangerous swelling. | Physically removed the battery assembly. Unit runs exclusively on MagSafe as a dedicated headless appliance. |
| **A5** | **Clamshell Core Thermal Throttling** | Hardware Layer | Closing the laptop lid during local ML tasks traps heat, triggering kernel-level CPU frequency scaling (throttling) and performance degradation. | Unit configured to operate vertically with lid partially open, leveraging passive convective airflow. Override also applied via `logind.conf`. |
| **A6** | **Host OS Disconnection on Remote Networks** | Distributed Overlay Plane | Dockerizing Headscale gives the *container* a mesh IP, but the *Host OS itself* was not enrolled — no SSH-accessible IP existed outside LAN. | Installed `tailscaled` daemon directly on bare-metal Ubuntu. Host now has its own mesh IP (`100.64.0.3`) for out-of-band remote management. |

---

## Part B — macOS Client / Tailscale Incidents

### Incident B1 — Authentication Loop & GUI Freezing State

| | |
|---|---|
| **Symptom** | Triggering login caused a stalled browser window and complete Tailscale GUI freeze |
| **Error** | *(No terminal output — visual lockup only)* |
| **Root Cause** | Coordination server endpoint was set to `http://100.64.0.3` — an internal overlay address. Classic bootstrap deadlock: the client cannot contact the control plane to build the WireGuard tunnel because the destination address **lives inside the tunnel being built**. |
| **Fix** | Bypassed the GUI entirely. Used the Tailscale binary directly from Terminal, targeting the **real LAN IP** — not the overlay IP. |

```bash
# ✅ Resolution — bypass GUI, point to actual LAN IP
/Applications/Tailscale.app/Contents/MacOS/Tailscale up \
  --login-server=http://192.168.1.50:8080 \
  --force-reauth \
  --reset
```

---

### Incident B2 — Persistent Cache & Argument Settings Conflict

| | |
|---|---|
| **Error Output** | `Error: changing settings via 'tailscale up' requires mentioning all non-default flags` |
| **Root Cause** | Tailscale caches its full parameter state across sessions. Attempting to change `--login-server` without satisfying all previously cached non-default flags violates internal state validation — the engine refuses the command. |
| **Fix** | Appended `--reset` flag to explicitly purge cached parameter state before reinitializing. |

```bash
# ✅ Resolution — --reset clears all cached non-default parameter state
/Applications/Tailscale.app/Contents/MacOS/Tailscale up \
  --login-server=http://192.168.1.50:8080 \
  --force-reauth \
  --reset
```

---

### Incident B3 — Cross-Domain Authority Change Enforcement Lockout

| | |
|---|---|
| **Error Output** | `can't change --login-server without --force-reauth` |
| **Root Cause** | Tailscale's multi-tenant security model prevents switching the active coordination server (e.g., from `controlplane.tailscale.com` to a private self-hosted instance) while an active session lease exists on the original domain. |
| **Fix** | Appended `--force-reauth` to explicitly revoke the existing session before switching servers. |

```bash
# ✅ Resolution — --force-reauth strips the existing session before switching servers
/Applications/Tailscale.app/Contents/MacOS/Tailscale up \
  --login-server=http://192.168.1.50:8080 \
  --force-reauth \
  --reset
```

> **Note:** Incidents B2 and B3 are both resolved by the same combined command above. All three flags are required together.

---

### Incident B4 — Stdout Interception by macOS Network Extension Daemon

| | |
|---|---|
| **Symptom** | Terminal command completed with exit code `0` but printed **no browser URL** / no registration token in stdout |
| **Root Cause** | The background `tailscaled` system extension daemon was deadlocked against macOS Network Extensions. Registration tokens (the browser URL containing the `nodekey:` string) were silently swallowed into the macOS system logger instead of being piped to the terminal stdout stream. |
| **Fix** | Forced a clean logout to break the daemon loop state and unlock the stdout pipe, then re-ran the `up` command. |

```bash
# ✅ Resolution — full logout breaks the extension deadlock and restores stdout
/Applications/Tailscale.app/Contents/MacOS/Tailscale logout

# Then re-run Phase 1 initialization
/Applications/Tailscale.app/Contents/MacOS/Tailscale up \
  --login-server=http://192.168.1.50:8080 \
  --force-reauth \
  --reset
```

---

### Incident B5 — Multi-Profile Synchronization Disconnect ("Offline" Node)

| | |
|---|---|
| **Symptom** | `headscale nodes list` showed node registration as **successful** in the server database, but status reported as `Offline` |
| **Root Cause** | The macOS Tailscale app maintained an active authenticated session under the primary commercial profile (`muditbhalla@outlook.com`). The newly registered `admin-mb` private profile was cached but functionally suspended in a background sleep pool — the daemon was not processing packets for it. |
| **Fix** | Manually switched the active profile context inside the **macOS Tailscale menu bar app**: `muditbhalla@outlook.com` → `admin-mb`. This woke the daemon to accept parameters from the Headscale control plane. |

```
macOS Menu Bar → Tailscale icon → Switch Account → admin-mb
```

---

### Incident B6 — Asynchronous Daemon Convergence Propagation Delay

| | |
|---|---|
| **Symptom** | Brief delay (5–15 seconds) after profile switch before `headscale nodes list` showed `Online` status |
| **Root Cause** | After a profile context switch, the `tailscaled` background routing service undergoes a multi-second asynchronous delay while re-mapping low-level virtual adapter rules at the kernel level. This is **expected, normal behavior** — not an error. |
| **Fix** | No intervention required. Allowed background polling threads to execute undisturbed. All systems converged automatically. |

---

## Part C — Linux Server / Docker Incidents

### Incident C1 — Host Binary Execution Failure (`command not found`)

| | |
|---|---|
| **Symptom** | Typing `headscale` directly in the Ubuntu shell returned `bash: headscale: command not found` |
| **Root Cause** | Headscale is deployed **entirely inside a Docker container** — it has no installation path on the host OS. The binary exists only within the container's isolated application layer. Calling it on the host searches `$PATH` and finds nothing. |
| **Fix** | All Headscale management commands must be routed through the Docker container exec interface. |

```bash
# ❌ WRONG — runs on host OS path (binary not installed there)
headscale users list

# ✅ CORRECT — routes the command through the container's runtime
sudo docker exec -it sovereign-headscale headscale users list
```

---

### Incident C2 — Docker API UNIX Socket Permission Denial

| | |
|---|---|
| **Error Output** | `permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock` |
| **Root Cause** | The active login session (`muditbhalla2008`) was not a member of the `docker` UNIX group. The Docker socket at `/var/run/docker.sock` requires group membership for non-root write access. |
| **Immediate Fix** | Prefix all Docker commands with `sudo`. |
| **Permanent Fix** | Add the user to the `docker` group (see `DEPLOYMENT.md` Phase 2, Step 3). |

```bash
# ✅ Immediate fix — elevate with sudo
sudo docker exec -it sovereign-headscale headscale users list

# ✅ Permanent fix — add user to docker group (requires re-login or newgrp)
sudo usermod -aG docker $USER
newgrp docker
```

---

### Incident C3 — Unidentified Target Container Runtime Context

| | |
|---|---|
| **Symptom** | Unable to construct `docker exec` commands — the precise container name string was unknown |
| **Root Cause** | Multiple containers were running simultaneously. Without knowing the exact container name (`sovereign-headscale`), `exec` commands could not be targeted correctly. |
| **Fix** | Query the Docker daemon for all running process instances to identify the correct container name. |

```bash
# ✅ Resolution — list all running containers and their names
sudo docker ps

# Look under the NAMES column for: sovereign-headscale
# Then use that exact string in exec commands:
sudo docker exec -it sovereign-headscale headscale users list
```

---

### Incident C4 — Unknown Headscale Namespace / User Not Found

| | |
|---|---|
| **Error Output** | `Cannot create Pre Auth Key: rpc error: code = Unknown desc = user not found` |
| **Root Cause** | Headscale manages **independent network sandbox namespaces** completely separate from Ubuntu OS user accounts. A casing mismatch (e.g., `Admin-mb` vs `admin-mb`) or using an incorrect user name causes a database lookup failure — Headscale is case-sensitive. |
| **Fix** | Query the user namespace directory from within the container to get the exact, correct string. |

```bash
# ✅ Resolution — retrieve the exact namespace name from the container database
sudo docker exec -it sovereign-headscale headscale users list

# Use the EXACT Name value from the output (e.g., admin-mb) in all subsequent commands
```

---

### Incident C5 — Shell Interpreter Stream Redirection Exception (Angle Brackets)

| | |
|---|---|
| **Error Output** | `-bash: admin-mb: No such file or directory` |
| **Root Cause** | Documentation-style placeholder angle brackets (`<user>`, `<key>`) were copied literally into the terminal. The Bash interpreter evaluated `<` and `>` as **file I/O redirection operators**, not text substitution placeholders. `<admin-mb>` was interpreted as "read input from a file called `admin-mb`." |
| **Fix** | Re-run with actual values substituted in place of the `<placeholder>` syntax. Never paste bracketed placeholders directly into a terminal. |

```bash
# ❌ WRONG — bash reads <admin-mb> as a file redirect
sudo docker exec -it sovereign-headscale headscale nodes register \
  --user <admin-mb> \
  --key <mkey:...>

# ✅ CORRECT — actual values substituted directly
sudo docker exec -it sovereign-headscale headscale nodes register \
  --user admin-mb \
  --key mkey:818b2f57e96289b1d82bc7ffef8f59800cef4597d9b3aa4f3eea3bdd81a7867
```

---

## Part D — Cryptographic & Schema Incidents

### Incident D1 — Cryptographic Machine Key Format Mismatch / Schema Drift

> **This is the most critical and non-obvious error in the entire deployment.**

| | |
|---|---|
| **Error Output** | `Cannot register node: key hex string doesn't have expected type prefix mkey` |
| **Root Cause** | A **build version schema drift** exists between the modern Tailscale macOS client and the deployed alpha container (`v0.23.0-alpha5`). The macOS client generates registration tokens prefixed with the **legacy** `nodekey:` identifier. The Headscale `v0.23.0-alpha5` backend **strictly enforces** the modern WireGuard Machine Key format requiring an `mkey:` prefix. The hex payload itself is identical — only the prefix differs. |
| **Fix** | Intercept the browser registration token. Manually substitute `nodekey:` → `mkey:` before executing the server registration command. |

```bash
# ❌ Token AS GENERATED by the macOS client browser screen:
nodekey:818b2f57e96289b1d82bc7ffef8f59800cef4597d9b3aa4f3eea3bdd81a7867
#  ↑ This prefix causes the "doesn't have expected type prefix mkey" error

# ✅ Corrected token for Headscale v0.23.0-alpha5 — change ONLY the prefix:
mkey:818b2f57e96289b1d82bc7ffef8f59800cef4597d9b3aa4f3eea3bdd81a7867
#  ↑ Only the prefix changes — the 64-char hex payload is identical
```

```bash
# ✅ Full corrected registration command:
sudo docker exec -it sovereign-headscale headscale nodes register \
  --user admin-mb \
  --key mkey:818b2f57e96289b1d82bc7ffef8f59800cef4597d9b3aa4f3eea3bdd81a7867
```

> **Future note:** When upgrading Headscale beyond `v0.23.0-alpha5`, verify whether client-generated tokens still require this manual prefix substitution. Newer stable releases may auto-negotiate the schema.

---

## Part E — Remote Access & Connectivity Incidents

### Incident E1 — `tailscaled` Service Not Running on Linux Host

| | |
|---|---|
| **Error Output** | `failed to connect to local tailscaled... systemd tailscaled.service not running` `Error: dial unix /var/run/tailscale/tailscaled.sock: connect: no such file or directory` |
| **Root Cause** | The local terminal client executed `tailscale` commands while the underlying background service daemon (`tailscaled`) was inactive or had crashed. Without the daemon, no socket exists to communicate with — all `tailscale` commands fail immediately. |
| **Fix** | Enable and start the `tailscaled` systemd service on the host. |

```bash
# ✅ Resolution — enable daemon to start on boot and start it immediately
sudo systemctl enable tailscaled
sudo systemctl start tailscaled

# Verify it's running
sudo systemctl status tailscaled
```

---

### Incident E2 — SSH Connection Timeout (Overlay IP Before Tunnel Established)

| | |
|---|---|
| **Error Output** | `ssh: connect to host 100.64.0.3 port 22: Connection timed out` |
| **Root Cause** | Attempting to SSH to a `100.64.x.x` mesh overlay IP before the WireGuard tunnel is established. Internal mesh addresses are only reachable **from inside an active tunnel** — outside nodes have no route to the `100.64.0.0/10` space until the tunnel exists. This is the same bootstrap deadlock as Incident B1, applied to SSH. |
| **Fix** | Join the mesh first via a public-facing address (real LAN IP on the local network, or a DDNS domain from WAN). Only after the tunnel is active can mesh IPs be reached. |

```bash
# ✅ Step 1 — Join via public address to establish the tunnel first
sudo tailscale up \
  --login-server http://yourname.duckdns.org:8080 \
  --authkey YOUR_AUTHKEY \
  --accept-routes

# ✅ Step 2 — Now SSH via mesh IP (tunnel is active)
ssh muditbhalla2008@100.64.0.3
```

---

## Quick Error Reference Table

Use this table to identify and jump to the correct resolution for any error encountered.

| Error / Symptom | Section | One-Line Fix |
|----------------|---------|--------------|
| Browser freezes / GUI lockup on login | [B1](#incident-b1--authentication-loop--gui-freezing-state) | Use CLI binary; point to `192.168.1.50:8080`, not `100.64.x.x` |
| `requires mentioning all non-default flags` | [B2](#incident-b2--persistent-cache--argument-settings-conflict) | Add `--reset` flag |
| `can't change --login-server without --force-reauth` | [B3](#incident-b3--cross-domain-authority-change-enforcement-lockout) | Add `--force-reauth` flag |
| Exit code `0` but no browser URL printed | [B4](#incident-b4--stdout-interception-by-macos-network-extension-daemon) | Run `Tailscale logout` first, then re-run `up` |
| Node registered but shows `Offline` | [B5](#incident-b5--multi-profile-synchronization-disconnect-offline-node) | Switch active account in Tailscale menu bar app |
| `headscale: command not found` | [C1](#incident-c1--host-binary-execution-failure-command-not-found) | Use `sudo docker exec -it sovereign-headscale headscale ...` |
| `permission denied ...docker.sock` | [C2](#incident-c2--docker-api-unix-socket-permission-denial) | Add `sudo` to all Docker commands |
| Don't know container name for `exec` | [C3](#incident-c3--unidentified-target-container-runtime-context) | Run `sudo docker ps` and read the NAMES column |
| `user not found` / namespace error | [C4](#incident-c4--unknown-headscale-namespace--user-not-found) | Run `headscale users list` to find exact namespace string |
| `-bash: admin-mb: No such file or directory` | [C5](#incident-c5--shell-interpreter-stream-redirection-exception-angle-brackets) | Remove `< >` brackets — paste actual values, not placeholders |
| `doesn't have expected type prefix mkey` | [D1](#incident-d1--cryptographic-machine-key-format-mismatch--schema-drift) | Replace `nodekey:` → `mkey:` in the key string manually |
| `tailscaled.service not running` | [E1](#incident-e1--tailscaled-service-not-running-on-linux-host) | `sudo systemctl enable --now tailscaled` |
| `ssh: Connection timed out` to `100.64.x.x` | [E2](#incident-e2--ssh-connection-timeout-overlay-ip-before-tunnel-established) | Join mesh via public/DDNS IP first — tunnel must exist before SSH |

---

## Final Verified Topology

Following complete resolution of all incidents above, the mesh stabilized to:

| Machine | Mesh IP | Namespace | Status |
|---------|---------|-----------|--------|
| `mb-macbook-air` | `100.64.0.1` | `admin-mb@headscale.net` | ✅ ONLINE / CONNECTED |
| `muditdatabase` | `100.64.0.3` | `admin-mb@headscale.net` | ✅ ONLINE / CONNECTED |

**Global transport confirmed operational:** Routing from external cellular/mobile networks to `100.64.0.3` verified. All traffic is captured, encrypted via symmetric WireGuard keys, and delivered transparently through external WAN firewalls to application endpoints.

---

*Project Sovereign — Incident Logbook | Operator: `muditbhalla2008` | May 29, 2026*
