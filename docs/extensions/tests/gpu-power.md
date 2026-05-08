```markdown
---
icon: material/gpu
---

# GPU Power Management

Fully automated eGPU on-demand power lifecycle with CPU TDP scaling, undervolt, and reliable idle auto-off.

!!! abstract "What this page adds"
    - **eGPU on/off without rebooting** via PCIe bus remove + smart plug
    - **CPU TDP switching** — 35 W idle → 65 W performance, no reboot
    - **GPU undervolt + power cap** — ~95 % performance at ~85 % power via LACT
    - **Automatic idle shutdown** — configurable inactivity timeout
    - **Manual control** — simple `gpu-on` / `gpu-off` commands

!!! warning "Hardware required"
    A **smart plug** on the be quiet! 750 W PSU power cord is mandatory.  
    Supports **Shelly Plug S** and **TP-Link Kasa**.

---

## How it works

| Phase | What happens |
|---|---|
| **GPU off** | Driver unbound → PCIe remove → smart plug cuts power |
| **GPU on**  | Smart plug restores power → PCIe rescan → driver bind |
| **Idle off** | Session ends → countdown → auto power off |

---

## Prerequisites

- Phase 8 — Session Scripts complete
- Smart plug installed on GPU PSU with static IP
- eGPU present for initial LACT setup

---

## 1 — Kernel Parameters

```bash
rpm-ostree kargs \
  --append-if-missing="amdgpu.ppfeaturemask=0xffffffff" \
  --append-if-missing="iomem=relaxed"
```

Reboot after verification.

---

## 2 — Install LACT

```bash
flatpak install flathub io.github.ilya_zlobintsev.LACT
sudo systemctl enable --now lact
```

---

## 3 — LACT Configuration (eGPU online)

1. Power limit: **290 W**
2. Voltage offset: **−70 mV**
3. Fan: Zero RPM below 50 °C
4. Apply & Save

Verify:
```bash
sudo systemctl restart lact
cat /sys/class/drm/card*/device/hwmon/hwmon*/power1_cap 2>/dev/null
```

---

## 4 — Disable LEDs

Use the physical LED button on the ASRock Taichi OC (recommended).

---

## 5 — ryzenadj

```bash
rpm-ostree install ryzenadj
systemctl reboot
```

Passwordless sudo:
```bash
RYADJ_PATH="$(which ryzenadj)"
printf "%s ALL=(ALL) NOPASSWD: %s\n" "$USER" "$RYADJ_PATH" | sudo tee /etc/sudoers.d/ryzenadj > /dev/null
sudo chmod 440 /etc/sudoers.d/ryzenadj
```

---

## 6 — Config File

```bash
lspci -nn | grep -E "1002:(15bf|7480)"
sudo mkdir -p /etc/gpu-power
sudo nano /etc/gpu-power/config
```

```bash
EGPU_PCI="0000:01:00.0"
PLUG_TYPE="shelly"          # or "kasa"
PLUG_IP="192.168.1.50"
IDLE_TIMEOUT_MIN=30

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

