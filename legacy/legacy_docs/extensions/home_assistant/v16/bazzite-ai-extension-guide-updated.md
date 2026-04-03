# Bazzite AI Smart Home — Extension Guide EX.0–EX.6
**AI chat · Voice assistant · Home automation · Nextcloud file access**

SSH-first · Zero New Open Ports · Headless by Default

Hardware: GMKtec NucBox K8 · Ryzen 7 8845HS · Radeon 780M (gfx1103) · 32 GB DDR5

*Updated March 2026*

---

> **■■ INFO — Prerequisite**
>
> The Bazzite Unified Home Server + Gaming Station guide must be fully complete — all Phase 0–11 containers healthy
> (DuckDNS, NPM, Nextcloud running). Specifically confirm:
> - **Phase 2 Step 1** (DHCP reservation) is complete. NPM forwards traffic to your server's LAN IP. If that IP
>   changes — due to a DHCP lease renewal — all external remote access breaks instantly. Verify your router has
>   assigned a permanent static reservation before continuing.
> - **Phase 5 Step 2** (Docker daemon log rotation) is complete. The `daemon.json` sets a global 10 MB / 3-file
>   log cap for all containers. The explicit `logging:` blocks in this guide's compose files match that global policy
>   and provide defense-in-depth — they are not redundant.
> - **Appendix D — Watchtower** is deployed. Every compose file in this guide uses
>   `com.centurylinklabs.watchtower.enable: true/false` labels. Those labels have no effect unless Watchtower is
>   running.
>
> **What this guide adds:** Ollama GPU inference · Open WebUI · mcpo (filesystem + memory MCP tools) · Home
> Assistant · Wyoming voice pipeline (Whisper STT + Piper TTS) · full remote access via HTTPS. No new hardware
> required for EX.0–EX.6.
>
> All Phase 0–11 services (Nextcloud, DuckDNS, NPM) run unchanged. This guide adds a completely separate Docker
> network (`ai-net`) and service stack. No existing containers are modified.

---

## TABLE OF CONTENTS

- ROCm + 780M — Definitive Assessment
- System Architecture
- Components · Network Topology · Port Registry · Data Flow
- Component Rationale & Full Compose Blocks
- EX.0 Pre-flight (~20 min)
- EX.1 Ollama with iGPU (~45 min)
- EX.2 Open WebUI + mcpo (~45 min)
- EX.3 Home Assistant (~60 min)
- EX.4 Voice Pipeline (~30 min)
- EX.5 HA → Ollama Conversation Agent (~20 min)
- EX.6 Remote Access & Mobile Clients (~20 min)
- Post-Execution Validation Checklist
- Critical Pitfalls
- Hardware Appendix (future phases)
- EX.A AI-Notes: Write Access to Nextcloud (~15 min)
- EX.B Smart Model Routing (~20 min)
- EX.C Web Search via Brave (~15 min)

---

## ROCM + 780M — DEFINITIVE ASSESSMENT

The Radeon 780M (gfx1103 / RDNA3) is usable for GPU inference on Bazzite (Fedora 42). Three environment variables
and one kernel parameter are mandatory. Without them, Ollama falls back silently to CPU or the iGPU hangs under
sustained inference.

### Required Kernel Parameter

> **■ CRITICAL**
>
> Apply this BEFORE any other step in this guide. A reboot is required.
> Without `amdgpu.noretry=0` the iGPU can suffer an unrecoverable MES (Micro Engine Scheduler) hang during
> sustained inference, requiring a hard power cycle.

```bash
# Apply kernel parameter and reboot:
sudo rpm-ostree kargs --append=amdgpu.noretry=0
sudo systemctl reboot

# After reboot, verify before continuing:
cat /proc/cmdline | grep -q "amdgpu.noretry=0" && echo "PASS" || echo "FAIL"
# → must print PASS
```

### Required Environment Variables

Both must be set inside the Ollama container's `environment:` block. Setting them in the host shell has no effect on
containers.

| Variable | Value | Effect |
|---|---|---|
| `HSA_OVERRIDE_GFX_VERSION` | `11.0.2` | Bypasses ROCm hardware allowlist. Without it, Ollama silently runs on CPU with no error message. |
| `OLLAMA_FLASH_ATTENTION` | `false` | RDNA3 has a Flash Attention numerical instability bug causing output to collapse into repeated tokens (e.g. `444444...`). Disabling it fixes this with negligible performance impact. |

### BIOS Prerequisite

The 780M uses system RAM as VRAM. A 14B Q4 model requires approximately 9 GB. Verify in
BIOS → APU/Advanced → UMA Frame Buffer Size: must be **Auto** or **≥ 8 GB**. GMKtec NucBox K8 default: Auto.
No change needed if using factory settings.

### Expected Performance

| Mode | Model | Speed | Voice latency |
|---|---|---|---|
| CPU-only fallback | 7B Q4 | ~3–6 tok/s | ~30–40 s end-to-end |
| 780M GPU (target) | 7B Q4 | ~15–20 tok/s | ~8–12 s end-to-end |
| 780M GPU (recommended) | 14B Q4 | ~10–14 tok/s | ~10–14 s end-to-end |

> **■■ INFO**
>
> eGPU note: when an OCuLink eGPU (RX 6000/7000 series) is connected, Ollama automatically selects it with no
> configuration changes required. Use `ROCR_VISIBLE_DEVICES` if explicit device selection is needed.

---

## SYSTEM ARCHITECTURE

### Components

| Service | Image | Folder |
|---|---|---|
| Ollama (GPU inference) | `ollama/ollama:rocm` | `~/docker/ollama` |
| Open WebUI | `ghcr.io/open-webui/open-webui:main` | `~/docker/openwebui` |
| mcpo (MCP proxy) | `ghcr.io/open-webui/mcpo:main` | `~/docker/mcpo` |
| Home Assistant | `ghcr.io/home-assistant/home-assistant:stable` | `~/docker/homeassistant` |
| wyoming-whisper (STT) | `rhasspy/wyoming-whisper:latest` | `~/docker/wyoming` |
| wyoming-piper (TTS) | `rhasspy/wyoming-piper:latest` | `~/docker/wyoming` |
| Mosquitto MQTT (future) | `eclipse-mosquitto:latest` | `~/docker/zigbee` |
| Zigbee2MQTT (future) | `ghcr.io/koenkk/zigbee2mqtt:latest` | `~/docker/zigbee` |

### Docker Network Topology

> **■ CONCEPT**
>
> Create the `ai-net` bridge network once, before deploying any AI stack container. It is a persistent host
> resource — it survives `compose up/down` cycles and reboots.
>
> Command: `docker network create ai-net`

| Network | Members | Reason |
|---|---|---|
| `ai-net` (bridge) | ollama, openwebui, mcpo | Containers communicate by hostname — e.g. `http://ollama:11434` |
| `host` network | homeassistant | Required for mDNS/UPnP device discovery. HA cannot join any bridge network. |
| `127.0.0.1` published ports | wyoming-whisper, wyoming-piper | HA (host network) reaches them via loopback. LAN devices cannot. |

> **■ CONCEPT — Hostname resolution summary:**
>
> - `openwebui → ollama:11434` via `ai-net` (internal Docker hostname)
> - `openwebui → mcpo:8000` via `ai-net` (internal Docker hostname)
> - `homeassistant → 127.0.0.1:11434` loopback — HA is on host network
> - `homeassistant → 127.0.0.1:10300` loopback — wyoming-whisper
> - `homeassistant → 127.0.0.1:10200` loopback — wyoming-piper
> - `NPM → :3000 and :8123` cross-network: use LAN IP, not container name

### Port Registry

| Port | Binding | Service | External Access |
|---|---|---|---|
| 22022 | 0.0.0.0 | SSH | Yes (existing router forward) |
| 80 / 443 | 0.0.0.0 | NPM HTTP/HTTPS | Yes (existing router forward) |
| 11434 | 127.0.0.1 | Ollama API | Loopback only — LAN devices cannot reach it |
| 3000 | 0.0.0.0 | Open WebUI | Via NPM → `ai.*.duckdns.org` |
| 8000 | `ai-net` only | mcpo | Docker-internal — not published to host |
| 8123 | host network | Home Assistant | Via NPM → `home.*.duckdns.org` |
| 10300 | 127.0.0.1 | wyoming-whisper | Loopback only |
| 10200 | 127.0.0.1 | wyoming-piper | Loopback only |
| 1883 (future) | 127.0.0.1 | Mosquitto MQTT | Loopback only |
| 8282 (future) | 127.0.0.1 | Zigbee2MQTT UI | Loopback only |

> **■ NOTE — LAN plaintext trade-off**
>
> Open WebUI binds to `0.0.0.0:3000` because NPM (on `nextcloud-aio` network) cannot reach it by container
> hostname — cross-network hostname resolution does not work between `nextcloud-aio` and `ai-net`. As a result,
> `http://<LAN-IP>:3000` is accessible in plain HTTP on your LAN. Open WebUI has its own authentication; all
> external traffic goes through HTTPS via NPM.
>
> **Mitigation options:** (1) Access Open WebUI exclusively via the HTTPS DuckDNS domain using a local DNS
> override (the same hairpin NAT solution described in Phase 10 Step 3 of the base guide). (2) Use an isolated
> management VLAN. For a single-user trusted home network these risks are accepted as standard trade-offs.

> **■ NOTE**
>
> Port 8123 (Home Assistant) must be explicitly opened in firewalld because HA uses `network_mode: host`, which
> binds directly to the host's network stack and does not benefit from Docker's nftables bypass. This is done in EX.3.
>
> No new router port forwards are required. All external access uses the existing NPM rules.

### Data Flow

