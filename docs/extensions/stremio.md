# Multi-User VPN-Isolated Stremio Setup with Hardware Transcoding

## 1. Project Preparation
```bash
mkdir -p ~/docker/stremio/stremio-data/{dashboard,user1,user2}
cd ~/docker/stremio
```

## 2. Docker Compose Configuration
Create `docker-compose.yml`. Replace subdomains and the `192.168.178.x` values with your local network data.

```yaml
# --- SHARED ANCHORS ---
x-vpn-env: &vpn-env
  VPN_SERVICE_PROVIDER: private internet access
  VPN_TYPE: openvpn
  OPENVPN_USER: 
  OPENVPN_PASSWORD:
  VPN_PORT_FORWARDING: "on"
  SERVER_REGIONS: Netherlands,Germany,Switzerland,Belgium,France,Austria
  FIREWALL_OUTBOUND_SUBNETS: 192.168.178.0/24

x-stremio-env: &stremio-env
  FF_MPEG_FORCE_HW: 1
  NO_CORS: 1

x-stremio-base: &stremio-base
  image: tsaridas/stremio-docker:latest
  devices: [/dev/dri:/dev/dri]
  group_add: ["39", "105"]
  restart: unless-stopped
  # Fixes the "-1" file index bug for HEVC/x265 streams automatically
  entrypoint: ["sh", "-c", "sed -i 's/\\[0-9\\]+)\\$$\"/[-0-9]+)\\$$\"/g' /etc/nginx/http.d/default.conf /etc/nginx/https.conf && /srv/stremio-server/stremio-web-service-run.sh"]

services:
  stremio-dashboard:
    image: lscr.io/linuxserver/heimdall:latest
    container_name: stremio-dashboard
    environment: { PUID: 1000, PGID: 1000, TZ: Europe/Berlin }
    volumes: [ ./stremio-data/dashboard:/config ]
    ports: [ "11470:80" ]
    restart: unless-stopped

  # --- USER 1 ---
  gluetun-user1:
    image: qmcgaw/gluetun:latest
    container_name: gluetun-user1
    cap_add: [NET_ADMIN]
    devices: [/dev/net/tun:/dev/net/tun]
    ports: ["11471:8080"]
    environment: *vpn-env
    extra_hosts: ["user1stream.yourdomain.duckdns.org:192.168.178.139"]
    restart: unless-stopped

  stremio-user1:
    <<: *stremio-base
    container_name: stremio-user1
    network_mode: "service:gluetun-user1"
    environment:
      <<: *stremio-env
      SERVER_URL: https://user1stream.yourdomain.duckdns.org
    volumes: [ ./stremio-data/user1:/root/.stremio-server ]

  # --- USER 2 ---
  gluetun-user2:
    image: qmcgaw/gluetun:latest
    container_name: gluetun-user2
    cap_add: [NET_ADMIN]
    devices: [/dev/net/tun:/dev/net/tun]
    ports: ["11472:8080"]
    environment: *vpn-env
    extra_hosts: ["user2stream.yourdomain.duckdns.org:192.168.178.139"]
    restart: unless-stopped

  stremio-user2:
    <<: *stremio-base
    container_name: stremio-user2
    network_mode: "service:gluetun-user2"
    environment:
      <<: *stremio-env
      SERVER_URL: https://user2stream.yourdomain.duckdns.org
    volumes: [ ./stremio-data/user2:/root/.stremio-server ]
```
**Deploy:** `docker compose up -d`

## 3. Nginx Proxy Manager (NPM) Setup

### A. The Access List (Fixes Audio/Subtitle Buttons)
1.  **Add Access List**: Name it `Stremio-Auth`.
2.  **Details**: Set **Satisfy Any** to **ON**.
3.  **Authorization**: Set your preferred username/password.
4.  **Access**: Add `172.16.0.0/12` set to **Allow** (whitelists Docker traffic).

### B. Proxy Hosts
For each user subdomain:
1.  **Details**: Forward to local server IP on ports `11471` (User 1) and `11472` (User 2).
2.  **Details**: Select `Stremio-Auth` Access List. Toggle **Websockets Support** ON.
3.  **SSL**: Enable **Force SSL** and **HSTS**.
4.  **Advanced Tab**: Paste the following to optimize streaming throughput:
```nginx
proxy_buffering off;
proxy_request_buffering off;
proxy_max_temp_file_size 0;
proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
send_timeout 3600s;
client_max_body_size 0;
```

## 4. Final App Configuration
Log into each Stremio URL and navigate to **Settings -> Streaming**:
1.  **Streaming server URL**: Ensure it shows "Online" at your subdomain.
2.  **Torrent Profile**: `Ultra Fast`.
3.  **Transcoding**: `Enabled`.
4.  **Transcode profile**: `vaapi-renderD128` (Hardware Acceleration).
