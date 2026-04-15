# 01 — Workstation Setup & Initial Hardening

## What I Did

Fresh install of Fedora 43 KDE Plasma on my HP Pavilion Gaming Laptop (Ryzen 5 5600H, RTX 3050 Mobile, 16GB RAM). This is my daily driver and control node for the homelab — everything I do on the server goes through this machine. The goal for this phase was a clean, stable baseline: system fully updated, SSH access to the server working, and Tailscale connected so I can reach the server from anywhere.

---

## Challenges

**System updates were painfully slow with timeouts.**
First `dnf upgrade` after a fresh Fedora install kept stalling with `Curl error 28` (connection timeout). My internet is fine — the problem was that DNF was pulling packages from a mirror in Japan by default.

**The whole system froze mid-update.**
While downloading and installing 2,200+ packages, my laptop completely locked up. Couldn't click anything, keyboard unresponsive. Full freeze.

---

## Solutions

**Fixing the slow mirrors:**
Edited `/etc/dnf/dnf.conf` and added two lines:

```ini
fastestmirror=True
max_parallel_downloads=10
```

`fastestmirror` makes DNF benchmark nearby mirrors and pick the fastest one automatically. `max_parallel_downloads=10` lets it download 10 packages simultaneously instead of one at a time. After this, updates maxed out my connection speed and the timeouts stopped completely.

**Recovering from the freeze:**
Hard rebooted. When the system came back up, I ran:

```bash
sudo dnf5 clean all
sudo dnf5 upgrade --refresh
```

`clean all` wipes the package cache — important after a hard power loss because partial downloads can corrupt the cache. `--refresh` forces DNF to re-fetch the latest metadata from the repos. Everything rebuilt cleanly, no data loss.

---

## SSH Setup & Server Access

Generated an Ed25519 keypair on this machine to use as my "digital passport" for both GitHub and the server:

```bash
ssh-keygen -t ed25519 -C "priyanshudeep@pavilion"
```

Pushed the public key to the server:

```bash
ssh-copy-id priyanshudeep@192.168.1.10
```

Set up a `~/.ssh/config` alias so connecting is just `ssh inspiron`:

```
Host inspiron
    HostName 100.98.79.12
    User priyanshudeep
    IdentityFile ~/.ssh/id_ed25519
```

Using the Tailscale IP instead of the local IP means this alias works from anywhere, not just when I'm home.

---

## Tailscale

Installed Tailscale on the laptop and connected it to my Tailscale account (professional Microsoft account). The Fedora 43 / DNF5 quirk here is covered in detail in the server's [03_Zero_Trust_Network.md](../inspiron/03_Zero_Trust_Network.md) — same fix applies on the laptop side.

Pavilion's Tailscale IP: `100.102.43.106`

---

## KDE Dolphin Integration

Connected to the server's storage directly from Dolphin using KIO:

```
sftp://inspiron/mnt/2TB_Storage
```

Pinned to the sidebar. I can now drag and drop files between my laptop and the server's 2TB drive exactly like a local folder. No terminal needed for basic file transfers.

---

## Learnings

**DNF5 is a full rewrite, not just an update.**
Fedora 40+ ships with DNF5, which has different command syntax and different config behavior compared to the old DNF. A lot of guides online are written for the old DNF. When something doesn't work, check whether the command has changed in DNF5 before assuming the guide is wrong.

**`clean all` after a hard crash is not optional.**
The package manager maintains a cache of downloaded files. If the system dies mid-download, that cache can have corrupted partial files. Running `clean all` first ensures you're starting fresh rather than trying to resume from a broken state.

**SSH keys are per-device on purpose.**
Each machine I use to access the server has its own keypair. This isn't just tidiness — it means if one device is compromised or lost, I revoke exactly that key and nothing else changes. Shared keys are a security anti-pattern.

---

## Current State

- [x] Fedora 43 installed and fully updated
- [x] DNF5 optimized (fastestmirror, parallel downloads)
- [x] Ed25519 SSH keypair generated
- [x] SSH access to `inspiron` working via `ssh inspiron`
- [x] Tailscale connected (`100.102.43.106`)
- [x] KDE Dolphin / KIO SFTP integration working
