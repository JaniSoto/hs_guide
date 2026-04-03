# Phases 0 - 3: OS Setup & Packages

## Phase 0: OS Installation

*Done physically at the machine.*

!!! concept "Why Bazzite-Deck specifically?"
    Bazzite has two variants: standard desktop, and `bazzite-deck` — the HTPC/living-room variant. The `-deck` suffix means it ships both `gamescope-session` (Steam Deck-style Gaming Mode) and KDE Plasma in a single signed image. Starting with `bazzite-deck` eliminates a separate rebase step and ensures Gaming Mode is part of the verified OS image.

**1. Download the correct ISO**  
Go to **bazzite.gg** and use the Image Picker tool. Select: Hardware = Desktop/HTPC, GPU = AMD/Intel or NVIDIA, Desktop = KDE Plasma, and enable Steam Gaming Mode. Download the ISO and flash it to a USB drive using Fedora Media Writer or Balena Etcher.

**2. Install Bazzite**  
Boot from the USB drive and follow the Anaconda installer. Set your username, password, hostname (under 20 characters, no spaces), timezone, and keyboard layout. Install to your OS drive. Network is not required during installation.

!!! warning "Dual Boot Warning"
    If you have a Windows drive in the same machine, physically disconnect it before installing Bazzite to prevent accidental data loss or bootloader modification.

**3. Note your GPU type**  
AMD and Intel GPUs are fully supported in Gaming Mode and for VA-API hardware acceleration. NVIDIA support in Gaming Mode is available but described as beta. NVIDIA GPUs are not compatible with VA-API hardware encoding for Sunshine.

---

## Phase 1: First Boot, SSH & Headless Config

!!! failure "Critical Sequence"
    Do ALL steps in this phase in sequence before rebooting. The headless SDDM override in Step 4 must be in place before your next reboot. If you reboot before completing Step 4, the machine will boot to Gaming Mode again — simply repeat this phase.

**1. Enter KDE Desktop Mode**  
Navigate to the Power menu in Steam (bottom-right power icon) and select **Switch to Desktop**.

**2. Open Konsole and enable SSH**
```bash
# Enable and start the SSH daemon.
sudo systemctl enable --now sshd
# Find your active network interface and local IP address.
ip route get 1.1.1.1
```
*(Write down the IP address — you will use it for every SSH connection in this guide).*

**3. Disable auto-suspend**  
Bazzite may suspend the machine after inactivity, silently killing all Docker containers. Permanently disable all suspend modes using two methods:

* **Step A:** KDE System Settings → Power Management → Energy Saving → On AC Power → set "Suspend session" to **Never**. Click Apply.
* **Step B (Authoritative):** Mask suspend targets at the OS level:
```bash
sudo systemctl mask sleep.target suspend.target \
  hibernate.target hybrid-sleep.target suspend-then-hibernate.target
```

**4. Apply the headless SDDM override**  
We want SDDM to start but show an idle blank login screen, with no automatic login into any session.
```bash
sudo mkdir -p /etc/sddm.conf.d
sudo tee /etc/sddm.conf.d/zzz-headless-server.conf > /dev/null << 'EOF'
[Autologin]
User=
Session=
Relogin=false
EOF
sudo systemctl restart sddm
```

**5. Verify the headless config survives a reboot**
```bash
sudo systemctl reboot
# Wait ~60 seconds. Physical display should show SDDM login screen — NOT Gaming Mode.
ssh username@192.168.1.50
loginctl list-sessions
# Expected: empty output, or only your SSH session with no seat0 column entry.
```

---

## Phase 2: Router & Network Preparation

**1. Assign a static local IP (DHCP reservation)**  
In your router's admin panel, find the DHCP section, add a reservation for your server's MAC address, and assign the IP you noted in Phase 1.

**2. Set up port forwarding**  
Your router has one public IP address and shares it among all devices via NAT. Open only the specific ports that need to be publicly reachable.

| Port / Protocol | Purpose |
|-----------------|---------|
| **80/TCP**      | Required by Let's Encrypt for SSL certificate issuance (HTTP-01 challenge). |
| **443/TCP**     | Your Nextcloud's public HTTPS entry point. All user traffic uses this port. |
| **22022/TCP**   | SSH — our custom hardened port (configured in Phase 4). |
| **51820/UDP**   | WireGuard VPN — Opens the secure tunnel for Sunshine game streaming. |

!!! danger "Never forward Sunshine ports"
    NEVER forward Sunshine ports (47984, 47989, 48010, etc.) at the router. These are LAN only. Sunshine is only reachable remotely via the WireGuard VPN tunnel.

---

## Phase 3: System Package Installation

**1. Add the Docker repository**  
Bazzite comes with Podman pre-installed, but Nextcloud AIO requires native Docker CE.
```bash
sudo curl -fsSL https://download.docker.com/linux/fedora/docker-ce.repo \
  -o /etc/yum.repos.d/docker-ce.repo
```

**2. Install all required packages**  
Bazzite is an immutable OS. We layer packages using `rpm-ostree`. We also safely remove the `podman-docker` shim if it exists to prevent conflicts.
```bash
if rpm -q podman-docker >/dev/null 2>&1; then
  rpm-ostree override remove podman-docker \
    --install docker-ce \
    --install docker-ce-cli \
    --install containerd.io \
    --install docker-buildx-plugin \
    --install docker-compose-plugin \
    --install fail2ban \
    --install fail2ban-firewalld \
    --install policycoreutils-python-utils
else
  rpm-ostree install \
    docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin \
    fail2ban fail2ban-firewalld \
    policycoreutils-python-utils
fi
systemctl reboot
```
*(Your SSH session will disconnect. Wait ~60 seconds, then reconnect).*
