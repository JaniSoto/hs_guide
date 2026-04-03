<div class="phase-header">
  <span class="phase-num">5</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 5 · Run over SSH</p>
    <h1>Docker Initialisation</h1>
    <p class="phase-header-sub">Enable Docker, add your user, and create the shared container network.</p>
  </div>
</div>

---

## Step 1 — Enable Docker and Add Your User

Start the Docker service and add your user to the `docker` group. Being in the `docker` group lets you run `docker` commands without typing `sudo` every time.

```bash
sudo systemctl enable --now docker

# Create the docker group if it doesn't exist yet, then add your user
grep -q "^docker:" /etc/group || sudo sh -c "getent group docker >> /etc/group"
sudo usermod -aG docker $USER
```

!!! info "Group membership takes effect on next login"
    The `docker` group change doesn't apply to your current SSH session. You'll need to log out and reconnect: `exit`, then `ssh -p 22022 username@your-server-ip`. After reconnecting, `docker ps` should work without `sudo`.

---

## Step 2 — Limit Docker Log Sizes

By default, Docker container logs can grow without limit and eventually fill your OS drive. This adds a safety cap — 10 MB per log file, maximum 3 rotating files per container.

```bash
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

sudo systemctl restart docker
```

---

## Step 3 — Create the Shared Docker Network

Docker containers on different `docker-compose.yml` files can only communicate with each other if they share a network. We create one shared network (`nextcloud-aio`) that all relevant services will join.

```bash
docker network create nextcloud-aio
```

Verify it was created:

```bash
docker network ls
```

You should see `nextcloud-aio` in the list.

---

## ✅ Phase 5 Complete

Docker is running, your user can manage containers, logs are capped, and the shared network is ready.

[Next: Phase 6 — Data Drive →](phase-6-drives.md){ .next-phase }
