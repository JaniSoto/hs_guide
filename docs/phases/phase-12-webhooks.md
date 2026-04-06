<div class="phase-header">
  <span class="phase-num">12</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 12 · Optional · Server + Client setup</p>
    <h1>One-Click Remote Gaming Client</h1>
    <p class="phase-header-sub">Add a webhook server and client launcher for one-click gaming sessions — no SSH commands needed.</p>
  </div>
</div>

This phase builds a "one-click" system: you press a single button on your laptop (or any client), and it automatically connects the VPN, resolves any session conflicts, starts the gaming session on the server, launches Moonlight, and cleans up the server when you're done.

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

## Part 2 — Client-Side Setup (Laptop / Desktop)

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

Choose the appropriate script below for your desktop environment. The **Admin** versions include a prompt asking if you want to launch a Gaming session or a remote KDE desktop session. **Make sure to update the variables at the top** to match your network and configuration.

=== "KDE Plasma"
    ```bash
    #!/bin/bash

    SERVER_IP="<your-server-ip>"
    VPN_NAME="<your-vpn-name>"
    APP_NAME="Desktop"         # ← Name of the app in Sunshine to launch
    WEBHOOK_PORT="9000"

    # Connect to VPN
    if nmcli connection up "$VPN_NAME"; then
        
        # Wait for the server to become reachable
        MAX_WAIT=15
        WAIT_COUNT=0
        while ! ping -c 1 -W 1 "$SERVER_IP" > /dev/null 2>&1; do
            sleep 1
            WAIT_COUNT=$((WAIT_COUNT + 1))
            if [ "$WAIT_COUNT" -ge "$MAX_WAIT" ]; then
                kdialog --error "Connected to VPN, but server $SERVER_IP is unreachable."
                nmcli connection down "$VPN_NAME"
                exit 1
            fi
        done

        # Trigger gaming session and capture the response
        RESPONSE=$(curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/start-gaming")

        # Handle session conflict
        if echo "$RESPONSE" | grep -iq "ERROR: A graphical session is already active"; then
            if kdialog --title "Session Conflict" --yesno "A session is already active on the server. Do you want to stop it and start a new one?"; then
                kdialog --passivepopup "Stopping existing session..." 3
                curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/stop-session" > /dev/null
                sleep 8
                kdialog --passivepopup "Starting gaming session..." 3
                curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/start-gaming" > /dev/null
                sleep 8
            fi
        fi

        kdialog --passivepopup "Ready. Launching Moonlight..." 3

        # Launch Moonlight
        if command -v flatpak >/dev/null 2>&1 && flatpak list | grep -q com.moonlight_stream.Moonlight; then
            flatpak run com.moonlight_stream.Moonlight stream "$SERVER_IP" "$APP_NAME"
            MOONLIGHT_EXIT=$?
        else
            moonlight stream "$SERVER_IP" "$APP_NAME"
            MOONLIGHT_EXIT=$?
        fi

        # On clean exit: stop the session on the server
        if [ $MOONLIGHT_EXIT -eq 0 ]; then
            kdialog --passivepopup "Cleaning up server session..." 3
            curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/stop-session" > /dev/null
        else
            # On crash/disconnect: keep the session alive so you can reconnect
            kdialog --passivepopup "Connection lost. Keeping server session alive..." 3
        fi

        # Always disconnect the VPN
        sleep 2
        nmcli connection down "$VPN_NAME"

    else
        kdialog --error "Failed to connect to VPN."
    fi
    ```

