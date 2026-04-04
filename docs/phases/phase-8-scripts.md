<div class="phase-header">
  <span class="phase-num">8</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 8 · Run over SSH</p>
    <h1>Session Management Scripts</h1>
    <p class="phase-header-sub">Write the three scripts that launch and stop KDE, Gaming Mode, and Sunshine.</p>
  </div>
</div>

??? info "How these scripts work"
    For Sunshine to capture the physical display, it needs exclusive access to the GPU's DRM/KMS (kernel display subsystem). Only one process can hold the "DRM master" lock at a time. These scripts handle the full transition:

    1. **Check** no session is already active
    2. **Stop** SDDM and Sunshine to release the DRM lock
    3. **Write** an SDDM autologin config for the target session
    4. **Start** SDDM so it auto-logs into the chosen session
    5. **Wait** for the session to appear and stabilise
    6. **Start** Sunshine to capture the session via KMS

---

## 1 — Create the Scripts

Create all three of the scripts below to manage your server's graphical sessions. Click each tab to view the instructions and code.

=== "start-kde.sh"
    Launches KDE Plasma on the physical display, then starts Sunshine to capture it.

    ```bash
    sudo nano /usr/local/bin/start-kde.sh
    ```

    Paste the entire script below:

    ```bash
    #!/usr/bin/env bash
    set -euo pipefail

    readonly USER_UID=1000
    readonly AUTOLOGIN_CONF="/etc/sddm.conf.d/zzzz-bazzite-htpc.conf"
    readonly SESSION_NAME="plasma.desktop"
    readonly SESSION_WAIT_TIMEOUT=90
    readonly SUNSHINE_SETTLE_SECS=6

    log()  { printf '[start-kde] %s\n'         "$*"; }
    warn() { printf '[start-kde] WARNING: %s\n' "$*" >&2; }
    die()  { printf '[start-kde] ERROR: %s\n'   "$*" >&2; exit 1; }

    detect_sunshine() {
        for name in sunshine-kms sunshine sunshine-headless; do
            if systemctl is-active --quiet "${name}" 2>/dev/null || systemctl is-enabled --quiet "${name}" 2>/dev/null; then
                echo "${name}"; return 0
            fi
        done
        systemctl list-units --type=service --no-legend 2>/dev/null | awk '$1 ~ /sunshine/ { print $1; exit }' | sed 's/\.service$//'
    }

    active_graphical_sessions() {
        loginctl list-sessions --no-legend | awk -v uid="${USER_UID}" '$2 == uid && $4 != "-" { print $1 }'
    }

    [[ "${EUID}" -eq 0 ]] || die "Must be run as root (sudo)."
    USERNAME="$(id -nu "${USER_UID}")" || die "Cannot resolve username."
    SUNSHINE_SERVICE="$(detect_sunshine)" || die "Cannot find Sunshine service."

    log "User: ${USERNAME} | Session: ${SESSION_NAME} | Sunshine: ${SUNSHINE_SERVICE}"

    if [[ -n "$(active_graphical_sessions)" ]]; then
        die "A graphical session is already active. Run 'sudo stop-session.sh' first."
    fi

    rm -f /tmp/chimeraos-short-session-tracker 2>/dev/null || true

    log "Writing SDDM autologin configuration..."
    mkdir -p "$(dirname "${AUTOLOGIN_CONF}")"
    cat > "${AUTOLOGIN_CONF}" <<EOF
    [Autologin]
    User=${USERNAME}
    Session=${SESSION_NAME}
    Relogin=true
    EOF

    log "Stopping SDDM and Sunshine to release DRM/KMS locks..."
    systemctl stop sddm 2>/dev/null || true
    systemctl stop "${SUNSHINE_SERVICE}" 2>/dev/null || true
    sleep 3

    log "Starting SDDM to trigger KDE autologin..."
    systemctl start sddm

    log "Waiting for KDE session to become active (timeout: ${SESSION_WAIT_TIMEOUT}s)..."
    elapsed=0; confirmed=false
    while (( elapsed < SESSION_WAIT_TIMEOUT )); do
        session_id="$(active_graphical_sessions | head -n 1)"
        if [[ -n "${session_id}" ]]; then
            state="$(loginctl show-session "${session_id}" -p State --value 2>/dev/null || true)"
            if [[ "${state}" == "active" ]]; then
                log "Session ${session_id} is active."
                confirmed=true; break
            fi
        fi
        sleep 2; (( elapsed += 2 ))
    done

    "${confirmed}" || warn "Session not confirmed after ${SESSION_WAIT_TIMEOUT}s. Proceeding anyway."

    log "Letting KWin compositor initialise for ${SUNSHINE_SETTLE_SECS}s..."
    sleep "${SUNSHINE_SETTLE_SECS}"

    log "Starting Sunshine to capture KDE via KMS..."
    systemctl reset-failed "${SUNSHINE_SERVICE}" 2>/dev/null || true
    systemctl start "${SUNSHINE_SERVICE}"

    echo -e "\n✓ KDE Plasma session started. Reconnect Moonlight now."
    ```

    Make it executable:

    ```bash
    sudo chmod +x /usr/local/bin/start-kde.sh
    ```

