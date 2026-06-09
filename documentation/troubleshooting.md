# TECHNICAL LEDGER: OPERATIONS RUNBOOKS, ERRORS & INCIDENT SPECIFICATIONS

This operations manual preserves all historical infrastructure logging errors, command-line micro-incidents, and structural corrections recorded during the full deployment lifecycle of Project Sovereign.

---

## System Environment Verification Matrix

* **Log/Audit Reference Timestamp:** May 29, 2026 @ 18:30 IST
* **Primary Operator Identity:** `muditbhalla2008`
* **Target Client Endpoint:** MacBook Air M4 (`XXXX-macbook-air`) running native macOS App Extension Layer
* **Target Server Host Node:** Linux Database Host (`XXXXdatabase`)
* **Orchestration Control Plane:** Containerized Headscale Instance (`v0.23.0-alpha5`)
* **Core Status Affirmation:** FULLY OPERATIONAL / VERIFIED OVERLAY MESH

---

## Step-by-Step Production Integration Runbook

Execute this sequence exactly to register nodes, fix state locks, or clean out system caches across the cluster topology:

### Phase 1: Local Client Subsystem Stabilization
Force clean hanging state parameters on your client laptop terminal to break deadlocks:
```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale logout

/Applications/Tailscale.app/Contents/MacOS/Tailscale up --login-server=[http://192.168.1.50:8080](http://192.168.1.50:8080) --force-reauth --reset


sudo docker ps

# Administrative Namespace Audits
sudo docker exec -it sovereign-headscale headscale users list

# Cryptographic Node Injection
sudo docker exec -it sovereign-headscale headscale nodes register \
  --user admin-mb \
  --key mkey:818b2f57e96289b1d82bc7ffef8f59800cef4597d9b3aa4f3eea3bdd81a7867

# Active Topology Validation
sudo docker exec -it sovereign-headscale headscale nodes list


Comprehensive Micro-Incident Diagnostic Ledger
Incident 1.1: Graphical Interface Lockup
Symptom: Triggering the login mechanism resulted in a stalled browser window canvas and absolute freeze.

Remediation: Dropped GUI control layers and invoked client activation paths directly via the macOS binary shell.

Incident 1.2: Persistent Environmental Cache
Error Output: Error: changing settings via 'tailscale up' requires mentioning all non-default flags...

Remediation: Added explicit argument state scrubbing directives (--reset) to clean existing parameter environments.

Incident 1.3: Schema Drift & Build Variance
Error Output: Cannot register node: key hex string doesn't have expected type prefix mkey

Remediation: Intercepted the validation string and manually rewrote the hex-encoded string prefix to mkey: directly during the server node registration routine.

(For the complete list of all 12 operational incidents, refer to the master architecture logs)



Architectural & Physical Hardware Defect Trackings
1. Host Battery Degradation Hazard (MacBook Pro 2012 Layer)
Defect: Lithium-ion batteries maintained at 100% capacity in a 24/7 server thermal setup face continuous physical breakdown, swelling risks, and hardware expansion.

Mitigation: Unscrewed the inner chassis and physically removed the integrated battery cell block entirely from the loop. The upcycled host functions strictly as a headless desktop unit drawing uninterrupted current from the MagSafe direct connection line.

2. Clamshell Thermal Throttling
Defect: Closing the laptop lid isolates keyboard dissipation areas, trapping core heat loops and triggering CPU thermal scale-downs.

Mitigation: Deployed the laptop chassis vertically in an open configuration cradle with the display cracked open at approximately 15° to secure a continuous fan exhaust airflow stream.
