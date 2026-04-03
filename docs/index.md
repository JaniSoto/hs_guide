# Bazzite Unified Home Server + Gaming Station

**SSH-first · Zero New Open Ports · Headless by Default**

This guide turns a single desktop PC into two things at once: a 24/7 home server running self-hosted cloud software, and an on-demand gaming station. The machine normally sits at a blank, idle login screen. You trigger either KDE Desktop or Steam Gaming Mode remotely over SSH whenever you want them — the machine returns to idle automatically. All server software runs in isolated Docker containers in the background, independent of any graphical session.

## What You Get

| Component | Description |
| :--- | :--- |
| **Headless server** | Boots to idle SDDM login screen. Docker containers run as systemd services. SSH always on port 22022. |
| **KDE Wayland** | Full KDE Plasma on the physical display, on demand via SSH. Required to launch Sunshine for streaming. |
| **Gaming Mode** | Steam Deck-style gamescope session. Direct GPU scanout, controller-native. |
| **WireGuard + Sunshine** | Hardware-accelerated game and desktop streaming over an encrypted VPN tunnel. No RDP. |
| **Nextcloud AIO** | Self-hosted cloud behind Nginx Proxy Manager with Let's Encrypt via DuckDNS. |

!!! warning "Recommended Hardware Requirements"
    * **CPU:** 4-core x86_64 minimum (6-core+ recommended)
    * **RAM:** 8 GB DDR4 minimum (16 GB+ recommended for Fulltextsearch)
    * **OS Drive:** 120 GB NVMe SSD minimum
    * **Data Drive:** Any size HDD or SSD (Dedicated, mounted at `/srv/nextcloud-data`)
    * **Display:** HDMI or DisplayPort dummy plug ($3–5) — **REQUIRED** for headless GPU init.
