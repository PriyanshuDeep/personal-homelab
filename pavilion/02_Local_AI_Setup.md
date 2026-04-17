# 02 — Local AI Infrastructure & LLM Deployment

## What I Did

Built a complete local AI inference stack on my workstation so I can run LLMs without relying on cloud APIs or hitting rate limits. The goal was to have a production-grade setup where I can get technical help for homelab work 24/7 without depending on external services.

The stack consists of Ollama running bare-metal for direct GPU access, model weights stored on the HDD to protect my NVMe from write wear, and Open WebUI running in Docker as the frontend. Everything is architected the same way as my server setup — fast storage for runtime components, bulk storage for data.

---

## Architecture Decisions

**Ollama on Bare Metal, Not Docker:**
Docker adds a CUDA passthrough layer that can cause headaches with GPU drivers. Since Ollama's entire job is talking to the GPU, I installed it directly on the host OS. This also means if Docker crashes or I break my container stack, Ollama is still running and accessible.

**Model Storage on HDD:**
LLM weights are multi-gigabyte files that rarely change once downloaded. There's zero benefit to keeping them on my NVMe — it just burns write cycles. By routing Ollama's model directory to the 1TB HDD, downloads and model swaps don't touch the SSD at all.

**Open WebUI in Docker:**
The web UI doesn't need GPU access — it's just a frontend that sends HTTP requests to Ollama's API. Running it in Docker makes it easy to update, reset, or replace without touching Ollama itself. The container talks to bare-metal Ollama using host networking mode.

**Storage Split:**
Following the same pattern as my server (NVMe for OS/runtime, HDD for data):
- NVMe: Docker images, container runtime, Compose files
- HDD: Ollama model weights, Open WebUI's database and conversation history

---

## Challenges

**Problem 1: Fedora 43 ships with DNF5, which broke Ollama's official install command.**

Ollama's docs provide a `curl | sh` install script that uses DNF commands written for the old package manager. The syntax changed in DNF5 and the script errored out.

**Problem 2: Open WebUI couldn't reach Ollama.**

After deploying Open WebUI with the standard Docker bridge network setup, the container threw connection errors trying to hit Ollama's API. The `host.docker.internal` DNS hack that works on Mac/Windows doesn't work reliably on Linux. I was getting `curl: (7) Failed to connect` errors from inside the container.

**Problem 3: Default port conflict awareness.**

I initially mapped Open WebUI to port 3000, but in host networking mode this became irrelevant and potentially confusing.

---

## Solutions

**Ollama installation workaround:**
Skipped the package manager entirely and used the official installer script:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

This downloads a pre-built binary and sets up the systemd service automatically. No DNF required.

**Storage routing with systemd override:**
Created a systemd drop-in config to tell Ollama where to store models:

```bash
sudo systemctl edit ollama.service
```

Added this override:

```ini
[Service]
Environment="OLLAMA_MODELS=/mnt/1TB_Storage/ai-models"
```

Applied with:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

Verified the path stuck:

```bash
systemctl show ollama.service | grep OLLAMA_MODELS
```

**Host networking mode for Open WebUI:**
Instead of fighting with Docker's bridge network DNS, I switched to `network_mode: host` in the Compose file. This makes the container use my laptop's network stack directly — `localhost:11434` inside the container is literally localhost on my laptop, where Ollama is listening.

**Directory structure on the HDD:**

```bash
sudo mkdir -p /mnt/1TB_Storage/ai-models
sudo mkdir -p /mnt/1TB_Storage/docker/open-webui/data
sudo chown -R ollama:ollama /mnt/1TB_Storage/ai-models
sudo chown -R $(id -u):$(id -g) /mnt/1TB_Storage/docker/open-webui
```

---

## Docker Compose Configuration

Created `~/docker/open-webui/docker-compose.yml`:

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    network_mode: host
    environment:
      - OLLAMA_BASE_URL=http://localhost:11434
    volumes:
      - /mnt/1TB_Storage/docker/open-webui/data:/app/backend/data
```

Deployed with:

```bash
cd ~/docker/open-webui
docker compose up -d
```

---

## Model Selection & Deployment

**Initial model pull:**

```bash
ollama pull llama3.1
```

This downloaded the 8B parameter version (~4.7GB). Verified it landed on the HDD:

```bash
ls -lh /mnt/1TB_Storage/ai-models/
```

**Specialized technical model:**

After realizing Llama 3.1 is a general-purpose model and not optimized for DevOps work, I pulled a code-specialized alternative:

```bash
ollama pull qwen2.5-coder:7b
```

Qwen 2.5 Coder is trained specifically on code, config files, Linux commands, and technical documentation. It understands Docker Compose syntax, systemd configs, and YAML structure better than a general model.

**Current model inventory:**

```bash
ollama list
```

NAME                    ID              SIZE
qwen2.5-coder:7b        e0d4315bb03b    4.7 GB
llama3.1:latest         42182419e950    4.7 GB

---

## Learnings

**VRAM management — don't over-optimize prematurely.**
I initially considered setting `OLLAMA_KEEP_ALIVE=0` to aggressively unload models from VRAM after every response. But this causes thermal cycling on the GPU — constantly loading and unloading 4GB+ models between requests puts unnecessary wear on the hardware. The default 5-minute timeout is smarter: if I'm actively working, the model stays warm in VRAM. If I take a break, it unloads automatically.

**Host networking is not a compromise, it's the right tool.**
On my server, I use bridge networks with custom Docker networks because multiple containers need to talk to each other (NPM routing to Jellyfin, Uptime Kuma, etc.). Here, only one container needs to reach something outside Docker. Host mode eliminates the complexity of DNS workarounds and port mapping for a single-service deployment.

**Systemd overrides vs editing service files directly.**
Using `systemctl edit` creates a drop-in config in `/etc/systemd/system/ollama.service.d/override.conf` instead of modifying the package-managed service file. This means my custom config survives package updates. If Ollama's installer updates the base service file, my override still applies on top of it.

**Model specialization matters more than size.**
A 7B parameter model trained on code is more useful for technical work than a 70B general model. Qwen 2.5 Coder understands `docker-compose.yml` syntax and can debug `systemctl` errors. Llama 3.1 is better at creative writing and casual conversation. Using the right model for the task is more important than using the biggest one.

---

## Storage Layout
/mnt/1TB_Storage/
├── ai-models/
│   ├── blobs/
│   │   └── sha256-[model weights]
│   └── manifests/
└── docker/
    └── open-webui/
    	└── data/
	├── webui.db
	└── vector_db/

The HDD now holds ~10GB of model weights and will accumulate conversation history and vector embeddings over time. The NVMe stays clean.

---

## Service Access

| Service | URL | Notes |
|---------|-----|-------|
| Ollama API | `http://localhost:11434` | Bare-metal, always running |
| Open WebUI | `http://localhost:8080` | Docker container, host network mode |

---

## Current State

```bash
# Ollama status
systemctl status ollama
● ollama.service - Ollama Service
     Loaded: loaded
     Active: active (running)

# Available models
ollama list
NAME                    SIZE
qwen2.5-coder:7b        4.7 GB
llama3.1:latest         4.7 GB

# Running containers
docker ps
CONTAINER ID   IMAGE                                STATUS
7b53d23d9331   ghcr.io/open-webui/open-webui:main   Up (healthy)
```

---

## Next Steps

- [ ] Set up Knowledge collections in Open WebUI with homelab docs
- [ ] Test model performance on Docker Compose file generation
- [ ] Consider pulling `gemma2:9b` as a secondary model for explanations
- [ ] Monitor HDD space usage as conversation history accumulates