```
Browser / Android / Linux laptop
  ↓ HTTPS via DuckDNS + NPM (existing, unchanged)
Open WebUI (ai-net) ←→ mcpo (ai-net) ←→ MCP servers (filesystem, memory)
  ↓ http://ollama:11434 (ai-net internal hostname)
Ollama :rocm [780M iGPU · qwen2.5:14b-instruct-q4_K_M]
  ↑ http://127.0.0.1:11434 (loopback)
Home Assistant (host network)
  ↔ 127.0.0.1:10300 wyoming-whisper (STT)
  ↔ 127.0.0.1:10200 wyoming-piper (TTS)
  ↓ ESPHome voice_assistant over Wi-Fi (future)
ESP32 satellite nodes
```

---

## COMPONENT RATIONALE & FULL COMPOSE BLOCKS

This section provides the authoritative compose files and configuration blocks. They are also shown inline during each
execution phase — you do not need to cross-reference back here during setup.

### Ollama (`ollama/ollama:rocm`)

> **■ CRITICAL**
>
> The `:rocm` tag is mandatory. The untagged `ollama/ollama` image is CPU-only and silently ignores all GPU
> environment variables, including `HSA_OVERRIDE_GFX_VERSION`.

```yaml
# docker-compose.yml (Ollama)
services:
  ollama:
    image: ollama/ollama:rocm
    container_name: ollama
    restart: unless-stopped
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    ports:
      - "127.0.0.1:11434:11434"
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri:/dev/dri
    group_add:
      - VIDEO_GID    # bare integer from EX.0.d, e.g.: - 44
      - RENDER_GID   # bare integer from EX.0.d, e.g.: - 107
    environment:
      - HSA_OVERRIDE_GFX_VERSION=11.0.2
      - OLLAMA_FLASH_ATTENTION=false
      - OLLAMA_KEEP_ALIVE=5m
      - OLLAMA_NUM_PARALLEL=1
      - OLLAMA_HOST=0.0.0.0
    volumes:
      - ./models:/root/.ollama:z
    networks:
      - ai-net

networks:
  ai-net:
    external: true
```

| Setting | Value / Notes |
|---|---|
| `group_add` integers | Must be bare integers — no quotes. Docker Compose YAML resolves group names from the container's own `/etc/group`, not the host's. String names cause startup crashes ("Unable to find group render"). Use the numeric GIDs from EX.0.d. |
| `OLLAMA_KEEP_ALIVE=5m` | Model stays loaded 5 minutes after the last request. Adjust lower if RAM pressure is observed during heavy Nextcloud + HA use. |
| `OLLAMA_NUM_PARALLEL=1` | One concurrent request prevents VRAM exhaustion on the shared-memory iGPU. |
| `OLLAMA_HOST=0.0.0.0` | Ollama must listen on all container interfaces so Open WebUI can reach it by hostname (`ollama`) over `ai-net`. |

> **■ CONCEPT — Recommended model:**
> `qwen2.5:14b-instruct-q4_K_M` — valid Ollama tag, 21M+ downloads, 9.0 GB, 10–14 tok/s on 780M.
> Fallback: `mistral:7b-instruct-q4_K_M` (~5 GB) if available RAM drops below 18 GB.

> **■■ INFO — SELinux `:z` flag on all local bind mounts**
>
> Bazzite runs SELinux in enforcing mode. Directories created in `~/docker/` carry the `user_home_t` context, which
> `container_t` processes cannot access. The `:z` suffix tells Docker to relabel the host directory to
> `container_file_t` before mounting. Every `./...` bind mount in this guide carries `:z` (or `:ro,z` for read-only).
> Paths that already have `container_file_t` from the base guide's `semanage` rule
> (`/var/srv/nextcloud-data`), device mounts (`/dev/kfd`, `/dev/dri`), and system socket mounts
> (`/etc/localtime`, `/run/dbus`) do not need `:z`.

### Open WebUI (`ghcr.io/open-webui/open-webui:main`)

```yaml
# docker-compose.yml (Open WebUI)
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: openwebui
    restart: unless-stopped
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - WEBUI_SECRET_KEY=<paste output of: openssl rand -hex 32>
    volumes:
      - ./data:/app/backend/data:z
    networks:
      - ai-net

networks:
  ai-net:
    external: true
```

> **■ NOTE**
> `WEBUI_SECRET_KEY`: run `openssl rand -hex 32` in your terminal first and paste the resulting 64-character hex
> string. Do not use the placeholder text. Remove angle brackets.

### mcpo (`ghcr.io/open-webui/mcpo:main`)

mcpo is a Python server that proxies MCP (Model Context Protocol) servers over HTTP/OpenAPI, making them available
to Open WebUI as tool servers. It runs `npx` to launch Node.js-based MCP packages. The mcpo Docker image includes
Node.js for this purpose.

> **■ CRITICAL**
>
> mcpo requires the explicit startup command: `--config /app/config/config.json`
> Without it the container starts but loads no MCP servers and exposes no tools.

```json
// config.json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/nc-data"]
    },
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"],
      "env": {
        "MEMORY_FILE_PATH": "/app/memory/graph.json"
      }
    }
  }
}
```

> **■ CONCEPT**
> `MEMORY_FILE_PATH` is an environment variable consumed by the memory server package — not a CLI argument.
> Without it the memory graph is written to a temporary location and lost on every container restart.

```yaml
# docker-compose.yml (mcpo)
services:
  mcpo:
    image: ghcr.io/open-webui/mcpo:main
    container_name: mcpo
    restart: unless-stopped
    command: --config /app/config/config.json
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    user: "NC_UID:NC_GID"    # e.g. "33:33" — from EX.0.g, no angle brackets
    environment:
      - npm_config_cache=/tmp/npm_cache
      - HOME=/tmp
    volumes:
      - ./config.json:/app/config/config.json:ro,z
      - /var/srv/nextcloud-data/<your_nc_username>/files:/nc-data:ro
      - ./memory:/app/memory:z
      - ./npm_cache:/tmp/npm_cache:z
    networks:
      - ai-net

networks:
  ai-net:
    external: true
```

| Setting | Reason |
|---|---|
| `user: "33:33"` | Runs mcpo as the Nextcloud data owner so it can read user files. Paste the numeric UID:GID from EX.0.g directly, with no angle brackets. |
| `npm_config_cache=/tmp/npm_cache` | Prevents EACCES: `npx` writes cache relative to the user home directory, which does not exist for a non-root UID without a `/etc/passwd` entry. |
| `./npm_cache:/tmp/npm_cache` | Persists the `npx` download cache across container restarts. Without this, the `@modelcontextprotocol` packages are re-downloaded from npm on every reboot. |
| `HOME=/tmp` | Node and npm resolve `HOME` from `/etc/passwd`. An arbitrary UID (e.g. 33) has no entry in the container's `/etc/passwd`, causing EACCES crashes. `HOME=/tmp` provides a writable virtual home directory. Mandatory. |
| `/var/srv/nextcloud-data/<user>/files:/nc-data:ro` | Scopes AI access to one user's files directory. `:ro` prevents any AI write. |
| `./memory:/app/memory` | Persists the `memory/graph.json` across restarts. Must be pre-created with correct ownership before first `docker compose up` — see EX.2. |

> **■ CRITICAL**
>
> The `./memory` and `./npm_cache` directories must be pre-created with correct ownership BEFORE `docker compose up`.
> Docker auto-creates missing bind-mount directories as `root:root`. mcpo running as a non-root UID cannot write to
> root-owned directories, causing silent memory failure and EACCES errors on first run.
>
> ```bash
> mkdir -p ~/docker/mcpo/memory ~/docker/mcpo/npm_cache
> sudo chown <NC_UID>:<NC_GID> ~/docker/mcpo/memory ~/docker/mcpo/npm_cache
> ```

### Home Assistant (`ghcr.io/home-assistant/home-assistant:stable`)

```yaml
# docker-compose.yml (Home Assistant)
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    restart: unless-stopped
    network_mode: host
    labels:
      - "com.centurylinklabs.watchtower.enable=false"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    environment:
      - TZ=Europe/Berlin    # adjust to your timezone
    volumes:
      - ./config:/config:z
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
```

| Setting | Reason |
|---|---|
| `network_mode: host` | Required for mDNS/UPnP device discovery. HA cannot join bridge networks. Do not add a `ports:` block — it is incompatible with host network mode. |
| `privileged: true` (absent) | NOT required for HA Container mode. Required only by HA Supervisor. Omitting it is the correct minimal-privilege posture. |
| `watchtower.enable: "false"` | Prevents Watchtower from restarting HA mid-session. Update HA manually. |
| `/run/dbus:ro` | Optional for this core build. Required for future Bluetooth device tracking. `:ro` limits scope to read-only socket access. |

> **■ CRITICAL**
>
> HA add-ons do not exist in Container mode. Do not follow any tutorial referencing the Supervisor, Add-on Store, or
> named add-ons — those apply to HAOS or Supervised installations only.

### Voice Pipeline: wyoming-whisper + wyoming-piper (`~/docker/wyoming`)

Both services in one `docker-compose.yml`. Ports bound to `127.0.0.1` — Home Assistant (host network) reaches them
via the shared loopback. LAN devices cannot reach these ports directly.

```yaml
# docker-compose.yml (Wyoming)
services:
  wyoming-whisper:
    image: rhasspy/wyoming-whisper:latest
    container_name: wyoming-whisper
    restart: unless-stopped
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    ports:
      - "127.0.0.1:10300:10300"
    volumes:
      - ./whisper-data:/data:z
    command: --model base --language en

  wyoming-piper:
    image: rhasspy/wyoming-piper:latest
    container_name: wyoming-piper
    restart: unless-stopped
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    ports:
      - "127.0.0.1:10200:10200"
    volumes:
      - ./piper-data:/data:z
    command: --voice en_US-lessac-high
```

