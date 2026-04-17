# 05 — Smart Home & NVR Integration: The Experiment That Taught Me More Than It Built

## What I Tried To Do

After getting the core Docker stack running, I wanted to go further. The idea
was to build a unified, local-first smart home and NVR control plane — one
app for my entire family to control smart lights, smart plugs, and view all
three IP cameras with live streaming, PTZ control, and historical SD card
playback. No cloud dependency, no vendor apps, full ownership.

The devices I was working with:

**Cameras:**
- Tapo C200 (with 64GB SD card)
- Tapo C210 (with 128GB SD card)
- Ezviz H8C (with 128GB SD card)

**Smart Devices:**
- TP-Link Tapo P110M smart plug (monitoring my server's power consumption)
- Havells Crabtree smart switch (controlling the house water motor)
- 2x Havells Glamax RGB smart bulbs (running on Tuya infrastructure)

The plan looked clean on paper. In practice, it ran into three separate walls
— hardware limits, vendor politics, and API lockdowns. I'm documenting every
path I tried and exactly why each one failed, because the decision to *not*
build something is just as worth documenting as the decision to build it.

---

## Architecture Decisions

### Why I Wanted Local-First

The whole point of a homelab is ownership. Cloud-dependent smart home systems
mean the manufacturer can shut down their servers tomorrow and your ₹2,000
smart bulb becomes a regular bulb. Local control means my automations run
even if the internet is down, my camera footage doesn't route through
someone else's servers, and nobody is selling my household data.

That was the goal. Hardware reality had other plans.

### Why Home Assistant Over Google Home

Google Home already had all my devices — the bulbs via Smart Life, the plugs
via Tapo, the cameras via their native apps. But Google Home is a cloud
dependency. Home Assistant running locally means my lights still work during
an internet outage, I get local network speed instead of cloud round-trips,
and I own my automation data. That was the theory. We'll get to where it
fell apart.

### Why Scrypted Over Frigate

My server is a Dell Inspiron 560s — Core 2 Duo, 4GB RAM. The first thing I
did before committing to any NVR software was check for AVX instruction
support:

```bash
grep -o 'avx[^ ]*' /proc/cpuinfo | sort -u
```

Empty output. My CPU has zero AVX support. Frigate's AI object detection
pipeline requires AVX instructions at minimum, and without a hardware
accelerator like a Google Coral TPU, Frigate would permanently peg both CPU
cores at 100% and starve every other service on the server. Frigate was
off the table before I even looked at the config docs.

Scrypted is a different beast. It's not an NVR in the traditional sense —
it's a camera integration and protocol bridge. Its job is to take RTSP
streams from cameras of any brand and make them appear natively in HomeKit,
Google Home, or Alexa. Much lighter than Frigate, and directly relevant to
my goal of getting all three cameras into one app.

### Why `network_mode: host` For Home Assistant

Home Assistant needs to discover devices on the local LAN — smart plugs,
bulbs, and cameras broadcasting their presence via multicast and broadcast
packets. Docker's bridge network blocks these packets by design. Running
Home Assistant with `network_mode: host` puts it directly on the server's
network interface so it can see everything on `192.168.1.0/24` without any
manual configuration. This is the documented approach from the Home Assistant
team for Docker deployments.

### Why Static IP Reservations Before Anything Else

Before touching any integration, I assigned DHCP reservations for every
smart device on the network via the Mercusys router. This is non-negotiable.

Home Assistant and Scrypted connect to devices by IP address. If a device's
IP changes after a router reboot — which standard DHCP will eventually do —
the integration silently breaks. You wake up to unavailable devices and have
to reconfigure manually. Static reservations mean the router always hands
the same device the same IP, forever.

Client devices (phones, laptops, the TV) don't need static IPs. Nothing
connects *to* them — they initiate connections outward. Only servers and
IoT devices that *receive* connections need stable addresses.

| Device | Reserved IP |
|---|---|
| inspiron (server) | `192.168.1.10` |
| Tapo P110M | `192.168.1.20` |
| Havells Bulb 1 | `192.168.1.21` |
| Havells Bulb 2 | `192.168.1.22` |
| Havells Crabtree Switch | `192.168.1.23` |
| Tapo C200 | `192.168.1.30` |
| Tapo C210 | `192.168.1.31` |
| Ezviz H8C | `192.168.1.32` |

These reservations are staying in the router even after this experiment ended.
They're a networking best practice regardless of what software is using them.

---

## The Three Paths We Tried (And Why Each Failed)

### Path A — Scrypted → Google Home

**The plan:** Deploy Scrypted, add all three cameras via their RTSP streams,
install the Google Home plugin, and have the cameras appear natively in the
Google Home Android app. One app, all cameras, live view and PTZ.

**What worked:** Scrypted deployed cleanly. The Tapo cameras connected
immediately via the Tapo plugin. The Ezviz H8C connected via RTSP. PTZ
controls worked inside Scrypted's own web UI. The Google Home plugin linked
successfully.

**Where it died:** Google. At some point Google locked down their Google Home
API for third-party camera integrations. Third-party cameras bridged through
Scrypted no longer stream live video in the Google Home mobile app. The
cameras appear in the app, you tap them, and you get a loading spinner that
never resolves. This isn't a Scrypted bug — it's a deliberate API restriction
on Google's side, and there's nothing to configure around it.

The fallback — Scrypted's own NVR for local recording directly to the 2TB
drive — requires a paid Scrypted NVR license. That's a non-starter. The
whole point is to not pay for cloud services.

Path A closed.

### Path B — Home Assistant As The Master App

**The plan:** Scrap the Google Home angle entirely. Use Home Assistant as the
single unified interface — cameras, smart devices, automations, everything.
The Home Assistant Companion app on Android becomes the one app for the whole
family.

**What worked:** The community Tapo integration for Home Assistant is
genuinely impressive. It connected to the C200 and C210, pulled live WebRTC
streams with sub-second latency, exposed full PTZ controls, and even synced
the SD card recordings so you could browse historical footage directly from
the Home Assistant dashboard. The Tapo P110M smart plug and Havells bulbs
(via the Tuya integration) also integrated cleanly. For a moment this looked
like it was actually going to work.

**Where it died:** The Ezviz H8C. Ezviz locks down their API significantly
more than Tapo. Home Assistant could pull the live RTSP stream from the H8C
and control PTZ, but it could not access historical SD card recordings. There
is no Ezviz integration in Home Assistant that supports SD card playback —
the API simply doesn't expose it.

This meant the family would still need the Ezviz app open for any historical
footage from one out of three cameras. That's not a unified experience — it's
a fragmented one with extra complexity layered on top.

Path B closed.

### Path C — Lightweight NVR (MotionEye / AgentDVR)

**The plan:** Forget SD card sync entirely. Record all three camera streams
locally to the 2TB drive via a lightweight NVR that my CPU can handle, and
view recordings from there.

**Why I didn't pursue this seriously:** My 2TB drive is a salvaged CCTV DVR
drive with over 35,000 hours of runtime. Writing continuous camera streams to
it — even motion-triggered — accelerates the wear on a drive that's already
living on borrowed time. Beyond the hardware risk, the UX for non-technical
family members browsing NVR recordings on a mobile browser is genuinely
terrible compared to the native apps. PTZ support across mixed Tapo and Ezviz
hardware in open-source NVR software is inconsistent at best.

The juice wasn't worth the squeeze.

Path C closed before it started.

---

## The Executive Decision

> The ideological benefit of 'total local ownership' is not worth it when the
> cost is a fragmented, broken UX for my family.

Every person in my house uses the Tapo app for the Tapo cameras and the Ezviz
app for the Ezviz camera. They already know how to use them. They work
perfectly, from anywhere, with full PTZ and SD card playback, completely for
free. That's the bar I was trying to clear. I didn't clear it.

The right call is to let the native apps do what they already do well, stop
burning server resources trying to replace them, and direct that energy toward
things the homelab can actually do better than the cloud.

Smart devices (bulbs and plugs) are already integrated in Google Home via
Smart Life and the Tapo app. That also works perfectly. Home Assistant running
24/7 on a Core 2 Duo to replicate functionality that Google Home already
provides for free is waste, not improvement.

---

## The Graceful Teardown

Stopped and deleted the `scrypted` and `homeassistant` stacks via the Dockge
UI, then purged all persistent data from the terminal:

```bash
sudo rm -rf /mnt/2TB_Storage/docker/homeassistant
sudo rm -rf /mnt/2TB_Storage/docker/scrypted
sudo rm -rf /opt/stacks/scrypted
sudo rm -rf /opt/stacks/homeassistant
```

Also deleted the local `admin` RTSP accounts that were created on the Tapo
cameras during the Scrypted integration. Those accounts were created
specifically to allow Scrypted to pull the RTSP stream. With Scrypted gone,
those accounts are an unnecessary attack surface — even behind Tailscale,
credentials that aren't needed shouldn't exist.

The DHCP static IP reservations in the Mercusys router were kept. They cost
nothing, they make the network more predictable, and they're good practice
regardless of what software is running.

---

## Learnings

**Check your CPU's instruction set before committing to any software.**
`grep -o 'avx[^ ]*' /proc/cpuinfo` returning empty output saved me from
deploying Frigate and watching it destroy my server's performance. Hardware
constraints aren't limitations you work around — they're facts you design
around. Knowing your hardware's actual capabilities before choosing software
is the first step in any real infrastructure decision.

**Vendor lock-in isn't just about your own choices.**
Google restricting their Home API for third-party cameras isn't something I
could anticipate or fix. In production environments, dependencies on third-
party APIs are risk — they can change, deprecate, or disappear without notice.
The lesson isn't to avoid APIs, it's to understand that every external
dependency is a single point of failure you don't control.

**Mixed camera ecosystems are where unified NVR dreams go to die.**
The Tapo SD card sync works because someone spent significant time reverse-
engineering Tapo's protocol. Ezviz SD card access doesn't work because nobody
has successfully done the same for Ezviz — or because Ezviz has actively
prevented it. When you buy cameras from two different manufacturers, you are
implicitly accepting two different ecosystems. The right lesson for future
hardware purchases: pick one camera brand and stick to it.

**Knowing what NOT to build is a senior engineering skill.**
In three separate paths, I stopped when the evidence said to stop. I didn't
push forward hoping something would change. In real engineering environments,
engineers who keep building toward a failed approach because they're
emotionally invested in the idea cause more damage than the ones who pivot
early. Sunk cost is not a reason to continue.

**End-user experience is a hard requirement, not a nice-to-have.**
The entire point of a smart home system is that non-technical family members
can use it without friction. A technically impressive setup that requires my
parents to learn a new app, understand why one camera works differently than
the others, or call me when something breaks — that's a failed system
regardless of how elegant the architecture looks. UX is a first-class
engineering constraint.

**The native apps won because they were actually better for this use case.**
This is uncomfortable to admit when the whole point is self-hosting, but it's
true. Tapo and Ezviz apps have full SD card playback, sub-second live view,
reliable PTZ, and push notifications. They're maintained by teams of engineers
whose entire job is making those apps work. Trying to replicate that on a Core
2 Duo with open-source software and limited API access was always going to be
a losing battle with this specific hardware and these specific camera brands.

---

## What's Next

This experiment wasn't wasted time. Every path I tried taught me something
real about API design, hardware constraints, vendor ecosystems, and
infrastructure decision-making. None of that would have been clear without
actually building it.

The server's current stack is stable and genuinely useful:

| Service | Status |
|---|---|
| Dockge | Running |
| Nginx Proxy Manager | Running |
| Uptime Kuma | Running |
| Jellyfin | Running |

Next milestone: **Automated rsync backups.** The 2TB drive has 35,000+ hours
on it. Everything that's been built needs to be backed up before that drive
decides it's done. That's the most important thing on the roadmap right now.
