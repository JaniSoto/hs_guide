# OS Updates & Rollbacks

How to update Bazzite, roll back if something breaks, and keep Docker containers current.

---

## Bazzite OS Updates

Bazzite downloads OS updates automatically in the background. Updates are staged — they don't apply until you reboot. After rebooting, all Docker containers restart automatically (they all have `restart: unless-stopped` or `restart: always`).

**Check pending update status:**

```bash
rpm-ostree status
```

**Reboot to apply a staged update:**

```bash
sudo systemctl reboot
```

---

## Rolling Back an OS Update

If an update causes problems, you can instantly roll back to the previous OS image. Bazzite always keeps the previous deployment available:

```bash
rpm-ostree rollback && sudo systemctl reboot
```

This takes effect on the next boot. Your layered packages (Docker, Fail2ban, etc.) are preserved across rollbacks.

**List all available stable builds** (useful if you want to pin a specific version):

```bash
skopeo list-tags docker://ghcr.io/ublue-os/bazzite-deck | grep -- 'stable-' | sort -rV
```

---

## Watchtower — Auto-Updating Containers

Watchtower watches your Docker containers for new image versions and updates them automatically. It's configured to only update containers that have the `com.centurylinklabs.watchtower.enable=true` label — so Nextcloud AIO is never auto-updated (it requires the manual process below).

**Deploy Watchtower** (if not already done):

```bash
mkdir -p ~/docker/watchtower && cd ~/docker/watchtower
TZ_VALUE=$(timedatectl show -p Timezone --value)

cat > docker-compose.yml << EOF
services:
  watchtower:
    image: nickfedor/watchtower
    restart: unless-stopped
    security_opt:
      - label:disable
    environment:
      - TZ=${TZ_VALUE}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: --label-enable --cleanup --schedule "0 0 4 * * *"
EOF

docker compose up -d
```

This runs at 4 AM daily and removes old image versions after updating (`--cleanup`).

!!! info "Why `nickfedor/watchtower`?"
    The official `containrrr/watchtower` image occasionally has issues with the SELinux `label:disable` flag on Fedora-based systems. The `nickfedor/watchtower` fork resolves this reliably.

---

## Updating Nextcloud AIO Manually

!!! danger "Never use `docker compose pull` for Nextcloud AIO"
    Updating Nextcloud by pulling a new container image directly skips database migrations, which can corrupt your data. Always update through the AIO admin panel.

**Steps:**

1. Open an SSH tunnel: `ssh -N -L 9443:127.0.0.1:9443 -p 22022 username@192.168.1.50`
2. Open `https://localhost:9443` in your browser
3. In the AIO admin panel, look for the **Update Available** button
4. Click it and wait for all containers to update and return to "healthy" status

The AIO panel handles the update sequence, database migrations, and health checks automatically.

---

## Updating Layered Packages (Docker, Fail2ban)

Because these are installed via `rpm-ostree`, they update automatically as part of the OS update:

```bash
rpm-ostree upgrade && sudo systemctl reboot
```

This updates both the base OS image and any layered packages together.
