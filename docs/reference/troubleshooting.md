# Troubleshooting

Solutions for the most common problems encountered during and after setup.

---

## Machine boots to Gaming Mode after install

**Symptom:** After Phase 0 installation, the machine boots straight into Steam's Gaming Mode. SSH isn't available yet.

**Fix:** This is the expected first-boot state for `bazzite-deck`. You need to complete Phase 1:

1. In Steam's Gaming Mode, press the Power icon → **Switch to Desktop**
2. Open Konsole and complete all steps in [Phase 1](../phases/phase-1-first-boot.md)

---

## `start-kde.sh` / `start-gaming.sh` says "session already active" but nothing is visible

**Symptom:** Running a start script returns `ERROR: A graphical session is already active`, but no session is visible on the screen.

**Cause:** A session is stuck in the `loginctl` session list — often left over from an interrupted stop.

**Fix:**

```bash
loginctl list-sessions          # Note the stuck session ID
loginctl terminate-session ID   # Replace ID with the number from above
sleep 2
sudo start-kde.sh               # Or start-gaming.sh
```

---

## SDDM shows the login screen instead of autologging in

**Symptom:** After running `sudo start-kde.sh`, the physical screen shows the SDDM login screen instead of autologging into KDE.

**Diagnose:**

```bash
# Check that both session desktop files exist
ls /usr/share/wayland-sessions/
# Expected: plasma.desktop AND gamescope-session.desktop (or gamescope-session-steam.desktop)

# Check SDDM logs
journalctl -u sddm -n 50 --no-pager
```

**Common causes:**

- Session desktop file name doesn't match — check the exact filename with `ls` and update the `SESSION_NAME` variable in `start-kde.sh`
- A Bazzite update moved the autologin conf — check `/etc/sddm.conf.d/` for conflicting files

---

## Sunshine not streaming / Moonlight can't connect

Work through this checklist in order:

**1. Check WireGuard is connected:**

```bash
ping 10.8.0.1   # Run on your client device with VPN active
```

If this fails, the VPN isn't connected or the server's wg-easy container isn't running.

**2. Check a session is actually running on the server:**

```bash
loginctl list-sessions   # Must show a session with seat0
```

If empty, no graphical session was launched. Run `sudo start-kde.sh` or `sudo start-gaming.sh`.

**3. Check Sunshine is running:**

```bash
systemctl status sunshine-kms   # Or: systemctl --user status sunshine
```

**4. Check Sunshine's firewall ports are open:**

```bash
sudo firewall-cmd --list-ports
# Expected: 47984/tcp 47989/tcp 48010/tcp 47998/udp 47999/udp 48000/udp 48002/udp 48010/udp
```

**5. Restart Sunshine:**

```bash
sudo systemctl restart sunshine-kms   # or sunshine
```

**6. Check the Sunshine web UI for errors:**

```bash
ssh -N -L 47990:127.0.0.1:47990 -p 22022 username@192.168.1.50
# Open https://localhost:47990 → Logs tab
```

---

## Docker containers not running after a reboot

**Symptom:** After rebooting the server, some or all Docker containers aren't running.

**Diagnose:**

```bash
sudo systemctl status docker
docker ps -a   # Shows all containers including stopped ones
```

**Fix:** Start all services:

```bash
cd ~/docker/duckdns    && docker compose up -d
cd ~/docker/npm        && docker compose up -d
cd ~/docker/nextcloud  && docker compose up -d
cd ~/docker/wireguard  && docker compose up -d
cd ~/docker/watchtower && docker compose up -d
```

If Docker itself isn't starting, check for errors:

```bash
sudo journalctl -u docker -n 50 --no-pager
```

---

## Data drive not mounting — system boots to emergency shell

**Symptom:** Server won't boot normally; drops to an emergency shell asking for root password.

**Cause:** The data drive is missing, failed, or the fstab entry is wrong. The `nofail` option in fstab (set in Phase 6) should prevent this — check if it's present.

**Fix:** At the emergency shell:

```bash
nano /etc/fstab
```

Find the `/srv/nextcloud-data` line. Make sure the options column contains `nofail`. Example:

```
UUID=xxxx  /srv/nextcloud-data  ext4  defaults,nofail  0  2
```

Save, then reboot:

```bash
systemctl reboot
```

---

## Nextcloud shows 502 Bad Gateway

**Symptom:** Visiting `yourname.duckdns.org` shows a 502 error from Nginx Proxy Manager.

**Possible causes and fixes:**

1. **Nextcloud AIO containers aren't running** — Open the AIO panel (`https://localhost:9443` via SSH tunnel) and check container status. Click Start if they're stopped.
2. **NPM proxy host is misconfigured** — Check that the forward hostname is exactly `nextcloud-aio-apache` and port is `11000`
3. **NPM and Nextcloud aren't on the same Docker network** — Both must use the `nextcloud-aio` network:
   ```bash
   docker network inspect nextcloud-aio
   ```

---

## Lost SSH access after Phase 4 (locked out)

**Symptom:** Can't connect with `ssh -p 22022 username@server-ip`.

**Recovery options (requires physical access to the server):**

1. Connect a monitor and keyboard to the server
2. Log in locally and check sshd status: `sudo systemctl status sshd`
3. Check if the SELinux label is on port 22022: `sudo semanage port -l | grep ssh`
4. Check the firewall: `sudo firewall-cmd --list-ports`
5. Check the sshd config for syntax errors: `sudo sshd -t`

If SELinux is blocking: `sudo semanage port -a -t ssh_port_t -p tcp 22022`

If the firewall blocked it: `sudo firewall-cmd --permanent --add-port=22022/tcp && sudo firewall-cmd --reload`
