<div class="phase-header">
  <span class="phase-num">0</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 0 · Done at the physical machine</p>
    <h1>OS Installation</h1>
    <p class="phase-header-sub">Download Bazzite, flash a USB drive, and install the OS.</p>
  </div>
</div>

!!! info "What you'll need for this phase"
    - A USB drive (10 GB or more free space) — contents will be erased
    - A keyboard, monitor, and internet connection plugged into the server
    - Fedora Media Writer **or** Balena Etcher installed on your personal PC (alternatively you can use Ventoy. Follow the instructions from their [website](https://www.ventoy.net/en/doc_start.html)

---

## Step 1 — Download the Correct Bazzite ISO

Go to **[bazzite.gg](https://bazzite.gg)** and click the **Download** button. Use the Image Picker tool to select the right image for your hardware:

| Setting | Choose |
|---------|--------|
| **Hardware** | Desktop / HTPC |
| **GPU** | AMD/Intel *or* NVIDIA (match your hardware) |
| **Desktop** | KDE Plasma |
| **Steam Gaming Mode** | ✅ Enabled |

!!! warning "Make sure you pick `bazzite-deck`"
    There are two Bazzite variants. The one with Steam Gaming Mode enabled is called `bazzite-deck`. It ships **both** KDE Plasma and the gamescope Steam session in a single signed image. Starting with `bazzite-deck` means you never need to manually add Gaming Mode later.

After downloading, **flash the ISO to your USB drive** using Fedora Media Writer or Balena Etcher or just move the ISO it to your Ventoy USB drive.

---

## Step 2 — Install Bazzite

Boot the server from the USB drive (usually by pressing `F11` or `F12` at startup to choose the boot device).

The Anaconda installer will appear. Work through the setup screens:

- **Username & Password** — choose a simple lowercase username (you'll type it often)
- **Hostname** — under 20 characters, no spaces (e.g. `homeserver`)
- **Timezone & Keyboard** — set these correctly, especially timezone (it affects Docker containers)
- **Installation Destination** — select your OS drive. Do **not** select your data drive.
- Network is **not required** during installation.

!!! danger "If you have a Windows drive in this machine"
    Physically **disconnect** your Windows drive (unplug its SATA/NVMe cable) before installing Bazzite. Anaconda can accidentally modify the Windows bootloader even if you don't select that drive. Reconnect it after installation if needed.

---

## Step 3 — Note Your GPU Type

After installation, you'll know which GPU you have. Here's what matters for this guide:

| GPU Type | Gaming Mode | VA-API Encoding (Sunshine) | Notes |
|----------|-------------|---------------------------|-------|
| **AMD** | ✅ Full support | ✅ Supported | Recommended for this guide |
| **Intel** | ✅ Full support | ✅ Supported | Good for lower-power builds |
| **NVIDIA** | ⚠️ Beta | ❌ Not supported | Sunshine must use software encoding |

---

## ✅ Phase 0 Complete

Your machine now has Bazzite installed and boots into Steam Gaming Mode.

!!! tip "First boot — don't panic"
    After installation, the machine will boot straight into Steam's Gaming Mode. This is normal and expected. Phase 1 walks you through switching to Desktop Mode and setting up SSH so you never need to sit at the machine again.

!!! warning "Lock **BIOS**, disable **boot from USB drive** and enable **secure boot**"
    It's recommended to set an admin password in your BIOS and disable boot from USB for a more secure and robust setup. This process can be different depending on the device. To enable secure boot, follow the ["Bazzite Documentation: Secure Boot Guide"](https://docs.bazzite.gg/General/Installation_Guide/secure_boot/)

[Next: Phase 1 — First Boot & SSH →](phase-1-first-boot.md){ .next-phase }
