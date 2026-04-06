


---
icon: material/gpu
---

# GPU Power Management

Fully automated eGPU on-demand power lifecycle — with CPU TDP scaling, undervolt, LED disable, and a reliable "Secretary-Boss" AI architecture.

!!! abstract "What this page adds"
    - **eGPU on/off without rebooting** via PCIe bus remove + smart plug
    - **CPU TDP switching** — 35 W idle → 65 W gaming, no reboot required
    - **GPU undervolt + power cap** — ~95 % performance at ~85 % power via LACT
    - **Automatic idle shutdown** — eGPU powers off after configurable inactivity
    - **Secretary-Boss AI Fallback** — iGPU runs a fast "secretary" model instantly; eGPU spins up automatically for the "boss" model via tool-calling.
    - **Home Assistant integration** — dashboard GPU on/off/status buttons

!!! warning "Hardware required before proceeding"
    A **smart plug** on the be quiet! 750 W PSU power cord is mandatory — it is
    the mechanism that cuts and restores actual wall power to the GPU.
    The guide supports **Shelly Plug S** and **TP-Link Kasa**. Any Home 
    Assistant–compatible plug works with minor script adaptation.

---

## How it works

OCuLink does not support hot-plug. This is worked around entirely in software:

| Phase | What happens |
|---|---|
| **GPU off** | `ollama-egpu` stopped → Driver unbound → PCIe device removed → smart plug cuts PSU |
| **GPU on** | Smart plug restores PSU → PCIe rescan → driver re-binds → `ollama-egpu` started |
| **Idle off** | Gaming/AI session ends → 30-min countdown → auto `gpu-off` |
| **AI Fallback** | The iGPU (780M) runs an isolated `ollama-igpu` container that is *never* interrupted. |

---

## Prerequisites

- Phase 8 — Session Scripts complete
- Phase 12 — Docker, Ollama, and OpenWebUI installed
- Smart plug installed on the GPU PSU mains cable, with a known static IP
- eGPU dock powered on and eGPU present (needed for initial LACT configuration)

---

## 1 — One-Time Kernel Parameters

Unlock amdgpu overdrive (needed for power limit and voltage control) and allow ryzenadj to access the CPU SMU without root memory restrictions.

```bash
rpm-ostree kargs \
  --append-if-missing="amdgpu.ppfeaturemask=0xffffffff" \
  --append-if-missing="iomem=relaxed"
```

Verify the changes will apply:

```bash
rpm-ostree kargs
```

Both parameters must appear in the output. Then reboot **once**:

```bash
systemctl reboot
```

---

## 2 — Install LACT (GPU Tuning Daemon)

LACT persists settings across reboots via a system daemon. RDNA 4 support (`gfx1201`) requires kernel 6.13+.

### Install

```bash
flatpak install flathub io.github.ilya_zlobintsev.LACT
```

### Set up the system daemon

```bash
sudo systemctl enable --now lact
```

If that fails, launch the LACT GUI once — it will detect the missing service and offer to install it automatically:

```bash
flatpak run io.github.ilya_zlobintsev.LACT
```

### Verify RDNA 4 is detected

```bash
lspci -nn | grep "1002:7480"
```

---

## 3 — Configure GPU Tuning in LACT

Open the LACT GUI while the eGPU is online:

```bash
flatpak run io.github.ilya_zlobintsev.LACT
```

1. **Power limit:** Navigate to **GPU → OC/PM → Power**. Set limit to **290 W** (safe sweet-spot).
2. **Voltage offset:** Navigate to **GPU → OC/PM → Clocks**. Set **GPU voltage offset** to **−70 mV**.
3. **Fan curve:** Navigate to **GPU → OC/PM → Fan**. Enable **Zero RPM** for temperatures below 50 °C.
4. **Apply & Save.**

Restart the LACT daemon to confirm settings persist:

```bash
sudo systemctl restart lact
sleep 5
cat /sys/class/drm/card*/device/hwmon/hwmon*/power1_cap 2>/dev/null
```
*(Expected: a value around `290000000`)*

---

