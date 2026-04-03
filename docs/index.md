---
hide:
  - toc
---

<div class="hero">
  <p class="hero-eyebrow">Bazzite · Docker · Nextcloud · Sunshine</p>
  <h1>One PC. Full Home Server.<br>On-Demand Gaming Station.</h1>
  <p class="hero-sub">
    This guide turns a single desktop PC into a 24/7 home server — running
    self-hosted cloud software, a WireGuard VPN, and hardware-accelerated
    game streaming — all hardened, headless, and controlled over SSH.
  </p>
  <div class="hero-badges">
    <span class="hero-badge">🔒 SSH-First</span>
    <span class="hero-badge">🐳 Dockerised</span>
    <span class="hero-badge">📡 WireGuard VPN</span>
    <span class="hero-badge">🎮 Sunshine/Moonlight</span>
    <span class="hero-badge">☁️ Nextcloud AIO</span>
  </div>
</div>

## What You Get

<div class="feature-grid">
  <div class="feature-card">
    <span class="feature-icon">🖥️</span>
    <h3>Headless Server</h3>
    <p>Boots to an idle login screen. All Docker containers run independently in the background, 24/7.</p>
  </div>
  <div class="feature-card">
    <span class="feature-icon">🎮</span>
    <h3>Gaming Mode</h3>
    <p>Steam Deck-style gamescope session. Direct GPU scanout, controller-native, launched on demand via SSH.</p>
  </div>
  <div class="feature-card">
    <span class="feature-icon">🖱️</span>
    <h3>KDE Wayland Desktop</h3>
    <p>Full KDE Plasma on the physical display whenever you need it, started and stopped remotely.</p>
  </div>
  <div class="feature-card">
    <span class="feature-icon">📡</span>
    <h3>WireGuard VPN</h3>
    <p>Encrypted tunnel for Sunshine game streaming and secure remote access — no open RDP ports.</p>
  </div>
  <div class="feature-card">
    <span class="feature-icon">☁️</span>
    <h3>Nextcloud AIO</h3>
    <p>Self-hosted cloud storage behind Nginx Proxy Manager with automatic HTTPS via Let's Encrypt.</p>
  </div>
  <div class="feature-card">
    <span class="feature-icon">🔒</span>
    <h3>Hardened Security</h3>
    <p>SSH on a custom port with key-only auth, SELinux enforcing, firewalld, and Fail2ban.</p>
  </div>
</div>

---

## Minimum Hardware Requirements

<div class="req-grid">
  <div class="req-item">
    <span class="req-icon">💻</span>
    <div>
      <strong>CPU</strong>
      <span>4-core x86_64 minimum. 6-core+ recommended.</span>
    </div>
  </div>
  <div class="req-item">
    <span class="req-icon">🧠</span>
    <div>
      <strong>RAM</strong>
      <span>8 GB DDR4 minimum. 16 GB+ recommended for Fulltextsearch.</span>
    </div>
  </div>
  <div class="req-item">
    <span class="req-icon">💾</span>
    <div>
      <strong>OS Drive</strong>
      <span>120 GB NVMe SSD minimum. Fast read/write recommended.</span>
    </div>
  </div>
  <div class="req-item">
    <span class="req-icon">🗄️</span>
    <div>
      <strong>Data Drive</strong>
      <span>A dedicated second drive (any size HDD or SSD) for Nextcloud data.</span>
    </div>
  </div>
  <div class="req-item">
    <span class="req-icon">🖥️</span>
    <div>
      <strong>Dummy Plug</strong>
      <span><strong>Required.</strong> HDMI or DisplayPort dummy plug ($3–5) for headless GPU init.</span>
    </div>
  </div>
  <div class="req-item">
    <span class="req-icon">🌐</span>
    <div>
      <strong>GPU</strong>
      <span>AMD or Intel recommended. NVIDIA works but has limitations in Gaming Mode and Sunshine encoding.</span>
    </div>
  </div>
</div>

