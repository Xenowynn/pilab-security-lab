# 07 - Fail2Ban to Wazuh Integration

**Date:** 2026-10-04
**Project:** 1 - SIEM and Threat Detection Lab
**Status:** Complete

## Objective
Turn Fail2Ban ban events on the Pi (sshd jail, port 2222) into structured, searchable Wazuh alerts: offending IP, jail, action, and timestamp as separate fields, with MITRE ATT&CK tagging. First working piece of the collection -> detection -> (future) enrichment -> (future) response pipeline.

## Steps Completed
1. Pre-session check: Tailscale up on Mac and Pi, SSH reachable, all five services active (wazuh-manager, wazuh-indexer, wazuh-dashboard, filebeat, fail2ban).
2. Simulated a ban with `fail2ban-client` using a TEST-NET-3 address to generate a real log line without banning the Mac's Tailscale IP.
3. Tested the line with `wazuh-logtest`: no decoder matched (no stock fail2ban decoder exists in 4.14.7).
4. Wrote a custom decoder, debugged it with probe decoders, and found the pre-decoder strips the timestamp and module name before decoders run.
5. Wrote rules 100100 (base, level 0), 100101 (ban, level 8, MITRE T1110), 100102 (unban, level 3).
6. Added a `<localfile>` block for /var/log/fail2ban.log to the manager's ossec.conf (backup taken first), validated with `wazuh-analysisd -t`, restarted the manager.
7. Triggered a live ban and confirmed the alert in alerts.json and in the dashboard (Discover, wazuh-alerts-*, `rule.id:100101`).
8. Unbanned the test IPs and confirmed rule 100102 fired for both.

## Config Files (in this repo)
- `configs/wazuh/fail2ban_decoders.xml` -> deploy to `/var/ossec/etc/decoders/` (owner wazuh:wazuh, mode 660)
- `configs/wazuh/fail2ban_rules.xml` -> deploy to `/var/ossec/etc/rules/` (owner wazuh:wazuh, mode 660)
- ossec.conf addition (localfile block):

```xml
<ossec_config>
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/fail2ban.log</location>
  </localfile>
</ossec_config>
```

## Key Commands
```bash
sudo fail2ban-client set sshd banip 203.0.113.51      # simulate a ban safely
sudo /var/ossec/bin/wazuh-logtest                      # test decoders and rules without a restart
sudo /var/ossec/bin/wazuh-analysisd -t                 # validate config before a slow restart
sudo systemctl restart wazuh-manager
sudo grep '"id":"100101"' /var/ossec/logs/alerts/alerts.json | tail -n 1
sudo fail2ban-client set sshd unbanip 203.0.113.51
```

## Validation Results
- logtest: ban line decoded to jail=sshd, action=Ban, srcip=203.0.113.50; rule 100101 fired at level 8 with MITRE Credential Access / Brute Force (T1110). Unban line fired 100102 at level 3.
- ossec.log: `wazuh-logcollector: INFO: (1950): Analyzing file: '/var/log/fail2ban.log'`.
- Live test (203.0.113.51): alert in alerts.json about one second after the log line, with correct srcip, jail, action, level 8.
- Dashboard: `rule.id:100101` returned the alert with data.srcip, data.jail, data.action, rule.level 8, and MITRE fields all populated.
- Cleanup: `Currently banned: 0`; two 100102 alerts recorded.

## Troubleshooting Notes
- The pre-decoder strips the leading timestamp AND the module name (`fail2ban.actions`) before decoders run. Prematch must anchor on text that survives (`[PID]: LEVEL`). Found by loading one probe decoder per line fragment and seeing which matched first.
- `<prematch>` and `<regex>` each need `type="pcre2"` when using PCRE escapes; the default engine rejects `\[`.
- `action` is a static field: it needs the `<action>` tag, not `<field name="action">`. Anchored values (`^Ban$`) did not match; plain `Ban` did.
- Terminal echo of pasted heredocs can look garbled even when the file is correct. Verify the file on disk instead of trusting the echo.

## Security Considerations
- Simulated bans use 203.0.113.0/24 (TEST-NET-3, never routed) so nothing real is affected and the Mac's Tailscale IP is never banned.
- The manager reads the log as root via logcollector; the decoder and rules files are wazuh:wazuh, mode 660, with no secrets in them.
- ossec.conf backup kept on the Pi (`ossec.conf.bak-sec07`) for one-command rollback.

## Known Limitations
- Decoder prematch is generic (`[PID]: LEVEL`); rules are not restricted to the Fail2Ban log path, so a lookalike line from another source could trigger them. To tighten after the log feed is stable.
- Regex extracts IPv4 only; IPv6 bans would not populate srcip.
- Only Ban and Unban have rules. Other actions (e.g. Found) decode but stay at base rule level 0.
- `fail2ban` appears twice in rule.groups (file-level and rule-level group names). Cosmetic.

## What You Just Built
The Pi now watches its own intrusion-prevention log. When Fail2Ban blocks an IP, Wazuh reads the log line, pulls out the IP, the jail, and the action as separate fields, and raises a severity-8 alert tagged as MITRE brute force. That alert is searchable in the dashboard by IP, jail, or rule. It is the first real detection in the lab, and the point where MISP IP enrichment and a Shuffle response can plug in later.