## 4 — Disable GPU RGB LEDs

### Option A — Physical toggle (most reliable)
The ASRock Taichi OC has a hardware **LED on/off button** on the card's PCB face. Press it once to toggle all onboard LEDs off permanently. **This is the recommended method.**

### Option B — OpenRGB (software)
```bash
flatpak install flathub org.openrgb.OpenRGB
flatpak run org.openrgb.OpenRGB --noautoconnect --list-devices
flatpak run org.openrgb.OpenRGB --noautoconnect --device 0 --mode static --color 000000
```

---

## 5 — Install ryzenadj (CPU TDP Control)

ryzenadj lets you change the Ryzen 8845HS TDP at runtime — no reboot.

```bash
rpm-ostree install ryzenadj
systemctl reboot
```

### Grant passwordless sudo

The TDP scripts run non-interactively. Add ryzenadj to the sudoers file:

```bash
RYADJ_PATH="$(which ryzenadj)"
printf "%s ALL=(ALL) NOPASSWD: %s\n" "$USER" "$RYADJ_PATH" \
  | sudo tee /etc/sudoers.d/ryzenadj > /dev/null
sudo chmod 440 /etc/sudoers.d/ryzenadj
sudo visudo -c -f /etc/sudoers.d/ryzenadj
```

---

## 6 — Detect Your eGPU PCI Address

The eGPU's PCI address is stable across reboots and required by the power scripts.

```bash
lspci -nn | grep -E "1002:(15bf|7480)"
```

Expected output:
```text
00:08.1 VGA compatible controller[0300]: AMD/ATI Phoenix3 [1002:15bf]   ← iGPU
01:00.0 VGA compatible controller [0300]: AMD/ATI Navi 48[1002:7480]    ← eGPU
```

Write your eGPU address (`01:00.0` above) and smart plug IP into a shared config file:

```bash
sudo mkdir -p /etc/gpu-power
sudo nano /etc/gpu-power/config
```

```bash
# /etc/gpu-power/config
# eGPU PCI address
EGPU_PCI="0000:01:00.0"

# Smart plug type: "shelly" or "kasa"
PLUG_TYPE="shelly"
PLUG_IP="192.168.1.50"

# Idle timeout in minutes before auto GPU-off (0 = disable)
IDLE_TIMEOUT_MIN=30

# CPU TDP in milliwatts
CPU_IDLE_STAPM=35000
CPU_IDLE_SLOW=38000
CPU_IDLE_FAST=42000

CPU_GAMING_STAPM=65000
CPU_GAMING_SLOW=68000
CPU_GAMING_FAST=75000
```

```bash
sudo chmod 640 /etc/gpu-power/config
```

---

## 7 — Install Smart Plug Controller

=== "Shelly (recommended)"

    Shelly devices have a built-in REST API. Test connectivity:
    ```bash
    curl -sf "http://192.168.1.50/relay/0?turn=off"
    curl -sf "http://192.168.1.50/relay/0?turn=on"
    ```

=== "TP-Link Kasa"

    Install the kasa CLI persistently:
    ```bash
    pipx install python-kasa
    ```
    Grant passwordless sudo:
    ```bash
    KASA_PATH="$(which kasa)"
    printf "%s ALL=(ALL) NOPASSWD: %s\n" "$USER" "$KASA_PATH" \
      | sudo tee -a /etc/sudoers.d/ryzenadj > /dev/null
    sudo visudo -c -f /etc/sudoers.d/ryzenadj
    ```

---

## 8 — Core Power Scripts

The core scripts handle the entire power lifecycle cleanly. Use the tabs below to copy each file.

