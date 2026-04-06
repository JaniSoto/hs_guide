


<div class="phase-header">
  <span class="phase-num">1</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 1 · Last time at the physical machine</p>
    <h1>First Boot, SSH & Headless Setup</h1>
    <p class="phase-header-sub">Enable SSH, disable sleep, and lock the machine into a true headless text-mode state.</p>
  </div>
</div>

!!! danger "Complete ALL steps before rebooting"
    Do every step in this phase **in order** before you reboot. The headless boot target in Step 4 must be in place before your next reboot. If you reboot early, the machine boots back into Gaming Mode — just repeat this phase.

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

## Step 4 — Configure True Headless Boot

This is the most important step. By default, Bazzite automatically boots into a graphical interface (Gaming Mode). To save maximum GPU/RAM resources and allow your GPU to deep-sleep, we will force the server to boot into a pure text-only headless state (`multi-user.target`) and ensure Sunshine starts in this mode.

```bash
sudo systemctl set-default multi-user.target

sudo sed -i 's/WantedBy=.*/WantedBy=multi-user.target graphical.target/' /etc/systemd/system/sunshine.service
sudo systemctl daemon-reload
sudo systemctl reenable sunshine.service
```

!!! info "What does this do?"
    It stops the graphical display manager (SDDM) from starting on boot. Your server will act like an enterprise data-center machine, dropping to a raw Linux text console (TTY) while keeping Docker and background services running perfectly.

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

**Expected result:** You should only see your current text sessions (like `tty` or `pts/0`). You should **not** see a `greeter` or any graphical `seat0` sessions. The physical monitor plugged into the server (if any) will just show a black screen with a text login prompt.

---

## ✅ Phase 1 Complete

You can now SSH into your server and the physical machine sits entirely idle at a Linux terminal. **You no longer need a keyboard or monitor plugged into the server.** All remaining steps in this guide are done over SSH from your personal computer.

[Next: Phase 2 — Router & Network →](phase-2-network.md){ .next-phase }