> **■ NOTE**
> Whisper model options: `tiny` (fastest, less accurate) · `base` (good balance for home commands) · `small` (better
> for accented speech) · `medium` (best accuracy, higher RAM use).
> Change `--model base` to `--model small` for better recognition if accuracy is insufficient.

### NPM Proxy Requirements

NPM cannot reach Open WebUI or Home Assistant by container hostname — they are on different networks. Use the
server's LAN IP as the forward target for both.

| Proxy Host | Forward Host | Port | Required Settings |
|---|---|---|---|
| `ai.yourdomain.duckdns.org` | `<LAN-IP>` | 3000 | WebSockets: ON · SSL: Let's Encrypt · Force SSL: ON · Cache Assets: ON · Block Exploits: ON |
| `home.yourdomain.duckdns.org` | `<LAN-IP>` | 8123 | WebSockets: ON · SSL: Let's Encrypt · Force SSL: ON · Cache Assets: ON · Block Exploits: ON |

> **■■ WARNING**
>
> **WebSockets Support must be ON for both proxy hosts.**
> Open WebUI uses WebSockets for AI response streaming. HA uses WebSockets for its real-time event bus. Turning
> either off breaks core functionality.

### Ollama Conversation Agent in Home Assistant

HA runs on host network and cannot resolve Docker container hostnames. Use `127.0.0.1`, not the container name
`ollama`.

> **■ CONCEPT**
>
> Settings → Devices & Services → Add Integration → **Ollama**
>
> URL: `http://127.0.0.1:11434` → Submit
>
> Then: Model: `qwen2.5:14b-instruct-q4_K_M` · Enable "Control Home Assistant" · Context window: 8192

---

## IMPLEMENTATION PHASES

> **■ CRITICAL — Phase 0 — Kernel Parameter — Mandatory — Requires Reboot**
>
> This is the first step. Do not begin EX.0–EX.6 until the kernel parameter is confirmed active.
>
> ```bash
> sudo rpm-ostree kargs --append=amdgpu.noretry=0
> sudo systemctl reboot
>
> # After reboot, verify before continuing:
> cat /proc/cmdline | grep -q "amdgpu.noretry=0" && echo "PASS" || echo "FAIL"
> # → must print PASS. Do not proceed until it does.
> ```
>
> This is the only persistent system-level change in this entire guide. It is fully reversible at any time — no data
> loss, no service disruption — with:
> ```bash
> rpm-ostree kargs --delete=amdgpu.noretry=0 && sudo systemctl reboot
> ```

---

## EX.0 — Pre-flight (~20 min)

All values gathered here feed directly into compose files. Record each output.

**Step a — Verify existing containers are healthy**

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
# → duckdns, npm, nextcloud-aio-mastercontainer: all "Up"
```

**Step b — Confirm kernel parameter (Phase 0 must be done first)**

```bash
cat /proc/cmdline | grep -q "amdgpu.noretry=0" && echo "PASS" || echo "FAIL"
```

**Step c — Verify GPU device nodes exist**

```bash
ls -la /dev/kfd /dev/dri/renderD128
# → both files must be present
```

**Step d — Retrieve numeric GIDs for Ollama `group_add` — record both**

```bash
getent group video | cut -d: -f3   # → e.g. 44
getent group render | cut -d: -f3  # → e.g. 107
```

> **■ CONCEPT**
>
> `cut -d: -f3` extracts only the third colon-delimited field (the numeric GID). Record both numbers — paste them as
> bare integers into the Ollama compose file, with no quotes. Example: `- 44` not `- "44"`

**Step e — Verify available RAM**

```bash
free -h | awk '/^Mem:/{print $7}'
# → aim for ≥ 18 GB available (includes reclaimable cache) before starting Ollama
```

**Step f — Verify BIOS UMA Frame Buffer Size**

Boot into BIOS → APU/Advanced → UMA Frame Buffer Size. Must be **Auto** or **≥ 8 GB**. GMKtec NucBox K8 factory
default: Auto. If set to a small fixed value, change it to Auto and save.

**Step g — Retrieve Nextcloud data directory owner UID:GID — record both**

```bash
sudo ls -lnd /var/srv/nextcloud-data
# -n: forces numeric UID/GID output (not usernames)
# -d: lists the directory itself, not its contents
# → note the third (UID) and fourth (GID) fields
# → example: drwxrwx--- 1 33 33 ... means UID=33, GID=33
```

**Step h — Identify your Nextcloud username**

```bash
sudo ls /var/srv/nextcloud-data/
# → your username appears as a subdirectory alongside appdata_* and others
```

**Step i — Verify Nextcloud SSE is disabled**

Nextcloud web UI → Settings → Administration → Security → "Server-side encryption" must be off. If SSE is enabled,
the filesystem tool returns only unreadable encrypted blobs.

**Step j — Create shared Docker network**

```bash
docker network create ai-net
# "network with name ai-net already exists" is fine — no action needed.
```

---

## EX.1 — Ollama with iGPU (~45 min — mostly model download)

Create the directory:

```bash
mkdir -p ~/docker/ollama && cd ~/docker/ollama
```

Write `docker-compose.yml`. Replace `44` and `107` with your actual numeric GIDs from EX.0.d:

```bash
# Replace 44 (VIDEO_GID) and 107 (RENDER_GID) with your actual values from EX.0.d.
# Use bare integers — no quotes around the numbers.
cat > docker-compose.yml << 'EOF'
services:
  ollama:
    image: ollama/ollama:rocm
    container_name: ollama
    restart: unless-stopped
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    ports:
      - "127.0.0.1:11434:11434"
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri:/dev/dri
    group_add:
      - 44    # ← replace with your VIDEO_GID
      - 107   # ← replace with your RENDER_GID
    environment:
      - HSA_OVERRIDE_GFX_VERSION=11.0.2
      - OLLAMA_FLASH_ATTENTION=false
      - OLLAMA_KEEP_ALIVE=5m
      - OLLAMA_NUM_PARALLEL=1
      - OLLAMA_HOST=0.0.0.0
    volumes:
      - ./models:/root/.ollama:z
    networks:
      - ai-net

networks:
  ai-net:
    external: true
EOF
```

> **■ CRITICAL**
>
> After writing the file, edit the `group_add` integers to match your actual GIDs:
>
> ```bash
> nano docker-compose.yml
> ```
>
> Replace `44` and `107` with the output of EX.0.d. Remove angle brackets. Integers must be bare — no quotes, no
> letter prefix.

Start Ollama:

```bash
docker compose up -d
```

> **■ NOTE — Model directory ownership**
>
> `~/docker/ollama/models/` is written by the Ollama container which runs as root internally. The host directory will be
> owned by `root:root`. This is harmless for normal operation but means you need `sudo` to delete models manually:
> `sudo rm -rf ~/docker/ollama/models/`. Alternatively, use `docker exec ollama ollama rm <model>` — this handles
> deletion from inside the container with correct permissions.

Pull the recommended model:

```bash
# Pull the recommended model (~9.0 GB, one-time — this will take several minutes)
docker exec ollama ollama pull qwen2.5:14b-instruct-q4_K_M
```

Verify GPU offload:

```bash
# Run this DURING an active generation (trigger one in the next step first)
docker exec ollama ollama ps
# → PROCESSOR column must show "100% GPU"

# If it shows CPU — confirm HSA_OVERRIDE_GFX_VERSION is set:
docker inspect ollama | grep -A1 HSA_OVERRIDE
```

Functional test:

```bash
curl http://127.0.0.1:11434/api/generate \
  -d '{"model":"qwen2.5:14b-instruct-q4_K_M","prompt":"Hello","stream":false}'
# → JSON response with a "response" field
```

> **■ NOTE**
>
> First-request latency: the initial request after starting Ollama takes up to 30 seconds while the model loads into
> VRAM. This is expected and normal. All subsequent requests respond in ~8–12 seconds at 780M GPU speed.
>
> PDF copy-paste warning: if copying the `curl` command from a PDF, verify that all quote characters are straight ASCII
> quotes. Some PDF viewers silently convert these to typographic curly quotes, which break JSON parsing. When in
> doubt, type the command manually rather than pasting it.

LAN isolation test — run from another device:

```bash
# Run from another device on your LAN.
# Must return 000 (connection refused). Returns 200 = FAIL (port is exposed).
curl -s -o /dev/null -w "%{http_code}" http://<server-LAN-IP>:11434
# → Must return 000
```

---

## EX.2 — Open WebUI + mcpo (~45 min)

### Open WebUI — Deploy Container

**Step 1 — Create directory and generate secret key:**

```bash
mkdir -p ~/docker/openwebui && cd ~/docker/openwebui

# Generate a secret key — copy the FULL output (64 hex characters):
openssl rand -hex 32
```

Copy the 64-character string that was output. You will paste it into the compose file below.

**Step 2 — Write `docker-compose.yml`:**

```bash
cat > docker-compose.yml << 'EOF'
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: openwebui
    restart: unless-stopped
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - WEBUI_SECRET_KEY=PASTE_YOUR_64_CHAR_KEY_HERE
    volumes:
      - ./data:/app/backend/data:z
    networks:
      - ai-net

networks:
  ai-net:
    external: true
