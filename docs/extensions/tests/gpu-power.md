# GPU Power Management (OCuLink)

Fully automated eGPU on-demand power lifecycle. Bypasses OCuLink hardware limits without rebooting. Achieves max power savings at idle and max performance for gaming.

!!! abstract "The OCuLink Hardware Curse & Our Solution"
    **The Problem:** OCuLink provides a raw PCIe connection. Unlike Thunderbolt, the GMKTec NucBox motherboard BIOS completely drops the PCIe link when external power is cut. If the PC boots without the eGPU on, the BIOS disables the PCIe port entirely. The OS cannot re-enable it while running.
    
    **The Solution:** 
    1. **Boot trick:** We force the eGPU to turn ON right before shutting down/rebooting. The BIOS sees it during POST and allocates the PCIe lanes. A startup script then immediately turns it OFF once Linux loads to save power.
    2. **Hot-plug trick:** To switch the eGPU on later, we turn on the smart plug, then force the OS into a 9-second micro-sleep. Waking up forces the BIOS to rescan the hardware, seamlessly restoring the eGPU connection without a full reboot.

!!! warning "Hardware specific"
    Built for **GMKTec NucBox K8 Plus** (mini PC) and **Minisforum DEG1** (eGPU dock) using an **AMD Radeon RX 9070 XT**.
    Requires a **Shelly Plug Gen3** on the eGPU PSU power cord for automation.

---

## 1 — BIOS & Physical Setup

### 1.1 BIOS Settings
Change **UMA Frame Buffer Size** to **4G** or **Auto**.  
*Why:* Setting it to 16GB permanently locks 16GB of system RAM exclusively for the iGPU. Lowering it frees ~12GB of RAM for the OS. Linux will dynamically share RAM with the iGPU for Docker/desktop tasks, leaving maximum system RAM available for heavy AAA gaming on the eGPU.

### 1.2 Shelly Plug Setup
1. Assign a static IP via your router (e.g., `192.168.178.140`).
2. Open the Shelly Web UI. Go to Settings and set **Power On Default** to **ON**.  
*Why:* If your house loses power, the plug will boot ON when power returns. This ensures the server boots with the eGPU powered, satisfying the BIOS requirement.

---

## 2 — OS Kernel & Udev Fixes

### 2.1 Enable GPU Tuning
By default, the Linux kernel locks AMD voltage controls. Enable the overdrive feature mask so we can undervolt natively without third-party apps like LACT.

```bash
rpm-ostree kargs --append-if-missing="amdgpu.ppfeaturemask=0xffffffff"
```

### 2.2 Pin iGPU for Docker
Dynamic nodes (`renderD128`, `renderD129`) shift whenever the eGPU connects or the system reboots. If Docker containers (Stremio, Nextcloud) map to a shifting node, they will crash. 
We create a permanent symlink (`/dev/igpu`) locked to the exact HawkPoint iGPU hardware ID (`0x1900`).

```bash
echo 'ACTION=="add|change", SUBSYSTEM=="drm", KERNEL=="renderD*", ATTRS{device}=="0x1900", SYMLINK+="igpu"' | sudo tee /etc/udev/rules.d/99-igpu.rules
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=drm
```

Update your `docker-compose.yml` files to use the permanent path:
```yaml
    devices:
      - "/dev/igpu:/dev/dri/renderD128"
```

Reboot the system to apply the kernel arg.
```bash
systemctl reboot
```

---

## 3 — Core Structure & Configuration

Create the script directory and configuration file.

```bash
sudo mkdir -p /opt/gpu-power
sudo nano /opt/gpu-power/config
```

```bash
# eGPU PCIe Address
EGPU_PCI="0000:03:00.0"

# Shelly Gen3 Configuration
PLUG_TYPE="shelly"          
PLUG_IP="192.168.178.140"
```

```bash
sudo chmod 640 /opt/gpu-power/config
```

---

## 4 — Core Library Script

This holds the shared logic: Shelly Gen3 API calls, hardware detection, CPU `tuned-adm` profiling, and direct `sysfs` GPU tuning.

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
        shelly) curl -m 3 -sf "http://${PLUG_IP}/rpc/Switch.Set?id=0&on=true" > /dev/null || true ;;
    esac
}

plug_off() {
    log "Powering OFF eGPU..."
    case "${PLUG_TYPE}" in
        shelly) curl -m 3 -sf "http://${PLUG_IP}/rpc/Switch.Set?id=0&on=false" > /dev/null || true ;;
    esac
}

plug_state() {
    case "${PLUG_TYPE}" in
        shelly) curl -m 2 -sf "http://${PLUG_IP}/rpc/Switch.GetStatus?id=0" 2>/dev/null | grep -q '"output":true' && echo "on" || echo "off" ;;
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
    log "CPU Profile → Bazzite Power Save"
    sudo tuned-adm profile powersave-bazzite >/dev/null 2>&1 || true
}

set_cpu_tdp_gaming() {
    log "CPU Profile → Performance"
    sudo tuned-adm profile throughput-performance >/dev/null 2>&1 || true
}

