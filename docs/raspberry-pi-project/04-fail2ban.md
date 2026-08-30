# Section 04 — Fail2Ban

**Date:** 2026-08-30

## Objective
Install and configure Fail2Ban on pilab to auto-ban IPs after repeated
failed SSH authentication attempts on port 2222, as a host-level layer
complementing the Tailscale grant restriction from Section 03.

## Steps Completed
- Installed fail2ban via apt
- Created /etc/fail2ban/jail.local (override file, never edit jail.conf
  directly — it's replaced on package updates)
- Configured sshd jail: port 2222, systemd backend, maxretry 3,
  findtime 10m, bantime 1h
- Enabled and started fail2ban.service
- Hit and resolved a packaging bug: Ubuntu 24.04's fail2ban 1.0.2 build
  depends on the 'asynchat' module, removed from Python 3.12's standard
  library. Ubuntu backported 'pyasyncore' but not its counterpart
  'pyasynchat', which doesn't exist in apt at all. Installed via pip
  instead (--break-system-packages, justified here as a system-service
  dependency fix, not general practice)
- Verified actual running state via journalctl, not just systemctl
  enable success (caught a misleading "active (running)" status that
  coincided with a crash-restart race condition)
- Validated sshd jail correctly bound to journal matches for
  sshd.service on the systemd backend
- Validated normal SSH access still succeeds from a fresh connection

## Commands Used
    sudo apt install -y fail2ban
    sudo tee /etc/fail2ban/jail.local > /dev/null << 'INNEREOF'
    [sshd]
    enabled  = true
    port     = 2222
    filter   = sshd
    logpath  = /var/log/auth.log
    backend  = systemd
    maxretry = 3
    findtime = 10m
    bantime  = 1h
    INNEREOF
    sudo systemctl enable --now fail2ban
    sudo journalctl -u fail2ban -n 50 --no-pager    # diagnosed asynchat error
    sudo pip3 install pyasynchat --break-system-packages
    sudo systemctl restart fail2ban
    sudo fail2ban-client status sshd
    ssh pilab "echo connection ok"                  # fresh-connection validation

## Validation Results
- fail2ban.service: active (running), stable across repeated checks
  (25s+ uptime, no crash-restart loop)
- sshd jail: enabled, 0 currently failed, 0 currently banned, journal
  match confirmed on sshd.service + sshd command
- Fresh SSH connection from macbook: succeeded

## Security Considerations
- SSH brute-force protection now active at the host level, independent
  of and complementary to the Tailscale grant restriction (Section 03)
- bantime of 1h is a starting baseline; consider increasing for
  production-grade posture, or layering in fail2ban's recidive jail
  for repeat offenders in a future hardening pass
- Manual pip-installed dependency (pyasynchat) sits outside apt's
  package tracking — re-check fail2ban service health after any future
  `apt upgrade` that touches the fail2ban package, in case Ubuntu ships
  a native fix and creates a conflict, or in case the upgrade reintroduces
  the missing-module crash
- Confirmed jail is reading via systemd backend/journal, not the
  logpath file — Ubuntu 24.04 does not reliably populate
  /var/log/auth.log by default, so logpath alone would have left the
  jail silently non-functional despite valid config syntax