=== "GNOME"
    ```bash
    #!/bin/bash

    SERVER_IP="<your-server-ip>"
    VPN_NAME="<your-vpn-name>"
    APP_NAME="Desktop"         # ← Name of the app in Sunshine to launch
    WEBHOOK_PORT="9000"

    # Connect to VPN
    if nmcli connection up "$VPN_NAME"; then
        
        # Wait for the server to become reachable
        MAX_WAIT=15
        WAIT_COUNT=0
        while ! ping -c 1 -W 1 "$SERVER_IP" > /dev/null 2>&1; do
            sleep 1
            WAIT_COUNT=$((WAIT_COUNT + 1))
            if [ "$WAIT_COUNT" -ge "$MAX_WAIT" ]; then
                zenity --error --text="Connected to VPN, but server $SERVER_IP is unreachable."
                nmcli connection down "$VPN_NAME"
                exit 1
            fi
        done

        # Trigger gaming session and capture the response
        RESPONSE=$(curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/start-gaming")

        # Handle session conflict
        if echo "$RESPONSE" | grep -iq "ERROR: A graphical session is already active"; then      
            if zenity --question --text="A session is already active on the server. Do you want to stop it and start a new one?"; then
                notify-send -t 3000 "Stopping existing session..."
                curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/stop-session" > /dev/null
                sleep 8
                notify-send -t 3000 "Starting gaming session..."
                curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/start-gaming" > /dev/null
                sleep 8
            fi
        fi

        notify-send -t 3000 "Ready. Launching Moonlight..."

        # Launch Moonlight
        if command -v flatpak >/dev/null 2>&1 && flatpak list | grep -q com.moonlight_stream.Moonlight; then
            flatpak run com.moonlight_stream.Moonlight stream "$SERVER_IP" "$APP_NAME"
            MOONLIGHT_EXIT=$?
        else
            moonlight stream "$SERVER_IP" "$APP_NAME"
            MOONLIGHT_EXIT=$?
        fi

        # On clean exit: stop the session on the server
        if [ $MOONLIGHT_EXIT -eq 0 ]; then
            notify-send -t 3000 "Cleaning up server session..."
            curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/stop-session" > /dev/null	
        else
            # On crash/disconnect: keep the session alive so you can reconnect
            notify-send -t 3000 "Connection lost. Keeping server session alive..."
        fi

        # Always disconnect the VPN
        sleep 2
        nmcli connection down "$VPN_NAME"

    else
        zenity --error --text="Failed to connect to VPN."
    fi
    ```

=== "KDE Plasma (Admin)"
    ```bash
    #!/bin/bash

    # --- Configuration ---
    SERVER_IP="<your-server-ip>"
    VPN_NAME="<your-vpn-name>"
    APP_NAME="Desktop"           # ← Name of the app in Sunshine to launch
    WEBHOOK_PORT="9000"
    SSH_ALIAS="<your-ssh-alias>"   # ← Your SSH config alias for the server

    # Connect to VPN
    if nmcli connection up "$VPN_NAME"; then

        # Wait for the server to become reachable
        MAX_WAIT=15
        WAIT_COUNT=0
        while ! ping -c 1 -W 1 "$SERVER_IP" > /dev/null 2>&1; do
            sleep 1
            WAIT_COUNT=$((WAIT_COUNT + 1))
            if [ "$WAIT_COUNT" -ge "$MAX_WAIT" ]; then
                kdialog --error "Connected to VPN, but server $SERVER_IP is unreachable."
                nmcli connection down "$VPN_NAME"
                exit 1
            fi
        done

        # Prompt user for session type
        SESSION_TYPE=$(kdialog --title "Select Session" \
            --radiolist "Which session would you like to start?" \
            "Gaming" "Gaming" on \
            "KDE" "KDE" off)

        # If the user clicks "Cancel" or closes the window, disconnect VPN and exit
        if [ -z "$SESSION_TYPE" ]; then
            nmcli connection down "$VPN_NAME"
            exit 0
        fi

        if [ "$SESSION_TYPE" == "KDE" ]; then
            kdialog --passivepopup "Starting KDE session..." 3

            # Create a temporary askpass script so SSH can use kdialog to ask for the passphrase
            ASKPASS_SCRIPT=$(mktemp)
            echo '#!/bin/bash' > "$ASKPASS_SCRIPT"
            echo 'kdialog --password "SSH Authentication for '"$SSH_ALIAS"'"' >> "$ASKPASS_SCRIPT"
            chmod +x "$ASKPASS_SCRIPT"

            export SSH_ASKPASS="$ASKPASS_SCRIPT"
            export SSH_ASKPASS_REQUIRE="force"

            # Run SSH and capture output to check for conflicts
            SSH_OUTPUT=$(setsid -w ssh "$SSH_ALIAS" "sudo start-kde.sh" 2>&1 < /dev/null)
            SSH_EXIT=$?

            # --- Handle Session Conflict for KDE ---
            if echo "$SSH_OUTPUT" | grep -iq "already active"; then      
                if kdialog --title "Session Conflict" --yesno "A graphical session is already active on the server. Do you want to stop it and start KDE?"; then
                    kdialog --passivepopup "Stopping existing session..." 3
                    # Stop existing session via SSH
                    ssh "$SSH_ALIAS" "sudo stop-session.sh" > /dev/null 2>&1
                    sleep 5
                    kdialog --passivepopup "Starting KDE session..." 3
                    # Try starting KDE again
                    SSH_OUTPUT=$(setsid -w ssh "$SSH_ALIAS" "sudo start-kde.sh" 2>&1 < /dev/null)
                    SSH_EXIT=$?
                else
                    rm -f "$ASKPASS_SCRIPT"
                    nmcli connection down "$VPN_NAME"
                    exit 0
                fi
            fi

            # Clean up the temporary askpass script
            rm -f "$ASKPASS_SCRIPT"

            # If SSH failed for any other reason
            if [ $SSH_EXIT -ne 0 ]; then
                kdialog --error "SSH Failed: $SSH_OUTPUT"
                nmcli connection down "$VPN_NAME"
                exit 1
            fi

            sleep 2 # Small pause to let the remote KDE session start up

        elif [ "$SESSION_TYPE" == "Gaming" ]; then
            # Trigger gaming session via Webhook
            RESPONSE=$(curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/start-gaming")

            # --- Handle Session Conflict for Gaming ---
            if echo "$RESPONSE" | grep -iq "ERROR: A graphical session is already active"; then      
                if kdialog --title "Session Conflict" --yesno "A session is already active on the server. Do you want to stop it and start a new one?"; then
                    kdialog --passivepopup "Stopping existing session..." 3
                    curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/stop-session" > /dev/null
                    sleep 8
                    kdialog --passivepopup "Starting gaming session..." 3
                    curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/start-gaming" > /dev/null
                    sleep 8
                fi
            fi
        fi

        kdialog --passivepopup "Ready. Launching Moonlight..." 3

        # Launch Moonlight
        if command -v flatpak >/dev/null 2>&1 && flatpak list | grep -q com.moonlight_stream.Moonlight; then
            flatpak run com.moonlight_stream.Moonlight stream "$SERVER_IP" "$APP_NAME"
            MOONLIGHT_EXIT=$?
        else
            moonlight stream "$SERVER_IP" "$APP_NAME"
            MOONLIGHT_EXIT=$?
        fi

        # On clean exit: stop the session on the server
        if [ $MOONLIGHT_EXIT -eq 0 ]; then
            kdialog --passivepopup "Cleaning up server session..." 3
            curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/stop-session" > /dev/null  
        else
            # On crash/disconnect: keep the session alive so you can reconnect
            kdialog --passivepopup "Connection lost. Keeping server session alive..." 3
        fi

        # Always disconnect the VPN
        sleep 2
        nmcli connection down "$VPN_NAME"

    else
        kdialog --error "Failed to connect to VPN."
    fi
    ```

