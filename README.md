# PROJECT SOVEREIGN: TECHNICAL SPECIFICATION & IAC OPERATIONS

[![IaC: Docker-Compose](https://img.shields.shields.shields.shields.shields.io/badge/IaC-Docker--Compose-blue?logo=docker&style=flat-square)](./docker-compose.yml)
[![Overlay Mesh: Headscale](https://img.shields.shields.shields.shields.shields.io/badge/Mesh-Headscale-orange?style=flat-square)](#-4-cryptographic-mesh-overlay-networking)
[![Security: Isolated Data Tier](https://img.shields.shields.shields.shields.shields.io/badge/Security-Hardened--Data-red?style=flat-square)](#-5-security-hardening--the-isolated-data-tier)

Project Sovereign is a self-hosted, hyper-converged edge cloud engineered to enforce absolute data sovereignty, zero-telemetry privacy, and physical hardware cycle optimization. The system handles decentralized cryptographic mesh networking, structured data storage tiers, local machine learning computer vision processing, and edge AI inferencing loops.

---

## Executive Summary & Architectural Evolution

Originally designed for an enterprise HP ProLiant MicroServer running a Type-1 Hypervisor (Proxmox VE) with a multi-drive ZFS storage pool, the system was dynamically refactored to eliminate hypervisor layer overhead (recovering 1–2 GB of system RAM and reducing scheduling latencies). 

### The Upcycled Edge Node Hardware Pivot
* **Host Layer System:** Ivy Bridge MacBook Pro (Legacy 2012 Hardware Node).
* **Operating System Engine:** Ubuntu Server 26.04 LTS installed directly onto bare metal.
* **Compute Optimization Loop:** To accommodate the lack of modern AVX2 data-center vector instructions on mobile Ivy Bridge chips, the containerized layer was stripped of heavy virtualization constraints—allowing `llama.cpp` to compile directly onto raw host bare metal to optimize native C++ loop speeds for GGUF quantized models.

---

## Repository & IaC Directory Layout

This project rejects manual environment configuration ("ClickOps"). The repository is built strictly using modular Infrastructure as Code (IaC) components:

```text
project-sovereign/
├── .gitignore                      # Cryptographic data leak barrier
├── README.md                       # Main architecture navigation hub
├── docker-compose.yml              # Central multi-tier service orchestrator
├── config/
│   └── headscale/
│       ├── config.yaml              # Core overlay networking control plane rules
│       └── acls.json                # Hardware access firewall policies
└── documentation/
    └── troubleshooting.md          # 12-Incident Diagnostic Ledger & Runbooks
```

Have questions or ideas? Join the [Discussions](https://github.com/muditbhalla/project-sovereign/discussions)!
