# Section 03 — Tailscale ACL Configuration

**Date:** 2026-08-30

## Objective
Close the enforcement gap identified in Section 02: UFW cannot filter
tailscale0 traffic because Tailscale's ts-input iptables chain is
evaluated upstream of UFW. Real access control for tailnet traffic
must be configured via the Tailscale policy file, not the host firewall.

## Steps Completed
- Opened Tailscale admin console policy editor
- Noted current schema uses `grants` (not the deprecated `acls` array)
- Defined host aliases for pilab (100.95.54.34) and macbook (100.125.75.26)
- Replaced default accept-all grant with explicit rule: only macbook
  may reach pilab, restricted to port 2222/tcp (SSH)
- Left Tailscale SSH (`ssh` block) at default — unused feature, does not
  affect system sshd
- Kept a live SSH session open during the policy change as a rollback
  safety net (Section 01 lockout lesson)
- Validated new SSH connections succeed post-change from a fresh session

## Commands Used
    ssh pilab                        # safety-net session, kept open during change
    ssh pilab "echo connection ok"   # fresh-connection validation

## Validation Results
- New SSH connection from macbook to pilab: succeeded (port 2222 only)
- Policy confirmed default-deny for all other src/dst/port combinations

## Security Considerations
- Tailscale grants are now the actual enforcement boundary for
  tailscale0 traffic; UFW rules on that interface remain non-functional
  (confirmed Section 02)
- Any new device added to this tailnet is denied by default until
  explicitly added to `hosts` and `grants`
- Next devices (e.g. planned Kali Pi in Project 2) must be added
  explicitly before they can reach pilab
- Note for future sessions: Tailscale's default policy schema changed
  from `acls` to `grants` — if referencing older Tailscale docs/tutorials,
  confirm current syntax against the live admin console first
