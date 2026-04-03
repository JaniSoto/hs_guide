# Quick Commands

All commands are run from your **personal computer's terminal** unless marked *(on server)*.

!!! danger "Always restart Docker after `firewall-cmd --reload`"
    firewalld flushes Docker's port-forwarding rules when it reloads. **Always** run `sudo systemctl restart docker` immediately after any `firewall-cmd --reload`. Without it, ports 80, 443, and 51820 go dark.

---

## Remote Gaming Sessions

**Stream KDE Desktop:**

```bash
ssh -p 22022 username@192.168.1.50 'sudo start-kde.sh'
# Then open Moonlight → Add Host → 10.8.0.1 → select Desktop
```

**Stream Steam Gaming Mode:**

```bash
ssh -p 22022 username@192.168.1.50 'sudo start-gaming.sh'
# Then open Moonlight → Add Host → 10.8.0.1 → select Steam Big Picture
```

**Stop the session and return to idle:**

```bash
ssh -p 22022 username@192.168.1.50 'sudo stop-session.sh'
```

!!! warning "Always stop the session when finished"
    Closing Moonlight alone does **not** stop the session on the server. KDE or Gaming Mode keeps running. Always run `stop-session.sh` to return the server to idle headless mode.

**Force-kill a stuck session:**

```bash
loginctl list-sessions               # (on server) — note the session ID
loginctl terminate-session SESSION_ID  # (on server) — replace SESSION_ID
```

---

## SSH Tunnels (for admin panels)

Open these on your personal computer and leave them running in background terminals:

```bash
# NPM admin panel → http://localhost:8081
ssh -N -L 8081:127.0.0.1:81 -p 22022 username@192.168.1.50

# Nextcloud AIO panel → https://localhost:9443
ssh -N -L 9443:127.0.0.1:9443 -p 22022 username@192.168.1.50

# Sunshine web UI → https://localhost:47990
ssh -N -L 47990:127.0.0.1:47990 -p 22022 username@192.168.1.50

# wg-easy admin panel → http://localhost:51821
ssh -N -L 51821:127.0.0.1:51821 -p 22022 username@192.168.1.50
```

---

## Docker Management

**Check all container status:**

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

**Restart a container:**

```bash
cd ~/docker/npm && docker compose restart
cd ~/docker/nextcloud && docker compose restart
```

**View container logs:**

```bash
docker logs nginx-proxy-manager --tail 50
docker logs nextcloud-aio-mastercontainer --tail 50
```

**Bring up all services after a reboot:**

```bash
cd ~/docker/duckdns    && docker compose up -d
cd ~/docker/npm        && docker compose up -d
cd ~/docker/nextcloud  && docker compose up -d
cd ~/docker/wireguard  && docker compose up -d
cd ~/docker/watchtower && docker compose up -d
```

---

## Nextcloud AIO

**Update Nextcloud** — always done through the AIO panel (SSH tunnel required):

```bash
ssh -N -L 9443:127.0.0.1:9443 -p 22022 username@192.168.1.50
# Then open https://localhost:9443 and click Update
```

Never use `docker compose pull` for Nextcloud AIO containers — updates must go through the AIO panel to run database migrations correctly.

---

## OS Updates

```bash
# Check current status
rpm-ostree status

# Roll back to the previous OS version instantly
rpm-ostree rollback && systemctl reboot
```

Bazzite stages OS updates automatically in the background. A reboot applies them. All containers resume automatically after reboot.

---

## Firewall Quick Reference

```bash
# List current rules
sudo firewall-cmd --list-all

# Check if a port is open
sudo firewall-cmd --query-port=22022/tcp

# After ANY reload — restart Docker!
sudo firewall-cmd --reload && sudo systemctl restart docker
```

---

## Fail2ban

```bash
# Check ban status
sudo fail2ban-client status sshd

# Unban an IP
sudo fail2ban-client set sshd unbanip 1.2.3.4
```
