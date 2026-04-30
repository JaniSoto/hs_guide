# 🖥 Server Control Panel — Complete Build Guide

A purpose-built, single-page admin dashboard for a Linux home server. Live system metrics, full container management, custom command shortcuts, scheduled reboots, and display-session monitoring (KDE / Gamescope / Idle) — all in one professional interface running on `http://<server-ip>:3000`.

---

## 📋 Table of Contents

1. [Overview & Architecture](#-overview--architecture)
2. [Prerequisites](#-prerequisites)
3. [Step 1 — Directory Setup](#step-1--directory-setup)
4. [Step 2 — Docker Compose Stack](#step-2--docker-compose-stack)
5. [Step 3 — Host Metrics Collector](#step-3--host-metrics-collector)
6. [Step 4 — Systemd Service](#step-4--systemd-service)
7. [Step 5 — Panel: Dockerfile](#step-5--panel-dockerfile)
8. [Step 6 — Panel: Python Backend](#step-6--panel-python-backend)
9. [Step 7 — Panel: HTML Template](#step-7--panel-html-template)
10. [Step 8 — Panel: Stylesheet](#step-8--panel-stylesheet)
11. [Step 9 — Panel: JavaScript App](#step-9--panel-javascript-app)
12. [Step 10 — Quick Commands](#step-10--quick-commands)
13. [Step 11 — Build & Launch](#step-11--build--launch)
14. [Step 12 — Verification](#step-12--verification)
15. [Customization](#-customization)
16. [Troubleshooting](#-troubleshooting)

---

## 🏗 Overview & Architecture

The control panel consists of **three Docker services** plus **one host-side script**:

| Component | Purpose | Runs As |
|---|---|---|
| **`server-panel`** | Web UI + REST API (Flask + vanilla JS) | Docker container |
| **`host-runner`** | Privileged sidecar to execute commands on the host (via `nsenter`) | Docker container, `pid=host`, `privileged` |
| **metrics collector** | Bash script that samples CPU/GPU/disk/services/sessions | Host systemd service |

```
┌────────────────────── Browser :3000 ──────────────────────┐
│  Overview · Containers · Commands                          │
└────────────────────────────┬───────────────────────────────┘
                             │ HTTP
                ┌────────────▼────────────┐
                │   server-panel (Flask)  │
                │   - reads docker.sock   │
                │   - reads metrics JSON  │
                └────┬──────────────┬─────┘
                     │              │
       /var/run/docker.sock    docker exec host-runner
                     │              │
              [Docker daemon]   [nsenter → host PID 1]
                                    │
                      ┌─────────────┴─────────────┐
                      │  host: shutdown, systemctl │
                      │   /usr/local/bin/*.sh      │
                      └────────────────────────────┘
                             ▲
                             │ writes data.json every 2 s
                  ┌──────────┴────────────┐
                  │  metrics-collector.sh │
                  │   (systemd service)   │
                  └───────────────────────┘
```

### Why this architecture?

- **No external dashboards** to wrestle with (no Homepage, no Portainer)
- **No iframes**, no nested UIs — one cohesive page
- **Host-runner sidecar** safely exposes host capabilities to the container without giving the panel itself privileged access
- **Metrics on the host** (systemd) means full access to `sensors`, `RAPL`, `loginctl`, `/sys`, etc. without container quirks
- **JSON file** as IPC between collector and panel is dead-simple, atomic, and survives panel restarts

---

## ✅ Prerequisites

- Linux host with **Docker** + **Docker Compose v2** installed
- **systemd** (any modern distro)
- `lm_sensors` package for CPU/RAM/NVMe temps (`sudo apt install lm-sensors` / `dnf install lm_sensors` etc.)
- A non-root user with sudo access
- Optional but recommended: **passwordless sudo** for `fail2ban-client` (only the metrics script uses it)

> 💡 The example uses `sotohome` as the user, IP `192.168.178.139`, timezone `Europe/Berlin`. **Replace these with your own values** wherever you see them.

---

## Step 1 — Directory Setup

Create the full directory tree:

```bash
mkdir -p ~/docker/homepage/{panel/templates,panel/static,data,custom-metrics}
cd ~/docker/homepage
```

Final structure:

```
~/docker/homepage/
├── docker-compose.yml
├── metrics-collector.sh
├── custom-metrics/
│   └── data.json              ← written by the metrics service
├── data/
│   └── commands.json          ← edit to add quick actions
└── panel/
    ├── Dockerfile
    ├── app.py
    ├── templates/
    │   └── index.html
    └── static/
        ├── style.css
        └── app.js
```

---

## Step 2 — Docker Compose Stack

**File:** `~/docker/homepage/docker-compose.yml`

```yaml
services:
  panel:
    build: ./panel
    container_name: server-panel
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - HOST_RUNNER=host-runner
      - TZ=Europe/Berlin
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/data
      - ./custom-metrics:/metrics:ro

  # Privileged sidecar that lets the panel execute host commands.
  # Effectively runs as root on the host — keep your panel access trusted.
  host-runner:
    image: alpine:latest
    container_name: host-runner
    restart: unless-stopped
    privileged: true
    pid: host
    network_mode: host
    command: ["sleep", "infinity"]
```

### 🔧 Replace before saving

| Value | Where | Example |
|---|---|---|
| `Europe/Berlin` | `TZ` env var | Your timezone (e.g. `America/New_York`) |

### Notes

- The **panel** container has read-only access to the metrics JSON and **read-write** to `data/` (so it can persist `commands.json` edits made via the UI in the future).
- The Docker socket is mounted so the panel can list, start, stop, and `exec` into containers.
- **`host-runner`** does nothing on its own — it just sits idle, exposing PID 1's namespaces. The panel calls `docker exec host-runner nsenter ...` to run commands on the host.
- ⚠ Anyone who can reach port 3000 has effectively root access to your host. Do **not** expose this port to the internet — restrict to LAN or put it behind authenticated reverse proxy.

---

## Step 3 — Host Metrics Collector

This script runs on the host (not in a container) and writes a JSON snapshot to `~/docker/homepage/custom-metrics/data.json` every 2 seconds. The panel reads it via the read-only volume mount.

**File:** `~/docker/homepage/metrics-collector.sh`

```bash
#!/usr/bin/env bash
# metrics-collector.sh — Host-side metrics for the Server Control Panel
set -u
DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
JSON_OUT="$DIR/custom-metrics/data.json"
mkdir -p "$DIR/custom-metrics"
REFRESH=2          # fast loop interval (seconds)
SLOW_EVERY=15      # network/docker probes every N cycles

# ── GPU detection (AMD 0x1002 / Intel 0x8086) ────────────────────────────────
GPU_DEV=""; GPU_HWMON=""
for c in /sys/class/drm/card*/; do
    v=$(cat "${c}device/vendor" 2>/dev/null)
    [[ "$v" == "0x1002" || "$v" == "0x8086" ]] && { GPU_DEV="${c}device"; break; }
done
[ -n "$GPU_DEV" ] && GPU_HWMON=$(find "${GPU_DEV}/hwmon" -maxdepth 2 -name temp1_input 2>/dev/null | head -1 | xargs -I{} dirname {})

# ── RAPL energy meter (CPU power) ────────────────────────────────────────────
RAPL_FILE=""
for f in /sys/class/powercap/intel-rapl:0/energy_uj \
         /sys/class/powercap/intel-rapl/intel-rapl:0/energy_uj \
         /sys/class/powercap/amd_energy:0/energy_uj \
         /sys/class/powercap/amd-rapl:0/energy_uj; do
    [ -r "$f" ] && { RAPL_FILE="$f"; break; }
done
prev_energy=""; prev_time=$(date +%s%3N)
if [ -n "$RAPL_FILE" ]; then
    prev_energy=$(cat "$RAPL_FILE" 2>/dev/null); cpu_power="0.0W"
else
    cpu_power="n/a"
fi

_cpustat() { awk '/^cpu /{print $2,$3,$4,$5,$6,$7,$8; exit}' /proc/stat; }
prev_cs=$(_cpustat); cpu_pct=0

# Slow-cache defaults (refreshed every SLOW_EVERY cycles)
WAN_IP="–"; F2B="0"; NPM_ST="–"; NC_ST="–"; CERT_ST="–"; BK_ST="–"

refresh_slow() {
    WAN_IP=$(curl -s --max-time 4 https://api.ipify.org 2>/dev/null); [ -z "$WAN_IP" ] && WAN_IP="unreachable"
    F2B=$(sudo -n fail2ban-client status sshd 2>/dev/null | grep 'Currently banned' | grep -oE '[0-9]+'); [ -z "$F2B" ] && F2B="0"
    code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 2 http://localhost:81 2>/dev/null)
    [ "${code:-000}" != "000" ] && NPM_ST="online" || NPM_ST="offline"
    nr=$(docker exec nextcloud-aio-nextcloud php occ maintenance:mode 2>/dev/null)
    if   echo "$nr" | grep -qi "disabled\|not enabled"; then NC_ST="online"
    elif echo "$nr" | grep -qi "enabled";               then NC_ST="maintenance"
    else NC_ST="n/a"; fi
    ce=$(docker exec nginx-proxy-manager openssl x509 -noout -enddate -in /etc/letsencrypt/live/npm-1/cert.pem 2>/dev/null | cut -d= -f2)
    if [ -n "$ce" ]; then
        d=$(( ( $(date -d "$ce" +%s) - $(date +%s) ) / 86400 ))
        (( d <= 30 )) && CERT_ST="renew in ${d}d" || CERT_ST="valid ${d}d"
    else CERT_ST="pending"; fi
    bk=$(docker logs nextcloud-aio-mastercontainer 2>/dev/null | grep -i "Backup finished successfully" | tail -1 | awk '{print $1,$2}')
    [ -n "$bk" ] && BK_ST="$bk" || BK_ST="never"
}

CYCLE=0
while true; do
    # ── CPU power (RAPL delta) ───────────────────────────────────────────────
    cur_time=$(date +%s%3N); elapsed_ms=$(( cur_time - prev_time ))
    if [ -n "$prev_energy" ] && [ -r "$RAPL_FILE" ]; then
        cur_energy=$(cat "$RAPL_FILE" 2>/dev/null)
        delta=$(( cur_energy - prev_energy )); (( delta < 0 )) && delta=$cur_energy
        if (( elapsed_ms > 0 && delta > 0 )); then
            wx10=$(( delta / (elapsed_ms * 100) ))
            cpu_power="$(( wx10 / 10 )).$(( wx10 % 10 ))W"
        fi
        prev_energy=$cur_energy
    fi
    prev_time=$cur_time

    # ── CPU usage ────────────────────────────────────────────────────────────
    cur_cs=$(_cpustat); read -ra p <<< "$prev_cs"; read -ra c <<< "$cur_cs"
    pt=0; ct=0
    for v in "${p[@]}"; do (( pt += v )); done
    for v in "${c[@]}"; do (( ct += v )); done
    dt=$(( ct - pt )); di=$(( c[3] - p[3] ))
    (( dt > 0 )) && cpu_pct=$(( 100 * (dt - di) / dt )) || cpu_pct=0
    prev_cs="$cur_cs"

    # ── CPU clock ────────────────────────────────────────────────────────────
    fsum=0; fn=0
    for f in /sys/devices/system/cpu/cpufreq/policy*/scaling_cur_freq; do
        [ -r "$f" ] || continue
        (( fsum += $(< "$f"), fn++ ))
    done
    if (( fn > 0 )); then
        mhz=$(( fsum / fn / 1000 ))
        (( mhz >= 1000 )) && cpu_freq="$(( mhz/1000 )).$(( (mhz%1000)/100 ))GHz" || cpu_freq="${mhz}MHz"
    else cpu_freq="n/a"; fi

    # ── Temps ────────────────────────────────────────────────────────────────
    CPU_TEMP=$(sensors 2>/dev/null | grep -E '(Tctl|Tdie|Package id 0)' | head -1 | awk '{print $2}')
    [ -z "$CPU_TEMP" ] && CPU_TEMP=$(sensors 2>/dev/null | grep 'temp1' | head -1 | awk '{print $2}')
    RAM_TEMP=$(sensors 2>/dev/null | grep -A2 'spd5118'   | awk '/temp1/{print $2; exit}')
    NVME_TEMP=$(sensors 2>/dev/null | grep -A2 'nvme-pci' | awk '/Composite/{print $2; exit}')

    # ── Memory ───────────────────────────────────────────────────────────────
    RAM_PCT=$(free   | awk '/^Mem:/{printf "%d%%", $3*100/$2}')
    RAM_USAGE=$(free -h | awk '/^Mem:/{printf "%s / %s", $3, $2}')
    SWAP_USAGE=$(free -h | awk '/^Swap:/{printf "%s / %s", $3, $2}')

    # ── GPU ──────────────────────────────────────────────────────────────────
    gpu_busy="0%"; gpu_freq="n/a"; gpu_temp="n/a"; gpu_vram="n/a"; gpu_power="n/a"
    if [ -n "$GPU_DEV" ]; then
        gb=$(cat "${GPU_DEV}/gpu_busy_percent" 2>/dev/null); gpu_busy="${gb:-0}%"
        ghz=${GPU_HWMON:+$(cat "${GPU_HWMON}/freq1_input" 2>/dev/null)}
        [ -n "${ghz:-}" ] && (( ghz > 0 )) && gpu_freq="$(( ghz / 1000000 ))MHz"
        gt=${GPU_HWMON:+$(cat "${GPU_HWMON}/temp1_input" 2>/dev/null)}
        [ -n "${gt:-}" ] && gpu_temp="+$(( gt / 1000 ))°C"
        vu=$(cat "${GPU_DEV}/mem_info_vram_used"  2>/dev/null)
        vt=$(cat "${GPU_DEV}/mem_info_vram_total" 2>/dev/null)
        if [ -n "$vu" ] && [ -n "$vt" ] && (( vt > 0 )); then
            vum=$(( vu / 1048576 )); vtm=$(( vt / 1048576 ))
            if (( vtm >= 1024 )); then
                gpu_vram="$(( vum/1024 )).$(( (vum%1024)*10/1024 ))/$(( vtm/1024 )).$(( (vtm%1024)*10/1024 ))GiB"
            else
                gpu_vram="${vum}/${vtm}MiB"
            fi
        fi
        gpw=${GPU_HWMON:+$(cat "${GPU_HWMON}/power1_average" 2>/dev/null)}
        [ -n "${gpw:-}" ] && (( gpw > 0 )) && gpu_power="$(( gpw / 1000000 )).$(( (gpw % 1000000) / 100000 ))W"
    fi

    # ── Storage ──────────────────────────────────────────────────────────────
    OS_S=$(df -h /var 2>/dev/null | awk 'NR==2{print $3" / "$2" ("$5")"}')
    DAT_RAW=$(df -h /var/srv/nextcloud-data 2>/dev/null | awk 'NR==2{print $3" / "$2" ("$5")"}')
    SSH_C=$(ss -tn state established 2>/dev/null | grep -c ":22022")
    LOAD=$(awk '{print $1", "$2", "$3}' /proc/loadavg)
    UPTIME=$(uptime -p | sed 's/up //')

    # ── Display session (idle / KDE / Gamescope) ─────────────────────────────
    DISPLAY_STATE="idle"; DISPLAY_LABEL="No session"
    SEAT0=$(loginctl list-sessions --no-legend 2>/dev/null | awk '$4=="seat0"{print $1;exit}')
    if [ -n "$SEAT0" ]; then
        CLS=$(loginctl show-session "$SEAT0" -p Class --value 2>/dev/null)
        if [ "$CLS" = "greeter" ]; then
            DISPLAY_STATE="idle"; DISPLAY_LABEL="SDDM Greeter"
        else
            DSK=$(loginctl show-session "$SEAT0" -p Desktop --value 2>/dev/null)
            DSK_LC=$(echo "$DSK" | tr '[:upper:]' '[:lower:]')
            case "$DSK_LC" in
                *gamescope*|*gaming*) DISPLAY_STATE="gaming"; DISPLAY_LABEL="Gamescope (Gaming)" ;;
                *plasma*|*kde*)       DISPLAY_STATE="kde";    DISPLAY_LABEL="KDE Plasma" ;;
                *)                    DISPLAY_STATE="active"; DISPLAY_LABEL="${DSK:-Active session}" ;;
            esac
        fi
    fi

    # ── Scheduled reboot detection ───────────────────────────────────────────
    REBOOT_SCHEDULED=""
    if [ -f /run/systemd/shutdown/scheduled ]; then
        USEC=$(awk -F= '/USEC=/{print $2}' /run/systemd/shutdown/scheduled 2>/dev/null)
        if [ -n "$USEC" ] && [ "$USEC" -gt 0 ] 2>/dev/null; then
            EPOCH=$(( USEC / 1000000 ))
            REBOOT_SCHEDULED=$(date -d "@$EPOCH" '+%a %H:%M' 2>/dev/null)
        fi
    fi

    # ── Failed systemd units ─────────────────────────────────────────────────
    FAIL_RAW=$(systemctl --failed --no-legend --plain 2>/dev/null)
    FAIL_N=$(echo "$FAIL_RAW" | grep -c '.' || true)
    if (( FAIL_N > 0 )); then
        FAIL_NAMES=$(echo "$FAIL_RAW" | awk '{print $1}' | head -3 | tr '\n' ' ' | sed 's/ $//; s/ /, /g')
        extra=$(( FAIL_N - 3 )); (( extra > 0 )) && FAIL_NAMES="${FAIL_NAMES} +${extra}"
        FIRST=$(echo "$FAIL_RAW" | head -1 | awk '{print $1}')
        FAIL_REASON=$(systemctl show "$FIRST" --property=Result --value 2>/dev/null)
        [ -n "$FAIL_REASON" ] && FAIL_NAMES="${FAIL_NAMES} (${FAIL_REASON})"
    else
        FAIL_NAMES="all healthy"
    fi
    FAIL_NAMES_J=$(printf '%s' "$FAIL_NAMES" | sed 's/\\/\\\\/g; s/"/\\"/g')

    (( CYCLE % SLOW_EVERY == 0 )) && refresh_slow

    # ── Atomic write ─────────────────────────────────────────────────────────
    cat <<JSON > "${JSON_OUT}.tmp"
{
  "cpu_pct": "${cpu_pct}%",
  "cpu_freq": "${cpu_freq}",
  "cpu_temp": "${CPU_TEMP:-n/a}",
  "cpu_power": "${cpu_power}",
  "ram_pct": "${RAM_PCT}",
  "ram_usage": "${RAM_USAGE}",
  "swap_usage": "${SWAP_USAGE}",
  "load_avg": "${LOAD}",
  "uptime": "${UPTIME}",
  "gpu_busy": "${gpu_busy}",
  "gpu_freq": "${gpu_freq}",
  "gpu_temp": "${gpu_temp}",
  "gpu_vram": "${gpu_vram}",
  "gpu_power": "${gpu_power}",
  "ram_temp": "${RAM_TEMP:-n/a}",
  "nvme_temp": "${NVME_TEMP:-n/a}",
  "os_storage": "${OS_S:-n/a}",
  "data_storage": "${DAT_RAW:-not mounted}",
  "ssh_conn": "${SSH_C:-0}",
  "f2b_banned": "${F2B:-0}",
  "wan_ip": "${WAN_IP:-n/a}",
  "systemd_failed": "${FAIL_N:-0}",
  "systemd_failed_units": "${FAIL_NAMES_J}",
  "npm_status": "${NPM_ST}",
  "nextcloud_status": "${NC_ST}",
  "cert_status": "${CERT_ST}",
  "backup_status": "${BK_ST}",
  "display_state": "${DISPLAY_STATE}",
  "display_label": "${DISPLAY_LABEL}",
  "reboot_scheduled": "${REBOOT_SCHEDULED}"
}
JSON
    mv "${JSON_OUT}.tmp" "$JSON_OUT"
    CYCLE=$(( CYCLE + 1 ))
    sleep "$REFRESH"
done
```

### 🔧 Replace before saving

The script references **specific containers and paths from the example homelab**. Adapt these to your setup or remove the lines you don't need:

| Reference | Where | Action |
|---|---|---|
| `nextcloud-aio-nextcloud` | `refresh_slow()` | Replace with your Nextcloud container, or delete the block |
| `nginx-proxy-manager` | `refresh_slow()` | Replace with your reverse-proxy container, or delete |
| `nextcloud-aio-mastercontainer` | `refresh_slow()` | Replace or delete |
| `/var/srv/nextcloud-data` | `DAT_RAW=...` | Path to your data volume — adjust |
| `:22022` | `SSH_C=...` | Your SSH port (default `:22`) |
| `localhost:81` | NPM check | Replace with your reverse-proxy admin URL |

> 💡 If a container or path doesn't exist, the script gracefully shows `–` or `n/a` for that field. You can leave the references and they'll just degrade silently.

### Make it executable

```bash
chmod +x ~/docker/homepage/metrics-collector.sh
```

---

## Step 4 — Systemd Service

Run the metrics collector as a managed background service so it survives reboots and restarts on failure.

**File:** `/etc/systemd/system/homepage-metrics.service`

Create with sudo:

```bash
sudo nano /etc/systemd/system/homepage-metrics.service
```

Paste:

```ini
[Unit]
Description=Server Control Panel — Host Metrics Collector
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/home/sotohome/docker/homepage
ExecStart=/bin/bash /home/sotohome/docker/homepage/metrics-collector.sh
Restart=always
RestartSec=5
Nice=10

[Install]
WantedBy=multi-user.target
```

### 🔧 Replace before saving

| Value | Action |
|---|---|
| `/home/sotohome/docker/homepage` | Replace with **absolute path** to your `~/docker/homepage` (run `echo ~/docker/homepage` to print it) |

### Enable and start

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now homepage-metrics.service
sudo systemctl status homepage-metrics --no-pager
```

After ~3 seconds verify the JSON is being written:

```bash
cat ~/docker/homepage/custom-metrics/data.json
```

---

## Step 5 — Panel: Dockerfile

A minimal Python image with the Docker CLI baked in (so it can talk to the mounted socket).

**File:** `~/docker/homepage/panel/Dockerfile`

```dockerfile
FROM python:3.12-alpine
RUN apk add --no-cache docker-cli && pip install --no-cache-dir flask
COPY app.py /app.py
COPY templates /templates
COPY static /static
EXPOSE 3000
CMD ["python", "-u", "/app.py"]
```

No edits needed.

---

## Step 6 — Panel: Python Backend

The Flask app. About 130 lines of pure Python. Endpoints:

| Endpoint | Method | Purpose |
|---|---|---|
| `/` | GET | Serves the SPA |
| `/api/overview` | GET | Aggregate snapshot for the Overview tab |
| `/api/containers` | GET | List + live stats |
| `/api/logs/<name>` | GET | Last 300 log lines |
| `/api/action/<name>/<action>` | POST | start, stop, restart, pause, unpause, kill |
| `/api/commands` | GET | List of quick commands |
| `/api/commands/<id>/run` | POST | Execute a command |
| `/api/system/restart-unit/<unit>` | POST | Restart a failed systemd unit |
| `/api/power/reboot` | POST | Reboot now or in N minutes |
| `/api/power/cancel` | POST | Cancel scheduled reboot |

**File:** `~/docker/homepage/panel/app.py`

```python
import json, re, os, subprocess
from pathlib import Path
from flask import Flask, jsonify, request, Response, send_file

DATA = Path('/data'); DATA.mkdir(parents=True, exist_ok=True)
COMMANDS  = DATA/'commands.json'
METRICS   = Path('/metrics/data.json')
HOST_RUNNER = os.environ.get('HOST_RUNNER', 'host-runner')

SAFE_NAME = re.compile(r'^[A-Za-z0-9_.\-]+$')
SAFE_ID   = re.compile(r'^[A-Za-z0-9_\-]+$')
SAFE_UNIT = re.compile(r'^[A-Za-z0-9_.\-@:]+$')

app = Flask(__name__, template_folder='/templates', static_folder='/static', static_url_path='/static')

def docker(*args, timeout=15):
    try:
        r = subprocess.run(['docker', *args], capture_output=True, text=True, timeout=timeout)
        return r.stdout, r.stderr, r.returncode
    except Exception as e:
        return '', str(e), 1

def jload(p, fallback):
    if p.exists():
        try: return json.loads(p.read_text())
        except: return fallback
    return fallback

def num(s):
    m = re.search(r'(\d+\.?\d*)', str(s or ''))
    return float(m.group(1)) if m else 0.0

# Explicit namespace flags — works with both util-linux and BusyBox nsenter
NSENTER = ['nsenter','-t','1','-m','-u','-i','-n','-p','--']

def host_run(*sub, timeout=10):
    full = ['docker','exec', HOST_RUNNER] + NSENTER + list(sub)
    try:
        r = subprocess.run(full, capture_output=True, text=True, timeout=timeout)
        return r.returncode == 0, r.stdout, r.stderr
    except Exception as e:
        return False, '', str(e)

@app.route('/')
def index(): return send_file('/templates/index.html')

@app.route('/api/containers')
def api_containers():
    out,_,_ = docker('ps','-a','--format','{{json .}}')
    cs = [json.loads(l) for l in out.strip().splitlines() if l.strip()]
    so,_,_ = docker('stats','--no-stream','--format','{{json .}}')
    stats = {}
    for l in so.strip().splitlines():
        if l.strip():
            try:
                s=json.loads(l); stats[s.get('Name','')]=s
            except: pass
    for c in cs:
        c['stats'] = stats.get(c.get('Names',''), {})
    return jsonify(cs)

@app.route('/api/logs/<name>')
def api_logs(name):
    if not SAFE_NAME.match(name): return 'invalid', 400
    out, err, _ = docker('logs','--tail=300', name, timeout=8)
    return Response(out + ('\n--- stderr ---\n'+err if err else ''), mimetype='text/plain')

@app.route('/api/action/<name>/<action>', methods=['POST'])
def api_action(name, action):
    if not SAFE_NAME.match(name): return jsonify(ok=False, err='invalid name'), 400
    if action not in ('start','stop','restart','pause','unpause','kill'):
        return jsonify(ok=False, err='invalid action'), 400
    out, err, rc = docker(action, name, timeout=30)
    return jsonify(ok=rc==0, out=out, err=err)

@app.route('/api/commands')
def api_commands():
    return jsonify(jload(COMMANDS, []))

@app.route('/api/commands/<cid>/run', methods=['POST'])
def api_run(cid):
    if not SAFE_ID.match(cid): return jsonify(ok=False, err='invalid id'), 400
    cmds = jload(COMMANDS, [])
    cmd = next((c for c in cmds if c.get('id')==cid), None)
    if not cmd: return jsonify(ok=False, err='not found'), 404
    runner = cmd.get('runner','self'); line = cmd.get('cmd',''); timeout = int(cmd.get('timeout', 60))
    if runner == 'host':
        full = ['docker','exec', HOST_RUNNER] + NSENTER + ['sh','-c', line]
    elif runner.startswith('container:'):
        cname = runner.split(':',1)[1]
        if not SAFE_NAME.match(cname): return jsonify(ok=False, err='bad container'), 400
        full = ['docker','exec', cname, 'sh','-c', line]
    else:
        full = ['sh','-c', line]
    try:
        r = subprocess.run(full, capture_output=True, text=True, timeout=timeout)
        return jsonify(ok=r.returncode==0, code=r.returncode, stdout=r.stdout, stderr=r.stderr)
    except subprocess.TimeoutExpired:
        return jsonify(ok=False, err=f'timeout after {timeout}s'), 504

@app.route('/api/overview')
def api_overview():
    out,_,_ = docker('ps','-a','--format','{{json .}}')
    cs = [json.loads(l) for l in out.strip().splitlines() if l.strip()]
    running = sum(1 for c in cs if 'running' in (c.get('State') or '').lower())
    total = len(cs)
    so,_,_ = docker('stats','--no-stream','--format','{{json .}}')
    stats = []
    for l in so.strip().splitlines():
        if l.strip():
            try: stats.append(json.loads(l))
            except: pass
    top_cpu = sorted(stats, key=lambda s: num(s.get('CPUPerc')), reverse=True)[:5]
    top_mem = sorted(stats, key=lambda s: num(s.get('MemPerc')), reverse=True)[:5]
    return jsonify({
        'containers': {'running': running, 'total': total, 'stopped': total-running},
        'metrics': jload(METRICS, {}),
        'top_cpu': [{'name': s.get('Name',''), 'pct': num(s.get('CPUPerc')), 'raw': s.get('CPUPerc','')} for s in top_cpu],
        'top_mem': [{'name': s.get('Name',''), 'pct': num(s.get('MemPerc')), 'raw': s.get('MemUsage','')} for s in top_mem],
    })

@app.route('/api/system/restart-unit/<unit>', methods=['POST'])
def api_restart_unit(unit):
    if not SAFE_UNIT.match(unit):
        return jsonify(ok=False, err='invalid unit'), 400
    ok, out, err = host_run('systemctl','restart', unit, timeout=30)
    return jsonify(ok=ok, stdout=out, stderr=err)

@app.route('/api/power/reboot', methods=['POST'])
def api_reboot():
    body = request.get_json(force=True, silent=True) or {}
    try: minutes = int(body.get('minutes', 0))
    except: return jsonify(ok=False, err='invalid minutes'), 400
    if minutes < 0 or minutes > 1440:
        return jsonify(ok=False, err='minutes out of range (0–1440)'), 400
    msg = "Scheduled by Control Panel"
    sub = ['shutdown','-r','now',msg] if minutes == 0 else ['shutdown','-r',f'+{minutes}',msg]
    ok, out, err = host_run(*sub)
    return jsonify(ok=ok, stdout=out, stderr=err)

@app.route('/api/power/cancel', methods=['POST'])
def api_cancel_reboot():
    ok, out, err = host_run('shutdown','-c')
    return jsonify(ok=ok, stdout=out, stderr=err)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3000)
```

> 💡 The `NSENTER` list uses **explicit namespace flags** (`-m -u -i -n -p`) instead of `-a` because Alpine's BusyBox `nsenter` doesn't support `-a`. This way the same code works regardless of which `nsenter` is in the host-runner image.

> 🔒 All path parameters in URLs are validated against strict regex (`SAFE_NAME`, `SAFE_ID`, `SAFE_UNIT`) before any `subprocess` call — no shell injection.

---

## Step 7 — Panel: HTML Template

Pure semantic HTML — no JS frameworks. Reads from CSS class hooks; logic lives in `app.js`.

**File:** `~/docker/homepage/panel/templates/index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<meta name="color-scheme" content="dark">
<title>Server Control Panel</title>
<link rel="stylesheet" href="/static/style.css">
</head>
<body>

<div class="app">

  <header class="header">
    <div class="brand">
      <div class="logo">HS</div>
      <div>
        <div class="title">HS Server</div>
        <div class="subtitle">Control Panel</div>
      </div>
    </div>
    <div class="status-bar">
      <div class="pill" id="pill-health">● <b>—</b></div>
      <div class="pill session" id="pill-session"><span class="led"></span><b id="session-label">—</b></div>
      <div class="pill">CPU <b id="hb-cpu">—</b></div>
      <div class="pill">GPU <b id="hb-gpu">—</b></div>
      <div class="pill">RAM <b id="hb-ram">—</b></div>
      <div class="pill" id="hb-clock">—</div>
    </div>
  </header>

  <nav class="tabs">
    <div class="tab active" data-page="overview">🏠 Overview</div>
    <div class="tab" data-page="containers">📦 Containers <span class="count" id="tab-count-c">—</span></div>
    <div class="tab" data-page="commands">⚡ Commands <span class="count" id="tab-count-cmd">—</span></div>
  </nav>

  <main>

    <section class="page active" id="page-overview">

      <h2 class="section-title">Top Resource Consumers</h2>
      <div class="card-grid cols-2">
        <div class="card"><h3>🔥 Top CPU Usage</h3><div id="top-cpu" class="hog-list"></div></div>
        <div class="card"><h3>🧠 Top Memory Usage</h3><div id="top-mem" class="hog-list"></div></div>
      </div>

      <h2 class="section-title">Hardware</h2>
      <div class="card-grid cols-3">
        <div class="card">
          <h3>🔲 Processor</h3>
          <div class="metric"><span class="k">Usage</span><span class="v" id="hw-cpu-pct">—</span></div>
          <div class="metric"><span class="k">Clock</span><span class="v" id="hw-cpu-freq">—</span></div>
          <div class="metric"><span class="k">Temp</span><span class="v" id="hw-cpu-temp">—</span></div>
          <div class="metric"><span class="k">Power</span><span class="v" id="hw-cpu-power">—</span></div>
        </div>
        <div class="card">
          <h3>🎮 Graphics</h3>
          <div class="metric"><span class="k">Load</span><span class="v" id="hw-gpu-busy">—</span></div>
          <div class="metric"><span class="k">VRAM</span><span class="v" id="hw-gpu-vram">—</span></div>
          <div class="metric"><span class="k">Temp</span><span class="v" id="hw-gpu-temp">—</span></div>
          <div class="metric"><span class="k">Power</span><span class="v" id="hw-gpu-power">—</span></div>
        </div>
        <div class="card">
          <h3>🌡 Memory & Thermals</h3>
          <div class="metric"><span class="k">RAM</span><span class="v" id="hw-ram-usage">—</span></div>
          <div class="metric"><span class="k">DDR5 Temp</span><span class="v" id="hw-ram-temp">—</span></div>
          <div class="metric"><span class="k">NVMe Temp</span><span class="v" id="hw-nvme-temp">—</span></div>
          <div class="metric"><span class="k">Load Avg</span><span class="v" id="hw-load">—</span></div>
        </div>
      </div>

      <h2 class="section-title">Storage</h2>
      <div class="card-grid cols-2">
        <div class="card">
          <h3>💾 OS Disk</h3>
          <div class="big-metric" id="st-os-text">—</div>
          <div class="progress"><div class="track"><div class="fill" id="st-os-bar" style="width:0%"></div></div></div>
        </div>
        <div class="card">
          <h3>🗄 Data Disk</h3>
          <div class="big-metric" id="st-data-text">—</div>
          <div class="progress"><div class="track"><div class="fill" id="st-data-bar" style="width:0%"></div></div></div>
        </div>
      </div>

      <h2 class="section-title">Services & Network</h2>
      <div class="card-grid cols-3">
        <div class="card">
          <h3>☁ Nextcloud</h3>
          <div class="metric"><span class="k">Status</span><span class="v" id="sv-nc-state">—</span></div>
          <div class="metric"><span class="k">Last Backup</span><span class="v" id="sv-nc-backup">—</span></div>
        </div>
        <div class="card">
          <h3>🔀 Reverse Proxy</h3>
          <div class="metric"><span class="k">NPM</span><span class="v" id="sv-npm">—</span></div>
          <div class="metric"><span class="k">SSL Cert</span><span class="v" id="sv-cert">—</span></div>
        </div>
        <div class="card">
          <h3>🛡 Network Security</h3>
          <div class="metric"><span class="k">WAN IP</span><span class="v" id="sv-wan">—</span></div>
          <div class="metric"><span class="k">SSH Sessions</span><span class="v" id="sv-ssh">—</span></div>
          <div class="metric"><span class="k">Fail2Ban Bans</span><span class="v" id="sv-f2b">—</span></div>
        </div>
      </div>

      <h2 class="section-title">Active Issues</h2>
      <div id="issues-list"></div>

      <h2 class="section-title">Power & Maintenance</h2>
      <div class="card">
        <div class="power">
          <div class="power-status" id="power-status">
            <span class="led"></span>
            <span id="reboot-status">No reboot scheduled</span>
          </div>
          <div class="power-actions">
            <input type="number" id="reboot-mins" value="10" min="1" max="1440" title="Minutes from now">
            <span style="color:var(--mu);font-size:12px">min</span>
            <button class="btn" onclick="App.scheduleReboot()">⏰ Schedule</button>
            <button class="btn danger" id="cancel-btn" onclick="App.cancelReboot()" style="display:none">✕ Cancel</button>
            <button class="btn danger" onclick="App.rebootNow()">⏻ Reboot Now</button>
          </div>
        </div>
      </div>
    </section>

    <section class="page" id="page-containers">
      <div class="toolbar">
        <input id="q" placeholder="🔍 Filter containers by name or image…">
        <button class="btn" onclick="App.refreshContainers()">↻ Refresh</button>
        <span class="cnt" id="cnt">—</span>
      </div>
      <div id="container-list"></div>
    </section>

    <section class="page" id="page-commands">
      <div class="toolbar">
        <span class="hint">Edit shortcuts via SSH: <code>~/docker/homepage/data/commands.json</code></span>
        <button class="btn" onclick="App.loadCommands()">↻ Refresh</button>
      </div>
      <div class="cmd-grid" id="cmd-list"></div>
    </section>

  </main>
</div>

<div class="modal" id="modal" onclick="if(event.target===this)App.closeModal()">
  <div class="mbox">
    <div class="mhd"><div id="mtitle">—</div><button class="btn ghost" onclick="App.closeModal()">✕</button></div>
    <div class="mbd" id="mbody"></div>
  </div>
</div>

<div class="toast" id="toast"></div>

<script src="/static/app.js" defer></script>
</body>
</html>
```

### 🔧 Replace before saving

| Element | Where | Action |
|---|---|---|
| `HS` (logo) | `<div class="logo">HS</div>` | Your initials, max 2-3 chars |
| `HS Server` | `<div class="title">` | Your server name |

---

## Step 8 — Panel: Stylesheet

Modern, dark, single CSS file. Custom-property based theme so colors are easy to tweak in one place.

**File:** `~/docker/homepage/panel/static/style.css`

```css
*,*::before,*::after{box-sizing:border-box;margin:0;padding:0}
:root{
  --bg:#0a0e1a; --bg2:#0f1525; --card:#131826; --card-hi:#1a2138;
  --hi:#2d3748; --bd:#1f2937; --bd-hi:#374151;
  --tx:#e5e7eb; --tx2:#d1d5db; --mu:#9ca3af; --mu2:#6b7280;
  --ac:#38bdf8; --ac-bg:rgba(56,189,248,.1);
  --gn:#22c55e; --gn-bg:rgba(34,197,94,.12);
  --rd:#ef4444; --rd-bg:rgba(239,68,68,.12);
  --yl:#eab308; --yl-bg:rgba(234,179,8,.12);
  --pp:#a78bfa; --pp-bg:rgba(167,139,250,.12);
}
html,body{background:var(--bg);color:var(--tx);font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,Helvetica,Arial,sans-serif;font-size:14px;line-height:1.5;-webkit-font-smoothing:antialiased}
body{min-height:100vh;background:radial-gradient(circle at 20% 0%,#101a2e 0%,var(--bg) 50%)}
code{font-family:ui-monospace,"SF Mono",Menlo,monospace;background:var(--bg2);padding:1px 6px;border-radius:3px;font-size:12px;color:var(--tx2)}

.app{max-width:1600px;margin:0 auto;padding:0 24px 60px}

/* Header */
.header{display:flex;align-items:center;justify-content:space-between;padding:18px 0;gap:24px;flex-wrap:wrap}
.brand{display:flex;align-items:center;gap:12px}
.brand .logo{width:36px;height:36px;background:linear-gradient(135deg,var(--ac) 0%,var(--pp) 100%);border-radius:8px;display:grid;place-items:center;font-weight:700;color:#fff;font-size:13px;letter-spacing:-.02em;box-shadow:0 2px 8px rgba(56,189,248,.25)}
.brand .title{font-size:15px;font-weight:600;letter-spacing:-.01em}
.brand .subtitle{font-size:11px;color:var(--mu);margin-top:1px}
.status-bar{display:flex;align-items:center;gap:8px;font-size:12px;color:var(--mu);flex-wrap:wrap}
.pill{display:flex;align-items:center;gap:6px;padding:5px 11px;background:var(--card);border:1px solid var(--bd);border-radius:6px;font-variant-numeric:tabular-nums}
.pill b{color:var(--tx);font-weight:500}
.pill.healthy{border-color:rgba(34,197,94,.3);color:var(--gn)}
.pill.healthy b{color:var(--gn)}
.pill.alert{border-color:rgba(239,68,68,.35);color:var(--rd);animation:pulse 2s ease-in-out infinite}
.pill.alert b{color:var(--rd)}
@keyframes pulse{0%,100%{opacity:1}50%{opacity:.65}}

/* Session pill LED */
.pill.session{padding-left:8px}
.pill.session .led{display:inline-block;width:9px;height:9px;border-radius:50%;background:var(--mu2);transition:all .3s;flex-shrink:0}
.pill.session.idle{border-color:var(--bd)}
.pill.session.idle .led{background:var(--mu2)}
.pill.session.kde{border-color:rgba(56,189,248,.35);color:var(--ac)}
.pill.session.kde .led{background:var(--ac);box-shadow:0 0 10px rgba(56,189,248,.6)}
.pill.session.kde b{color:var(--ac)}
.pill.session.gaming{border-color:rgba(167,139,250,.4);color:var(--pp)}
.pill.session.gaming .led{background:var(--pp);box-shadow:0 0 10px rgba(167,139,250,.7)}
.pill.session.gaming b{color:var(--pp)}
.pill.session.active{border-color:rgba(234,179,8,.35);color:var(--yl)}
.pill.session.active .led{background:var(--yl);box-shadow:0 0 10px rgba(234,179,8,.5)}
.pill.session.active b{color:var(--yl)}

/* Tabs */
.tabs{display:flex;gap:2px;border-bottom:1px solid var(--bd);margin:8px 0 24px}
.tab{padding:11px 18px;cursor:pointer;color:var(--mu);font-size:13px;font-weight:500;border-bottom:2px solid transparent;display:flex;align-items:center;gap:8px;transition:color .15s,border-color .15s}
.tab:hover{color:var(--tx)}
.tab.active{color:var(--ac);border-bottom-color:var(--ac)}
.tab .count{background:var(--bd);color:var(--tx2);padding:1px 7px;border-radius:10px;font-size:10px;font-weight:600;font-variant-numeric:tabular-nums}
.tab.active .count{background:var(--ac-bg);color:var(--ac)}

/* Pages */
.page{display:none}
.page.active{display:block;animation:fadeIn .22s ease}
@keyframes fadeIn{from{opacity:0;transform:translateY(4px)}to{opacity:1;transform:translateY(0)}}

.section-title{font-size:11px;text-transform:uppercase;letter-spacing:.08em;color:var(--mu);font-weight:600;margin:24px 0 10px;padding:0 2px}

/* Cards */
.card{background:var(--card);border:1px solid var(--bd);border-radius:8px;padding:14px 16px;transition:border-color .15s}
.card:hover{border-color:var(--bd-hi)}
.card h3{font-size:11px;text-transform:uppercase;letter-spacing:.06em;color:var(--mu);font-weight:600;margin-bottom:10px}
.card-grid{display:grid;gap:12px}
.card-grid.cols-3{grid-template-columns:repeat(3,1fr)}
.card-grid.cols-2{grid-template-columns:repeat(2,1fr)}
@media(max-width:980px){.card-grid.cols-3,.card-grid.cols-2{grid-template-columns:1fr}}

.metric{display:flex;justify-content:space-between;align-items:center;padding:6px 0;font-size:13px}
.metric+.metric{border-top:1px solid var(--bd)}
.metric .k{color:var(--mu);font-size:12px}
.metric .v{color:var(--tx);font-family:ui-monospace,"SF Mono",Menlo,monospace;font-size:12px;font-variant-numeric:tabular-nums;text-align:right}

.big-metric{font-size:18px;font-weight:600;font-variant-numeric:tabular-nums;color:var(--tx);margin-bottom:6px}

.progress .track{height:6px;background:var(--bg2);border-radius:3px;overflow:hidden}
.progress .fill{height:100%;background:var(--ac);transition:width .4s ease,background .2s;border-radius:3px}
.progress .fill.warn{background:var(--yl)}
.progress .fill.crit{background:var(--rd)}

/* Top consumers */
.hog-list{display:flex;flex-direction:column;gap:8px}
.hog{display:grid;grid-template-columns:1fr auto;gap:2px 12px;align-items:center}
.hog-name{font-family:ui-monospace,monospace;font-size:12px;color:var(--tx2);overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
.hog-val{font-family:ui-monospace,monospace;font-variant-numeric:tabular-nums;color:var(--mu);font-size:11px;white-space:nowrap}
.hog-bar{grid-column:1/-1;height:4px;background:var(--bg2);border-radius:2px;overflow:hidden}
.hog-fill{height:100%;background:var(--ac);transition:width .35s ease;border-radius:2px}
.hog-fill.warn{background:var(--yl)}
.hog-fill.crit{background:var(--rd)}
.hog-empty{color:var(--mu);font-size:12px;padding:14px 4px;text-align:center}

/* Issues */
.issue{display:flex;justify-content:space-between;align-items:center;padding:10px 14px;background:var(--card);border:1px solid var(--bd);border-left:3px solid var(--rd);border-radius:6px;margin-bottom:6px}
.issue .un{font-family:ui-monospace,monospace;font-size:13px}
.issue .desc{font-size:11px;color:var(--mu);margin-top:2px}
.no-issues{color:var(--mu);font-size:12px;padding:12px;text-align:center;background:var(--gn-bg);border:1px solid rgba(34,197,94,.2);border-radius:6px}

/* Power section */
.power{display:flex;justify-content:space-between;align-items:center;gap:14px;flex-wrap:wrap}
.power-status{display:flex;align-items:center;gap:9px;font-size:13px;color:var(--tx2)}
.power-status .led{width:9px;height:9px;border-radius:50%;background:var(--gn);box-shadow:0 0 8px rgba(34,197,94,.4)}
.power-status.scheduled{color:var(--yl)}
.power-status.scheduled .led{background:var(--yl);box-shadow:0 0 10px rgba(234,179,8,.6)}
.power-actions{display:flex;align-items:center;gap:6px;flex-wrap:wrap}
.power-actions input{background:var(--bg2);border:1px solid var(--bd);color:var(--tx);padding:7px 10px;border-radius:6px;font-size:12px;width:80px;outline:none;font-family:inherit;font-variant-numeric:tabular-nums}
.power-actions input:focus{border-color:var(--ac)}

/* Toolbar */
.toolbar{display:flex;gap:10px;align-items:center;margin-bottom:12px;flex-wrap:wrap}
.toolbar input{flex:1;min-width:200px;background:var(--card);border:1px solid var(--bd);color:var(--tx);padding:9px 12px;border-radius:6px;outline:none;font-size:13px}
.toolbar input:focus{border-color:var(--ac)}
.toolbar .hint{flex:1;color:var(--mu);font-size:12px}
.toolbar .cnt{color:var(--mu);font-size:12px;font-variant-numeric:tabular-nums}

/* Buttons */
.btn{display:inline-flex;align-items:center;gap:6px;padding:7px 12px;background:var(--card);border:1px solid var(--bd);border-radius:6px;color:var(--tx);font-size:12px;cursor:pointer;transition:all .12s;font-family:inherit}
.btn:hover{background:var(--card-hi);border-color:var(--bd-hi)}
.btn.primary{background:var(--ac);color:var(--bg);border-color:var(--ac);font-weight:600}
.btn.primary:hover{filter:brightness(1.1)}
.btn.danger{color:var(--rd)}
.btn.danger:hover{border-color:var(--rd);background:var(--rd-bg)}
.btn.ghost{background:transparent;border-color:transparent;color:var(--mu)}
.btn.ghost:hover{background:var(--card-hi);color:var(--tx)}

/* Container rows */
.row{background:var(--card);border:1px solid var(--bd);border-radius:8px;margin-bottom:6px;overflow:hidden;transition:border-color .15s}
.row:hover{border-color:var(--bd-hi)}
.row-head{display:grid;grid-template-columns:14px 1fr auto auto;gap:12px;align-items:center;padding:11px 14px;cursor:pointer;user-select:none}
.row-head:hover{background:rgba(255,255,255,.02)}
.chev{color:var(--mu);transition:transform .15s;font-size:10px}
.row.open .chev{transform:rotate(90deg)}
.row-title{display:flex;align-items:center;gap:10px;font-weight:600}
.dot{width:8px;height:8px;border-radius:50%;background:var(--mu);flex-shrink:0}
.dot.running{background:var(--gn);box-shadow:0 0 8px rgba(34,197,94,.5)}
.dot.exited,.dot.dead,.dot.created{background:var(--rd)}
.dot.paused,.dot.restarting{background:var(--yl)}
.row-sub{font-size:11px;color:var(--mu);font-family:ui-monospace,monospace;margin-top:3px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;max-width:60ch}
.row-stats{display:flex;gap:14px;font-size:11px;color:var(--mu);font-variant-numeric:tabular-nums;white-space:nowrap;align-items:center}
.row-stats b{color:var(--tx);font-weight:500}
.row-acts{display:flex;gap:4px}
.ib{width:30px;height:30px;display:inline-flex;align-items:center;justify-content:center;background:transparent;border:1px solid var(--bd);border-radius:5px;cursor:pointer;color:var(--mu);font-size:12px;transition:all .12s}
.ib:hover{background:var(--hi);color:var(--tx)}
.ib.gn:hover{color:var(--gn);border-color:var(--gn);background:var(--gn-bg)}
.ib.rd:hover{color:var(--rd);border-color:var(--rd);background:var(--rd-bg)}
.ib.yl:hover{color:var(--yl);border-color:var(--yl);background:var(--yl-bg)}
.row-body{display:none;padding:14px;border-top:1px solid var(--bd);background:rgba(0,0,0,.18)}
.row.open .row-body{display:block}
.subtabs{display:flex;gap:2px;border-bottom:1px solid var(--bd);margin-bottom:12px}
.subtab{padding:7px 13px;cursor:pointer;color:var(--mu);border-bottom:2px solid transparent;font-size:12px}
.subtab.active{color:var(--ac);border-bottom-color:var(--ac)}
.subtab-content{display:none}
.subtab-content.active{display:block}
.kv{display:grid;grid-template-columns:130px 1fr;gap:6px 14px;font-size:12px}
.kv>div:nth-child(odd){color:var(--mu)}
.kv>div:nth-child(even){font-family:ui-monospace,monospace;font-size:11px;word-break:break-all;color:var(--tx2)}
.logs-pre{background:#000;border:1px solid var(--bd);border-radius:6px;padding:10px;font-family:ui-monospace,monospace;font-size:11px;max-height:380px;overflow:auto;white-space:pre-wrap;color:#cbd5e1;margin-top:8px;line-height:1.4}

.empty{color:var(--mu);padding:40px;text-align:center;font-size:13px}

/* Commands */
.cmd-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(260px,1fr));gap:12px}
.cmd{background:var(--card);border:1px solid var(--bd);border-radius:8px;padding:16px;display:flex;flex-direction:column;gap:8px;transition:all .15s;position:relative;overflow:hidden}
.cmd:hover{border-color:var(--ac);transform:translateY(-1px);box-shadow:0 4px 12px rgba(0,0,0,.2)}
.cmd::before{content:"";position:absolute;top:0;left:0;width:3px;height:100%;background:var(--ac)}
.cmd.host::before{background:var(--rd)}
.cmd.container::before{background:var(--pp)}
.cmd .ic{font-size:24px;line-height:1}
.cmd .nm{font-weight:600;font-size:13px}
.cmd .ds{color:var(--mu);font-size:11px;flex:1;line-height:1.4}
.cmd .meta{color:var(--mu);font-size:10px;font-family:ui-monospace,monospace;background:var(--bg2);padding:4px 8px;border-radius:4px;overflow:hidden;white-space:nowrap;text-overflow:ellipsis;border:1px solid var(--bd)}
.cmd .runner-tag{font-size:9px;font-weight:700;letter-spacing:.06em;text-transform:uppercase;padding:2px 6px;border-radius:3px;display:inline-block;width:fit-content}
.cmd .runner-tag.self{background:var(--ac-bg);color:var(--ac)}
.cmd .runner-tag.host{background:var(--rd-bg);color:var(--rd)}
.cmd .runner-tag.container{background:var(--pp-bg);color:var(--pp)}

/* Modal */
.modal{display:none;position:fixed;inset:0;background:rgba(0,0,0,.7);z-index:200;align-items:center;justify-content:center;padding:20px;backdrop-filter:blur(4px)}
.modal.show{display:flex;animation:fadeIn .15s ease}
.mbox{background:var(--card);border:1px solid var(--bd);border-radius:10px;width:100%;max-width:820px;max-height:85vh;overflow:hidden;display:flex;flex-direction:column;box-shadow:0 20px 50px rgba(0,0,0,.5)}
.mhd{padding:14px 18px;border-bottom:1px solid var(--bd);display:flex;justify-content:space-between;align-items:center;font-weight:600;font-size:14px}
.mbd{padding:18px;overflow:auto;flex:1}

/* Toast */
.toast{position:fixed;bottom:18px;right:18px;background:var(--card);border:1px solid var(--bd);padding:10px 16px;border-radius:8px;font-size:13px;z-index:300;opacity:0;transform:translateY(8px);transition:all .2s;pointer-events:none;max-width:480px;box-shadow:0 8px 20px rgba(0,0,0,.4)}
.toast.show{opacity:1;transform:translateY(0)}
.toast.ok{border-color:var(--gn);color:var(--gn)}
.toast.err{border-color:var(--rd);color:var(--rd)}

/* Scrollbars */
::-webkit-scrollbar{width:9px;height:9px}
::-webkit-scrollbar-thumb{background:var(--hi);border-radius:5px}
::-webkit-scrollbar-thumb:hover{background:var(--bd-hi)}
::-webkit-scrollbar-track{background:transparent}
```

> 💡 To re-theme, edit only the `:root{}` block. The accent color (`--ac`) propagates to active tabs, primary buttons, focused inputs, KDE LED, and progress bars.

---

## Step 9 — Panel: JavaScript App

Vanilla JS. ~400 lines. State-managed with a single `App` object. No build step.

**File:** `~/docker/homepage/panel/static/app.js`

```javascript
const $ = id => document.getElementById(id);
const esc = s => (s == null ? '' : String(s)).replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));
const pctOf = s => { const m = (s||'').toString().match(/\((\d+(?:\.\d+)?)%\)/); return m ? +m[1] : 0; };
const barCls = p => p>=90?'crit':p>=75?'warn':'';

const App = {
  state: { containers: [], commands: [], overview: {}, openName: null, currentTab: 'overview' },

  init() {
    document.querySelectorAll('.tab').forEach(t => t.addEventListener('click', () => this.switchTab(t.dataset.page)));
    $('q').addEventListener('input', () => this.renderContainers());
    this.refreshAll();
    setInterval(() => this.refreshOverview(), 4000);
    setInterval(() => { if (this.state.currentTab === 'containers') this.refreshContainers(); }, 6000);
    setInterval(() => this.tickClock(), 1000);
    this.tickClock();
  },

  switchTab(page) {
    this.state.currentTab = page;
    document.querySelectorAll('.tab').forEach(t => t.classList.toggle('active', t.dataset.page === page));
    document.querySelectorAll('.page').forEach(p => p.classList.toggle('active', p.id === 'page-'+page));
    if (page === 'commands') this.loadCommands();
    if (page === 'containers') this.refreshContainers();
  },

  tickClock() {
    const d = new Date();
    const day = d.toLocaleDateString('en-GB',{weekday:'short'});
    const time = d.toLocaleTimeString('en-GB',{hour:'2-digit',minute:'2-digit'});
    $('hb-clock').textContent = `${day} ${time}`;
  },

  async refreshAll() { await Promise.all([this.refreshOverview(), this.refreshContainers(), this.loadCommands()]); },

  async refreshOverview() {
    try {
      this.state.overview = await (await fetch('/api/overview')).json();
      this.renderOverview(); this.renderHeader();
    } catch (e) {}
  },

  renderHeader() {
    const o = this.state.overview, m = o.metrics || {}, c = o.containers || {};
    const failed = parseInt(m.systemd_failed || 0, 10);
    const stopped = c.stopped || 0;
    const healthy = failed === 0 && stopped === 0;
    const pill = $('pill-health');
    pill.className = 'pill ' + (healthy ? 'healthy' : 'alert');
    if (healthy) {
      pill.innerHTML = `● <b>All Healthy</b>`;
    } else {
      const parts = [];
      if (failed) parts.push(`${failed} failed unit${failed>1?'s':''}`);
      if (stopped) parts.push(`${stopped} stopped`);
      pill.innerHTML = `⚠ <b>${parts.join(' · ')}</b>`;
    }

    const ses = m.display_state || 'idle';
    $('pill-session').className = 'pill session ' + ses;
    $('session-label').textContent = m.display_label || 'Idle';

    $('hb-cpu').textContent = m.cpu_pct || '—';
    $('hb-gpu').textContent = m.gpu_busy || '—';
    $('hb-ram').textContent = m.ram_pct || '—';
  },

  renderOverview() {
    const o = this.state.overview, m = o.metrics || {};
    this.renderHogs('top-cpu', o.top_cpu || []);
    this.renderHogs('top-mem', o.top_mem || []);

    $('hw-cpu-pct').textContent = m.cpu_pct || '—';
    $('hw-cpu-freq').textContent = m.cpu_freq || '—';
    $('hw-cpu-temp').textContent = m.cpu_temp || '—';
    $('hw-cpu-power').textContent = m.cpu_power || '—';
    $('hw-gpu-busy').textContent = m.gpu_busy || '—';
    $('hw-gpu-vram').textContent = m.gpu_vram || '—';
    $('hw-gpu-temp').textContent = m.gpu_temp || '—';
    $('hw-gpu-power').textContent = m.gpu_power || '—';
    $('hw-ram-usage').textContent = m.ram_usage ? `${m.ram_usage} (${m.ram_pct||'—'})` : '—';
    $('hw-ram-temp').textContent = m.ram_temp || '—';
    $('hw-nvme-temp').textContent = m.nvme_temp || '—';
    $('hw-load').textContent = m.load_avg || '—';

    $('st-os-text').textContent = m.os_storage || '—';
    $('st-data-text').textContent = m.data_storage || '—';
    const osP = pctOf(m.os_storage), dP = pctOf(m.data_storage);
    $('st-os-bar').style.width = osP + '%';
    $('st-os-bar').className = 'fill ' + barCls(osP);
    $('st-data-bar').style.width = dP + '%';
    $('st-data-bar').className = 'fill ' + barCls(dP);

    $('sv-nc-state').textContent = m.nextcloud_status || '—';
    $('sv-nc-backup').textContent = m.backup_status || '—';
    $('sv-npm').textContent = m.npm_status || '—';
    $('sv-cert').textContent = m.cert_status || '—';
    $('sv-wan').textContent = m.wan_ip || '—';
    $('sv-ssh').textContent = m.ssh_conn || '0';
    $('sv-f2b').textContent = m.f2b_banned || '0';

    const failed = parseInt(m.systemd_failed || 0, 10);
    const units = (m.systemd_failed_units || '').split(/,\s*/).filter(u => u && u !== 'all healthy');
    const list = $('issues-list');
    if (failed > 0 && units.length) {
      list.innerHTML = units.map(u => {
        const name = u.replace(/\s.*/, '');
        return `<div class="issue">
          <div><div class="un">${esc(u)}</div><div class="desc">systemd unit failed</div></div>
          <button class="btn danger" onclick="App.restartUnit('${esc(name)}')">↻ Restart</button>
        </div>`;
      }).join('');
    } else {
      list.innerHTML = `<div class="no-issues">✓ No active issues — all host services are healthy.</div>`;
    }

    const sched = m.reboot_scheduled || '';
    const ps = $('power-status');
    if (sched) {
      ps.classList.add('scheduled');
      $('reboot-status').textContent = `Reboot scheduled · ${sched}`;
      $('cancel-btn').style.display = '';
    } else {
      ps.classList.remove('scheduled');
      $('reboot-status').textContent = 'No reboot scheduled';
      $('cancel-btn').style.display = 'none';
    }
  },

  renderHogs(elId, items) {
    const el = $(elId);
    if (!items || !items.length) { el.innerHTML = '<div class="hog-empty">No data</div>'; return; }
    el.innerHTML = items.map(it => {
      const p = +it.pct || 0;
      const cls = barCls(p);
      const right = it.raw || `${p.toFixed(1)}%`;
      return `<div class="hog">
        <div class="hog-name" title="${esc(it.name)}">${esc(it.name)}</div>
        <div class="hog-val">${esc(right)}</div>
        <div class="hog-bar"><div class="hog-fill ${cls}" style="width:${Math.min(p,100)}%"></div></div>
      </div>`;
    }).join('');
  },

  async restartUnit(unit) {
    if (!confirm(`Restart systemd unit "${unit}"?`)) return;
    this.toast(`Restarting ${unit}…`);
    const j = await (await fetch(`/api/system/restart-unit/${encodeURIComponent(unit)}`, {method:'POST'})).json();
    this.toast(j.ok ? `${unit}: restarted ✓` : 'Failed: ' + (j.err||j.stderr||''), j.ok ? 'ok' : 'err');
    setTimeout(() => this.refreshOverview(), 1500);
  },

  async scheduleReboot() {
    const m = parseInt($('reboot-mins').value || '0', 10);
    if (!m || m < 1) { this.toast('Enter minutes ≥ 1', 'err'); return; }
    if (!confirm(`Schedule reboot in ${m} minute${m>1?'s':''}?`)) return;
    const r = await fetch('/api/power/reboot', {method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({minutes: m})});
    const j = await r.json();
    this.toast(j.ok ? `Reboot scheduled in ${m}m ✓` : 'Failed: ' + (j.err||j.stderr||''), j.ok?'ok':'err');
    setTimeout(() => this.refreshOverview(), 1500);
  },
  async cancelReboot() {
    if (!confirm('Cancel scheduled reboot?')) return;
    const j = await (await fetch('/api/power/cancel', {method:'POST'})).json();
    this.toast(j.ok ? 'Reboot canceled ✓' : 'Failed', j.ok?'ok':'err');
    setTimeout(() => this.refreshOverview(), 1500);
  },
  async rebootNow() {
    if (!confirm('⚠ REBOOT THE SERVER NOW?\n\nAll services will go down briefly.')) return;
    if (!confirm('Are you SURE? Last chance to cancel.')) return;
    const r = await fetch('/api/power/reboot', {method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({minutes: 0})});
    const j = await r.json();
    this.toast(j.ok ? 'Reboot initiated…' : 'Failed: ' + (j.err||j.stderr||''), j.ok?'ok':'err');
  },

  async refreshContainers() {
    try { this.state.containers = await (await fetch('/api/containers')).json(); this.renderContainers(); }
    catch (e) { this.toast('Connection error', 'err'); }
  },
  renderContainers() {
    const q = $('q').value.toLowerCase();
    const all = this.state.containers;
    const f = all.filter(c => (c.Names||'').toLowerCase().includes(q) || (c.Image||'').toLowerCase().includes(q));
    f.sort((a, b) => (a.Names||'').localeCompare(b.Names||''));
    $('cnt').textContent = `${f.length} / ${all.length}`;
    $('tab-count-c').textContent = all.length;
    const list = $('container-list');
    if (!f.length) { list.innerHTML = '<div class="empty">No containers match.</div>'; return; }
    list.innerHTML = f.map(c => {
      const n = c.Names, st = (c.State||'').toLowerCase();
      const s = c.stats || {}, isOpen = this.state.openName === n;
      const cpu = s.CPUPerc || '—';
      const mem = s.MemUsage ? s.MemUsage.split(' / ')[0] : '—';
      return `<div class="row ${isOpen?'open':''}">
        <div class="row-head" onclick="App.toggleRow('${esc(n)}')">
          <span class="chev">▶</span>
          <div style="min-width:0">
            <div class="row-title"><span class="dot ${st}"></span>${esc(n)}</div>
            <div class="row-sub">${esc(c.Image)} · ${esc(c.Status||'')}</div>
          </div>
          <div class="row-stats"><span><b>${esc(cpu)}</b> cpu</span><span><b>${esc(mem)}</b> ram</span></div>
          <div class="row-acts" onclick="event.stopPropagation()">
            <button class="ib gn" title="Start" onclick="App.act('${esc(n)}','start')">▶</button>
            <button class="ib rd" title="Stop" onclick="App.act('${esc(n)}','stop')">■</button>
            <button class="ib yl" title="Restart" onclick="App.act('${esc(n)}','restart')">↻</button>
          </div>
        </div>
        <div class="row-body">${isOpen ? this.renderRowBody(c) : ''}</div>
      </div>`;
    }).join('');
  },
  renderRowBody(c) {
    const n = c.Names;
    return `<div class="subtabs">
      <div class="subtab active" onclick="App.subtab(this,'st-i-${n}')">Info</div>
      <div class="subtab" onclick="App.subtab(this,'st-l-${n}');App.loadLogs('${n}')">Logs</div>
    </div>
    <div class="subtab-content active" id="st-i-${n}">
      <div class="kv">
        <div>Image</div><div>${esc(c.Image)}</div>
        <div>State</div><div>${esc(c.State)} — ${esc(c.Status)}</div>
        <div>Created</div><div>${esc(c.CreatedAt)}</div>
        <div>Ports</div><div>${esc(c.Ports||'—')}</div>
        <div>Networks</div><div>${esc(c.Networks||'—')}</div>
        <div>Mounts</div><div>${esc(c.Mounts||'—').split(',').join('<br>')}</div>
        <div>Labels</div><div>${esc(c.Labels||'—').split(',').slice(0,10).join('<br>')||'—'}</div>
      </div>
      <div style="margin-top:12px;display:flex;gap:6px;flex-wrap:wrap">
        <button class="btn" onclick="App.act('${n}','pause')">⏸ Pause</button>
        <button class="btn" onclick="App.act('${n}','unpause')">▶ Unpause</button>
        <button class="btn danger" onclick="if(confirm('Force kill ${n}?'))App.act('${n}','kill')">✕ Kill</button>
      </div>
    </div>
    <div class="subtab-content" id="st-l-${n}">
      <button class="btn" onclick="App.loadLogs('${n}')">↻ Reload Logs</button>
      <pre class="logs-pre" id="lp-${n}">Loading…</pre>
    </div>`;
  },
  subtab(el, id) {
    const p = el.closest('.row-body');
    p.querySelectorAll('.subtab').forEach(x => x.classList.remove('active'));
    p.querySelectorAll('.subtab-content').forEach(x => x.classList.remove('active'));
    el.classList.add('active'); $(id).classList.add('active');
  },
  toggleRow(n) { this.state.openName = this.state.openName === n ? null : n; this.renderContainers(); },
  async act(name, action) {
    this.toast(`${action} ${name}…`);
    const j = await (await fetch(`/api/action/${name}/${action}`, {method:'POST'})).json();
    this.toast(j.ok ? `${name}: ${action} ✓` : 'Error: ' + (j.err||''), j.ok ? 'ok' : 'err');
    setTimeout(() => this.refreshContainers(), 500);
  },
  async loadLogs(name) {
    const r = await fetch(`/api/logs/${name}`); const el = $(`lp-${name}`); if (el) el.textContent = await r.text();
  },

  async loadCommands() {
    try { this.state.commands = await (await fetch('/api/commands')).json(); }
    catch (e) { this.state.commands = []; }
    $('tab-count-cmd').textContent = this.state.commands.length;
    const list = $('cmd-list');
    if (!this.state.commands.length) { list.innerHTML = '<div class="empty" style="grid-column:1/-1">No commands defined.</div>'; return; }
    list.innerHTML = this.state.commands.map(c => {
      const r = c.runner || 'self';
      const cls = r === 'host' ? 'host' : r.startsWith('container:') ? 'container' : 'self';
      const lbl = r === 'host' ? 'host' : r.startsWith('container:') ? r.split(':')[1] : 'panel';
      return `<div class="cmd ${cls}">
        <div class="ic">${esc(c.icon||'⚡')}</div>
        <div class="nm">${esc(c.name||c.id)}</div>
        <div class="ds">${esc(c.description||'')}</div>
        <span class="runner-tag ${cls}">${esc(lbl)}</span>
        <div class="meta" title="${esc(c.cmd||'')}">${esc(c.cmd||'')}</div>
        <button class="btn primary" onclick="App.runCmd('${esc(c.id)}',${c.confirm?true:false})">▶ Run</button>
      </div>`;
    }).join('');
  },
  async runCmd(id, confirmFirst) {
    const c = this.state.commands.find(x => x.id === id); if (!c) return;
    if (confirmFirst && !confirm(`Run "${c.name}"?\n\n${c.cmd}`)) return;
    this.showModal(`⚡ ${c.name}`, '<div style="color:var(--mu)">Running…</div>');
    const j = await (await fetch(`/api/commands/${id}/run`, {method:'POST'})).json();
    const html = `<div style="margin-bottom:10px;font-weight:600;color:${j.ok?'var(--gn)':'var(--rd)'}">${j.ok?'✓ Success':'✗ Failed'} (exit ${j.code??'?'})</div>` +
      (j.stdout ? `<div style="color:var(--mu);font-size:11px;margin-bottom:4px;text-transform:uppercase;letter-spacing:.06em">stdout</div><pre class="logs-pre">${esc(j.stdout)||'(empty)'}</pre>` : '') +
      (j.stderr ? `<div style="color:var(--mu);font-size:11px;margin:10px 0 4px;text-transform:uppercase;letter-spacing:.06em">stderr</div><pre class="logs-pre">${esc(j.stderr)}</pre>` : '') +
      (j.err ? `<div style="color:var(--rd)">${esc(j.err)}</div>` : '');
    $('mbody').innerHTML = html;
  },

  showModal(title, html) { $('mtitle').textContent = title; $('mbody').innerHTML = html; $('modal').classList.add('show'); },
  closeModal() { $('modal').classList.remove('show'); },
  toast(msg, type) { const e = $('toast'); e.textContent = msg; e.className = 'toast show ' + (type||''); clearTimeout(e._t); e._t = setTimeout(() => e.className = 'toast', 2400); }
};

document.addEventListener('DOMContentLoaded', () => App.init());
```

---

## Step 10 — Quick Commands

The commands tab reads from a JSON file. Edit it via SSH at any time — no rebuild needed.

**File:** `~/docker/homepage/data/commands.json`

```json
[
  {
    "id": "host-status", "name": "Host Status", "icon": "📈",
    "description": "uptime · free memory · top CPU processes",
    "runner": "host",
    "cmd": "uptime && echo && free -h && echo && ps -eo pid,%cpu,%mem,comm --sort=-%cpu | head -10"
  },
  {
    "id": "list-failed", "name": "List Failed Units", "icon": "⚠",
    "description": "Show all failed systemd units with reasons",
    "runner": "host", "cmd": "systemctl --failed --no-legend"
  },
  {
    "id": "journal-errors", "name": "Recent Errors", "icon": "📋",
    "description": "Last 50 error-priority log lines",
    "runner": "host", "cmd": "journalctl -p err -n 50 --no-pager"
  },
  {
    "id": "disk-top10", "name": "Top 10 Largest", "icon": "💾",
    "description": "10 largest items in /var",
    "runner": "host", "cmd": "du -sh /var/* 2>/dev/null | sort -rh | head -10"
  },
  {
    "id": "docker-prune", "name": "Docker Cleanup", "icon": "🧹",
    "description": "Remove unused images / networks / build cache",
    "runner": "self", "cmd": "docker system prune -af", "confirm": true
  },
  {
    "id": "restart-metrics", "name": "Restart Metrics", "icon": "📊",
    "description": "Restart metrics collector on host",
    "runner": "host", "cmd": "systemctl restart homepage-metrics", "confirm": true
  }
]
```

### Schema reference

| Field | Required | Description |
|---|---|---|
| `id` | ✅ | Unique slug, `[A-Za-z0-9_-]+` only |
| `name` | ✅ | Display name |
| `icon` | – | Emoji shown on the card |
| `description` | – | Short subtitle |
| `runner` | – | `self` (default — runs in panel container), `host` (via host-runner), or `container:<name>` |
| `cmd` | ✅ | Shell command to execute |
| `confirm` | – | If `true`, shows a confirmation prompt before running |
| `timeout` | – | Seconds before timeout (default `60`) |

### Runner color codes (in the UI)

- 🔵 **Blue** stripe = `self` — safe, runs inside the panel container with `docker` CLI
- 🟣 **Purple** stripe = `container:<name>` — runs `docker exec` inside the named container
- 🔴 **Red** stripe = `host` — runs as **root on the host** via `host-runner`

### Optional: streaming/desktop launcher commands

If you have host scripts like `/usr/local/bin/start-kde.sh` for Sunshine streaming workflows, add entries like:

```json
{
  "id": "session-kde", "name": "Launch KDE Plasma", "icon": "🖥",
  "description": "Start KDE for Sunshine streaming",
  "runner": "host", "cmd": "/usr/local/bin/start-kde.sh",
  "timeout": 180, "confirm": true
}
```

---

## Step 11 — Build & Launch

```bash
cd ~/docker/homepage
docker compose build
docker compose up -d
```

Confirm both services are running:

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

You should see `server-panel` (port `0.0.0.0:3000`) and `host-runner` (no ports — uses host network).

---

## Step 12 — Verification

### Endpoint smoke tests

```bash
# Should return JSON with system metrics
curl -s http://localhost:3000/api/overview | head -c 400

# Should return your container list
curl -s http://localhost:3000/api/containers | head -c 200

# Should return your commands
curl -s http://localhost:3000/api/commands
```

### Browser test

Open `http://<server-ip>:3000` (or `http://localhost:3000` on the host).

You should see:

- **Header** — brand · health pill · session LED · CPU/GPU/RAM live · clock
- **🏠 Overview** — top consumers, hardware, storage, services, issues, power
- **📦 Containers** — full list, filterable, expandable
- **⚡ Commands** — your shortcut grid

### Logs (if anything misbehaves)

```bash
docker logs server-panel --tail=50
docker logs host-runner --tail=20
sudo journalctl -u homepage-metrics --tail=30
```

---

## 🎨 Customization

### Change theme colors

Edit `~/docker/homepage/panel/static/style.css` — modify only the `:root{}` block. The accent (`--ac`) propagates everywhere. Reload the page; no rebuild needed.

### Add a server logo / rename

Edit `~/docker/homepage/panel/templates/index.html`:

- `<div class="logo">HS</div>` — your initials
- `<div class="title">HS Server</div>` — your server name

Then:
```bash
docker compose build panel && docker compose up -d
```

### Add quick commands

Just edit `~/docker/homepage/data/commands.json` — changes appear on next browser refresh of the **Commands** tab. No rebuild, no restart.

### Tune metric refresh rate

In `metrics-collector.sh`:
- `REFRESH=2` — fast loop (CPU/GPU/temps)
- `SLOW_EVERY=15` — slow probes (WAN IP, certs, NPM, Nextcloud) run every `REFRESH × SLOW_EVERY` seconds (default 30 s)

After editing: `sudo systemctl restart homepage-metrics`.

### Add new metric fields

1. Add the field to the JSON in `metrics-collector.sh`
2. Add a `<div class="metric"><span class="k">…</span><span class="v" id="hw-newthing">—</span></div>` in `index.html`
3. Add `$('hw-newthing').textContent = m.new_field || '—';` in `app.js → renderOverview()`
4. `docker compose build panel && docker compose up -d`

---

## 🔧 Troubleshooting

### `nsenter: unrecognized option: a`
Already addressed — `app.py` uses explicit flags (`-m -u -i -n -p`) compatible with both util-linux and BusyBox. If you ever modify the code, do **not** revert to `-a`.

### Permission denied writing to `config/` or `data/`
A previous root-owned container left behind files:
```bash
sudo chown -R "$USER:$USER" ~/docker/homepage
```

### `data.json` shows `"calc..."` or stays empty
RAPL energy file isn't readable. The script auto-falls-back to `n/a` for `cpu_power`. Verify:
```bash
ls -la /sys/class/powercap/intel-rapl:0/energy_uj
```
If missing, your CPU may not expose RAPL. Power will show `n/a`; everything else works.

### Storage progress bars all red
Your storage values aren't using the `(NN%)` pattern. Check `df -h` output:
```bash
df -h /var
```
If your locale prints `,` instead of `.` for percentages, adjust the `pctOf()` regex in `app.js`.

### Container shows running but stats are blank
Docker stats returns nothing for paused/stopped containers. Normal behavior.

### `homepage-metrics.service` keeps restarting
Check logs:
```bash
sudo journalctl -u homepage-metrics -n 50 --no-pager
```
Most common cause: missing `lm_sensors` package or wrong `WorkingDirectory` path in the service file.

### Host commands fail with "host-runner not found"
```bash
docker ps | grep host-runner
```
If absent: `docker compose up -d host-runner`.

### Reboot button does nothing
Ensure the host-runner has the correct namespaces. Test manually:
```bash
docker exec host-runner nsenter -t 1 -m -u -i -n -p -- shutdown -r +99 "test"
docker exec host-runner nsenter -t 1 -m -u -i -n -p -- shutdown -c
```
If that works from CLI but not the panel, check `docker logs server-panel` for the actual error.

---

## 🧹 Maintenance

### Update dependencies (Flask, Alpine packages)

```bash
cd ~/docker/homepage
docker compose pull
docker compose build --no-cache
docker compose up -d
```

### Backup

The whole setup is portable — back up `~/docker/homepage/` and `/etc/systemd/system/homepage-metrics.service`. To restore on a new host:
1. Copy the directory
2. Adjust paths in the systemd unit
3. `docker compose up -d --build`
4. `sudo systemctl daemon-reload && sudo systemctl enable --now homepage-metrics`

### Logs rotation

The panel and host-runner log to Docker's default driver. To cap log size, add to `docker-compose.yml` under each service:
```yaml
logging:
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
```

---

## 🔐 Security Notes

- Port 3000 has **no authentication**. Bind it to LAN only or front it with an authenticated reverse proxy (Nginx Proxy Manager + Authelia, Caddy + basic auth, etc.).
- The **`host-runner` sidecar** has full root access to the host. Anyone who reaches the panel UI can run arbitrary commands as root via the Commands tab. Treat panel access as equivalent to root SSH.
- All API path parameters are validated against strict regexes before any subprocess call — no shell injection from URLs.
- Command bodies in `commands.json` are executed verbatim — only put commands there you trust, and edit the file only via SSH.

---

## ✅ Done

You now have a fully self-hosted, single-page server admin panel with:

- Live system metrics (CPU/GPU/RAM/disks/temps/load/power)
- Display session monitoring (KDE / Gamescope / Idle)
- Top resource consumer tracking
- Full Docker container management (start/stop/restart/pause/kill/logs/info)
- Custom command shortcuts editable via SSH
- Failed-unit detection with one-click restart
- Reboot now / scheduled reboot / cancel scheduled
- Zero external dependencies (no Homepage, no Portainer, no Grafana)

Bookmark `http://<server-ip>:3000` and enjoy.
