


```markdown
---
icon: material/gpu
---

# GPU Power Management (OCuLink)

Fully automated eGPU on-demand power lifecycle designed to bypass OCuLink hardware limitations without requiring system reboots.

!!! abstract "How this approach works"
    OCuLink provides a raw PCIe connection. Unlike Thunderbolt, the motherboard BIOS completely drops the PCIe link when external power is cut and refuses to renegotiate the link while the OS is running. 
    
    To bypass this without a full reboot, we use a **micro-sleep cycle (9 seconds)**. 
    - **GPU off:** Standard PCIe device removal via sysfs, followed by a physical smart plug power cut.
    - **GPU on:** Smart plug restores power, then the OS suspends to memory. Upon waking, the BIOS is forced to perform a hardware-level PCIe initialization, seamlessly restoring the eGPU connection.

!!! warning "Hardware required"
    A **smart plug** on the eGPU PSU power cord is mandatory for full automation. Supports **Shelly Plug S**, **TP-Link Kasa**, or **manual** switching.

---

## 1 — BIOS Configuration

Change the **UMA Frame Buffer Size** (iGPU dedicated RAM) to **4G** or **Auto** in the BIOS. 

*Reason:* Reserving 16GB exclusively for the iGPU permanently removes that memory from the system pool. By reducing it, the OS reclaims ~12GB of RAM. The system will dynamically allocate system RAM to the iGPU if needed, while leaving plenty of memory for the external RX 9070 XT during heavy AAA gaming.

---

## 2 — Pin iGPU for Docker Containers

If you use Docker containers that require hardware acceleration (like Stremio or Nextcloud), dynamically assigned render nodes (like `/dev/dri/renderD128`) will shift when the eGPU is connected or disconnected, crashing your containers.

Create a permanent symlink (`/dev/igpu`) bound exactly to the Minisforum iGPU's PCIe address.

```bash
echo 'SUBSYSTEM=="drm", KERNEL=="renderD*", KERNELS=="0000:c9:00.0", SYMLINK+="igpu"' | sudo tee /etc/udev/rules.d/99-igpu.rules
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=drm --action=add
```

Update your `docker-compose.yml` files to use the new permanent symlink:
```yaml
    devices:
      - "/dev/igpu:/dev/dri/renderD128"
```

---

## 3 — Install Dependencies

Install LACT (for GPU undervolting/power limiting) and Ryzenadj (for CPU TDP scaling).

```bash
# Install LACT
flatpak install flathub io.github.ilya_zlobintsev.LACT
sudo systemctl enable --now lact

# Install ryzenadj
rpm-ostree install ryzenadj
systemctl reboot
```

---

## 4 — Core Structure & Configuration

Create the centralized directory and configuration file.

```bash
sudo mkdir -p /opt/gpu-power
sudo nano /opt/gpu-power/config
```

```bash
# eGPU hardware address
EGPU_PCI="0000:03:00.0"

# Smart Plug config ("shelly", "kasa", or "manual")
PLUG_TYPE="manual"          
PLUG_IP="192.168.1.50"

# CPU TDP (mW) - Drops to 35W idle, boosts to 65W for eGPU gaming
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

This script contains the shared logic for plug control, hardware state detection, and tuning.

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
    
    # 290W Power limit
    local hwmon_path="${drm_dev}/hwmon/$(ls "${drm_dev}/hwmon/" | head -n 1)"
    if [[ -n "${hwmon_path}" && -f "${hwmon_path}/power1_cap" ]]; then
        echo 290000000 | tee "${hwmon_path}/power1_cap" > /dev/null 2>&1
    fi

    # -70mV Undervolt
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
Restores power, forces the 9-second micro-sleep to negotiate the PCIe link, applies overclocks, and shifts the CPU to 65W gaming mode.

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

egpu_is_online || { echo "ERROR: eGPU did not appear after wake"; exit 1; }

apply_gpu_settings
set_cpu_tdp_gaming
rm -f /run/gpu-idle-timer.pid

echo -e "\n✓ eGPU online. CPU at performance TDP."
```

### Turn OFF (`gpu-off.sh`)
Safely unbinds the AMD driver, drops the PCIe device from the kernel, cuts physical power via the smart plug, and restricts CPU back to 35W.

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

# Bulletproof mask enforcement
systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target >/dev/null 2>&1 || true

echo -e "\n✓ eGPU offline. CPU at idle TDP."
```

---

## 7 — Permissions & System Integration

Set executable permissions, symlink into the user binary path, and configure passwordless sudo execution.

```bash
sudo chmod 644 /opt/gpu-power/gpu-power-lib.sh
sudo chmod 755 /opt/gpu-power/gpu-on.sh
sudo chmod 755 /opt/gpu-power/gpu-off.sh

sudo ln -sf /opt/gpu-power/gpu-on.sh /usr/local/bin/gpu-on.sh
sudo ln -sf /opt/gpu-power/gpu-off.sh /usr/local/bin/gpu-off.sh

# Passwordless Sudo for commands
sudo tee /etc/sudoers.d/gpu-power > /dev/null << 'EOF'
sotohome ALL=(ALL) NOPASSWD: /usr/local/bin/gpu-on.sh
sotohome ALL=(ALL) NOPASSWD: /usr/local/bin/gpu-off.sh
EOF
sudo chmod 440 /etc/sudoers.d/gpu-power

# Passwordless Sudo for ryzenadj
RYADJ_PATH="$(which ryzenadj)"
printf "sotohome ALL=(ALL) NOPASSWD: %s\n" "$RYADJ_PATH" | sudo tee /etc/sudoers.d/ryzenadj > /dev/null
sudo chmod 440 /etc/sudoers.d/ryzenadj
```

**Workflow:**
1. Run `gpu-on.sh` from SSH/Terminal before launching heavy workloads or Gamescope.
2. Launch your desired graphical session (`start-gaming.sh` or `start-kde.sh`).
3. When finished, end the session (`stop-session.sh`).
4. Run `gpu-off.sh` to return to low-power idle mode.
```
