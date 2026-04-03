```markdown
# Appendix C: Optional AMD GPU Acceleration for Nextcloud

!!! warning "Hardware Limitation"
    AMD and Intel GPUs only. Do not follow this appendix on NVIDIA systems. 

VA-API (Video Acceleration API) is a Linux interface for hardware-accelerated video operations. Instead of the CPU processing every video frame, VA-API offloads it to the GPU.

**1. Verify your AMD GPU and render nodes are present**
```bash
ls -la /dev/dri/
# Expected: card0 (display output) and renderD128 (render node)
```

**2. Remove the immutable guard**
The AIO mastercontainer's first-start initialisation writes ownership data to the data directory. The `chattr +i` immutable flag blocks this.
```bash
sudo chattr -i /srv/nextcloud-data
```

**3. Update the Nextcloud AIO compose file**
```bash
cd ~/docker/nextcloud
nano docker-compose.yml
```
Add the `devices` and `NEXTCLOUD_ENABLE_DRI_DEVICE` lines to your config:
```yaml
    devices:
      - /dev/dri:/dev/dri # Passes GPU render node into the container.
    environment:
      - APACHE_PORT=11000
      - APACHE_IP_BINDING=127.0.0.1
      - NEXTCLOUD_DATADIR=/srv/nextcloud-data
      - NEXTCLOUD_ENABLE_DRI_DEVICE=true
```

**4. Start containers and re-apply the immutable guard**
```bash
docker compose up -d
# Wait until all AIO containers show "(healthy)".
sudo chattr +i /srv/nextcloud-data
```

**5. Verify hardware acceleration is active**
```bash
docker exec nextcloud-aio-nextcloud ffmpeg -hwaccels 2>/dev/null | grep -i vaapi
# Expected: "vaapi" listed as an available hardware accelerator.
```
```
