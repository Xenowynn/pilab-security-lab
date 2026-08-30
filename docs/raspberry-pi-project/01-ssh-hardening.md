## Section 01: SSH Hardening ✅

**Date:** 2026-08-29
**Host:** pilab (192.168.7.100 / Tailscale 100.95.54.34)

### Changes
- Generated ED25519 keypair (passphrase-protected) on workstation, deployed via ssh-copy-id
- Disabled ssh.socket activation, enabled standalone ssh.service (required on
  Ubuntu 24.04 — socket activation ignores sshd_config's Port directive)
- sshd_config: Port 2222, PermitRootLogin no, PasswordAuthentication no,
  PubkeyAuthentication yes, KbdInteractiveAuthentication no
- Found and fixed cloud-init override: /etc/ssh/sshd_config.d/50-cloud-init.conf
  had PasswordAuthentication yes, silently beating the main config (Include
  directives process before the rest of sshd_config — first match wins)
- UFW confirmed inactive; port 2222 firewall rule not needed at this time
  (flagged as a gap for a future section)
- Configured ~/.ssh/config alias `pilab` on workstation
- Verified and removed orphaned key/config from a retired prior project
  (pi_ed25519 / Host pi block) — confirmed it was never installed in this
  Pi's authorized_keys, deleted local key files for cleanliness

### Validation
- Key-based login confirmed on port 2222, no password prompt
- Port 22 connection refused
- Root login refused immediately via publickey rejection (no password fallback)
- Effective config verified via `sshd -T`, not just file contents
- ssh.service active, ssh.socket disabled
- authorized_keys confirmed to contain exactly one key (current, in use)

### Incidents during this section
- Ran `systemctl enable ssh.service` without `start` after disabling
  ssh.socket — caused a full port-22 lockout requiring physical power-cycle
  to recover (ssh.service was enabled for boot, so reboot resolved it)
- Cloud-init drop-in config silently overrode PasswordAuthentication despite
  correct edits in the main sshd_config file — required checking effective
  config (`sshd -T`) rather than trusting the file alone

### Open item
- UFW is inactive on this host — no firewall filtering at all currently.
  Flagged for a dedicated future section (default-deny-incoming policy).
