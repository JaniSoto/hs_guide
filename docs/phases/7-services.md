# Phase 7: Deploying Services

!!! concept "What is a Docker Compose file?"
    `docker-compose.yml` is a YAML configuration file that describes one or more containers as a single unit. YAML is indentation-sensitive — 2 spaces per level. Never use tabs; always use spaces.

## 1. DuckDNS Auto-Updater

!!! warning "Clock Check"
    Check your clock before proceeding by running `timedatectl | grep "Time zone"` and `timedatectl | grep "RTC in local TZ"`.
    *Expected:* `RTC in local TZ: no`. If it says "yes", run: `sudo timedatectl set-local-rtc 0`

Create the directory and environment file:

```bash
mkdir -p ~/docker/duckdns && cd ~/docker/duckdns
TZ_VALUE=$(timedatectl show -p Timezone --value)
nano .env
```

Paste this into `.env` (replace with your actual info):
```bash
DUCKDNS_SUBDOMAINS=yourname
DUCKDNS_TOKEN=your-token-here
```
Press `Ctrl+O`, `Enter`, `Ctrl+X` to save and exit. Then create the compose file:

```bash
chmod 600 .env
cat > docker-compose.yml << EOF
services:
  duckdns:
    image: lscr.io/linuxserver/duckdns:latest
    container_name: duckdns
    restart: unless-stopped
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    env_file:
      - .env
    environment:
      - SUBDOMAINS=\${DUCKDNS_SUBDOMAINS}
      - TOKEN=\${DUCKDNS_TOKEN}
      - TZ=${TZ_VALUE}
EOF
docker compose up -d
```

## 2. Nginx Proxy Manager (NPM)

!!! concept "What is a Reverse Proxy?"
    A reverse proxy sits in front of your services and routes incoming requests. When a visitor connects to `yourname.duckdns.org`, their request arrives at NPM (on port 443). NPM looks at the domain name, finds the matching proxy host rule, and forwards the request internally to the correct container.

```bash
mkdir -p ~/docker/npm && cd ~/docker/npm
mkdir -p data letsencrypt
sudo chown -R $USER:$USER ./data ./letsencrypt

cat > docker-compose.yml << 'EOF'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    ports:
      - '80:80' # HTTP and Let's Encrypt challenges.
      - '443:443' # HTTPS.
      - '127.0.0.1:81:81' # Admin panel — localhost only.
    networks:
      - nextcloud-aio
    volumes:
      - ./data:/data:z
      - ./letsencrypt:/etc/letsencrypt:z
networks:
  nextcloud-aio:
    external: true
EOF

docker compose up -d
```

## 3. Nextcloud All-in-One

!!! info "Port 9443 Update (Fixes Decky Loader Conflict)"
    Steam's Gaming Mode uses a Chromium-based UI (CEF) that opens a remote debugging port on `localhost:8080`. Nextcloud AIO also defaults to `8080`. If Nextcloud takes it, plugins like **Decky Loader** will fail to load in Gaming Mode. To fix this, we bind Nextcloud AIO to `9443` below.

!!! warning "Hardware Acceleration Note"
    **AMD GPU hardware acceleration:** If you want VA-API GPU acceleration for Nextcloud video transcoding, do NOT run `docker compose up -d` below yet. See Appendix C first.

```bash
mkdir -p ~/docker/nextcloud && cd ~/docker/nextcloud

cat > docker-compose.yml << 'EOF'
volumes:
  nextcloud_aio_mastercontainer:
    name: nextcloud_aio_mastercontainer

services:
  nextcloud-aio-mastercontainer:
    image: ghcr.io/nextcloud-releases/all-in-one:latest
    init: true
    restart: always
    container_name: nextcloud-aio-mastercontainer
    security_opt:
      - label:disable # REQUIRED: allows access to the Docker socket under SELinux.
    labels:
      - "com.centurylinklabs.watchtower.enable=false" # Never auto-update AIO.
    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - '127.0.0.1:9443:8080' # Changed from 8080 to 9443 to avoid conflict with Steam CEF.
    networks:
      - nextcloud-aio
    environment:
      - APACHE_PORT=11000
      - APACHE_IP_BINDING=127.0.0.1
      - NEXTCLOUD_DATADIR=/srv/nextcloud-data
networks:
  nextcloud-aio:
    external: true
EOF

docker compose up -d
```