=== "gpu-power-lib.sh"

    ```bash
    sudo nano /usr/local/lib/gpu-power-lib.sh
    ```

    ```bash
    #!/usr/bin/env bash
    GPU_CONFIG="/etc/gpu-power/config"

    load_config() {
        [[ -f "${GPU_CONFIG}" ]] || { echo "[gpu-lib] ERROR: config not found"; exit 1; }
        source "${GPU_CONFIG}"
    }

    log()  { echo "[gpu-power] $*"; }
    warn() { echo "[gpu-power] WARNING: $*" >&2; }
    die()  { echo "[gpu-power] ERROR: $*" >&2; exit 1; }

    plug_on() {
        log "Smart plug ON (${PLUG_TYPE} @ ${PLUG_IP})"
        case "${PLUG_TYPE}" in
            shelly) curl -sf "http://${PLUG_IP}/relay/0?turn=on" > /dev/null ;;
            kasa)   kasa --host "${PLUG_IP}" on > /dev/null ;;
        esac
    }

    plug_off() {
        log "Smart plug OFF (${PLUG_TYPE} @ ${PLUG_IP})"
        case "${PLUG_TYPE}" in
            shelly) curl -sf "http://${PLUG_IP}/relay/0?turn=off" > /dev/null ;;
            kasa)   kasa --host "${PLUG_IP}" off > /dev/null ;;
        esac
    }

    plug_state() {
        case "${PLUG_TYPE}" in
            shelly) curl -sf "http://${PLUG_IP}/relay/0" 2>/dev/null | grep -q '"ison":true' && echo "on" || echo "off" ;;
            kasa)   kasa --host "${PLUG_IP}" state 2>/dev/null | grep -qi "is on" && echo "on" || echo "off" ;;
        esac
    }

    egpu_drm_path() {
        local domain bus slot fn
        IFS=':.' read -r domain bus slot fn <<< "${EGPU_PCI}"
        local sysfs_addr="${domain}:${bus}:${slot}.${fn}"
        for drm_dir in /sys/class/drm/card*/device; do
            local link="$(readlink -f "${drm_dir}")"
            if [[ "${link}" == *"${sysfs_addr}"* ]]; then
                echo "${drm_dir}"; return 0
            fi
        done
        return 1
    }

    egpu_is_online() {
        [[ -e "/sys/bus/pci/devices/${EGPU_PCI}" ]]
    }

    set_cpu_tdp_idle() {
        log "CPU TDP → Idle (${CPU_IDLE_STAPM} mW)"
        sudo ryzenadj --stapm-limit="${CPU_IDLE_STAPM}" --slow-limit="${CPU_IDLE_SLOW}" --fast-limit="${CPU_IDLE_FAST}" >/dev/null 2>&1 || true
    }

    set_cpu_tdp_gaming() {
        log "CPU TDP → Gaming (${CPU_GAMING_STAPM} mW)"
        sudo ryzenadj --stapm-limit="${CPU_GAMING_STAPM}" --slow-limit="${CPU_GAMING_SLOW}" --fast-limit="${CPU_GAMING_FAST}" >/dev/null 2>&1 || true
    }

    apply_gpu_settings() {
        local drm_dev="$(egpu_drm_path)" || return 1
        local hwmon_path="${drm_dev}/hwmon/$(ls "${drm_dev}/hwmon/" | head -n 1)"
        
        if [[ -f "${hwmon_path}/power1_cap" ]]; then
            echo 290000000 | tee "${hwmon_path}/power1_cap" > /dev/null
        fi

        local od_path="${drm_dev}/pp_od_clk_voltage"
        if [[ -f "${od_path}" ]]; then
            echo "vo -70" | tee "${od_path}" > /dev/null
            echo "c" | tee "${od_path}" > /dev/null
        fi
        log "GPU tuning applied."
    }
    ```
    ```bash
    sudo chmod 644 /usr/local/lib/gpu-power-lib.sh
    ```

