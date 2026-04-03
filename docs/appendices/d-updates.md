```markdown
# Appendix D: OS Updates & Containers

### OS Updates & Rollbacks
Bazzite stages OS updates silently in the background. Applying them requires a reboot. All containers come back up automatically.

```bash
# Roll back to the previous OS image instantly:
rpm-ostree rollback && systemctl reboot

# List all available stable builds:
skopeo list-tags docker://ghcr.io/ublue-os/bazzite-deck | grep -- 'stable-' | sort -rV
```

### Auto-update Containers (Watchtower)
Watchtower watches your containers for new image versions and updates them automatically.

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

### Nextcloud AIO — Manual Update Procedure
Nextcloud AIO child containers must be updated through the AIO admin panel (`https://localhost:9443` over SSH tunnel) to ensure database migrations happen correctly.
```
