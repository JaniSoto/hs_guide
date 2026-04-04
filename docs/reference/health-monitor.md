# Server Health Monitor

A live terminal dashboard that shows CPU, RAM, GPU, storage, Docker container status, Fail2ban bans, and SDDM session state — all updating every 2 seconds.

---

## Install the Script

```bash
sudo nano /usr/local/bin/server-health
```

Paste the full script:

```bash
#!/bin/bash
# server-health — Bazzite server status dashboard
# Live mode: updates every 2 seconds · Ctrl+C to exit

R=$'\033[0m'; DIM=$'\033[2;37m'; BLD=$'\033[1m'
CYN=$'\033[1;36m'; BLU=$'\033[1;34m'; WHT=$'\033[1;37m'
GRN=$'\033[1;32m'; YEL=$'\033[1;33m'; RED=$'\033[1;31m'

THICK="${CYN}$(printf '━%.0s' {1..72})${R}"
THIN="${DIM}$(printf '─%.0s' {1..72})${R}"
REFRESH=2
SLOW_EVERY=4

row()     { printf " ${BLU}%-10s${R} %b\n" "$1" "${*:2}"; }
dot_ok()  { printf "${GRN}●${R}"; }
dot_warn(){ printf "${YEL}●${R}"; }
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

# Detect AMD/Intel GPU sysfs node
GPU_DEV=""
for _card in /sys/class/drm/card*/; do
  _vendor=$(cat "${_card}device/vendor" 2>/dev/null)
  [[ "$_vendor" == "0x1002" || "$_vendor" == "0x8086" ]] && \
    { GPU_DEV="${_card}device"; break; }
done

_cpustat() { awk '/^cpu /{print $2,$3,$4,$5,$6,$7,$8; exit}' /proc/stat; }
prev_cs=$(_cpustat); cpu_pct=0

# ── Slow refresh data (network, services) ─────────────────────────
WAN_IP="..."; F2B="0"; NPM_DOT="$(dot_dim)"; NPM_ST="?"
LOOPS=0

refresh_slow() {
  WAN_IP=$(curl -s --max-time 4 https://api.ipify.org 2>/dev/null || echo "unreachable")

  F2B=$(sudo fail2ban-client status sshd 2>/dev/null \
    | grep 'Currently banned' | grep -oE '[0-9]+' || echo "0")

  _npm=$(curl -s -o /dev/null -w "%{http_code}" --max-time 2 http://localhost:81 2>/dev/null)
  if [[ "${_npm:-000}" != "000" ]]; then
    NPM_DOT=$(dot_ok); NPM_ST="reachable"
  else
    NPM_DOT=$(dot_err); NPM_ST="down"
  fi

  _nc=$(curl -s -o /dev/null -w "%{http_code}" --max-time 2 https://localhost:9443 -k 2>/dev/null)
  if [[ "${_nc:-000}" != "000" ]]; then
    AIO_DOT=$(dot_ok); AIO_ST="reachable"
  else
    AIO_DOT=$(dot_err); AIO_ST="down"
  fi
}

refresh_slow

# ── Main loop ──────────────────────────────────────────────────────
while true; do
  (( LOOPS++ ))
  (( LOOPS % SLOW_EVERY == 0 )) && refresh_slow

  # CPU %
  curr_cs=$(_cpustat)
  read -r u1 n1 s1 i1 w1 q1 x1 <<< "$prev_cs"
  read -r u2 n2 s2 i2 w2 q2 x2 <<< "$curr_cs"
  total=$(( (u2-u1)+(n2-n1)+(s2-s1)+(i2-i1)+(w2-w1)+(q2-q1)+(x2-x1) ))
  idle=$(( i2-i1 ))
  (( total > 0 )) && cpu_pct=$(( 100 - 100*idle/total )) || cpu_pct=0
  prev_cs="$curr_cs"

  # RAM %
  read -r mem_total mem_avail < <(awk '/MemTotal/{t=$2} /MemAvailable/{a=$2} END{print t,a}' /proc/meminfo)
  mem_used=$(( mem_total - mem_avail ))
  mem_pct=$(( 100*mem_used/mem_total ))
  mem_used_gb=$(awk "BEGIN{printf \"%.1f\", ${mem_used}/1048576}")
  mem_total_gb=$(awk "BEGIN{printf \"%.1f\", ${mem_total}/1048576}")

  # GPU (AMD/Intel via hwmon)
  gpu_line="n/a"
  if [[ -n "$GPU_DEV" ]]; then
    _hwmon=$(ls "${GPU_DEV}/hwmon" 2>/dev/null | head -1)
    if [[ -n "$_hwmon" ]]; then
      _t=$(cat "${GPU_DEV}/hwmon/${_hwmon}/temp1_input" 2>/dev/null)
      [[ -n "$_t" ]] && gpu_line="${_t: : -3}°C"
    fi
  fi

  # Storage
  read -r disk_used disk_total disk_pct < <(df -BG / | awk 'NR==2{gsub(/G/,""); print $3,$2,$5+0}')
  read -r data_used data_total data_pct < <(df -BG /srv/nextcloud-data 2>/dev/null | awk 'NR==2{gsub(/G/,""); print $3,$2,$5+0}' || echo "? ? 0")

  # SDDM session
  sess=$(loginctl list-sessions --no-legend 2>/dev/null \
    | awk '$2==1000 && $4!="-" {print $5; exit}')
  [[ -z "$sess" ]] && sess="idle (headless)"

  # Docker containers
  docker_status=$(docker ps --format "{{.Names}} {{.Status}}" 2>/dev/null)

  printf '\033[H\033[2J'
  printf "%s\n" "$THICK"
  printf " ${CYN}${BLD}BAZZITE SERVER STATUS${R}  ${DIM}$(date '+%Y-%m-%d %H:%M:%S')${R}\n"
  printf "%s\n" "$THICK"

  cpu_dot=$(pct_dot $cpu_pct)
  mem_dot=$(pct_dot $mem_pct)
  disk_dot=$(pct_dot $disk_pct)
  data_dot=$(pct_dot $data_pct)

  row "CPU"     "${cpu_dot} ${cpu_pct}%"
  row "RAM"     "${mem_dot} ${mem_used_gb}G / ${mem_total_gb}G  (${mem_pct}%)"
  row "GPU"     "${gpu_line}"
  printf "%s\n" "$THIN"
  row "OS disk"   "${disk_dot} ${disk_used}G / ${disk_total}G  (${disk_pct}%)"
  row "Data disk" "${data_dot} ${data_used}G / ${data_total}G  (${data_pct}%)"
  printf "%s\n" "$THIN"
  row "Session" "${sess}"
  row "WAN IP"  "${WAN_IP}"
  row "Fail2ban" "${F2B} banned"
  printf "%s\n" "$THIN"
  row "NPM"     "${NPM_DOT} ${NPM_ST}"
  row "AIO"     "${AIO_DOT:-$(dot_dim)} ${AIO_ST:-?}"

  printf "%s\n" "$THIN"
  while IFS= read -r line; do
    name=$(awk '{print $1}' <<< "$line")
    stat=$(cut -d' ' -f2- <<< "$line")
    if [[ "$stat" == Up* ]]; then d=$(dot_ok)
    elif [[ "$stat" == Exited* ]]; then d=$(dot_err)
    else d=$(dot_warn); fi
    printf "  %s %-35s %b\n" "$d" "$name" "${DIM}${stat}${R}"
  done <<< "$docker_status"

  printf "\n${DIM} refresh ${REFRESH}s · Ctrl+C to exit${R}\n"
  sleep "$REFRESH"
done
```