=== "gpu-on.sh"

    ```bash
    sudo nano /usr/local/bin/gpu-on.sh
    ```

    ```bash
    #!/usr/bin/env bash
    set -euo pipefail
    [[ "${EUID}" -eq 0 ]] || exec sudo "$0" "$@"

    source /usr/local/lib/gpu-power-lib.sh
    load_config

    readonly GPU_RESCAN_TIMEOUT=20
    readonly GPU_PSU_SETTLE=3

    if egpu_is_online; then
        apply_gpu_settings
        set_cpu_tdp_gaming
        exit 0
    fi

    plug_on
    sleep "${GPU_PSU_SETTLE}"

    echo 1 | tee /sys/bus/pci/rescan > /dev/null

    elapsed=0
    while (( elapsed < GPU_RESCAN_TIMEOUT )); do
        if egpu_is_online; then break; fi
        sleep 1; (( elapsed++ ))
    done

    egpu_is_online || die "eGPU did not enumerate."

    elapsed=0
    while (( elapsed < 10 )); do
        if [[ -d "/sys/bus/pci/devices/${EGPU_PCI}/drm" ]]; then break; fi
        sleep 1; (( elapsed++ ))
    done

    apply_gpu_settings
    set_cpu_tdp_gaming

    if [[ -f /run/gpu-idle-timer.pid ]]; then
        kill "$(cat /run/gpu-idle-timer.pid)" 2>/dev/null || true
        rm -f /run/gpu-idle-timer.pid
    fi

    # Start the eGPU AI Container if it exists
    if docker ps -a --format '{{.Names}}' | grep -q "^ollama-egpu$"; then
        log "Starting eGPU AI container (Boss model)..."
        docker start ollama-egpu >/dev/null || true
    fi

    echo -e "\n✓ eGPU online. CPU at gaming TDP."
    ```
    ```bash
    sudo chmod +x /usr/local/bin/gpu-on.sh
    ```

=== "gpu-off.sh"

    ```bash
    sudo nano /usr/local/bin/gpu-off.sh
    ```

    ```bash
    #!/usr/bin/env bash
    set -euo pipefail
    [[ "${EUID}" -eq 0 ]] || exec sudo "$0" "$@"

    source /usr/local/lib/gpu-power-lib.sh
    load_config

    readonly FORCE="${1:-}"

    if ! egpu_is_online; then
        plug_off; set_cpu_tdp_idle; exit 0
    fi

    if [[ "${FORCE}" != "--force" ]]; then
        active_session="$(loginctl list-sessions --no-legend | awk '$2 == 1000 && $4 != "-" { print $1 }' | head -n 1)"
        if [[ -n "${active_session}" ]]; then
            die "Graphical session active. Stop it first."
        fi
    fi

    # Stop the eGPU AI Container BEFORE unbinding driver to prevent kernel panic
    if docker ps -a --format '{{.Names}}' | grep -q "^ollama-egpu$"; then
        log "Stopping eGPU AI container..."
        docker stop ollama-egpu >/dev/null || true
    fi

    systemctl stop lact 2>/dev/null || true

    if [[ -e "/sys/bus/pci/devices/${EGPU_PCI}/driver" ]]; then
        echo "${EGPU_PCI}" | tee /sys/bus/pci/drivers/amdgpu/unbind > /dev/null
        sleep 1
    fi

    if [[ -e "/sys/bus/pci/devices/${EGPU_PCI}" ]]; then
        echo 1 | tee "/sys/bus/pci/devices/${EGPU_PCI}/remove" > /dev/null
        sleep 1
    fi

    plug_off
    systemctl start lact 2>/dev/null || true
    set_cpu_tdp_idle
    rm -f /run/gpu-idle-timer.pid /run/gpu-idle-since

    echo -e "\n✓ eGPU offline. CPU at idle TDP."
    ```
    ```bash
    sudo chmod +x /usr/local/bin/gpu-off.sh
    ```

=== "gpu-status.sh"

    ```bash
    sudo nano /usr/local/bin/gpu-status.sh
    ```

    ```bash
    #!/usr/bin/env bash
    set -euo pipefail
    source /usr/local/lib/gpu-power-lib.sh
    load_config

    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━ GPU Power Status ━━━━━━━━━━━━━━━━━━━━━━━━━━"
    if egpu_is_online; then
        echo "  eGPU:      ONLINE  (${EGPU_PCI})"
    else
        echo "  eGPU:      OFFLINE"
    fi
    echo "  GPU PSU:   $(plug_state 2>/dev/null | tr a-z A-Z)"
    stapm="$(sudo ryzenadj --info 2>/dev/null | awk '/STAPM LIMIT/ { print $4 }' | head -n 1)" || stapm="unknown"
    echo "  CPU STAPM: ${stapm} W"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    ```
    ```bash
    sudo chmod +x /usr/local/bin/gpu-status.sh
    ```

