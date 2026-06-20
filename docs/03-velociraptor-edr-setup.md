# Velociraptor EDR Setup

Covers Velociraptor server deployment, client enrollment, and the custom VQL
detection artifacts written specifically for Pterodactyl-hosted game server threats.

## Velociraptor vs. Wazuh: Why Both

Wazuh is continuous and rule-based — it watches log sources and the filesystem
constantly and fires an alert the moment something matches a known-bad pattern.
Velociraptor is on-demand and query-driven — an investigator (or a scheduled hunt)
actively asks an endpoint a question in real time: "show me every process consuming
more than 50% CPU," "find every PHP file containing `eval(`," "list every cron job
referencing `curl` or `wget`."

```
Wazuh:          passive, continuous, rule-matching      → the tripwire
Velociraptor:   active, on-demand, investigator-driven   → the flashlight
```

The two don't share a data pipeline by default — Velociraptor doesn't feed into
OpenSearch/Wazuh's alert stream. The intended workflow is: Wazuh flags something →
an analyst opens Velociraptor and runs a targeted hunt on the affected endpoint to
get the deeper forensic picture.

## Server Install

Pulled from GitHub's latest-release API rather than a pinned version string — fast
moving security tooling like this has shipped real CVEs around input validation and
permission handling in past versions, so hardcoding a version means the deployment
silently goes stale and potentially vulnerable:

```bash
curl -s https://api.github.com/repos/Velocidex/velociraptor/releases/latest \
  | grep "browser_download_url.*linux-amd64\"" \
  | cut -d '"' -f 4 \
  | wget -i -
mv velociraptor-*-linux-amd64 velociraptor
chmod +x velociraptor
sudo mv velociraptor /usr/local/bin/velociraptor
```

Config generation is interactive and produces two distinct files:

```bash
sudo mkdir -p /etc/velociraptor && cd /etc/velociraptor
sudo velociraptor config generate -i
```

```
server.config.yaml    server identity, private keys — stays on the manager VPS only
client.config.yaml    what gets baked into client packages and deployed to nodes
```

This split matters operationally: `server.config.yaml` is the single most sensitive
file in the whole stack. Losing it means re-enrolling every client from scratch.

## A Multi-Layer Permissions Failure (Worth Documenting Because It Recurred)

The Velociraptor server systemd unit runs as a dedicated `velociraptor` user, not
root. Every directory the service needs to touch has to be explicitly owned by that
user — and the failure mode for each one was a different cryptic crash, not a
consistent "permission denied":

**Failure 1 — config unreadable:**
```
velociraptor: error: frontend: Unable to load config file: open
/etc/velociraptor/server.config.yaml: permission denied
```
The config was `600 root:root`. Fixed ownership and the parent directory (which was
itself `drwx--x--x`, blocking even directory listing by the service account):
```bash
sudo chown velociraptor:velociraptor /etc/velociraptor/server.config.yaml
sudo chmod 640 /etc/velociraptor/server.config.yaml
sudo chown velociraptor:velociraptor /etc/velociraptor
sudo chmod 750 /etc/velociraptor
```

**Failure 2 — different error after the first fix:**
```
panic: Unable to create logging directory.
```
The config's `Logging.output_directory` pointed to `/var/lib/velociraptor/logs`,
which didn't exist yet and wasn't owned by the service account when created:
```bash
sudo mkdir -p /var/lib/velociraptor/logs
sudo chown -R velociraptor:velociraptor /var/lib/velociraptor
```

The pattern worth taking away: systemd `User=` directives are a real hardening
control, not just a formality — but every directory the service touches needs the
same ownership discipline, and Velociraptor's error messages don't always make it
obvious which directory is the current blocker. Checking `journalctl -u
velociraptor_server -n 50` after every attempted fix was what actually moved this
forward, rather than guessing at the full set of required directories up front.

## Client Enrollment

