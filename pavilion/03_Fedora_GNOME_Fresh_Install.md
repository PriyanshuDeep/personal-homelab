# 03 — Fedora KDE to GNOME: Hardware Compatibility & Fresh Start

## The Problem (Again)

One month ago, I installed Fedora 43 with KDE Plasma on my HP Pavilion Gaming laptop. After the first boot, the desktop refused to load — I'd get to the login screen, enter my password, and then... black screen. Nothing. KDE crashed silently every single time.

I worked around it by simply **not rebooting** for a month. I kept the laptop running, hibernating between sessions. Today I rebooted (needed to test something), and immediately hit the same black screen issue again.

**This was the second time in a month.** That's a pattern. That's a problem.

---

## What We Tried (Exhaustive Debugging)

I'm documenting this because the debugging process was thorough and taught me a lot about Linux boot processes, display servers, and when to give up chasing ghosts.

### Attempt 1: Clear KDE Session Cache
**The theory:** KDE's session state files were corrupted from the interrupted boot.

```bash
rm -rf ~/.cache/ksmserver*
rm -rf ~/.cache/plasmarc
rm -rf ~/.config/ksmserver*
```

**Result:** ✗ Black screen persisted.

---

### Attempt 2: Force X11 Instead of Wayland
**The theory:** Wayland (the newer display server) might have compatibility issues with my RTX 3050 Mobile.

```bash
sudo nano /etc/gdm/custom.conf
# Uncommented: WaylandEnable=false
sudo systemctl restart gdm
```

**Result:** ✗ Still black screen after login.

---

### Attempt 3: Boot with `nomodeset` (Disable GPU Drivers)
**The theory:** Maybe the NVIDIA drivers were crashing the session.

Added `nomodeset` to the kernel boot parameters in GRUB. This tells Linux to use ultra-basic fallback graphics, no GPU acceleration.

**Result:** ✗ Surprisingly, the screen got past login but STILL went black. The desktop environment itself was the problem, not the graphics.

---

### Attempt 4: Try Rescue Mode
**The theory:** Boot into a minimal environment and manually restart the display manager.

```bash
# Added to GRUB kernel line:
systemd.unit=rescue.target
```

This should boot into a root shell bypassing systemd completely.

**Result:** ✗ Got stuck — the system told me "root account is locked" (because I skipped setting a root password during install, which is actually good practice).

---

### Attempt 5: Boot with `init=/bin/bash` (Kernel-Level Shell)
**The theory:** Bypass systemd entirely and get a bash shell before anything else loads.

```bash
# Added to GRUB kernel line:
init=/bin/bash
```

**Result:** ✗ Never got the bash prompt. The system hung somewhere in the early boot sequence.

---

### Attempt 6: Live USB Recovery (The Nuclear Option)
**The theory:** Boot from a live Fedora ISO, mount my NVMe drive, and surgically delete ALL KDE configuration files.

Created a Ventoy USB with Fedora 43 ISO, booted into live environment:

```bash
# Mount the NVMe drive
sudo mount /dev/nvme0n1p3 /mnt/fix

# Delete every trace of KDE
sudo rm -rf /mnt/fix/home/priyanshudeep/.cache/ksmserver*
sudo rm -rf /mnt/fix/home/priyanshudeep/.cache/plasmarc*
sudo rm -rf /mnt/fix/home/priyanshudeep/.local/share/ksmserver
sudo rm -rf /mnt/fix/home/priyanshudeep/.config/ksmserver*
sudo rm -rf /mnt/fix/home/priyanshudeep/.config/plasmarc

# Unmount and reboot
sudo umount /mnt/fix
sudo poweroff
```

**Result:** ✗ Rebooted into the same black screen. KDE was still broken.

---

## The Realization

After 6 attempts with increasing desperation, the pattern was undeniable:

- System boots fine (BIOS, bootloader, kernel all work)
- Login screen appears and accepts credentials
- KDE Plasma crashes **during session initialization** on this specific hardware
- Every kernel from GRUB failed the same way
- Even with graphics disabled (`nomodeset`), the desktop wouldn't load

**This wasn't a corrupted cache or a driver issue. This was KDE Plasma incompatible with my HP Pavilion hardware.**

And the fact that it happened **twice in a month** (once on first boot, once after today's reboot) meant this wasn't a one-time glitch. This was systematic.

---

## The Decision: Switch to GNOME

**Why GNOME and not keep fighting KDE?**

1. **Pragmatism over stubbornness.** After 6 debugging attempts, the pattern was clear. Continuing to chase ghosts wastes time I could spend learning the actual tools (Docker, Kubernetes, Infrastructure as Code).

2. **Hardware fit matters.** GNOME is built for consumer laptops and handles GPU configurations more gracefully than KDE. Sometimes the right tool isn't about features — it's about compatibility.

3. **Fedora GNOME is the standard.** KDE is Fedora's "flashy" option; GNOME is their proven, battle-tested default. Start with what works.

4. **Repetition signals incompatibility.** When the same issue happens twice the same way, it's not bad luck. It's a signal to try a different approach.

I decided to do a **complete fresh install** of Fedora 43 with GNOME. Nuclear option, but justified by the data.
