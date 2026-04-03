<div class="phase-header">
  <span class="phase-num">7</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 7 · Run over SSH</p>
    <h1>Deploying Core Services</h1>
    <p class="phase-header-sub">Deploy DuckDNS, Nginx Proxy Manager, and Nextcloud AIO using Docker Compose.</p>
  </div>
</div>

??? tip "What is a `docker-compose.yml` file?"
    A `docker-compose.yml` file describes one or more containers as a single unit. It defines the image to use, environment variables, ports to expose, storage volumes, and which network to join. YAML files are **indentation-sensitive** — always use 2 spaces per level, never tabs.

---

## 1 — DuckDNS Auto-Updater

DuckDNS gives you a free domain name (`yourname.duckdns.org`) that always points to your home's current public IP address, even when your ISP changes it. The DuckDNS container updates this automatically every 5 minutes.

**Check your system clock first** (important for Let's Encrypt later):

```bash
timedatectl | grep "RTC in local TZ"
```

If the result says `yes`, fix it:

```bash
sudo timedatectl set-local-rtc 0
```

**Create the configuration:**

```bash
mkdir -p ~/docker/duckdns && cd ~/docker/duckdns
```

Create a `.env` file to store your credentials (replace the values with your own):

```bash
nano .env
```

Paste into the file:

```
DUCKDNS_SUBDOMAINS=yourname
DUCKDNS_TOKEN=your-token-here
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`. Then secure the file:

```bash
chmod 600 .env
```

Create the compose file:

```bash
TZ_VALUE=$(timedatectl show -p Timezone --value)

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
```

Start it:

```bash
docker compose up -d
```

---

## 2 — Nginx Proxy Manager (NPM)

NPM is your reverse proxy — it routes incoming HTTPS traffic to the right container and handles Let's Encrypt SSL certificates automatically.

??? tip "What is a Reverse Proxy?"
    You have one public IP address but potentially many services. NPM sits at the front door: when someone connects to `yourname.duckdns.org`, NPM checks the domain name, finds the matching rule, and forwards the request internally to the right container — in this case, Nextcloud. The outside world only ever sees port 80 and 443.

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
      - '80:80'            # HTTP and Let's Encrypt challenges
      - '443:443'          # HTTPS
      - '127.0.0.1:81:81'  # Admin panel — localhost only
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

!!! info "The admin panel port 81 is only on localhost"
    Notice `127.0.0.1:81:81` — the `127.0.0.1:` prefix binds the port to localhost only. You can't reach it directly from your network or the internet. You'll access it via an SSH tunnel in Phase 10.

---

## 3 — Nextcloud All-in-One (AIO)

Nextcloud AIO is a "master container" that manages all the sub-containers Nextcloud needs (database, caching, web server, etc.) through a single admin panel.

!!! warning "Port 9443 — important conflict fix"
    Steam's Gaming Mode opens a remote debugging port on `localhost:8080`. Nextcloud AIO also defaults to `8080`. If Nextcloud takes it first, the Decky Loader plugin system in Gaming Mode fails. We bind Nextcloud AIO to `9443` to avoid this conflict.

!!! info "AMD GPU acceleration"
    If you want Nextcloud to use your AMD GPU for video transcoding, **do not run the compose command below yet**. First read the [AMD GPU Acceleration](../reference/amd-gpu.md) reference page, then come back and use the modified compose file from there.

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
      - label:disable   # Required: allows access to the Docker socket under SELinux
    labels:
      - "com.centurylinklabs.watchtower.enable=false"  # Never auto-update AIO
    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - '127.0.0.1:9443:8080'  # Changed from 8080 to avoid Steam CEF conflict
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

---

## ✅ Phase 7 Complete

Three services are now running:

- 🦆 **DuckDNS** — keeping your domain pointed at your home IP
- 🔀 **Nginx Proxy Manager** — ready to route traffic (admin setup in Phase 10)
- ☁️ **Nextcloud AIO** — master container waiting for first-run setup (Phase 10)

[Next: Phase 8 — Session Scripts →](phase-8-scripts.md){ .next-phase }
