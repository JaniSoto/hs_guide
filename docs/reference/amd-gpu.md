# AMD GPU Acceleration for Nextcloud

Enable VA-API hardware-accelerated video transcoding inside the Nextcloud container using your AMD or Intel GPU.

!!! warning "AMD and Intel GPUs only"
    This guide does not apply to NVIDIA GPUs. NVIDIA GPUs are not compatible with the VA-API interface used by Nextcloud.

!!! info "When to read this page"
    If you want GPU acceleration, **read this before completing Phase 7**. You need to modify the Nextcloud `docker-compose.yml` before running `docker compose up -d` for the first time. If you've already started Nextcloud, follow the additional steps marked "existing install" below.

---

## What is VA-API?

VA-API (Video Acceleration API) is a Linux interface for hardware-accelerated video operations. Without it, Nextcloud uses your CPU to process every frame of every video — thumbnails, previews, and transcoding. With VA-API, these operations are offloaded to the GPU, dramatically reducing CPU usage and speeding up video processing.

---

## Step 1 — Verify GPU and Render Node

```bash
ls -la /dev/dri/
```

**Expected output:**

```
crw-rw---- 1 root video    226, 0 ... card0
crw-rw---- 1 root render  226, 128 ... renderD128
```

You need both `card0` (display output) and `renderD128` (the render node that VA-API uses). If `renderD128` is missing, GPU acceleration isn't available on your system.

---

## Step 2 — Remove the Immutable Guard (if already running)

The `chattr +i` flag set in Phase 6 protects the data directory but also blocks Nextcloud AIO's first-start initialisation. Remove it before starting containers with GPU support:

```bash
sudo chattr -i /srv/nextcloud-data
```

!!! note "Re-apply the guard after startup"
    You'll add `+i` back in Step 4 after Nextcloud's containers are healthy.

---

## Step 3 — Update the Nextcloud Compose File

```bash
cd ~/docker/nextcloud
nano docker-compose.yml
```

Add the `devices` block and `NEXTCLOUD_ENABLE_DRI_DEVICE` environment variable to the mastercontainer service. Your updated service section should look like this:

```yaml
services:
  nextcloud-aio-mastercontainer:
    image: ghcr.io/nextcloud-releases/all-in-one:latest
    init: true
    restart: always
    container_name: nextcloud-aio-mastercontainer
    security_opt:
      - label:disable
    labels:
      - "com.centurylinklabs.watchtower.enable=false"
    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - '127.0.0.1:9443:8080'
    networks:
      - nextcloud-aio
    devices:
      - /dev/dri:/dev/dri            # Passes the GPU render node into the container
    environment:
      - APACHE_PORT=11000
      - APACHE_IP_BINDING=127.0.0.1
      - NEXTCLOUD_DATADIR=/srv/nextcloud-data
      - NEXTCLOUD_ENABLE_DRI_DEVICE=true   # Tells AIO to enable VA-API
```

---

## Step 4 — Start Containers and Re-apply the Guard

```bash
docker compose up -d

# Wait until all AIO containers show "(healthy)" — this can take several minutes
docker ps

# Re-apply the immutable guard
sudo chattr +i /srv/nextcloud-data
```

---

## Step 5 — Verify VA-API is Active

```bash
docker exec nextcloud-aio-nextcloud ffmpeg -hwaccels 2>/dev/null | grep -i vaapi
```

**Expected output:** `vaapi`

If vaapi is not listed, the GPU render node isn't being passed through correctly or the driver isn't loaded.

---

## Return to Phase 7

If you came here from Phase 7, [return to complete the Nextcloud deployment →](../phases/phase-7-services.md)
