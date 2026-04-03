# Phase 12: Seamless Remote Client Integration

This guide provides a seamless, "one-click" solution to launch remote gaming sessions on a Bazzite-Deck host from any KDE Plasma client (like a laptop or Steam Deck in Desktop mode). 

It automates VPN connectivity, handles session conflicts (force-closing active sessions), launches Moonlight, and cleans up the server state upon exit.

---

## Part 1: Server-Side Setup (Bazzite-Deck)

Bazzite is an immutable OS; we install the Webhook binary natively to ensure it survives updates and has direct access to the host session management scripts.

### 1. Install the Webhook Binary

Download and move the standalone binary to your local `bin` directory:

```bash
cd /tmp
wget https://github.com/adnanh/webhook/releases/download/2.8.1/webhook-linux-amd64.tar.gz
tar -xzf webhook-linux-amd64.tar.gz
sudo mv webhook-linux-amd64/webhook /usr/local/bin/webhook
sudo chmod +x /usr/local/bin/webhook
sudo chown root:root /usr/local/bin/webhook
sudo restorecon -Rv /usr/local/bin/webhook
```

### 2. Configure the Hooks

Create the configuration file. We use a shell wrapper (`2>&1 || true`) to ensure that error messages are sent back to the client and that the Webhook always returns a "Success" status so the client can parse the output.

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

### 3. Create the Systemd Service

Create a background service to keep the Webhook listener active. 

!!! warning "Replace `YOUR_USER`"
    Make sure to replace `YOUR_USER` below with your actual Bazzite username!

```bash
sudo nano /etc/systemd/system/webhook.service
```

Paste this content:

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

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now webhook
```

### 4. Firewall Configuration

Open port `9000` and trust the Docker interface (since WireGuard runs in a container).

```bash
sudo firewall-cmd --permanent --add-port=9000/tcp
sudo firewall-cmd --permanent --zone=trusted --add-interface=docker0
sudo firewall-cmd --reload
sudo systemctl restart docker
```

---

## Part 2: Client-Side Setup (KDE Desktop PC/Laptop)

Perform these steps on the device you use to play games remotely.

### 1. Import and Configure the VPN

First, import your `sotohome.conf` (or whatever your VPN profile is named) file into Network Manager:

```bash
nmcli connection import type wireguard file /path/to/sotohome.conf
```

To ensure your internet remains active while the VPN is connected and Moonlight can find the server:

1. Open **System Settings > Connections**.
2. Select the `sotohome` VPN.
3. Go to the **IPv4 tab > Routes...**
4. Check **"IPv4 is required for this connection"**.
5. Check **"Use only for resources on this connection"**. (This prevents the VPN from hijacking your entire internet connection).
6. Click **Add** and enter your server's LAN IP (e.g., `192.168.178.139`) with Netmask `32` and Gateway `0.0.0.0`.

### 2. Create the Master Launch Script

Create the script that orchestrates the entire session.

```bash
mkdir -p ~/.local/bin
nano ~/.local/bin/play-sotohome.sh
```

Paste the final script:

```bash
#!/bin/bash

# Configuration
SERVER_IP="192.168.178.139"
WEBHOOK_PORT="9000"

# Connect to VPN
if nmcli connection up sotohome; then
    sleep 3
    kdialog --passivepopup "Checking server status..." 3
    
    # Check server status and capture the response
    RESPONSE=$(curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/start-gaming")

    # Check if a session is already active
    if echo "$RESPONSE" | grep -iq "ERROR: A graphical session is already active"; then
        kdialog --title "SotoHome: Session Conflict" --yesno "Do you want to stop the active session to start a new one?"
        
        if [ $? -eq 0 ]; then
            # User clicked "Yes" - Restart the session
            kdialog --passivepopup "Force-closing existing session... please wait." 5
            curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/stop-session" > /dev/null
            sleep 8
            kdialog --passivepopup "Restarting gaming session..." 3
            curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/start-gaming" > /dev/null
            sleep 6
        fi
        # If user clicked "No", the script just skips the restart and joins the active session
    fi

    kdialog --passivepopup "Sotohome: Ready. Launching Moonlight..." 5
    
    # Launch Moonlight and capture its exit code immediately after it closes
    if command -v flatpak >/dev/null 2>&1 && flatpak list | grep -q com.moonlight_stream.Moonlight; then
        flatpak run com.moonlight_stream.Moonlight stream hs "sotohome"
        MOONLIGHT_EXIT=$?
    else
        moonlight stream hs "sotohome"
        MOONLIGHT_EXIT=$?
    fi

    # Check if Moonlight closed normally (0) or crashed/dropped connection (non-zero)
    if [ $MOONLIGHT_EXIT -eq 0 ]; then
        # Normal exit: Run the stop-session script to clean up the host
        kdialog --passivepopup "Sotohome: Cleaning up server session..." 5
        curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/stop-session" > /dev/null
    else
        # Crash/Disconnect exit: Do NOT stop the session on the host
        kdialog --passivepopup "Sotohome: Connection lost. Keeping host session alive..." 4
    fi

    # ALWAYS disconnect the VPN as a fail-safe
    sleep 2
    nmcli connection down sotohome

else
    # VPN failed to connect
    kdialog --error "Failed to connect to the Sotohome VPN."
fi
```

Make it executable: 
```bash
chmod +x ~/.local/bin/play-sotohome.sh
```

### 3. Create Desktop Integration

To add this to your KDE Application Menu, create a desktop entry. Replace `YOUR_USER` with your local client's username.

```bash
nano ~/.local/share/applications/sotohome.desktop
```

Paste this content:

```ini
[Desktop Entry]
Name=Play SotoHome
Exec=/home/YOUR_USER/.local/bin/play-sotohome.sh
Icon=moonlight
Type=Application
Categories=Game;
Terminal=false
```

**Final Step:** Open your Application Menu, search for *Play SotoHome*, and launch. The system will now handle the VPN, check for session conflicts, launch Moonlight, and manage the server state automatically!
