# 06 - Wazuh Cold-Boot Stability Fixes

**Date:** 2026-09-13 to 2026-09-14

## Objective
Diagnose and resolve wazuh-manager startup failures occurring on Pi
cold boot (power cycle), discovered while preparing for Section 06
agent enrollment work.

## Incident 1: Manager Start-Timeout on Cold Boot

**Symptom:** wazuh-manager failed to start after Pi power-on, with
orphaned child processes remaining after systemd terminated the unit.
Symptom pattern superficially resembled the Section 05 stop-timeout
race but had a different root cause.

**Root cause:** Default systemd TimeoutStartSec=45s (inherited from
vendor unit's TimeoutSec=45) was insufficient for the full manager
startup sequence (apid, csyslogd, integratord, agentlessd, authd,
db, execd, analysisd, syscheckd, remoted, logcollector, monitord,
modulesd) on cold-boot microSD I/O, which lacks the OS page cache
built up during a warm restart.

**Fix:** systemd drop-in override at
/etc/systemd/system/wazuh-manager.service.d/override.conf:
[Service]
TimeoutStartSec=300

**Distinguishing this from Section 05:** journalctl shows "start
operation timed out. Terminating" (this incident, occurs after
"Starting...") vs. "Failed with result 'timeout'" following a stop
sequence (Section 05, occurs after "Stopping..."). Same visible
symptom (orphaned processes), different systemd timeout being hit.

## Incident 2: No-RTC Clock Race Across All Three Services

**Symptom:** wazuh-dashboard's systemd ActiveEnterTimestamp showed a
date over a week stale relative to the actual boot. Investigation
traced this to the Raspberry Pi 4 lacking an onboard RTC battery —
on every cold boot, system clock starts from last-saved time and
only corrects once systemd-timesyncd completes its first NTP sync.
Services that finish starting before sync completes get a
permanently incorrect start timestamp, and more importantly log
events with incorrect timestamps during the pre-sync window — a
real log-integrity concern for a SIEM lab.

**Root cause:** systemd-time-wait-sync.service, which blocks and
signals time-sync.target on first successful clock sync, existed on
the system but was not enabled by default (preset: disabled). The
vendor wazuh-manager.service already declared Wants=time-sync.target
but not After=, meaning the dependency was pulled in without
enforcing start-order — a partial fix already present upstream.
Neither wazuh-indexer.service nor wazuh-dashboard.service declared
any time-sync dependency at all.

**Fix:**
1. Enabled the sync-blocking service:
   sudo systemctl enable --now systemd-time-wait-sync.service
2. Added systemd drop-in overrides for all three services ordering
   startup after time-sync.target:
   /etc/systemd/system/wazuh-manager.service.d/override.conf
   /etc/systemd/system/wazuh-indexer.service.d/override.conf
   /etc/systemd/system/wazuh-dashboard.service.d/override.conf
   [Unit]
   After=time-sync.target
   Wants=time-sync.target

## Commands Used
- sudo mkdir -p /etc/systemd/system/<service>.service.d
- sudo tee /etc/systemd/system/<service>.service.d/override.conf > /dev/null << 'INNEREOF'
- sudo systemctl enable --now systemd-time-wait-sync.service
- sudo systemctl daemon-reload
- systemctl cat <service> --no-pager (verification)
- sudo reboot

## Validation Results
Verified across three independent cold boots (2026-09-13,
2026-09-14, and 2026-10-03):
- wazuh-manager: active (running), no timeout, started within
  ~50-60s of time-sync completing
- wazuh-indexer: active (running), full Java/OpenSearch startup
  ~2-3min (expected baseline, not a failure)
- wazuh-dashboard: active (running), correct post-sync timestamp
- journalctl confirms systemd-time-wait-sync completes before any
  wazuh-* service begins starting, on all test boots

## Security Considerations
- Accurate timestamps on log ingestion are a correctness requirement
  for a SIEM, not just a cosmetic concern — pre-sync-window events
  could otherwise carry misleading timestamps in correlation/alerting
- File-editing discipline lesson: nano silently discarded two drop-in
  edits mid-session (Ctrl+X without confirming save). Heredoc-based
  file writes (`tee ... << 'EOF'`) are now the standard method for
  all systemd override files, consistent with the project's existing
  heredoc convention for documentation files
- Undervoltage warning (hwmon: Undervoltage detected!) observed in
  kernel log during troubleshooting, not yet root-caused — flagged
  for future investigation as a potential contributor to
  timing-sensitive boot failures generally
