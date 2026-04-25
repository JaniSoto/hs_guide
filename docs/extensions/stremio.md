# Stremio + PIA Auto-Rotating WireGuard Stack

Self-hosted Stremio with automatic PIA WireGuard key rotation, stream-safe restarts, and per-user VPN tunnels. No manual VPN management required after first setup.

---

## How It Works

| Container | Role |
|---|---|
| `gluetun-user1/2` | WireGuard VPN tunnel per user |
| `stremio-user1/2` | Stremio server, routed through its gluetun |
| `pia-watcher` | Picks best PIA region by latency, generates WireGuard keys, rotates every 6h, waits for streams to finish before restarting |
| `stremio-dashboard` | Heimdall dashboard at port 11470 |

The watcher writes `wg0.userN.conf` files directly to disk. Gluetun reads these on restart — no env variable rebaking, keys are always fresh.

---

## Prerequisites

- Docker + Docker Compose installed
- A PIA subscription (username + password)
- Ports `11470–11472` available on the host
- A domain pointing to your server (e.g. via DuckDNS) — one per user

---

## Step 1 — Create the project folder

```bash
mkdir -p ~/docker/stremio
cd ~/docker/stremio
```

---

## Step 2 — Create placeholder conf files

Docker will create these as **directories** on first start if they don't exist as files. Do this first:

```bash
touch wg0.user1.conf wg0.user2.conf
```

---

## Step 3 — Create `docker-compose.yml`

Replace every value marked `CHANGE_ME`.

```yaml
# --- SHARED ANCHORS ---
x-vpn-env: &vpn-env
  VPN_SERVICE_PROVIDER: custom
  VPN_TYPE: wireguard
  FIREWALL_OUTBOUND_SUBNETS: 192.168.178.0/24   # CHANGE_ME: your LAN subnet

x-stremio-env: &stremio-env
  FF_MPEG_FORCE_HW: 1
  NO_CORS: 1

x-stremio-base: &stremio-base
  image: tsaridas/stremio-docker:latest
  devices: [/dev/dri:/dev/dri]
  group_add: ["39", "105"]
  restart: unless-stopped
  entrypoint: ["sh", "-c", "sed -i 's/\\[0-9\\]+)\\$$\"/[-0-9]+)\\$$\"/g' /etc/nginx/http.d/default.conf /etc/nginx/https.conf && /srv/stremio-server/stremio-web-service-run.sh"]

services:
  stremio-dashboard:
    image: lscr.io/linuxserver/heimdall:latest
    container_name: stremio-dashboard
    environment: { PUID: 1000, PGID: 1000, TZ: Europe/Berlin }  # CHANGE_ME: your timezone
    volumes:
      - ./stremio-data/dashboard:/config
    ports: ["11470:80"]
    restart: unless-stopped

  # --- USER 1 ---
  gluetun-user1:
    image: qmcgaw/gluetun:latest
    container_name: gluetun-user1
    cap_add: [NET_ADMIN]
    devices: [/dev/net/tun:/dev/net/tun]
    ports: ["11471:8080"]
    environment:
      <<: *vpn-env
    volumes:
      - ./wg0.user1.conf:/gluetun/wireguard/wg0.conf:ro
    extra_hosts:
      - "user1.yourdomain.com:192.168.1.100"   # CHANGE_ME: user1 domain + server LAN IP
    restart: unless-stopped

  stremio-user1:
    <<: *stremio-base
    container_name: stremio-user1
    network_mode: "service:gluetun-user1"
    environment:
      <<: *stremio-env
      SERVER_URL: https://user1.yourdomain.com  # CHANGE_ME: user1 public URL
    volumes: [./stremio-data/user1:/root/.stremio-server]

  # --- USER 2 ---
  gluetun-user2:
    image: qmcgaw/gluetun:latest
    container_name: gluetun-user2
    cap_add: [NET_ADMIN]
    devices: [/dev/net/tun:/dev/net/tun]
    ports: ["11472:8080"]
    environment:
      <<: *vpn-env
    volumes:
      - ./wg0.user2.conf:/gluetun/wireguard/wg0.conf:ro
    extra_hosts:
      - "user2.yourdomain.com:192.168.1.100"   # CHANGE_ME: user2 domain + server LAN IP
    restart: unless-stopped

  stremio-user2:
    <<: *stremio-base
    container_name: stremio-user2
    network_mode: "service:gluetun-user2"
    environment:
      <<: *stremio-env
      SERVER_URL: https://user2.yourdomain.com  # CHANGE_ME: user2 public URL
    volumes: [./stremio-data/user2:/root/.stremio-server]

  pia-watcher:
    image: alpine:latest
    container_name: pia-watcher
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./:/work
    working_dir: /work
    environment:
      PIA_USER: "your_pia_username"             # CHANGE_ME
      PIA_PASS: "your_pia_password"             # CHANGE_ME
      REFRESH_HOURS: "6"
      REGIONS: "nl_netherlands-so,de_germany-so,nl_amsterdam,de-frankfurt,de_berlin,swiss,belgium,france,austria"
      HEALTH_POLL: "60"
      STREAM_THRESHOLD_KBPS: "500"
    command: >
      sh -c "apk add -q curl jq wireguard-tools iputils &&
             sh /work/pia-watcher.sh"
```