EOF
```

> **■ CRITICAL**
>
> Open the file and replace `PASTE_YOUR_64_CHAR_KEY_HERE` with the actual key you generated above:
>
> ```bash
> nano docker-compose.yml
> ```
>
> The key must be the raw 64-character hex string — no quotes needed in the YAML value, no angle brackets.

**Step 3 — Start Open WebUI:**

```bash
docker compose up -d
```

### Open WebUI — Initial Account Setup

> **■■ INFO**
>
> Open WebUI has no pre-created accounts. The first account you register automatically becomes the administrator.
> All subsequent registrations require admin approval. Complete this setup before connecting mcpo.

Open a browser on any device on your LAN and navigate to:

```
http://<server-LAN-IP>:3000
```

You will see the Open WebUI welcome screen. Follow these steps to create your admin account:

1. Click **Get Started** (or **Sign Up** if shown).
2. Enter your Name, a valid Email address, and a strong Password. These are stored locally on your server — no external services are contacted.
3. Click **Create Account**.
4. You are now logged in as administrator. Your avatar appears in the top-right corner.

> **■ NOTE**
>
> If you see "Account pending approval" after registering, it means someone else already created the first account.
> Log in with that first account to approve additional users. On a fresh single-user install this will not happen.

Verify Ollama is connected — you should see your model in the model selector at the top of the chat page. If no model
appears, check that the Ollama container is running and the `OLLAMA_BASE_URL` environment variable is set correctly.

### mcpo — Deploy Container

**Step 1 — Create directory:**

```bash
mkdir -p ~/docker/mcpo && cd ~/docker/mcpo
```

**Step 2 — Write `config.json`:**

```bash
cat > config.json << 'EOF'
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/nc-data"]
    },
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"],
      "env": {
        "MEMORY_FILE_PATH": "/app/memory/graph.json"
      }
    }
  }
}
EOF
```

**Step 3 — Pre-create directories with correct ownership:**

```bash
# MANDATORY before compose up. Replace <NC_UID> and <NC_GID> with numeric
# values from EX.0.g (e.g. 33 and 33). No angle brackets.
mkdir -p ~/docker/mcpo/memory ~/docker/mcpo/npm_cache
sudo chown <NC_UID>:<NC_GID> ~/docker/mcpo/memory ~/docker/mcpo/npm_cache
```

> **■ CRITICAL**
>
> These `chown` commands must run before `docker compose up`. Docker auto-creates missing bind-mount directories
> as `root:root`. mcpo running as a non-root UID cannot write to root-owned directories — causing silent memory
> failure and EACCES errors on startup.

**Step 4 — Write `docker-compose.yml`:**

Substitute your actual values before saving. Use the numeric UID, GID, and username from EX.0.g and EX.0.h.

```bash
nano docker-compose.yml
```

Content to type into `docker-compose.yml`:

```yaml
services:
  mcpo:
    image: ghcr.io/open-webui/mcpo:main
    container_name: mcpo
    restart: unless-stopped
    command: --config /app/config/config.json
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    user: "33:33"   # ← replace with your NC_UID:NC_GID
    environment:
      - npm_config_cache=/tmp/npm_cache
      - HOME=/tmp
    volumes:
      - ./config.json:/app/config/config.json:ro,z
      - /var/srv/nextcloud-data/alice/files:/nc-data:ro   # ← replace "alice" with your NC_USER
      - ./memory:/app/memory:z
      - ./npm_cache:/tmp/npm_cache:z
    networks:
      - ai-net

networks:
  ai-net:
    external: true
```

After typing, save and exit nano with `Ctrl+O` → Enter → `Ctrl+X`.

**Step 5 — Start mcpo:**

```bash
docker compose up -d
```

**Step 6 — Verify mcpo started (wait 60 s first):**

```bash
# Wait at least 60 seconds on first run (npx downloads packages).
# Then check:
docker logs mcpo | grep -E "Successfully connected|Application startup complete"
# → "Successfully connected to: filesystem" must appear
# → "Successfully connected to: memory" must appear
# → "INFO: Application startup complete." must appear

# Verify Node.js is present in the image:
docker exec mcpo node --version && docker exec mcpo npx --version
# → both must return version strings
```

### mcpo — Connect to Open WebUI

> **■ CONCEPT**
>
> Connecting Open WebUI to mcpo requires registering each MCP server at its individual path. The root URL
> (`http://mcpo:8000`) exposes no tool definitions. You must register each tool server separately at its sub-path.
>
> Tools added through the Admin Panel are **Global Tool Servers**. They are accessible to all users but must be
> explicitly activated in each chat session by the user.

Navigate to Open WebUI in your browser, then:

1. Click your user avatar (top-right corner) → **Admin Panel**.
2. In the Admin Panel, click **Settings** (left sidebar) → **Tools**.

Register the filesystem tool server:
1. Click **+** (Add Tool Server).
2. URL: `http://mcpo:8000/filesystem` · Type: OpenAPI · Auth: None
3. Click **Save**. A green status dot confirms the connection.

Register the memory tool server:
1. Click **+** again.
2. URL: `http://mcpo:8000/memory` · Type: OpenAPI · Auth: None
3. Click **Save**.

> **■■ INFO**
>
> Global tool servers are not shown automatically in the chat input area. To use a tool in a conversation:
> 1. Open a new chat.
> 2. Click the **■** button in the message input area (bottom-left of the chat box).
> 3. Toggle on the `filesystem` and/or `memory` tool to activate it for that session.
> The tools remain active for the duration of that chat. You must re-enable them in each new chat session.

### Add NPM Proxy Host for Open WebUI

> **■ NOTE**
>
> Access NPM via SSH tunnel. The admin panel is bound to localhost only.
> From your personal computer: `ssh -N -L 8081:127.0.0.1:81 -p 22022 username@<server-LAN-IP>`
> Then open `http://localhost:8081` in your browser.

1. NPM → Hosts → Proxy Hosts → **Add Proxy Host**.
2. Domain Names: `ai.yourdomain.duckdns.org`
3. Forward Hostname: `<server-LAN-IP>` · Port: `3000`
4. Turn **WebSockets Support ON**.
5. SSL tab: Request new certificate → Force SSL ON → agree to Terms of Service → Save.

### EX.2 Functional Tests

```bash
# In Open WebUI chat (with filesystem tool active via ■ button):
# Test 1: "List the files in /nc-data"
# → AI returns your Nextcloud files

# Test 2: "Remember that my test value is 42"
# → AI confirms storage

# Test 3: "What is my test value?"
# → returns 42

# Test 4 — memory persistence across restarts:
cd ~/docker/mcpo && docker compose restart mcpo
# Then in Open WebUI: "What is my test value?" → still returns 42
```

---

## EX.3 — Home Assistant (~60 min)

**Step 1 — Create directory and capture timezone:**

```bash
mkdir -p ~/docker/homeassistant && cd ~/docker/homeassistant

# Capture your timezone:
TZ_VALUE=$(timedatectl show -p Timezone --value)
echo "Using timezone: $TZ_VALUE"
```

**Step 2 — Write `docker-compose.yml`:**

```bash
# The unquoted EOF allows ${TZ_VALUE} to expand to your actual timezone string.
cat > docker-compose.yml << EOF
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    restart: unless-stopped
    network_mode: host
    labels:
      - "com.centurylinklabs.watchtower.enable=false"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    environment:
      - TZ=${TZ_VALUE}
    volumes:
      - ./config:/config:z
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
EOF
```

**Step 3 — Open port 8123 in firewalld, then start Home Assistant**

> **■ CRITICAL**
>
> Home Assistant uses `network_mode: host` — it binds directly to the host's network stack, completely bypassing
> Docker's nftables rules. Port 8123 must be opened explicitly in firewalld or HA will be unreachable from the LAN
> and from NPM.
>
> Reloading firewalld flushes Docker's nftables injection, taking Nextcloud and NPM temporarily offline. Docker must
> be restarted immediately after to restore their networking. This causes ~10–15 seconds of Nextcloud downtime.
>
> Run the firewalld steps **first**, then start HA. This is the correct order — do not run `docker compose up -d`
> before the firewall is open.

```bash
# Step A — Open port 8123:
sudo firewall-cmd --permanent --add-port=8123/tcp

# Step B — Reload firewalld (Nextcloud/NPM drop briefly here):
sudo firewall-cmd --reload

# Step C — Restart Docker to restore nftables rules for all bridge containers:
sudo systemctl restart docker

# Step D — Wait ~15 s, then confirm existing containers are back up:
sleep 15 && docker ps --format "table {{.Names}}\t{{.Status}}"
# → duckdns, nginx-proxy-manager, nextcloud-aio-mastercontainer: all "Up"

# Step E — Now start Home Assistant:
docker compose up -d

# Watch HA start up:
docker logs -f homeassistant | head -30
# Press Ctrl+C once you see "Home Assistant initialized"
```

**Step 4 — Complete HA Onboarding (UI)**

> **■ CRITICAL**
>
> Do NOT edit `configuration.yaml` yet. HA generates this file during onboarding. Editing or creating it before
> onboarding completes causes a file conflict that prevents HA from booting.

From any browser on your LAN, navigate to:

```
http://<server-LAN-IP>:8123
```

1. The onboarding wizard opens. Click **Create my smart home**.
2. Enter your name, username, and password for the HA admin account. Click **Create Account**.
3. Enter your home location (used for timezone and sunrise/sunset). Click **Next**.
4. Select which analytics to share (or opt out). Click **Next**.
5. Complete the wizard. The HA dashboard appears.
6. Wait until the HA dashboard is fully visible before proceeding to Step 5.

**Step 5 — Configure Reverse Proxy Trust**

> **■■ WARNING — YAML indentation**
>
> YAML is strictly space-sensitive. Every level of indentation uses exactly 2 spaces. Do not use tabs. If copying
> from a PDF, verify each indent level after pasting — some PDF viewers silently strip or alter leading whitespace. A
> single mis-indented line causes HA to crash with a YAML parse error on next start. When in doubt, retype the
> indentation manually rather than pasting it.

