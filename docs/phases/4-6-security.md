```markdown
# Phases 4 - 6: Security & Drives

## Phase 4: Ironclad Security
*SELinux Port · SSH Keys · firewalld · Fail2ban*

!!! failure "Critical SELinux Step"
    SELinux controls which ports `sshd` is permitted to bind to. Only port 22 has the `ssh_port_t` label by default. You MUST label port 22022 BEFORE restarting sshd, or SELinux will silently block the bind syscall and lock you out permanently.

**1. Label port 22022 in the SELinux policy**
```bash
sudo semanage port -a -t ssh_port_t -p tcp 22022
# Verify:
sudo semanage port -l | grep ssh
# Expected: ssh_port_t tcp 22022, 22
```

**2. Generate and copy SSH keys (Run on Personal Computer)**
*Linux / macOS / WSL:*
```bash
ssh-keygen -t ed25519
ssh-copy-id username@192.168.1.50
```
*Windows (PowerShell):*
```powershell
ssh-keygen -t ed25519
scp $env:USERPROFILE\.ssh\id_ed25519.pub username@192.168.1.50:~/id_ed25519.pub
ssh username@192.168.1.50 "mkdir -p ~/.ssh && cat ~/id_ed25519.pub >> ~/.ssh/authorized_keys && rm ~/id_ed25519.pub"
```

**3. Create the SSH hardening drop-in config (Back on Server)**
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
sudo sshd -t # Validate syntax. No output = no errors.
```

**4. Configure firewalld and restart sshd**
```bash
sudo firewall-cmd --permanent --add-port=22022/tcp
sudo firewall-cmd --reload
sudo systemctl restart sshd
```
*TEST NOW:* Open a NEW terminal on your personal PC and connect: `ssh -p 22022 username@192.168.1.50`. If successful, remove port 22:
```bash
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --reload
```

**5. Configure and enable Fail2ban**
```bash
sudo tee /etc/fail2ban/jail.local > /dev/null << 'EOF'
[sshd]
enabled = true
port = 22022
ignoreip = 127.0.0.1/8 ::1 192.168.0.0/16 10.0.0.0/8 172.16.0.0/12
backend = systemd
banaction = firewallcmd-rich-rules
EOF
sudo systemctl enable --now fail2ban
```

---

## Phase 5: Docker Initialisation

**1. Enable Docker and add your user to the docker group**
```bash
sudo systemctl enable --now docker
grep -q "^docker:" /etc/group || sudo sh -c "getent group docker >> /etc/group"
sudo usermod -aG docker $USER
```

**2. Prevent Docker logs from filling your disk**
```bash
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
EOF
sudo systemctl restart docker
```

**3. Create the shared Docker network**
```bash
docker network create nextcloud-aio
```

---

## Phase 6: Data Drive Setup

!!! danger "Format Warning"
    This will ERASE ALL DATA on the target drive. Confirm the correct drive with `lsblk` before proceeding.

**1. Format, mount, then label the data drive**
```bash
DATA_DRIVE="/dev/nvmeXnY" # <-- REPLACE THIS with your actual data drive
sudo parted "$DATA_DRIVE" --script mklabel gpt
sudo parted "$DATA_DRIVE" --script mkpart primary ext4 1MiB 100%
sudo partprobe "$DATA_DRIVE"
sleep 2
DATA_PART=$(lsblk -rn -o NAME "$DATA_DRIVE" | tail -n 1 | awk '{print "/dev/"$1}')
sudo mkfs.ext4 "$DATA_PART"
```

**2. Apply `chattr +i` and mount**
The `+i` flag sets the "immutable" attribute on the empty directory. If the drive ever fails to mount at boot, Docker gets a "permission denied" error instead of corrupting the OS drive.
```bash
sudo mkdir -p /srv/nextcloud-data
sudo chattr +i /srv/nextcloud-data

PART_UUID=$(sudo blkid -s UUID -o value "$DATA_PART")
echo "UUID=${PART_UUID} /srv/nextcloud-data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
sudo mount -a
```

**3. Apply SELinux rules**
```bash
sudo semanage fcontext -a -t container_file_t "/var/srv/nextcloud-data(/.*)?"
sudo restorecon -Rv /var/srv/nextcloud-data
```
```