The Velociraptor frontend (client check-in port, 8000) defaults to binding
`127.0.0.1` only — correct for a server you're only ever going to manage via SSH
tunnel to the GUI, but it also means client nodes outside `localhost` can't reach it
until the bind address is corrected to `0.0.0.0` (with the firewall restricting who
can actually reach that port).

Client packages are generated server-side and transferred to each node:

```bash
sudo /usr/local/bin/velociraptor --config server.config.yaml \
  debian client --output /root/velociraptor-client.deb
```

```bash
# on the monitored node
sudo dpkg -i velociraptor-client.deb
sudo systemctl enable --now velociraptor_client
```

**One enrollment failure worth noting:** the client repeatedly crash-looped with:
```
velociraptor_client: error: client: Unable to load config file:
No Client.ca_certificate configured
```
The `client.config.yaml` generated via `config client` extraction was missing its
CA certificate section — root cause wasn't fully isolated, but the reliable fix was
using the *full* server config as the client config instead of the extracted
subset. The Velociraptor client binary only reads its own `Client{}` section
regardless of what else is in the file, so handing it the complete server config
(transferred securely, then access-restricted on the client) gets all the required
crypto material without needing the extraction step to work correctly:
```bash
sudo cp server.config.yaml /tmp/client.config.yaml
# transfer to node, then on node:
sudo cp client.config.yaml /etc/velociraptor/client.config.yaml
sudo systemctl restart velociraptor_client
```

Confirmed enrollment via the GUI (Clients view) and via server-side logs:
```
"msg":"Enrolling","time":"..."
```

## GUI Access (Why It's SSH-Tunneled, Not Public)

The Velociraptor web GUI (port 8889) is intentionally not exposed publicly. Access
is via SSH local port forwarding:
```bash
ssh -L 8889:localhost:8889 <user>@<manager-host>
```
then `https://localhost:8889` in a local browser. This keeps the EDR control plane
off the public internet entirely — there's no operational reason an investigator
needs to reach it from anywhere other than an already-authenticated SSH session, and
every additional exposed surface on a security tool is itself a target.

## Custom Detection Artifacts

Velociraptor's equivalent of a Wazuh rule is an **artifact** — a named, reusable VQL
query (or set of queries) that can be run on-demand or scheduled as a recurring
hunt. Five were authored specifically for Pterodactyl game server threat patterns;
full VQL is in [`configs/velociraptor/artifacts/`](../configs/velociraptor/artifacts/).

| Artifact | Detects |
|---|---|
| `GameServer.Detect.BackdoorPlugin` | Recently-modified or suspiciously-named `.jar` plugins, with a YARA pass for known backdoor indicators (`Runtime.exec`, `ProcessBuilder`, suspicious network calls) |
| `GameServer.Detect.WebShell` | Pterodactyl panel PHP files containing shell-execution patterns (`eval(`, `base64_decode`, `shell_exec`, `passthru`) |
| `GameServer.Detect.Miner` | High-CPU processes matching known cryptominer command-line patterns, plus cron entries referencing common dropper commands |
| `GameServer.Detect.ContainerEscape` | Privileged container launches and sensitive host-path mounts via `docker inspect` |
| `GameServer.Detect.UnauthorizedOP` | Recent changes to `ops.json`, `whitelist.json`, and `server.properties` across all hosted server volumes |

These artifacts are templates — they define *what to look for*, not a continuous
watch. Turning one into ongoing coverage means scheduling it as a recurring hunt
(e.g. miner detection every 5 minutes, web shell sweep hourly) or, for true
real-time coverage, configuring it as a Client Monitoring event artifact instead of
a scheduled hunt. Hunt scheduling and the CLI-vs-GUI workflow for that is still
being finalized — tracked in the main repo [Roadmap](../README.md#roadmap).

## What Velociraptor Deliberately Doesn't Do Here

No built-in alerting (no email/webhook on hunt results) — that's intentionally left
to the Wazuh side of the stack, which already owns alerting. Velociraptor's role
stays scoped to investigation: Wazuh says something happened, Velociraptor answers
exactly what.