### Grant passwordless sudo to all scripts

```bash
for script in gpu-on.sh gpu-off.sh gpu-status.sh; do
    printf "%s ALL=(ALL) NOPASSWD: /usr/local/bin/%s\n" "$USER" "$script"
done | sudo tee /etc/sudoers.d/gpu-power > /dev/null

sudo chmod 440 /etc/sudoers.d/gpu-power
sudo visudo -c -f /etc/sudoers.d/gpu-power
```

---

## 9 — Idle Auto-Off Timer

When a session ends, a countdown starts. If no new session begins within `IDLE_TIMEOUT_MIN`, the eGPU powers off.

```bash
sudo nano /usr/local/bin/gpu-idle-start.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail
source /usr/local/lib/gpu-power-lib.sh
load_config

(( IDLE_TIMEOUT_MIN == 0 )) && exit 0
egpu_is_online || exit 0
date +%s > /run/gpu-idle-since
log "Idle timer started: ${IDLE_TIMEOUT_MIN} min until GPU off."

(
    sleep $(( IDLE_TIMEOUT_MIN * 60 ))
    if egpu_is_online; then
        active="$(loginctl list-sessions --no-legend | awk '$2 == 1000 && $4 != "-" { print $1 }' | head -n 1)"
        if [[ -z "${active}" ]]; then
            /usr/local/bin/gpu-off.sh
        fi
    fi
    rm -f /run/gpu-idle-timer.pid /run/gpu-idle-since
) &
echo "$!" > /run/gpu-idle-timer.pid
```

Make executable and grant sudo:
```bash
sudo chmod +x /usr/local/bin/gpu-idle-start.sh
printf "%s ALL=(ALL) NOPASSWD: /usr/local/bin/gpu-idle-start.sh\n" "$USER" | sudo tee -a /etc/sudoers.d/gpu-power > /dev/null
sudo visudo -c -f /etc/sudoers.d/gpu-power
```

---

## 10 — Integrate with Session Scripts

We must inject our new GPU scripts into the Phase 8 session scripts so power states change automatically when you stream.

### start-gaming.sh & start-kde.sh

Open **both** scripts (`sudo nano /usr/local/bin/start-gaming.sh` and `sudo nano /usr/local/bin/start-kde.sh`) and locate the section verifying the username and Sunshine service. Add the highlighted lines exactly as shown:

```bash hl_lines="5 6 7 8"
    [[ "${EUID}" -eq 0 ]] || die "Must be run as root (sudo)."
    USERNAME="$(id -nu "${USER_UID}")" || die "Cannot resolve username."
    SUNSHINE_SERVICE="$(detect_sunshine)" || die "Cannot find Sunshine service."

    # >>> ADD THESE THREE LINES >>>
    log "Ensuring eGPU is online before starting graphical session..."
    /usr/local/bin/gpu-on.sh || die "Failed to bring eGPU online. Check 'sudo gpu-status.sh'."
    # <<< END ADDITION <<<

    log "User: ${USERNAME} | Session: ${SESSION_NAME} | Sunshine: ${SUNSHINE_SERVICE}"

    if [[ -n "$(active_graphical_sessions)" ]]; then
```

### stop-session.sh

Open the script (`sudo nano /usr/local/bin/stop-session.sh`) and scroll to the very bottom. Add the highlighted lines immediately before the final `echo` command:

```bash hl_lines="5 6 7 8"
    log "Starting Sunshine to capture SDDM Greeter via KMS..."
    systemctl reset-failed "${SUNSHINE_SERVICE}" 2>/dev/null || true
    systemctl start "${SUNSHINE_SERVICE}"

    # >>> ADD THESE THREE LINES >>>
    log "Starting GPU idle-off countdown..."
    sudo /usr/local/bin/gpu-idle-start.sh || true
    # <<< END ADDITION <<<

    echo -e "\n✓ Session ended. Server is idle at the SDDM greeter. Reconnect Moonlight to view."
```

