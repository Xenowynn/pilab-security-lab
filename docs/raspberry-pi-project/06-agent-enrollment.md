# 06 - Wazuh Agent Enrollment (macOS)

**Date:** 2026-10-04

## Objective
Enroll the MacBook Pro as a monitored Wazuh agent, establishing the
first real detection surface in the SIEM deployment.

## Steps Completed
- Attempted standard password-based auto-enrollment via wazuh-authd
  (registration password set, Tailscale-restricted network access).
  Enrollment consistently failed with "Invalid request for new
  agent" despite verified-correct message format, byte-exact
  password file (confirmed via hex dump), and a single confirmed
  authd process — root cause not identified; suspected bug or
  undocumented requirement in wazuh-authd's request parser
  (auth.c w_auth_parse_data) for this version/platform combination
- Abandoned authd debugging after exhausting reasonable diagnostic
  steps (message-format verification against source, password
  file byte-verification, process-identity confirmation) —
  deliberate decision to avoid unproductive time sink
- Switched to manual key-based enrollment via manage_agents:
  generated agent key on manager (Pi), imported identical key on
  agent (Mac) via manage_agents' Import function
- Started agent via launchctl; confirmed connection on port 1514
  (data channel), bypassing the broken 1515 (authd/registration)
  path entirely

## Commands Used
- sudo /var/ossec/bin/manage_agents (Pi) - Add agent, Extract key
- sudo /Library/Ossec/bin/manage_agents (Mac) - Import key
- sudo launchctl bootout system/com.wazuh.agent (Mac)
- sudo launchctl bootstrap system /Library/LaunchDaemons/com.wazuh.agent.plist (Mac)
- sudo /var/ossec/bin/agent_control -l (Pi, validation)

## Validation Results
- Agent log (Mac): clean connection sequence, no registration
  errors -- "Trying to connect to server" followed by normal FIM/
  syscollector startup and a "reloading due to shared configuration
  changes" message confirming live manager communication
- Manager (Pi) agent_control -l: Agent 001 "macbook-pro-2019"
  listed with status Active -- authoritative confirmation from
  the server side, not just agent self-reporting

## Security Considerations
- Primary access control remains the Tailscale ACL restricting
  network reachability to the Pi (established Section 03) -- the
  authd password was a secondary layer that failed; manual key
  enrollment relies on the same Tailscale-restricted network path
  plus a unique per-agent key, so security posture is not weakened
  by this pivot
- Known limitation: authd password-based auto-enrollment is
  non-functional on this deployment for an unidentified reason --
  flagged for future investigation if additional agents are added
  later and auto-enrollment becomes worth revisiting at scale