```bash
# Only after onboarding completes and the dashboard is visible:
nano ~/docker/homeassistant/config/configuration.yaml
```

Add the `http:` block (or insert into an existing `http:` block if present):

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 10.0.0.0/8        # Class A LANs — Unifi, prosumer routers
    - 172.16.0.0/12     # standard Docker bridge range (172.x.x.x)
    - 192.168.0.0/16    # standard home LAN range
    - 127.0.0.1         # loopback
```

**Step 6 — Validate config and restart:**

```bash
# Validate before restarting — a single YAML indentation error prevents HA from booting:
docker exec homeassistant python -m homeassistant \
  --script check_config --config /config
# → must print "Configuration is valid!"

# Restart only after validation passes:
cd ~/docker/homeassistant && docker compose restart
```

> **■■ WARNING — Crash-loop recovery**
>
> If you restart HA with a YAML error and the container enters a crash-loop, `check_config` cannot run because the
> container is down. Recovery path:
> 1. Fix the YAML error in `configuration.yaml`.
> 2. Run: `cd ~/docker/homeassistant && docker compose up -d`
>    (`up -d` re-reads the config and starts the container; `restart` does not help if the container is stopped).
> 3. Once running, run `check_config` to confirm the fix, then restart normally.

**Step 7 — Add NPM Proxy Host for Home Assistant**

> **■ NOTE**
> Open the SSH tunnel to NPM if you closed it:
> `ssh -N -L 8081:127.0.0.1:81 -p 22022 username@<server-LAN-IP>`

1. NPM → Hosts → Proxy Hosts → **Add Proxy Host**.
2. Domain Names: `home.yourdomain.duckdns.org`
3. Forward Hostname: `<server-LAN-IP>` · Port: `8123`
4. Turn **WebSockets Support ON**.
5. SSL tab: Request new certificate → Force SSL ON → agree to ToS → Save.

**Step 8 — Set HA External URL:**

HA → Settings → System → Network → External URL → set to: `https://home.yourdomain.duckdns.org`

**Step 9 — Verify remote access:**

```bash
# Navigate to https://home.yourdomain.duckdns.org
# → HA dashboard loads with no "Unable to connect" error toast

# If 400 Bad Request: check docker logs nginx-proxy-manager for source IP.
# Confirm trusted_proxies covers it. The 10.0.0.0/8 range covers Unifi LANs.
```

---

## EX.4 — Voice Pipeline (~30 min)

**Step 1 — Create directory:**

```bash
mkdir -p ~/docker/wyoming && cd ~/docker/wyoming
```

**Step 2 — Write `docker-compose.yml`:**

```bash
cat > docker-compose.yml << 'EOF'
services:
  wyoming-whisper:
    image: rhasspy/wyoming-whisper:latest
    container_name: wyoming-whisper
    restart: unless-stopped
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    ports:
      - "127.0.0.1:10300:10300"
    volumes:
      - ./whisper-data:/data:z
    command: --model base --language en

  wyoming-piper:
    image: rhasspy/wyoming-piper:latest
    container_name: wyoming-piper
    restart: unless-stopped
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    ports:
      - "127.0.0.1:10200:10200"
    volumes:
      - ./piper-data:/data:z
    command: --voice en_US-lessac-high
EOF
```

**Step 3 — Start Wyoming services:**

```bash
docker compose up -d

# Verify both containers started:
docker ps | grep wyoming
# → wyoming-whisper and wyoming-piper must both show "Up"
```

**Step 4 — Add Wyoming Integrations in Home Assistant**

You must add the Wyoming Protocol integration **twice** — once for Whisper (STT) and once for Piper (TTS).

Add **Faster Whisper** (Speech-to-Text):
1. HA → Settings → Devices & Services → **Add Integration**.
2. Search for and select **Wyoming Protocol**.
3. Host: `127.0.0.1` · Port: `10300`
4. Click **Submit**. HA detects Faster Whisper and names the integration accordingly.

Add **Piper** (Text-to-Speech):
1. HA → Settings → Devices & Services → **Add Integration** again.
2. Search for and select **Wyoming Protocol**.
3. Host: `127.0.0.1` · Port: `10200`
4. Click **Submit**. HA detects Piper and names the integration accordingly.

> **■ NOTE**
>
> Both integrations must show a green checkmark in Settings → Devices & Services. If either shows a red error,
> confirm the wyoming containers are running (`docker ps | grep wyoming`) and that HA is using
> `network_mode: host`.

**Step 5 — Create the Assist Pipeline**

1. HA → Settings → Voice Assistants → **Add Assistant** (the **+** button).
2. Name: `Wall-E`
3. Language: select your preferred language.
4. Conversation Agent: leave as **Home Assistant** for now (will be changed in EX.5).
5. Speech-to-text: select `faster-whisper`.
6. Text-to-speech: select `piper`.
7. Click **Create**.

**Step 6 — STT Test**

Verify transcription before proceeding to EX.5:

1. HA → Settings → Voice Assistants → **Wall-E**.
2. Click the three-dot menu → **Debug** (or open Assist from the dashboard).
3. Click the microphone icon and speak a short sentence.
4. Verify the transcript appears correctly in the panel.

---

## EX.5 — HA → Ollama Conversation Agent (~20 min)

**Step 1 — Add the Ollama Integration**

> **■ CONCEPT**
>
> HA uses `127.0.0.1` because it is on host network and cannot resolve the container hostname `ollama` — only
> `ai-net` bridge containers share that DNS namespace.

