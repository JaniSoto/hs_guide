# Appendix B: Quick Reference (Workflows)

All commands are run from your personal computer's terminal.

!!! danger "CRITICAL: firewall-cmd reload warning"
    Never run `firewall-cmd --reload` while Docker is running without immediately running `sudo systemctl restart docker` afterward. Firewalld flushes and rebuilds the entire nftables ruleset, wiping Docker's port-forwarding rules. Ports 80, 443, and 51820/UDP will become unreachable.

### Stream the desktop remotely (Sunshine + Moonlight)
1. Connect WireGuard on your client device.
2. Launch KDE: `ssh -p 22022 username@your-server-ip 'sudo start-kde.sh'`
3. Open Moonlight, connect to `10.8.0.1`, select Desktop.

### Stream Gaming Mode remotely
1. Connect WireGuard on your client device.
2. Launch Gaming Mode: `ssh -p 22022 username@your-server-ip 'sudo start-gaming.sh'`
3. In Moonlight, connect to `10.8.0.1`, select Steam Big Picture.

### Game locally (physical display only)
```bash
ssh -p 22022 username@your-server-ip 'sudo start-gaming.sh'
# To stop: Power menu in Steam Big Picture → Exit to Desktop.
```

### Force-stop a stuck session
```bash
ssh -p 22022 username@your-server-ip 'sudo stop-session.sh'
# WARNING: all unsaved work in the terminated session is lost.
```
