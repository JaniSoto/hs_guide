# SELF-HOSTED CLOUD + GAMING STATION — BAZZITE-DECK EDITION — FULLY UNIFIED GUIDE

**SSH-first · Zero New Open Ports · Wayland KDE · True Gaming Mode · Headless by Default**

`bazzite-deck` · Docker · firewalld · Fail2ban · Nginx Proxy Manager · Nextcloud AIO · DuckDNS · krdp · gamescope-session-plus · SSH Scripts

This is a single, self-contained guide that builds a headless home server with two on-demand graphical sessions — KDE Plasma and Gaming Mode — from a single bazzite-deck installation. It supersedes the two-document workflow (main guide + extension) by integrating every phase in the correct order from the very first boot. No rebase is ever required. All prior errors and outdated information have been corrected.

---

| Component | What You Get |
|---|---|
| **Headless server default** | The machine boots to an idle SDDM login screen with no graphical session. All Docker containers run normally as systemd services. SSH is always available on port 53222. |
| **KDE Wayland session** | Full KDE Plasma desktop on the physical display, launched on demand via a single SSH command. Also enables remote desktop access via krdp. |
| **Gaming Mode session** | True Steam Deck-style gamescope session on the physical display. Direct GPU scanout, controller-native. Launched via SSH. Refused if another session is already active. |
| **krdp remote desktop** | KDE's native Wayland RDP server. Shares the running KDE compositor session. Access tunneled exclusively through SSH — port 3389 is blocked by firewalld on the LAN. |
| **Nextcloud AIO** | Self-hosted cloud: files, calendar, contacts. Behind Nginx Proxy Manager with Let's Encrypt SSL via DuckDNS dynamic DNS. |
| **SSH-first design** | Every operation — starting sessions, stopping sessions, tunneling RDP — is a single SSH command from any device. No new ports beyond 53222 are ever opened. |

---

## ARCHITECTURE OVERVIEW

The server runs bazzite-deck — Bazzite's HTPC/living-room image — which ships both gamescope Gaming Mode and KDE Plasma in the same image. On first boot, the machine enters Gaming Mode once (expected behavior). We immediately apply a headless SDDM override from the KDE terminal, so every subsequent boot lands at an idle login screen with no session active. Docker containers start at boot via systemd, completely independently of any graphical session.

**Launching KDE:** SSH → `start-kde.sh` → script writes a one-shot SDDM autologin config → restarts SDDM → KDE Wayland appears on the physical display → headless config is restored 7 seconds later. Logging out of KDE via its own button returns to the idle SDDM screen.

**Remote desktop into KDE:** KDE must already be running. An SSH tunnel on port 53222 forwards krdp's port 3389 to your personal computer. Connect any RDP client to `localhost:3389`. Monitors can be off; krdp captures the compositor output independently of display power state. **Critical security note:** krdp listens on all network interfaces, not just localhost. firewalld is what keeps port 3389 off your LAN — never open 3389 in firewalld.

**Launching Gaming Mode:** SSH → `start-gaming.sh` → script checks for an active seat0 session and exits with an error if one is found → writes one-shot SDDM autologin config → restarts SDDM → gamescope session appears on the physical display. The Power menu in Steam Big Picture exits back to the idle SDDM screen.

---

## CRITICAL READING: COMMON PITFALLS & HOW THIS GUIDE AVOIDS THEM

| Pitfall | What Goes Wrong & The Fix |
|---|---|
| **Pitfall 1 — bazzite-deck boots to Gaming Mode on first install** | After a fresh bazzite-deck install, SDDM autologins into Gaming Mode on the very first boot. Phase 1 of this guide turns this into an opportunity: we switch to Desktop Mode from within Gaming Mode, enable SSH from the KDE terminal, then immediately apply the headless SDDM override — all in one physical session before leaving the machine. |
| **Pitfall 2 — SSH is not enabled by default on Bazzite** | Bazzite ships with SSH disabled. The very first action in Phase 1 Step 2 is enabling sshd from the KDE terminal. All subsequent steps are done remotely over SSH. |
| **Pitfall 3 — SDDM reads config once per start; timing matters** | The launch scripts write a one-shot autologin config, restart SDDM (which reads it and autologins), then restore the headless config in a background subshell after a 7-second delay. If the server lost power in that 7-second window, the next boot would autologin to the last launched session. Both scripts restore the headless config as their last action — the 7-second delay only exists so SDDM has time to start and read the config. |
| **Pitfall 4 — krdp listens on ALL interfaces, not just localhost** | Unlike what some documentation suggests, krdp (the Plasma 6 integrated RDP server) listens on `0.0.0.0` (all interfaces), not `127.0.0.1`. Port 3389 is therefore reachable on your LAN interface unless blocked. firewalld blocks it because we never open 3389. Phase 9 includes an explicit firewall verification step. Never open port 3389 in firewalld. |
| **Pitfall 5 — krdp requires an active KDE session** | krdp is KDE's native Wayland RDP server. Unlike xrdp it does not create a virtual session — it shares the running KDE compositor session. If no KDE session is active, krdp is not running and RDP connections will be refused. Always run `start-kde.sh` first. |
| **Pitfall 6 — Sudoers entries require absolute paths and root-owned scripts** | sudo never expands `~` or `$HOME` in sudoers files. All script paths must be absolute. More critically, any script with NOPASSWD sudo access must be owned by root and stored in a path the regular user cannot write to. This guide uses `/usr/local/bin/` with `root:root` ownership. A script in `~/scripts/` is user-writable and creates a trivial privilege escalation vector. |
| **Pitfall 7 — SELinux blocks sshd on non-standard ports** | Bazzite runs SELinux in enforcing mode. By default, SELinux only permits sshd to bind to ports labelled `ssh_port_t` — and only port 22 has that label. Restarting sshd on port 53222 without first updating the SELinux policy causes sshd to silently fail to bind, making the server unreachable. Fix: run `semanage port -a -t ssh_port_t -p tcp 53222` before restarting sshd. semanage is in policycoreutils-python-utils, which must therefore be installed before the SSH hardening phase. |
| **Pitfall 8 — Package installation must precede security hardening** | The SSH hardening phase needs semanage (from policycoreutils-python-utils) and the Fail2ban service (from fail2ban). Both are installed via rpm-ostree. If you attempt to configure or enable either tool before running the rpm-ostree deployment, the commands fail with 'service not found' or 'command not found'. This guide installs all packages in a dedicated phase before any security hardening begins. |
| **Pitfall 9 — Auto-suspend silently kills all containers** | Bazzite can suspend after 15 minutes of inactivity, silently killing all Docker containers. Phase 1 Step 4 permanently disables all suspend targets at the OS level before anything else. |
| **Pitfall 10 — Docker bypasses firewalld for published ports** | Docker injects nftables rules directly, bypassing firewalld for any port mapped in a Compose file. Only add firewalld rules for SSH (port 53222). Never add rules for ports 80, 443, or 3389 — Docker handles the first two, and 3389 must stay blocked. |

---

## CONTENTS

- Phase 0 — OS Installation (bazzite-deck from ISO)
- Phase 1 — First Boot: SSH Bootstrap & Headless Server Configuration
- Phase 2 — Router & Network Preparation
- Phase 3 — System Package Installation (Docker · Fail2ban · SELinux tools)
- Phase 4 — Ironclad Security (SELinux Port · SSH Keys · firewalld · Fail2ban)
- Phase 5 — Docker Initialization
- Phase 6 — Data Drive Setup
- Phase 7 — Deploying Services (DuckDNS · NPM · Nextcloud AIO)
- Phase 8 — Session Management Scripts (start-kde · start-gaming · stop-session)
- Phase 9 — krdp Remote Desktop
- Phase 10 — Link & Initialize (NPM config · Nextcloud setup)
- Phase 11 — Testing & Backups
- Appendix A — Troubleshooting
- Appendix B — Normal Workflows (Quick Reference)
- Appendix C — Optional Home Assistant Integration
- Appendix D — OS Updates & Rollbacks
- Appendix E — Storage Roadmap (SnapRAID + MergerFS)
- Appendix F — Understanding Your Stack (Deep Dive)

---

## RECOMMENDED MINIMUM HARDWARE

| Component | Minimum / Recommended |
|---|---|
| CPU | 4-core x86-64 minimum · 6-core or better recommended for combined server + gaming workloads |
| RAM | 8 GB DDR4 minimum · 16 GB+ recommended (gaming and AI workloads benefit significantly) |
| OS Drive | 120 GB NVMe SSD minimum · 500 GB+ NVMe recommended (OS + Docker images + databases) |
| Data Drive | Any size HDD or SSD · Must be a dedicated drive mounted at `/srv/nextcloud-data` |
| GPU | Integrated graphics (server only) · AMD recommended for gaming (plug-and-play on Bazzite) · NVIDIA in beta for Gaming Mode |
| Network | Gigabit Ethernet · Wi-Fi strongly discouraged for servers |

---

## PHASE 0 — OS INSTALLATION (BAZZITE-DECK FROM ISO)

We install the bazzite-deck image directly from an ISO — no rebase from a standard Bazzite install is ever needed. This is the HTPC/living-room variant of Bazzite that ships both gamescope Gaming Mode and full KDE Plasma in a single image. Starting here eliminates the entire rebase phase and the Docker reinstall that came with it.

### Step 1 · Download the correct ISO