1. HA → Settings → Devices & Services → **Add Integration**.
2. Search for and select **Ollama**.
3. URL: `http://127.0.0.1:11434`. Click **Submit**.
4. Model: `qwen2.5:14b-instruct-q4_K_M`
5. Instructions (system prompt): paste the text below (or leave blank for now).
6. Enable the **"Control Home Assistant"** checkbox.
7. Context window: `8192`
8. Keep alive: `5m` (matches Ollama's `OLLAMA_KEEP_ALIVE` setting)
9. Click **Submit**.

> **■ NOTE**
>
> If you need to change any setting after initial setup: Settings → Devices & Services → Ollama → **Configure**.
> All fields are editable from the Configure screen.

Optional system prompt:

```
You are Wall-E, a helpful home assistant. Answer concisely.
Report device states accurately when asked.
```

**Step 2 — Assign Ollama to the Wall-E Assist Pipeline**

1. HA → Settings → Voice Assistants → **Wall-E**.
2. Click the three-dot menu → **Edit**.
3. Conversation Agent: select **Ollama** (`qwen2.5:14b...`) from the dropdown.
4. STT and TTS remain unchanged (Faster Whisper and Piper).
5. Click **Update**.

**Step 3 — End-to-End Tests**

```bash
# Typed test:
# HA → Settings → Voice Assistants → Wall-E → three-dot → Debug
# Type: "What time is it?"
# → Ollama responds, Piper speaks. First query ~15 s. Subsequent: ~8–12 s.

# Voice test:
# Browser mic in HA Assist. Speak a command.
# → Verify: transcript appears → AI text response → Piper voice output.
```

---

## EX.6 — Remote Access & Mobile Clients (~20 min)

### Verify HTTPS from External Network

```bash
# From any browser or phone NOT on your home Wi-Fi (use mobile data):
# https://ai.yourdomain.duckdns.org  → Open WebUI loads
# https://home.yourdomain.duckdns.org → HA loads without errors
```

### Install HA Companion App (Android)

1. Google Play → search **Home Assistant** → Install.
2. Open the app → **Get Started**.
3. Enter your server URL: `https://home.yourdomain.duckdns.org`.
4. Log in with your HA admin credentials.
5. Settings → Companion App → Manage sensors — allow location if desired.
6. Settings → Companion App → Voice → Enable Assist.
7. Optionally enable microWakeWord for hands-free activation.
8. Test: say the wake word → issue a command → AI responds via phone speaker.

### Open WebUI on Android (PWA)

1. Open Chrome on Android.
2. Navigate to `https://ai.yourdomain.duckdns.org` and log in.
3. Tap the Chrome three-dot menu → **Add to Home Screen**.
4. Open WebUI installs as a PWA-style app on your home screen.
5. First time you use the microphone in a chat, Chrome will prompt for microphone permission — allow it.

> **■ CONCEPT — ■■ CORE BUILD COMPLETE ■■**
>
> At the end of EX.6:
> - ✓ AI chat via browser on any device, anywhere over HTTPS
> - ✓ Voice via Android Companion App (hands-free, on-phone wake word)
> - ✓ Persistent memory (store, recall, delete — survives restarts)
> - ✓ Nextcloud file access (plain text, PDFs, config files)
> - ✓ HA voice assistant with Ollama conversation agent
> - ✓ HA device control via AI (turn on the living room light)
> - ✓ All main guide containers (Nextcloud, DuckDNS, NPM) unchanged

---

## POST-EXECUTION VALIDATION CHECKLIST

Run in order. Each check must pass before the next is meaningful.

**1. Kernel parameter active**

```bash
cat /proc/cmdline | grep -q "amdgpu.noretry=0" && echo "PASS" || echo "FAIL"
# → Must print PASS.
```

**2. ai-net exists**

```bash
docker network ls | grep -q ai-net && echo "PASS" || echo "FAIL"
# → Must print PASS.
```

**3. GPU inference confirmed** (run during an active generation)

```bash
docker exec ollama ollama ps | grep -q "100% GPU" && echo "PASS" || echo "FAIL"
# → Must print PASS. Trigger a generation in Open WebUI first.
```

**4. GPU devices accessible inside the container**

```bash
docker exec ollama ls -la /dev/kfd /dev/dri/renderD128
# → Both files must be present inside the container.
```

**5. LAN isolation: Ollama not reachable from LAN** (run from another device)

```bash
# Run this from another machine on your LAN — NOT the server itself.
# 000 = connection refused (correct). 200 = FAIL — Ollama is dangerously exposed.
curl -s -o /dev/null -w "%{http_code}" http://<server-LAN-IP>:11434
# → Must return 000
```

**6. SELinux volume contexts correct**

```bash
# Returns empty if every local volume dir has container_file_t.
# Any output = a :z flag is missing from a compose file.
ls -Zd ~/docker/*/* 2>/dev/null | grep -v container_file_t
# → Must return no output (empty)
```

**7. Firewalld has port 8123 open for Home Assistant**

```bash
sudo firewall-cmd --list-all | grep 8123
# → Must show: 8123/tcp
```

**8. mcpo memory directory ownership**

```bash
stat -c "%u" ~/docker/mcpo/memory | \
  grep -qF "$(sudo ls -lnd /var/srv/nextcloud-data | awk '{print $3}')" \
  && echo "PASS" || echo "FAIL"
# → Must print PASS — confirms mcpo can write the memory graph.
```

**9. mcpo memory persistence**

```bash
# In Open WebUI: "Remember the test word Alpha"
cd ~/docker/mcpo && docker compose restart mcpo
# In Open WebUI: "What test word did I ask you to remember?"
# → Returns "Alpha"
```

**10. mcpo memory file exists and is owned by Nextcloud UID**

```bash
docker exec mcpo ls -la /app/memory/graph.json
# → File must exist and show UID 33 (or your NC_UID) as owner.
```

**11. HA configuration valid**

```bash
docker exec homeassistant python -m homeassistant \
  --script check_config --config /config | \
  grep -q "Configuration is valid!" && echo "PASS" || echo "FAIL"
# → Must print PASS.
```

**12. Wyoming protocol integrations active** (UI check)

HA → Settings → Devices & Services → Wyoming Protocol. Both must show a green checkmark:
- `127.0.0.1:10300` → Faster Whisper (STT)
- `127.0.0.1:10200` → Piper TTS

```bash
# Confirm Wyoming ports are bound to the loopback (run on the server):
ss -tlnp | grep -E '10300|10200'
# → Two lines must appear, both bound to 127.0.0.1
```

**13. HA WebSocket integrity**

```bash
# Navigate to https://home.yourdomain.duckdns.org
# → Dashboard loads, no error toast.
# Open browser dev tools (F12). No wss:// WebSocket drop errors = PASS.
```

**14. Voice pipeline end-to-end**

```bash
# HA Assist or Android: say "What is today's date?"
# → STT transcribes → Ollama responds → Piper speaks. All local.
# All three stages must complete.
```

**15. Remote access from mobile data** (not home Wi-Fi)

```bash
# https://ai.yourdomain.duckdns.org  → Open WebUI loads
# https://home.yourdomain.duckdns.org → HA loads
# Confirms the full external path: DuckDNS → public IP → NPM → services.
```

---

## CRITICAL PITFALLS

Ordered by phase. Each has caused real deployment failures.

### Phase 0

**1. Kernel parameter before everything.**
`rpm-ostree kargs` + `sudo systemctl reboot` required first. The iGPU hangs the OS under sustained inference
without `amdgpu.noretry=0`.

### EX.1 — Ollama

**2. Wrong Ollama image tag.**
`ollama/ollama:rocm` is the GPU image. The untagged image is CPU-only and silently ignores
`HSA_OVERRIDE_GFX_VERSION` and all GPU variables.

**3. `HSA_OVERRIDE_GFX_VERSION` must be in the compose `environment:` block.**
Shell-level environment variables do not propagate into containers.

**4. `OLLAMA_FLASH_ATTENTION=false` is mandatory on gfx1103.**
RDNA3 Flash Attention has a numerical instability bug causing output to collapse into repeated tokens.

**5. Bare integers in `group_add` — no quotes.**
Use `- 44` not `- "44"`. Docker Compose YAML expects Linux GIDs as raw integers. String names are resolved from the
container's own `/etc/group`, not the host's.

**6. Missing `:z` on local bind mounts causes silent SELinux denial.**
Bazzite enforces SELinux. Directories created under `~/docker/` carry `user_home_t`. Container processes run as
`container_t` and are denied access without the `:z` relabel flag. Containers may start without error but fail to read
or write their data directories. Every `./...` volume mount in this guide carries `:z` (or `:ro,z`).

### EX.2 — Open WebUI + mcpo

**7. `ai-net` must exist before `docker compose up` on any stack that references it.**
Run `docker network create ai-net` in EX.0.j before all other compose steps.

**8. First user registration in Open WebUI becomes the admin.**
Complete the admin account setup before sharing the URL with anyone else.

**9. Global tool servers require per-chat activation.**
Tools added via Admin Panel → Tools do not appear automatically in chat. Users must click the **■** button in the
message input area and toggle each tool on.

**10. Ollama and Wyoming ports bound to `127.0.0.1`, not `0.0.0.0`.**
HA connects via `127.0.0.1` (loopback, host network). Setting HA's Ollama URL to `http://ollama:11434` fails — HA is
not on `ai-net` and cannot resolve that hostname.

**11. mcpo requires `command: --config /app/config/config.json`.**
Without it the container starts with no MCP servers loaded and exposes no tools.

**12. `MEMORY_FILE_PATH` is an env var inside `config.json`, not a CLI argument.**
Without it, the memory graph is not persisted to disk and is lost on every restart.

**13. mcpo tools must be registered at individual paths in Open WebUI.**
Use `http://mcpo:8000/filesystem` and `http://mcpo:8000/memory` — not the root URL.

**14. Both `./memory` and `./npm_cache` must be pre-chowned before `docker compose up`.**
Docker auto-creates missing bind-mount directories as `root:root`. mcpo (running as a non-root UID) cannot write to
root-owned directories.

**15. `npm_config_cache` and `HOME=/tmp` in mcpo are both mandatory.**
`npm_config_cache` prevents EACCES from `npx`. `HOME=/tmp` prevents startup crashes for arbitrary UIDs with no
`/etc/passwd` entry.

**16. `ls -lnd` for Nextcloud UID:GID — both flags mandatory.**
`-n` forces numeric output; `-d` lists the directory itself. Omitting either gives wrong or ambiguous results.

**17. Nextcloud data path uses a hyphen: `nextcloud-data`, not `nextcloud/data`.**
Correct: `/var/srv/nextcloud-data/<user>/files`. This matches the `/srv → /var/srv` symlink.

**18. Nextcloud SSE must be disabled.**
If server-side encryption is enabled, `files/` contains encrypted blobs — unreadable.

**19. Wait 60 seconds before checking mcpo startup logs.**
`npx -y` downloads `@modelcontextprotocol` packages on first run (30–60 seconds).

**20. Remove all angle brackets from placeholders.**
`user: "33:33"` NOT `user: "<33>:<33>"`. Angle brackets cause a YAML parse error.

### EX.3 — Home Assistant

**21. Port 8123 must be opened in firewalld before starting HA.**
HA uses `network_mode: host`. Without a firewalld rule for 8123, HA is unreachable. The firewalld commands in Step 3
must run before `docker compose up -d`.

**22. `firewalld --reload` requires immediate Docker restart.**
Reloading firewalld flushes Docker's nftables rules. Run `sudo systemctl restart docker` immediately after `--reload`,
then verify existing containers are back up before proceeding.

**23. Complete HA onboarding before editing `configuration.yaml`.**
HA generates `configuration.yaml` during onboarding. Editing it before causes a file conflict that prevents HA from
booting.

**24. Include `10.0.0.0/8` in `trusted_proxies`.**
Omitting it causes 400 Bad Request for users on Unifi or other Class A (10.x.x.x) LAN topologies.

**25. HA `network_mode: host` — no `ports:` block, no `privileged: true`.**
`privileged: true` is not required for HA Container mode and grants unrestricted host access to an internet-facing
container.

**26. HA add-ons do not exist in Container mode.**
Do not follow tutorials referencing Supervisor, Add-on Store, or named add-ons.

### NPM / Remote Access

**27. WebSockets Support must be ON for both NPM proxy hosts.**
Open WebUI streaming and the HA event bus both require WebSockets.

**28. Watchtower disabled on HA.**
`com.centurylinklabs.watchtower.enable: "false"` prevents mid-session restarts.

**29. NPM forward targets use the server's LAN IP, not a container name.**
Cross-network hostname resolution does not work between `nextcloud-aio` and `ai-net`.

---

## HARDWARE APPENDIX — Future Phases

> **■■ INFO**
>
> EX.0 through EX.6: no new hardware required. All items below are for future extension only.

### ESP32 Voice Satellite Nodes

Existing ESP32 WROOM boards are usable. Wake word runs server-side in HA via openWakeWord. ESPHome firmware
with `voice_assistant` component.

| Component | Purpose | Approx. cost |
|---|---|---|
| INMP441 I2S microphone breakout | Microphone input | ~€3 each |
| MAX98357A I2S amplifier breakout | Audio output amplifier | ~€3 each |
| Small 4 Ω passive speaker | Speaker output | ~€5 each |
| USB-C power (any phone charger) | Power supply | €0 |
| ESP32-S3 N8R8 (optional upgrade) | On-device wake word — same wiring | ~€8 each |

### Zigbee / Thread Hub

| Option | Protocol | Approx. cost |
|---|---|---|
| HA Connect ZBT-2 (recommended) | Zigbee + Thread (Matter-ready) | ~€30 |
| Sonoff ZBDongle-E | Zigbee only | ~€20 |
| USB 2.0 extension cable, 20–30 cm | Always required — keeps dongle away from CPU USB RF interference | ~€3 |

> **■ NOTE**
>
> When adding a Zigbee dongle to the HA compose file, use a targeted `devices:` entry pointing to
> `/dev/serial/by-id/`. Never use `privileged: true` — it is not needed and grants unrestricted host access to an
> internet-facing container.

---

## EX.A — AI-Notes: Write Access to Nextcloud (~15 min)

*Requires EX.2 complete*

> **■ CONCEPT**
>
> By default mcpo mounts your Nextcloud files read-only (`:ro`) to prevent the AI from corrupting Nextcloud's file
> index. This appendix adds a second, writable mount for a dedicated AI-Notes subfolder. The rest of your Nextcloud
> remains read-only.
>
> Files written there are owned by UID 33 (Nextcloud's www-data) and a cron job runs `occ files:scan` every 5
> minutes so they appear in the Nextcloud UI on your phone without delay. A system prompt tells the AI which storage
> to use for what — automatically, without you having to specify it at runtime.

### Step 1 — Create the AI-Notes Directory with Correct Ownership

> **■ CRITICAL**
>
> The AI-Notes directory must be owned by UID 33 (www-data) before mcpo starts. Replace `alice` with the username
> from EX.0.h.

```bash
# Create the AI-Notes folder inside your Nextcloud files directory.
# Replace alice with your actual Nextcloud username (from EX.0.h).
sudo mkdir -p /var/srv/nextcloud-data/alice/files/AI-Notes

# Set ownership to UID 33:33 (www-data — Nextcloud's runtime user).
# Replace 33:33 with your NC_UID:NC_GID from EX.0.g.
sudo chown 33:33 /var/srv/nextcloud-data/alice/files/AI-Notes

# Verify: third and fourth fields must match NC_UID and NC_GID:
sudo ls -lnd /var/srv/nextcloud-data/alice/files/AI-Notes
# → drwxr-xr-x 1 33 33 ...

# Verify SELinux label (must show container_file_t):
ls -Zd /var/srv/nextcloud-data/alice/files/AI-Notes
# → system_u:object_r:container_file_t:s0 ...
```

### Step 2 — Add the Notes Server to mcpo `config.json`

```bash
nano ~/docker/mcpo/config.json
```

Replace the entire contents with the following (adds the `notes` entry alongside the existing `filesystem` and `memory`
servers):

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/nc-data"]
    },
    "notes": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/nc-notes"]
    },
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"],
      "env": {
        "MEMORY_FILE_PATH": "/app/memory/graph.json"
      }
    }
  }
}
```

> **■ CONCEPT**
>
> Two separate mcpo filesystem server entries point at different directories. `:ro` on `/nc-data` enforces read-only at
> the kernel level. `/nc-notes` has no `:ro` — the AI can create, read, update, and delete files there.

### Step 3 — Add the Writable Volume to mcpo `docker-compose.yml`

```bash
nano ~/docker/mcpo/docker-compose.yml
```

In the `volumes:` block of the mcpo service, add the `/nc-notes` line. Replace `alice` with your Nextcloud username.
No `:ro` suffix — this mount is intentionally writable.

```yaml
# The full volumes: block should now read:
    volumes:
      - ./config.json:/app/config/config.json:ro,z
      - /var/srv/nextcloud-data/alice/files:/nc-data:ro
      - /var/srv/nextcloud-data/alice/files/AI-Notes:/nc-notes
      - ./memory:/app/memory:z
      - ./npm_cache:/tmp/npm_cache:z
