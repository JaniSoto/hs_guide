```markdown
# Phases 10 & 11: Initialization & Backups

## Phase 10: Link & Initialise Nextcloud

**1. Access admin panels via SSH tunnel**
The admin panels are bound to `127.0.0.1` (server's localhost) — invisible to your LAN. Create SSH tunnels from your personal computer and leave them running in separate terminal windows.

```bash
# NPM admin panel — available at http://localhost:8081
ssh -N -L 8081:127.0.0.1:81 -p 22022 username@192.168.1.50

# Nextcloud AIO setup panel — now symmetrical (9443 to 9443)
ssh -N -L 9443:127.0.0.1:9443 -p 22022 username@192.168.1.50
```

**2. Configure Nginx Proxy Manager**
Open `http://localhost:8081`. Log in with default credentials (`admin@example.com` / `changeme`) and change them immediately.

Go to **Hosts → Proxy Hosts → Add Proxy Host**:

!!! danger "Critical Save Order"
    Do NOT click Save until you have configured the Details, SSL, and Advanced tabs in the same session. If you save early, NPM silently resets the SSL tab.

*   **Details Tab:**
    *   Domain Names: `yourname.duckdns.org`
    *   Scheme: `http`
    *   Forward Hostname / IP: `nextcloud-aio-apache`
    *   Forward Port: `11000`
    *   Cache Assets: **OFF**
    *   Block Common Exploits & Websockets Support: **ON**
*   **SSL Tab:** Request new certificate · Force SSL · HTTP/2 · Agree to ToS
*   **Advanced Tab:** Paste the following:
    ```nginx
    client_max_body_size 0;
    proxy_read_timeout 36000s;
    proxy_send_timeout 36000s;
    proxy_request_buffering off;
    proxy_hide_header Upgrade;
    ```
Click **Save**. *(Note: A 502 Bad Gateway error when visiting your domain right now is normal, as the Nextcloud container hasn't been started yet).*

**3. Fix local network access (NAT loopback)**
If you access your domain from inside your home and your router doesn't support loopback, try these in order:
1.  **Router Local DNS:** Add `yourname.duckdns.org → 192.168.1.50` in your router's admin panel.
2.  **Pi-hole:** Add a Local DNS Record to Pi-hole.
3.  **Docker Bypass (Last Resort):** Uncomment `SKIP_DOMAIN_VALIDATION=true` in your Nextcloud `docker-compose.yml` and restart the container.

**4. Initialise Nextcloud AIO**
Open `https://localhost:9443` (accept the self-signed cert warning). Enter the setup password provided and your domain. Select the latest Nextcloud Hub, and optionally add Fulltextsearch or Imaginary. Click **Start Containers**.

---

## Phase 11: Testing & Backups

**1. System State Validation Checklist**
Run these on the server over SSH.

*Group A — OS & Security Layer*
```bash
ls /etc/sddm.conf.d/    # Expected: zzz-headless-server.conf only.
ss -tlnp | grep sshd    # Expected: exactly one line showing *:22022.
sudo semanage port -l | grep ssh # Expected: ssh_port_t tcp 22022, 22
```

*Group B — Storage & Docker*
```bash
ls -Zd /srv/nextcloud-data
# Expected: system_u:object_r:container_file_t:s0 /srv/nextcloud-data
sudo firewall-cmd --list-all
# Expected: ports: 22022/tcp plus Sunshine ports. 80/443/51820 managed by Docker — absent is correct.
```

**2. Backups (Borg)**
!!! tip "Why Borg?"
    Borg is free, open-source backup software integrated directly into the AIO panel. It creates compressed, deduplicated snapshots of your live SSD data onto a second physical drive.

Inside the AIO setup panel (`https://localhost:9443` via SSH tunnel): find the **Borg Backup** section. Set a destination path on your second drive (e.g., `/mnt/backup-drive`) and schedule daily backups.
```
