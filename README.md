# Components
Data source:
<img width="1665" height="548" alt="image" src="https://github.com/user-attachments/assets/94050da9-d339-4dfb-a6c0-862bb50f21c6" />

#System Import Sets>Administration>Data Sources
#Import set-->staging table 

#Query:SELECT event_id,title, event_date, location,last_updated
FROM events_log
WHERE location LIKE '%Toronto%'

#Transform Map

## Other Config
Last run datetime:'2026-11-01 12:09:41'    -->system populates after a load
#Last run database field  :last_updated

#Logs
under this page
https://dev296154.service-now.com/now/nav/ui/classic/params/target/sys_import_set.do%3Fsysparm_query%3Dnumber%3DISET0010009
Import Set Runs
<img width="1739" height="282" alt="image" src="https://github.com/user-attachments/assets/e0bb1454-2572-4edc-aadc-2ba059c2f79c" />
Import Set Rows
<img width="1649" height="491" alt="image" src="https://github.com/user-attachments/assets/c540fc10-feaf-4a1e-8f2c-f87adacf7bea" />



Note in the SQL, you can't have ORDER BY or ; , if you are using the Last run database field, so the system will populate  Last run datetime
"Don't add ORDER BY or a semicolon. On every run after the first, ServiceNow adds AND last_updated >= '...' ORDER BY last_updated ASC to the end of your query, so the query has to end with a condition it can add AND to. ServiceNow's own ORDER BY handles the sorting."

# Created the Table under System Definition > Tables
<img width="1804" height="798" alt="image" src="https://github.com/user-attachments/assets/95416749-3e53-4f91-9e80-312fed231753" />
Added Index under database indexes related list at the bottom  of https://dev296154.service-now.com/now/nav/ui/classic/params/target/sys_db_object.do%3Fsys_id%3Df2070865c3509750310abf0d05013136%26sysparm_record_target%3Dsys_db_object
<img width="1739" height="246" alt="image" src="https://github.com/user-attachments/assets/2e2f5165-87a6-4df2-92a9-6e97099f6bc8" />
Full page looked like this:
<img width="1307" height="855" alt="image" src="https://github.com/user-attachments/assets/719f5a5f-8683-437c-9ef0-2533d9df8def" />


#Monitoring Run times > Load Run Time/ Load COmpleted under Import Sets https://dev296154.service-now.com/now/nav/ui/classic/params/target/sys_import_set_list.do%3Fsysparm_query%3Ddata_source.import_set_table_name%3Du_torontoevents2%5EORDERBYDESCsys_created_on
<img width="1865" height="495" alt="image" src="https://github.com/user-attachments/assets/e997055c-a479-4181-b213-988e39e703fb" />


#Subsequent Import ignores 598 records.. 
<img width="1895" height="146" alt="image" src="https://github.com/user-attachments/assets/d73b3339-1f12-416a-8675-23113c5897e5" />
#why ignored, Checking whether 598 matches a recent `last_updated` window in the database.

Ran terminal command: $env:MYSQL_PWD='events1234560'; & 'C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe' -h mysql.50webs.com -u ejaguiar1_events --connect-timeout=20 ejaguiar1_events -e "SELECT COUNT(*) total, MAX(last_updated) max_lu FROM events_log WHERE location LIKE '%Toronto%'; SELECT last_updated, COUNT(*) n FROM events_log WHERE location LIKE '%Toronto%' GROUP BY last_updated ORDER BY last_updated DESC LIMIT 8; SELECT pull_timestamp, events_count, source, notes FROM event_pulls ORDER BY id DESC LIMIT 5;"; Remove-Item Env:\MYSQL_PWD

#Scheduled Data Imports
https://dev296154.service-now.com/now/event_jobs/record/sys_trigger/27dfc069c3d09750310abf0d050131e2
<img width="1774" height="184" alt="image" src="https://github.com/user-attachments/assets/05ec935d-64b2-4715-8b2b-4045a3bfc17a" />
e.g. daily at midnight
<img width="1687" height="597" alt="image" src="https://github.com/user-attachments/assets/325cc74c-7853-4e55-b853-770e08699202" />




