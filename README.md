# Self-Hosted SOC Stack for Game Server Hosting Infrastructure

A from-scratch Security Operations Center build for a Bangladesh-based Minecraft/game
server hosting provider — designed, deployed, and tuned solo as their early-stage
infrastructure security function.

> **Status:** SIEM and EDR layers built, validated, and tuned on a staging environment.
> IDS (Suricata) and the production monitoring/alerting workflow are tracked as the
> next phase — see [Roadmap](#roadmap).

---

## Why This Exists

The company hosts third-party Minecraft servers via Pterodactyl Panel on a fleet of
Linux VPS nodes. Before this project, there was no centralized logging, no file
integrity monitoring, no intrusion detection, and no structured way to answer "did
something just get compromised" beyond manually SSHing into a box. This project
builds that capability from the ground up, on a self-hosted budget (no paid SIEM/EDR
licensing), sized for a small hosting operation rather than an enterprise.

Everything here was first validated on an isolated staging VPS pair before being
considered for the production fleet — config decisions, failure modes, and fixes
documented as they happened rather than written up after the fact.

## What's Actually Built (vs. What's Documented as Reference)

| Layer | Tool | Status |
|---|---|---|
| SIEM | Wazuh (Manager + Indexer + Dashboard) | ✅ Deployed, hardened, validated |
| FIM / Host Intrusion Detection | Wazuh syscheck + auditd (whodata) | ✅ Deployed, tuned for game server file volume |
| Container Monitoring | Wazuh docker-listener + logcollector | ✅ Deployed, validated against live Docker events |
| EDR | Velociraptor (Server + Client) | ✅ Deployed, client enrolled, custom hunt artifacts authored |
| Dashboard Layer | Grafana → OpenSearch | ✅ Deployed, connected as a second query layer over Wazuh's data |
| IDS | Suricata | 🔜 Next phase — architecture decided, not yet deployed |
| Alerting / Notification | Discord/Slack webhook integration | 🔜 Next phase |
| Production Monitoring Runbook | Alert triage, on-call procedures | 🔜 Separate project (post-restoration) |

## Architecture

```
                          MONITORED NODE
                          (game server VPS — Pterodactyl Wings)
                          ───────────────────────────────────────
                          Wazuh Agent: FIM (whodata) + Docker
                          events + auth logs + command monitoring
                                    │
                                    │ TCP 1514 (encrypted)
                                    ▼
═══════════════════════════════════════════════════════════════
                          SOC MANAGER VPS
═══════════════════════════════════════════════════════════════
                          Wazuh Manager
                          applies detection rules
                                    │
                          Filebeat ships alerts
                                    ▼
                          Wazuh Indexer (OpenSearch)
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                                ▼
          Wazuh Dashboard                       Grafana
          deep alert investigation              custom SOC view


                          PARALLEL INVESTIGATIVE PATH
                          ───────────────────────────
          Velociraptor Client (monitored node)
                    │ TCP 8000 (check-in/control)
                    ▼
          Velociraptor Server (SOC Manager VPS)
          on-demand VQL hunts, live forensics — pulled
          up manually when Wazuh/Grafana flags something
          worth digging into deeper
```

**Design principle:** Wazuh is the tripwire (continuous, rule-based, always watching).
Velociraptor is the flashlight (on-demand, query-driven, pointed wherever an
investigation needs it). They don't replace each other — they cover different jobs.

## Why These Tools, Specifically

- **Wazuh over a commercial SIEM** — fully self-hosted, no per-endpoint licensing
  cost, which matters for a hosting company running this on thin margins. Built on
  OpenSearch so it scales the same way commercial ELK-based SIEMs do.
- **Velociraptor over a commercial EDR** — same cost reasoning, plus VQL gives
  query-time flexibility that's closer to how an actual incident investigation
  happens (ask a new question, don't wait for a vendor to ship a new detection rule).
- **Grafana as a second visualization layer** — Wazuh's own dashboard is built for
  deep alert investigation, not at-a-glance status. Grafana reads the same
  OpenSearch data and gives a faster "is everything okay right now" view without
  duplicating any data pipeline.

## Repo Structure

```
.
├── README.md                          ← you are here
├── docs/
│   ├── 01-wazuh-siem-setup.md         ← manager + agent deployment, full walkthrough
│   ├── 02-wazuh-hardening.md          ← FIM tuning for game server file volume, whodata, Docker
│   ├── 03-velociraptor-edr-setup.md   ← server/client deployment, hunt artifacts
│   ├── 04-grafana-dashboard.md        ← dashboard layer setup
│   ├── 05-troubleshooting-log.md      ← real failures hit during deployment and how they were fixed
│   └── 06-verification-checklist.md   ← how each control was proven to actually work
├── configs/
│   ├── wazuh/
│   │   ├── ossec.conf.sanitized       ← production agent config, IPs/hostnames redacted
│   │   └── local_internal_options.conf
│   └── velociraptor/
│       └── artifacts/                 ← custom VQL hunt artifacts for game server threats
└── scripts/
    └── (automation scripts — see Roadmap)
```

## Skills Demonstrated

- SIEM deployment and tuning at a scale appropriate to actual infrastructure
  constraints (not a textbook default install — file integrity monitoring limits,
  whodata/auditd integration, and Docker event monitoring all required
  troubleshooting real failures, documented in [docs/05-troubleshooting-log.md](docs/05-troubleshooting-log.md))
- EDR deployment and custom detection authoring (VQL artifacts written specifically
  for Pterodactyl-hosted game server threat patterns — backdoored plugins, web
  shells, container escape attempts, unauthorized server config changes)
- Linux systems administration under production constraints (resource budgeting on
  modest VPS hardware, systemd service debugging, file permission / service-account
  troubleshooting across multiple Linux distros' Python environments)
- Security architecture decisions made for cost and operational fit, not just
  technical correctness
- Working solo as the sole infrastructure security function for a live company

## Roadmap

- [ ] Suricata IDS deployment on monitored nodes, feeding into the existing Wazuh
      pipeline (no new ingestion path needed — Wazuh already has built-in
      Suricata decoders)
- [ ] Discord/Slack webhook alerting for high-severity events
- [ ] Bash automation scripts for repeatable manager/agent deployment (manual
      install was deliberately done first — see [docs/05-troubleshooting-log.md](docs/05-troubleshooting-log.md)
      for why understanding the failure modes mattered more than speed here)
- [ ] Production monitoring runbook — alert triage procedures, escalation paths,
      on-call documentation (separate repo, once the company's broader systems
      restoration is complete)

## Notes on Scope and Confidentiality

This repo documents infrastructure security work for a real hosting company.
IP addresses, hostnames, and any company-identifying infrastructure details have
been redacted or replaced with placeholders throughout. Configs included here are
sanitized versions of what's actually deployed — structurally identical, but with
no information that could be used to locate or target the real infrastructure.