Make it executable:

```bash
sudo chmod 755 /usr/local/bin/server-health
```

---

## Create a Sudoers Drop-in File

Because the script runs privileged commands (like `sudo fail2ban-client`) and requires root access to fetch full system data, you should configure a sudoers drop-in to allow running the script without being prompted for a password.

```bash
echo "$(whoami) ALL=(root) NOPASSWD: /usr/local/bin/server-health" | sudo tee /etc/sudoers.d/server-health
sudo chmod 0440 /etc/sudoers.d/server-health
```

---

## Create a Convenience Alias

To make launching the dashboard even easier, add an alias to your shell profile so you can simply type `health` from anywhere:

```bash
echo 'alias health="sudo server-health"' >> ~/.bashrc
source ~/.bashrc
```

*(If you are using Zsh, replace `.bashrc` with `.zshrc` in the above commands.)*

---

## Run the Dashboard

If you set up the alias, simply run:

```bash
health
```

Otherwise, run the script directly:

```bash
sudo server-health
```

Press `Ctrl+C` to exit.

---

## What It Shows

| Section | Details |
|---------|---------|
| **CPU** | Current usage %, colour-coded (green/yellow/red) |
| **RAM** | Used / Total in GB + percentage |
| **GPU** | Temperature (AMD/Intel via sysfs hwmon) |
| **OS disk** | Usage on the OS drive (`/`) |
| **Data disk** | Usage on `/srv/nextcloud-data` |
| **Session** | Current SDDM session type or "idle (headless)" |
| **WAN IP** | Your current public IP address |
| **Fail2ban** | Number of currently banned IPs |
| **NPM** | Nginx Proxy Manager reachability |
| **AIO** | Nextcloud AIO master container reachability |
| **Docker containers** | All running containers with live status |
