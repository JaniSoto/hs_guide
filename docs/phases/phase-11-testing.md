<div class="phase-header">
  <span class="phase-num">11</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 11 · Run over SSH</p>
    <h1>Testing & Backups</h1>
    <p class="phase-header-sub">Validate the complete setup with a checklist and configure automated backups.</p>
  </div>
</div>

---

## System State Validation Checklist

Run each command over SSH and compare the output to the expected result. Every item should pass before you consider setup complete.

### Group A — OS & Headless Config

```bash
ls /etc/sddm.conf.d/
```
**Expected:** `zzz-headless-server.conf` only (no `bazzite-autologin.conf`)

```bash
loginctl list-sessions
```
**Expected:** `greeter` on `seat0` (unless start-kde or start-gaming were executed) 

```bash
ss -tlnp | grep sshd
```
**Expected:** Exactly one line showing `*:22022`

```bash
sudo semanage port -l | grep ssh
```
**Expected:** `ssh_port_t   tcp   22022, 22`

---

### Group B — Storage & Docker

```bash
ls -Zd /srv/nextcloud-data
```
**Expected:** `system_u:object_r:container_file_t:s0 /srv/nextcloud-data`

```bash
df -h /srv/nextcloud-data
```
**Expected:** Shows your data drive's capacity (not the OS drive's capacity)

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```
**Expected:** `duckdns`, `nginx-proxy-manager`, `nextcloud-aio-mastercontainer`, `wg-easy` all showing `Up`

---

### Group C — Firewall

```bash
sudo firewall-cmd --list-all
```
**Expected output should include:**

```
ports: 22022/tcp 47984/tcp 47989/tcp 48010/tcp 47998/udp 47999/udp 48000/udp 48002/udp 48010/udp
```

!!! note "Ports 80, 443, and 51820 are absent — this is correct"
    Docker manages these ports directly via iptables rules, bypassing firewalld's interface. Absent from `firewall-cmd --list-all` does not mean they're blocked.

---

### Group D — Streaming & VPN

1. Connect WireGuard on your device
2. Confirm `ping <your-ip-address>` succeeds
3. Run `sudo start-kde.sh` over SSH, then open Moonlight and connect to `<your-ip-address>`
4. Confirm a stream appears, then run `sudo stop-session.sh`

---

## Setting Up Backups (Borg)

Borg Backup is built directly into Nextcloud AIO. It creates compressed, deduplicated snapshots of your live Nextcloud data onto a second physical drive — so a drive failure can't take your data with it.

??? info "Why Borg?"
    Borg is free, open-source, and produces highly efficient incremental backups. Each backup snapshot only stores the data that changed since the last one, so daily backups use very little extra space.

**Access the AIO panel** via SSH tunnel:

```bash
ssh -N -L 9443:127.0.0.1:9443 -p 22022 username@192.168.1.50
```

Open `https://localhost:9443`. In the AIO admin panel, find the **Borg Backup** section:

1. Set the backup destination path (e.g. `/mnt/backup-drive` — mount your second drive there first)
2. Set a backup schedule — daily at a quiet time like 3 AM is typical
3. Set a backup retention policy (e.g. keep 7 daily, 4 weekly, 3 monthly)
4. Click **Save** and run a first manual backup to verify it works

---

## ✅ Phase 11 Complete

Your server passed all validation checks and automated backups are configured. The core server setup is complete.

Phase 12 is optional — it adds a webhook server and client launcher script for one-click remote gaming sessions without any SSH commands.

[Next: Phase 12 — One-Click Client (Optional) →](phase-12-webhooks.md){ .next-phase }
