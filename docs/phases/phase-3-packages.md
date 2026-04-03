<div class="phase-header">
  <span class="phase-num">3</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 3 · Run over SSH</p>
    <h1>System Package Installation</h1>
    <p class="phase-header-sub">Install Docker CE, Fail2ban, and SELinux tools on the immutable OS.</p>
  </div>
</div>

!!! info "From this point on — everything is done over SSH"
    Connect from your personal computer: `ssh username@192.168.1.50` (use your actual IP). You no longer need to touch the server physically.

---

## Step 1 — Add the Docker Repository

Bazzite comes with Podman pre-installed. However, **Nextcloud AIO requires native Docker CE** — it communicates directly with the Docker socket in a way that's incompatible with Podman's compatibility shim.

Add Docker's official repository for Fedora:

```bash
sudo curl -fsSL https://download.docker.com/linux/fedora/docker-ce.repo \
  -o /etc/yum.repos.d/docker-ce.repo
```

---

## Step 2 — Install All Required Packages

Because Bazzite is an **immutable OS**, you layer packages on top using `rpm-ostree` instead of the usual `dnf install`. The packages become part of your OS layer and survive all future OS updates.

The command below handles both cases — whether or not the `podman-docker` compatibility shim is currently installed:

```bash
if rpm -q podman-docker >/dev/null 2>&1; then
  rpm-ostree override remove podman-docker \
    --install docker-ce \
    --install docker-ce-cli \
    --install containerd.io \
    --install docker-buildx-plugin \
    --install docker-compose-plugin \
    --install fail2ban \
    --install fail2ban-firewalld \
    --install policycoreutils-python-utils
else
  rpm-ostree install \
    docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin \
    fail2ban fail2ban-firewalld \
    policycoreutils-python-utils
fi
```

**What each package does:**

| Package | Purpose |
|---------|---------|
| `docker-ce` | Docker engine — runs your containers |
| `docker-ce-cli` | The `docker` command-line tool |
| `containerd.io` | Low-level container runtime (required by Docker) |
| `docker-buildx-plugin` | Extended build capabilities |
| `docker-compose-plugin` | Enables `docker compose` command |
| `fail2ban` | Bans IPs with too many failed SSH logins |
| `fail2ban-firewalld` | Integrates Fail2ban with firewalld |
| `policycoreutils-python-utils` | Provides the `semanage` command for SELinux policy management |

---

## Step 3 — Reboot to Apply

`rpm-ostree` stages changes — they don't take effect until you reboot:

```bash
systemctl reboot
```

Your SSH session will disconnect. Wait about 60 seconds, then reconnect:

```bash
ssh username@192.168.1.50
```

Verify Docker is available:

```bash
docker --version
```

---

## ✅ Phase 3 Complete

All required packages are installed. The server is ready for security hardening and Docker configuration.

[Next: Phase 4 — Security Hardening →](phase-4-security.md){ .next-phase }