=== "start-gaming.sh"
    Launches Steam Gaming Mode (gamescope), then starts Sunshine to capture it.

    ```bash
    sudo nano /usr/local/bin/start-gaming.sh
    ```

    Paste the entire script below:

    ```bash
    #!/usr/bin/env bash
    set -euo pipefail

    readonly USER_UID=1000
    readonly AUTOLOGIN_CONF="/etc/sddm.conf.d/zzzz-bazzite-htpc.conf"
    readonly SESSION_NAME="gamescope-session.desktop"
    readonly SESSION_WAIT_TIMEOUT=180
    readonly SUNSHINE_SETTLE_SECS=15

    log()  { printf '[start-gaming] %s\n'         "$*"; }
    warn() { printf '[start-gaming] WARNING: %s\n' "$*" >&2; }
    die()  { printf '[start-gaming] ERROR: %s\n'   "$*" >&2; exit 1; }

    detect_sunshine() {
        for name in sunshine-kms sunshine sunshine-headless; do
            if systemctl is-active --quiet "${name}" 2>/dev/null || systemctl is-enabled --quiet "${name}" 2>/dev/null; then
                echo "${name}"; return 0
            fi
        done
        systemctl list-units --type=service --no-legend 2>/dev/null | awk '$1 ~ /sunshine/ { print $1; exit }' | sed 's/\.service$//'
    }

    active_graphical_sessions() {
        loginctl list-sessions --no-legend | awk -v uid="${USER_UID}" '$2 == uid && $4 != "-" { print $1 }'
    }

    [[ "${EUID}" -eq 0 ]] || die "Must be run as root (sudo)."
    USERNAME="$(id -nu "${USER_UID}")" || die "Cannot resolve username."
    SUNSHINE_SERVICE="$(detect_sunshine)" || die "Cannot find Sunshine service."

    log "User: ${USERNAME} | Session: ${SESSION_NAME} | Sunshine: ${SUNSHINE_SERVICE}"

    if [[ -n "$(active_graphical_sessions)" ]]; then
        die "A graphical session is already active. Run 'sudo stop-session.sh' first."
    fi

    rm -f /tmp/chimeraos-short-session-tracker 2>/dev/null || true

    log "Disabling any conflicting user-level Sunshine hooks..."
    sudo -Eu "${USERNAME}" \
        XDG_RUNTIME_DIR="/run/user/${USER_UID}" \
        DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/${USER_UID}/bus" \
        systemctl --user disable --now sunshine.service 2>/dev/null || true

    log "Writing SDDM autologin configuration..."
    mkdir -p "$(dirname "${AUTOLOGIN_CONF}")"
    cat > "${AUTOLOGIN_CONF}" <<EOF
    [Autologin]
    User=${USERNAME}
    Session=${SESSION_NAME}
    Relogin=true
    EOF

    log "Stopping SDDM and Sunshine to release DRM/KMS locks..."
    systemctl stop sddm 2>/dev/null || true
    systemctl stop "${SUNSHINE_SERVICE}" 2>/dev/null || true
    sleep 3

    log "Starting SDDM to trigger Gaming Mode autologin..."
    systemctl start sddm

    log "Waiting for Gamescope/Steam session to appear (timeout: ${SESSION_WAIT_TIMEOUT}s)..."
    elapsed=0; confirmed=false
    while (( elapsed < SESSION_WAIT_TIMEOUT )); do
        session_id="$(active_graphical_sessions | head -n 1)"
        if [[ -n "${session_id}" ]]; then
            log "Gaming Mode session ${session_id} detected."
            confirmed=true; break
        fi
        sleep 3; (( elapsed += 3 ))
    done

    "${confirmed}" || warn "Session did not appear after ${SESSION_WAIT_TIMEOUT}s. Proceeding anyway."

    log "Giving Steam and Gamescope ${SUNSHINE_SETTLE_SECS}s to finish loading..."
    sleep "${SUNSHINE_SETTLE_SECS}"

    log "Starting Sunshine to capture Gaming Mode via KMS..."
    systemctl reset-failed "${SUNSHINE_SERVICE}" 2>/dev/null || true
    systemctl start "${SUNSHINE_SERVICE}"

    echo -e "\n✓ Gaming Mode session started. Reconnect Moonlight now."
    ```

    Make it executable:

    ```bash
    sudo chmod +x /usr/local/bin/start-gaming.sh
    ```

