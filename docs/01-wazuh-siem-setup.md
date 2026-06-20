# Wazuh SIEM Setup

Covers deployment of the Wazuh Manager stack (Manager + Indexer + Dashboard) on a
dedicated SOC manager VPS, and enrollment of the first monitored agent node.

## Why the All-in-One Installer

Wazuh's three components — Manager, Indexer (an OpenSearch fork), and Dashboard (an
OpenSearch Dashboards fork) — all authenticate to each other via TLS certificates
that must be mutually trusted. Generating those certs by hand and wiring three
separate services together manually is the single most common source of "dashboard
won't load" failures in self-hosted Wazuh deployments. The official all-in-one
installer handles certificate generation, repo setup, and correct service start
order in one pass, so that's what was used here rather than reinventing it.

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
chmod +x wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

The `-a` flag installs all three components on a single host — appropriate for this
scale of operation (single hosting company, modest node count) rather than splitting
Manager/Indexer/Dashboard across separate hosts, which only becomes necessary at much
higher alert volume.

## Pre-Install: Why It Matters Before Touching Wazuh

Two things will silently break Wazuh if skipped:

- **Clock sync.** TLS certificate validation is time-sensitive. If the VPS clock has
  drifted, agent enrollment can fail with no obviously-clock-related error message.
- **Hostname set before install.** The installer bakes the hostname into generated
  certs.

```bash
sudo hostnamectl set-hostname soc-manager
sudo timedatectl set-timezone Asia/Dhaka
timedatectl status   # confirm "System clock synchronized: yes"
```

## Firewall

Opened incrementally, one port per service, rather than blanket-opening a list —
mainly so each rule has a clear, intentional reason rather than being cargo-culted
from a guide.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp     comment 'SSH'
sudo ufw allow 1514/tcp   comment 'Wazuh agent data'
sudo ufw allow 1515/tcp   comment 'Wazuh agent enrollment'
sudo ufw allow 55000/tcp  comment 'Wazuh REST API'
sudo ufw allow 443/tcp    comment 'Wazuh dashboard HTTPS'
sudo ufw enable
```

Port 9200 (the Indexer's actual OpenSearch port) is **deliberately not opened**.
Grafana and the Wazuh Dashboard both query it over localhost only — there's no
legitimate reason for 9200 to be internet-facing, and leaving it closed removes an
entire class of "someone found my OpenSearch instance on Shodan" risk.

## Resource Sizing

The installer defaults the Indexer's JVM heap to roughly half of system RAM. On a
budget-tier staging VPS that default was too aggressive and needed capping:

```bash
sudo nano /etc/wazuh-indexer/jvm.options
```

```
-Xms1g
-Xmx1g
```

```bash
sudo systemctl restart wazuh-indexer
```

## Verifying the Manager Stack

```bash
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
curl -k -u admin:<password> "https://localhost:55000/?pretty=true"
curl -k -u admin:<password> "https://localhost:9200/_cluster/health?pretty"
```

All three services need `active (running)`. If the Indexer fails to start, check
`journalctl -u wazuh-indexer -n 50` first — in this deployment it was the JVM heap
sizing issue above nearly every time.

## Agent Enrollment (Monitored Node Side)

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.5-1_amd64.deb
sudo WAZUH_MANAGER='<manager-ip>' WAZUH_AGENT_NAME='<node-name>' dpkg -i ./wazuh-agent_4.14.5-1_amd64.deb
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Then `sudo systemctl status wazuh-agent` to confirm it's running, and check the
manager side for the agent showing as `active`:

```bash
sudo /var/ossec/bin/agent_control -l
```

## What's Out of Scope for This Document

The default Wazuh install ships with most of its deeper security modules either
disabled or set to defaults that don't fit a game-hosting workload (default file
integrity monitoring limits get exhausted almost immediately by Docker's overlay
filesystem, for instance). That tuning work is documented separately in
[02-wazuh-hardening.md](02-wazuh-hardening.md), since it's substantial enough — and
was iterated on enough — to deserve its own writeup rather than being folded into
the base install steps above.

## Directory Reference

```
/var/ossec/                          Wazuh Manager root
├── etc/ossec.conf                   main manager config
├── etc/rules/                       custom detection rules
├── logs/ossec.log                   manager's own operational log
├── logs/alerts/alerts.json          every alert the manager generates
└── bin/                             wazuh-control, agent_control, etc.

/etc/wazuh-indexer/opensearch.yml    Indexer config
/etc/wazuh-indexer/jvm.options       memory tuning lives here
/var/lib/wazuh-indexer/              actual alert data storage (grows continuously)
/etc/wazuh-dashboard/                Dashboard config
/etc/filebeat/filebeat.yml           ships alerts from Manager → Indexer
```

`alerts.json` and `filebeat.yml` are the two files worth knowing by heart for
troubleshooting — if alerts aren't reaching the dashboard, the question is almost
always "did the event reach `alerts.json` on the manager, and if so, did Filebeat
ship it onward."
