# ServiceNowJDBC

Quick-reference commands for two unrelated-but-adjacent jobs:

1. **Querying the events database** (`ejaguiar1_events`) from a shell.
2. **Bringing a ServiceNow MID Server back up** — Linux (Docker/ARM64) first, Windows at the end.

Linux is the primary platform here. The full MID Server install/repair runbook lives in
[`eltonaguiar/ServiceNowMidServer`](https://github.com/eltonaguiar/ServiceNowMidServer).

## Contents

- [1. Events database (MySQL)](#1-events-database-mysql)
  - [Linux](#linux) · [Windows](#windows) · [Queries worth knowing](#queries-worth-knowing)
- [2. MID Server on Linux (Docker, ARM64 / NVIDIA GX10)](#2-mid-server-on-linux-docker-arm64--nvidia-gx10)
  - [Is it actually down?](#is-it-actually-down) · [The fix](#the-fix) · [Verify](#verify)
  - [If a restart isn't enough](#if-a-restart-isnt-enough) · [Why it keeps happening](#why-it-keeps-happening)
- [3. MID Server on Windows 11 (service)](#3-mid-server-on-windows-11-service)
- [4. Troubleshooting quick table](#4-troubleshooting-quick-table)

---

## 1. Events database (MySQL)

| | |
|---|---|
| Host | `mysql.50webs.com:3306` (from the web host itself: `localhost`) |
| Database | `ejaguiar1_events` |
| Main table | `events_log` (one row per event) |
| Bookkeeping | `event_pulls` (one row per ingest run), `stats_summary` (key/value counters), `event_sources`, `event_title_index` |

### Linux

The `mysql` client must be on `PATH`:

```bash
sudo apt-get update && sudo apt-get install -y default-mysql-client   # Debian/Ubuntu
mysql --version
```

Connect without putting the password on the command line (`-p` prompts; an inline
`-pPASSWORD` ends up in shell history and `ps`):

```bash
mysql -h mysql.50webs.com -u ejaguiar1_events -p ejaguiar1_events
```

One-shot query, no interactive session:

```bash
mysql -h mysql.50webs.com -u ejaguiar1_events -p -e \
  "SELECT COUNT(*) AS total FROM ejaguiar1_events.events_log;"
```

### Windows

`mysql.exe` is not on `PATH` by default on a stock install — call it by full path:

```bat
"C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe" ^
  -h mysql.50webs.com -u ejaguiar1_events -p ^
  -e "SELECT * FROM ejaguiar1_events.events_log LIMIT 5;"
```

Or add `C:\Program Files\MySQL\MySQL Server 8.0\bin` to `PATH` and use the same commands as Linux.

### Queries worth knowing

```sql
-- how many events, and are any duplicated?
SELECT COUNT(*) AS total, COUNT(DISTINCT event_id) AS unique_events FROM events_log;

-- is the data fresh?
SELECT MAX(last_updated) AS newest_row FROM events_log;
SELECT id, events_count, source, status FROM event_pulls ORDER BY id DESC LIMIT 5;

-- what's coming up
SELECT event_id, event_date, title, location, source
FROM events_log
WHERE event_date >= NOW()
ORDER BY event_date
LIMIT 10;

-- volume per month
SELECT DATE_FORMAT(event_date, '%Y-%m') AS month, COUNT(*) AS n
FROM events_log
WHERE event_date >= CURDATE()
GROUP BY month ORDER BY month;

-- where events come from
SELECT source, COUNT(*) AS n FROM events_log GROUP BY source ORDER BY n DESC LIMIT 10;

-- sanity check: no rows without a date
SELECT COUNT(*) FROM events_log WHERE event_date IS NULL;
```

Gotchas:

- The date column is **`event_date`**, not `date`. Querying `date` fails with `Unknown column`.
- `categories` and `tags` are JSON stored in `text`; filter with `JSON_CONTAINS` / `LIKE`, not `=`.
- `COUNT(*)` can differ from the number of unique events in `events.json` by a few rows —
  the published feed itself contains duplicate `id` values, and `event_id` is the dedupe key.

---

## 2. MID Server on Linux (Docker, ARM64 / NVIDIA GX10)

This host runs the MID as a container: `sn-mid` (`moers/mid-server:latest`), agent volume
`mid_data`, JDK volume `mid_jdk17`. Set these once per shell:

```bash
export SN_HOST="dev296154.service-now.com"     # no https://
export SN_USER="admin"                         # any user that can read ecc_agent
read -rsp "ServiceNow password: " SN_PASS; echo; export SN_PASS
```

### Is it actually down?

`docker ps` is not the answer — the container stays **Up** because PID 1 is only a `tail`,
while the MID JVM inside it can be dead.

```bash
docker ps --filter name=^sn-mid$ --format '{{.Names}} | {{.Status}} | {{.RunningFor}}'

# is the JVM alive? (the real test)
docker exec sn-mid pgrep -af service_now.mid.Main || echo "JVM DEAD — container is only a tail"

# is the log still advancing? frozen size = dead JVM
A=$(docker exec sn-mid stat -c %s /opt/agent/logs/agent0.log.0); sleep 15
B=$(docker exec sn-mid stat -c %s /opt/agent/logs/agent0.log.0)
echo "agent0.log.0 ${A} -> ${B}"

# what does the instance think?
curl -s -m 30 -u "$SN_USER:$SN_PASS" \
  "https://$SN_HOST/api/now/table/ecc_agent?sysparm_query=nameLIKEgx10&sysparm_fields=name,status,validated,last_refreshed,version"
```

`"status":"Up"` + `"validated":"true"` + a `last_refreshed` within minutes = healthy.
`"validated":"validating"` for the first minute after a start is normal.

### The fix

```bash
docker restart sn-mid
```

That is usually the whole fix. Do **not** click *Restart MID* in the ServiceNow UI on a
container that has no supervisor — see [Why it keeps happening](#why-it-keeps-happening).

### Verify

```bash
# log side: connected + operational
docker exec sn-mid sh -c 'grep -aE "connected to instance host|Operational State" /opt/agent/logs/agent0.log.0 | tail -3'

# instance side: poll until Up
for i in $(seq 1 10); do
  R=$(curl -s -m 30 -u "$SN_USER:$SN_PASS" \
      "https://$SN_HOST/api/now/table/ecc_agent?sysparm_query=name=gx10-mid&sysparm_fields=name,status,validated,last_refreshed")
  echo "[$i] $R"
  echo "$R" | grep -q '"status":"Up"' && break
  sleep 30
done
```

Expected: `connected to instance host: <host>` and `MID server Operational State=UP` in the log,
then `gx10-mid | status Up | validated true` from the API. Allow ~45 s to reconnect.

### If a restart isn't enough

```bash
# stale pid file
docker exec sn-mid sh -c 'rm -f /opt/agent/work/mid.pid' && docker restart sn-mid

# keystore mismatch: "Unable to load key store" / UnrecoverableKeyException
docker exec sn-mid sh -c 'rm -f /opt/agent/security/agent_keystore /opt/agent/security_backup/agent_keystore.*'
docker restart sn-mid

# volume ownership after a uid change
docker run --rm -v mid_data:/v --entrypoint /bin/bash moers/mid-server:latest -c 'chown -R 999:999 /v'
docker restart sn-mid

# host-level sanity
free -h; df -h /var/lib/docker
journalctl -k --since "1 hour ago" | grep -iE "oom|killed process"
```

For a `401 Unauthorized` / `User Not Authenticated`, the problem is the MID user, not the
container: check the user is **Active**, has the **`mid_server`** role, is **not locked out**,
has **"Password needs reset" unchecked**, and that the password change was **saved** on the
user form. Then re-apply the config by re-running `setup-mid-fast.sh` from the
[ServiceNowMidServer](https://github.com/eltonaguiar/ServiceNowMidServer) repo.

### Why it keeps happening

The MID runs **wrapper-less** (`com.service_now.mid.Main start` is exec'd directly) because
ServiceNow's service wrapper is x86-only and this host is ARM64. A wrapper-less MID cannot
honour a restart request; when the instance asks it to restart, it logs and exits:

```
[RestartHandler] Wrapper-less MID Server cannot be automatically restarted.
Shutting down. MANUALLY START THE MID SERVER TO CONTINUE
[MIDServerState] MID Server stopped
```

Nothing brings it back, and since PID 1 is `tail`, Docker's `--restart unless-stopped` never
fires either — container **Up**, instance **Down**.

Permanent fix: run `mid-supervisor.sh` inside the container so any JVM exit self-heals
(10 s relaunch, backing off to 60 s after three fast crashes). `setup-mid-fast.sh` already
wires it in; the container is created with:

```bash
docker run -d --name sn-mid --platform linux/arm64 --restart unless-stopped --user root \
  -v mid_data:/opt/agent -v mid_jdk17:/opt/jdk17 \
  -v "$PWD/mid-supervisor.sh":/opt/mid-supervisor.sh:ro \
  --entrypoint /bin/bash moers/mid-server:latest \
  -c "chmod +x /opt/mid-supervisor.sh 2>/dev/null; su mid -s /bin/bash -c 'exec /opt/mid-supervisor.sh' & exec tail -F /opt/agent/logs/agent0.log.0 /opt/agent/logs/console.log"
```

Confirm it took over: `docker exec sn-mid sh -c 'grep -a "\[supervisor\]" /opt/agent/logs/console.log | tail -5'`.

---

## 3. MID Server on Windows 11 (service)

The Windows MID installs as a service driven by the Tanuki wrapper:

```
Service name : snc_mid
Display name : ServiceNow MID Server
Path         : C:\ServiceNow\MID\agent\bin\wrapper-windows-x86-64.exe -s C:\ServiceNow\MID\agent\conf\wrapper.conf
Logon        : Local System account
```

Manage it from an elevated shell:

```bat
sc query snc_mid
net stop  snc_mid
net start snc_mid
```

Or **Services (`services.msc`)** → *ServiceNow MID Server* → Restart. The same wrapper command
can be used directly to run it in the foreground for debugging:

```bat
C:\ServiceNow\MID\agent\bin\wrapper-windows-x86-64.exe -s C:\ServiceNow\MID\agent\conf\wrapper.conf
```

Unlike the Linux container, the Windows MID *does* honour *Restart MID* — the wrapper restarts it.

---

## 4. Troubleshooting quick table

| Symptom | Likely cause | Do this |
|---|---|---|
| Container `Up`, instance `Down` | wrapper-less MID exited after a restart request | `docker restart sn-mid`; add `mid-supervisor.sh` |
| Log frozen, `docker ps` says Up | JVM dead, only `tail` alive | `docker restart sn-mid` |
| `Unable to load key store` / `UnrecoverableKeyException` | stale keystore vs new `config.xml` | delete `agent_keystore`, restart (see above) |
| `401 Unauthorized` in `agent0.log` | wrong/unsaved MID user password, missing `mid_server` role | fix user + Save, re-run setup |
| `Down` right after a start | still validating | wait ~1 min, re-poll `ecc_agent` |
| Two records (`gx10-mid`, `gx10-mid.server2`) | second MID from another container/volume | delete the stale record in the UI |
| `Unknown column 'date'` in SQL | real column is `event_date` | use `event_date` |

## Related

- [eltonaguiar/ServiceNowMidServer](https://github.com/eltonaguiar/ServiceNowMidServer) — ARM64 MID Server install scripts, `mid-supervisor.sh`, DISCOVERY/APPENDIX docs.
- `ecc_agent` is the instance-side source of truth for MID status (`name`, `status`, `validated`, `last_refreshed`).
