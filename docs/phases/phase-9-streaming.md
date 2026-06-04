<div class="phase-header">
  <span class="phase-num">9</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 9 · Run over SSH</p>
    <h1>WireGuard VPN & Sunshine Streaming</h1>
    <p class="phase-header-sub">Deploy the WireGuard VPN, configure Sunshine, and connect Moonlight.</p>
  </div>
</div>

??? info "Why WireGuard + Sunshine instead of RDP?"
    RDP uses TCP, which suffers from "TCP Meltdown" — when packets are lost, TCP resends them, causing stuttering and freezing that makes gaming unplayable. Sunshine uses UDP with hardware-accelerated video encoding, delivering 4K/60fps with near-zero latency. WireGuard (also UDP) provides the secure tunnel: unauthenticated packets are silently dropped, so port 51820 is invisible to port scanners.

---

## Step 1 — Host OS Preparation

**1a — Load the legacy NAT kernel modules** needed for WireGuard's container networking:

```bash
sudo modprobe iptable_nat iptable_mangle
printf "iptable_nat\niptable_mangle\n" | sudo tee /etc/modules-load.d/iptables-legacy.conf
```

**1b — Grant hardware permissions and set up Sunshine:**

```bash
# Add your user to the input, video, and render groups
grep -q "^input:" /etc/group || sudo sh -c "getent group input >> /etc/group"
grep -q "^video:" /etc/group || sudo sh -c "getent group video >> /etc/group"
grep -q "^render:" /etc/group || sudo sh -c "getent group render >> /etc/group"
sudo usermod -aG input,video,render $(whoami)

# Run Bazzite's built-in Sunshine setup (sets cap_sys_admin capability and udev rules)
sudo ujust setup-sunshine
```

!!! info "What `ujust setup-sunshine` does"
    This sets the `cap_sys_admin` Linux capability on the Sunshine binary and configures udev rules. This is what allows Sunshine to use **KMS (Kernel Mode Setting)** to directly capture the GPU's display output — which is how it streams your desktop or gaming session without needing a virtual display.

**1c — Configure Sunshine to use KMS:**

By default, Sunshine tries to use X11 or Wayland screen capture protocols, which require an active user session. For a headless setup, you must explicitly tell Sunshine to capture directly from the kernel using KMS.

```bash
mkdir -p ~/.config/sunshine

cat > ~/.config/sunshine/sunshine.conf << 'EOF'
capture = kms
encoder = vaapi
EOF
```

!!! warning "Multi-GPU & eGPU Setups"
    If both your iGPU and eGPU are connected with dummy plugs, Sunshine's KMS capture defaults to `/dev/dri/card0` (the iGPU) and will stream a black screen. Our `gpu-power-init.service` boot script automatically appends and updates the correct stable `/dev/dri/by-path/` PCIe slot path to the `adapter_name` parameter at boot.

**1d — Enable login persistence and reboot:**

```bash
sudo loginctl enable-linger $(whoami)
sudo systemctl reboot
```

Wait ~60 seconds, then reconnect:

```bash
ssh -p 22022 username@192.168.1.50
```

---

## Step 2 — Deploy wg-easy (WireGuard VPN)

!!! warning "Router prerequisite"
    Confirm that port **51820/UDP** is forwarded to your server's IP in your router (set up in Phase 2).

**Generate your admin password hash** (replace `your-admin-password` with a real password):

```bash
docker run --rm ghcr.io/wg-easy/wg-easy wgpw 'your-admin-password'
```

The output will be a bcrypt hash like: `$2a$12$xxxxxxxxxx...`

!!! warning "Dollar signs must be doubled in docker-compose.yml"
    In YAML files, `$` has special meaning. Every `$` in your bcrypt hash must be written as `$$`. For example: `$2a$12$abc...` becomes `$$2a$$12$$abc...`

