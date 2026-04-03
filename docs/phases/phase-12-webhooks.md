<div class="phase-header">
  <span class="phase-num">12</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 12 · Optional · Server + Client setup</p>
    <h1>One-Click Remote Gaming Client</h1>
    <p class="phase-header-sub">Add a webhook server and client launcher for one-click gaming sessions — no SSH commands needed.</p>
  </div>
</div>

This phase builds a "one-click" system: you press a single button on your KDE laptop (or any client), and it automatically connects the VPN, resolves any session conflicts, starts the gaming session on the server, launches Moonlight, and cleans up the server when you're done.

---

## Part 1 — Server-Side Setup

### 1 — Install the Webhook Binary

The `webhook` tool listens for HTTP requests and runs shell commands in response. We install it as a standalone binary so it works on Bazzite's immutable filesystem and survives OS updates.

```bash
cd /tmp
wget https://github.com/adnanh/webhook/releases/download/2.8.1/webhook-linux-amd64.tar.gz
tar -xzf webhook-linux-amd64.tar.gz
sudo mv webhook-linux-amd64/webhook /usr/local/bin/webhook
sudo chmod +x /usr/local/bin/webhook
sudo chown root:root /usr/local/bin/webhook
sudo restorecon -Rv /usr/local/bin/webhook
```

---

### 2 — Configure the Webhook Hooks

Create the configuration file:

```bash
sudo nano /etc/webhook.conf
```

Paste this configuration:

```json
[
  {
    "id": "start-gaming",
    "execute-command": "/bin/bash",
    "include-command-output-in-response": true,
    "pass-arguments-to-command": [
      { "source": "string", "name": "-c" },
      { "source": "string", "name": "sudo /usr/local/bin/start-gaming.sh 2>&1 || true" }
    ]
  },
  {
    "id": "stop-session",
    "execute-command": "/bin/bash",
    "include-command-output-in-response": true,
    "pass-arguments-to-command": [
      { "source": "string", "name": "-c" },
      { "source": "string", "name": "sudo /usr/local/bin/stop-session.sh 2>&1 || true" }
    ]
  }
]
```

The `|| true` at the end ensures that even if the script exits with an error, the webhook returns a response. This lets the client script read the error message and handle it (for example, detecting "session already active").

---

### 3 — Create the Systemd Service

```bash
sudo nano /etc/systemd/system/webhook.service
```

Paste this content — **replace `YOUR_USER` with your Bazzite username:**

```ini
[Unit]
Description=Webhook Service
After=network.target

[Service]
ExecStart=/usr/local/bin/webhook -hooks /etc/webhook.conf -port 9000
Restart=always
User=YOUR_USER
Group=YOUR_USER

[Install]
WantedBy=multi-user.target
```

Enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now webhook
```

---

### 4 — Firewall Configuration

Open port 9000 and trust the Docker network interface (since wg-easy runs in a container, its traffic arrives on `docker0`):

```bash
sudo firewall-cmd --permanent --add-port=9000/tcp
sudo firewall-cmd --permanent --zone=trusted --add-interface=docker0
sudo firewall-cmd --reload
sudo systemctl restart docker
```

---

## Part 2 — Client-Side Setup (KDE Laptop / Desktop)

Do these steps on the device you play games from.

### 1 — Import and Configure the VPN

Import your WireGuard profile into NetworkManager:

```bash
nmcli connection import type wireguard file /path/to/your-vpn-profile.conf
```

To keep your internet working while the VPN is connected (so Moonlight can reach the server by local IP):

1. Open **System Settings → Connections**
2. Select your VPN connection
3. Go to the **IPv4** tab → click **Routes...**
4. Check **"Use only for resources on this connection"** — this prevents the VPN from routing all your traffic
5. Click **Add** and enter your server's LAN IP (e.g. `192.168.1.50`) with Netmask `32`, Gateway `0.0.0.0`
6. Click **OK** and save

---

### 2 — Create the Master Launch Script

```bash
mkdir -p ~/.local/bin
nano ~/.local/bin/play-home.sh
```

Paste this script — **update `SERVER_IP` to your server's local IP:**

```bash
#!/bin/bash

SERVER_IP="192.168.1.50"   # ← Your server's local IP
VPN_NAME="your-vpn-name"   # ← Name of your WireGuard connection in NetworkManager
WEBHOOK_PORT="9000"

# Connect to VPN
if nmcli connection up "$VPN_NAME"; then
    sleep 3
    kdialog --passivepopup "Checking server status..." 3

    # Trigger gaming session and capture the response
    RESPONSE=$(curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/start-gaming")

    # Handle session conflict
    if echo "$RESPONSE" | grep -iq "ERROR: A graphical session is already active"; then
        kdialog --title "Session Conflict" \
          --yesno "A session is already active on the server. Stop it and start a new one?"

        if [ $? -eq 0 ]; then
            kdialog --passivepopup "Stopping existing session..." 5
            curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/stop-session" > /dev/null
            sleep 8
            kdialog --passivepopup "Starting gaming session..." 3
            curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/start-gaming" > /dev/null
            sleep 6
        fi
    fi

    kdialog --passivepopup "Ready. Launching Moonlight..." 5

    # Launch Moonlight
    if command -v flatpak >/dev/null 2>&1 && flatpak list | grep -q com.moonlight_stream.Moonlight; then
        flatpak run com.moonlight_stream.Moonlight
        MOONLIGHT_EXIT=$?
    else
        moonlight
        MOONLIGHT_EXIT=$?
    fi

    # On clean exit: stop the session on the server
    if [ $MOONLIGHT_EXIT -eq 0 ]; then
        kdialog --passivepopup "Cleaning up server session..." 5
        curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/stop-session" > /dev/null
    else
        # On crash/disconnect: keep the session alive so you can reconnect
        kdialog --passivepopup "Connection lost. Keeping server session alive..." 4
    fi

    # Always disconnect the VPN
    sleep 2
    nmcli connection down "$VPN_NAME"

else
    kdialog --error "Failed to connect to VPN."
fi
```

Make it executable:

```bash
chmod +x ~/.local/bin/play-home.sh
```

---

### 3 — Create the Desktop Application Shortcut

```bash
nano ~/.local/share/applications/play-home.desktop
```

Paste — **replace `YOUR_USER` with your local username:**

```ini
[Desktop Entry]
Name=Play Home
Exec=/home/YOUR_USER/.local/bin/play-home.sh
Icon=moonlight
Type=Application
Categories=Game;
Terminal=false
```

Open your application launcher, search for **"Play Home"**, and run it. The system will now handle VPN, session management, streaming, and cleanup automatically.

---

## ✅ Phase 12 Complete — Setup Finished 🎉

Your complete home server setup is done. Here's what you built:

| Component | Status |
|-----------|--------|
| Headless Bazzite server | ✅ Running 24/7 |
| Nextcloud AIO | ✅ Live at `yourname.duckdns.org` |
| Nginx Proxy Manager + Let's Encrypt | ✅ Auto-renewing HTTPS |
| DuckDNS dynamic DNS | ✅ Auto-updating every 5 min |
| WireGuard VPN | ✅ Accessible from anywhere |
| Sunshine/Moonlight streaming | ✅ Hardware-accelerated |
| Session management scripts | ✅ KDE + Gaming Mode on demand |
| Watchtower auto-updates | ✅ Containers updated at 4 AM |
| Borg backups | ✅ Daily snapshots |
| One-click client launcher | ✅ Single button to play |

---

For quick reference commands, see [Quick Commands →](../reference/quick-commands.md)

For troubleshooting, see [Troubleshooting →](../reference/troubleshooting.md)
