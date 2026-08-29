# 00 — OS Setup and Static IP Configuration

**Date:** 2026-08-29
**Section:** 00
**Status:** ✅ Complete

---

## Objective

Fresh flash of Ubuntu Server 24.04.4 LTS on Raspberry Pi 4 8GB.
System updates, static IP configuration, timezone, hostname verification,
cloud-init disabled, and Tailscale remote access installed.

---

## System Details

| Item | Value |
|------|-------|
| OS | Ubuntu Server 24.04.4 LTS (aarch64) |
| Hostname | pilab |
| Username | xenowynn |
| Static IP | 192.168.7.100 |
| Tailscale IP | 100.95.54.34 |
| Network | Agnes eero (192.168.7.x) |
| Timezone | America/Los_Angeles (PDT) |

---

## Steps Completed

1. Flashed Ubuntu Server 24.04.4 LTS via Raspberry Pi Imager
2. Applied 133 pending security updates
3. Set timezone to America/Los_Angeles
4. Verified hostname: pilab
5. Disabled cloud-init network management
6. Configured static IP 192.168.7.100 via Netplan
7. Validated static IP active on wlan0
8. Installed Tailscale for remote access from any location
9. Confirmed SSH access via Tailscale from remote location

---

## Key Commands

```bash
sudo apt update && sudo apt upgrade -y
sudo timedatectl set-timezone America/Los_Angeles
sudo bash -c 'echo "network: {config: disabled}" > /etc/cloud/cloud.cfg.d/99-disable-network.cfg'
sudo chmod 600 /etc/netplan/99-static.yaml
sudo netplan try
ip a show wlan0
curl -fsSL https://tailscale.com/install.sh | sudo sh
sudo tailscale up
```

---

## Validation Results

- 133 security patches applied
- Timezone: America/Los_Angeles (PDT) confirmed
- Hostname: pilab confirmed
- Static IP: 192.168.7.100/24 active on wlan0 (valid_lft forever)
- Cloud-init network management disabled
- Tailscale connected: 100.95.54.34
- SSH confirmed from remote location via Tailscale

---

## Security Considerations

- Static IP eliminates DHCP IP changes that would break SSH access
- Cloud-init disabled prevents network config being overwritten on reboot
- All 133 security updates applied before hardening begins
- Netplan config permissions set to 600 — WiFi password not world-readable
- Tailscale provides encrypted remote access without exposing ports to internet