=== "stop-session.sh"
    Gracefully stops whichever session is active and returns the server to its headless idle state.

    ```bash
    sudo nano /usr/local/bin/stop-session.sh
    ```

    Paste the entire script below:

    ```bash
    #!/usr/bin/env bash
    set -euo pipefail

    readonly USER_UID=1000
    readonly AUTOLOGIN_CONF="/etc/sddm.conf.d/zzzz-bazzite-htpc.conf"
    readonly SESSION_STOP_TIMEOUT=45
    readonly SUNSHINE_SETTLE_SECS=4

    log()  { printf '[stop-session] %s\n'         "$*"; }
    warn() { printf '[stop-session] WARNING: %s\n' "$*" >&2; }
    die()  { printf '[stop-session] ERROR: %s\n'   "$*" >&2; exit 1; }

    detect_sunshine() {
        for name in sunshine-kms sunshine sunshine-headless; do
            if systemctl is-active --quiet "${name}" 2>/dev/null || systemctl is-enabled --quiet "${name}" 2>/dev/null; then
                echo "${name}"; return 0
            fi
        done
        systemctl list-units --type=service --no-legend 2>/dev/null | awk '$1 ~ /sunshine/ { print $1; exit }' | sed 's/\.service$//'
    }

    active_graphical_sessions() {
        loginctl list-sessions --no-legend | awk -v uid="${USER_UID}" '$2 == uid && $4 != "-" { print $1 }'
    }

    [[ "${EUID}" -eq 0 ]] || die "Must be run as root (sudo)."
    USERNAME="$(id -nu "${USER_UID}")" || die "Cannot resolve username."
    SUNSHINE_SERVICE="$(detect_sunshine)" || die "Cannot find Sunshine service."

    session_id="$(active_graphical_sessions | head -n 1)"
    [[ -n "${session_id}" ]] || die "No active graphical session found."

    log "Active graphical session: ${session_id}"
    rm -f /tmp/chimeraos-short-session-tracker 2>/dev/null || true

    log "Disabling SDDM autologin to enforce return to greeter..."
    mkdir -p "$(dirname "${AUTOLOGIN_CONF}")"
    cat > "${AUTOLOGIN_CONF}" <<EOF
    [Autologin]
    User=
    Session=
    Relogin=false
    EOF

    log "Stopping Sunshine to release DRM/KMS locks..."
    systemctl stop "${SUNSHINE_SERVICE}" 2>/dev/null || true

    desktop="$(loginctl show-session "${session_id}" -p Desktop --value 2>/dev/null || true)"
    is_gaming=false
    if [[ "${desktop,,}" == *"gamescope"* ]] || pgrep -u "${USER_UID}" -x gamescope >/dev/null; then
        is_gaming=true
    fi

    if "${is_gaming}"; then
        log "Session type: Gamescope/Steam Mode"
        sudo -Eu "${USERNAME}" \
            XDG_RUNTIME_DIR="/run/user/${USER_UID}" \
            DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/${USER_UID}/bus" \
            systemctl --user stop gamescope-session-plus@steam.service || true
    else
        log "Session type: KDE Plasma"
        sudo -Eu "${USERNAME}" \
            XDG_RUNTIME_DIR="/run/user/${USER_UID}" \
            DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/${USER_UID}/bus" \
            qdbus org.kde.Shutdown /Shutdown org.kde.Shutdown.logout || true
    fi

    log "Waiting for session to close (timeout: ${SESSION_STOP_TIMEOUT}s)..."
    elapsed=0; closed=false
    while (( elapsed < SESSION_STOP_TIMEOUT )); do
        if [[ -z "$(active_graphical_sessions)" ]]; then
            log "Session closed."; closed=true; break
        fi
        sleep 2; (( elapsed += 2 ))
    done

    if ! "${closed}"; then
        warn "Session stuck. Forcing termination..."
        loginctl terminate-session "${session_id}" 2>/dev/null || true
        sleep 3
    fi

    log "Waiting for SDDM greeter..."
    elapsed=0; greeter_ready=false
    while (( elapsed < 20 )); do
        if loginctl list-sessions --no-legend | awk '$6 == "greeter" { print $1 }' | grep -q .; then
            greeter_ready=true; break
        fi
        sleep 1; (( elapsed++ ))
    done

    "${greeter_ready}" || warn "Greeter did not appear cleanly."

    log "Letting greeter render for ${SUNSHINE_SETTLE_SECS}s..."
    sleep "${SUNSHINE_SETTLE_SECS}"

    log "Starting Sunshine to capture SDDM Greeter via KMS..."
    systemctl reset-failed "${SUNSHINE_SERVICE}" 2>/dev/null || true
    systemctl start "${SUNSHINE_SERVICE}"

    echo -e "\n✓ Session ended. Server is idle at the SDDM greeter. Reconnect Moonlight to view."
    ```

    Make it executable:

    ```bash
    sudo chmod +x /usr/local/bin/stop-session.sh
    ```