```bash
mkdir -p ~/docker/wireguard && cd ~/docker/wireguard

cat > docker-compose.yml << 'EOF'
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:latest
    container_name: wg-easy
    restart: unless-stopped
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    environment:
      - WG_HOST=yourname.duckdns.org   # ← Your DuckDNS domain
      - PASSWORD_HASH=$$2a$$12$$...    # ← Paste your hash here ($ doubled, no quotes)
    volumes:
      - ./data:/etc/wireguard:Z
    ports:
      - "51820:51820/udp"
      - "127.0.0.1:51821:51821/tcp"   # Web UI — localhost only
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
EOF

docker compose up -d
```

**Access the wg-easy admin panel** via SSH tunnel from your personal computer:

```bash
ssh -N -L 51821:127.0.0.1:51821 -p 22022 username@192.168.1.50
```

Open `http://localhost:51821` in your browser. Create a client profile for each device you want to connect (phone, laptop, etc.) and download the `.conf` file or scan the QR code.

---

## Step 3 — Configure Sunshine

**3a — Open Sunshine's firewall ports:**

```bash
sudo firewall-cmd --permanent --add-port={47984,47989,48010}/tcp
sudo firewall-cmd --permanent --add-port={47998,47999,48000,48002,48010}/udp
sudo firewall-cmd --reload

# Always restart Docker after a firewalld reload!
sudo systemctl restart docker
```

!!! danger "Always restart Docker after `firewall-cmd --reload`"
    firewalld flushes and rebuilds the entire nftables ruleset when you reload it. This wipes Docker's internal port-forwarding rules. Running `sudo systemctl restart docker` immediately after restores them.

**3b — Create and enable a System Service for Sunshine:**

By default, Sunshine runs as a *user* service, meaning it only starts *after* an active session starts. To guarantee Sunshine is always running and accessible upon boot (crucial for a headless server), we will disable the user service and install it as a system-level service.

```bash
# Disable user services that might have been enabled by `ujust setup-sunshine` or `brew`
systemctl --user disable --now sunshine.service sunshine-kms.service homebrew.sunshine.service 2>/dev/null || true

# Create the system service drop-in file
sudo tee /etc/systemd/system/sunshine.service << EOF
[Unit]
Description=Sunshine Game Stream Server (KMS System Service)
After=network-online.target
Wants=network-online.target

[Service]
User=$(whoami)
Group=$(whoami)
ExecStartPre=/bin/sleep 5
ExecStart=/usr/bin/sunshine /home/$(whoami)/.config/sunshine/sunshine.conf
Restart=on-failure
RestartSec=5s
# Grant KMS permissions directly to the service
AmbientCapabilities=CAP_SYS_ADMIN
CapabilityBoundingSet=CAP_SYS_ADMIN

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd and start Sunshine on boot
sudo systemctl daemon-reload
sudo systemctl enable --now sunshine.service
```

**3c — Set up Sunshine's web UI:**

Create an SSH tunnel from your personal computer:

```bash
ssh -N -L 47990:127.0.0.1:47990 -p 22022 username@192.168.1.50
```

Open `https://localhost:47990` in your browser. Accept the self-signed certificate warning. Set a username and password when prompted.

---

## Step 4 — Connect Moonlight

This is the full remote streaming workflow once everything is configured:

1. **Connect VPN** — Open WireGuard on your device and activate your profile. Your server becomes accessible at `192.168.1.50` (use your private IP address.
2. **Launch a session** — SSH into the server and run `sudo start-kde.sh` or `sudo start-gaming.sh`
3. **Open Moonlight** — Click **Add Host** and enter `<your-ip-address>`
4. **Stream** — Select **Desktop** or **Steam Big Picture**
5. **Stop** — Close Moonlight, then run `sudo stop-session.sh` over SSH to return to headless mode

!!! tip "Phase 12 makes this one click"
    Phase 12 adds a webhook server and a client launcher script that automates all five steps above into a single button press. It's optional but highly recommended.

---

## ✅ Phase 9 Complete

WireGuard VPN and Sunshine are running as a system service with KMS enabled. You can stream your desktop or Gaming Mode over an encrypted VPN tunnel, and Sunshine will always be ready after a reboot without needing an active user session.

[Next: Phase 10 — Nextcloud Init →](phase-10-nextcloud.md){ .next-phase }
