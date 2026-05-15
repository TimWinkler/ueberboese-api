# Bose SoundTouch — Allow Access from External Subnet

## Problem

The SoundTouch has a built-in iptables firewall that drops all inbound traffic on
`wlan0` that is not ESTABLISHED/RELATED or sourced from the same local subnet.
Devices on other subnets (e.g. a separate LAN) cannot reach the speaker even if
the router permits the traffic.

Relevant rules on the device:

```
ACCEPT  all  ctstate RELATED,ESTABLISHED
ACCEPT  all  from <LOCAL_SUBNET>   on wlan0
DROP    all  from 0.0.0.0/0        on wlan0   ← blocks everything else
```

## Fix

Insert an ACCEPT rule for your LAN subnet before the DROP. The SoundTouch root
filesystem is read-only, so the only persistent writable location is `/mnt/nv/`.

`/etc/init.d/shelby_local` executes `/mnt/nv/rc.local` at boot (if executable).
Shepherd's Firewall daemon rebuilds all iptables rules on every WiFi reconnect, so
a watchdog is needed to re-apply the rule after Shepherd runs.

Create `/mnt/nv/rc.local` on the SoundTouch:

```bash
cat > /mnt/nv/rc.local << 'EOF'
#!/bin/sh
(
  while true; do
    iptables -C INPUT -i wlan0 -s <YOUR_LAN_SUBNET>/24 -j ACCEPT 2>/dev/null || \
      iptables -I INPUT 3 -i wlan0 -s <YOUR_LAN_SUBNET>/24 -j ACCEPT
    sleep 30
  done
) &
EOF
chmod +x /mnt/nv/rc.local
```

Replace `<YOUR_LAN_SUBNET>/24` with your actual subnet (e.g. `192.168.1.0/24`).

## How it works

- The watchdog runs as a background process from boot
- Every 30 seconds it checks if the rule exists (`iptables -C`); if not, it inserts it
- Position 3 places it after the ESTABLISHED rule and before the DROP
- Survives reboots (written to persistent UBI flash volume `/mnt/nv/`)
- Survives WiFi reconnects (Shepherd re-applies its rules; watchdog restores ours
  within 30 seconds)

## Notes

- The SoundTouch filesystem is BusyBox Linux with a read-only rootfs
- SSH access requires legacy key algorithm flags:
  `ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa root@<IP>`
- If direct SSH times out (same return-path issue), use a jump host on the same
  subnet as the SoundTouch
