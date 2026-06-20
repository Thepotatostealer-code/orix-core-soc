# Grafana Dashboard Layer

A second visualization layer on top of Wazuh's own dashboard, reading the same
underlying data.

## Why Add Grafana When Wazuh Already Has a Dashboard

Wazuh's bundled dashboard (a fork of OpenSearch Dashboards, itself a fork of Kibana)
is built for deep alert investigation — drilling into a specific event, tracing a
rule match back through raw log data. It's not built for an at-a-glance "is
everything okay right now" operational view, which matters more day-to-day for a
one-person security function watching infrastructure across multiple nodes.

Grafana doesn't duplicate any data pipeline to get this — it queries the same
OpenSearch instance Wazuh's Indexer already populates:

```
Wazuh Dashboard ──┐
                   ├──▶ OpenSearch (localhost:9200 only)
Grafana ───────────┘
```

Neither tool stores its own copy of the security data. Grafana's only local state is
its own dashboard *definitions* (panel layouts, saved queries) — if Grafana were
wiped and reinstalled, all historical alert data would still be intact in
OpenSearch; only the dashboard layouts would need rebuilding.

## Install

```bash
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt update
sudo apt install -y grafana
sudo systemctl enable --now grafana-server
```

```bash
sudo ufw allow 3000/tcp comment 'Grafana web UI'
```

## Connecting to the Wazuh Indexer

```
Connections → Data Sources → Add data source → OpenSearch

URL:        https://localhost:9200
Auth:       Basic auth (same admin credentials as the Wazuh indexer)
Index name: wazuh-alerts-*
Time field: @timestamp
TLS:        Skip verify (self-signed cert, internal-only deployment)
```

That single connection is sufficient to visualize every alert flowing through the
stack — Wazuh FIM/auth/Docker events now, Suricata IDS alerts once that phase is
deployed (Wazuh already decodes Suricata's `eve.json` output natively, so no new
ingestion path is needed for Grafana to pick those up too).

## What Lives Where

```
/etc/grafana/grafana.ini       main config (port, auth)
/var/lib/grafana/grafana.db    Grafana's own sqlite DB — dashboard layouts, users
/var/log/grafana/grafana.log   operational log
```

Worth backing up `grafana.db` separately once dashboards are built out — it's small,
self-contained, and is the only piece of this layer that isn't already durable in
OpenSearch.
