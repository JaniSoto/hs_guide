<div class="phase-header">
  <span class="phase-num">2</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 2 · Done in your router's admin panel</p>
    <h1>Router & Network Preparation</h1>
    <p class="phase-header-sub">Give your server a permanent local IP and open only the ports it needs.</p>
  </div>
</div>

!!! info "How to access your router's admin panel"
    Open a browser and go to `192.168.1.1` or `192.168.0.1` (try both). Log in with your router's admin credentials — often found on a sticker on the router itself.

---

## Step 1 — Set a Static IP (DHCP Reservation)

By default, your router assigns IP addresses dynamically — your server's IP could change after a restart, breaking all your SSH connections and port-forward rules.

**DHCP Reservation** solves this: you tell the router "always give this specific device this specific IP address." The device connects normally, but always gets the same IP.

In your router's admin panel, find the section called **DHCP Reservations**, **Static Leases**, or **Address Binding** (the name varies by router brand). Add a new reservation:

- **MAC Address:** your server's network card MAC address (shown in the output of `ip route get 1.1.1.1` in Phase 1)
- **IP Address:** pick an IP outside the DHCP range — usually something like `192.168.1.50` or `192.168.178.50`

!!! tip "Finding your MAC address"
    SSH into your server and run: `ip link show`. Look for the interface that starts with `en` (e.g. `enp3s0`) or `eth`. The MAC address is listed after `link/ether`, e.g. `aa:bb:cc:dd:ee:ff`.

---

## Step 2 — Set Up Port Forwarding

Your router has one public IP address shared between all your devices via NAT. Port forwarding tells the router: "when a connection arrives on port X, send it to the server."

Open the **Port Forwarding** or **Virtual Servers** section in your router and add these rules:

| Port / Protocol | Forward To | Purpose |
|-----------------|------------|---------|
| **80 / TCP** | Your server IP | Let's Encrypt certificate issuance (HTTP-01 challenge) |
| **443 / TCP** | Your server IP | Nextcloud HTTPS — all user traffic |
| **22022 / TCP** | Your server IP | SSH — hardened custom port (set up in Phase 4) |
| **51820 / UDP** | Your server IP | WireGuard VPN tunnel |

!!! warning "Never forward Sunshine ports"
    Do **NOT** forward ports 47984, 47989, 47990, 48010 (Sunshine's ports) at the router. Sunshine is only reachable remotely via the WireGuard VPN tunnel — exposing it directly to the internet is a security risk.

!!! tip "Port 22 (standard SSH)"
    You don't need to forward port 22. We'll move SSH to port 22022 in Phase 4. You can forward 22022 now, or wait until after Phase 4.

---

## ✅ Phase 2 Complete

Your server now has a permanent local IP address and the correct ports are forwarded. All SSH connections for the rest of this guide use the static IP you just set.

[Next: Phase 3 — System Packages →](phase-3-packages.md){ .next-phase }