> **Region IDs** — these are exact PIA API identifiers. To find IDs for other countries run:
> ```bash
> docker exec pia-watcher sh -c "curl -sf 'https://serverlist.piaservers.net/vpninfo/servers/v5' | head -1 | jq -r '.regions[] | \"\(.id) → \(.name)\"'"
> ```

---

## Step 4 — Create `pia-watcher.sh`

```sh
#!/bin/sh
# pia-watcher.sh — Auto best-region WireGuard manager for PIA + Gluetun

set -e

# ── CONFIG ───────────────────────────────────────────────────────────────
PIA_USER="${PIA_USER:-}"
PIA_PASS="${PIA_PASS:-}"
REGIONS="${REGIONS:-nl_netherlands-so,de_germany-so,nl_amsterdam,de-frankfurt,de_berlin,swiss,belgium,france,austria}"
REFRESH_SECONDS=$(( ${REFRESH_HOURS:-6} * 3600 ))
HEALTH_POLL="${HEALTH_POLL:-60}"
STREAM_THRESHOLD_KBPS="${STREAM_THRESHOLD_KBPS:-500}"
WORK_DIR="${WORK_DIR:-/work}"
USERS="user1 user2"
DOCKER_SOCK="/var/run/docker.sock"

# ── HELPERS ──────────────────────────────────────────────────────────────
log() { printf '[%s] %s\n' "$(date '+%H:%M:%S')" "$*" >&2; }
docker_post() { curl -sf --unix-socket "$DOCKER_SOCK" -X POST "http://localhost$1" > /dev/null 2>&1 || true; }
docker_get()  { curl -sf --unix-socket "$DOCKER_SOCK" "http://localhost$1"; }

container_health() {
  docker_get "/containers/$1/json" 2>/dev/null | jq -r '.State.Health.Status // "unknown"' 2>/dev/null || echo "unknown"
}

wait_healthy() {
  local CONTAINER="$1"
  local TIMEOUT="${2:-90}"
  local ELAPSED=0
  log " Waiting for $CONTAINER to become healthy (max ${TIMEOUT}s)..."
  while [ "$ELAPSED" -lt "$TIMEOUT" ]; do
    STATUS=$(container_health "$CONTAINER")
    case "$STATUS" in
      healthy)   log " [OK] $CONTAINER is healthy."; return 0 ;;
      unhealthy) log " [FAIL] $CONTAINER unhealthy after restart."; return 1 ;;
    esac
    sleep 4
    ELAPSED=$(( ELAPSED + 4 ))
  done
  log " [WARN] $CONTAINER did not become healthy within ${TIMEOUT}s"
  return 1
}

stop_container()    { docker_post "/containers/$1/stop?t=10"; }
start_container()   { docker_post "/containers/$1/start"; }
restart_container() { docker_post "/containers/$1/restart?t=10"; }

# ── STREAM DETECTION ─────────────────────────────────────────────────────
exec_in() {
  local CONTAINER="$1"
  local CMD="$2"
  EXEC_ID=$(curl -sf --unix-socket "$DOCKER_SOCK" \
    -X POST -H "Content-Type: application/json" \
    "http://localhost/containers/$CONTAINER/exec" \
    -d "{\"AttachStdout\":true,\"AttachStderr\":false,\"Tty\":true,\"Cmd\":[\"sh\",\"-c\",\"$CMD\"]}" 2>/dev/null | jq -r '.Id' 2>/dev/null)
  [ -z "$EXEC_ID" ] || [ "$EXEC_ID" = "null" ] && return 1
  curl -sf --unix-socket "$DOCKER_SOCK" \
    -X POST -H "Content-Type: application/json" \
    "http://localhost/exec/$EXEC_ID/start" \
    -d '{"Detach":false,"Tty":true}' 2>/dev/null
}

get_eth0_bytes() {
  exec_in "$1" "awk '/eth0/{print \$2+\$10}' /proc/net/dev" 2>/dev/null | tr -d '[:space:][:cntrl:]'
}

is_streaming() {
  local CONTAINER="$1"
  local THRESHOLD=$(( STREAM_THRESHOLD_KBPS * 1024 ))
  local SAMPLE_SECS=3
  BYTES1=$(get_eth0_bytes "$CONTAINER")
  sleep "$SAMPLE_SECS"
  BYTES2=$(get_eth0_bytes "$CONTAINER")
  case "${BYTES1}${BYTES2}" in
    *[!0-9]*|"") log " [stream check] Could not read stats from $CONTAINER -- assuming idle."; return 1 ;;
  esac
  RATE=$(( (BYTES2 - BYTES1) / SAMPLE_SECS ))
  RATE_KBPS=$(( RATE / 1024 ))
  if [ "$RATE" -gt "$THRESHOLD" ]; then
    log " [stream check] $CONTAINER: ${RATE_KBPS} KB/s -- STREAMING"
    return 0
  else
    log "[stream check] $CONTAINER: ${RATE_KBPS} KB/s -- idle"
    return 1
  fi
}

any_user_streaming() {
  for USER in $USERS; do
    if is_streaming "gluetun-${USER}"; then return 0; fi
  done
  return 1
}

# ── LATENCY TEST ─────────────────────────────────────────────────────────
ping_ms() {
  RESULT=$(ping -c 3 -W 2 "$1" 2>/dev/null \
    | grep -i 'round-trip\|rtt' \
    | grep -oE '[0-9]+\.[0-9]+' | head -1 | awk '{print int($1)}')
  echo "${RESULT:-9999}"
}

pick_best_region() {
  log "Fetching PIA server list..."
  SERVERS=$(curl -sf "https://serverlist.piaservers.net/vpninfo/servers/v5" | head -1)
  [ -z "$SERVERS" ] && { log "ERROR: Could not reach PIA server list API."; exit 1; }

  BEST_REGION="" BEST_RTT=9999 BEST_NAME="" FIRST_REGION=""

  for REGION in $(echo "$REGIONS" | tr ',' ' '); do
    WG_IP=$(echo "$SERVERS" | jq -r ".regions[] | select(.id == \"$REGION\") | .servers.wg[0].ip" 2>/dev/null)
    WG_NAME=$(echo "$SERVERS" | jq -r ".regions[] | select(.id == \"$REGION\") | .name" 2>/dev/null)
    if [ -z "$WG_IP" ] || [ "$WG_IP" = "null" ]; then
      log "[$REGION] Not found in server list -- skipping."
      continue
    fi
    [ -z "$FIRST_REGION" ] && FIRST_REGION="$REGION"
    RTT=$(ping_ms "$WG_IP")
    log "  $WG_NAME ($REGION): ${RTT}ms"
    if [ "$RTT" -lt "$BEST_RTT" ] 2>/dev/null; then
      BEST_REGION="$REGION"; BEST_RTT="$RTT"; BEST_NAME="$WG_NAME"
    fi
  done

  if [ -z "$BEST_REGION" ]; then
    [ -n "$FIRST_REGION" ] && { BEST_REGION="$FIRST_REGION"; log "WARNING: Could not ping any region. Falling back to $BEST_REGION"; } \
      || { log "ERROR: No valid regions found."; exit 1; }
  else
    log "* Best region: $BEST_NAME ($BEST_REGION) -- ${BEST_RTT}ms"
  fi
  echo "$BEST_REGION"
}

# ── CREDENTIAL GENERATION ────────────────────────────────────────────────
generate_creds() {
  local REGION="$1" LABEL="$2"
  local OUT="${WORK_DIR}/wg0.${LABEL}.conf"

  log "[$LABEL] Authenticating with PIA..."
  TOKEN=$(curl -sf -u "$PIA_USER:$PIA_PASS" "https://privateinternetaccess.com/gtoken/generateToken" | jq -r '.token')
  [ -z "$TOKEN" ] || [ "$TOKEN" = "null" ] && { log "ERROR: PIA auth failed."; exit 1; }

  SERVERS=$(curl -sf "https://serverlist.piaservers.net/vpninfo/servers/v5" | head -1)
  WG_IP=$(echo "$SERVERS" | jq -r ".regions[] | select(.id == \"$REGION\") | .servers.wg[0].ip")
  WG_CN=$(echo "$SERVERS" | jq -r ".regions[] | select(.id == \"$REGION\") | .servers.wg[0].cn")
  [ -z "$WG_IP" ] || [ -z "$WG_CN" ] && { log "ERROR: Could not fetch IP/CN for '$REGION'."; exit 1; }

  log "[$LABEL] Generating WireGuard keypair..."
  PRIVATE_KEY=$(wg genkey)
  PUBLIC_KEY=$(echo "$PRIVATE_KEY" | wg pubkey)

  log "[$LABEL] Registering key with PIA ($WG_IP / $WG_CN)..."
  curl -sLo /tmp/pia-ca.crt "https://raw.githubusercontent.com/pia-foss/manual-connections/master/ca.rsa.4096.crt" 2>/dev/null || true

  RESULT=$(curl -sf -G \
    --cacert /tmp/pia-ca.crt \
    --resolve "$WG_CN:1337:$WG_IP" \
    --data-urlencode "pt=$TOKEN" \
    --data-urlencode "pubkey=$PUBLIC_KEY" \
    "https://$WG_CN:1337/addKey")

  SERVER_KEY=$(echo "$RESULT" | jq -r '.server_key')
  PEER_IP=$(echo "$RESULT" | jq -r '.peer_ip')
  [ -z "$SERVER_KEY" ] || [ "$SERVER_KEY" = "null" ] && { log "ERROR: Key registration failed. Response: $RESULT"; exit 1; }

  cat > "$OUT" << EOF
[Interface]
PrivateKey = $PRIVATE_KEY
Address = $PEER_IP/32

[Peer]
PublicKey = $SERVER_KEY
Endpoint = $WG_IP:1337
AllowedIPs = 0.0.0.0/0
EOF
  log "[$LABEL] [OK] Written -- VPN IP: $PEER_IP"
}

# ── STACK CYCLE ──────────────────────────────────────────────────────────
cycle_user() {
  local USER="$1"
  log "Cycling $USER stack..."
  stop_container "stremio-${USER}"
  sleep 2
  restart_container "gluetun-${USER}"
  if wait_healthy "gluetun-${USER}" 90; then
    start_container "stremio-${USER}"
    log "[$USER] [OK] Stack back online."
  else
    log "[$USER] WARNING: gluetun unhealthy -- stremio not started. Check: docker logs gluetun-${USER}"
  fi
}

# ── FULL REFRESH ─────────────────────────────────────────────────────────
refresh_all() {
  local REASON="${1:-scheduled}" FORCE="${2:-0}"
  log "========================================"; log "REFRESH triggered: $REASON"; log "========================================"

  if [ "$FORCE" -eq 0 ]; then
    log "Checking for active streams..."
    if any_user_streaming; then
      log "Active stream detected -- waiting until stream ends before restarting..."
      while any_user_streaming; do
        log "  Still streaming... rechecking in 30s"
        sleep 30
      done
      log "Stream ended -- proceeding."
    else
      log "No active streams -- proceeding immediately."
    fi
  else
    log "(Forced refresh -- stream check skipped)"
  fi

  BEST_REGION=$(pick_best_region)
  for USER in $USERS; do generate_creds "$BEST_REGION" "$USER"; done
  for USER in $USERS; do cycle_user "$USER"; done

  log "========================================"; log "Refresh complete. Next in ${REFRESH_HOURS:-6}h."; log "========================================"
}

# ── ENTRY POINT ──────────────────────────────────────────────────────────
[ -z "$PIA_USER" ] || [ -z "$PIA_PASS" ] && { log "FATAL: PIA_USER and PIA_PASS must be set."; exit 1; }

log "PIA WireGuard Watcher v2.0 starting"
log "Candidates : $REGIONS"
log "Refresh    : every ${REFRESH_HOURS:-6}h"
log "Stream guard: wait indefinitely for stream to end before refresh"
log "Health poll: every ${HEALTH_POLL}s"

refresh_all "startup" 1
LAST_REFRESH=$(date +%s)

while true; do
  sleep "$HEALTH_POLL"
  NOW=$(date +%s)
  TRIGGERED=0
  for USER in $USERS; do
    GLUETUN="gluetun-${USER}"
    if [ "$(container_health "$GLUETUN")" = "unhealthy" ] && [ "$TRIGGERED" -eq 0 ]; then
      log "ALERT: $GLUETUN is UNHEALTHY -- emergency refresh"
      refresh_all "health failure ($GLUETUN)" 1
      LAST_REFRESH=$(date +%s)
      TRIGGERED=1
    fi
  done
  if [ "$TRIGGERED" -eq 0 ] && [ $(( NOW - LAST_REFRESH )) -ge "$REFRESH_SECONDS" ]; then
    refresh_all "scheduled ${REFRESH_HOURS:-6}h interval" 0
    LAST_REFRESH=$(date +%s)
  fi
done
```