!!! tip "What's a Dummy Plug?"
    Without a real monitor connected, most GPUs refuse to initialise their display output. A dummy plug is a tiny $3 adapter that pretends to be a monitor — it tricks the GPU into thinking a screen is attached so Gaming Mode and Sunshine can capture the display correctly.

---

## Guide Roadmap

<div class="phase-grid">
  <a class="phase-card" href="before-you-begin/">
    <p class="card-label">Prerequisites</p>
    <p class="card-title">Before You Begin</p>
    <p class="card-desc">What to prepare, what to have on hand, and key concepts explained.</p>
  </a>
  <a class="phase-card" href="phases/phase-0-install/">
    <p class="card-label">Phase 0</p>
    <p class="card-title">OS Installation</p>
    <p class="card-desc">Download Bazzite, flash the USB, and install to your machine.</p>
  </a>
  <a class="phase-card" href="phases/phase-1-first-boot/">
    <p class="card-label">Phase 1</p>
    <p class="card-title">First Boot & SSH</p>
    <p class="card-desc">Enable SSH, disable sleep, and lock the machine into headless mode.</p>
  </a>
  <a class="phase-card" href="phases/phase-2-network/">
    <p class="card-label">Phase 2</p>
    <p class="card-title">Router & Network</p>
    <p class="card-desc">Set a static IP reservation and configure port forwarding.</p>
  </a>
  <a class="phase-card" href="phases/phase-3-packages/">
    <p class="card-label">Phase 3</p>
    <p class="card-title">System Packages</p>
    <p class="card-desc">Install Docker CE, Fail2ban, and other required packages.</p>
  </a>
  <a class="phase-card" href="phases/phase-4-security/">
    <p class="card-label">Phase 4</p>
    <p class="card-title">Security Hardening</p>
    <p class="card-desc">SSH keys, custom port, SELinux, firewalld, and Fail2ban.</p>
  </a>
  <a class="phase-card" href="phases/phase-5-docker/">
    <p class="card-label">Phase 5</p>
    <p class="card-title">Docker Setup</p>
    <p class="card-desc">Enable Docker, add your user to the group, and configure log limits.</p>
  </a>
  <a class="phase-card" href="phases/phase-6-drives/">
    <p class="card-label">Phase 6</p>
    <p class="card-title">Data Drive</p>
    <p class="card-desc">Format, mount, and apply SELinux labels to your Nextcloud data drive.</p>
  </a>
  <a class="phase-card" href="phases/phase-7-services/">
    <p class="card-label">Phase 7</p>
    <p class="card-title">Core Services</p>
    <p class="card-desc">Deploy DuckDNS, Nginx Proxy Manager, and Nextcloud AIO with Docker Compose.</p>
  </a>
  <a class="phase-card" href="phases/phase-8-scripts/">
    <p class="card-label">Phase 8</p>
    <p class="card-title">Session Scripts</p>
    <p class="card-desc">Write the scripts that start and stop KDE, Gaming Mode, and Sunshine.</p>
  </a>
  <a class="phase-card" href="phases/phase-9-streaming/">
    <p class="card-label">Phase 9</p>
    <p class="card-title">VPN & Streaming</p>
    <p class="card-desc">Deploy wg-easy, configure Sunshine, and connect Moonlight for game streaming.</p>
  </a>
  <a class="phase-card" href="phases/phase-10-nextcloud/">
    <p class="card-label">Phase 10</p>
    <p class="card-title">Nextcloud Init</p>
    <p class="card-desc">Link Nextcloud to NPM via SSH tunnels and complete the first-run setup.</p>
  </a>
  <a class="phase-card" href="phases/phase-11-testing/">
    <p class="card-label">Phase 11</p>
    <p class="card-title">Testing & Backups</p>
    <p class="card-desc">Validate the full setup with a checklist and configure Borg backups.</p>
  </a>
  <a class="phase-card" href="phases/phase-12-webhooks/">
    <p class="card-label">Phase 12</p>
    <p class="card-title">One-Click Client</p>
    <p class="card-desc">Add a webhook server and client script for one-click gaming sessions.</p>
  </a>
</div>

---

[Start: Before You Begin →](before-you-begin.md){ .next-phase }
