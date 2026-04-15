# Personal DevOps Homelab

## The Goal

This repo is the living documentation of my homelab — a real, self-managed infrastructure I built from scratch while learning DevOps. Every file here is a milestone log: what I did, why I made those decisions, what broke, and what I learned from it.

The goal isn't just to run some services. I'm treating this like a production environment from day one — security first, everything documented, and no shortcuts. I'm using this homelab to build the practical skills that get you hired, not just the theory.

---

## Infrastructure Overview

### The Workstation — `pavilion` (Control Node)
- **Hardware:** HP Pavilion Gaming Laptop — AMD Ryzen 5 5600H, RTX 3050 Mobile, 16GB RAM
- **OS:** Fedora 43 KDE Plasma
- **User / Hostname:** `priyanshudeep` / `pavilion`
- **Role:** My daily driver and main control node. I use it to write code, manage the server over SSH, and do all my DevOps learning.

### The Server — `inspiron` (Production Node)
- **Hardware:** Dell Inspiron 560s — Core 2 Duo, 4GB RAM
- **Backstory:** This is literally my childhood PC that I repurposed into a headless server.
- **OS:** Debian 13 (Trixie) — Headless, no GUI
- **User / Hostname:** `priyanshudeep` / `inspiron`
- **Local IP:** `192.168.1.10` (static reservation on router)
- **Tailscale IP:** `100.98.79.12`
- **Storage:** 288GB OS volume (LVM) + 2TB Toshiba XFS drive mounted at `/mnt/2TB_Storage`
- **Role:** 24/7 headless server for self-hosted services.

---

## Tech Stack

| Layer | Tool |
|---|---|
| OS | Debian 13 (Server), Fedora 43 (Workstation) |
| Networking | Tailscale (WireGuard mesh), SSH with Ed25519 keys |
| Containerization | Docker Engine |
| Storage | XFS filesystem on 2TB data drive |
| Remote Access | KDE Dolphin via KIO/SFTP, Termius (Android) |

---

## Repository Structure

```
personal-homelab/
├── README.md
├── inspiron/               # Server setup and configuration logs
│   ├── 01_Server_Init.md
│   ├── 02_Storage_Provisioning.md
│   ├── 03_Zero_Trust_Network.md
│   └── 04_Docker_Setup.md
└── pavilion/               # Workstation setup logs
    └── 01_Workstation_Setup.md
```

---

## Progress Tracker

### inspiron (Server)
- [x] Bare-metal Debian 13 headless install + BIOS hardening
- [x] LVM partitioning — OS on dedicated volume, data drive untouched during install
- [x] SSH hardened — Ed25519 keys only, password auth disabled
- [x] Static IP reservation on Mercusys router (`192.168.1.10`)
- [x] 2TB drive formatted to XFS, mounted at `/mnt/2TB_Storage`, fstab with `nofail`
- [x] Tailscale installed bare-metal (not in Docker — intentional)
- [x] Docker Engine installed and verified
- [ ] Deploy service stacks (Dockge, Nginx Proxy Manager, Jellyfin, Uptime Kuma)
- [ ] Automated rsync backups
- [ ] Monitoring and alerting

### pavilion (Workstation)
- [x] Fresh Fedora 43 install
- [x] SSH keypair generated, server access configured
- [x] Tailscale installed and connected
- [x] KDE Dolphin / KIO SFTP integration to server
- [ ] Full dev environment setup (Git, VSCode, etc.)