Go to [bazzite.gg](https://bazzite.gg) and use the Image Picker tool. Select: Hardware = Desktop/HTPC, GPU = AMD/Intel or NVIDIA, Desktop = KDE Plasma, and enable Steam Gaming Mode (this selects the bazzite-deck variant). Download the ISO and flash it to a USB drive using Fedora Media Writer (free, cross-platform) or Balena Etcher.

> **NOTE:** The bazzite-deck ISO is the officially correct choice for a machine that will serve as both a headless server and a gaming station. It includes gamescope-session-plus for Gaming Mode and KDE Plasma for the desktop session. The two images are completely independent sessions — you switch between them via SSH commands as this guide describes.

### Step 2 · Install Bazzite

Boot from the USB drive and follow the Anaconda installer. Set your username, password, hostname (keep it under 20 characters), timezone, and keyboard layout. Install Bazzite to your OS drive. The installer does not require a network connection to complete.

> ⚠️ **WARNING: Dual Boot Users** — If you have a Windows drive in the same machine, physically disconnect it before installing Bazzite to prevent accidental data loss or bootloader modification.

### Step 3 · Identify your GPU for later reference

Note which GPU type your machine has (AMD, Intel, or NVIDIA). You will need this when enabling Gaming Mode in Phase 1. AMD and Intel GPUs are fully supported in Gaming Mode. NVIDIA support in Gaming Mode is available but described as beta.

---

## PHASE 1 — FIRST BOOT: SSH BOOTSTRAP & HEADLESS SERVER CONFIGURATION

After the first reboot following installation, bazzite-deck boots directly into Gaming Mode (Steam Big Picture). This is expected and intentional — the image is designed for living-room setups. Phase 1 converts the machine into a headless server during this very first boot, using the KDE Desktop Mode accessible from within Gaming Mode.

> ⚠️ **WARNING: Do ALL steps in this phase in sequence before rebooting.** The headless SDDM override in Step 3 must be in place before your next reboot. If you reboot before completing Step 3, the machine will boot to Gaming Mode again — which is not a disaster, simply repeat this phase.

### Step 1 · Enter KDE Desktop Mode from Gaming Mode

You will see Steam Big Picture on your TV or monitor. Use your controller or a mouse/keyboard to navigate to the Power menu (bottom-right power icon in Steam Big Picture). Select **Switch to Desktop**. KDE Plasma will launch on the physical display. This is a full KDE Plasma session — everything you need is available here.

> **NOTE:** Desktop Mode and Gaming Mode share the same underlying KDE installation. Desktop Mode is just a regular KDE Plasma session. Any configuration you do here persists across sessions.

### Step 2 · Open Konsole and enable SSH

In KDE, open Konsole (the terminal emulator) from the application menu or taskbar. SSH is not enabled by default on Bazzite. Enable it now:

```bash
# Enable and start the SSH daemon:
sudo systemctl enable --now sshd

# Find your server's local IP address (look for inet 192.168.x.x
# under your ethernet interface — usually eth0 or enp3s0):
ip a

# Note this IP address — you will need it throughout this guide.
# Example: 192.168.1.50
```

### Step 3 · Disable auto-suspend (CRITICAL for servers)

Bazzite may suspend the machine after inactivity, which silently kills all Docker containers. Permanently disable all suspend modes at the OS level before anything else.

**Step A — Disable in KDE System Settings (clean UX):** Open System Settings → Power Management → Energy Saving → On AC Power → set 'Suspend session' to 'Never'. Apply.

**Step B — Mask at the OS level (authoritative — do this too):**

```bash
# This is the hard guarantee — works even with no desktop session active.
# Step A is for clean UX; Step B is what actually prevents suspend.
sudo systemctl mask sleep.target suspend.target \
    hibernate.target hybrid-sleep.target

# Verify — all four must show 'masked':
systemctl status sleep.target suspend.target \
    hibernate.target hybrid-sleep.target | grep -E 'Loaded:|●'
```

### Step 4 · Apply the headless SDDM override (THE MOST CRITICAL STEP)

SDDM is the login screen manager. By default, bazzite-deck configures SDDM to autologin into Gaming Mode on every boot. For a headless server, we want no session active at boot. We create a configuration file that loads last (thanks to the `zzz-` prefix) and overrides the autologin with an empty value.

```bash
sudo mkdir -p /etc/sddm.conf.d

# Create the headless override file:
sudo tee /etc/sddm.conf.d/zzz-headless-server.conf > /dev/null << 'EOF'
[Autologin]
User=
Session=
Relogin=false
EOF

# Verify it was written correctly:
cat /etc/sddm.conf.d/zzz-headless-server.conf

# Apply it now without rebooting:
sudo systemctl restart sddm
# SDDM will now show a login screen or go dark (if no display is connected).
# Your KDE session may end — that is expected. SSH still works.
```

> **NOTE:** SDDM merges configuration files from `/usr/lib/sddm.conf.d/` (system defaults, including the bazzite-deck Gaming Mode autologin) and `/etc/sddm.conf.d/` (your overrides) in alphabetical order. Files in `/etc/` take precedence over `/usr/lib/`, and the `zzz-` prefix guarantees our file is the very last one processed. An empty `User=` field explicitly wins over any `User=` value earlier in the merge chain — this is SDDM's documented behavior.

### Step 5 · Verify the headless config survives a reboot

This is the moment of truth. Reboot the machine and verify it comes up headless.

```bash
# Reboot from the KDE terminal (or from SSH if you have already connected):
sudo systemctl reboot

# Wait ~60 seconds. The machine should NOT enter Gaming Mode.
# Now connect from your personal computer using the temporary port 22:
ssh username@192.168.1.50

# Confirm no graphical session is running on the physical display (seat0):
loginctl list-sessions
# Expected output: empty, or only your SSH session with no 'seat0' entry.

# If Gaming Mode appeared on the TV: the override was not applied correctly.
# SSH in, re-run Step 4, and reboot again.
```

> **If you cannot SSH in on port 22:** If sshd was not enabled before the reboot, boot back to Desktop Mode via Gaming Mode Power menu → Switch to Desktop, re-run Step 2, and then Step 5.

---

## PHASE 2 — ROUTER & NETWORK PREPARATION

These steps require access to your router's admin panel (typically at `192.168.1.1` or `192.168.0.1` in your browser). All steps are done on your personal computer.

### Step 1 · Assign a static local IP to your server (DHCP reservation)

By MAC address, assign your server a permanent local IP in your router's DHCP reservation settings. This is sometimes called 'Static Lease' or 'Address Reservation'. This ensures port-forwarding rules never break when the router reboots. The IP you noted in Phase 1 Step 2 (e.g., `192.168.1.50`) is what you're reserving.

### Step 2 · Set up port forwarding

Forward these external ports to your server's local IP:

| Port / Protocol | Purpose |
|---|---|
| 80 TCP | Required by Let's Encrypt for SSL certificate issuance (HTTP challenge) |
| 443 TCP | Your Nextcloud's public HTTPS entry point |
| 53222 TCP | SSH — our custom hardened port (configured in Phase 4) |
| 3478 TCP+UDP | Required ONLY if you plan to enable Nextcloud Talk (video/audio chat) later |

> ⚠️ **NEVER forward port 3389 (RDP).** Port 3389 must never be forwarded or opened in firewalld. krdp is only accessible via SSH tunnel. See Phase 9 for details.

### Step 3 · Create your DuckDNS account

Go to [duckdns.org](https://duckdns.org) and sign in with a Google or GitHub account. Create a free subdomain (e.g., `yourname.duckdns.org`) and copy your Token. Note both the subdomain name and the token — you will need them in Phase 7.

---

## PHASE 3 — SYSTEM PACKAGE INSTALLATION

All packages needed for the rest of this guide are installed here in a single rpm-ostree deployment. This phase runs on port 22 (before SSH hardening). The reason packages come before security hardening is critical: the SSH hardening phase needs semanage (from policycoreutils-python-utils) to label port 53222 for SELinux, and needs fail2ban to be installed before it can be enabled. Attempting to do security hardening before installing these tools causes both to silently fail.

### Step 1 · System auto-updates (no action needed)

Bazzite uses an immutable OS image: updates arrive as complete new images staged in the background and activated atomically on the next reboot. No unattended-upgrades needed. If an OS update ever causes an issue, one command returns to the previous working state:

```bash
rpm-ostree rollback && systemctl reboot
```

### Step 2 · Add the Docker repository

```bash
sudo curl -fsSL https://download.docker.com/linux/fedora/docker-ce.repo \
    -o /etc/yum.repos.d/docker-ce.repo
```

> **NOTE:** We use curl instead of `dnf config-manager` because dnf5's addrepo command had a parsing bug on earlier Fedora versions with Docker's .repo comment lines. The curl method is simpler and reliable across all Bazzite versions.

### Step 3 · Install all required packages (one reboot)

We check for the podman-docker shim and handle it conditionally. podman-docker provides `/usr/bin/docker` as a symlink to podman; if present, it conflicts with docker-ce-cli and must be removed atomically. The if/else handles both cases without requiring the user to investigate their system manually.

```bash
# Check whether the podman-docker compatibility shim is installed:
if rpm -q podman-docker &>/dev/null; then
    # It IS present — remove it and install Docker CE in one atomic operation:
    rpm-ostree override remove podman-docker \
        --install docker-ce --install docker-ce-cli --install containerd.io \
        --install docker-buildx-plugin --install docker-compose-plugin \
        --install fail2ban --install policycoreutils-python-utils
else
    # Not present — plain install:
    rpm-ostree install docker-ce docker-ce-cli containerd.io \
        docker-buildx-plugin docker-compose-plugin \
        fail2ban policycoreutils-python-utils
fi

systemctl reboot
```

> **NOTE:** Why not `ujust install-docker`? Bazzite's ujust install-docker has a confirmed bug (GitHub #2130, Jan 2025): the Docker service fails to start after install and docker-compose is not included. The rpm-ostree method is the correct path for Nextcloud AIO, which requires the native Docker socket for multi-container orchestration — rootless Podman's Docker API compatibility is insufficient. policycoreutils-python-utils provides semanage — needed in the very next phase to label the SSH port for SELinux, and again in Phase 6 for Nextcloud data labeling.

---

## PHASE 4 — IRONCLAD SECURITY (SELinux Port · SSH Keys · firewalld · Fail2ban)

This phase hardens SSH access and configures the firewall. It runs after Phase 3 because it requires semanage (from policycoreutils-python-utils) and fail2ban — both now installed. Step 1 runs on the server via SSH on port 22. Step 2 runs on your personal computer. Steps 3–5 run on the server via SSH on port 22.

### Step 1 · Label port 53222 in the SELinux policy (Server — MUST BE FIRST)

Bazzite runs SELinux in enforcing mode. By default, only port 22 is labelled `ssh_port_t` — the type that permits sshd to bind. If sshd is restarted on port 53222 without this label, SELinux silently blocks the bind and sshd stops listening entirely, locking you out permanently. This command adds port 53222 to the `ssh_port_t` set. It is persistent across reboots.

```bash
# Add port 53222 to SELinux's allowed SSH port list:
sudo semanage port -a -t ssh_port_t -p tcp 53222

# Verify — output must show '53222' in the ssh_port_t line:
sudo semanage port -l | grep ssh
# Expected: ssh_port_t    tcp    53222, 22
```

> ⚠️ **Do not skip or reorder this step.** sshd will fail to bind if you restart it on port 53222 before this SELinux label is applied. The server becomes unreachable. Recovery requires physical console access.

### Step 2 · Generate and copy SSH keys (Personal Computer)

SSH key authentication is the only login method after this phase. Passwords are disabled permanently. If you already have an SSH key pair on your personal computer, skip the ssh-keygen line and go straight to ssh-copy-id.

```bash
# Run on your PERSONAL COMPUTER:
# Generate a modern ed25519 key pair if you don't already have one.
# ed25519 is shorter, faster to verify, and more resistant to brute-force
# than the legacy RSA default:
ssh-keygen -t ed25519

# Copy your public key to the server (still on temporary port 22):
ssh-copy-id username@192.168.1.50

# Test key login works before proceeding:
ssh username@192.168.1.50
```

### Step 3 · Create the SSH hardening drop-in config (Server)

We use a drop-in file in `/etc/ssh/sshd_config.d/` instead of editing `sshd_config` directly. On Fedora Atomic, OS updates perform a 3-way merge of `/etc` files. If a future Fedora release changes `sshd_config` and you have also edited it, a merge conflict could silently revert your hardening. Drop-in files are never touched by OS updates.

```bash
sudo nano /etc/ssh/sshd_config.d/99-hardening.conf
```

Paste the entire block below:

```
Port 53222
PubkeyAuthentication yes
PasswordAuthentication no
PermitRootLogin no
MaxAuthTries 3
LoginGraceTime 20
AllowUsers YOUR_USERNAME
```

> ⚠️ **Replace `YOUR_USERNAME` with your actual Bazzite login name.** Run `whoami` in your SSH session to confirm it. AllowUsers ensures no other account can log in via SSH even with a valid key.

```bash
# Validate syntax — no output means no errors:
sudo sshd -t

# DO NOT restart sshd yet — open the firewall port first (Step 4).
```

| Option | Purpose |
|---|---|
| `Port 53222` | Moves SSH off port 22, eliminating the vast majority of automated scan traffic. |
| `PubkeyAuthentication yes` | Explicitly enables key login — prevents silent override by other configs. |
| `PasswordAuthentication no` | Only your SSH key can log in. Passwords are fully disabled. |
| `PermitRootLogin no` | Blocks direct root access — one of the most common attack vectors. |
| `MaxAuthTries 3` / `LoginGraceTime 20` | Limits the brute-force window before a connection is dropped. |
| `AllowUsers your_name` | Only your account can log in remotely. All other system users are blocked. |

### Step 4 · Configure firewalld and restart sshd in the correct order

firewalld is pre-installed and active on Bazzite. The SELinux label is already in place (Step 1). Now open the firewall port, restart sshd, verify it works, then remove port 22. Never skip the verification step — if something went wrong, port 22 is your safety net.

```bash
# Step 4a — Open port 53222 in the firewall BEFORE restarting sshd:
sudo firewall-cmd --permanent --add-port=53222/tcp
sudo firewall-cmd --reload

# Step 4b — NOW restart sshd (SELinux label + firewall rule both in place):
sudo systemctl restart sshd
```

**Step 4c — TEST NOW. Open a new terminal on your personal computer:**

```bash
# On your PERSONAL COMPUTER (new terminal window):
ssh -p 53222 username@192.168.1.50
# Do not proceed until this succeeds.
```

```bash
# Step 4d — Remove port 22 only after confirming port 53222 works:
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --reload

# Verify: port 22 gone, 53222/tcp present:
sudo firewall-cmd --list-all

# Confirm SELinux port label is in effect:
sudo semanage port -l | grep ssh
# Expected: ssh_port_t    tcp    53222, 22
```

> **NOTE:** firewalld on Bazzite enforces rules on both IPv4 and IPv6. Confirm that ports appear without an `(ipv4)` qualifier in `firewall-cmd --list-all` output. Docker bypasses firewalld for published ports (80, 443) — it injects nftables rules directly. Only the SSH port requires a firewalld rule.

### Step 5 · Configure and enable Fail2ban

Fail2ban watches the systemd journal for failed SSH authentication attempts and bans attacking IPs via firewalld. fail2ban is now installed (Phase 3), so this step succeeds.

```bash
sudo nano /etc/fail2ban/jail.local
```

Paste exactly:

```ini
[sshd]
enabled = true
port = 53222
ignoreip = 127.0.0.1/8 192.168.0.0/16 10.0.0.0/8
backend = systemd
banaction = firewallcmd-rich-rules
```

```bash
sudo systemctl enable --now fail2ban

# Verify jail is active and monitoring port 53222:
sudo fail2ban-client status sshd
```

> **NOTE:**
> - `backend = systemd`: Fedora uses the systemd journal, not `/var/log/auth.log`.
> - `banaction = firewallcmd-rich-rules`: Bans issued through firewalld's rich rules API cover both IPv4 and IPv6 — equivalent to `banaction = ufw` on Debian/Ubuntu.
> - `ignoreip`: Exempts localhost and common private subnets (`192.168.x.x`, `10.x.x.x`) from being banned. Without this, accidentally fat-fingering your key passphrase from your own local laptop could lock you out of your own server. Adjust the subnets if your LAN uses a different private range.

---

## PHASE 5 — DOCKER INITIALIZATION

All remaining steps run on the server via SSH on port 53222. Docker is installed (Phase 3) but not yet enabled or configured. This phase activates it, adds your user to the docker group, configures log rotation, and creates the shared network all services will use.

### Step 1 · Enable Docker and add your user to the docker group

```bash
ssh -p 53222 username@192.168.1.50

# Enable Docker daemon at boot and start it now:
# (Docker does NOT auto-start after install on Fedora/RPM-based systems)
sudo systemctl enable --now docker

# Add your user to the docker group (allows running docker without sudo):
sudo usermod -aG docker $USER

# Log out and back in for the group change to take effect:
exit
ssh -p 53222 username@192.168.1.50
```

### Step 2 · Prevent Docker logs from filling your disk

Docker logs grow indefinitely by default and will silently fill your disk over weeks. Configure log rotation before starting any containers.

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

```bash
sudo systemctl restart docker
# Each container is now capped at ~30 MB of logs total (3 files x 10 MB).
```

### Step 3 · Create the shared Docker network

Nginx Proxy Manager and Nextcloud AIO communicate over a dedicated internal Docker network. Creating it first ensures both Compose stacks can reference it. This keeps the Nextcloud backend completely off the host network stack.

```bash
docker network create nextcloud-aio
# This command only needs to be run once.
# Docker persists the network until you explicitly remove it.
```

---

## PHASE 6 — DATA DRIVE SETUP

> ⚠️ **Complete this before starting any Nextcloud containers.** The `NEXTCLOUD_DATADIR` path must exist, be mounted, and have the correct SELinux label before AIO's first start. It cannot be changed afterward without a full data migration.

### Step 1 · Format, mount, then label the data drive

The ordering of these commands matters critically for SELinux. We must mount the filesystem first, then apply the SELinux label. If you run `restorecon` before mounting, you label the empty mount-point directory on the OS drive. When the ext4 partition is mounted over it, the mount root inode inherits the filesystem's own label (usually `unlabeled_t`), not the one you applied. Nextcloud containers will then fail with permission denied errors.

```bash
# Identify your drives (find your data drive — NOT the OS drive):
lsblk

# All commands below use /dev/nvme1n1 as an example.
# Replace with your actual device name from lsblk output.

# Create a GPT partition table (WARNING: destroys all existing data on this drive):
sudo parted /dev/nvme1n1 --script mklabel gpt

# Create a single partition spanning the entire drive:
sudo parted /dev/nvme1n1 --script mkpart primary ext4 0% 100%

# Refresh the kernel's partition table view:
sudo partprobe /dev/nvme1n1

# Format the partition (note: nvme1n1p1, not nvme1n1):
sudo mkfs.ext4 /dev/nvme1n1p1

# Create the mount point:
sudo mkdir -p /srv/nextcloud-data

# Get the partition UUID for stable fstab mounting:
sudo blkid /dev/nvme1n1p1

# Add to /etc/fstab (replace UUID= value with the output from blkid above):
sudo nano /etc/fstab
# Add this line at the bottom:
# UUID=your-uuid  /srv/nextcloud-data  ext4  defaults  0 2

# MOUNT FIRST — the SELinux label must be applied to the live mounted filesystem,
# not to the empty mount-point directory on the OS drive:
sudo mount -a

# Verify the drive is mounted:
df -h /srv/nextcloud-data

# NOW apply the persistent SELinux file-context rule:
sudo semanage fcontext -a -t container_file_t "/srv/nextcloud-data(/.*)?"

# Apply the rule to the mounted filesystem:
sudo restorecon -Rv /srv/nextcloud-data

# Verify the label was applied correctly:
ls -Z /srv/nextcloud-data
# Expected: system_u:object_r:container_file_t:s0  /srv/nextcloud-data
```

> **NOTE:**
>
> **Why `semanage fcontext` instead of `chcon`?** `chcon` applies a label to existing files but the label is lost if the filesystem is relabelled during Fedora Atomic OS updates. `semanage fcontext` writes a persistent policy rule; `restorecon` applies it. Any newly created files under `/srv/nextcloud-data` automatically inherit `container_file_t`, and future relabels will correctly restore it.
>
> **Why mount before restorecon?** The mounted filesystem's root inode has its own SELinux extended attributes. `restorecon` must run on the live mounted filesystem to label those inodes correctly — not on the empty directory underneath the mount point.

---

## PHASE 7 — DEPLOYING SERVICES (DuckDNS · NPM · Nextcloud AIO)

### Step 1 · DuckDNS auto-updater

Keeps your domain pointed at your home IP. Uses a named volume — no bind mounts, no SELinux changes needed.

```bash
mkdir -p ~/docker/duckdns && cd ~/docker/duckdns
nano docker-compose.yml
```

```yaml
services:
  duckdns:
    image: lscr.io/linuxserver/duckdns:latest
    container_name: duckdns
    restart: unless-stopped
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    environment:
      - SUBDOMAINS=yourname        # subdomain only, not the full domain
      - TOKEN=your-token-here
      - TZ=Europe/Berlin           # e.g. America/New_York, Asia/Tokyo
```

```bash
docker compose up -d
```

### Step 2 · Nginx Proxy Manager (NPM)

Handles SSL certificates and routes external HTTPS traffic to Nextcloud. Port 81 (admin panel) is bound to `127.0.0.1` — accessible only via SSH tunnel (Phase 10).

```bash
mkdir -p ~/docker/npm && cd ~/docker/npm
nano docker-compose.yml
```

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    ports:
      - '80:80'              # internet access
      - '443:443'            # HTTPS
      - '127.0.0.1:81:81'   # admin panel: localhost only
    networks:
      - nextcloud-aio
    volumes:
      - ./data:/data:z               # :z = SELinux shared label
      - ./letsencrypt:/etc/letsencrypt:z

networks:
  nextcloud-aio:
    external: true   # uses the network created in Phase 5
```

```bash
docker compose up -d
```

> **NOTE:** NPM and Nextcloud share the `nextcloud-aio` Docker bridge network. NPM reaches Nextcloud's Apache container by container name (`nextcloud-aio-apache:11000`) without going through the host network stack. Ports 80 and 443 are managed by Docker directly — never add them to firewalld.

### Step 3 · Nextcloud All-in-One

Nextcloud AIO requires Docker to orchestrate multiple child containers. It connects to the Docker socket — this is why Podman is not a supported alternative. The mastercontainer is placed on the `nextcloud-aio` network.

```bash
mkdir -p ~/docker/nextcloud && cd ~/docker/nextcloud
nano docker-compose.yml
```

```yaml
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
      - label:disable          # SELinux: allows Docker socket access
    labels:
      - "com.centurylinklabs.watchtower.enable=false"
    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - '127.0.0.1:8080:8080'  # setup panel: localhost only
    networks:
      - nextcloud-aio
    environment:
      - APACHE_PORT=11000
      - APACHE_IP_BINDING=127.0.0.1   # port 11000 on localhost only
      - NEXTCLOUD_DATADIR=/srv/nextcloud-data
      # Uncomment ONLY if your router lacks NAT loopback and Pi-hole is not an option:
      # - SKIP_DOMAIN_VALIDATION=true

networks:
  nextcloud-aio:
    external: true
```

```bash
docker compose up -d
```

> **NOTE:** Why `:ro` on the Docker socket? The `:ro` flag means the container cannot delete or replace the socket file itself on the host filesystem. All bidirectional API communication — including starting and stopping child containers — still flows through it normally. `APACHE_IP_BINDING=127.0.0.1` means AIO publishes port 11000 only on loopback, preventing LAN or IPv6 access without needing to disable IPv6 system-wide.

---

## PHASE 8 — SESSION MANAGEMENT SCRIPTS

Three scripts control all graphical session transitions. They are called via SSH from any device — Termux on your phone, your laptop, your desktop. No daemon, no service, no scheduler. Each script is self-contained. The scripts are placed in `/usr/local/bin/` with `root:root` ownership — a mandatory security requirement for any script granted passwordless sudo access (explained in Step 1). All three scripts require root to manipulate SDDM config files and call `systemctl`/`loginctl`.

> **NOTE — How the SDDM one-shot pattern works:** The start scripts write a temporary autologin config, restart SDDM (which reads it and launches the session), then restore the headless override 7 seconds later in a background subshell. After those 7 seconds the headless config is back in place — all future SDDM restarts (including from session logouts) find `User=` empty and show the idle login screen. See Appendix F for the complete technical explanation of why each step is necessary.

### Step 1 · Understand the security model for these scripts

We are granting passwordless root execution (NOPASSWD) to these scripts via sudoers. This means that if a script file is writable by the regular user, any compromised process running as that user could overwrite it with arbitrary commands and then invoke sudo to execute them as root — a complete privilege escalation. The fix is simple: scripts granted NOPASSWD sudo must be owned by root and stored in a path the regular user cannot write to.

We write the scripts to `/usr/local/bin/` with `root:root` ownership. On Fedora Atomic, `/usr/local/bin/` is a mutable, persistent directory (unlike the read-only `/usr/bin/`). Only root can modify files in it, which is exactly the protection needed.

```bash
# Confirm your username — you will need it in SDDM_USER inside each script.
whoami
```

### Step 2 · Write start-kde.sh

Replace `yourusername` with your actual Bazzite login name in the `SDDM_USER` variable.

```bash
sudo nano /usr/local/bin/start-kde.sh
```

Paste the following as the complete file content, then save and exit nano (Ctrl+X, Y, Enter):

```bash
#!/bin/bash
# start-kde.sh — Launch a KDE Wayland session on the physical display.
# Refuses if a graphical session is already active on seat0.
# Location: /usr/local/bin/start-kde.sh  (root-owned, not user-writable)
# Usage: ssh -p 53222 user@host 'sudo start-kde.sh'

SDDM_USER="yourusername"         # ← REPLACE with your Bazzite username
KDE_SESSION="plasma"             # matches /usr/share/wayland-sessions/plasma.desktop
LAUNCH_CONF="/etc/sddm.conf.d/zz-session-launch.conf"
HEADLESS_CONF="/etc/sddm.conf.d/zzz-headless-server.conf"

log() { echo "[start-kde $(date '+%H:%M:%S')] $*"; }

# Refuse if a seat0 session is already active
ACTIVE=$(loginctl list-sessions --no-legend 2>/dev/null \
    | awk '{print $1}' \
    | while read -r sid; do
        seat=$(loginctl show-session "$sid" -p Seat  --value 2>/dev/null) || continue
        state=$(loginctl show-session "$sid" -p State --value 2>/dev/null) || continue
        if [[ "$seat" == "seat0" ]] && \
           [[ "$state" == "active" || "$state" == "online" ]]; then
            echo "$sid"; break
        fi
    done)

if [[ -n "$ACTIVE" ]]; then
    log "ERROR: A graphical session ($ACTIVE) is already active on seat0."
    log "Log out of the current session first, then re-run this script."
    exit 1
fi

# Remove headless override and write one-shot autologin config
rm -f "$HEADLESS_CONF"
mkdir -p /etc/sddm.conf.d
cat > "$LAUNCH_CONF" << EOF
[Autologin]
User=${SDDM_USER}
Session=${KDE_SESSION}
Relogin=false
EOF
log "SDDM autologin config written for KDE Wayland."

# Restart SDDM — reads config and autologins to KDE
log "Restarting SDDM..."
systemctl reset-failed sddm
systemctl restart sddm

# Restore headless config in background after SDDM has started.
# SDDM reads config once on startup. The 7-second wait is conservative —
# long enough that SDDM has reliably started even under I/O load.
# stdout/stderr are redirected to /dev/null so the SSH connection
# returns to the caller immediately rather than hanging until the
# subshell exits. See Appendix F for the full technical explanation.
(
    sleep 7
    rm -f "$LAUNCH_CONF"
    cat > "$HEADLESS_CONF" << EOF
[Autologin]
User=
Session=
Relogin=false
EOF
) >/dev/null 2>&1 &
disown $!

log "KDE Wayland session launching on physical display."
```

After saving and exiting nano, set ownership and permissions in the terminal:

```bash
# Set root ownership and correct permissions:
sudo chown root:root /usr/local/bin/start-kde.sh
sudo chmod 755 /usr/local/bin/start-kde.sh
# chmod 755 = owner (root) can read/write/execute;
#             group and others can read/execute (needed for sudo to run it).
# chown root:root = only root can modify this file. Your user account cannot.
```

### Step 3 · Write start-gaming.sh

Same SDDM pattern, but for the gamescope Gaming Mode session. Replace `yourusername`.

```bash
sudo nano /usr/local/bin/start-gaming.sh
```

Paste the following as the complete file content, then save and exit nano (Ctrl+X, Y, Enter):

```bash
#!/bin/bash
# start-gaming.sh — Launch Gaming Mode on the physical display.
# REFUSES if any graphical session is active on seat0.
# Log out of KDE (or any session) first using its own logout button.
# Location: /usr/local/bin/start-gaming.sh  (root-owned, not user-writable)
# Usage: ssh -p 53222 user@host 'sudo start-gaming.sh'

SDDM_USER="yourusername"         # ← REPLACE with your Bazzite username
GAMING_SESSION="gamescope-session-plus"
LAUNCH_CONF="/etc/sddm.conf.d/zz-session-launch.conf"
HEADLESS_CONF="/etc/sddm.conf.d/zzz-headless-server.conf"

log() { echo "[start-gaming $(date '+%H:%M:%S')] $*"; }

# Refuse if a seat0 session is already active
ACTIVE=$(loginctl list-sessions --no-legend 2>/dev/null \
    | awk '{print $1}' \
    | while read -r sid; do
        seat=$(loginctl show-session "$sid" -p Seat  --value 2>/dev/null) || continue
        state=$(loginctl show-session "$sid" -p State --value 2>/dev/null) || continue
        if [[ "$seat" == "seat0" ]] && \
           [[ "$state" == "active" || "$state" == "online" ]]; then
            echo "$sid"; break
        fi
    done)

if [[ -n "$ACTIVE" ]]; then
    log "ERROR: A session ($ACTIVE) is already active on seat0."
    log "Log out using the session's own logout button, then re-run this script."
    exit 1
fi

# Verify Gaming Mode session file is present
SESSION_FILE="/usr/share/wayland-sessions/${GAMING_SESSION}.desktop"
if [[ ! -f "$SESSION_FILE" ]]; then
    log "ERROR: Session file not found: $SESSION_FILE"
    log "Ensure bazzite-deck was installed (not the standard bazzite image)."
    exit 1
fi

# Remove headless override and write one-shot autologin config
rm -f "$HEADLESS_CONF"
mkdir -p /etc/sddm.conf.d
cat > "$LAUNCH_CONF" << EOF
[Autologin]
User=${SDDM_USER}
Session=${GAMING_SESSION}
Relogin=false
EOF
log "SDDM autologin config written for Gaming Mode."

# Restart SDDM
log "Restarting SDDM..."
systemctl reset-failed sddm
systemctl restart sddm

# Restore headless config in background.
# stdout/stderr redirected to /dev/null so SSH returns immediately.
(
    sleep 7
    rm -f "$LAUNCH_CONF"
    cat > "$HEADLESS_CONF" << EOF
[Autologin]
User=
Session=
Relogin=false
EOF
) >/dev/null 2>&1 &
disown $!

log "Gaming Mode launching on physical display."
```

After saving and exiting nano, set ownership and permissions in the terminal:

```bash
sudo chown root:root /usr/local/bin/start-gaming.sh
sudo chmod 755 /usr/local/bin/start-gaming.sh
```

### Step 4 · Write stop-session.sh

Emergency fallback — terminates whichever graphical session is active on seat0. Also restores the headless SDDM config as a safety net. For normal use, log out via KDE's own logout button or the Power menu in Steam Big Picture.

```bash
sudo nano /usr/local/bin/stop-session.sh
```

Paste the following as the complete file content, then save and exit nano (Ctrl+X, Y, Enter):

```bash
#!/bin/bash
# stop-session.sh — Emergency: terminate whichever seat0 session is active.
# Normal use: log out via KDE's logout button or Steam's Power menu.
# Location: /usr/local/bin/stop-session.sh  (root-owned, not user-writable)
# Usage: ssh -p 53222 user@host 'sudo stop-session.sh'

LAUNCH_CONF="/etc/sddm.conf.d/zz-session-launch.conf"
HEADLESS_CONF="/etc/sddm.conf.d/zzz-headless-server.conf"

log() { echo "[stop-session $(date '+%H:%M:%S')] $*"; }

# Safety net: always restore headless config first
rm -f "$LAUNCH_CONF"
cat > "$HEADLESS_CONF" << EOF
[Autologin]
User=
Session=
Relogin=false
EOF
log "SDDM configs confirmed in headless state."

# Find and terminate the active seat0 session
ACTIVE=$(loginctl list-sessions --no-legend 2>/dev/null \
    | awk '{print $1}' \
    | while read -r sid; do
        seat=$(loginctl show-session "$sid" -p Seat  --value 2>/dev/null) || continue
        state=$(loginctl show-session "$sid" -p State --value 2>/dev/null) || continue
        if [[ "$seat" == "seat0" ]] && \
           [[ "$state" == "active" || "$state" == "online" ]]; then
            echo "$sid"; break
        fi
    done)

if [[ -n "$ACTIVE" ]]; then
    log "Terminating session $ACTIVE on seat0..."
    loginctl terminate-session "$ACTIVE"
    log "Done. SDDM login screen should appear on the physical display."
else
    log "No active seat0 session found. Nothing to terminate."
fi

log "WARNING: All unsaved work in the terminated session is lost."
```

After saving and exiting nano, set ownership and permissions in the terminal:

```bash
sudo chown root:root /usr/local/bin/stop-session.sh
sudo chmod 755 /usr/local/bin/stop-session.sh
```

### Step 5 · Configure targeted sudo for the scripts

The scripts need root to manipulate SDDM config files and call `systemctl` and `loginctl`. We grant passwordless sudo for only these three specific paths in `/usr/local/bin/`. Because the files are root-owned, your regular user account cannot modify them — the privilege escalation vector is closed.

```bash
# Open a new sudoers drop-in file using the safe visudo editor:
sudo visudo -f /etc/sudoers.d/session-scripts

# Paste these three lines.
# Replace yourusername with your actual username (run 'whoami' to confirm).
yourusername ALL=(ALL) NOPASSWD: /usr/local/bin/start-kde.sh
yourusername ALL=(ALL) NOPASSWD: /usr/local/bin/start-gaming.sh
yourusername ALL=(ALL) NOPASSWD: /usr/local/bin/stop-session.sh
```

```bash
# Save and exit visudo. Then validate the file syntax:
sudo visudo -c -f /etc/sudoers.d/session-scripts
# Expected: /etc/sudoers.d/session-scripts: parsed OK
```

> **NOTE:** Why `visudo -c` after saving? When you use `sudo visudo -f /etc/sudoers.d/...` (editing a drop-in file directly rather than the main sudoers), visudo performs its syntax check on save for the main file but NOT always for drop-in files edited with `-f`. Running `visudo -c` explicitly confirms the drop-in parses correctly. A broken sudoers file does not lock you out of the system immediately — sudo simply stops working until fixed. Recovery: boot into single-user/recovery mode and delete the broken file from `/etc/sudoers.d/`.

### Step 6 · Test the scripts manually

```bash
# Launch KDE Wayland on the physical display:
sudo start-kde.sh
# Expected: KDE appears on monitor or TV within a few seconds.
# The SSH terminal returns your prompt immediately (no 7-second hang).

# While KDE is running, try to launch Gaming Mode (must be refused):
sudo start-gaming.sh
# Expected: ERROR message — session already active. Gaming Mode does NOT launch.

# From the physical display: log out of KDE using its own logout button.
# Then from SSH:
sudo start-gaming.sh
# Expected: Gaming Mode (Steam Big Picture) appears on the physical display.

# Test emergency stop:
sudo stop-session.sh
# Expected: Gaming Mode terminates. SDDM login screen appears.

# Confirm both session files exist:
ls /usr/share/wayland-sessions/
# Must include: gamescope-session-plus.desktop  and  plasma.desktop

# Confirm scripts are root-owned (security verification):
ls -la /usr/local/bin/start-kde.sh \
       /usr/local/bin/start-gaming.sh \
       /usr/local/bin/stop-session.sh
# Expected: -rwxr-xr-x 1 root root ...  (owner = root, not your username)
```

> **NOTE:** Restarting SDDM never affects your SSH connection. SDDM manages the physical display (seat0) and SSH is a completely independent systemd service. Your terminal stays connected and responsive throughout all session transitions.

---

## PHASE 9 — KRDP REMOTE DESKTOP

krdp is KDE's native Wayland RDP server, integrated into KDE Plasma 6.1 and later. It shares the running KDE Wayland compositor session — no virtual framebuffer, no X11, no extra packages required. Monitors can be off; krdp captures the Wayland compositor output regardless of physical display power state.

> ⚠️ **SECURITY: krdp listens on all network interfaces (`0.0.0.0`), not just localhost.** Despite what some documentation suggests, krdp does NOT bind to `127.0.0.1` by default. It listens on `QHostAddress::Any` (all interfaces), meaning port 3389 is reachable on your LAN IP unless blocked. firewalld blocks it because we never add port 3389 to any allowed rule. Step 2 includes an explicit verification of this. **NEVER open port 3389 in firewalld under any circumstances.** Access is exclusively via SSH tunnel.

### Step 1 · Enable krdp in KDE System Settings

This step requires a running KDE session on the physical display. Run `start-kde.sh` first if KDE is not already active:

```bash
sudo start-kde.sh
```

Then, at the physical display or monitor:

| Step | Action |
|---|---|
| 1. Open System Settings | Navigate to: System Settings → Remote Desktop |
| 2. Enable RDP | Toggle 'Enable Remote Desktop' to ON |
| 3. Set credentials | Set a username and password for RDP login. These can differ from your system login — they are krdp-specific credentials. |
| 4. Check the port | Default port is 3389. Leave it as-is. |
| 5. Apply | Click Apply. krdp begins listening immediately on all interfaces. |

### Step 2 · Verify firewall protection (mandatory after enabling krdp)

After enabling krdp, verify that port 3389 is NOT open in firewalld. This confirms that krdp is protected by the firewall despite listening on all interfaces.

```bash
# Confirm krdp is listening (should show 0.0.0.0:3389 or *:3389):
ss -tlnp | grep 3389
# Expected: a line showing port 3389 is listening.

# Verify firewalld is NOT allowing port 3389 from the LAN:
sudo firewall-cmd --list-all
# Search the output for '3389' — it must NOT appear in 'ports:' or 'services:'.

# If 3389 appears: remove it immediately with:
# sudo firewall-cmd --permanent --remove-port=3389/tcp && sudo firewall-cmd --reload

# As a final test, try connecting directly from another LAN device to
# 192.168.1.50:3389 WITHOUT an SSH tunnel — it must be refused.
```

### Step 3 · Connect via SSH tunnel from your personal computer

All RDP access flows through the SSH tunnel on port 53222. Open a terminal on your personal computer and run the tunnel command:

```bash
# On your PERSONAL COMPUTER — keep this terminal open while working remotely:
ssh -N -L 3389:127.0.0.1:3389 -p 53222 username@your-server-ip

# Then connect any RDP client to: localhost:3389
# Use the krdp credentials set in System Settings (Step 1).
```

| RDP Client | Platform |
|---|---|
| Remote Desktop Connection (built-in) | Windows → connect to: `localhost:3389` |
| Microsoft Remote Desktop | macOS → connect to: `localhost:3389` |
| Remmina | Linux → connect to: `localhost:3389` |
| Microsoft Remote Desktop | Android — run the SSH tunnel in Termux first, then connect |

> ⚠️ **Keep the SSH tunnel terminal open.** If the SSH tunnel terminal is closed, the RDP session disconnects immediately.

> **NOTE:** krdp only runs while a KDE Wayland session is active. It starts automatically each time KDE starts (the setting persists). When KDE is not running — for example during Gaming Mode — krdp is not running and RDP connections will be refused. This is by design.

---

## PHASE 10 — LINK & INITIALIZE NEXTCLOUD

### Step 1 · Access admin panels via SSH tunnel

The admin panels are bound to `127.0.0.1` — invisible to your local network. Access them by creating an SSH tunnel from your personal computer. Run the relevant command and leave it running while you use the panel.

```bash
# NPM admin panel → http://localhost:8081
ssh -N -L 8081:127.0.0.1:81 -p 53222 username@192.168.1.50

# Nextcloud AIO setup panel → https://localhost:9443
ssh -N -L 9443:127.0.0.1:8080 -p 53222 username@192.168.1.50
```

> **NOTE:** `-N` means 'do not execute a remote command' (tunnel only). `-L` binds a local port to a remote address through the encrypted SSH connection.

### Step 2 · Configure Nginx Proxy Manager

Start the NPM tunnel (first command above), then open `http://localhost:8081`. Log in with `admin@example.com` / `changeme` — change both immediately. Go to Hosts → Proxy Hosts → Add Proxy Host:

| Field | Value |
|---|---|
| Domain Names | `yourname.duckdns.org` |
| Scheme | `http` |
| Forward Hostname / IP | `nextcloud-aio-apache` (the Docker container name — not an IP address) |
| Forward Port | `11000` |
| Toggles | Cache Assets, Block Common Exploits, Websockets Support — all ON |
| SSL tab | Request new SSL Certificate · Force SSL · HTTP/2 Support · Agree to ToS → Save |

> **NOTE:** The Forward Hostname is the Docker container name `nextcloud-aio-apache`, not an IP. NPM and Nextcloud are on the same `nextcloud-aio` Docker network, so NPM resolves the container name directly. NPM saves the proxy host configuration regardless of whether the backend is currently running.

### Step 3 · Fix local network access (NAT loopback)

Many routers cannot route traffic from your local network back through their own public IP. `yourname.duckdns.org` resolves to your public IP, but the connection fails from inside your home network even though it works from the internet.

**Solution 1 — Router local DNS override (recommended):** In your router's settings, find 'Local DNS', 'DNS Entries', or 'Static Hosts' and add: `yourname.duckdns.org → 192.168.1.50`. External devices continue using DuckDNS normally.

**Solution 2 — Pi-hole local DNS:** Add the same DNS override in Pi-hole's 'Local DNS Records' if your router does not support custom DNS entries.

**Solution 3 — SKIP_DOMAIN_VALIDATION (last resort):** Edit the Nextcloud AIO compose file, uncomment the `SKIP_DOMAIN_VALIDATION` line, then:

```bash
cd ~/docker/nextcloud
docker compose down && docker compose up -d
```

> ⚠️ **Client `/etc/hosts` files do NOT solve this.** Adding `yourname.duckdns.org` to `/etc/hosts` on your PC only helps your browser. It has no effect on the domain validation request, which originates from inside the Docker container on the server — not from your client device.

### Step 4 · Initialize Nextcloud AIO

Stop the NPM tunnel (Ctrl+C). Start the Nextcloud AIO tunnel (second command in Step 1). Go to `https://localhost:9443` and accept the browser security warning (expected — this is AIO's self-signed certificate, not your real domain cert). Copy the setup password shown on screen. Enter your domain: `yourname.duckdns.org`. The data directory is already configured via `NEXTCLOUD_DATADIR`. Choose optional containers and click Start Containers. Wait 5–10 minutes for Nextcloud to pull and configure all components.

---

## PHASE 11 — TESTING & BACKUPS

### DNS Test

```bash
# Bypasses your local DNS override and queries Google DNS directly.
# The result must be your home's public IP.
nslookup yourname.duckdns.org 8.8.8.8
```

### External Test

Turn off Wi-Fi on your phone and visit `https://yourname.duckdns.org` over cellular. A green padlock and the Nextcloud login screen confirms end-to-end functionality.

### Session Management Test

```bash
# From any SSH client — verify all three scripts work:
ssh -p 53222 username@your-server-ip 'sudo start-kde.sh'
# KDE appears on the physical display. SSH terminal returns promptly.

ssh -p 53222 username@your-server-ip 'sudo start-gaming.sh'
# Must output ERROR — KDE is still active.

# Log out of KDE via its logout button, then:
ssh -p 53222 username@your-server-ip 'sudo start-gaming.sh'
# Gaming Mode appears on the physical display.

ssh -p 53222 username@your-server-ip 'sudo stop-session.sh'
# Gaming Mode terminates. SDDM login screen appears.
```

### System State Validation Checklist

Run these checks after completing all phases to confirm every subsystem is correctly configured. Each check has a specific expected output — if any differ, the corresponding phase has a problem and its troubleshooting section in Appendix A applies.

```bash
# 1. Headless SDDM default — only the headless config should be present:
ls /etc/sddm.conf.d/
# Expected: zzz-headless-server.conf  (only this file, nothing else after idle boot)

# 2. No active graphical session at idle:
loginctl list-sessions
# Expected: empty output or only your SSH session with no seat0 entry

# 3. SELinux port label — 53222 must be in the ssh_port_t line:
sudo semanage port -l | grep ssh
# Expected: ssh_port_t    tcp    53222, 22

# 4. Nextcloud data drive SELinux label:
ls -Z /srv/nextcloud-data
# Expected: system_u:object_r:container_file_t:s0

# 5. Docker running without sudo (confirms group membership active):
docker info 2>&1 | grep 'Server Version'
# Expected: Server Version: (no permission denied error)

# 6. Fail2ban active and watching port 53222:
sudo fail2ban-client status sshd
# Expected: 'Status for the jail: sshd' with 'Port list: 53222'

# 7. Scripts owned by root (privilege escalation protection):
ls -la /usr/local/bin/start-kde.sh \
       /usr/local/bin/start-gaming.sh \
       /usr/local/bin/stop-session.sh
# Expected: -rwxr-xr-x 1 root root ...  for all three files

# 8. All containers healthy:
docker ps --format 'table {{.Names}}\t{{.Status}}'
# Expected: duckdns, nginx-proxy-manager, nextcloud-aio-mastercontainer all 'Up'
```

### Backups

Inside the AIO setup panel (`https://localhost:9443`) you will find the built-in Borg Backup tool. Configure it before storing any personal data in Nextcloud.

```bash
# Set a destination path, e.g. /mnt/backups or a mounted external USB drive.
# Set backups to run daily.
# Perform a test restore immediately after the first backup completes.
```

> ⚠️ **Perform a test restore once a month.** A backup you have never tested cannot be trusted. Schedule monthly restore tests.

---

## APPENDIX A — TROUBLESHOOTING

### Machine boots to Gaming Mode after initial install (expected once)

```bash
# SSH is not available yet. Go to Gaming Mode → Power menu → Switch to Desktop.
# Enable SSH and apply the headless SDDM override (Phase 1 Steps 2–4).
```

### start-kde.sh or start-gaming.sh: 'session already active' but nothing visible on screen

A stale logind session may be present. List sessions, then terminate by session ID:

```bash
loginctl list-sessions
# Note the session ID from the output (e.g. "3" or "c1").
# If a seat0 session shows as 'closing' or 'lingering', force terminate it:
loginctl terminate-session SESSION_ID   # replace SESSION_ID with the actual ID
sleep 2
# Re-run the desired start script.
sudo start-kde.sh   # or: sudo start-gaming.sh
```

### SDDM shows login screen instead of autologining

```bash
# Check the exact session name (must match the .desktop filename without extension):
ls /usr/share/wayland-sessions/
# KDE:    plasma.desktop             → Session=plasma
# Gaming: gamescope-session-plus.desktop → Session=gamescope-session-plus

# Confirm the variable in your script matches exactly:
grep 'KDE_SESSION='    /usr/local/bin/start-kde.sh
grep 'GAMING_SESSION=' /usr/local/bin/start-gaming.sh

# Check SDDM logs (note: the service name is 'sddm', not 'ssdm'):
journalctl -u sddm -n 50 --no-pager
```

### krdp RDP connection refused

```bash
# 1. Confirm KDE is running — krdp only runs with an active KDE session:
loginctl list-sessions
# Must show a seat0 session.

# 2. Confirm krdp is listening:
ss -tlnp | grep 3389
# Must show port 3389 is listening (on all interfaces — this is normal).

# 3. If nothing: re-check Remote Desktop is enabled in KDE System Settings.

# 4. Confirm the SSH tunnel is open on your personal computer:
# The 'ssh -N -L 3389:127.0.0.1:3389' terminal must still be running.
```

### Docker containers not running after a reboot

```bash
# Check if Docker is enabled and running:
sudo systemctl status docker

# If not enabled:
sudo systemctl enable --now docker

# Restart all Docker stacks:
cd ~/docker/npm       && docker compose up -d
cd ~/docker/nextcloud && docker compose up -d
cd ~/docker/duckdns   && docker compose up -d

# Verify all containers are healthy:
docker ps --format 'table {{.Names}}\t{{.Status}}'
```

### zz-session-launch.conf persists after a script run (power loss edge case)

This means the 7-second background cleanup was interrupted. Manual fix:

```bash
sudo rm -f /etc/sddm.conf.d/zz-session-launch.conf
sudo tee /etc/sddm.conf.d/zzz-headless-server.conf > /dev/null << 'EOF'
[Autologin]
User=
Session=
Relogin=false
EOF

# Verify clean state:
ls /etc/sddm.conf.d/
# Expected: only zzz-headless-server.conf
```

---

## APPENDIX B — NORMAL WORKFLOWS (QUICK REFERENCE)

### I want to use KDE on the TV or monitor

```bash
ssh -p 53222 username@your-server-ip 'sudo start-kde.sh'
# KDE Wayland session appears on the physical display.
# To stop: Application Menu → Leave → Log Out.
# Server returns to headless idle.
```

### I want to work remotely via RDP

```bash
# Step 1 — Start KDE if not already running:
ssh -p 53222 username@your-server-ip 'sudo start-kde.sh'

# Step 2 — Open the SSH tunnel (keep this terminal open):
ssh -N -L 3389:127.0.0.1:3389 -p 53222 username@your-server-ip

# Step 3 — Connect your RDP client to: localhost:3389
# Physical monitors can be off — krdp works regardless.
```

### I want to game

```bash
# If KDE is running: log out first using KDE's own logout button.
# Then from SSH:
ssh -p 53222 username@your-server-ip 'sudo start-gaming.sh'
# Gaming Mode appears on the physical display.
# To stop: Power menu in Steam Big Picture → Exit → Exit to Desktop.
# Server returns to headless idle.
```

### I need to force-stop a stuck session

```bash
ssh -p 53222 username@your-server-ip 'sudo stop-session.sh'
# WARNING: All unsaved work in the terminated session is lost.
```

### Verify what is currently running

```bash
ssh -p 53222 username@your-server-ip

# Check active sessions:
loginctl list-sessions
# seat0 entry = a physical display session is active
# No seat0 entry = server is in headless idle

# Identify which session type is running:
loginctl show-session -p Type -p Desktop
# Type=wayland, Desktop=KDE    → KDE Wayland session
# Type=wayland, Desktop=gamescope → Gaming Mode

# Check SDDM config state:
ls /etc/sddm.conf.d/
# Healthy headless state: only zzz-headless-server.conf present
# If zz-session-launch.conf is also present: the 7s background cleanup
# has not yet completed. Wait a moment and check again.
```

---

## APPENDIX C — OPTIONAL HOME ASSISTANT INTEGRATION

Because the scripts are plain SSH commands, adding Home Assistant control requires only a few lines in `configuration.yaml` — zero changes to the scripts themselves. The HA container needs a dedicated SSH keypair to authenticate with the server. This keypair must be generated on the host and mounted into the HA container.

### Step 1 · Generate a dedicated SSH keypair for Home Assistant

```bash
# Create the target directory first — ssh-keygen will fail if it does not exist:
mkdir -p ~/docker/ai-home/homeassistant-data/.ssh

# Generate a dedicated key for HA — keep it separate from your personal key.
# We place it in the HA data directory so it persists across container rebuilds.
ssh-keygen -t ed25519 -f ~/docker/ai-home/homeassistant-data/.ssh/id_ed25519_ha -N ''

# Authorise this key on the server (adds it to your ~/.ssh/authorized_keys):
cat ~/docker/ai-home/homeassistant-data/.ssh/id_ed25519_ha.pub >> ~/.ssh/authorized_keys

# Pre-populate known_hosts so HA does not fail on first connection:
ssh-keyscan -p 53222 127.0.0.1 >> ~/docker/ai-home/homeassistant-data/.ssh/known_hosts

# Lock down permissions on the .ssh directory:
chmod 700 ~/docker/ai-home/homeassistant-data/.ssh
chmod 600 ~/docker/ai-home/homeassistant-data/.ssh/id_ed25519_ha
chmod 644 ~/docker/ai-home/homeassistant-data/.ssh/id_ed25519_ha.pub
chmod 644 ~/docker/ai-home/homeassistant-data/.ssh/known_hosts
```

> **NOTE:** The key is stored inside the HA data directory (`homeassistant-data/.ssh/`) rather than at `/root/.ssh` on the host. This way the HA container accesses it via its mounted data volume — no separate volume mount is needed and the key persists through container updates.

### Step 2 · Add the SSH key volume to the HA compose file

The HA container must be able to read the key. Since the key lives inside the `homeassistant-data` directory, which is already mounted as a volume, it is automatically accessible inside the container at `/config/.ssh/`. No additional volume entry is needed — but the path inside `configuration.yaml` must use the container-internal path `/config/.ssh/id_ed25519_ha`.

### Step 3 · Add shell commands to configuration.yaml

```bash
# In ~/docker/ai-home/homeassistant-data/configuration.yaml:
```

```yaml
shell_command:
  start_kde: >-
    ssh -i /config/.ssh/id_ed25519_ha
    -o StrictHostKeyChecking=yes
    -o UserKnownHostsFile=/config/.ssh/known_hosts
    -p 53222 yourusername@127.0.0.1
    'sudo start-kde.sh'
  start_gaming: >-
    ssh -i /config/.ssh/id_ed25519_ha
    -o StrictHostKeyChecking=yes
    -o UserKnownHostsFile=/config/.ssh/known_hosts
    -p 53222 yourusername@127.0.0.1
    'sudo start-gaming.sh'
  stop_session: >-
    ssh -i /config/.ssh/id_ed25519_ha
    -o StrictHostKeyChecking=yes
    -o UserKnownHostsFile=/config/.ssh/known_hosts
    -p 53222 yourusername@127.0.0.1
    'sudo stop-session.sh'
```

```bash
docker restart homeassistant
# Then create buttons in your HA dashboard calling each shell_command.
```

> **NOTE:** Why `127.0.0.1`? Home Assistant uses `network_mode: host` and shares the host's network namespace. Port 53222 on the host is reachable from inside the HA container as `127.0.0.1:53222`. Why `UserKnownHostsFile`? The HA process runs as root inside the container. Without an explicit known_hosts path, SSH looks in `/root/.ssh/known_hosts`, which does not exist — causing StrictHostKeyChecking to refuse the connection. The explicit path points to the pre-populated file we created in Step 1.

---

## APPENDIX D — OS UPDATES & ROLLBACKS

Bazzite stages OS updates silently in the background. Applying them requires a reboot — during which your server is offline for 1–2 minutes. All containers come back up automatically on reboot (`restart: always` / `restart: unless-stopped`). Schedule reboots during low-usage hours.

```bash
# If an update causes any problem, one command returns to the previous state:
rpm-ostree rollback && systemctl reboot
```

Bazzite retains OS images for the last 90 days. You can rebase to any specific build:

```bash
# List available stable builds:
skopeo list-tags docker://ghcr.io/ublue-os/bazzite-deck \
    | grep -- 'stable-' | sort -rV

# Rebase to a specific build:
rpm-ostree rebase ostree-image-signed:docker://ghcr.io/ublue-os/bazzite-deck:stable-YEARMONTHDAY
```

### Watchtower — Auto-update containers

Docker containers do not update themselves. Watchtower automates this using opt-in mode: it only touches containers marked with `enable=true` (DuckDNS and NPM). Nextcloud AIO containers are intentionally excluded and updated manually via the AIO panel.

```bash
mkdir -p ~/docker/watchtower && cd ~/docker/watchtower
nano docker-compose.yml
```

```yaml
services:
  watchtower:
    image: containrrr/watchtower
    restart: unless-stopped
    security_opt:
      - label:disable          # SELinux: allows Docker socket access
    environment:
      - TZ=Europe/Berlin
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: --label-enable --cleanup --schedule "0 0 4 * * *"
    # Only updates containers with enable=true (DuckDNS, NPM).
    # All Nextcloud AIO containers are untouched. Runs at 04:00 local time.
```

```bash
docker compose up -d
```

---

## APPENDIX E — STORAGE ROADMAP (SnapRAID + MergerFS)

This appendix describes the recommended storage expansion path when you add additional HDDs. It does not apply to initial setup. SnapRAID + MergerFS is the recommended approach for a home server with mixed-size drives: unlike RAID1 or ZFS, it works with drives of different sizes, has no continuous write overhead (parity computed once daily), and files remain independently accessible on individual drives.

```bash
# 1. Add the mergerfs Copr repository (not in official Fedora repos):
sudo curl -fsSL \
    "https://copr.fedorainfracloud.org/coprs/errornointernet/mergerfs/repo/fedora-$(rpm -E %fedora)/errornointernet-mergerfs-fedora-$(rpm -E %fedora).repo" \
    -o /etc/yum.repos.d/mergerfs.repo

# 2. Install MergerFS + SnapRAID:
rpm-ostree install mergerfs snapraid
systemctl reboot

# 3. Format and configure HDDs (parted + mkfs.ext4 as in Phase 6)

# 4. Set up MergerFS unified pool: /mnt/disk1 + /mnt/disk2 → /srv/media

# 5. Configure SnapRAID with one disk as parity

# 6. Migrate Nextcloud data to the new pool:
cd ~/docker/nextcloud && docker compose stop
sudo rsync -aHAX /srv/nextcloud-data/ /srv/media/nextcloud/

# 7. Update NEXTCLOUD_DATADIR in compose file

# 8. Apply SELinux label to new path:
sudo semanage fcontext -a -t container_file_t "/srv/media/nextcloud(/.*)?"
sudo restorecon -Rv /srv/media/nextcloud

# 9. Start Nextcloud AIO:
docker compose up -d
```

---

## APPENDIX F — UNDERSTANDING YOUR STACK (DEEP DIVE)

### Why does stop-session.sh not kill the SSH session?

SSH runs as a systemd service (sshd) and SSH connections are logind sessions on a virtual terminal (TTY) with no assigned seat. `loginctl terminate-session` targets only the seat0 session — the one owning the physical display, keyboard, and mouse. Your SSH session has no seat assignment and is completely unaffected. You will see the script output and receive your prompt back normally.

### Why can't Gaming Mode and KDE run simultaneously?

Both are Wayland compositors competing for the same physical GPU and display output (seat0). A Wayland compositor takes exclusive ownership of the GPU's DRM device and output connectors via the DRM master mechanism. Only one process can hold DRM master on a given seat at a time. gamescope additionally uses direct scanout for its low-latency path, which requires exclusive GPU access. There is no configuration workaround — this is a fundamental hardware constraint.

### Why does krdp work with monitors off?

krdp captures frames directly from the KDE Wayland compositor via PipeWire's screen capture protocol. The compositor renders frames to an internal framebuffer regardless of whether a physical display is connected or in DPMS sleep. Monitor power state is irrelevant to compositor rendering. krdp streams these frames to your RDP client over the SSH tunnel.

### Why does logging out of KDE land at SDDM login, not Gaming Mode?

When KDE logs out, it exits the Wayland session and hands control back to SDDM. SDDM reads its config: since the headless override (`zzz-headless-server.conf`) was restored by the background cleanup 7 seconds after the start script ran, SDDM finds `User=` empty and shows a login screen rather than autologining again. The server is now in headless idle — this is correct and intentional.

### Why the 7-second background delay for config restore?

SDDM reads its configuration once during startup. The script must allow SDDM to start and process the autologin config before the launch config file is removed. If the headless override were written back before SDDM started, it would prevent autologin. The delay is a conservative blind wait — there is no reliable synchronous signal from SDDM that confirms it has read the config and initiated the session. 7 seconds was chosen over the original 5 seconds to accommodate systems under I/O load (e.g., a Nextcloud backup running at the same moment): on a slow HDD-based system with high concurrent I/O, SDDM's startup time can exceed 2 seconds. 7 seconds provides a comfortable margin while still restoring the headless config well before any realistic scenario that might inadvertently trigger another SDDM restart.

### Why does the SSH terminal return immediately after running the start scripts?

The background subshell that restores the headless config has its stdout and stderr redirected to `/dev/null` before being detached with `disown`. The SSH daemon on the server tracks open file descriptors to know when a remote command has finished — it closes the connection once all processes that inherited the original stdout/stderr have exited. By redirecting the subshell's output to `/dev/null`, it holds no reference to the SSH session's file descriptors, so the connection closes as soon as the main script body finishes — instantly, not after the 7-second sleep completes.

### Why bazzite-deck and not standard bazzite + rebase?

Starting with bazzite-deck directly from the ISO: (1) eliminates an entire rebase phase and the Docker reinstall that follows, (2) ensures gamescope-session-plus is part of the signed base image rather than a post-install addition, (3) means every subsequent OS update automatically includes Gaming Mode infrastructure, and (4) keeps the system on the signed image track from day one.

### Why is Docker installed via rpm-ostree rather than Podman?

Nextcloud AIO (All-in-One) manages multiple child containers by connecting to the Docker socket (`/var/run/docker.sock`). Its mastercontainer requires a native Docker API for this multi-container orchestration. Rootless Podman's Docker API compatibility is not sufficient for AIO's needs. rpm-ostree layering is the correct method for system-level packages on Fedora Atomic — the packages survive OS image updates.

### Why are the scripts in /usr/local/bin/ and owned by root?

The scripts have NOPASSWD sudo access — meaning sudo will execute them as root without asking for a password. If the script file itself were writable by the regular user (as it would be if stored in `~/scripts/`), any process running as that user — a compromised SSH session, a rogue Docker container, a malicious web app — could overwrite the script with arbitrary commands and then call sudo to execute them as root. That is a complete privilege escalation requiring no exploits at all.

`/usr/local/bin/` is chosen because it is a persistent, mutable directory on Fedora Atomic (unlike the immutable `/usr/bin/` which is part of the OS image). Files placed there survive OS updates. By setting ownership to `root:root` and permissions to `755`, only root can modify the files, while sudo and any user can read and execute them. The regular user account has no write access to `/usr/local/bin/` — the privilege escalation vector is closed at the filesystem level.

### Why must SELinux be updated before restarting sshd on port 53222?

SELinux operates on a type-enforcement model: every action a process attempts is checked against a policy that specifies which types a process of a given domain can use. The `ssh_port_t` type labels ports that sshd is permitted to bind to. Port 22 carries this label by default. Port 53222 does not — it has no label at all. When sshd (running in the `sshd_t` domain) attempts to `bind(2)` to port 53222, the SELinux kernel module intercepts the system call and denies it because port 53222 lacks the `ssh_port_t` label. sshd fails silently: it starts, but does not listen on any port, and the connection is refused. The remedy is `semanage port -a -t ssh_port_t -p tcp 53222`, which writes a persistent entry into the SELinux port context database. The label survives reboots and OS image updates. The verification command `semanage port -l | grep ssh` confirms the label is present before sshd is restarted.

### Why must the filesystem be mounted before running restorecon?

SELinux file contexts are stored as extended attributes (xattrs) on the filesystem inodes themselves — specifically in the `security.selinux` xattr namespace. When you run `mkdir /srv/nextcloud-data`, you create a directory inode on the OS drive. When you subsequently run `restorecon /srv/nextcloud-data`, it sets the xattr on that inode. However, when you mount the ext4 data partition with `mount -a`, the mount operation makes the new filesystem's root inode visible at `/srv/nextcloud-data`, completely replacing the OS drive's directory inode. The label you set earlier is now on a hidden inode beneath the mount — not on the mounted root. The ext4 partition's root inode carries its own xattr, which defaults to `unlabeled_t` on a freshly formatted filesystem. Nextcloud containers running in the `container_t` domain cannot write to `unlabeled_t` paths, resulting in permission denied errors. The fix — and the order enforced in Phase 6 — is: format, create mount point, add to fstab, mount, then run `semanage` + `restorecon` on the live mounted filesystem's root inode.

### Why does packages-before-security ordering matter?

This guide installs policycoreutils-python-utils (which provides semanage) and fail2ban in Phase 3, before the SSH hardening in Phase 4. This ordering is not arbitrary. The SSH hardening phase requires semanage to label port 53222 in the SELinux policy before sshd can be safely restarted. If semanage is not yet installed when that command runs, the command fails and the user either skips the SELinux step (causing a lockout when sshd restarts) or encounters a confusing error with no clear recovery path. Similarly, `systemctl enable --now fail2ban` in Phase 4 Step 5 requires the fail2ban binary to exist on disk. rpm-ostree deployments take effect only after a reboot, so the single reboot at the end of Phase 3 activates all tools needed by Phase 4 and beyond.

### Why does krdp listen on all interfaces?

krdp (the Plasma 6 integrated RDP server) uses `QHostAddress::Any`, which binds to all available network interfaces including the LAN interface. There is currently no configuration option in KDE System Settings to restrict the binding address to localhost only. This is a known limitation (KDE bug #515737). firewalld compensates by dropping all inbound connections to port 3389 from the LAN because we never add a firewalld rule allowing that port. The SSH tunnel on port 53222 provides the only path to reach krdp from outside the server — which goes through loopback and is therefore reachable even though krdp binds to all interfaces.

### Time synchronization

Incorrect system time silently breaks TLS certificate validation and Nextcloud login sessions. Bazzite uses `chronyd` as its default NTP daemon — not `systemd-timesyncd`. Never enable `systemd-timesyncd` on Bazzite; it conflicts with `chronyd`.

```bash
timedatectl
# Confirm: System clock synchronized: yes

systemctl status chronyd
# If chronyd is not running: sudo systemctl enable --now chronyd
```

### Disk usage monitoring

```bash
df -h
# When any partition hits 90%+ usage, act immediately.
# Clean old Docker images: docker system prune
```

---

*This guide is fully self-contained. All phases are additive — they do not conflict with one another. All Docker containers, volumes, network configuration, and firewall rules operate independently of graphical sessions and SDDM restarts.*
