# Wazuh Hardening for a Game Server Hosting Workload

The default Wazuh agent config is tuned for a generic Linux server. A game server
host running Docker-containerized Pterodactyl instances breaks several of those
defaults almost immediately. This document covers what broke, why, and how the
config was iteratively corrected — including the dead ends, because the dead ends
are most of what's worth learning from this.

## The Core Problem: Default FIM File Limits Are Sized for the Wrong Workload

Wazuh's File Integrity Monitoring (FIM) module has a hard cap on how many files it
will track (`<file_limit>`). The out-of-the-box default assumes a relatively static
Linux server — a few system directories, maybe a web root. A game server VPS running
Docker is nothing like that:

- Docker's `overlay2` storage driver alone can generate 50,000–200,000+ tracked
  files per container layer.
- Game server data — Steam downloads, Proton prefixes, plugin caches, world data —
  adds tens of thousands more.

The first config deployed here hit its 100,000-file cap within roughly an hour of
going live, logged as a level-12 Wazuh alert:

```
rule.description: The maximum limit of files monitored has been reached.
At this moment there are 100000 files and the limit is 100000.
From this moment some events can be lost.
```

**Fix, in stages:**

| Iteration | `file_limit` | Outcome |
|---|---|---|
| Default | 100,000 | Capped within ~1 hour |
| First fix | 500,000 | Better, but still tight once Pterodactyl volumes were added |
| Final | 1,000,000 | Stable, with aggressive `<ignore>` rules to keep noise down (see below) |

Raising the limit alone isn't the right fix on its own — it just delays hitting the
ceiling. The real fix is pairing a realistic limit with **deliberate exclusions** so
Wazuh isn't wasting tracked-file budget on data that doesn't need integrity
monitoring (container layers, Steam caches, world saves):

```xml
<!-- Docker internals — never need FIM, change constantly, not security-relevant -->
<ignore>/var/lib/docker/overlay2</ignore>
<ignore>/var/lib/docker/containers</ignore>
<ignore>/run/containerd</ignore>
<ignore>/run/docker.sock</ignore>

<!-- Pterodactyl: game data and caches, not configuration -->
<ignore>/var/lib/pterodactyl/volumes/*/world*</ignore>
<ignore>/var/lib/pterodactyl/volumes/*/cache/</ignore>
<ignore>/var/lib/pterodactyl/volumes/*/libraries/</ignore>
<ignore>/var/lib/pterodactyl/volumes/*/steamapps/</ignore>
<ignore>/var/lib/pterodactyl/backups/</ignore>
```

## What Actually Gets Real-Time + Whodata Monitoring

The opposite side of the same tuning decision: a short list of paths where a change
is *always* worth knowing about immediately, with full user/process attribution
(whodata — see below):

- `/etc/ssh`, `/root/.ssh` — key injection
- `/etc/sudoers`, `/etc/sudoers.d` — privilege escalation
- `/etc/systemd/system` — persistence mechanisms
- `/etc/docker` — container runtime tampering
- Pterodactyl panel web root — web shell upload
- Per-server `ops.json`, `whitelist.json`, `banned-players.json`,
  `server.properties` — in-game privilege abuse, ban evasion
- Plugin and mod directories — backdoored plugin uploads

Everything else either gets a scheduled (non-real-time) scan or is excluded
entirely. This is the actual performance/security tradeoff at the center of the
whole config: real-time + whodata is expensive per-path, so it's reserved for paths
where a compromise would actually show up.

## Whodata: Knowing *Who*, Not Just *What*

Wazuh's `realtime="yes"` mode tells you a file changed. `whodata="yes"` additionally
tells you which user, which process, and which parent process made the change — the
difference between "ops.json changed" and "ops.json was changed by `java` running as
`pterodactyl` under PID 4821, invoked by the panel's plugin manager."

Whodata requires Linux's `auditd` running underneath it. This was the single most
time-consuming part of the hardening work — not because the concept is complex, but
because of a chain of environment-specific failures:

1. **`auditd` wasn't installed at all initially.** Wazuh logged:
   ```
   WARNING: (6913): Who-data engine cannot start because Auditd is not running.
   Switching who-data to real-time.
   ```
   Silent degradation — FIM kept working, just without attribution. Easy to miss if
   you're not specifically checking for it.

   Fix: `sudo apt install auditd audispd-plugins -y && sudo systemctl enable --now auditd`

2. **After installing auditd, some audit rules still failed to register**, logged as:
   ```
   WARNING: (6925): Unable to add audit rule for '/var/lib/pterodactyl/.config'
   ```
   Root cause was simpler than it looked: those specific paths didn't exist yet on
   the *staging* node (Pterodactyl wasn't installed there — it's a test box for
   validating the Wazuh config itself, with the real Pterodactyl paths only present
   on production nodes). Confirmed with `ls -la <path>`, then removed the
   non-existent paths from the staging config rather than chasing a non-issue.

This is a good example of why staging validation matters: a warning that looks like
a Wazuh bug is sometimes just a config written for production being run against an
environment that doesn't fully match it yet.

## Docker Event Monitoring: A Multi-Layer Permissions Problem

