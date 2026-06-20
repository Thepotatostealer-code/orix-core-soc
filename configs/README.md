# Configs

Sanitized, production-structure configuration files referenced throughout `/docs`.

## Sanitization Approach

Every file here is structurally identical to what's actually deployed, with the
following replaced by placeholders:

| Real value | Placeholder |
|---|---|
| Manager VPS IP | `MANAGER_IP_REDACTED` |
| Game server panel/volume paths | `GAMESERVER_PANEL_ROOT`, `GAMESERVER_VOLUMES_ROOT`, `PANEL_WEB_ROOT` |
| Backup storage paths | `PTERODACTYL_BACKUPS_ROOT` |

No hostnames, real IPs, or path structures that could be used to fingerprint or
locate the actual infrastructure are present in this directory.

## Contents

- `wazuh/ossec.conf.sanitized` — production agent config, see
  [docs/02-wazuh-hardening.md](../docs/02-wazuh-hardening.md) for the reasoning
  behind every section
- `wazuh/local_internal_options.conf` — the one-line fix for silently-ignored
  command-based monitoring, see
  [docs/05-troubleshooting-log.md](../docs/05-troubleshooting-log.md)
- `velociraptor/artifacts/` — custom VQL hunt definitions for Pterodactyl-hosted
  game server threats, see
  [docs/03-velociraptor-edr-setup.md](../docs/03-velociraptor-edr-setup.md)
