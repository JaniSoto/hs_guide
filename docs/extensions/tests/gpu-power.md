# GPU Power Management (OCuLink)

Fully automated eGPU on-demand power lifecycle. Achieves max power savings at idle and max performance for gaming using a stable reboot-based transition.

!!! abstract "The OCuLink Hardware Curse & Our Solution"
    **The Problem:** OCuLink provides a raw PCIe connection. Unlike Thunderbolt, the GMKTec NucBox motherboard BIOS completely drops the PCIe link when external power is cut. If the PC boots without the eGPU on, the BIOS disables the PCIe port entirely. The OS cannot re-enable it while running.
    
    **The Solution:** We use a reboot-based lifecycle to ensure motherboard alignment:
    * To turn the eGPU **ON**, we trigger the smart plug and reboot. The BIOS detects the card during POST, maps the PCIe lanes, and boots cleanly.
    * To turn the eGPU **OFF**, we gracefully unbind the driver and remove the device from the PCIe bus to prevent kernel panics, cut the smart plug power, and reboot to return the system to its low-power idle state.

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
By default, the Linux kernel locks AMD voltage controls. Enable the overdrive feature mask so we can undervolt natively.

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
# Shelly Gen3 Configuration
PLUG_TYPE="shelly"          
PLUG_IP="192.168.178.140"
```

```bash
sudo chmod 640 /opt/gpu-power/config
```

---

## 4 — Core Library Script

This holds the shared logic: Shelly Gen3 API calls, CPU `tuned-adm` profiling, direct `sysfs` GPU tuning, and dynamic PCIe slot scanning.

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

egpu_pci() {
    for dev in /sys/bus/pci/devices/*; do
        if [[ -f "$dev/vendor" && "$(cat "$dev/vendor")" == "0x1002" ]]; then
            if [[ -f "$dev/class" && "$(cat "$dev/class")" == 0x0300* ]]; then
                if [[ "$(cat "$dev/device")" != "0x1900" ]]; then
                    basename "$dev"
                    return 0
                fi
            fi
        fi
    done
    return 1
}

egpu_is_online() {
    egpu_pci >/dev/null
}

egpu_drm_path() {
    local pci
    pci="$(egpu_pci)" || return 1
    for drm_dir in /sys/class/drm/card*/device; do
        if [[ "$(readlink -f "${drm_dir}")" == *"${pci}"* ]]; then
            echo "${drm_dir}"
            return 0
        fi
    done
    return 1
}

set_cpu_tdp_idle() {
    log "CPU Profile → Bazzite Powersave"
    sudo tuned-adm profile powersave-bazzite >/dev/null 2>&1 || true
}

set_cpu_tdp_gaming() {
    log "CPU Profile → Bazzite Performance"
    sudo tuned-adm profile throughput-performance-bazzite >/dev/null 2>&1 || true
}

apply_gpu_settings() {
    local drm_dev
    drm_dev="$(egpu_drm_path)" || return 1
    
    local hwmon_dir
    hwmon_dir="$(ls "${drm_dev}/hwmon/" 2>/dev/null | head -n 1)" || true
    if [[ -n "${hwmon_dir}" && -f "${drm_dev}/hwmon/${hwmon_dir}/power1_cap" ]]; then
        echo 290000000 | tee "${drm_dev}/hwmon/${hwmon_dir}/power1_cap" > /dev/null 2>&1
    fi

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

These are the user-facing binaries to safely toggle the physical state of the eGPU.

### 5.1 Turn ON (`gpu-on.sh`)
Powers on the smart plug to boot the eGPU and triggers an immediate reboot so the motherboard BIOS discovers and maps the card.

```bash
sudo nano /opt/gpu-power/gpu-on.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail
[[ "${EUID}" -eq 0 ]] || exec sudo "$0" "$@"

source /opt/gpu-power/gpu-power-lib.sh
load_config

log "Powering ON eGPU and instantly rebooting..."
plug_on

systemctl reboot
```

### 5.2 Turn OFF (`gpu-off.sh`)
Safely unbinds the AMD driver, removes the card from the PCIe bus, powers off the smart plug, and triggers a clean reboot to lock out the PCIe lane and return the server to its idle state.

```bash
sudo nano /opt/gpu-power/gpu-off.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail
[[ "${EUID}" -eq 0 ]] || exec sudo "$0" "$@"

source /opt/gpu-power/gpu-power-lib.sh
load_config

local_pci="$(egpu_pci || true)"

