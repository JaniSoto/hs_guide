```markdown
# Appendix A: Troubleshooting

### Machine boots to Gaming Mode after initial install
*   Go to Gaming Mode → Power menu → Switch to Desktop.
*   Then run Phase 1 Steps 2–4 to enable SSH and apply the headless override.

### start-kde.sh / start-gaming.sh reports "session already active" but nothing is visible
```bash
loginctl list-sessions
# Note the stuck session ID.
loginctl terminate-session SESSION_ID
sleep 2
sudo start-kde.sh # or start-gaming.sh
```

### SDDM shows login screen instead of autologging in
```bash
ls /usr/share/wayland-sessions/
# Expected: plasma.desktop and gamescope-session-steam.desktop
journalctl -u sddm -n 50 --no-pager
```

### Sunshine not streaming / Moonlight cannot connect
1.  Confirm WireGuard is connected: `ping 10.8.0.1`
2.  Confirm KDE or Gaming Mode is running: `loginctl list-sessions`
3.  Confirm Sunshine is running: `systemctl --user status sunshine`
4.  Confirm Sunshine ports are open: `sudo firewall-cmd --list-ports`
5.  On server, re-run: `systemctl --user restart sunshine`

### Docker containers not running after a reboot
```bash
sudo systemctl status docker
cd ~/docker/npm && docker compose up -d
cd ~/docker/nextcloud && docker compose up -d
# Repeat for duckdns and wireguard
```

### Data drive not mounting — system dropped to emergency mode
This should not happen if `nofail` is in fstab (Phase 6). Connect a display and keyboard. At the emergency shell:
```bash
nano /etc/fstab
# Add "nofail" to the /srv/nextcloud-data options column.
systemctl reboot
```
```
