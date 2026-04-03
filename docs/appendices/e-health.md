# Appendix E: Server Health Monitor

The `server-health` script is a live terminal dashboard that displays CPU, RAM, GPU, storage, Docker container status, Nextcloud health, Fail2ban count, and SDDM session state.

**1. Install the script**
```bash
sudo nano /usr/local/bin/server-health
```
Paste this script:
```bash
#!/bin/bash
# server-health — Bazzite server status dashboard
# Live mode: updates every few seconds · Ctrl+C to exit

R=$'\033[0m'; DIM=$'\033[2;37m'; BLD=$'\033[1m'
CYN=$'\033[1;36m'; BLU=$'\033[1;34m'; WHT=$'\033[1;37m'
GRN=$'\033[1;32m'; YEL=$'\033[1;33m'; RED=$'\033[1;31m'

THICK="${CYN}$(printf '━%.0s' {1..72})${R}"
THIN="${DIM}$(printf '─%.0s' {1..72})${R}"
REFRESH=2 
SLOW_EVERY=4 

row() { printf " ${BLU}%-8s${R} %b\n" "$1" "${*:2}"; }
dot_ok() { printf "${GRN}●${R}"; }
dot_warn() { printf "${YEL}●${R}"; }
dot_err() { printf "${RED}●${R}"; }
dot_dim() { printf "${DIM}●${R}"; }

pct_dot() {
  local p=$1 warn=${2:-70} err=${3:-90}
  if (( p >= err )); then dot_err
  elif (( p >= warn )); then dot_warn
  else dot_ok; fi
}

trap 'tput cnorm; printf "\n"; exit 0' INT TERM
[ -t 1 ] && tput civis

GPU_DEV=""
for _card in /sys/class/drm/card*/; do
  _vendor=$(cat "${_card}device/vendor" 2>/dev/null)
  [[ "$_vendor" == "0x1002" || "$_vendor" == "0x8086" ]] && \
    { GPU_DEV="${_card}device"; break; }
done

_cpustat() { awk '/^cpu /{print $2,$3,$4,$5,$6,$7,$8; exit}' /proc/stat; }
prev_cs=$(_cpustat)
cpu_pct=0

refresh_slow() {
  WAN_IP=$(curl -s --max-time 4 https://api.ipify.org 2>/dev/null)
  [ -z "$WAN_IP" ] && WAN_IP="unreachable"
  
  F2B=$(sudo fail2ban-client status sshd 2>/dev/null | grep 'Currently banned' | grep -oE '[0-9]+')
  
  _nc=$(curl -s -o /dev/null -w "%{http_code}" --max-time 2 http://localhost:81 2>/dev/null)
  [ "${_nc:-000}" != "000" ] && { NPM_DOT=$(dot_ok); NPM_ST="ok"; } || { NPM_DOT=$(dot_err); NPM_ST="down"; }
}

# Main loop simplified for space... (See full script in original guide for exact layout)
while true; do
  refresh_slow
  printf '\033[H\033[2J'
  printf "%s\n" "$THICK"
  printf " BAZZITE SERVER STATUS\n"
  printf "%s\n" "$THICK"
  
  row "NETWORK" "SSH conn · F2B ${F2B} banned · WAN $WAN_IP"
  row "SERVICES" "NPM $NPM_DOT $NPM_ST"
  
  printf "\n%s\n" "$THIN"
  printf " refresh ${REFRESH}s · Ctrl+C to exit\n"
  sleep "$REFRESH"
done
```
Make it executable: `sudo chmod 755 /usr/local/bin/server-health`

Run the script by typing `server-health` in your terminal.
