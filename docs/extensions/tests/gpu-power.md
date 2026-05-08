# GPU Power Management (OCuLink)

Fully automated eGPU on-demand power lifecycle. Bypass OCuLink hardware limits without reboot.

!!! abstract "How approach works"
    OCuLink use raw PCIe. Motherboard BIOS drop link when power cut. BIOS refuse renegotiate while OS run. 
    
    Bypass use micro-sleep cycle (9s).
    - **GPU off:** OS remove PCIe device → smart plug cut power.
    - **GPU on:** Smart plug restore power → OS suspend → BIOS hardware init → OS wake → eGPU ready.

!!! warning "Hardware required"
    Smart plug on eGPU PSU cord mandatory. Support **Shelly Plug S**, **TP-Link Kasa**, or **manual** switch.  
    Setup built for **GMKTec NucBox K8 Plus** (mini PC) and **Minisforum DEG1** (eGPU dock).

---

## 1 — BIOS Configuration

Set **UMA Frame Buffer Size** to **4G** or **Auto** in BIOS.

*Reason:* 16GB UMA lock memory from system pool. 4G free ~12GB RAM for OS. Linux auto-share RAM to iGPU if needed. AAA games on eGPU get max system RAM.

---

## 2 — Pin iGPU for Docker Containers

Dynamic nodes (`renderD128`, `renderD129`) shift on reboot/eGPU switch. Break Docker containers (Stremio, Nextcloud).

Create permanent symlink (`/dev/igpu`) pinned to exact HawkPoint iGPU hardware ID (`0x1900`). Survive reboots forever.

```bash
echo 'ACTION=="add|change", SUBSYSTEM=="drm", KERNEL=="renderD*", ATTRS{device}=="0x1900", SYMLINK+="igpu"' | sudo tee /etc/udev/rules.d/99-igpu.rules
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=drm
```

Update `docker-compose.yml` files:
```yaml
    devices:
      - "/dev/igpu:/dev/dri/renderD128"
```

---

## 3 — Install Dependencies

Install LACT (GPU tune) and Ryzenadj (CPU TDP scale).

```bash
flatpak install flathub io.github.ilya_zlobintsev.LACT
sudo systemctl enable --now lact

rpm-ostree install ryzenadj
systemctl reboot
```

---

## 4 — Core Structure & Configuration

Create directory and config file.

```bash
sudo mkdir -p /opt/gpu-power
sudo nano /opt/gpu-power/config
```

```bash
EGPU_PCI="0000:03:00.0"
PLUG_TYPE="manual"          
PLUG_IP="192.168.1.50"

CPU_IDLE_STAPM=35000
CPU_IDLE_SLOW=38000
CPU_IDLE_FAST=42000

CPU_GAMING_STAPM=65000
CPU_GAMING_SLOW=68000
CPU_GAMING_FAST=75000
```

```bash
sudo chmod 640 /opt/gpu-power/config
```

---

## 5 — Core Library Script

Shared logic. Hardware state, plug control, GPU tuning.

```bash
sudo nano /opt/gpu-power/gpu-power-lib.sh
```

```bash
#!/usr/bin/env bash
GPU_CONFIG="/opt/gpu-power/config"

load_config() {
    [[ -f "${GPU_CONFIG}" ]] || { echo "[gpu-lib] ERROR: config not found"; exit 1; }
    source "${GPU_CONFIG}"
}

log()  { echo "[gpu-power] $*"; }
die()  { echo "[gpu-power] ERROR: $*" >&2; exit 1; }

plug_on() {
    log "Powering ON eGPU..."
    case "${PLUG_TYPE}" in
        shelly) curl -m 3 -sf "http://${PLUG_IP}/relay/0?turn=on" > /dev/null || true ;;
        kasa)   kasa --host "${PLUG_IP}" on > /dev/null 2>&1 || true ;;
        manual) log ">>> TURN ON eGPU NOW (Waiting 5s) <<<"; sleep 5 ;;
    esac
}

plug_off() {
    log "Powering OFF eGPU..."
    case "${PLUG_TYPE}" in
        shelly) curl -m 3 -sf "http://${PLUG_IP}/relay/0?turn=off" > /dev/null || true ;;
        kasa)   kasa --host "${PLUG_IP}" off > /dev/null 2>&1 || true ;;
        manual) log ">>> TURN OFF eGPU NOW (Waiting 5s) <<<"; sleep 5 ;;
    esac
}

plug_state() {
    case "${PLUG_TYPE}" in
        shelly) curl -m 2 -sf "http://${PLUG_IP}/relay/0" 2>/dev/null | grep -q '"ison":true' && echo "on" || echo "off" ;;
        kasa)   kasa --host "${PLUG_IP}" state 2>/dev/null | grep -qi "is on" && echo "on" || echo "off" ;;
        manual) echo "unknown" ;;
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
    if [[ -n "${hwmon_path}" && -f "${hwmon_path}/power1_cap" ]]; then
        echo 290000000 | tee "${hwmon_path}/power1_cap" > /dev/null 2>&1
    fi

    local od_path="${drm_dev}/pp_od_clk_voltage"
    if [[ -f "${od_path}" ]]; then
        echo "vo -70" | tee "${od_path}" > /dev/null 2>&1 || true
        echo "c" | tee "${od_path}" > /dev/null 2>&1 || true
    fi
    log "GPU tuning applied"
}
```

