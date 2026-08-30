## Section 02: UFW Baseline (Default-Deny-Incoming) ✅

**Date:** 2026-08-29
**Host:** pilab (192.168.7.100 / Tailscale 100.95.54.34)

### Changes
- Enabled UFW with default deny incoming / allow outgoing
- Allowed TCP 2222 (SSH) explicitly before enabling, to avoid a lockout
- Enabled UFW logging to /var/log/ufw.log

### Key finding: UFW does not filter Tailscale traffic on this host
Tailscale manages its own iptables chain (`ts-input`) which is evaluated
*before* UFW's chains in the INPUT chain order. `ts-input` contains a
blanket ACCEPT rule for all traffic arriving on `tailscale0`
(proto=all, source=0.0.0.0/0), inserted automatically by Tailscale to
guarantee its own connectivity. As a result:
- Any UFW rule scoped to `tailscale0` is unreachable — Tailscale's own
  chain resolves the packet first
- Initially added `ufw allow in on tailscale0 to any port 2222`, then
  removed it after confirming via `iptables -L ts-input -n -v` that it
  had no effect — kept it would have been misleading documentation
- Verified via a live test: attempted connection to a closed port
  (8080) over Tailscale was rejected instantly with zero corresponding
  UFW log entry, confirming UFW never evaluated the packet

### Validation
- ufw status verbose confirms active, correct default policies
- LAN interface (wlan0, 192.168.7.100): default-deny confirmed working —
  only 2222/tcp reachable
- New SSH session via `ssh pilab` (Tailscale IP) succeeds post-enable
- Rule list confirmed clean: only 2222/tcp (v4/v6), no tailscale0-scoped
  UFW rules remaining

### Scope and known limitation
- **LAN interface: fully enforced.** Previously wide open, now
  default-deny with only SSH allowed.
- **Tailscale interface: not enforced by UFW.** Port-level access
  control for tailnet traffic must be configured via Tailscale ACLs
  (in the Tailscale admin console), not the host firewall. This is
  the correct tool for this job — not a gap in this section's
  execution, but a boundary of what UFW can do on this host.
- Outbound traffic remains fully open by design — egress restriction
  is a separate, more advanced hardening step, out of scope here.

### Open item (carried forward)
- Configure Tailscale ACLs to restrict which tailnet devices/ports can
  reach pilab — this is the actual enforcement point for Tailscale-side
  access control. Candidate for Section 03.
