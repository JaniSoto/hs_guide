<div class="phase-header">
  <span class="phase-num">1</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 1 · Last time at the physical machine</p>
    <h1>First Boot, SSH & Headless Setup</h1>
    <p class="phase-header-sub">Enable SSH, disable sleep, and lock the machine into headless server mode.</p>
  </div>
</div>

!!! danger "Complete ALL steps before rebooting"
    Do every step in this phase **in order** before you reboot. The headless SDDM override in Step 4 must be in place before your next reboot. If you reboot early, the machine boots back into Gaming Mode — just repeat this phase.

---

## Step 1 — Switch to Desktop Mode

On the machine, in Steam's Gaming Mode:

1. Press the **Steam button** (bottom-left of the Steam UI)
2. Scroll down to **Power**
3. Click **Switch to Desktop**

The machine will switch to KDE Plasma desktop. Open **Konsole** (the terminal app — search for it in the application menu).

---

## Step 2 — Enable SSH

In Konsole, run:

```bash
# Enable and start the SSH service
sudo systemctl enable --now sshd
```

Then find your server's local IP address:

```bash
ip route get 1.1.1.1
```

Look for `src` in the output — that IP is your server's local address. It will look like `192.168.1.50` or `192.168.178.20`.

!!! tip "Write this IP address down"
    You'll use this IP for every SSH connection in this guide. It may change after a router restart — that's why Phase 2 assigns a permanent static IP reservation.

---

## Step 3 — Disable Auto-Suspend

Bazzite may suspend the machine after a period of inactivity, silently killing all Docker containers. Disable it with **both** methods:

**Method A — KDE System Settings:**

1. Open **System Settings** → **Power Management** → **Energy Saving**
2. Under **On AC Power**, set "Suspend session" to **Never**
3. Click **Apply**

**Method B — OS Level (the authoritative method):**

```bash
sudo systemctl mask sleep.target suspend.target \
  hibernate.target hybrid-sleep.target suspend-then-hibernate.target
```

This permanently prevents the system from sleeping, even if some application tries to trigger it.

---

## Step 4 — Apply the Headless SDDM Override

This is the most important step. SDDM is the login screen manager. By default on Bazzite, it auto-logs into Gaming Mode. We want it to start and sit idle — showing a blank login screen — so Docker containers run without any graphical session active.

```bash
sudo mkdir -p /etc/sddm.conf.d
sudo tee /etc/sddm.conf.d/zzz-headless-server.conf > /dev/null << 'EOF'
[Autologin]
User=
Session=
Relogin=false
EOF
```

Apply it immediately:

```bash
sudo systemctl restart sddm
```

!!! info "What does this file do?"
    SDDM reads all `.conf` files in `/etc/sddm.conf.d/` in alphabetical order. The `zzz-` prefix ensures our file loads **last**, overriding any Bazzite defaults (which use names like `bazzite-autologin.conf`). Setting `User=` and `Session=` to blank disables autologin entirely.

---

## Step 5 — Verify After a Reboot

```bash
sudo systemctl reboot
```

Your SSH session will disconnect. Wait about 60 seconds, then reconnect **from your personal computer**:

```bash
ssh username@192.168.1.50
```

Once connected, check that no graphical session started:

```bash
loginctl list-sessions
```

**Expected result:** Empty output, or only your SSH session listed with no seat (`-`) in the seat column. If you see a session with `seat0`, the headless override didn't apply — re-run Step 4.

---

## ✅ Phase 1 Complete

You can now SSH into your server and the screen shows a blank SDDM login. **You no longer need a keyboard or monitor plugged into the server.** All remaining steps in this guide are done over SSH from your personal computer.

[Next: Phase 2 — Router & Network →](phase-2-network.md){ .next-phase }
