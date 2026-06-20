# Troubleshooting Log

A chronological record of real failures hit during this deployment and how each was
diagnosed and fixed. Kept separate from the setup docs because the diagnostic
process — not just the final fix — is the part worth showing: most of these issues
had a misleading first symptom, and the actual root cause took a couple of
iterations to isolate.

This is also, practically, the most useful document in this repo for anyone running
into the same stack: searching "wazuh docker-listener module finished" or
"velociraptor unable to create logging directory" is more likely to land someone
here than on a polished setup guide.

---

## Issue: FIM File Limit Exhausted Within an Hour

**Symptom:** Level-12 Wazuh alert within roughly an hour of the agent going live:
```
The maximum limit of files monitored has been reached.
At this moment there are 100000 files and the limit is 100000.
```

**Diagnosis:** Default `file_limit` (100,000) assumes a static Linux server file
count. This node runs Docker (`overlay2` alone can be 50K–200K+ files per container
layer) plus game server data.

**Fix:** Raised `file_limit` in stages (100K → 500K → 1M) while simultaneously
adding `<ignore>` rules for Docker internals and non-security-relevant game data
(world saves, Steam caches). Raising the limit without adding exclusions just delays
the same problem — both were necessary. Full detail in
[02-wazuh-hardening.md](02-wazuh-hardening.md#the-core-problem-default-fim-file-limits-are-sized-for-the-wrong-workload).

---

## Issue: Docker Listener Crash-Looping Silently

**Symptom:** `docker-listener` module logs showed it starting, then exiting ~15
seconds later, repeatedly, with no error:
```
docker-listener: INFO: Module docker-listener started.
docker-listener: INFO: Module finished.
```
Docker *stats* (CPU/memory graphs via `logcollector` commands) were working fine,
which was misleading — it looked like Docker monitoring was functional when the
actual event stream (container create/start/stop, privileged launches) was dead.

**Diagnosis path (four layers, each masking the next):**
1. `wazuh` user wasn't in the `docker` group → fixed, no change (needed agent
   restart to take effect, group membership doesn't apply retroactively to a
   running service)
2. `pip3` wasn't installed on the box at all
3. After installing via `apt`, the module still wasn't importable as the `wazuh`
   user — turned out the box had **two Python interpreters** (`apt`'s system Python
   at `/usr/lib/python3.12`, and a from-source build at `/usr/local/bin/python3`
   that took `$PATH` precedence). The apt-installed `python3-docker` package went
   to the wrong interpreter entirely.
4. Docker socket permissions also needed correcting (`chmod 660`,
   `chown root:docker`)

**Fix:** Installed the `docker` Python module against the *actual* interpreter in
use (`/usr/local/bin/python3 -m pip install docker==7.1.0 ...`), confirmed with
`sudo -u wazuh /usr/local/bin/python3 -c "import docker"` before restarting the
agent.

**Lesson:** "Module not found" errors are only as useful as your certainty about
*which* interpreter is actually running. `which python3` and `sudo -u <service-user>
python3 -c "import sys; print(sys.executable)"` should be the first two commands
run, not the fifth.

---

## Issue: Command-Based Monitoring Producing No Data, No Errors

**Symptom:** FIM, Docker stats, and auth log monitoring all confirmed working, but
command-based checks (system load average, listening port count, zombie process
count) produced zero output — no alerts, no errors, nothing in the agent's own
operational log referencing the commands at all.

**Diagnosis:** Wazuh has an internal option, `logcollector.remote_commands`, that
must be explicitly set to `1` or every `<command>` block in `ossec.conf` is silently
ignored — not logged as disabled, just never executed.

**Fix:**
```bash
echo "logcollector.remote_commands=1" | sudo tee -a /var/ossec/etc/local_internal_options.conf
sudo systemctl restart wazuh-agent
```

**Lesson:** This was the highest-impact single-line fix of the whole project,
purely because of how silent the failure was. Worth checking this setting *first*
on any fresh Wazuh deployment that uses command-based log sources at all.

---

## Issue: Confusing Manager-Side Logs with Agent-Side Logs

**Symptom:** Repeatedly grepping for test alerts (`fim_test`, a fake SSH failure) in
`/var/ossec/logs/ossec.log` **on the agent** and finding nothing, leading to a false
diagnosis that FIM and auth monitoring were broken.

