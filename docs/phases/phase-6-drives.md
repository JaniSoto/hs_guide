<div class="phase-header">
  <span class="phase-num">6</span>
  <div class="phase-header-text">
    <p class="phase-header-eyebrow">Phase 6 · Run over SSH</p>
    <h1>Data Drive Setup</h1>
    <p class="phase-header-sub">Format, mount, and secure your dedicated Nextcloud storage drive.</p>
  </div>
</div>

!!! danger "This will permanently erase the target drive"
    The formatting command below destroys all data on the drive you specify. **Triple-check the device name** before running it.

---

## Step 1 — Identify Your Data Drive

List all block devices to find your data drive:

```bash
lsblk
```

Look for the drive that is **not** your OS drive. It will show as a raw device with no mounted partitions (no `/` or `/boot` next to it). Common device names:

- `sda`, `sdb` — SATA drives
- `nvme1n1` — a second NVMe drive

**Example output:**
```
NAME        SIZE  TYPE  MOUNTPOINT
nvme0n1     500G  disk
└─nvme0n1p1 500G  part  /          ← This is the OS drive. Leave it alone.
sda         2TB   disk             ← This is your data drive.
```

In this example, `sda` is the data drive. Your device path would be `/dev/sda`.

---

## Step 2 — Format the Drive

Replace `/dev/sdX` with your actual device path (e.g. `/dev/sda` or `/dev/nvme1n1`):

```bash
DATA_DRIVE="/dev/sdX"   # ← CHANGE THIS to your actual drive

sudo parted "$DATA_DRIVE" --script mklabel gpt
sudo parted "$DATA_DRIVE" --script mkpart primary ext4 1MiB 100%
sudo partprobe "$DATA_DRIVE"
sleep 2

# Get the new partition name (e.g. /dev/sda1 or /dev/nvme1n1p1)
DATA_PART=$(lsblk -rn -o NAME "$DATA_DRIVE" | tail -n 1 | awk '{print "/dev/"$1}')
echo "Partition: $DATA_PART"

sudo mkfs.ext4 "$DATA_PART"
```

---

## Step 3 — Mount the Drive Permanently

**3a — Create the mount point with an immutable guard:**

```bash
sudo mkdir -p /srv/nextcloud-data
sudo chattr +i /srv/nextcloud-data
```

The `+i` (immutable) flag on the **empty** directory is a safety measure: if the data drive ever fails to mount at boot, Docker tries to write to the bare directory on your OS drive. The immutable flag causes a "permission denied" error instead, which is much easier to diagnose than discovering your Nextcloud data silently ended up on the wrong drive.

**3b — Add to `/etc/fstab` for automatic mounting at boot:**

```bash
PART_UUID=$(sudo blkid -s UUID -o value "$DATA_PART")

echo "UUID=${PART_UUID}  /srv/nextcloud-data  ext4  defaults,nofail  0  2" \
  | sudo tee -a /etc/fstab

sudo mount -a
```

The `nofail` option in fstab means: if the drive is missing at boot, the system still boots normally instead of dropping to an emergency shell.

Verify the drive is mounted:

```bash
df -h /srv/nextcloud-data
```

You should see the drive's capacity listed. If you see the OS drive's capacity, something went wrong.

---

## Step 4 — Apply SELinux Labels

SELinux needs to know that Docker containers are allowed to read and write this directory. Apply the `container_file_t` label:

```bash
sudo semanage fcontext -a -t container_file_t "/srv/nextcloud-data(/.*)?"
sudo restorecon -Rv /srv/nextcloud-data
```

Verify the label:

```bash
ls -Zd /srv/nextcloud-data
```

**Expected output:**
```
system_u:object_r:container_file_t:s0 /srv/nextcloud-data
```

---

## ✅ Phase 6 Complete

Your data drive is formatted, mounted at `/srv/nextcloud-data`, set to auto-mount at boot safely, and labelled correctly for Docker containers.

[Next: Phase 7 — Core Services →](phase-7-services.md){ .next-phase }
