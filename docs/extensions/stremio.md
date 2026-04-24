# Multi-User VPN-Isolated Stremio Setup with Hardware Transcoding

## 1. Project Preparation
```bash
mkdir -p ~/docker/stremio/stremio-data/{dashboard,user1,user2}
cd ~/docker/stremio
```

## 2. Docker Compose Configuration
Create the file: `nano docker-compose.yml`. Replace placeholders with your PIA credentials, subdomains, and server local IP/subnet.

```yaml
services:
  # DASHBOARD: Heimdall
  stremio-dashboard:
    image: lscr.io/linuxserver/heimdall:latest
    container_name: stremio-dashboard
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Berlin
    volumes:
      - ./stremio-data/dashboard:/config
    ports:
      - "11470:80"
    restart: unless-stopped

  # USER 1: VPN & Stremio
  gluetun-user1:
    image: qmcgaw/gluetun:latest
    container_name: gluetun-user1
    cap_add: [NET_ADMIN]
    devices: [/dev/net/tun:/dev/net/tun]
    ports: ["11471:8080"]
    environment:
      - VPN_SERVICE_PROVIDER=private internet access
      - VPN_TYPE=openvpn
      - OPENVPN_USER=YOUR_PIA_USERNAME
      - OPENVPN_PASSWORD=YOUR_PIA_PASSWORD
      - SERVER_REGIONS=Netherlands
      - VPN_PORT_FORWARDING=on
      - DNS_KEEP_NAMESERVERS=on
      - FIREWALL_OUTBOUND_SUBNETS=192.168.178.0/24 # Your local network
    extra_hosts:
      - "user1.yourname.duckdns.org:192.168.178.139" # Your local server IP
    restart: unless-stopped

  stremio-user1:
    image: tsaridas/stremio-docker:latest
    container_name: stremio-user1
    network_mode: "service:gluetun-user1"
    devices: [/dev/dri:/dev/dri]
    group_add: ["39", "105"]
    environment:
      - AUTO_SERVER_URL=1
      - NO_CORS=1
      - FF_MPEG_FORCE_HW=1
      - SERVER_URL=https://user1.yourname.duckdns.org
    volumes:
      - ./stremio-data/user1:/root/.stremio-server
    depends_on:
      gluetun-user1: { condition: service_healthy }
    restart: unless-stopped

  # USER 2: VPN & Stremio
  gluetun-user2:
    image: qmcgaw/gluetun:latest
    container_name: gluetun-user2
    cap_add: [NET_ADMIN]
    devices: [/dev/net/tun:/dev/net/tun]
    ports: ["11472:8080"]
    environment:
      - VPN_SERVICE_PROVIDER=private internet access
      - VPN_TYPE=openvpn
      - OPENVPN_USER=YOUR_PIA_USERNAME
      - OPENVPN_PASSWORD=YOUR_PIA_PASSWORD
      - SERVER_REGIONS=Switzerland
      - VPN_PORT_FORWARDING=on
      - DNS_KEEP_NAMESERVERS=on
      - FIREWALL_OUTBOUND_SUBNETS=192.168.178.0/24
    extra_hosts:
      - "user2.yourname.duckdns.org:192.168.178.139"
    restart: unless-stopped

  stremio-user2:
    image: tsaridas/stremio-docker:latest
    container_name: stremio-user2
    network_mode: "service:gluetun-user2"
    devices: [/dev/dri:/dev/dri]
    group_add: ["39", "105"]
    environment:
      - AUTO_SERVER_URL=1
      - NO_CORS=1
      - FF_MPEG_FORCE_HW=1
      - SERVER_URL=https://user2.yourname.duckdns.org
    volumes:
      - ./stremio-data/user2:/root/.stremio-server
    depends_on:
      gluetun-user2: { condition: service_healthy }
    restart: unless-stopped
```
**Deploy:** `docker compose up -d`

## 3. Nginx Route Patch (Fixes HEVC/x265 Error)
Run these commands to fix the bug preventing playback of `-1` index files (common in 4K/HEVC torrents):
```bash
docker exec -it stremio-user1 sed -i 's/\[0-9\]+)\$"/[-0-9]+)\$"/g' /etc/nginx/http.d/default.conf /etc/nginx/https.conf
docker exec -it stremio-user1 nginx -s reload

docker exec -it stremio-user2 sed -i 's/\[0-9\]+)\$"/[-0-9]+)\$"/g' /etc/nginx/http.d/default.conf /etc/nginx/https.conf
docker exec -it stremio-user2 nginx -s reload
```

## 4. Nginx Proxy Manager (NPM) Setup

### A. The Access List
1. Create List: `Stremio-Auth`.
2. **Details Tab**: Set **Satisfy Any** to **ON**.
3. **Authorization Tab**: Set your master username/password.
4. **Access Tab**: Add `172.16.0.0/12` and set to **Allow** (Whitelists Docker internals).

### B. Proxy Hosts
For all 3 hosts: Use **Scheme: http**, **Forward IP: <Bazzite_Local_IP>**. Enable **Force SSL** and **HSTS**.

1. **Dashboard**: Domain `stremio.yourname...` | Port `11470` | Access List `Publicly Accessible`.
2. **User 1**: Domain `user1.yourname...` | Port `11471` | Access List `Stremio-Auth`. Enable **Websockets Support**.
3. **User 2**: Domain `user2.yourname...` | Port `11472` | Access List `Stremio-Auth`. Enable **Websockets Support**.

### C. Advanced Tab (User 1 & 2 Only)
Paste this to optimize video chunk delivery and prevent disconnects:
```nginx
proxy_buffering off;
proxy_request_buffering off;
proxy_max_temp_file_size 0;
proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
send_timeout 3600s;
client_max_body_size 0;
```

## 5. Stremio Web UI Configuration
Log into each user's URL and apply these under **Settings -> Streaming**:
1. **Streaming server URL**: Ensure it shows "Online" at your subdomain.
2. **Torrent Profile**: `Ultra Fast`.
3. **Transcoding**: `Enabled`.
4. **Transcode profile**: `vaapi-renderD128` (Enables GPU Hardware Acceleration).