**Diagnosis:** `alerts.json` / `alerts.log` — the actual generated alerts — only
exist on the **manager**. The agent's `ossec.log` is its own operational log (did
syscheck start, is logcollector reading files) — it will show that an event was
*sent*, but not whether the manager turned it into an alert. Several rounds of
troubleshooting wasted time treating "no alert visible" as a single failure
condition when it could mean either "agent didn't send it" or "I'm checking the
wrong host."

**Fix:** No config fix needed — this was a workflow correction. Established a
standard two-sided check from then on: confirm the event left the agent
(`ossec.log` on the agent shows the relevant module activity), then separately
confirm the manager turned it into an alert (`alerts.json` on the manager).

**Lesson:** In a manager/agent split architecture, define *which host* a given log
file lives on before troubleshooting, every time — don't assume.

---

## Issue: Velociraptor Server Crash-Looping on Startup

**Symptom:** `systemctl status velociraptor_server` showed `activating
(auto-restart)` indefinitely, cycling every few seconds.

**Diagnosis path (two distinct errors, sequentially):**

1. First error, from `journalctl`:
   ```
   velociraptor: error: frontend: Unable to load config file: open
   /etc/velociraptor/server.config.yaml: permission denied
   ```
   The systemd unit runs the service as `User=velociraptor`, but the config file
   was `600 root:root` — unreadable by the service account. The parent directory
   was also `drwx--x--x`, which would have blocked the service from even *listing*
   the directory regardless of the file's own permissions.

2. After fixing ownership and permissions, a **second, different** error appeared:
   ```
   panic: Unable to create logging directory.
   ```
   The config's `Logging.output_directory` (`/var/lib/velociraptor/logs`) didn't
   exist and wasn't owned by the service account.

**Fix:** Corrected ownership at every layer the service touches — config file,
config directory, log directory, and datastore directory — all to
`velociraptor:velociraptor`.

**Lesson:** Don't treat the first error message as the only error. After fixing one
permission issue, immediately re-check `journalctl` rather than assuming a clean
start — there were two genuinely separate root causes here, encountered
sequentially.

---

## Issue: Velociraptor Client Refusing to Start — Missing CA Certificate

**Symptom:**
```
velociraptor_client: error: client: Unable to load config file:
No Client.ca_certificate configured
```

**Diagnosis:** The `client.config.yaml` produced via `velociraptor config client`
extraction (intended to pull just the client-relevant section out of the full
server config) was missing its CA certificate block. Root cause in the extraction
process itself wasn't fully isolated — diagnosed as a config generation issue rather
than a permissions or connectivity issue, since `nc`/`telnet` to the frontend port
both succeeded.

**Fix:** Used the complete `server.config.yaml` as the client config instead of the
extracted subset. The client binary only reads its own `Client{}` section from
whatever file it's given, so the full server config provides everything required
without depending on the extraction step working correctly. (Tradeoff: the full
server config contains more than the client strictly needs, so it's
access-restricted on the client side after transfer — not left world-readable.)

**Lesson:** When a generated "subset" config is suspected of being incomplete, the
fastest diagnostic step is trying the full source config rather than debugging the
extraction logic — confirms whether the problem is the extraction itself or
something else entirely.

---

## Issue: Stray Audit Rule Warnings for Paths That Don't Exist Yet

**Symptom:**
```
WARNING: (6925): Unable to add audit rule for '/var/lib/pterodactyl/.config'
WARNING: (6925): Unable to add audit rule for '/var/www/pterodactyl'
```

**Diagnosis:** These warnings looked like a Wazuh/auditd bug at first glance. Actual
cause: this is a *staging* VPS used specifically to validate the Wazuh config before
it touches production — Pterodactyl itself isn't installed there. The config being
tested was written for the production node layout, which does have those paths.

**Fix:** Confirmed with `ls -la <path>` that the paths genuinely didn't exist on
this host, then removed them from the staging-specific config rather than treating
it as a defect to chase.

**Lesson:** Not every warning is a bug in the tool — sometimes it's a config
correctly written for an environment it isn't currently running in. Worth
confirming the environment assumption before debugging the tool.