```

> **■■ WARNING**
>
> YAML is strictly space-sensitive. The `volumes:` entries use 6 spaces of leading indent under the service definition.
> Verify indentation after editing — one wrong space causes `docker compose up` to fail with a YAML parse error.

Apply changes and verify:

```bash
cd ~/docker/mcpo && docker compose up -d

# Wait 60 seconds, then verify all three servers started:
docker logs mcpo | grep -E "Successfully connected|Application startup complete"
# → "Successfully connected to: filesystem" must appear
# → "Successfully connected to: notes" must appear
# → "Successfully connected to: memory" must appear
# → "INFO: Application startup complete." must appear
```

### Step 4 — Register the Notes Tool in Open WebUI

1. Open WebUI → Admin Panel → Settings → **Tools**.
2. Click **+** → URL: `http://mcpo:8000/notes` · Type: OpenAPI · Auth: None → Save.
3. To use it in a chat: click **■** in the message input area and toggle on the `notes` tool.

### Step 5 — Set Up the Nextcloud File-Scan Cron Job

> **■ CONCEPT**
>
> Files written directly to the filesystem bypass Nextcloud's database cache. Without a scan, they do not appear in
> the Nextcloud UI or mobile app until the next full background scan (which can take hours). This cron job runs a
> targeted scan of `AI-Notes` every 5 minutes so newly created and updated files appear promptly.
>
> **Why `--path` without `--unscanned`:** The `--unscanned` flag would skip files that Nextcloud already knows about,
> meaning updates to existing files (e.g., appending items to `shopping-list.txt`) would not be detected. A full path
> scan is required to pick up both new and modified files. The scan is scoped to a single subfolder, making its
> resource impact minimal.
>
> The `--user www-data` flag runs `occ` as Nextcloud's runtime user inside the container. The `--path` argument is
> relative to Nextcloud's data root — not the full filesystem path.

Identify the Nextcloud container name:

```bash
# Confirm the exact Nextcloud app container name:
docker ps --format "{{.Names}}" | grep nextcloud
# → One line must appear, e.g. nextcloud-aio-nextcloud
```

Test the `occ` command first:

```bash
# Test before adding to cron (replace alice with your NC_USER):
/usr/bin/docker exec --user www-data nextcloud-aio-nextcloud \
  php occ files:scan --path="alice/files/AI-Notes" --quiet
# → no output = success. Any output indicates an error — check the username spelling.
```

Open your user crontab:

```bash
crontab -e
```

Add the following line at the end of the file. This is **one single line** — do not split it. Replace `alice` with your
Nextcloud username and `nextcloud-aio-nextcloud` with your actual container name if different:

```
*/5 * * * * /usr/bin/docker exec --user www-data nextcloud-aio-nextcloud php occ files:scan --path="alice/files/AI-Notes" --quiet 2>/dev/null
```

> **■■ WARNING**
>
> The entire crontab entry must be on a single line. If your text editor wraps it visually, that is fine — it must not
> contain an actual line break in the middle.

### Step 6 — Set the System Prompt

Set the system prompt in two places.

**For HA voice — edit the Ollama integration:**
1. HA → Settings → Devices & Services → Ollama → **Configure**.
2. Find the Instructions (or "Prompt") field.
3. Paste the system prompt below. Click **Submit**. No restart needed.

**For Open WebUI chat — create a custom model:**
1. Open WebUI → Workspace (left sidebar) → Models → **+ (New Model)**.
2. Name: `Wall-E` · Base Model: `qwen2.5:14b-instruct-q4_K_M`
3. System Prompt: paste the text below.
4. Click **Save**. Select `Wall-E` as your default model in new chats.

```
You are Wall-E, a helpful home assistant. Answer concisely.
Report device states accurately when asked.

FILE AND MEMORY RULES — apply automatically, never ask which to use:
Lists and documents (shopping list, to-do list, work log, notes,
any document the user wants to keep): save as a file in /nc-notes
using the notes tool.
- Shopping items → shopping-list.txt
  Read the file first, append the new items, then write the full file.
- Work hours → work-log.txt
  Append one line per entry: YYYY-MM-DD HH:MM-HH:MM
- General notes → notes-YYYY-MM-DD.txt or a descriptive name

Personal facts (passwords, IBAN, WiFi credentials, recurring dates,
preferences): store with the memory tool.

Apply these rules silently. Confirm what was saved and where.
```

### Step 7 — Functional Tests

```bash
# Test 1: file creation
# In Open WebUI: "Add milk, eggs, and bread to the shopping list"
# → AI calls the notes tool, creates/updates shopping-list.txt.
# After ~5 minutes: Nextcloud app on phone → AI-Notes → shopping-list.txt exists.

# Test 2: file update
# In Open WebUI: "Also add coffee"
# → AI reads the existing file, appends coffee, writes it.
# → shopping-list.txt now has 4 items.

# Test 3: work log
# In Open WebUI: "I worked from 9 to 17 today"
# → AI appends a dated line to work-log.txt.

# Test 4: verify read-only protection on main files still active
# In Open WebUI: "Delete the file Notes/some-important-file.txt"
# → AI must report it cannot modify files in /nc-data (read-only mount).
```

---

## EX.B — Smart Model Routing (~20 min)

*Requires EX.1 and EX.6 complete*

> **■ CONCEPT**
>
> **Architecture:** Open WebUI has a "Functions" system. A Filter function intercepts every message — including API
> calls from Home Assistant — before it reaches the model. The `inlet()` method reads the user's text and swaps the
> `model` field.
>
> **Result:** voice and chat default to `qwen2.5:7b` (fast, ~18–22 tok/s). Saying "think carefully about this" or
> "analyze in depth" triggers 14B automatically. No mode switching, no separate wake word, no configuration per
> session.
>
> **HA change:** the native Ollama integration is replaced by the OpenAI Conversation integration pointing at Open
> WebUI's API. This routes all HA voice requests through Open WebUI so the filter can apply. **Important:** Open WebUI
> becomes a dependency for voice — if Open WebUI is down, voice stops working. Wyoming STT/TTS are unaffected.

### Step 1 — Pull the Fast Model

