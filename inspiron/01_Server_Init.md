# Server Initialization & Headless OS Provisioning

## What I Did
I finally set up my bare-metal server for the homelab. I'm using an old Dell Inspiron 560s (Core 2 Duo, 4GB RAM), so I had to be really strict about saving resources. Instead of setting it up like a desktop, I configured the BIOS and OS to run completely "headless"—meaning no monitor, no keyboard, and absolutely no Graphical User Interface (GUI). I installed Debian 12 in text mode, set up a static IP on my router, and locked down the server so I can only access it securely via SSH keys from my laptop.

## Challenges
* **The BIOS Limitations:** The Dell motherboard is customized and didn't have a straightforward "Halt On" setting to ignore missing keyboards, which is a problem for a headless setup.
* **Storage Paranoia:** I have a 321GB drive for the OS and a 2TB drive full of personal data. I was nervous about accidentally wiping my data drive during the Linux installation.
* **Typing Passwords:** Constantly typing out my server IP and user password to log in over SSH was getting annoying and isn't good for future automation.

## Solutions
* **BIOS Hardening:** I disabled unnecessary hardware (like onboard audio and PXE boot) to save IRQs and power. I also set "AC Recovery" to "Power On" so the server automatically boots up if my smart plug cycles the power. The Dell BIOS actually auto-bypasses the keyboard error after a few seconds anyway!
* **LVM Partitioning:** I specifically selected the 321GB drive and used LVM (Logical Volume Management). I left the 2TB drive completely untouched for now. 
* **Static IP Block:** On my Mercusys router, I assigned the server a static IP (`192.168.x.10`) that is outside my normal dynamic DHCP pool. I kept it at `.10` so I have room to assign `.11`, `.12`, etc., to my future IP cameras.
* **Passwordless SSH:** I pushed my laptop's ED25519 public key to the server using `ssh-copy-id`. To make it seamless, I set up a local `~/.ssh/config` alias so I can just type `ssh inspiron`. Finally, I edited `/etc/ssh/sshd_config` on the server and changed `PasswordAuthentication no`, locking out anyone trying to guess a password.

## Learnings
* **LVM vs. Standard Partitions:** I used to just pick manual partitioning from tutorials. Now I know LVM is the professional standard because it lets you resize volumes dynamically while the server is running—super useful for when Docker logs eventually eat up my storage space.
* **Identity vs. Routing:** Pushing an SSH key attaches it to the user account on the server, not the IP address. This means when I eventually set up Tailscale (VPN) to access this outside the house, my SSH keys will still work perfectly without needing to be re-copied.
* **Skipping the Root Password:** Leaving the root password blank during the Debian install automatically locks the root account and sets up the standard user with `sudo`. It's way more secure.

## Notes & Code Snippets
My local client-side `~/.ssh/config` block looks like this:

```text
Host inspiron
    HostName 192.168.x.x
    User [REDACTED_USERNAME]
    IdentityFile ~/.ssh/id_ed25519
