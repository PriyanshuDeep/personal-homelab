# Storage Provisioning & Health Verification

## What I Did
I successfully evacuated all my personal legacy data from my server's 2TB drive to my laptop. Before provisioning the server drives for production use, I ran S.M.A.R.T. hardware diagnostics on everything. Finally, I formatted the 2TB drive as a high-performance XFS filesystem and mounted it over the network to my KDE workstation.

## The Hardware Backstory & Diagnostics
Pulling the S.M.A.R.T. telemetry revealed the actual history of my homelab hardware:

* **Workstation NVMe (512GB) & HDD (1TB):** Both drives in my Fedora laptop are perfectly healthy with 0 errors and plenty of lifespan left.
* **Server OS HDD (321GB):** This server is actually my very first childhood PC! The telemetry showed roughly 6,900 hours of spin time but almost 6,000 power cycles. It perfectly reflects how I used it growing up: booting it up to use, and shutting it down every night. It has 0 bad sectors and is perfectly healthy to run my Debian OS.
* **Server Data HDD (2TB):** This is a Toshiba Video-series drive I salvaged from our house's old CCTV DVR before we upgraded to IP cameras. The telemetry shows over 35,000 hours (4+ years) of 24/7 spinning. The magnetic platters are flawless, but because the physical motor is so old, I am treating this drive as volatile. I will be building automated backup cronjobs for it later.

## Execution
* **Format:** Wiped the 2TB drive and formatted to `xfs` using `mkfs.xfs` for optimized large-file handling (perfect for my future NVR).
* **Mounting:** Created the permanent mount point at `/mnt/2TB_Storage`. Extracted the cryptographic UUID and added it to `/etc/fstab` using the `nofail` flag. This ensures the server doesn't hang on boot if this aging 2TB drive suddenly dies.
* **Workstation Integration:** Connected to the drive directly from my Fedora KDE Plasma workstation using Dolphin and KIO (`sftp://[REDACTED_ALIAS]/mnt/2TB_Storage`). Pinned it to my sidebar so I can seamlessly drag-and-drop files to the server over the local network.
