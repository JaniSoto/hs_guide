<div class="phase-header">
  <span class="phase-num">10</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 10 · Done via SSH tunnel in a browser</p>
    <h1>Linking & Initialising Nextcloud</h1>
    <p class="phase-header-sub">Connect NPM to Nextcloud via SSH tunnels and complete the first-run setup.</p>
  </div>
</div>

!!! info "Why SSH tunnels for admin panels?"
    The admin panels for NPM (`port 81`) and Nextcloud AIO (`port 9443`) are bound to `127.0.0.1` — the server's own localhost. This means they are completely invisible to your local network and the internet. To reach them from your personal computer, you create an SSH tunnel — a secure forwarded port that makes the server's localhost port appear on your own machine.

---

## Step 1 — Open SSH Tunnels

**Open two new terminals on your personal computer** (keep them running in the background while you work):

```bash
# Terminal 1 — NPM admin panel at http://localhost:8081
ssh -N -L 8081:127.0.0.1:81 -p 22022 username@192.168.1.50
```

```bash
# Terminal 2 — Nextcloud AIO setup panel at https://localhost:9443
ssh -N -L 9443:127.0.0.1:9443 -p 22022 username@192.168.1.50
```

The `-N` flag means "don't open a shell, just forward the port." These terminals will appear to hang — that's correct.

---

## Step 2 — Configure Nginx Proxy Manager

Open **`http://localhost:8081`** in your browser.

Log in with the default credentials:

- **Email:** `admin@example.com`
- **Password:** `changeme`

**Change these immediately** after logging in.

Then go to **Hosts → Proxy Hosts → Add Proxy Host** and configure it:

!!! danger "Fill in ALL tabs before clicking Save"
    NPM silently resets the SSL tab if you save early. Configure Details, SSL, and Advanced tabs **in the same session** before clicking Save.

**Details tab:**

| Setting | Value |
|---------|-------|
| Domain Names | `yourname.duckdns.org` |
| Scheme | `http` |
| Forward Hostname/IP | `nextcloud-aio-apache` |
| Forward Port | `11000` |
| Cache Assets | **Off** |
| Block Common Exploits | **On** |
| Websockets Support | **On** |

**SSL tab:**

- Request a new SSL certificate
- Force SSL: **On**
- HTTP/2 Support: **On**
- Agree to Let's Encrypt Terms of Service

**Advanced tab** — paste this configuration:

```nginx
client_max_body_size 0;
proxy_read_timeout 36000s;
proxy_send_timeout 36000s;
proxy_request_buffering off;
proxy_hide_header Upgrade;
```

Click **Save**.

!!! note "502 Bad Gateway is expected right now"
    Visiting `yourname.duckdns.org` in your browser right now will show a 502 error — because the Nextcloud container hasn't been started yet. This is normal. You'll fix it in the next step.

---

## Step 3 — Fix Local Network Access (if needed)

If you're accessing your domain from inside your home network and it doesn't work, your router may not support "NAT loopback" (routing traffic back to your own network). Try these fixes in order:

1. **Router Local DNS** — In your router admin panel, add a DNS override: `yourname.duckdns.org → 192.168.1.50` (your server's local IP).
2. **Pi-hole** — If you use Pi-hole, add a Local DNS Record there instead.
3. **Hosts file (last resort)** — Add `192.168.1.50 yourname.duckdns.org` to `/etc/hosts` on each local device.

---

## Step 4 — Initialise Nextcloud AIO

Open **`https://localhost:9443`** in your browser and accept the self-signed certificate warning.

1. Copy the generated **AIO passphrase** and store it safely
2. Enter your domain: `yourname.duckdns.org`
3. Select the latest **Nextcloud Hub** version
4. Optionally enable add-ons (Fulltextsearch requires 16 GB+ RAM; Imaginary speeds up thumbnail generation)
5. Click **Start Containers**

The containers will download and start — this takes several minutes. Wait until all show a green **(healthy)** status before proceeding.

After all containers are healthy, open `https://yourname.duckdns.org` in your browser to complete the Nextcloud admin account setup.

---

## ✅ Phase 10 Complete

Nextcloud is live at `https://yourname.duckdns.org` with a valid Let's Encrypt certificate.

[Next: Phase 11 — Testing & Backups →](phase-11-testing.md){ .next-phase }