---

## Step 5 — Make the script executable

```bash
chmod +x pia-watcher.sh
```

---

## Step 6 — Start the stack

```bash
docker compose up -d
```

Watch the watcher initialize:

```bash
docker logs -f pia-watcher
```

Expected output on a healthy start:

```
[07:29:05] PIA WireGuard Watcher v2.0 starting
[07:29:08]   Switzerland (swiss): 9ms
[07:29:14] * Best region: Switzerland (swiss) -- 9ms
[07:29:14] [user1] [OK] Credentials written -- assigned VPN IP: 10.x.x.x
[07:29:35]  [OK] gluetun-user1 is healthy.
[07:29:56]  [OK] gluetun-user2 is healthy.
[07:29:56] Refresh complete. Next in 6h.
```

---

## Ports Reference

| Port | Service |
|---|---|
| `11470` | Stremio dashboard (Heimdall) |
| `11471` | Stremio user1 (via gluetun-user1) |
| `11472` | Stremio user2 (via gluetun-user2) |

---

## Troubleshooting

**Gluetun unhealthy after restart**
```bash
docker logs gluetun-user1
```
Usually means the conf file is empty (placeholder wasn't overwritten yet) or PIA auth failed. Check `docker logs pia-watcher`.

**Region skipped with "Not found in server list"**
The region ID is wrong. Find correct IDs:
```bash
docker exec pia-watcher sh -c "curl -sf 'https://serverlist.piaservers.net/vpninfo/servers/v5' | head -1 | jq -r '.regions[] | \"\(.id) → \(.name)\"'"
```

**Watcher crashing on start**
Confirm `wg0.user1.conf` and `wg0.user2.conf` exist as **files** (not directories):
```bash
ls -la wg0.*.conf
```
If they're directories, remove them and re-run Step 2.

**Force an immediate refresh**
```bash
docker restart pia-watcher
```
