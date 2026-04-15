# 01 — Server Initialization & Headless OS Provisioning

## What I Did

Set up my Dell Inspiron 560s as a headless Debian 13 server. "Headless" means no monitor, no keyboard, no GUI — just a box sitting somewhere that I SSH into remotely. This was the right call for a low-resource machine like this (Core 2 Duo, 4GB RAM). Every bit of RAM saved from not running a desktop environment is RAM available for actual services.

I installed Debian 13 (Trixie) in text mode, configured a static IP reservation on my router, locked down SSH to key-only authentication, and set up a clean SSH alias on my laptop so connecting is a single command.

---

## Challenges

**The BIOS didn't cooperate cleanly.**
The Dell Inspiron 560s has a pretty customized BIOS that doesn't have a clean "ignore missing keyboard" setting, which is a problem when you want a headless machine that boots without any peripherals attached.

**I was paranoid about the 2TB data drive.**
I had personal data on it and the Debian installer shows you all your drives at once. One wrong click during partitioning and it's gone. The stock installer flow isn't super obvious about which drive it's targeting.

**Typing the full IP and password every SSH login was getting old fast.**
It's also bad practice for automation — you can't script anything cleanly if you're manually authenticating every time.

---

## Solutions

**BIOS hardening:**
I went through the BIOS and disabled everything that wasn't needed — onboard audio, PXE boot, anything burning IRQs for no reason. For the keyboard issue, the Dell BIOS actually auto-bypasses the "keyboard not found" error after a few seconds on its own, so it wasn't a real blocker. I also set **AC Recovery to "Power On"** so the server automatically boots back up after a power cut or if I cycle it with a smart plug. No manual intervention needed.

**LVM partitioning to protect the data drive:**
During the Debian install, I manually selected the 321GB OS drive and used LVM (Logical Volume Manager) partitioning on it. I completely ignored the 2TB drive during this step. LVM also means I can resize my root volume later without reinstalling — useful when Docker logs eventually start eating storage.

**Static IP + SSH config alias + key-only auth:**
Three things working together here:
- Assigned a static IP (`192.168.1.10`) to the server on my Mercusys router via DHCP reservation. I put it at `.10` to leave room for IP cameras and other devices at `.11`, `.12`, etc.
- Generated an Ed25519 keypair on my laptop and pushed the public key to the server with `ssh-copy-id`.
- Created a `~/.ssh/config` entry on my laptop so I can just type `ssh inspiron` instead of the full address.
- Disabled password authentication entirely in `/etc/ssh/sshd_config`.

---

## Learnings

**LVM vs standard partitions:**
I used to just pick "guided partitioning" from tutorials and not think about it. Now I understand why LVM is the professional standard — it creates an abstraction layer between your logical volumes and physical disks, which means you can grow, shrink, and snapshot volumes while the system is live. That's not a nice-to-have in production, it's essential.

**SSH keys are tied to user accounts, not IPs:**
This clicked for me when I set up Tailscale later. My SSH key was pushed to `~/.ssh/authorized_keys` on the server — that's a file in my user's home directory. It doesn't care what IP I'm connecting from. So when I started accessing the server over the Tailscale IP instead of the local IP, my keys worked perfectly without any changes.

**Skipping the root password during Debian install:**
If you leave the root password blank during the Debian installer, it automatically locks the root account and gives your main user full `sudo` access instead. That's actually more secure than having a separate root password you might forget or reuse somewhere.

---

## Notes & Configuration

### SSH Config on `pavilion` (`~/.ssh/config`)

```
Host inspiron
    HostName 100.98.79.12
    User priyanshudeep
    IdentityFile ~/.ssh/id_ed25519
```

> Using the Tailscale IP here instead of the local IP so this works from anywhere — home network, mobile data, anywhere.

### Key sshd_config changes on `inspiron`

```
PasswordAuthentication no
PubkeyAuthentication yes
```

Applied with: `sudo systemctl restart ssh`