---

## 2 — Configure Passwordless Sudo

To use these scripts seamlessly (especially via the webhook in Phase 12), allow your user to run them without a password:

```bash
printf "%s ALL=(ALL) NOPASSWD: /usr/local/bin/start-kde.sh\n"    "$USER" \
  | sudo tee    /etc/sudoers.d/session-scripts > /dev/null
printf "%s ALL=(ALL) NOPASSWD: /usr/local/bin/start-gaming.sh\n" "$USER" \
  | sudo tee -a /etc/sudoers.d/session-scripts > /dev/null
printf "%s ALL=(ALL) NOPASSWD: /usr/local/bin/stop-session.sh\n" "$USER" \
  | sudo tee -a /etc/sudoers.d/session-scripts > /dev/null

sudo chmod 440 /etc/sudoers.d/session-scripts
sudo visudo -c -f /etc/sudoers.d/session-scripts
```

The last command should output: `parsed OK`. If it reports an error, do not proceed — edit the file with `sudo visudo -f /etc/sudoers.d/session-scripts` to fix it.

---

## ✅ Phase 8 Complete

Three session management scripts are installed and sudoers is configured. You can already test them:

```bash
sudo start-kde.sh      # Launches KDE Plasma
sudo stop-session.sh   # Returns to idle
sudo start-gaming.sh   # Launches Steam Gaming Mode
```
!!! note "Streaming won't work yet"
    Sunshine still needs to be configured in Phase 9.

[Next: Phase 9 — VPN & Streaming →](phase-9-streaming.md){ .next-phase }
