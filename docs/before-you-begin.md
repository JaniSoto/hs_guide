# Before You Begin

This page covers everything you should know and prepare **before** running a single command. Take 10 minutes here — it will save you hours later.

---

## What This Guide Builds

You will end up with a single PC that does two things at once:

- **Always on (24/7):** The machine runs headless — no monitor, no graphical interface active. Docker containers serving Nextcloud, a reverse proxy, a VPN, and automatic updates all run silently in the background.
- **On demand:** From any device you SSH in and run one command to launch KDE Plasma or Steam Gaming Mode. Sunshine then streams the display to Moonlight. When you're done, you run another command and the machine returns to idle. You can also use webhooks to run the scripts remotly over the VPN on any web browser.

```
Normal state:       Machine idle at blank SDDM login screen
                    └── All Docker services running in background

When you want to:   ssh into server → run sudo start-kde.sh
stream Desktop           └── KDE launches → Sunshine captures it → stream via Moonlight

When you want to:   ssh into server → run sudo start-gaming.sh  
stream games             └── Steam Gaming Mode launches → Sunshine captures it → stream via Moonlight
```

---

## Software & Tools to Install on Your Personal Computer

Before starting, install these on the **PC or laptop you'll use to SSH into the server**:

| Tool | Purpose | Where to Get |
|------|----------|--------------|
| **SSH client** | Connect to the server from your computer | Built-in on Linux/macOS/Windows 10+ |
| **Ventoy (Recommended)**, **Fedora Media Writer** or **Balena Etcher** | Flash the Bazzite ISO to a USB drive | [ventoy.net](https://www.ventoy.net/) · [fedoraproject.org](https://fedoraproject.org/en/workstation/download) · [etcher.balena.io](https://etcher.balena.io) |
| **Moonlight** | Stream games/desktop from the server | [moonlight-stream.org](https://moonlight-stream.org) |
| **WireGuard** | VPN client for remote access | [wireguard.com/install](https://www.wireguard.com/install/) |

---

## Accounts You'll Need

- **DuckDNS account** — free dynamic DNS at [duckdns.org](https://www.duckdns.org). You'll get a free subdomain like `yourname.duckdns.org` that always points to your home's IP address, even when it changes.

---

## Key Concepts for Beginners

Don't worry if these are new to you — each concept is explained again where it's used. This is just a quick primer.

??? note "What is SSH?"
    SSH (Secure Shell) is how you control the server remotely from your personal computer — like having a terminal window directly on the server. Once SSH is set up, you never need a keyboard or monitor plugged into the server again.

    ```
    ssh -p 22022 yourusername@192.168.1.50
    ```
    Think of it as: `ssh -p [port] [username]@[server IP]`

??? note "What is Docker?"
    Docker packages software into isolated "containers" that run independently of the host OS. Each service (Nextcloud, the VPN, the reverse proxy) lives in its own container. They can be started, stopped, or updated without affecting each other or the OS.

    `docker-compose.yml` files describe how a container should run — what image to use, what ports to open, what volumes to mount.

??? note "What is a Reverse Proxy (Nginx Proxy Manager)?"
    You have one public IP address, but potentially many services. A reverse proxy sits in front of everything — it receives requests coming into port 443 (HTTPS), looks at the domain name, and routes them to the right internal service. This is how `yourname.duckdns.org` reaches your Nextcloud container internally.

??? note "What is WireGuard?"
    WireGuard is a fast, modern VPN protocol. When your phone or laptop connects to your WireGuard VPN, it's as if they're on your home network — even if you're on the other side of the world. This is how Sunshine/Moonlight game streaming works securely without exposing Sunshine ports to the internet.

??? note "What is Sunshine/Moonlight?"
    Sunshine is a game streaming server that runs on your server. Moonlight is the client app on your phone, tablet, laptop, smart TV or Steam Deck. Together they stream your gaming session over the network with low latency — better than RDP or VNC because they use hardware-accelerated video encoding and UDP.

??? note "What is SELinux?"
    SELinux (Security-Enhanced Linux) is a mandatory access control system built into Fedora/Bazzite. It enforces strict rules about which programs can access which files and ports. Bazzite ships with it enabled, and this guide keeps it on. A few steps in the guide apply "SELinux labels" — this just tells SELinux that a specific file or port is allowed to be used by a specific service.

??? note "What is an Immutable OS (Bazzite/rpm-ostree)?"
    Bazzite is built on an "immutable" base OS — the system files are read-only and cannot be changed the usual way. This makes the OS extremely stable and reliable. The trade-off: you can't use `dnf install` directly. Instead, you use `rpm-ostree install` to "layer" extra packages on top. The packages survive OS updates. A reboot is required after installing packages.

---

## Understanding the File Structure You'll Create

By the end of this guide, your home directory will look like this:

```
~/docker/
├── duckdns/
│   ├── .env              ← Your DuckDNS token (kept secret)
│   └── docker-compose.yml
├── npm/
│   ├── data/
│   ├── letsencrypt/
│   └── docker-compose.yml
├── nextcloud/
│   └── docker-compose.yml
├── wireguard/
│   ├── data/
│   └── docker-compose.yml
└── watchtower/
    └── docker-compose.yml

/srv/nextcloud-data/      ← Your data drive mount point
/usr/local/bin/
├── start-kde.sh          ← Launch KDE Plasma session
├── start-gaming.sh       ← Launch Steam Gaming Mode
└── stop-session.sh       ← Return server to idle
```

---

## Keeping Track of Your Values

Throughout this guide, you'll create passwords, tokens, and IP addresses that you'll need again later. Keep a **local, private note** (not in the cloud yet — Nextcloud isn't set up!) with these:

| Value | Your Entry |
|-------|-----------|
| Server local IP | `192.168.___.___` |
| SSH username | |
| DuckDNS subdomain | `_______.duckdns.org` |
| DuckDNS token | |
| WireGuard admin password | |
| NPM admin email / password | |
| Nextcloud AIO passphrase | |

---

[Start: Phase 0 — OS Installation →](phases/phase-0-install.md){ .next-phase }