apply_gpu_settings() {
    local drm_dev="$(egpu_drm_path)" || return 1
    
    # Apply 290W Power Limit
    local hwmon_path="${drm_dev}/hwmon/$(ls "${drm_dev}/hwmon/" | head -n 1)"
    if [[ -n "${hwmon_path}" && -f "${hwmon_path}/power1_cap" ]]; then
        echo 290000000 | tee "${hwmon_path}/power1_cap" > /dev/null 2>&1
    fi

    # Apply -70mV Undervolt
    local od_path="${drm_dev}/pp_od_clk_voltage"
    if [[ -f "${od_path}" ]]; then
        echo "vo -70" | tee "${od_path}" > /dev/null 2>&1 || true
        echo "c" | tee "${od_path}" > /dev/null 2>&1 || true
    fi
    log "GPU tuning applied (-70mV, 290W)"
}
```

---

## 5 — Control Scripts

### 5.1 Turn ON (`gpu-on.sh`)
Restores power, waits 5s for PSU capacitors to charge, executes the 9s micro-sleep, triggers udev to remap devices, tunes the GPU, and boosts the CPU profile.

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
sleep 5 # Allow PSU and GPU firmware to boot

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

echo -e "\n✓ eGPU online. CPU at performance profile."
```

### 5.2 Turn OFF (`gpu-off.sh`)
Unbinds the AMD driver to prevent kernel panic, removes the PCIe device, cuts smart plug power, drops CPU to powersave, and enforces sleep masks.

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

if [[ -e "/sys/bus/pci/devices/${EGPU_PCI}/driver" ]]; then
    echo "${EGPU_PCI}" > /sys/bus/pci/drivers/amdgpu/unbind
    sleep 2
fi

if [[ -e "/sys/bus/pci/devices/${EGPU_PCI}" ]]; then
    echo 1 > "/sys/bus/pci/devices/${EGPU_PCI}/remove"
    sleep 2
fi

plug_off
set_cpu_tdp_idle

# Bulletproof mask enforcement
systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target >/dev/null 2>&1 || true

udevadm trigger --subsystem-match=drm
sleep 1

echo -e "\n✓ eGPU offline. CPU at idle profile."
```

---

## 6 — Systemd Boot/Shutdown Automation

*Why:* The BIOS must see the eGPU during power-on/reboot. These services automatically turn the plug ON before shutting down, and immediately turn it OFF after Linux finishes booting.

```bash
# 1. Turn ON before Reboot/Shutdown
cat << 'EOF' | sudo tee /etc/systemd/system/egpu-reboot-on.service
[Unit]
Description=Turn ON eGPU before reboot/shutdown
DefaultDependencies=no
Before=shutdown.target halt.target reboot.target NetworkManager.service

[Service]
Type=oneshot
ExecStart=/usr/bin/curl -m 3 -sf "http://192.168.178.140/rpc/Switch.Set?id=0&on=true"
TimeoutSec=5

[Install]
WantedBy=shutdown.target halt.target reboot.target
EOF

# 2. Turn OFF automatically after Boot
cat << 'EOF' | sudo tee /etc/systemd/system/egpu-boot-off.service[Unit]
Description=Turn OFF eGPU after boot
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/opt/gpu-power/gpu-off.sh --force
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

# 3. Enable the services
sudo systemctl daemon-reload
sudo systemctl enable egpu-reboot-on.service
sudo systemctl enable egpu-boot-off.service
```

---

## 7 — Permissions & System Integration

Set executable rights, symlink binaries into the environment, and configure passwordless sudo execution.

```bash
# Set permissions
sudo chmod 644 /opt/gpu-power/gpu-power-lib.sh
sudo chmod 755 /opt/gpu-power/gpu-on.sh
sudo chmod 755 /opt/gpu-power/gpu-off.sh

# Create global symlinks
sudo ln -sf /opt/gpu-power/gpu-on.sh /usr/local/bin/gpu-on.sh
sudo ln -sf /opt/gpu-power/gpu-off.sh /usr/local/bin/gpu-off.sh

# Passwordless Sudo for power scripts
sudo tee /etc/sudoers.d/gpu-power > /dev/null << 'EOF'
sotohome ALL=(ALL) NOPASSWD: /usr/local/bin/gpu-on.sh
sotohome ALL=(ALL) NOPASSWD: /usr/local/bin/gpu-off.sh
EOF
sudo chmod 440 /etc/sudoers.d/gpu-power

# Passwordless Sudo for tuned-adm
printf "sotohome ALL=(ALL) NOPASSWD: /usr/sbin/tuned-adm\n" | sudo tee /etc/sudoers.d/tuned-adm > /dev/null
sudo chmod 440 /etc/sudoers.d/tuned-adm
```

### Standard Workflow:
1. Run `gpu-on.sh` from SSH before launching heavy workloads.
2. Launch your graphical session (e.g. `start-gaming.sh`).
3. When finished, end the session (`stop-session.sh`).
4. Run `gpu-off.sh` to return the server to its low-power idle profile.
