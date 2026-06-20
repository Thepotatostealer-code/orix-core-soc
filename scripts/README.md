# Scripts

Automation scripts for repeatable manager/agent deployment.

**Status: not yet built — tracked in the main [Roadmap](../README.md#roadmap).**

The manual setup documented in `/docs` was deliberately done first, before writing
any automation. A meaningful fraction of that manual process involved diagnosing
non-obvious, environment-specific failures (silently-ignored config options,
multiple-Python-interpreter conflicts, service-account permission chains across
several directories) — see
[docs/05-troubleshooting-log.md](../docs/05-troubleshooting-log.md). Writing a
deployment script before understanding those failure modes firsthand would have
meant baking in untested assumptions about what "correct" looks like.

Planned scripts, once written:

- `deploy-manager.sh` — Wazuh Manager + Indexer + Dashboard + Velociraptor Server +
  Grafana, idempotent (safe to re-run), with conditional resource sizing based on
  detected RAM rather than hardcoded values
- `deploy-agent.sh` — Wazuh Agent + Velociraptor Client + auditd, with the
  Python-interpreter and Docker-permission fixes from the troubleshooting log
  applied automatically rather than left as manual steps
- `verify-stack.sh` — automated version of the
  [verification checklist](../docs/06-verification-checklist.md)
