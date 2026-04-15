# 03 — Zero Trust Network & Global Access (Tailscale)

## What I Did

I wanted to be able to SSH into my server from anywhere — not just my home network. The traditional way to do this is port forwarding: open a port on your router and expose it to the internet. I didn't want to do that. Opening ports on a home router is a real attack surface, and my server is running on hardware that has zero capacity to handle a brute force attack.

Instead I set up a **Zero Trust mesh network using Tailscale**, which runs WireGuard under the hood. Both my laptop and server now have a private Tailscale IP that can reach each other directly from anywhere in the world, without any open ports on my router. I also set up mobile SSH access from my Android phone via Termius for on-the-go troubleshooting.

---

## Architecture Decision: Bare Metal, Not Docker

I deliberately installed Tailscale directly on the OS of both machines — not inside a Docker container. This might seem like an odd choice given that everything else is going to be containerized, but the reasoning is straightforward:

If the Docker engine crashes or I accidentally break my container stack, I still need a way to SSH into the server and fix it. If Tailscale were running inside Docker, a Docker crash would take out my remote access at the same time. By running Tailscale on bare metal, it's completely independent of whatever happens in my container environment. It's always there as a lifeline.

---

## Challenges

**Problem 1: Fedora's DNF5 broke the official Tailscale install command.**

The official Tailscale docs give you a `dnf config-manager --add-repo` command to add their repository. On my Fedora 43 workstation, this threw an error. Fedora 43 ships with `dnf5`, which changed the command syntax from the older `dnf`.

**Problem 2: My phone couldn't authenticate to the server.**

After setting up Termius on my Android for mobile SSH access, I got an authentication error trying to log in. I couldn't figure out why my password wasn't working.

---

## Solutions

**Fixing the DNF5 issue:**
Instead of fighting with the package manager flags, I went around the problem entirely. I used `curl` to download the `.repo` file directly from Tailscale's servers and dropped it into `/etc/yum.repos.d/` manually. After that, `dnf install tailscale` worked perfectly. Sometimes the direct filesystem approach is cleaner than whatever the package manager wrapper is trying to do.

**The phone SSH issue — server was doing its job:**
I almost went down a rabbit hole thinking something was broken. Then I remembered: I had previously set `PasswordAuthentication no` in `sshd_config`. My server was correctly rejecting my password because I told it to. The fix was to generate a fresh Ed25519 keypair inside Termius, copy that public key, and append it to `~/.ssh/authorized_keys` on the server manually.

This actually led me to a policy I'm now deliberately following: **One Device = One Key**. Every device that can access my server has its own unique keypair. If I lose my phone, I delete that specific key from `authorized_keys` without touching anything else. No shared keys, no "I'll just copy my laptop's key to my phone."

---

## Learnings

**Zero Trust means "never assume you're safe because you're on the right network."**
Tailscale's model is that every device authenticates individually using cryptographic identity, regardless of what network it's on. There's no concept of "inside the firewall = trusted." This is how modern enterprise networks are designed, and running it on my homelab means I'm learning that model hands-on.

**WireGuard is the underlying protocol — Tailscale is the control plane.**
Tailscale handles the key exchange, the device registry, and the routing. WireGuard handles the actual encrypted tunnel. Understanding this distinction matters — WireGuard is the engine, Tailscale is the management layer that makes WireGuard practical to run across many devices without manually managing keys.

**The DNF5 workaround taught me something about Linux package management.**
The package manager commands are just wrappers. Underneath, they're working with files in specific directories. When the wrapper breaks, going directly to the filesystem is always an option. That's a mindset shift that's come up more than once already.

---

## Network Configuration

| Device | Hostname | Tailscale IP |
|---|---|---|
| HP Pavilion Laptop | pavilion | `100.102.43.106` |
| Dell Inspiron 560s | inspiron | `100.98.79.12` |
| Android (Termius) | — | Connected via Tailscale |

### SSH Config on `pavilion` (`~/.ssh/config`)

```
Host inspiron
    HostName 100.98.79.12
    User priyanshudeep
    IdentityFile ~/.ssh/id_ed25519
```

Using the Tailscale IP as the `HostName` means `ssh inspiron` works identically whether I'm at home on the local network, on mobile data, or anywhere else in the world.