Getting Wazuh's `docker-listener` module to actually stream container lifecycle
events (container created/started/stopped, privileged container launched, sensitive
host paths mounted) took several iterations, because the failure was silent — the
module would start, then exit ~15 seconds later with no error in the default logs:

```
docker-listener: INFO: Module docker-listener started.
docker-listener: INFO: Starting to listening Docker events.
docker-listener: INFO: Module finished.        <- crash, no error shown
```

**Layer 1 — group membership.** The `wazuh` service account needs to be in the
`docker` group to talk to the Docker socket:
```bash
sudo usermod -aG docker wazuh
```
This alone wasn't sufficient — group membership changes don't apply to an
already-running service without a restart (or in some cases, a full reboot if the
service was started before the group existed).

**Layer 2 — missing Python `docker` module.** The docker-listener is a Python
subprocess. `pip3` wasn't even installed on the staging box initially
(`pip3 not found`), so the first fix was distro-level:
```bash
sudo apt install python3-pip python3-docker -y
```

**Layer 3 — the real problem: multiple Python interpreters.** `apt`'s
`python3-docker` installs to the system Python at `/usr/lib/python3.12`. But this
particular VPS image had a from-source Python build at `/usr/local/bin/python3`
which took precedence in `$PATH` — so the apt-installed module was invisible to
whatever Python actually ran when Wazuh spawned the listener:
```bash
sudo -u wazuh python3 -c "import docker; print(docker.__version__)"
# ModuleNotFoundError: No module named 'docker'
```
Fix was installing the module against the *correct* interpreter explicitly:
```bash
sudo /usr/local/bin/python3 -m ensurepip --upgrade
sudo /usr/local/bin/python3 -m pip install docker==7.1.0 urllib3==1.26.20 requests==2.32.2
```

**Layer 4 — socket permissions**, the more standard fix:
```bash
sudo chmod 660 /var/run/docker.sock
sudo chown root:docker /var/run/docker.sock
```

After all four layers and a full agent restart, `docker-listener` stayed running
and the process tree showed it persisting rather than cycling:
```
└─24983 python3 wodles/docker/DockerListener
```

Validated by triggering real Docker events and confirming alerts on the manager
side — privileged container creation, sensitive host-path bind mounts (`/etc`,
the Docker socket itself), and host namespace sharing (`--pid=host`,
`--net=host`) all fire distinct, correctly-categorized alerts.

## The Setting That Silently Disables Most Command-Based Monitoring

Wazuh can run shell commands on a schedule and treat their output as a log source
(used here for `docker stats`, load average, listening port counts, zombie process
counts). None of this works unless one specific internal option is set — and Wazuh
does not error or warn if it's missing, it just silently ignores every
`<command>` block in the config:

```bash
echo "logcollector.remote_commands=1" | sudo tee -a /var/ossec/etc/local_internal_options.conf
sudo systemctl restart wazuh-agent
```

This was the single highest-impact one-line fix in the entire hardening process —
before it was set, FIM, Docker monitoring, and auth logs all looked fine, but every
command-based check (system load, listening ports, container stats) was producing
no data at all with zero indication why.

## A Config Element That Doesn't Exist in This Wazuh Version

`<diff_size_limit>` (intended to cap how much of a changed file's diff gets stored)
produced a startup warning on every restart:
```
WARNING: (1230): Invalid element in the configuration: 'diff_size_limit'.
```
Wazuh 4.14.5 doesn't support this element under `<syscheck>` — it was present from
an earlier reference config that targeted a different version. Removed rather than
left as permanent log noise:
```bash
sudo sed -i '/diff_size_limit/d' /var/ossec/etc/ossec.conf
```

## Resource Budget

Sized against a 2GB RAM allowance, since this runs alongside the actual game server
workload on the same node and can't be allowed to compete meaningfully for
resources:

| Component | Observed |
|---|---|
| Wazuh agent (all modules) | ~370 MB RAM steady-state |
| Velociraptor client | ~50 MB RAM steady-state |
| Disk (FIM database) | 300–500 MB |
| CPU, idle monitoring | 1–5% |
| CPU, during scheduled scans | 10–30%, for 1–5 minutes |

Comfortably inside budget with headroom for burst load during an actual incident
(which is exactly when you don't want monitoring itself to be the thing eating CPU).

## Detection Speed Reference

What "real-time + whodata" actually buys, measured against the paths configured
above:

| Event | Detection Time |
|---|---|
| SSH key injection, sudoers edit, systemd persistence | < 5 seconds |
| Docker config tampering, privileged container launch | < 5 seconds |
| Plugin/mod directory changes, `ops.json` tampering | < 5 seconds |
| New listening port, new login user (command-based) | up to 6 minutes (polling interval) |
| Rootkit signature (rootcheck) | up to 6 hours (scheduled scan) |
| Configuration drift (SCA baseline) | up to 12 hours (scheduled scan) |

The gap between "instant" and "hours" is intentional, not an oversight — real-time
whodata monitoring on every path would exceed the resource budget above. The paths
that got real-time treatment were chosen specifically because they're the highest
signal-to-noise attack surface for this workload.