## 7 — Core Scripts

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
    die()  { echo "[gpu-power] ERROR: $*" >&2; exit 1; }

    plug_on() {
        log "Smart plug ON"
        case "${PLUG_TYPE}" in
            shelly) curl -sf "http://${PLUG_IP}/relay/0?turn=on" > /dev/null ;;
            kasa)   kasa --host "${PLUG_IP}" on > /dev/null ;;
        esac
    }

    plug_off() {
        log "Smart plug OFF"
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
            [[ "$(readlink -f "${drm_dir}")" == *"${sysfs_addr}"* ]] && echo "${drm_dir}" && return 0
        done
        return 1
    }

    egpu_is_online() {
        [[ -e "/sys/bus/pci/devices/${EGPU_PCI}" ]]
    }

    set_cpu_tdp_idle() {
        log "CPU TDP → Idle"
        sudo ryzenadj --stapm-limit="${CPU_IDLE_STAPM}" --slow-limit="${CPU_IDLE_SLOW}" --fast-limit="${CPU_IDLE_FAST}" >/dev/null 2>&1 || true
    }

    set_cpu_tdp_gaming() {
        log "CPU TDP → Performance"
        sudo ryzenadj --stapm-limit="${CPU_GAMING_STAPM}" --slow-limit="${CPU_GAMING_SLOW}" --fast-limit="${CPU_GAMING_FAST}" >/dev/null 2>&1 || true
    }

    apply_gpu_settings() {
        local drm_dev="$(egpu_drm_path)" || return 1
        local hwmon_path="${drm_dev}/hwmon/$(ls "${drm_dev}/hwmon/" | head -n 1)"
        echo 290000000 | tee "${hwmon_path}/power1_cap" > /dev/null 2>&1

        local od_path="${drm_dev}/pp_od_clk_voltage"
        if [[ -f "${od_path}" ]]; then
            echo "vo -70" | tee "${od_path}" > /dev/null
            echo "c" | tee "${od_path}" > /dev/null
        fi
        log "GPU tuning applied"
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

    if egpu_is_online; then
        apply_gpu_settings
        set_cpu_tdp_gaming
        echo "✓ eGPU already online"
        exit 0
    fi

    plug_on
    sleep 3
    echo 1 > /sys/bus/pci/rescan

    for ((i=0; i<20; i++)); do
        egpu_is_online && break
        sleep 1
    done

    egpu_is_online || { echo "ERROR: eGPU did not appear"; exit 1; }

    apply_gpu_settings
    set_cpu_tdp_gaming
    rm -f /run/gpu-idle-timer.pid

    echo -e "\n✓ eGPU online. CPU at performance TDP."
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

    if ! egpu_is_online; then
        plug_off
        set_cpu_tdp_idle
        exit 0
    fi

    if [[ "${1}" != "--force" ]]; then
        active="$(loginctl list-sessions --no-legend | awk '$2 == 1000 && $4 != "-" {print $1}' | head -n1)"
        [[ -n "$active" ]] && { echo "ERROR: Graphical session active. Use --force to override."; exit 1; }
    fi

    systemctl stop lact 2>/dev/null || true

    if [[ -e "/sys/bus/pci/devices/${EGPU_PCI}/driver" ]]; then
        echo "${EGPU_PCI}" > /sys/bus/pci/drivers/amdgpu/unbind
        sleep 1
    fi

    if [[ -e "/sys/bus/pci/devices/${EGPU_PCI}" ]]; then
        echo 1 > "/sys/bus/pci/devices/${EGPU_PCI}/remove"
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
    source /usr/local/lib/gpu-power-lib.sh
    load_config

    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━ GPU Status ━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "eGPU:      $(egpu_is_online && echo "ONLINE" || echo "OFFLINE")"
    echo "PSU:       $(plug_state | tr '[:lower:]' '[:upper:]')"
    stapm="$(sudo ryzenadj --info 2>/dev/null | awk '/STAPM LIMIT/ {print $4}' | head -n1 || echo unknown)"
    echo "CPU STAPM: ${stapm} W"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    ```
    ```bash
    sudo chmod +x /usr/local/bin/gpu-status.sh
    ```

---

## 8 — Sudo Rights

```bash
for s in gpu-on.sh gpu-off.sh gpu-status.sh gpu-idle-start.sh; do
    printf "%s ALL=(ALL) NOPASSWD: /usr/local/bin/%s\n" "$USER" "$s"
done | sudo tee /etc/sudoers.d/gpu-power > /dev/null
sudo chmod 440 /etc/sudoers.d/gpu-power
```

---

## 9 — Idle Timer

```bash
sudo nano /usr/local/bin/gpu-idle-start.sh
```

```bash
#!/usr/bin/env bash
source /usr/local/lib/gpu-power-lib.sh
load_config

(( IDLE_TIMEOUT_MIN == 0 )) && exit 0
egpu_is_online || exit 0

(
    sleep $((IDLE_TIMEOUT_MIN * 60))
    if egpu_is_online; then
        active="$(loginctl list-sessions --no-legend | awk '$2 == 1000 && $4 != "-" {print $1}' | head -n1)"
        [[ -z "$active" ]] && /usr/local/bin/gpu-off.sh
    fi
) &
```

```bash
sudo chmod +x /usr/local/bin/gpu-idle-start.sh
```

---

## 10 — Session Integration

**In `start-gaming.sh` & `start-kde.sh`** (after user check):
```bash
/usr/local/bin/gpu-on.sh || echo "Warning: Could not start eGPU"
```

**In `stop-session.sh`** (before final echo):
```bash
sudo /usr/local/bin/gpu-idle-start.sh || true
```

**Manual usage:**
- `sudo gpu-on.sh`
- `sudo gpu-off.sh`
- `sudo gpu-off.sh --force`
- `gpu-status.sh`
```
