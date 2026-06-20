# Verification Checklist

How each security control was proven to actually function — not just "service is
running," but "this specific detection fires correctly end to end." Documented
because "I installed it" and "I verified it detects what it's supposed to detect"
are different claims, and only the second one is actually useful in an incident.

## Tier 1 — Critical Controls

| Control | Verification Method | Result |
|---|---|---|
| FIM (real-time) | Touched a file in a monitored path (`/etc/ssh/`), confirmed event in agent log within seconds, confirmed corresponding alert on manager's `alerts.json` | ✅ Confirmed, < 5s |
| Whodata attribution | Same FIM test, checked alert JSON for populated `audit.user.name` / `audit.process.name` fields (requires auditd running underneath) | ✅ Confirmed after auditd install |
| Auth log monitoring | Injected a synthetic failed-login event via `logger -p auth.warning`, confirmed manager-side rule match | ✅ Confirmed |
| Rootcheck | Confirmed scheduled scan start/end entries in agent log at expected interval | ✅ Confirmed |
| Active response | Confirmed `active-response/bin/restart.sh` execution entries in agent log following a config reload | ✅ Confirmed |

## Tier 2 — High-Value Controls

| Control | Verification Method | Result |
|---|---|---|
| Docker event monitoring | Ran a sequence of deliberately flaggable container operations (privileged container, sensitive host-path mount, host PID/net namespace sharing) and confirmed each produced a distinct, correctly-categorized alert | ✅ Confirmed, see test matrix below |
| Syscollector | Confirmed scheduled inventory evaluation start/end in agent log | ✅ Confirmed |
| SCA (Security Configuration Assessment) | Confirmed CIS Ubuntu 24.04 policy loaded and scan completing on schedule | ✅ Confirmed |
| Command-based monitoring (load avg, listening ports, zombie processes) | Confirmed `Monitoring output of command` entries in agent log after enabling `remote_commands` | ✅ Confirmed |
| Velociraptor client enrollment | Confirmed client visible in server-side client list, confirmed `Enrolling` event in server log | ✅ Confirmed |
| Velociraptor on-demand query | Ran `Linux.Sys.Pslist` against the enrolled client, confirmed live process list returned | ✅ Confirmed |

## Docker Security Test Matrix

Specific test commands used to validate the Docker monitoring rules — each maps to a
distinct attack technique a compromised game server container might exhibit:

| Test | Command Pattern | Detects |
|---|---|---|
| Privileged container | `docker run --privileged ...` | Container escape risk via full device/capability access |
| Sensitive host mount | `docker run -v /etc:/host-etc ...` | Host filesystem exposure to container |
| Docker socket mount | `docker run -v /var/run/docker.sock:/var/run/docker.sock ...` | Full host compromise vector — container gains control of the Docker daemon itself |
| Host PID namespace | `docker run --pid=host ...` | Container can see/signal host processes |
| Host network namespace | `docker run --net=host ...` | Container bypasses network isolation entirely |

All five produced distinct alerts on the manager, confirming the Docker rule set
distinguishes between routine container lifecycle events and genuinely
security-relevant container configurations.

## Game Server Attack → Detection Mapping

The threat model this stack is actually built against, and which layer catches it:

| Attack | Detected By | Speed |
|---|---|---|
| Backdoored plugin upload | Wazuh FIM (whodata) on plugin directories | < 5 sec |
| Web shell in panel | Wazuh FIM on panel web root | < 5 sec |
| Unauthorized OP / whitelist tampering | Wazuh FIM on `ops.json` / `whitelist.json` | < 5 sec |
| SSH key injection | Wazuh FIM (whodata) on `/root/.ssh/` | < 5 sec |
| Sudoers backdoor | Wazuh FIM on `/etc/sudoers*` | < 5 sec |
| Systemd persistence | Wazuh FIM on `/etc/systemd/system` | < 5 sec |
| Privileged / escaping container | Wazuh docker-listener | < 5 sec |
| Hidden miner process | Wazuh rootcheck (scheduled) + Velociraptor `GameServer.Detect.Miner` (on-demand) | up to 6 hrs scheduled / instant on-demand |
| New listening port, new user account | Wazuh command monitoring / syscollector | up to 6 min / 1 hr |
| Config drift from baseline | Wazuh SCA | up to 12 hrs |
| Deep forensic question ("what exactly did this process touch") | Velociraptor VQL hunt | on-demand, investigator-driven |

The split between "fast automated detection" (Wazuh) and "deep on-demand
investigation" (Velociraptor) is deliberate — see
[03-velociraptor-edr-setup.md](03-velociraptor-edr-setup.md#velociraptor-vs-wazuh-why-both)
for the reasoning.

## What Hasn't Been Verified Yet

Honesty matters more than completeness here:

- Suricata IDS — not yet deployed, so nothing to verify
- End-to-end alert-to-notification (Discord/Slack webhook) — not yet built
- Behavior under actual attack conditions on the production fleet — everything
  above was validated on an isolated staging node; production validation is part of
  the next phase once the company's broader systems restoration is complete
