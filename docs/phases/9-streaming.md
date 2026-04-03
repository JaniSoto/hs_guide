```markdown
# Phase 9: WireGuard VPN + Sunshine Streaming

!!! concept "Why WireGuard + Sunshine instead of RDP?"
    RDP (and krdp) uses TCP, which suffers from "TCP Meltdown" during packet loss — stuttering and freezing that makes gaming unplayable. Sunshine uses UDP with VA-API hardware encoding, delivering 4K/60fps with near-zero latency. 
    
    WireGuard (also UDP) provides the secure tunnel: unauthenticated packets are silently dropped, making port 51820 invisible to portscanners. **Zero RDP required.**

## 1. Host OS Preparation

**Step 1a — Load legacy NAT kernel modules.**
```bash
sudo modprobe iptable_nat iptable_mangle
printf "iptable_nat\niptable_mangle\n" | sudo tee /etc/modules-load.d/iptables-legacy.conf
```

**Step 1b — Grant hardware permissions and set up Sunshine.**
```bash
# Add your user to input, video, and render groups
sudo usermod -aG input,video,render $(whoami)

# Bazzite built-in Sunshine setup
sudo ujust setup-sunshine
```

!!! info "How Sunshine captures via KMS"
    Running `ujust setup-sunshine` is critical. It sets the `cap_sys_admin` capability on the Sunshine binary and configures udev rules. This allows Sunshine to use **KMS (Kernel Mode Setting)** to directly capture the DRM master display output. 
    
    Because only one process can be the DRM master at a time, our custom session scripts (`start-gaming.sh` and `start-kde.sh`) explicitly stop `sddm` and existing `sunshine` services to fully release DRM locks before starting a new session. This guarantees Sunshine can grab KMS without "display not found" errors.

**Step 1c — Enable login persistence (linger) and reboot.**
```bash
sudo loginctl enable-linger $(whoami)
sudo systemctl reboot
```
*(Wait ~60 seconds, then reconnect via SSH: `ssh -p 22022 username@...`)*

## 2. Deploy wg-easy (WireGuard VPN)

*Router prerequisite:* Ensure port 51820/UDP is forwarded to your server's local IP in your router.

```bash
# Generate the admin password hash (Replace 'your-admin-password')
docker run --rm ghcr.io/wg-easy/wg-easy wgpw 'your-admin-password'
# Output: $2a$12$xxxxxxxxx...
```
*(Important: In docker-compose.yml, every `$` in this hash must be doubled: `$$`)*

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
      - WG_HOST=yourname.duckdns.org # Your public DuckDNS domain.
      - PASSWORD_HASH=$$2a$$12$$...  # Paste bcrypt hash ($ doubled). No quotes.
    volumes:
      - ./data:/etc/wireguard:Z
    ports:
      - "51820:51820/udp"
      - "127.0.0.1:51821:51821/tcp"
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
EOF
docker compose up -d
```

## 3. Configure Sunshine Streaming

**Step 3a — Open Sunshine firewall ports.**
```bash
sudo firewall-cmd --permanent --add-port={47984,47989,48010}/tcp
sudo firewall-cmd --permanent --add-port={47998,47999,48000,48002,48010}/udp
sudo firewall-cmd --reload

# CRITICAL: Always restart Docker after a firewalld reload!
sudo systemctl restart docker
```

**Step 3b — Enable the Sunshine user service.**
```bash
systemctl --user enable --now sunshine
```

**Step 3c — Initial Sunshine web UI setup.**
1. On your PERSONAL COMPUTER — create an SSH tunnel to the admin panel:
   `ssh -N -L 47990:127.0.0.1:47990 -p 22022 username@your-server-ip`
2. Open `https://localhost:47990` in your browser.
3. Accept the self-signed certificate warning and set a username/password.

## 4. Remote workflow — Client Side

1. **Connect VPN:** Open WireGuard on your device and activate the profile. (Server IP becomes `10.8.0.1`).
2. **Launch session:** SSH into the server and run `sudo start-kde.sh` or `sudo start-gaming.sh`. *(Alternatively, use the Webhook setup in Phase 12!)*
3. **Open Moonlight:** Click "+ Add Host" and enter `10.8.0.1`.
4. **Stream:** Select "Desktop" or "Steam Big Picture".
5. **Stop:** Close Moonlight. Run `sudo stop-session.sh` over SSH to return to headless mode.
```