---

## 11 — The "Secretary-Boss" AI Architecture

By isolating Ollama into two separate Docker containers, you achieve a highly reliable setup. The iGPU (Secretary) is always active, providing instant, reliable responses. When asked a complex task, the Secretary uses a tool to power on the eGPU (Boss). 

### 1. Docker Compose Structure

Modify your `docker-compose.yml` from Phase 12 to run two separate Ollama instances. One runs purely on the 780M, the other is mapped to the eGPU.

```yaml
services:
  # The Secretary (Always On)
  ollama-igpu:
    image: ollama/ollama:rocm
    container_name: ollama-igpu
    restart: unless-stopped
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri:/dev/dri
    environment:
      - HSA_OVERRIDE_GFX_VERSION=11.0.0 # Force 780M compatibility
    # Ensure this port doesn't conflict
    ports:
      - "11434:11434" 

  # The Boss (On Demand)
  ollama-egpu:
    image: ollama/ollama:rocm
    container_name: ollama-egpu
    # Do NOT use 'restart: unless-stopped' here. Our scripts manage it.
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri:/dev/dri
    ports:
      - "11435:11434" # Exposed on port 11435

  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: openwebui
    restart: unless-stopped
    ports:
      - "3000:8080"
    environment:
      # Tell OpenWebUI about both AI engines
      - OLLAMA_BASE_URLS=http://ollama-igpu:11434;http://ollama-egpu:11434
```

*Note: Once updated, run `docker compose up -d`. You will manually stop `ollama-egpu` so the script can manage it.*

### 2. The Secretary's Python Tool

In OpenWebUI, load a fast 3B or 4B model (like Llama-3.2-3B) into the `ollama-igpu` engine. Then, create this new Tool and enable it for the Secretary model:

```python
"""
title: Enable Boss Mode (eGPU Power)
description: Powers on the high-performance eGPU so a smarter AI model can take over complex tasks.
author: LocalAdmin
version: 1.0
"""
import requests

class Tools:
    def __init__(self):
        # The URL of the webhook server from Phase 12/Section 12 below
        self.webhook_url = "http://192.168.1.xxx:8090/gpu-on"

    def activate_more_power(self) -> str:
        """
        Use this tool ONLY when the user asks a complex question that requires heavy reasoning, 
        coding, deep knowledge, or when the user explicitly asks to enable more computing power.
        """
        try:
            response = requests.get(self.webhook_url, timeout=5)
            if response.status_code == 200:
                return "Success! Reply exactly with: 'Hold on, let me enable more computing power. The eGPU is spinning up. Please wait 30 seconds and then select the Boss model from the menu at the top to continue.'"
            else:
                return "Failed to turn on the eGPU."
        except Exception as e:
            return f"Error contacting server: {e}"
```

When you speak to the Secretary and it realizes the task is complex, it will trigger this tool. Our `gpu-on.sh` script will run, restoring PSU power, rescanning PCIe, and automatically running `docker start ollama-egpu`. You then switch models in the UI and continue seamlessly.

---

## 12 — Home Assistant Integration

Extend your webhook server from Phase 12 to handle the GPU power calls natively.

Open `nano ~/webhook-server.sh` and add these routes:

```bash
"/gpu-on")
    log "Webhook: GPU on"
    sudo /usr/local/bin/gpu-on.sh >> /tmp/gpu-webhook.log 2>&1 &
    echo -e "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\nGPU powering on..."
    ;;
"/gpu-off")
    log "Webhook: GPU off"
    sudo /usr/local/bin/gpu-off.sh >> /tmp/gpu-webhook.log 2>&1 &
    echo -e "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\nGPU powering off..."
    ;;
```

In your Home Assistant `configuration.yaml`:

```yaml
rest_command:
  gpu_on:
    url: "http://<SERVER_IP>:<WEBHOOK_PORT>/gpu-on"
    method: GET
  gpu_off:
    url: "http://<SERVER_IP>:<WEBHOOK_PORT>/gpu-off"
    method: GET
```