# Gracefully detach the eGPU before cutting power to avoid kernel panics
if [[ -n "${local_pci}" ]]; then
    log "Gracefully unbinding and removing eGPU from PCIe bus..."
    if [[ -e "/sys/bus/pci/devices/${local_pci}/driver" ]]; then
        echo "${local_pci}" > "/sys/bus/pci/drivers/amdgpu/unbind" || true
        sleep 1
    fi
    if [[ -e "/sys/bus/pci/devices/${local_pci}" ]]; then
        echo 1 > "/sys/bus/pci/devices/${local_pci}/remove" || true
        sleep 1
    fi
fi

log "Powering OFF eGPU..."
plug_off

log "Rebooting system to apply changes..."
systemctl reboot
```

---

## 6 — Systemd Boot Automation

We use a one-shot system service to run a script at every boot. It detects if the eGPU is online, configures your CPU and GPU profiles, and automatically maps Sunshine's KMS display capture to the correct physical GPU before SDDM loads.

```bash
# 1. Create the boot service
cat << 'EOF' | sudo tee /etc/systemd/system/gpu-power-init.service
[Unit]
Description=Auto-configure GPU and CPU profiles based on eGPU presence
After=local-fs.target
Before=sddm.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/opt/gpu-power/gpu-init-boot.sh

[Install]
WantedBy=multi-user.target
EOF
```

```bash
# 2. Create the boot-initialization script
sudo nano /opt/gpu-power/gpu-init-boot.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail

source /opt/gpu-power/gpu-power-lib.sh
load_config

# Helper to dynamically update Sunshine's config to target a specific GPU path
update_sunshine_config() {
    local target_pci="$1"
    local conf_file="/home/sotohome/.config/sunshine/sunshine.conf"
    local target_path="/dev/dri/by-path/pci-${target_pci}-render"

    if [[ -f "${conf_file}" ]]; then
        log "Targeting Sunshine to stable GPU path: ${target_path}"
        if grep -q "^adapter_name" "${conf_file}"; then
            sed -i "s|^adapter_name = .*|adapter_name = ${target_path}|" "${conf_file}"
        else
            echo "adapter_name = ${target_path}" >> "${conf_file}"
        fi
        # Ensure file ownership remains correct for the user session
        chown sotohome:sotohome "${conf_file}"
    else
        log "WARNING: Sunshine configuration file not found at ${conf_file}"
    fi
}

# Check if the eGPU physically exists on the PCIe bus at boot
if egpu_is_online; then
    log "eGPU detected! Configuring Performance/Gaming profiles..."
    set_cpu_tdp_gaming
    apply_gpu_settings

    # Resolve eGPU slot dynamically and point Sunshine to it
    EGPU_ADDR="$(egpu_pci)"
    update_sunshine_config "${EGPU_ADDR}"
else
    log "eGPU not detected. Configuring Powersave/iGPU profiles..."
    set_cpu_tdp_idle

    # Resolve iGPU slot dynamically (AMD device ID 0x1900) and point Sunshine to it
    IGPU_ADDR=""
    for dev in /sys/bus/pci/devices/*; do
        if [[ -f "${dev}/device" && "$(cat "${dev}/device")" == "0x1900" ]]; then
            IGPU_ADDR="$(basename "${dev}")"
            break
        fi
    done
    # Fallback to hardcoded address if dynamic discovery fails
    [[ -n "${IGPU_ADDR}" ]] || IGPU_ADDR="0000:c9:00.0"
    
    update_sunshine_config "${IGPU_ADDR}"
fi
```

```bash
# 3. Enable the boot service
sudo systemctl daemon-reload
sudo systemctl enable gpu-power-init.service
```

---

## 7 — Permissions & System Integration

Set executable rights, symlink binaries into the environment, and configure passwordless sudo execution.

```bash
# Set permissions
sudo chmod 644 /opt/gpu-power/gpu-power-lib.sh
sudo chmod 755 /opt/gpu-power/gpu-on.sh
sudo chmod 755 /opt/gpu-power/gpu-off.sh
sudo chmod 755 /opt/gpu-power/gpu-init-boot.sh

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
1. Run `gpu-on.sh` from SSH (or a webhook). The system will boot ON with the eGPU active.
2. Launch your graphical session (e.g. `start-gaming.sh`). Sunshine automatically targets the eGPU.
3. When finished, end the session (`stop-session.sh`).
4. Run `gpu-off.sh`. The system will cleanly unbind, turn the plug off, and reboot into its low-power idle profile.