=== "GNOME (Admin)"
    ```bash
    #!/bin/bash

    # --- Configuration ---
    SERVER_IP="<your-server-ip>"
    VPN_NAME="<your-vpn-name>"
    APP_NAME="Desktop"           # ← Name of the app in Sunshine to launch
    WEBHOOK_PORT="9000"
    SSH_ALIAS="<your-ssh-alias>"   # ← Your SSH config alias for the server

    # Connect to VPN
    if nmcli connection up "$VPN_NAME"; then

        # Wait for the server to become reachable
        MAX_WAIT=15
        WAIT_COUNT=0
        while ! ping -c 1 -W 1 "$SERVER_IP" > /dev/null 2>&1; do
            sleep 1
            WAIT_COUNT=$((WAIT_COUNT + 1))
            if [ "$WAIT_COUNT" -ge "$MAX_WAIT" ]; then
                zenity --error --text="Connected to VPN, but server $SERVER_IP is unreachable."
                nmcli connection down "$VPN_NAME"
                exit 1
            fi
        done

        # Prompt user for session type
        SESSION_TYPE=$(zenity --list \
            --title="Select Session" \
            --text="Which session would you like to start?" \
            --radiolist \
            --column="Select" --column="Session Type" \
            TRUE "Gaming" \
            FALSE "KDE" \
            --width=300 --height=200)

        # If the user clicks "Cancel" or closes the window, disconnect VPN and exit
        if [ -z "$SESSION_TYPE" ]; then
            nmcli connection down "$VPN_NAME"
            exit 0
        fi

        if [ "$SESSION_TYPE" == "KDE" ]; then
            notify-send -t 3000 "Starting KDE session..."

            # Create a temporary askpass script for GUI SSH authentication
            ASKPASS_SCRIPT=$(mktemp)
            echo '#!/bin/bash' > "$ASKPASS_SCRIPT"
            echo 'zenity --password --title="SSH Authentication for '"$SSH_ALIAS"'"' >> "$ASKPASS_SCRIPT"
            chmod +x "$ASKPASS_SCRIPT"

            export SSH_ASKPASS="$ASKPASS_SCRIPT"
            export SSH_ASKPASS_REQUIRE="force"

            # Try to start KDE and capture output/errors
            SSH_OUTPUT=$(setsid -w ssh "$SSH_ALIAS" "sudo start-kde.sh" 2>&1 < /dev/null)
            SSH_EXIT=$?

            # --- Handle Session Conflict for KDE ---
            if echo "$SSH_OUTPUT" | grep -iq "already active"; then      
                if zenity --question --text="A graphical session is already active on the server. Do you want to stop it and start KDE?"; then
                    notify-send -t 3000 "Stopping existing session..."
                    # Run stop script via SSH
                    ssh "$SSH_ALIAS" "sudo stop-session.sh" > /dev/null 2>&1
                    sleep 5
                    notify-send -t 3000 "Starting KDE session..."
                    # Try starting KDE again
                    SSH_OUTPUT=$(setsid -w ssh "$SSH_ALIAS" "sudo start-kde.sh" 2>&1 < /dev/null)
                    SSH_EXIT=$?
                else
                    rm -f "$ASKPASS_SCRIPT"
                    nmcli connection down "$VPN_NAME"
                    exit 0
                fi
            fi

            rm -f "$ASKPASS_SCRIPT"

            # If SSH failed for any other reason
            if [ $SSH_EXIT -ne 0 ]; then
                zenity --error --text="SSH Failed: $SSH_OUTPUT"
                nmcli connection down "$VPN_NAME"
                exit 1
            fi

            sleep 2 

        elif [ "$SESSION_TYPE" == "Gaming" ]; then
            # Trigger gaming session via Webhook
            RESPONSE=$(curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/start-gaming")

            # --- Handle Session Conflict for Gaming ---
            if echo "$RESPONSE" | grep -iq "ERROR: A graphical session is already active"; then      
                if zenity --question --text="A session is already active on the server. Do you want to stop it and start a new one?"; then
                    notify-send -t 3000 "Stopping existing session..."
                    curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/stop-session" > /dev/null
                    sleep 8
                    notify-send -t 3000 "Starting gaming session..."
                    curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/start-gaming" > /dev/null
                    sleep 8
                fi
            fi
        fi

        notify-send -t 3000 "Ready. Launching Moonlight..."

        # Launch Moonlight
        if command -v flatpak >/dev/null 2>&1 && flatpak list | grep -q com.moonlight_stream.Moonlight; then
            flatpak run com.moonlight_stream.Moonlight stream "$SERVER_IP" "$APP_NAME"
            MOONLIGHT_EXIT=$?
        else
            moonlight stream "$SERVER_IP" "$APP_NAME"
            MOONLIGHT_EXIT=$?
        fi

        # On clean exit: stop the session on the server
        if [ $MOONLIGHT_EXIT -eq 0 ]; then
            notify-send -t 3000 "Cleaning up server session..."
            curl -s -m 15 "http://$SERVER_IP:$WEBHOOK_PORT/hooks/stop-session" > /dev/null  
        else
            # On crash/disconnect: keep the session alive so you can reconnect
            notify-send -t 3000 "Connection lost. Keeping server session alive..."
        fi

        # Always disconnect the VPN
        sleep 2
        nmcli connection down "$VPN_NAME"

    else
        zenity --error --text="Failed to connect to VPN."
    fi 
    ```

Make your script executable:

```bash
chmod +x ~/.local/bin/play-home.sh
```

---

### 3 — Create the Desktop Application Shortcut

```bash
nano ~/.local/share/applications/play-home.desktop
```

Paste — **replace `YOUR_USER` with your local client username:**

```ini
[Desktop Entry]
Name=Play Home
Exec=/home/YOUR_USER/.local/bin/play-home.sh
Icon=moonlight
Type=Application
Categories=Game;
Terminal=false
```

Open your application launcher, search for **"Play Home"**, and run it. The system will now handle VPN routing, ping verification, session management, Moonlight streaming, and cleanup automatically.

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
