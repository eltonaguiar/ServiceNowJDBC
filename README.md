# ServiceNowJDBC

How the Toronto events data gets **from MySQL into ServiceNow over JDBC**, the commands for **querying that
database**, and the runbook for **keeping the MID Server that does the work alive**. Everything here was run against
a live instance — the screenshots are the real configuration, not a diagram of it.

Linux is the primary platform. The full MID Server install/repair runbook lives in
[`eltonaguiar/ServiceNowMidServer`](https://github.com/eltonaguiar/ServiceNowMidServer).

## Before you start — prerequisites checklist

Tick these off before step 1. Everything here is a hard prerequisite unless marked optional — the common
failures are a missing item from this list, not a step done wrong.

- [ ] MySQL credentials for the **source**, plus a **grant for the MID's IP** (`'user'@'%'` or the MID subnet). A user
      that works from `localhost` is still denied when the MID connects
- [ ] Network path: the MID host can reach `mysql.50webs.com:3306` (`nc -vz mysql.50webs.com 3306` from the MID box)
- [ ] A **MID Server** showing *Validated: Yes* (`win11-mid` here) — the SQL runs from *its* network, not the instance's
- [ ] Roles `import_admin` / `admin` on the instance
- [ ] The SQL ready, **including `last_updated`** in the `SELECT`, and with **no `ORDER BY` and no trailing `;`**
- [ ] Target table name agreed (`u_torontoeventsfinal`) and its column types decided before you build the transform map
- [ ] `mysql` client installed locally if you want to verify row counts from the source side

---

## Contents

- [A. Build order — how the JDBC import was wired](#a-build-order--how-the-jdbc-import-was-wired) — the records to
  create, in order, with the UI path for each
- [B. The same build, in screenshots](#b-the-same-build-in-screenshots) — the same steps as pictures, captioned, in
  the same order
- [1. Events database (MySQL)](#1-events-database-mysql)
  - [Connecting — the two rules that trip everyone up](#connecting--the-two-rules-that-trip-everyone-up)
  - [Queries worth knowing](#queries-worth-knowing) ·
    [How the import stays idempotent](#how-the-import-stays-idempotent-and-only-takes-newer-rows)
- [2. MID Server on Linux (Docker, ARM64 / NVIDIA GX10)](#2-mid-server-on-linux-docker-arm64--nvidia-gx10)
  - [Is it actually down?](#is-it-actually-down) · [The fix](#the-fix) · [Verify](#verify)
  - [If a restart isn't enough](#if-a-restart-isnt-enough) · [Why it keeps happening](#why-it-keeps-happening)
- [3. MID Server on Windows 11 (service)](#3-mid-server-on-windows-11-service)
- [4. Where the "service" is — and how autostart actually works](#part-c-where-the-service-is)
  - [The four layers](#eli5--on-windows-there-is-one-row-here-there-are-four-layers) ·
    [Auto-start](#auto-start-how-windows-does-it-vs-how-linux-does-it) ·
    [`services.msc` field by field](#servicesmsc-field-by-field--linux-equivalent)
  - [See the full configuration](#see-the-full-configuration-the-servicesmsc-path-detail) ·
    [GUI route](#gui-route-optional-for-people-who-prefer-a-file-manager) ·
    [Changing it](#changing-the-configuration--safely)
- [5. Troubleshooting quick table](#5-troubleshooting-quick-table)

---

## A. Build order — how the JDBC import was wired

The import is a chain, and every link is a record: **data source → import set table → transform map → target table →
scheduled import**. Build in that order; nothing below depends on a later step. Each step gives the page to stand on,
what to have ready first, the fields to fill, and the one suggestion that saves rework.

### Step 1 — Data Source
> **Breadcrumb:** `All → System Import Sets → Data Sources → New`

- **Prerequisite:** a **Validated** MID Server that can reach the database host on `3306`, and a DB user granted for
  *the MID's* IP (`'user'@'%'` or the MID subnet) — the SQL runs from the MID's network, not the instance's.
- **Fields to fill:** `Name` = `torontoeventsv2` · `Type` = JDBC · `JDBC Driver Class` =
  `org.mariadb.jdbc.Driver` · `Connection URL` = `jdbc:mysql://mysql.50webs.com:3306/ejaguiar1_events` ·
  `Database user` / `Database password` · `MID Server` = `win11-mid` · `Query type` = *Specific SQL* · `SQL
  statement` = the query in [section B step 1](#step-1--the-data-source).
- **Suggestion:** use **Test Load 20 Records** here, not the full query — it proves driver, URL, network and grants
  before you move 8,000 rows.
- **Checkpoint:** the test load returns columns.

### Step 2 — the watermark, on the same form
> **Breadcrumb:** same Data Source form, *Data Load Type* / advanced fields

- **Prerequisite:** the SQL **selects `last_updated`**; without that column in the result there is nothing to compare.
- **Fields to fill:** `Use last run datetime` = `true` · `Last run database field` = `last_updated` ·
  `Last run datetime` = **leave empty** (the platform fills it after the first successful run).
- **Suggestion:** never add `ORDER BY` or a trailing `;` to the SQL, and never filter on `last_updated` yourself —
  ServiceNow appends `AND last_updated >= '…' ORDER BY last_updated ASC` and the two clauses fight.
- **Checkpoint:** the preview shows the watermark clause appended.

### Step 3 — the staging table, on the same form
> **Breadcrumb:** same Data Source form → `Import set table name`

- **Prerequisite:** none.
- **Fields to fill:** `Import set table name` = `u_torontoevents2` (a name you can recognise later in Import Sets).
- **Suggestion:** let ServiceNow create it; give it a `u_` prefix so it is obviously custom, and do not clean it
  between runs — it is the audit trail of what actually arrived.
- **Checkpoint:** the staging table exists after the test load.

### Step 4 — Transform Map
> **Breadcrumb:** `All → System Import Sets → Transform Maps → New`

- **Prerequisite:** staged rows to transform, and the **target table already created** (step 5 — do it first if it
  does not exist yet).
- **Fields to fill:** `Name` = `TorontoEventsTransformMap` · `Source table` = `u_torontoevents2` ·
  `Target table` = `u_torontoeventsfinal` · `Run business rules` = **off** · mappings via **Auto Map Matching
  Fields** (the *Related Link*) · per-entry `Date format` = `yyyy-MM-dd HH:mm:ss` · **`Coalesce` = true on
  `u_event_id`**.
- **Suggestion:** get the coalesce key right **before** the first big load. Retrofitting it means deleting
  duplicates by hand; adding it later cannot undo what is already there.
- **Checkpoint:** **Run Transform** on the test rows inserts, and running it again updates instead of duplicating.

### Step 5 — Target table
> **Breadcrumb:** `All → System Definition → Tables` → New

- **Prerequisite:** none — but create it **before** step 4 so the transform has somewhere to map to.
- **Fields to fill:** `Label` = Toronto Events · `Name` = `u_torontoeventsfinal` · `Extends` = (none) ·
  `Columns`: `u_event_id` string(64), `u_title` string(500), `u_location` string(500), `u_event_date` date/time,
  `u_last_updated` **string(40)**.
- **Suggestion:** add a database index on `u_event_id` from the *Database Indexes* related list — it is the column
  every coalesce lookup hits. Keep `u_last_updated` a string; it carries MySQL's text verbatim, so do date maths in a
  transform script.
- **Checkpoint:** all five columns show in the dictionary.

### Step 6 — Scheduled Import
> **Breadcrumb:** Data Source form → **Scheduled Import**

- **Prerequisite:** one successful manual load (a `TH…` import set in `complete`), so the watermark has a value.
- **Fields to fill:** `Run` = *Daily* · `Time` = the hour your source data has settled · `Time zone` = the zone you
  actually mean (the trigger stores what you pick).
- **Suggestion:** schedule it *after* the upstream MySQL load, and leave an hour of slack — an import that starts
  while the scraper is still writing will simply miss the tail of the batch until tomorrow.
- **Checkpoint:** one `sys_import_set_run` row per scheduled execution.

### Step 7 — Import Sets (verification)
> **Breadcrumb:** `All → System Import Sets → Import Sets`

- **Prerequisite:** at least one run.
- **Fields to fill:** none — read-only.
- **Suggestion:** the funnel is `Load Run Time` (fetch) → `Import Set Rows` (arrived) → transform result
  (inserted / updated / ignored). Read all three; "Ignored" is usually success, not failure (see
  [section B step 5](#step-5--the-delta-behaviour-on-the-second-load)).
- **Checkpoint:** `state = complete`, counts sane.

Roles: `import_admin`/`admin` to build. The MID must reach the database host on **3306** — the SQL runs from the MID's
network, not the instance's.

**Parameters to consider**

| Parameter | Where | Why it matters |
|---|---|---|
| Query type (*All rows* vs *Specific SQL*) | data source | *Specific SQL* is the only mode where the watermark append is predictable |
| JDBC driver + URL | data source | `org.mariadb.jdbc.Driver` for MySQL 8 / MariaDB; a wrong driver gives "no suitable driver" |
| MID Server | data source | chooses the network the query runs from |
| *Use last run datetime* + *Last run database field* | data source | incremental sync with no code |
| *Import set table* | data source | staging; big loads fill it until housekeeping runs |
| Coalesce field | transform entry | the idempotency key — wrong here and every run duplicates |
| Date format | transform entry | `yyyy-MM-dd HH:mm:ss`; a mismatch silently nulls dates |
| Run business rules | transform map | expensive at 10 K rows — off unless you need them |
| Transform order | transform map | only matters with multiple maps on one staging table |

**Things to watch out for**

- **The SQL must return the watermark column.** Drop `last_updated` from the `SELECT` and the delta has nothing to
  compare — you get full loads that look like deltas.
- **Never filter on the watermark yourself** (no `WHERE last_updated > …` inside `sql_statement`); ServiceNow appends
  its own condition and the two fight.
- **Remote MySQL grants.** The MID connects from its own host, so the DB user needs a grant for that IP
  (`'user'@'%'` or the MID subnet). `Access denied` here is usually a grant problem, not a password problem.
- **`u_last_updated` is a string.** It carries MySQL's text verbatim — compare/parse in a transform script, not on the
  column.
- **The first run after enabling the watermark is a full load** (`last_run_datetime` starts empty). That is correct,
  and it is why the coalesce key must exist *before* the first run.
- **Clearing the watermark** re-delivers everything: harmless with coalesce, catastrophic without it.

**Validation — four checks, with numbers from this instance**

```bash
# 1 did it run, and how long did it take?
curl -s -u "$SN_USER:$SN_PASS" "https://<instance>/api/now/table/sys_import_set_run?sysparm_limit=5&sysparm_fields=number,state,sys_created_on,completed"
#   TH0001006 | complete | 2026-11-01 12:39:29 | 12:39:31   (~2 s)

# 2 duplicates in the target? total vs grouped count must be equal
curl -s -u "$SN_USER:$SN_PASS" "https://<instance>/api/now/stats/u_torontoeventsfinal?sysparm_count=true&sysparm_group_by=u_event_id&sysparm_max_rows=50000"
#   number of groups == total rows  => coalesce is working

# 3 is the watermark advancing?
curl -s -u "$SN_USER:$SN_PASS" "https://<instance>/api/now/table/sys_data_source?sysparm_query=name=torontoeventsv2&sysparm_fields=name,last_run_datetime,use_last_run_datetime,last_run_database_field"

# 4 does the source agree with what landed?
mysql -h mysql.50webs.com -u ejaguiar1_events -p -e \
  "SELECT COUNT(*) AS src_rows, MAX(last_updated) AS src_max FROM ejaguiar1_events.events_log WHERE location LIKE '%Toronto%';"
```

---

## B. The same build, in screenshots

Walk these in order and you end up with the same pipeline as [section A](#a-build-order--how-the-jdbc-import-was-wired).
Every picture carries a **step label** (`1A`, `2A`, …) and the **breadcrumb** you need to be standing on to see it:

| Label | Step | Screenshots |
|---|---|---|
| **1** | the data source | `1A` |
| **2** | the transform map | `2A` |
| **3** | the target table | `3A` `3B` `3C` |
| **4** | the first load, and where to watch it | `4A` `4B` `4C` |
| **5** | the delta behaviour on later loads | `5A` |
| **6** | the schedule | `6A` `6B` |

The letters are ordering within a step, so `3A → 3B → 3C` is always the intended reading order.

### Step 1 — the data source

**Screenshot 1A · Data source form** — type, driver, MID, query mode, staging table

> `All → System Import Sets → Administration → Data Sources` → `torontoeventsv2`

<img width="1665" height="548" alt="Screenshot 1A - data source torontoeventsv2: type JDBC, driver org.mariadb.jdbc.Driver, MID win11-mid, Query = Specific SQL" src="https://github.com/user-attachments/assets/94050da9-d339-4dfb-a6c0-862bb50f21c6" />

*Shows:* **type JDBC**, `org.mariadb.jdbc.Driver`, the **MID Server `win11-mid`** (this is the field that decides
which network the SQL runs from), **Query = Specific SQL**, and the staging table `u_torontoevents2` that rows land
in before any transform.

The query — explicit column list, a `LIKE` filter, and **`last_updated` included**, because the delta has nothing to
compare without it:

```sql
SELECT event_id, title, event_date, location, last_updated
FROM events_log
WHERE location LIKE '%Toronto%'
```

> ⚠️ **No `ORDER BY`, no trailing semicolon.** With *Last run database field* set, ServiceNow appends
> `AND last_updated >= '…' ORDER BY last_updated ASC` on every run after the first — so the statement must end with a
> condition it can bolt `AND` onto, and ServiceNow owns the sorting.

### Step 2 — the transform map

**Screenshot 2A · Transform map** — field mapping and the coalesce key

> `All → System Import Sets → Transform Maps` → `TorontoEventsTransformMap`

<img width="1479" height="743" alt="Screenshot 2A - transform map TorontoEventsTransformMap mapping staging fields to u_torontoeventsfinal" src="https://github.com/user-attachments/assets/6889c5ef-c286-49aa-b231-f167f4f6b97e" />

*Shows:* the mapping from `u_torontoevents2` to `u_torontoeventsfinal`, built with **Auto Map Matching Fields** (the
*Related Link*), and the one setting that does all the work — **Coalesce = true on `u_event_id`**. That is what turns
a re-delivered row into an update instead of a duplicate.

**The two delta settings live back on the 1A form:**

| Field | Value | Who sets it |
|---|---|---|
| *Use last run datetime* | on | you |
| *Last run database field* | `last_updated` | you |
| *Last run datetime* | `2026-11-01 12:09:41` | **the system, after each successful run — never type it** |

### Step 3 — the target table

**Screenshot 3A · Table record** — columns, including the string `u_last_updated`

> `All → System Definition → Tables` → `u_torontoeventsfinal`

<img width="1804" height="798" alt="Screenshot 3A - u_torontoeventsfinal table record and its columns" src="https://github.com/user-attachments/assets/95416749-3e53-4f91-9e80-312fed231753" />

*Shows:* `u_event_id`, `u_title`, `u_location`, `u_event_date`, and **`u_last_updated` as string(40)** — MySQL text
carried through untouched, so do date maths in the transform, not on the column.

**Screenshot 3B · Database Indexes related list** — the index on the coalesce key

> Same record, scrolled to **Database Indexes** at the bottom
> ([open it](https://dev296154.service-now.com/now/nav/ui/classic/params/target/sys_db_object.do%3Fsys_id%3Df2070865c3509750310abf0d05013136%26sysparm_record_target%3Dsys_db_object))

<img width="1739" height="246" alt="Screenshot 3B - Database Indexes related list on the table record" src="https://github.com/user-attachments/assets/2e2f5165-87a6-4df2-92a9-6e97099f6bc8" />

**Screenshot 3C · the full table form** — for side-by-side comparison with yours

<img width="1307" height="855" alt="Screenshot 3C - full u_torontoeventsfinal table form" src="https://github.com/user-attachments/assets/719f5a5f-8683-437c-9ef0-2533d9df8def" />

### Step 4 — the first load, and where to watch it

**Screenshot 4A · Import set list** — `Load Run Time` and `Load Completed`

> `All → Import Sets`, filtered to the staging table
> ([this instance](https://dev296154.service-now.com/now/nav/ui/classic/params/target/sys_import_set_list.do%3Fsysparm_query%3Ddata_source.import_set_table_name%3Du_torontoevents2%5EORDERBYDESCsys_created_on), newest first)

<img width="1865" height="495" alt="Screenshot 4A - import sets for u_torontoevents2 with load run time and load completed" src="https://github.com/user-attachments/assets/e997055c-a479-4181-b213-988e39e703fb" />

*Shows:* one row per load. `Load Run Time` is the JDBC fetch, `Load Completed` is the transform.

**Screenshot 4B · Import Set Runs tab** — one row per execution, with counts and state

> `All → Import Sets` → the import set → **Import Set Runs**
> ([`ISET0010009`](https://dev296154.service-now.com/now/nav/ui/classic/params/target/sys_import_set.do%3Fsysparm_query%3Dnumber%3DISET0010009))

<img width="1739" height="282" alt="Screenshot 4B - Import Set Runs tab for ISET0010009" src="https://github.com/user-attachments/assets/e0bb1454-2572-4edc-aadc-2ba059c2f79c" />

**Screenshot 4C · Import Set Rows tab** — what arrived, before the transform

> Same import set → **Import Set Rows**

<img width="1649" height="491" alt="Screenshot 4C - Import Set Rows tab for ISET0010009" src="https://github.com/user-attachments/assets/c540fc10-feaf-4a1e-8f2c-f87adacf7bea" />

### Step 5 — the delta behaviour on the *second* load

**Screenshot 5A · 598 rows, all Ignored** — looks like a failure, isn't one

> Same list as `4A`, next run

<img width="1895" height="146" alt="Screenshot 5A - import set showing 598 rows, all ignored" src="https://github.com/user-attachments/assets/d73b3339-1f12-416a-8675-23113c5897e5" />

*Why only 598:* the previous 8,235-row load saved `2026-11-01 12:09:41` as *Last run datetime*, so the next query
asked for `last_updated >= '2026-11-01 12:09:41'` — and exactly 598 Toronto rows carry that timestamp.

*Why all 598 were Ignored:* the transform matched each row on `event_id` (the coalesce key), found the existing
record, saw no changed values, and skipped the update. No duplicates, no writes.

*It will repeat:* with `>=`, every later load re-fetches rows stamped exactly at the saved time and ignores them
again. Harmless.

How it was verified — the grouped query is the answer, because it shows those 598 rows sharing one `last_updated`
(the watermark the import picked up). Password kept out of the command line:

```powershell
$env:MYSQL_PWD = '<password>'
& 'C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe' -h mysql.50webs.com -u ejaguiar1_events `
  --connect-timeout=20 ejaguiar1_events -e "
    SELECT COUNT(*) total, MAX(last_updated) max_lu FROM events_log WHERE location LIKE '%Toronto%';
    SELECT last_updated, COUNT(*) n FROM events_log WHERE location LIKE '%Toronto%'
      GROUP BY last_updated ORDER BY last_updated DESC LIMIT 8;
    SELECT pull_timestamp, events_count, source, notes FROM event_pulls ORDER BY id DESC LIMIT 5;"
Remove-Item Env:\MYSQL_PWD
```

To confirm the *full* load instead, open **Transform History** for the earlier 8,235-row import set (a different
`ISET` number) — it should show ~8,235 inserts.

### Step 6 — schedule it

**Screenshot 6A · Scheduled Import on the data source**

> `All → System Import Sets → Data Sources` → `torontoeventsv2` → **Scheduled Import** (creates a `sys_trigger`)

<img width="1774" height="184" alt="Screenshot 6A - Scheduled Import link on the data source form" src="https://github.com/user-attachments/assets/05ec935d-64b2-4715-8b2b-4045a3bfc17a" />

**Screenshot 6B · the trigger record** — daily at midnight, run through the MID

> [`sys_trigger` record](https://dev296154.service-now.com/now/event_jobs/record/sys_trigger/27dfc069c3d09750310abf0d050131e2)

<img width="1687" height="597" alt="Screenshot 6B - scheduled import trigger, daily at midnight" src="https://github.com/user-attachments/assets/325cc74c-7853-4e55-b853-770e08699202" />

---

## 1. Events database (MySQL)

| | |
|---|---|
| Host | `mysql.50webs.com:3306` (from the web host itself: `localhost`) |
| Database | `ejaguiar1_events` |
| Main table | `events_log` (one row per event) |
| Bookkeeping | `event_pulls` (one row per ingest run), `stats_summary` (key/value counters), `event_sources`, `event_title_index` |

### Connecting — the two rules that trip everyone up

**Rule 1 — no space after `-p`.** `-p` on its own means *"prompt me for the password"*. `-pSECRET` means *"use
SECRET"*. `-p SECRET` (with a space) means *"prompt me, and hand `SECRET` over as the next positional argument"* —
and the positional argument that follows is the **database name**. That is why a command that looks correct fails
with `Unknown database '...'` or `Table 'events_log' doesn't exist`.

*(ELI5: `-p` is a sticky label, not a flag with a value. Whatever is glued to it is the password; whatever comes
after a space is a different passenger — the database.)*

**Rule 2 — the last bare word is the database.** Either name the database as the final argument, or fully qualify
every table as `database.table`:

```bash
mysql -h mysql.50webs.com -u ejaguiar1_events -pPASS ejaguiar1_events
#                                                         ^^^^^^^^^^^^^^^^ DB selected -> events_log works

mysql -h mysql.50webs.com -u ejaguiar1_events -pPASS -e "SELECT * FROM ejaguiar1_events.events_log LIMIT 5;"
#                                                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ no DB selected -> fully qualify
```

Those two rules explain the confusing case: write `-p MYPASS` and `MYPASS` is swallowed as the *database name*, so
the client ends up with **no database selected** and every unqualified table fails — which pushes you into the
`db.table` form for the rest of the session. Both of the things below are the same bug wearing two hats:

```powershell
# how it looks when it goes wrong
.\mysql -h mysql.50webs.com -u ejaguiar1_events -p MYPASS -e "SELECT * FROM events_log;"
#   ERROR 1049 (42000): Unknown database 'MYPASS'          <- the space after -p
#   ERROR 1146 (42S02): Table 'events_log' doesn't exist    <- no database selected
```

### Linux / macOS

The `mysql` client must be on `PATH`:

```bash
sudo apt-get update && sudo apt-get install -y default-mysql-client   # Debian/Ubuntu
mysql --version
```

Interactive session — `-p` prompts, so the password never reaches shell history or the process list:

```bash
mysql -h mysql.50webs.com -u ejaguiar1_events -p ejaguiar1_events
```

One-shot query, no interactive session (fully qualified, because `-e` with `-p` leaves no room for a database
argument when the password is typed at the prompt):

```bash
mysql -h mysql.50webs.com -u ejaguiar1_events -p -e \
  "SELECT COUNT(*) AS total FROM ejaguiar1_events.events_log;"
```

### Windows — PowerShell (what you get from the Start menu)

`mysql.exe` is not on `PATH` on a stock install, so call it by full path. In PowerShell the continuation character is
a **backtick**, *not* `^` — `^` is cmd.exe syntax and does nothing here:

```powershell
& "C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe" `
  -h mysql.50webs.com -u ejaguiar1_events -p `
  -e "SELECT * FROM ejaguiar1_events.events_log LIMIT 5;"
```

Or `cd` into the bin directory and use `.\mysql`, which is the same thing with less typing:

```powershell
cd "C:\Program Files\MySQL\MySQL Server 8.0\bin"
.\mysql -h mysql.50webs.com -u ejaguiar1_events -p -e "SELECT * FROM ejaguiar1_events.events_log LIMIT 5;"
```

Two PowerShell-specific notes:

- Keep the SQL in **double quotes**. Unquoted, PowerShell treats `;` as the end of a statement and splits your query.
- Use backtick + newline for multi-line commands; after a backtick there must be **no trailing space**.

### Windows — cmd.exe

```bat
"C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe" ^
  -h mysql.50webs.com -u ejaguiar1_events -p ^
  -e "SELECT * FROM ejaguiar1_events.events_log LIMIT 5;"
```

Here `^` *is* the continuation character (and it does nothing in PowerShell), so the two above are not
interchangeable. Adding `C:\Program Files\MySQL\MySQL Server 8.0\bin` to `PATH` lets every command in this repo be
typed as plain `mysql`.

### Inline passwords: `-pPASS` works, and is a bad habit

```powershell
& mysql.exe -h mysql.50webs.com -u ejaguiar1_events -pPASS -e "SELECT 1;"   # no space after -p
```

It works, and it also lands in PowerShell history (`Get-Content (Get-PSReadLineOption).HistorySavePath`), in Bash
history, in `ps` output and in the Windows process list for as long as the query runs. For a shared or scripted
client, use an options file instead — `--defaults-extra-file` must be the **first** option:

```ini
# %APPDATA%\MySQL\.my.cnf   (Linux/macOS: ~/.my.cnf, chmod 600)
[client]
host=mysql.50webs.com
user=ejaguiar1_events
password=REDACTED
```

```bash
mysql --defaults-extra-file="$HOME/.my.cnf" -e "SELECT COUNT(*) FROM ejaguiar1_events.events_log;"   # Linux
mysql --defaults-extra-file="%APPDATA%\MySQL\.my.cnf" -e "SELECT COUNT(*) FROM ejaguiar1_events.events_log;"   # cmd.exe
```

**Password characters that need care:** `%` is an environment-variable marker in `cmd.exe` (double it: `%%`), `$`
and backticks expand inside PowerShell double quotes, and `!` is history expansion in interactive Bash. None of this
matters with `-p` (prompt) or an options file, which is the real reason to prefer them.

### Queries worth knowing

The block below starts with `USE`, so the unqualified table names work. For a one-shot `-e` query without
that line, fully qualify them (`ejaguiar1_events.events_log`).

```sql
USE ejaguiar1_events;

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

## 4. Where the "service" is — and how autostart actually works (guide for Linux newcomers)

<a id="part-c-where-the-service-is"></a>

> **Written for someone whose only mental model is `services.msc`.** Every Windows concept is translated line by
> line, and every Linux command below is followed by what it *means* in Windows terms.

### ELI5 — on Windows there is **one** row, here there are **four** layers

On Windows, `services.msc` gives you one row that answers everything at once: *what* runs, *as whom*, and *does it
start at boot*. All three facts live in one object. *(ELI5: one box with one label.)*

On this Linux machine the same job is split into four stacked layers, and each layer can be healthy or broken
independently. That is the entire reason "where is the service?" feels unanswerable — the honest answer is
"which layer do you mean?".

```text
 ┌─ layer 1 ────────────────────────────────────────────────────────────┐
 │ systemd  →  docker.service            "is Docker itself running?"    │  ← the only real Windows-style service
 └──────────────────────────────┬───────────────────────────────────────┘
                                │ starts / adopts containers with a restart policy
 ┌─ layer 2 ────────────────────▼───────────────────────────────────────┐
 │ container  sn-mid            --restart unless-stopped                │  ← "the service instance"
 └──────────────────────────────┬───────────────────────────────────────┘
                                │ PID 1 inside the container = the supervisor
 ┌─ layer 3 ────────────────────▼───────────────────────────────────────┐
 │ /opt/mid-supervisor.sh       relaunches the JVM whenever it exits    │  ← replaces the x86-only wrapper
 └──────────────────────────────┬───────────────────────────────────────┘
                                │ exec
 ┌─ layer 4 ────────────────────▼───────────────────────────────────────┐
 │ java com.service_now.mid.Main start    -Xmx1024m  --add-opens ...     │  ← the actual MID "program"
 └──────────────────────────────────────────────────────────────────────┘
```

| Layer | ELI5 | Windows equivalent | How to see it | Who starts it |
|---|---|---|---|---|
| 1 — `docker.service` | the engine that runs containers | a service with *Startup type: Automatic* | `systemctl status docker` | **systemd** (at boot, because it is *enabled*) |
| 2 — container `sn-mid` | one running "instance" of the MID | the service *instance* | `docker ps -a` | the Docker engine (at boot, because of the restart policy) |
| 3 — `/opt/mid-supervisor.sh` | babysitter that restarts the program | *Recovery → Restart the service* | `docker exec sn-mid pgrep -af mid-supervisor` | container `CMD` (PID 1) |
| 4 — `java … mid.Main start` | the program itself | *Path to executable* | `docker exec sn-mid pgrep -af service_now.mid.Main` | the supervisor (layer 3) |

**So, "where is the service?"** — there is deliberately **no `sn-mid.service`** file on this box. The autostart
chain is: *systemd* starts `docker.service` → Docker starts the container because of `--restart unless-stopped` →
the container runs `mid-supervisor.sh` → the supervisor runs the JVM. Delete any one link and the MID does not come
back after a reboot; the other three still *look* fine, which is exactly how people end up staring at a "Down" MID.

### Auto-start: how Windows does it vs how Linux does it

Windows has one mechanism: **Startup type** (*Automatic* / *Manual* / *Disabled*) plus the **Recovery** tab
(restart-on-failure). Linux splits that into two mechanisms that get confused constantly:

1. **`systemctl enable`** — "start this at boot". This is literally *Startup type: Automatic*. It creates a symlink
   under `/etc/systemd/system/multi-user.target.wants/`; nothing in the unit changes, the link is the switch.
2. **Docker restart policies** — "if this container exits, start it again". `unless-stopped` also means "when the
   engine comes up at boot, bring this container up too".
3. **Inside the container**, something has to supervise the program. In Windows the wrapper does that; here
   `mid-supervisor.sh` does, because the wrapper is x86-only.

| Question | Windows answer | Answer on this box | Command |
|---|---|---|---|
| Does it start at boot? | Startup type: Automatic | `docker.service` is enabled **and** the container's policy is `unless-stopped` | see the 3-line check below |
| Does it restart if it crashes? | Recovery → Restart | supervisor relaunches the JVM (10 s, then 60 s after 3 fast crashes) | `grep '\[supervisor\]' …/console.log` |
| Does "Stop" survive a reboot? | Yes, until you set Automatic again | Yes — `unless-stopped` means exactly "don't bring it back if a human stopped it" | `docker stop sn-mid` |
| Can I start it by hand? | `net start snc_mid` | `docker start sn-mid` | — |

**The three-line check** — run it after every reboot, or whenever the instance says *Down*:

```bash
# LAYER 1 — "is the engine set to start at boot?"   (Windows: Startup type)
systemctl is-enabled docker      # -> enabled   (disabled = the container can never come up at boot)
systemctl is-active docker       # -> active    (running right now)

# LAYER 2 — "will this container be brought back?"  (Windows: Recovery / Automatic)
docker inspect sn-mid --format '{{.HostConfig.RestartPolicy.Name}}'   # -> unless-stopped
docker inspect sn-mid --format '{{.State.Status}} since {{.State.StartedAt}}'

# LAYERS 3+4 — "is the program actually running inside it?"
#   (the container can be "Up" with nothing but a `tail` alive — this is the classic false green)
docker exec sn-mid pgrep -af service_now.mid.Main || echo "JVM DEAD - restart: docker restart sn-mid"
```

*(ELI5: layer 1 says the building has power, layer 2 says your office is rented, layer 3 says somebody is actually
sitting in it. Windows hides all three in one row of `services.msc`.)*

**If layer 1 is missing** (fresh Docker install, or someone disabled it):

```bash
sudo systemctl enable --now docker    # = set Startup type to Automatic AND start it now
systemctl is-enabled docker           # -> enabled
```

**If layer 2 is missing** (`RestartPolicy` is `no` or empty — e.g. the container was created without `--restart`):

```bash
docker update --restart unless-stopped sn-mid     # change it in place, no need to recreate the container
docker inspect sn-mid --format '{{.HostConfig.RestartPolicy.Name}}'   # -> unless-stopped
```

Same effect as editing *Recovery* in Windows service properties — and it needs no restart of anything.

### Optional: a real `sn-mid.service` (one row, like `services.msc`)

If you would rather have a single object you can `systemctl status` and see on boot, wrap the container in a unit.
This is the Linux equivalent of "create the service entry":

```bash
sudo tee /etc/systemd/system/sn-mid.service >/dev/null <<'EOF'
[Unit]
Description=ServiceNow MID Server (ARM64 container sn-mid)
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target

[Service]
Restart=always
RestartSec=10
ExecStart=/usr/bin/docker start -a sn-mid
ExecStop=/usr/bin/docker stop -t 30 sn-mid
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF
```

Then:

```bash
sudo systemctl daemon-reload          # make systemd notice the new file (always after editing units)
sudo systemctl enable --now sn-mid     # Startup type: Automatic + start it now
systemctl status sn-mid --no-pager     # the services.msc "row": state, PID-less, recent log lines
systemctl is-enabled sn-mid            # -> enabled
```

Two cautions, both learned the hard way:

- **Do not run both supervisors.** If you add this unit, set the container policy to `no`
  (`docker update --restart no sn-mid`), otherwise systemd and the Docker engine both think they own the container.
  With the unit in place, *Recovery → Restart* comes from `Restart=always`, not from Docker.
- **`ExecStart` must attach** (`docker start -a`). A plain `docker start` returns immediately, so systemd thinks the
  service exited and restart-loops it. *(ELI5: don't hand the job to someone who leaves before it's finished.)*

**Test that autostart really works** — do this once, on purpose, rather than discovering it during an outage:

```bash
sudo reboot
# ...after it comes back:
systemctl is-enabled docker && docker ps --filter name=sn-mid --format '{{.Status}}'
curl -s -u "$SN_USER:$SN_PASS" \
  "https://$SN_HOST/api/now/table/ecc_agent?sysparm_query=nameLIKEgx10&sysparm_fields=name,status,validated,last_refreshed"
# expect: docker enabled, container "Up …", instance status=Up validated=true
```

### `services.msc`, field by field → Linux equivalent

| Windows `services.msc` shows | Linux / Docker equivalent | Command |
|---|---|---|
| The service list | running containers + systemd services | `docker ps -a` / `systemctl list-units --type=service --state=running` |
| Service name `snc_mid` | container **`sn-mid`**, kept alive by systemd unit `docker.service` | `docker ps -a --filter name=sn-mid` |
| Display name *ServiceNow MID Server* | container `Name` / the MID name shown in the instance (`gx10-mid`) | `docker inspect sn-mid --format '{{.Name}}'` |
| *Path to executable* | the container's `Path` + `Args` (`/bin/bash -c "…supervisor… tail"`) | `docker inspect sn-mid --format '{{.Path}} {{.Args}}'` |
| *Start in* (working directory) | `Config.WorkingDir` | `docker inspect sn-mid --format '{{.Config.WorkingDir}}'` |
| *Log On As: Local System* | `Config.User=root`; the JVM inside drops to user **`mid`** (uid 999) | `docker inspect sn-mid --format '{{.Config.User}}'` |
| *Startup type: Automatic* | `RestartPolicy=unless-stopped` (plus `docker.service` enabled) | `docker inspect sn-mid --format '{{.HostConfig.RestartPolicy.Name}}'` |
| *Start / Stop / Restart* buttons | `docker start` / `stop` / `restart sn-mid` | same |
| *Program parameters* / config file | `/opt/agent/config.xml` inside the container | `docker exec sn-mid cat /opt/agent/config.xml` |
| *Environment variables* | `Config.Env` (`MID_XMX`, `OPENS`) | `docker inspect sn-mid --format '{{range .Config.Env}}{{println .}}{{end}}'` |
| *Dependencies* tab | `Requires=docker.service` / `After=…` in the unit, or `docker inspect` `Mounts` for volumes | `docker inspect sn-mid --format '{{range .Mounts}}{{.Name}} -> {{.Destination}}{{"\n"}}{{end}}'` |
| *Event Viewer* → Application log | `/opt/agent/logs/agent0.log.0` (MID log) + `/opt/agent/logs/console.log` (supervisor) | `docker exec sn-mid tail -50 /opt/agent/logs/agent0.log.0` |
| *Recovery* tab | supervisor loop; `Restart=always` if you installed the unit | `grep '\[supervisor\]' …/console.log` |

### See the full configuration (the `services.msc` "Path" detail)

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

The MID's own settings file is `/opt/agent/config.xml` — instance URL, MID user, MID name, threads, proxy:

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

Ubuntu has no built-in `services.msc`. _(ELI5: there is no single window that lists everything, because "everything"
is four layers.)_ What you can still do:

- **Terminal** — `Ctrl`+`Alt`+`T`, or Activities → type "Terminal". This is the real tool; everything below is
  convenience only.
- **Docker Desktop for Linux** *(if you install it)* — shows the container, its start command, exit state and live
  logs in a window. Closest visual match to `services.msc` for layer 2.
- **Processes** — open "System Monitor" (`gnome-system-monitor`). You will see `java`, `dockerd`, `containerd`.
  Right-click → *End* is the GUI version of "Stop".
- **Files (Nautilus)** — browse the volume as root:
  `sudo -E nautilus /var/lib/docker/volumes/mid_data/_data &` — handy for reading `config.xml` and `logs/` in a
  text editor.
- **Settings** (`gnome-control-center`) manages *nothing* here — no services, no containers.
- Optional GUI service managers, not preinstalled: `sudo apt install stacer` (or `systemd-manager`). These list
  **systemd** units, so they show `docker.service` (and `sn-mid.service` if you added it) but **not** the container.

### Changing the configuration — safely

Don't hand-edit credentials into `config.xml`. The setup script regenerates that file *and* re-derives the keystore
from it; a half-edited file is what produces the `UnrecoverableKeyException` / `401` pair described in section 2.
_(ELI5: two locks share one key — change one by hand and the other stops opening.)_ Supported route:

```bash
export SN_HOST="dev296154.service-now.com" MID_USER="mid.server" MID_NAME="gx10-mid"
read -rsp "MID user password: " MID_PASS; export MID_PASS
bash ./setup-mid-fast.sh     # regenerates config.xml, clears keystore, relaunches, polls
```

For something the script doesn't cover (thread count, proxy), copy the file out, edit, put it back, restart:

```bash
docker cp sn-mid:/opt/agent/config.xml ./config.xml
nano ./config.xml                                        # or any editor you like
docker cp ./config.xml sn-mid:/opt/agent/config.xml
docker restart sn-mid
```

**Never use the instance's "Restart MID" button on this box** — a wrapper-less MID cannot restart itself, so it
shuts down and stays down while `docker ps` still says *Up*. `docker restart sn-mid` is the correct action; see
[Why it keeps happening](#why-it-keeps-happening) and the
[recovery runbook](https://github.com/eltonaguiar/ServiceNowMidServer#recovering-a-down-mid--the-runbook-we-actually-ran).

### Open a shell inside the container (the closest thing to RDP)

```bash
docker exec -it sn-mid bash      # or: sh
pwd                              # /opt/agent
ls                               # config.xml  logs/  lib/  extlib/  work/  security/
```

### TL;DR of this section

- There is **no `sn-mid.service`**. The "service" is four layers: `docker.service` → container `sn-mid` →
  `mid-supervisor.sh` → `java … mid.Main start`.
- Boot autostart needs **two** things: `systemctl is-enabled docker` = `enabled`, **and** the container's
  `RestartPolicy` = `unless-stopped`. Either one missing = no MID after a reboot.
- Restart-on-crash is the **supervisor** (10 s, backing off to 60 s), not Docker — Docker only sees the supervisor.
- Want a single `services.msc`-style row? Add the optional unit, and switch the container policy to `no` so two
  supervisors don't fight.

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