```bash
# Pull qwen2.5:7b (~4.7 GB). The 14B model from EX.1 remains — both will be used.
docker exec ollama ollama pull qwen2.5:7b-instruct-q4_K_M

# Verify both models are available:
docker exec ollama ollama list
# → qwen2.5:14b-instruct-q4_K_M ... must be present
# → qwen2.5:7b-instruct-q4_K_M  ... must be present
```

### Step 2 — Create the Routing Filter in Open WebUI

1. Open WebUI → Workspace (left sidebar) → **Functions**.
2. Click **+** (Create Function).
3. Name: `Wall-E Router`
4. Paste the entire code block below into the function editor.
5. Click **Save**.

```python
"""
Wall-E Model Router — Open WebUI Filter
Routes to 7B (fast) by default. Routes to 14B on deep-think keywords.
Applies to all models and all API requests including Home Assistant voice.
"""
from pydantic import BaseModel
from typing import Optional

class Filter:
    class Valves(BaseModel):
        fast_model: str = "qwen2.5:7b-instruct-q4_K_M"
        deep_model: str = "qwen2.5:14b-instruct-q4_K_M"
        deep_keywords: str = (
            "think deeply,think carefully,analyze,reason through,"
            "explain in detail,in depth,go deeper,be thorough,"
            "think step by step,elaborate,break this down"
        )

    def __init__(self):
        self.valves = self.Valves()

    def inlet(self, body: dict, __user__: Optional[dict] = None) -> dict:
        messages = body.get("messages", [])
        last_user_msg = ""
        for msg in reversed(messages):
            if msg.get("role") == "user":
                content = msg.get("content", "")
                if isinstance(content, list):
                    last_user_msg = " ".join(
                        c.get("text", "")
                        for c in content
                        if isinstance(c, dict) and c.get("type") == "text"
                    ).lower()
                else:
                    last_user_msg = str(content).lower()
                break

        keywords = [k.strip() for k in self.valves.deep_keywords.split(",")]
        if any(kw in last_user_msg for kw in keywords):
            body["model"] = self.valves.deep_model
        else:
            body["model"] = self.valves.fast_model
        return body
```

> **■ NOTE**
>
> The `Valves` class creates a configuration UI in Open WebUI. After saving, click the function settings icon to edit
> `fast_model`, `deep_model`, and `deep_keywords` without touching the code. Add your own trigger phrases to
> `deep_keywords` — they are comma-separated, case-insensitive substrings of the user's message.

Enable the filter globally:
1. In the Functions list, find **Wall-E Router**.
2. Toggle it **ON** (blue toggle).
3. Click the settings icon → enable the **Global** toggle.
4. **Global = the filter intercepts ALL requests, including API calls from HA.** Without Global, the filter only applies
   to Open WebUI's own chat UI.

### Step 3 — Generate an Open WebUI API Key

Home Assistant needs an API key to authenticate with Open WebUI's API.

1. Open WebUI → click your user avatar (top-right) → **Settings** → **Account**.
2. Find **API Keys** → click **Create new secret key**.
3. Copy the key (starts with `sk-...`). Save it — it is shown only once.

### Step 4 — Switch HA to Use Open WebUI as Its Conversation Agent

> **■ CONCEPT**
>
> HA's native Ollama integration talks directly to Ollama at `127.0.0.1:11434`. The routing filter lives in Open WebUI,
> not in Ollama. To apply the filter to voice, HA must send requests to Open WebUI's OpenAI-compatible API instead.
>
> Open WebUI exposes this API at `/openai`. HA's OpenAI Conversation integration automatically appends
> `/v1/chat/completions` to the base URL it is given, so the correct base URL to supply is
> `http://127.0.0.1:3000/openai`.

**Step 4a — Remove the existing Ollama integration:**
1. HA → Settings → Devices & Services → **Ollama**.
2. Click the three-dot menu → **Delete**. Confirm deletion.

**Step 4b — Add the OpenAI Conversation integration:**
1. HA → Settings → Devices & Services → **Add Integration**.
2. Search for and select **OpenAI Conversation**.
3. API Key: paste the `sk-...` key from Step 3.
4. Base URL: `http://127.0.0.1:3000/openai`
5. Click **Submit**.
6. Model: `qwen2.5:7b-instruct-q4_K_M` (the filter will override this — 7B is the fallback label).
7. Instructions: paste the Wall-E system prompt from EX.A Step 6 (or the minimal version below).
8. Context window: `8192` · Enable **"Control Home Assistant"**.
9. Click **Submit**.

Minimal system prompt (if you skipped EX.A):

```
You are Wall-E, a helpful home assistant. Answer concisely.
Report device states accurately when asked.
```

**Step 4c — Reassign the Wall-E Assist pipeline:**
1. HA → Settings → Voice Assistants → **Wall-E** → **Edit**.
2. Conversation Agent → select **OpenAI Conversation** (the new integration).
3. STT and TTS remain unchanged (Whisper and Piper).
4. Click **Update**.

### Step 5 — Functional Tests

```bash
# Test 1: fast path (should route to 7B)
# In Open WebUI or HA Assist: "What time is it?"
# → Response in 3-5 seconds. Run during query:
docker exec ollama ollama ps
# → PROCESSOR shows qwen2.5:7b-instruct-q4_K_M running

# Test 2: deep path (should route to 14B)
# In Open WebUI: "Think carefully about the pros and cons of solar panels"
# → Response takes longer (~10–15 s). Run during query:
docker exec ollama ollama ps
# → PROCESSOR shows qwen2.5:14b-instruct-q4_K_M running

# Test 3: voice fast path
# HA Assist or Android: "Turn on the living room light"
# → HA executes command, AI confirms. ~3–5 seconds total.

# Test 4: voice deep path
# HA Assist or Android: "Think carefully and explain how solar panels work"
# → Routes to 14B, longer response. ~10–15 seconds.
```

| Trigger phrase examples | Routes to |
|---|---|
| "turn on the lights" | 7B (fast, ~3–5 s) |
| "what's on my shopping list" | 7B (fast) |
| "think carefully about..." | 14B (deep, ~10–15 s) |
| "analyze in depth..." | 14B (deep) |
| "reason through this step by step" | 14B (deep) |
| "elaborate on..." | 14B (deep) |

---

## EX.C — Web Search via Brave (~15 min)

*Requires EX.2 complete*

> **■ CONCEPT**
>
> This appendix adds a `web_search` tool to the AI via Brave Search's API. When you ask something current ("what's
> the weather today", "search for X"), the AI calls the Brave API, reads the results, and synthesises a reply.
>
> **Privacy note:** search queries leave the server and go to Brave's API. Brave does not tie queries to an account
> and has a stronger privacy posture than Google or Bing, but this is no longer fully offline.
>
> **Free tier:** 2,000 queries per month. Sufficient for personal use. No credit card required.
>
> **Secret storage:** The Brave API key is stored in plaintext in `config.json` on your server. It is readable only by
> users with SSH access to your machine. For a single-user home server this is an accepted trade-off. Do not commit
> `config.json` to any public repository.

### Step 1 — Get a Brave Search API Key

1. Go to: `https://api.search.brave.com`
2. Click **Get Started** → create a free account.
3. Dashboard → API Keys → **Add Subscription** → select **Free tier**.
4. Copy the API key (a long alphanumeric string starting with `BSA...`).

### Step 2 — Add `brave-search` to mcpo `config.json`

```bash
nano ~/docker/mcpo/config.json
```

Add the `brave-search` block inside `"mcpServers": {}`. Replace the placeholder with your actual API key. Remove the
`notes` block if you skipped EX.A:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/nc-data"]
    },
    "notes": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/nc-notes"]
    },
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"],
      "env": {
        "MEMORY_FILE_PATH": "/app/memory/graph.json"
      }
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "BSAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
      }
    }
  }
}
```

### Step 3 — Restart mcpo and Verify

```bash
cd ~/docker/mcpo && docker compose up -d

# Wait 60 seconds (brave-search downloads on first run), then verify:
docker logs mcpo | grep -E "Successfully connected|Application startup complete"
# → "Successfully connected to: filesystem" must appear
# → "Successfully connected to: notes" must appear (if EX.A installed)
# → "Successfully connected to: memory" must appear
# → "Successfully connected to: brave-search" must appear
# → "INFO: Application startup complete." must appear

# If brave-search shows an error: verify the API key is correct.
# Common mistake: extra spaces or quotes around the key in config.json.
```

### Step 4 — Register the Tool in Open WebUI

1. Open WebUI → Admin Panel → Settings → **Tools**.
2. Click **+** → URL: `http://mcpo:8000/brave-search` · Type: OpenAPI · Auth: None → Save.
3. To use it in a chat: click **■** in the message input area and toggle on `brave-search`.

### Step 5 — Functional Tests

```bash
# Test 1: explicit search
# In Open WebUI (brave-search active): "Search for the latest Fedora 42 release notes"
# → AI calls brave_web_search tool, reads results, summarises.
# → Response includes real, current information.

# Test 2: implicit current-events query
# In Open WebUI: "What is the current price of a Raspberry Pi 5?"
# → AI recognises this requires current data and calls the search tool.

# Test 3: quota awareness
# → The Brave API returns an error if the monthly quota (2,000) is reached.
# → The AI will report the tool call failed — not silently answer from training.

# Voice search (if EX.B is also installed):
# HA Assist: "Search for how to set a timer in Home Assistant"
# → Routes to 7B or 14B per EX.B, calls brave-search, responds.
```

| Detail | Value |
|---|---|
| Free tier quota | 2,000 queries / month |
| API endpoint | `https://api.search.brave.com/res/v1/web/search` |
| Data leaves server? | Yes — query text goes to Brave's API over HTTPS |
| Response content | Web page titles, URLs, and short descriptions |
| AI behaviour | Calls the tool when it judges current data is needed |