**Task completed:**

Not quite. The 598 rows did match your query, and **Ignored** here is good news: they were already in your target table.

**Why only 598 were attempted:** after the 8,235-row load, ServiceNow saved the newest `last_updated` value, `2026-11-01 12:09:41`, as the last run datetime. The next load used `last_updated >= '2026-11-01 12:09:41'`. In the database, exactly **598** Toronto rows have that timestamp, so it fetched only those. That import set, `ISET0010009`, has 598 rows, and its transform handled only those.

**Why all 598 were ignored:** the transform matched each row on `event_id` (your coalesce field), found the existing record, and saw that no values had changed, so it skipped the update. That means coalesce is working and no duplicates were created.

**This will repeat:** with `>=`, every later load re-fetches the rows stamped exactly at the saved time. They'll keep showing as ignored, which is harmless.

**To confirm the full load:** in **Transform History**, find the run for the earlier 8,235-row import set (it's a different ISET number). It should show about 8,235 inserts. You can also check that your target table has 8,235 records.

# ServiceNowJDBC

Quick-reference commands for two unrelated-but-adjacent jobs:

1. **Querying the events database** (`ejaguiar1_events`) from a shell.
2. **Bringing a ServiceNow MID Server back up** — Linux (Docker/ARM64) first, Windows at the end.

Linux is the primary platform here. The full MID Server install/repair runbook lives in
[`eltonaguiar/ServiceNowMidServer`](https://github.com/eltonaguiar/ServiceNowMidServer).

## Contents

- [1. Events database (MySQL)](#1-events-database-mysql)
  - [How the import stays idempotent (and only takes newer rows)](#how-the-import-stays-idempotent-and-only-takes-newer-rows)
  - [Linux](#linux) · [Windows](#windows) · [Queries worth knowing](#queries-worth-knowing)
- [2. MID Server on Linux (Docker, ARM64 / NVIDIA GX10)](#2-mid-server-on-linux-docker-arm64--nvidia-gx10)
  - [Is it actually down?](#is-it-actually-down) · [The fix](#the-fix) · [Verify](#verify)
  - [If a restart isn't enough](#if-a-restart-isnt-enough) · [Why it keeps happening](#why-it-keeps-happening)
- [3. MID Server on Windows 11 (service)](#3-mid-server-on-windows-11-service)
- [4. Where the "service" and its configuration live — a guide for Linux newcomers](#4-where-the-service-and-its-configuration-live--a-guide-for-linux-newcomers)
  - [The three layers](#the-three-layers-top-to-bottom) · [See the full configuration](#see-the-full-configuration-the-servicesmsc-path-detail)
  - [GUI route](#gui-route-optional-for-people-who-prefer-a-file-manager) · [Changing it](#changing-the-configuration)
- [5. Troubleshooting quick table](#5-troubleshooting-quick-table)

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

### How the import stays idempotent (and only takes newer rows)

The pipeline is MySQL → MID → import set → transform map → table, and it never re-imports the whole table:

| Piece | Where it lives | Value in this instance |
|---|---|---|
| Data source | `sys_data_source` | `torontoeventsv2`, type **JDBC**, driver `org.mariadb.jdbc.Driver`, MID `win11-mid` |
| SQL | `sql_statement` (Query = *Specific SQL*) | `SELECT event_id,title, event_date, location,last_updated FROM events_log WHERE location LIKE '%Toronto%'` |
| **Helper field** | `last_run_database_field` | **`last_updated`** |
| **The one parameter** | `last_run_datetime` with `use_last_run_datetime = true` | `'2026-11-01 12:09:41'` (a single quoted datetime) |
| Staging | `import_set_table_name` | `u_torontoevents2` |
| Transform | `sys_transform_map` | `TorontoEventsTransformMap` — `u_torontoevents2` → `u_torontoeventsfinal` |
| **Coalesce key** | `sys_transform_entry` | `u_event_id`, `coalesce = true` |

Two mechanisms do the work, and they solve different problems:

1. **Only newer rows.** With *Use last run datetime* on and the helper field set to `last_updated`,
   ServiceNow appends `AND last_updated >= '<last_run_datetime>'` to the SQL. That one parameter is the
   watermark — the timestamp of the last successful run — so a daily run pulls edits, not 12 K rows.
2. **Idempotency.** The transform map **coalesces on `u_event_id`**, so a row delivered twice (overlapping
   window, retried run, manual *Execute Now*) **updates** the existing record instead of creating a second one.
   `COUNT(*)` and `COUNT(DISTINCT u_event_id)` stay equal however often the import runs.

Gotchas:

- Clearing `last_run_datetime` forces a full re-read — the recovery move after a broken window, but it
  re-delivers everything, which is only safe because of the coalesce key.
- The watermark compares against `last_updated` **as MySQL stores it**: keep that column `datetime` and
  timezone-stable, or boundary rows repeat or get skipped.
- `u_last_updated` is a **string** (max 40) on `u_torontoeventsfinal`, not a date — it carries the MySQL text
  through unchanged, so do date maths in the transform script, not on the column.

---

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

## 4. Where the "service" and its configuration live — a guide for Linux newcomers

On Windows 11 you open **`services.msc`** and get one tidy row: *ServiceNow MID Server*,
service name `snc_mid`, **path to executable**, **log on as**, startup type. Ubuntu has no
`services.msc`, and the MID here is not a Windows-style service at all — it is a **container**.
Same information, different layers:

| Windows `services.msc` shows | Linux / Docker equivalent | Command |
|---|---|---|
| The service list | running containers + systemd services | `docker ps -a` / `systemctl list-units --type=service --state=running` |
| Service name `snc_mid` | container **`sn-mid`**, kept alive by systemd unit `docker.service` | `docker ps -a --filter name=sn-mid` |
| Path to executable | the container's `Path` + `Args` (`/bin/bash -c "…supervisor… tail"`) | `docker inspect sn-mid --format '{{.Path}} {{.Args}}'` |
| Log On As: Local System | `Config.User=root`; the JVM inside drops to user **`mid`** (uid 999) | `docker inspect sn-mid --format '{{.Config.User}}'` |
| Startup type: Automatic | `RestartPolicy=unless-stopped` (plus `docker.service` enabled) | `docker inspect sn-mid --format '{{.HostConfig.RestartPolicy.Name}}'` |
| Start / Stop / Restart buttons | `docker start` / `stop` / `restart sn-mid` | same |
| Program parameters / config | `/opt/agent/config.xml` inside the container | `docker exec sn-mid cat /opt/agent/config.xml` |

### The three layers, top to bottom

```text
systemd  ── docker.service            <- "the service" from Ubuntu's point of view
             └── container sn-mid     <- restart: unless-stopped
                   └── /opt/mid-supervisor.sh
                         └── java com.service_now.mid.Main start   <- the actual MID
```

1. **systemd** keeps the Docker engine running — `systemctl status docker`.
2. **Docker** keeps the container alive — `docker ps -a`.
3. **the supervisor + java** inside the container run the MID. This is the layer that dies
   silently while the first two still look healthy (see [Why it keeps happening](#why-it-keeps-happening)).

### See the full configuration (the services.msc "Path" detail)

```bash
# everything at once: how it starts, as which user, which volumes, restart policy
docker inspect sn-mid | less

# the short version
docker inspect sn-mid --format 'Path:    {{.Path}}
Args:    {{.Args}}
Runs as: {{.Config.User}}
Restart: {{.HostConfig.RestartPolicy.Name}}
Image:   {{.Config.Image}}
Started: {{.State.StartedAt}}
Status:  {{.State.Status}}'

# volumes — where the data really lives, plus the supervisor bind-mount
docker inspect sn-mid --format '{{range .Mounts}}{{.Type}} {{.Name}} -> {{.Destination}}{{"\n"}}{{end}}'

# environment the container was launched with (JVM memory etc.)
docker inspect sn-mid --format '{{range .Config.Env}}{{println .}}{{end}}' | grep -E 'MID_XMX|OPENS'
```

The MID's own settings file is `/opt/agent/config.xml` — instance URL, MID user, MID name,
threads, proxy:

```bash
# the whole file
docker exec sn-mid cat /opt/agent/config.xml

# just the parts that matter
docker exec sn-mid grep -E 'name="(url|mid.instance.username|name)"' /opt/agent/config.xml
```

That file lives on the `mid_data` volume, which is an ordinary directory on the host:

```bash
# host path: /var/lib/docker/volumes/mid_data/_data
sudo ls -l /var/lib/docker/volumes/mid_data/_data/config.xml
sudo ls -l /var/lib/docker/volumes/mid_data/_data/logs/     # agent0.log.0 lives here
```

(`sudo` asking for your password is normal — you are reading files owned by root.)

### GUI route (optional, for people who prefer a file manager)

Ubuntu has no built-in `services.msc`. What you can still do:

- **Terminal** — `Ctrl`+`Alt`+`T`, or Activities → type “Terminal”. This is the real tool;
  everything below is convenience only.
- **Processes** — open “System Monitor” (`gnome-system-monitor`). You will see `java`,
  `dockerd`, `containerd`. Right-click → *End* is the GUI version of “Stop”.
- **Docker Desktop for Linux** *(if you install it)* — shows the container, its start command,
  exit state and live logs in a window. Closest visual match to `services.msc`.
- **Files (Nautilus)** — browse the volume as root:
  `sudo -E nautilus /var/lib/docker/volumes/mid_data/_data &` — handy for reading
  `config.xml` and `logs/` in a text editor.
- **Settings** (`gnome-control-center`) manages *nothing* here — no services, no containers.
- Optional GUI service managers, not preinstalled: `sudo apt install stacer` (or
  `systemd-manager`). These list **systemd** units, so they show `docker.service` but **not**
  the MID container.

### Changing the configuration

Don't hand-edit credentials into `config.xml` — the setup script regenerates that file, and a
half-edited one causes the keystore / 401 failures described in section 2. Supported route:

```bash
export SN_HOST="dev296154.service-now.com" MID_USER="mid.server" MID_NAME="gx10-mid"
read -rsp "MID user password: " MID_PASS; export MID_PASS
bash ./setup-mid-fast.sh     # regenerates config.xml, clears keystore, relaunches, polls
```

For something the script doesn't cover (thread count, proxy), copy the file out, edit, put it
back, restart:

```bash
docker cp sn-mid:/opt/agent/config.xml ./config.xml
nano ./config.xml                                        # or any editor you like
docker cp ./config.xml sn-mid:/opt/agent/config.xml
docker restart sn-mid
```

### Open a shell inside the container (the closest thing to RDP)

```bash
docker exec -it sn-mid bash      # or: sh
pwd                              # /opt/agent
ls                               # config.xml  logs/  lib/  extlib/  work/  security/
```

## 5. Troubleshooting quick table

| Symptom | Likely cause | Do this |
|---|---|---|
| Container `Up`, instance `Down` | wrapper-less MID exited after a restart request | `docker restart sn-mid`; add `mid-supervisor.sh` |
| Log frozen, `docker ps` says Up | JVM dead, only `tail` alive | `docker restart sn-mid` |
| `Unable to load key store` / `UnrecoverableKeyException` | stale keystore vs new `config.xml` | delete `agent_keystore`, restart (see above) |
| `401 Unauthorized` in `agent0.log` | wrong/unsaved MID user password, missing `mid_server` role | fix user + Save, re-run setup |
| `Down` right after a start | still validating | wait ~1 min, re-poll `ecc_agent` |
| Two records (`gx10-mid`, `gx10-mid.server2`) | second MID from another container/volume | delete the stale record in the UI |
| Can't find the service / config on Linux | the MID is a container, not a systemd service | see section 4: `docker inspect sn-mid`, `/opt/agent/config.xml` |
| `Unknown column 'date'` in SQL | real column is `event_date` | use `event_date` |

## Related

- [eltonaguiar/ServiceNowMidServer](https://github.com/eltonaguiar/ServiceNowMidServer) — ARM64 MID Server install scripts, `mid-supervisor.sh`, DISCOVERY/APPENDIX docs.
- `ecc_agent` is the instance-side source of truth for MID status (`name`, `status`, `validated`, `last_refreshed`).
