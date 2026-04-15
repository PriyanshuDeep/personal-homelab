# 02 — Storage Provisioning & Health Verification

## What I Did

Before touching the 2TB drive for production use, I ran full S.M.A.R.T. diagnostics on every drive in the system. Once I knew what I was working with, I evacuated the personal data off the 2TB drive to my laptop, formatted it to XFS, and mounted it permanently with a safe fstab configuration. I also connected the drive to my KDE workstation over the network so I can drag and drop files to the server from Dolphin.

---

## The Hardware Story (What S.M.A.R.T. Told Me)

Running diagnostics before provisioning any drive is non-negotiable. You don't want to build a media server on a drive that's about to fail. Here's what the telemetry showed:

**Workstation drives (pavilion — NVMe + HDD):**
Both healthy. Zero errors, plenty of lifespan left. Nothing to worry about.

**Server OS drive (321GB HDD):**
This is literally my childhood PC's original hard drive. The telemetry showed around **6,900 hours of total spin time** but nearly **6,000 power cycles**. That perfectly matches how I used this machine growing up — boot it up, use it, shut it down every night. Zero bad sectors. Healthy enough to run the OS for years.

**Server data drive (2TB Toshiba):**
This one I salvaged from our house's old CCTV DVR system before we upgraded to IP cameras. The telemetry showed over **35,000 hours of runtime** — that's 4+ continuous years of 24/7 spinning. The magnetic platters are in perfect condition (zero reallocated sectors), but the physical motor has serious mileage on it. I'm treating this drive as **volatile** — it could mechanically fail with little warning. Automated backups are on my roadmap specifically because of this.

---

## Challenges

**Data evacuation before formatting.**
The drive had years of personal files on it from the DVR. I had to move everything to my laptop first before I could wipe it, and I wanted to verify the drive's health before trusting it with new data.

**Making the mount survive reboots safely.**
Adding a drive to `/etc/fstab` is straightforward, but if that drive ever fails to appear at boot (which is realistic given the motor hours on this one), the server will hang waiting for it. That's a remote server I can't physically access in an emergency — a boot hang would be a real problem.

**Making the drive accessible from my laptop.**
SSH is great for terminal work but not for casually moving large media files. I needed a more natural way to interact with the storage from my workstation.

---

## Solutions

**XFS filesystem:**
Formatted the 2TB drive with `mkfs.xfs`. XFS is optimized for large sequential files — exactly what a media server deals with. It handles large video files more efficiently than ext4 and has better performance for this specific use case.

**fstab with `nofail`:**
Instead of referencing the drive by its device path (`/dev/sdb1` — which can change), I extracted the drive's cryptographic UUID and used that in `/etc/fstab`. UUIDs are permanent and don't shift around when you add hardware. The `nofail` flag is critical — it tells the boot process to skip this drive and continue booting normally if it can't be found. Given this drive's age, that flag is not optional.

```
# /etc/fstab entry for 2TB data drive
UUID=<drive-uuid>  /mnt/2TB_Storage  xfs  defaults,nofail  0  2
```

**KDE Dolphin via KIO/SFTP:**
Connected to the server's storage directly from Dolphin on my Fedora workstation using `sftp://inspiron/mnt/2TB_Storage`. Pinned it to the sidebar. Now I can move files to the server exactly like dragging to an external drive — no terminal needed for basic file operations.

---

## Learnings

**Always run S.M.A.R.T. before provisioning.**
I didn't just do this because it's good practice — I genuinely didn't know the history of either server drive before running diagnostics. The telemetry told me a story about my own hardware I couldn't have known otherwise. On production hardware you'd do this before every new deployment.

**UUID over device paths in fstab.**
Device paths like `/dev/sdb1` are assigned at boot based on detection order. Add a USB drive, and suddenly your `/dev/sdb1` might be something else entirely. UUIDs are burned into the drive's firmware and never change. Always use UUIDs in fstab.

**`nofail` is a safety net, not laziness.**
Some tutorials skip this flag. On a desktop it doesn't matter much. On a headless server you can't physically reach in an emergency, a boot hang because of a dead drive is a serious problem. `nofail` means the OS keeps running even if the data drive dies — you just lose access to the data, not the whole machine.

---

## Current Storage Layout

```
Filesystem                       Size    Used   Avail   Mount
/dev/mapper/inspiron--vg-root    288G    1.7G   272G    /
/dev/sda1                        943M    124M   754M    /boot
/dev/sdb1                        1.9T     36G   1.8T    /mnt/2TB_Storage
```