---

## 6 — Control Scripts

### Turn ON (`gpu-on.sh`)
Restore power, force 9s micro-sleep for PCIe sync, trigger udev, apply GPU tune, boost CPU TDP.

```bash
sudo nano /opt/gpu-power/gpu-on.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail
[[ "${EUID}" -eq 0 ]] || exec sudo "$0" "$@"

source /opt/gpu-power/gpu-power-lib.sh
load_config

if egpu_is_online; then
    apply_gpu_settings
    set_cpu_tdp_gaming
    echo "✓ eGPU already online"
    exit 0
fi

plug_on

log "Suspending system for 9s to force OCuLink PCIe training..."
systemctl unmask sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
rtcwake -m mem -s 9 || true
systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target

sleep 2
echo 1 > /sys/bus/pci/rescan 2>/dev/null || true
sleep 2

udevadm trigger --subsystem-match=drm
sleep 1

egpu_is_online || { echo "ERROR: eGPU did not appear after wake"; exit 1; }

apply_gpu_settings
set_cpu_tdp_gaming
rm -f /run/gpu-idle-timer.pid

echo -e "\n✓ eGPU online. CPU at performance TDP."
```

### Turn OFF (`gpu-off.sh`)
Unbind AMD driver, remove PCIe device, cut smart plug power, drop CPU TDP, enforce sleep mask.

```bash
sudo nano /opt/gpu-power/gpu-off.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail
[[ "${EUID}" -eq 0 ]] || exec sudo "$0" "$@"

source /opt/gpu-power/gpu-power-lib.sh
load_config

if ! egpu_is_online; then
    plug_off
    set_cpu_tdp_idle
    systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target >/dev/null 2>&1 || true
    exit 0
fi

if [[ "${1:-}" != "--force" ]]; then
    active="$(loginctl list-sessions --no-legend | awk '$2 == 1000 && $4 != "-" {print $1}' | head -n1)"
    [[ -n "$active" ]] && { echo "ERROR: Graphical session active. Use --force to override."; exit 1; }
fi

systemctl stop lact 2>/dev/null || true

if [[ -e "/sys/bus/pci/devices/${EGPU_PCI}/driver" ]]; then
    echo "${EGPU_PCI}" > /sys/bus/pci/drivers/amdgpu/unbind
    sleep 2
fi

if [[ -e "/sys/bus/pci/devices/${EGPU_PCI}" ]]; then
    echo 1 > "/sys/bus/pci/devices/${EGPU_PCI}/remove"
    sleep 2
fi

plug_off
systemctl start lact 2>/dev/null || true
set_cpu_tdp_idle
rm -f /run/gpu-idle-timer.pid /run/gpu-idle-since

systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target >/dev/null 2>&1 || true

udevadm trigger --subsystem-match=drm
sleep 1

echo -e "\n✓ eGPU offline. CPU at idle TDP."
```

---

## 7 — Permissions & System Integration

Set executable rights. Symlink binaries. Config passwordless sudo.

```bash
sudo chmod 644 /opt/gpu-power/gpu-power-lib.sh
sudo chmod 755 /opt/gpu-power/gpu-on.sh
sudo chmod 755 /opt/gpu-power/gpu-off.sh

sudo ln -sf /opt/gpu-power/gpu-on.sh /usr/local/bin/gpu-on.sh
sudo ln -sf /opt/gpu-power/gpu-off.sh /usr/local/bin/gpu-off.sh

sudo tee /etc/sudoers.d/gpu-power > /dev/null << 'EOF'
sotohome ALL=(ALL) NOPASSWD: /usr/local/bin/gpu-on.sh
sotohome ALL=(ALL) NOPASSWD: /usr/local/bin/gpu-off.sh
EOF
sudo chmod 440 /etc/sudoers.d/gpu-power

RYADJ_PATH="$(which ryzenadj)"
printf "sotohome ALL=(ALL) NOPASSWD: %s\n" "$RYADJ_PATH" | sudo tee /etc/sudoers.d/ryzenadj > /dev/null
sudo chmod 440 /etc/sudoers.d/ryzenadj
```

**Workflow:**
1. Run `gpu-on.sh` before heavy workload.
2. Launch graphical session (`start-gaming.sh`).
3. End session (`stop-session.sh`).
4. Run `gpu-off.sh` return low-power idle.
