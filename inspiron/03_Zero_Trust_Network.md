# Zero Trust Network & Global Access (Tailscale)

## What I Did
I wanted to be able to access my server from anywhere without opening dangerous ports on my Mercusys router. To solve this, I built a Zero Trust mesh network using Tailscale (WireGuard). I installed it directly on the bare metal of my Debian server and my Fedora KDE laptop. I deliberately avoided putting Tailscale in a Docker container because if the Docker engine ever crashes, I would lose all remote SSH access to my server. 

To keep my infrastructure clean, I bound the Tailscale network to my professional Microsoft developer account, keeping it strictly separated from my personal consumer accounts.

## Problems I Faced & How I Solved Them

**Problem 1: Fedora's DNF5 Package Manager**
While trying to add the Tailscale repository on my laptop, the official command (`dnf config-manager --add-repo`) threw an error. I realized that my Fedora Workstation uses the brand-new `dnf5` package manager, which completely changed the command syntax.
* **The Solution:** Instead of fighting the package manager, I interacted with the Linux filesystem directly. I used `curl` to download the official `.repo` file straight into the `/etc/yum.repos.d/` directory. After that, `dnf install tailscale` worked perfectly.

**Problem 2: Mobile SSH Password Rejection**
I set up the Termius app on my phone so I can troubleshoot the server on the go (Incident Response). However, when I tried to log into the server using my password, the connection failed with an authentication error. 
* **The Solution:** I initially thought something was broken, but I realized my server was actually just doing its job! Because I had previously hardened my SSH daemon to reject passwords and only accept cryptographic keys, it was blocking my phone. I fixed this by generating a fresh `Ed25519` keypair directly inside the Termius app, copying the public key, and appending it to the server's `~/.ssh/authorized_keys` file. My server now enforces a strict "One Device = One Key" policy.
