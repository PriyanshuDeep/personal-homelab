# Phase 1: Pavilion Workstation Setup & Hardening

## Overview
I just did a fresh install of Fedora 43 Workstation on my HP Pavilion laptop (Ryzen 5 / RTX 3050). I am setting this up as my daily driver and main machine for my homelab and DevOps learning. The goal today was just to get a stable, fast baseline running using the new `dnf5` package manager without draining my battery unnecessarily. 

## Challenges I Faced
1. **Super Slow Updates:** When I ran my first system update, the terminal was painfully slow and kept throwing a `Curl error 28` timeout. My internet is 30 Mbps, but the system was trying to download packages from a mirror way over in Japan.
2. **The Whole System Froze:** While downloading and installing over 2,200 packages, my laptop completely locked up. The screen froze, I couldn't click anything, and I was just stuck.

## How I Solved Them
* **Fixing the Slow Downloads:** Instead of just dealing with it, I went into the configuration file (`/etc/dnf/dnf.conf`) and added `fastestmirror=True` and `max_parallel_downloads=10`. This forced the system to find the fastest server near me and download 10 things at once. It maxed out my internet speed and fixed the timeouts perfectly.
* **Recovering from the Freeze:** I ended up having to do a hard hardware reboot. When it booted back up, I immediately ran `sudo dnf5 clean all` to wipe out the corrupted cache from the crash. Then I ran `sudo dnf5 upgrade --refresh` to safely rebuild everything and finish the update without losing any data.

## What I Learned
* **Don't Panic on a Crash:** I learned about the difference between User Space (where my graphical interface froze) and Kernel Space. Also, modern package managers like `dnf5` are actually really tough; if you clean the cache properly after a hard power loss, it can recover safely.
* **Public/Private Keys:** I finally understood how SSH keys work (the "padlock and master key" concept) instead of just using passwords for everything. 

## Important Notes & Configs
* **Hybrid Graphics (Saving Battery):** I installed the proprietary NVIDIA drivers (`akmod-nvidia`). Since I want to save battery for daily tasks, I created an alias called `nvrun` in my `.bashrc`. By default, the laptop uses the Ryzen integrated graphics. But if I want to force a heavy app to use the RTX 3050, I just type `nvrun` before the command.
* **My SSH Key:** I generated my first ED25519 SSH keypair to act as my "digital passport" for connecting to GitHub and my future server securely. I used a specific, detailed naming convention for the key to keep my hardware organized, and the private key/passphrase are kept strictly local and secure.
