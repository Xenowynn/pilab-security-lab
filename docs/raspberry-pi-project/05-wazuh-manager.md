# Section 05 — Wazuh Manager Deployment

**Date:** 2026-08-30

## Objective
Deploy Wazuh (manager, indexer, dashboard — single-node all-in-one) on
pilab using the official assisted installer, marking the start of
Project 1 (SIEM & Threat Detection Lab) and the first real hardware
stress test for the Pi 4.

## Steps Completed
- Confirmed ARM64/aarch64 support across the full Wazuh 4.14 stack
  (manager, indexer, dashboard) via current official documentation
- Verified hardware against Wazuh's stated minimum (4 vCPU / 8GB RAM /
  50GB disk for 1-25 agents) — Pi 4 meets this at the floor, not with
  headroom
- Diagnosed and resolved a system-level package conflict blocking the
  installer: a locally-built libbz2-1.0 version (pre-existing on the
  base image, not from Ubuntu's repo) was incompatible with the
  dpkg-dev/debhelper dependency chain required by the Wazuh installer.
  Fixed via forced reinstall of the repo-tracked version
  (--allow-downgrades)
- Ran the official all-in-one installer (wazuh-install.sh -a),
  completing in ~52 minutes total (indexer ~10min, manager ~13min,
  dashboard ~17min) — confirms microSD I/O as the primary hardware
  constraint on this deployment, not CPU or RAM
- Verified all three core services (wazuh-indexer, wazuh-manager,
  wazuh-dashboard) independently, plus filebeat, actually running
  (not just install-script success)
- Secured the generated wazuh-install-files.tar credentials archive
  (chmod 600, already root-owned by default)
- Extended the Tailscale grants policy (from Section 03) to permit
  macbook -> pilab access on port 443 (dashboard) and ports 1514/1515
  (future agent enrollment/communication)
- Rotated the auto-generated admin password after it was exposed in
  plaintext during setup, using the proper wazuh-passwords-tool.sh
  (indexer-level credential, not a dashboard-managed account — the
  dashboard's self-service password change correctly rejected the
  attempt with a "reserved resource" error)
- Recovered wazuh-manager from a systemd restart-timeout failure
  (Result: timeout) caused by a multi-service simultaneous restart
  exceeding systemd's default stop timeout; orphaned child daemons
  were cleanly stopped via wazuh-control directly, then the unit was
  reset and restarted successfully
- Disabled the Wazuh apt repository post-install per Wazuh's own
  recommendation, preventing accidental version-mismatch breakage from
  a routine system upgrade

## Commands Used
    curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
    sudo bash ./wazuh-install.sh -a
    sudo apt install --reinstall --allow-downgrades libbz2-1.0=1.0.8-5.1
    sudo systemctl status wazuh-indexer wazuh-manager wazuh-dashboard filebeat --no-pager
    sudo bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -u admin -p '<new-password>'
    sudo /var/ossec/bin/wazuh-control stop      # manual recovery from restart timeout
    sudo systemctl reset-failed wazuh-manager
    sudo systemctl start wazuh-manager
    sudo sed -i 's/^deb /#deb /' /etc/apt/sources.list.d/wazuh.list

## Validation Results
- All four services (wazuh-indexer, wazuh-manager, wazuh-dashboard,
  filebeat) confirmed active (running) independently, stable for
  multiple hours post-install
- Dashboard login confirmed working end-to-end at
  https://100.95.54.34 with rotated admin credential
- Wazuh apt repo confirmed removed from apt update's source list

## Security Considerations
- Dashboard and agent-communication ports are reachable only via
  Tailscale grants (macbook-only), consistent with the access-control
  model established in Section 03 — not exposed on the LAN-facing UFW
  rules
- Admin credential rotated after chat-session exposure during setup;
  original auto-generated password treated as compromised and retired
- wazuh-install-files.tar (contains cluster key, certs, all component
  passwords) restricted to root-only access
- Wazuh apt repo disabled — version upgrades to the stack must now be
  performed deliberately, not via routine apt upgrade, to avoid
  breaking manager/indexer/dashboard version alignment
- Noted for future sessions: this Pi 4 is now running near its
  practical resource ceiling for this stack. Adding real agent load,
  Shuffle SOAR, or MISP (per the original roadmap) may require the
  mini PC upgrade flagged early in this project's planning
