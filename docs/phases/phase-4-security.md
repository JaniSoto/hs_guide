<div class="phase-header">
  <span class="phase-num">4</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 4 · Run over SSH</p>
    <h1>Security Hardening</h1>
    <p class="phase-header-sub">SSH key authentication, custom port, SELinux labels, firewalld, and Fail2ban.</p>
  </div>
</div>

!!! danger "Critical ordering — do NOT skip ahead"
    This phase has a strict sequence. Specifically, **the SELinux label for port 22022 must be applied before you restart sshd**, and **you must test the new SSH port in a separate terminal before closing your existing session**. Doing these out of order will lock you out of the server.

---

## Step 1 — Label Port 22022 in SELinux

SELinux controls which ports `sshd` is allowed to bind to. By default, only port 22 has the `ssh_port_t` label. If you tell sshd to use port 22022 without labelling it first, **SELinux silently blocks the bind and sshd fails — you cannot get back in**.

```bash
sudo semanage port -a -t ssh_port_t -p tcp 22022
```

Verify it worked:

```bash
sudo semanage port -l | grep ssh
```

**Expected output:** `ssh_port_t    tcp    22022, 22`

---

## Step 2 — Generate SSH Keys

SSH key authentication replaces passwords with a cryptographic key pair. The **private key** stays on your personal computer. The **public key** goes on the server. Only your private key can prove your identity — password brute-forcing becomes impossible.

**Run these commands on your personal computer** (not the server):

=== "Linux / macOS / WSL"
    ```bash
    # Generate the key pair (press Enter to accept defaults, or set a passphrase for extra security)
    ssh-keygen -t ed25519

    # Copy the public key to the server (still using port 22 at this point)
    ssh-copy-id username@192.168.1.50
    ```

=== "Windows (PowerShell)"
    ```powershell
    # Generate the key pair
    ssh-keygen -t ed25519

    # Copy the public key to the server
    scp $env:USERPROFILE\.ssh\id_ed25519.pub username@192.168.1.50:~/id_ed25519.pub

    # Add the key to authorized_keys on the server
    ssh username@192.168.1.50 "mkdir -p ~/.ssh && cat ~/id_ed25519.pub >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && rm ~/id_ed25519.pub"
    ```

---

## Step 3 — Create the SSH Hardening Config

**Back on the server**, create a configuration drop-in that changes the SSH port, disables password login, and restricts access to your user:

```bash
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf > /dev/null << EOF
Port 22022
PubkeyAuthentication yes
PasswordAuthentication no
PermitRootLogin no
MaxAuthTries 3
LoginGraceTime 20
AllowUsers $(whoami)
EOF
```

Validate the config file (no errors should be reported):

```bash
sudo sshd -t
```

**Expected:** No output = no errors. Any output means there's a problem — do not proceed.

---

## Step 4 — Open the Firewall and Restart SSH

```bash
sudo firewall-cmd --permanent --add-port=22022/tcp
sudo firewall-cmd --reload
sudo systemctl restart sshd
```

!!! warning "Test in a NEW terminal window before closing this one"
    Open a **second** terminal on your personal computer and connect using the new port:
    ```bash
    ssh -p 22022 username@192.168.1.50
    ```
    If this works — great! Proceed to remove port 22. If it doesn't work, your old terminal is still open and you can diagnose the problem.

Once you've confirmed the new port works, remove the standard port 22:

```bash
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --reload
```

From now on, all SSH connections use: `ssh -p 22022 username@192.168.1.50`

---

## Step 5 — Configure Fail2ban

Fail2ban monitors your SSH log and automatically bans IP addresses that have too many failed login attempts. This prevents brute-force attacks.

```bash
sudo tee /etc/fail2ban/jail.local > /dev/null << 'EOF'
[sshd]
enabled = true
port = 22022
ignoreip = 127.0.0.1/8 ::1 192.168.0.0/16 10.0.0.0/8 172.16.0.0/12
backend = systemd
banaction = firewallcmd-rich-rules
EOF
```

```bash
sudo systemctl enable --now fail2ban
```

The `ignoreip` line ensures your local network (192.168.x.x) is never accidentally banned.

---

## ✅ Phase 4 Complete

Your server is now significantly hardened:

- ✅ SSH listens only on port 22022
- ✅ Password authentication is disabled (key-only)
- ✅ Root login is disabled
- ✅ Fail2ban is blocking brute-force attempts
- ✅ SELinux allows sshd on port 22022

[Next: Phase 5 — Docker Setup →](phase-5-docker.md){ .next-phase }
