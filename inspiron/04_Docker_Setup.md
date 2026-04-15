# 04 — Docker Service Stack Deployment

## What I Did

This was the big one. I deployed my initial service stack using Docker Compose
— a media server, a monitoring dashboard, a reverse proxy, and a management
UI to tie it all together. Every service runs in its own container, all
connected through a shared Docker network, with data stored on the 2TB drive
so it survives container rebuilds.

The stack I deployed:

- **Dockge** — web UI for managing Docker Compose stacks
- **Nginx Proxy Manager** — reverse proxy with SSL termination
- **Uptime Kuma** — service monitoring and uptime tracking
- **Jellyfin** — self-hosted media server

I deliberately skipped Vaultwarden for now. My 2TB drive is a salvaged CCTV
drive with 35,000+ hours on it — putting a password vault on volatile storage
without a backup system in place first is a bad idea. I'll revisit it after
automated backups are running.

---

## Directory Structure

Before touching any containers, I built a clean directory structure on the
2TB drive. Every service gets its own folder. Service config and data are
completely separate from media files.

/mnt/2TB_Storage/
├── docker/
│   ├── dockge/
│   │   └── stacks/
│   ├── nginx-proxy-manager/
│   │   ├── data/
│   │   └── letsencrypt/
│   ├── uptime-kuma/
│   │   └── data/
│   └── jellyfin/
│       ├── config/
│       └── cache/
└── media/
├── movies/
├── shows/
├── music/
├── anime/
└── anime-movies/

The separation between `docker/` and `media/` is intentional. If I ever
delete and recreate the Jellyfin container, the media library is completely
untouched. Config and content should never live in the same place.

---

## Shared Docker Network

All containers are attached to a single custom bridge network called
`homelab`. I created this manually before deploying anything:

```bash
docker network create homelab
```

The reason for a shared network is NPM. For Nginx Proxy Manager to route
traffic to other containers, it needs to be able to reach them by container
name. Docker's internal DNS only works within the same network. If each
Compose stack used its own isolated network, NPM couldn't see any of them.
One shared network solves this for every service at once.

---

## Challenges

**Jellyfin's IP filtering blocked my Tailscale IP.**

During the setup wizard I said no to remote connections, thinking I didn't
need it since I'm on Tailscale. What I didn't realize is that Jellyfin
considers anything outside `192.168.x.x` as "remote" — including Tailscale
IPs in the `100.x.x.x` range. So it was correctly following my instruction
and blocking my own laptop.

The wizard also got interrupted when I refreshed the page mid-setup, which
left a corrupted database and a root-owned hidden file `.jellyfin-data` in
the config directory that prevented a clean restart.

**The fix** was to fully stop the container, wipe the config directory
including the hidden file with `sudo rm`, and run the wizard again from
scratch — this time saying yes to remote access. Behind Tailscale, remote
access being enabled is fine because there are no open ports on my router.
Tailscale handles authentication at the network level.

---

## Solutions

**Organized everything on the 2TB drive first.**
No containers touch the OS drive for data. Everything is bind-mounted from
`/mnt/2TB_Storage/docker/`. If I reinstall Debian tomorrow, I reconnect the
drive and my service data is intact.

**Deployed Dockge manually first, everything else through Dockge.**
Dockge itself was the only container I deployed via a raw `docker compose`
command in the terminal. Every other service was deployed through Dockge's
web UI. This is the right workflow — get your management tooling running
first, then use it.

**Monitoring went in early, not at the end.**
Uptime Kuma was the third service deployed, not the last. Every service gets
a monitor added immediately after it comes up. I'm not waiting until the
stack is "complete" to start monitoring it.

**Skipped Vaultwarden deliberately.**
Not every planned service needs to be deployed immediately. Vaultwarden on
a volatile drive without backups is more risk than it's worth right now.
The decision of what NOT to deploy is just as important as what to deploy.

---

## Learnings

**Bind mounts vs Docker volumes.**
Docker has two ways to persist data: named volumes (Docker manages the
location) and bind mounts (you specify the exact path on the host). I used
bind mounts for everything. This means I always know exactly where my data
is on disk, I can browse it with Dolphin over SFTP, and it's not hidden
inside Docker's internal directory structure. For a homelab, bind mounts are
the right call.

**Container networking is DNS-based.**
Containers on the same Docker network can reach each other by container name
— `http://jellyfin:8096` resolves correctly from inside NPM's container
without any manual IP configuration. Docker runs an internal DNS server that
handles this automatically. Understanding this is essential for setting up
reverse proxies.

**`restart: unless-stopped` is the correct policy for services.**
This tells Docker to restart a container automatically if it crashes, but
respect a manual `docker stop` command. `always` would restart even after
a deliberate stop, which gets annoying. `on-failure` only restarts on
crashes, which means your services don't come back after a server reboot.
`unless-stopped` is the middle ground that makes sense for 24/7 services.

**Read the logs before doing anything else.**
When Jellyfin wasn't loading, I ran `docker logs jellyfin --tail 50` and
found the exact problem in under a minute. The logs told me precisely which
IP was being blocked and exactly why. On production systems, logs are always
the first place you look — not forums, not random Stack Overflow answers.

---

## Service Access

| Service | URL | Port |
|---|---|---|
| Dockge | http://100.98.79.12:5001 | 5001 |
| Nginx Proxy Manager | http://100.98.79.12:81 | 81 |
| Uptime Kuma | http://100.98.79.12:3001 | 3001 |
| Jellyfin | http://100.98.79.12:8096 | 8096 |

All accessible over Tailscale from anywhere. No ports open on the router.

## Current Stack

```bash
CONTAINER       IMAGE                         STATUS
dockge          louislam/dockge:1             Up (healthy)
nginx-proxy-manager  jc21/nginx-proxy-manager  Up
uptime-kuma     louislam/uptime-kuma:1        Up (healthy)
jellyfin        jellyfin/jellyfin:latest      Up (healthy)
```
