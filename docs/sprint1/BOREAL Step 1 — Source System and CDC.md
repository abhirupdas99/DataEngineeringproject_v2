# BOREAL Step 1 — Source System and CDC

Oct 24, 2026 · @Abhirup

## Goal

By the end of this step you will have watched a deleted row travel out of a transaction log and into a change table. That is a different kind of knowing to having read about CDC, and it is what this step exists for.

Concretely you will have:

- A `boreal` git repo with three SQL scripts in it
- A local SQL Server running `boreal_ops`
- CDC enabled on `dbo.employer`
- Evidence, in the form of change-table rows, that inserts, updates and deletes are all captured

### Why the source system comes first

Everything in BOREAL is downstream of a change feed. There is no point building a landing zone before you can prove something changes in a way you can detect.

There is a second reason, and it is the more important one: this SQL Server is a sandbox you can break. You have a live PostgreSQL database feeding real job postings, and that database is load-bearing for your actual job search. You do not learn change data capture on a system you depend on. You learn it here, break it here, and then apply what you learned to the real source in a later step.

### What this step earns you in an interview

"Have you worked with CDC?" is a question most candidates answer from a blog post. After this step you can answer it from a transaction log.

Specifically you will be able to say why a timestamp watermark is not good enough, what an LSN is and why it beats a timestamp as a watermark, and what happens to your pipeline when SQL Server Agent stops. The third one is the tell. Nobody who has only read about CDC knows that it runs as Agent jobs.

## Prerequisites and install

Budget about 30 minutes, most of it waiting on the installer.

| What | Which | Why this one |
| --- | --- | --- |
| SQL Server | 2022 or 2025, **Developer Edition** | Free for non-production and supports CDC. Express does **not** support CDC, so it is a dead end for this project. 2025 is current; CDC behaves identically in both. |
| Client | SSMS 22 | Microsoft has retired Azure Data Studio, so SSMS is the tool. Separate download — see the section below. |
| Git | Any recent version | The scripts go in a repo from the first line, not after. |

### The one setting that will waste your evening

**SQL Server Agent must be running, and set to start automatically.**

CDC is not a feature that runs inside your query. It is two SQL Server Agent jobs: a *capture* job that reads the transaction log and writes rows into the change tables, and a *cleanup* job that deletes change rows past the retention window. If Agent is stopped, CDC silently stops capturing. You get no error. Your change table just stays empty while everything else looks fine.

During install, on the Server Configuration page, set SQL Server Agent to **Automatic** rather than Manual. If you have already installed, fix it in SQL Server Configuration Manager.

Verify it before you go further:

```sql
SELECT servicename, status_desc, startup_type_desc
FROM   sys.dm_server_services;
```

You want to see SQL Server Agent with `status_desc = 'Running'`. If it says Stopped, nothing later in this document will work and the failure will not announce itself.

### Authentication

Windows authentication against `localhost` is fine for this step. You will add a least-privilege SQL login for Data Factory in step 2, when there is something for it to connect to. Do not create service accounts before you have a service.

## Installing SQL Server, page by page

Choose **Custom**, not Basic. Basic skips the two configuration pages you need, leaving SQL Server Agent on Manual startup and the server on Windows authentication only — both of which you would have to retrofit later.

Fourteen pages. Four need a real decision; the rest you click through.

1. **Download and run the installer.** Get Developer Edition from Microsoft. The download is a small bootstrapper, not the full product.
2. **Installation Type → Custom.** It then asks where to put the setup files. Accept the default, click Install, and wait while it downloads roughly 1.5 GB.
3. **SQL Server Installation Center** opens. In the left nav choose **Installation**, then **New SQL Server standalone installation or add features to an existing installation**.
4. **Edition → Developer.** Free, full feature set, licensed for non-production use. Not Express — Express cannot do CDC.
5. **License Terms** → accept.
6. **Microsoft Update** → optional. Leaving it unchecked makes the install faster.
7. **Install Rules** → you will get a **Windows Firewall warning**. Expected, and safe to ignore — you are connecting over `localhost`, not across a network.
8. **Azure Extension for SQL Server → UNCHECK IT.** This page is new in 2022 and it is checked by default. It wants an Azure subscription and Arc onboarding, contributes nothing to this project, and will stall you if you leave it on without credentials to hand.
9. **Feature Selection → Database Engine Services only.** Skip Machine Learning Services, Analysis Services, Integration Services, PolyBase, Data Quality Services and Full-Text Search. None are used here and together they add gigabytes.
10. **Instance Configuration → Default instance** (`MSSQLSERVER`). Choose a named instance only if this machine already has SQL Server. A default instance means you connect to `localhost` instead of `localhost\SOMETHING`, which keeps every connection string in the project simpler.
11. **Server Configuration** → two tabs. **This page is the reason you picked Custom.**
    - *Service Accounts*: set **SQL Server Agent → Startup Type: Automatic**. It defaults to Manual. Leave SQL Server Database Engine on Automatic, and SQL Server Browser can stay Disabled for a default instance.
    - *Collation*: leave the default. You are using `NVARCHAR` for the name columns, so accented Canadian company names are handled regardless.
12. **Database Engine Configuration** → six tabs, three worth your attention.
    - *Server Configuration*: choose **Mixed Mode**, set an `sa` password, and click **Add Current User** so your Windows account is a SQL admin. You need Mixed Mode in step 2 for the Data Factory login. If you skip the Add Current User click you get a "Missing system administrator account" error and cannot continue.
    - *Memory*: accept the **recommended** maximum rather than the default unlimited. On a laptop this is what stops SQL Server gradually claiming all your RAM and making everything else crawl.
    - *Data Directories, TempDB, MaxDOP, FILESTREAM*: defaults are correct for a development machine.
13. **Ready to Install** → review the summary and click Install. Roughly ten minutes.
14. **Complete** → then install **SQL Server Management Studio separately**. It has not shipped with the installer for several versions.

### Confirm it worked

Before anything else, run this:

```sql
SELECT servicename, status_desc, startup_type_desc
FROM   sys.dm_server_services;
```

You want SQL Server Agent showing `Running` and `Automatic`. If it is stopped or Manual, fix it in SQL Server Configuration Manager now. Discovering it later, when your change table is inexplicably empty, is a far worse way to spend an evening.

## SQL Server Management Studio

SSMS is where you will spend the next two steps, so it is worth ten minutes of orientation rather than fumbling through it.

It does not ship with the SQL Server installer and has not for several versions. Azure Data Studio, which used to be the alternative, has been retired by Microsoft in favour of the MSSQL extension for VS Code — so for this project, use SSMS.

### Installing it

SSMS 22 is the current version and it installs through the **Visual Studio Installer**, not a standalone MSI. That surprises people who last installed SSMS a few years ago.

1. Download the bootstrapper, `vs_SSMS.exe`, from the Microsoft SSMS download page.
2. **Run it as administrator.** Accept the UAC prompt and the licence acknowledgment.
3. The Visual Studio Installer opens. Leave all six workloads unchecked — the base install has everything this project needs. The core editor is always included and gives you the query editor, Object Explorer and Agent job management. Skip AI Assistance in particular: this step asks you to look up procedure parameters from the documentation yourself, and Copilot would simply hand them to you. Any workload can be added later through the installer's Modify option, so nothing here is a permanent decision.
4. If your C: drive is tight, this is the only moment you can choose a different drive for SSMS and its download cache.
5. Click **Install**.

It installs side-by-side with older versions, so an existing SSMS 20 or 21 stays put.

### Connecting for the first time

| Field | Value |
| --- | --- |
| Server type | Database Engine |
| Server name | `localhost` |
| Authentication | Windows Authentication |
| Encryption | Optional, or trust the server certificate |

If the connection is refused, the SQL Server service is not running — start it in SQL Server Configuration Manager.

On **Encryption**: SSMS 20 and later default to requiring encryption, and a local SQL Server has only a self-signed certificate. You will likely need to tick **Trust server certificate** on the connection dialog. That is fine on `localhost`. It would not be fine against a production server, where that checkbox is how you silently disable the protection against a man-in-the-middle, and an interviewer may well ask you about it.

### The habit that matters most

**A new query window connects to `master`, not to your database.**

This catches everyone once. You write `CREATE TABLE dbo.employer`, hit F5, get a success message, and have just created the table in `master`. Then nothing downstream can find it and the error makes no sense.

Two defences, and use both:

- Start **every** script with `USE boreal_ops;` followed by `GO`. Your scripts are run by other tools later — sqlcmd, CI, Data Factory — none of which have a database dropdown.
- Watch the database dropdown in the toolbar before you press F5.

### Do not use the GUI designers

SSMS will happily let you create a table by right-clicking and filling in a grid. **Do not**, not once, in this project.

Everything that defines the database lives in a script in your git repo. The moment you make a change through a designer, your database and your repo have diverged, and you can no longer rebuild from source. This is the same principle as the Terraform rule later on: if it was clicked into existence, it does not really exist.

The one acceptable GUI use is right-click → **Script Table as → CREATE To** on something you want to inspect, which generates SQL you can read.

### Where the CDC objects actually live

There is no "Change Data Capture" node in Object Explorer, which confuses people looking for one. The objects are scattered:

| What | Where in Object Explorer |
| --- | --- |
| Change tables (`cdc.dbo_employer_CT`) | `boreal_ops` → Tables → **System Tables** |
| CDC functions (`fn_cdc_get_all_changes_*`) | `boreal_ops` → Programmability → Functions → **Table-valued Functions** |
| Capture and cleanup jobs | **SQL Server Agent** → Jobs (at server level, not under the database) |
| Job run history | Right-click a job → **View History** |

That last one is worth knowing now. When CDC appears to have stopped, job history is where you find out why.

### Settings worth changing immediately

Tools → Options:

- **Text Editor → Transact-SQL → General → Line numbers.** Error messages reference line numbers; without this you are counting by hand.
- **Query Results → SQL Server → Results to Grid → Maximum Characters Retrieved**. The default truncates long text columns, which will bite you when you start reading HTML job descriptions.
- **Text Editor → All Languages → Tabs → Insert spaces.** Keeps diffs clean in git.

### Shortcuts you will use constantly

| Keys | Does |
| --- | --- |
| `F5` | Execute. Selecting text first executes **only** the selection — the single most useful habit in SSMS |
| `Ctrl` + `R` | Show or hide the results pane |
| `Ctrl` + `K`, `Ctrl` + `C` | Comment the selected lines |
| `Ctrl` + `K`, `Ctrl` + `U` | Uncomment |
| `Ctrl` + `Shift` + `U` | Uppercase the selection — handy for keywords |
| `F8` | Toggle Object Explorer |
| `Ctrl` + `M` | Include the actual execution plan, for when you start tuning |

Executing only the selection is the one to internalise. It is how you run a single statement out of a long script without splitting the file.

## Concepts before you write any SQL

Four ideas. Do not start typing until each one makes sense, because the scripts are trivial and the understanding is the whole point.

### 1. Why a timestamp watermark is not enough

The obvious way to load incrementally is `WHERE updated_at > @last_run`. It has three failure modes, and you should be able to name all three without thinking.

1. **Hard deletes are invisible.** The row is gone. There is nothing left to select, so your target keeps a record the source no longer has.
2. **Updates that miss the column are invisible.** If any code path updates a row without touching `updated_at`, that change never ships. This is extremely common and nobody notices for months.
3. **Clock skew loses rows at the boundary.** A transaction that commits at 02:00:00.001 but stamped itself 01:59:59.999 falls in the gap between two runs.

CDC reads the transaction log, which is the database's own record of what it actually did. It sees every operation, including the deletes.

### 2. LSN — Log Sequence Number

An LSN is a position in the transaction log. Monotonically increasing, totally ordered, assigned by the database itself.

This is what makes it a better watermark than a timestamp: it is the database's own notion of ordering, not a wall clock that can drift, repeat during a DST change, or disagree between the source and your pipeline. When you ask CDC for "everything since X", X is an LSN.

### 3. `__$operation` — what happened to the row

Every row in a change table carries this column:

| Value | Meaning |
| --- | --- |
| 1 | Delete |
| 2 | Insert |
| 3 | Update — the **before** image |
| 4 | Update — the **after** image |

An update produces two rows, not one. Work out why before you run the experiment; the reason is in question 2 at the end of this document.

### 4. Change data has a retention window

The cleanup job deletes change rows older than a retention period. Find out what the default is — it is shorter than you would guess, and it is the single most common way a CDC pipeline breaks in production.

Think through what happens if your pipeline is down for longer than that window, and what your extract should do when it asks for an LSN that has already been cleaned up. You do not need to solve it now. Step 2 does. But notice the problem exists.

## Task A — repo and database

Create the repo first. Scripts that exist outside version control for "just today" are how projects end up with three divergent copies of the same DDL.

```bash
mkdir boreal && cd boreal
git init
mkdir -p sql/onprem
```

Add a `.gitignore` before the first commit. At minimum: `*.bak`, `*.mdf`, `*.ldf`, `.env`, and your editor's junk. Database files in git is a mistake you only make once.

### Write `sql/onprem/01_database.sql`

It should create `boreal_ops` if it does not already exist, and set the recovery model to FULL.

Make the script **re-runnable**. Running it twice should not error. Every script in this project will be run more than once — that is not a nice-to-have, it is the same idempotency principle the pipelines run on, applied to DDL. Wrap the create in a check against `sys.databases`.

### Why FULL recovery

CDC reads the transaction log. Under SIMPLE recovery the log is truncated at every checkpoint, so you would be reading a record that keeps being thrown away underneath you.

The trade-off worth knowing, because it is a reasonable interview follow-up: FULL recovery means the log grows until you back it up. On a laptop with no backup job, that log will grow forever. For a sandbox this is fine. In production it is a thing you plan for, and "we turned on CDC and filled the disk" is a real incident that happens to real teams.

### Commit

```bash
git add -A && git commit -m "Create boreal_ops with FULL recovery"
```

Commit after each of the three tasks, not once at the end. Small commits with real messages are a habit worth building now, while the stakes are zero.

## Task B — the employer table

Write `sql/onprem/02_employer.sql` creating `dbo.employer`. This table is the reference data for the whole platform: the employers you track, their ATS board tokens, their industry codes. Low volume, slowly changing — which makes it the right source for an SCD Type 2 dimension later.

Use exactly these columns so everything downstream lines up:

| Column | Type | Notes |
| --- | --- | --- |
| `employer_id` | `INT IDENTITY(1,1)` | Primary key |
| `employer_code` | `VARCHAR(60) NOT NULL` | Unique. Your stable business key, e.g. `shopify` |
| `legal_name` | `NVARCHAR(200) NOT NULL` | `NVARCHAR`, not `VARCHAR` — see below |
| `website_domain` | `VARCHAR(200) NULL` |  |
| `naics_code` | `VARCHAR(10) NULL` | Industry classification |
| `hq_city` | `NVARCHAR(100) NULL` |  |
| `hq_province` | `CHAR(2) NULL` | Two-letter code, constrained |
| `ats_provider` | `VARCHAR(30) NULL` | greenhouse, lever, ashby, workday, other |
| `ats_board_token` | `VARCHAR(100) NULL` | The public board token |
| `is_active` | `BIT NOT NULL DEFAULT 1` |  |
| `created_at` | `DATETIME2(3) NOT NULL` | Default `SYSUTCDATETIME()` |
| `updated_at` | `DATETIME2(3) NOT NULL` | Default `SYSUTCDATETIME()` |

### Three decisions in that table worth understanding

**`NVARCHAR` on the name and city columns.** You are storing Canadian company names. `Hydro-Québec` and `Société générale` are not hypothetical. `VARCHAR` in a non-Unicode collation will mangle them, and you will find out months later when a dashboard shows `Hydro-Qu?bec`.

**`SYSUTCDATETIME()`, not `GETDATE()`.** Store UTC. Your sources span time zones, your pipeline runs on Toronto time, and Azure runs on UTC. Mixing them produces off-by-five-hours bugs that are miserable to diagnose. Convert to local time at the presentation layer, never in storage.

**`employer_code` separate from `employer_id`.** The identity column is a surrogate key — meaningless, database-generated. `employer_code` is the business key. You need both: the surrogate for joins, the business key so that when a record is deleted and re-inserted it is still recognisably the same employer. Conflating them is a classic modelling mistake.

### Add two CHECK constraints

Constrain `hq_province` to the thirteen valid Canadian province and territory codes, and `ats_provider` to its allowed values.

This is deliberate. Constraints at the source are the cheapest possible data quality control — bad data never enters the system, so no downstream rule has to catch it. Later in the project you will build an entire validation framework for rules that cannot be expressed as constraints. Anything that *can* be a constraint should be one.

## Task C — enable CDC

Write `sql/onprem/03_enable_cdc.sql`. CDC is enabled in two stages: once at database level, then separately for each table you want captured.

The two procedures you need:

- `sys.sp_cdc_enable_db`
- `sys.sp_cdc_enable_table`

**Look up the parameters yourself.** This is the exercise, not a detail I am withholding to be difficult. Working out from the documentation which arguments matter is the skill; copying a snippet teaches you nothing and will not survive the follow-up question in an interview.

### What to pay attention to while you read the docs

Four of `sp_cdc_enable_table`'s parameters deserve a decision rather than a default:

- **`@role_name`** — controls who can read the change table. Think about what passing `NULL` means, and whether that is what you want. There is a least-privilege argument here.
- **`@capture_instance`** — names the capture instance, and therefore the change table and the functions generated from it. Naming it explicitly rather than taking the default matters more than it looks: you will reference this name in code.
- **`@supports_net_changes`** — enables a second function that collapses several changes to one row into its final state. Read what it requires of the table before you set it.
- **`@captured_column_list`** — you will not need it here, but know it exists. Capturing every column of a wide table when you need four is a real cost.

Make this script re-runnable too. Enabling CDC on a table that already has it will error, so guard it — `cdc.change_tables` joined to `sys.tables` will tell you what is already enabled.

### Verify before you move on

```sql
SELECT name, is_cdc_enabled FROM sys.databases WHERE name = 'boreal_ops';

SELECT s.name AS [schema], t.name AS [table],
       ct.capture_instance, ct.supports_net_changes
FROM   cdc.change_tables AS ct
JOIN   sys.tables   AS t ON t.object_id = ct.source_object_id
JOIN   sys.schemas  AS s ON s.schema_id = t.schema_id;
```

Also confirm the two Agent jobs now exist. Enabling CDC creates them, and seeing them appear is the clearest possible demonstration that CDC is Agent-driven:

```sql
SELECT name FROM msdb.dbo.sysjobs WHERE name LIKE '%cdc%';
```

You should see a capture job and a cleanup job, both named after the database.

## The experiment

This is the part that teaches. Everything before it was setup.

Run these as separate statements, in order, and watch what happens.

**1. Insert two employers.**

```sql
INSERT INTO dbo.employer (employer_code, legal_name, hq_province, ats_provider, ats_board_token)
VALUES ('shopify',      N'Shopify Inc.',             'ON', 'greenhouse', 'shopify'),
       ('wealthsimple', N'Wealthsimple Technologies','ON', 'greenhouse', 'wealthsimple');
```

**2. Update one of them.**

```sql
UPDATE dbo.employer
SET    legal_name = N'Shopify Commerce Inc.'
WHERE  employer_code = 'shopify';
```

**3. Delete the other.**

```sql
DELETE FROM dbo.employer WHERE employer_code = 'wealthsimple';
```

**4. Wait about 15 seconds.** The capture job polls the log on an interval; it is not instant. If you query immediately you will see an empty table and conclude CDC is broken. Waiting is part of the lesson — CDC is asynchronous, and anything you build on it inherits that latency.

**5. Look at what you caught.**

```sql
SELECT __$operation,
       __$seqval,
       CONVERT(VARCHAR(42), __$start_lsn, 1) AS lsn,
       sys.fn_cdc_map_lsn_to_time(__$start_lsn) AS commit_time,
       employer_code,
       legal_name
FROM   cdc.dbo_employer_CT
ORDER  BY __$start_lsn, __$seqval;
```

### What to notice

Read the output properly before moving on. Specifically:

- The **row count** against the number of statements you ran
- Which `__$operation` values appear, and in what order
- What `legal_name` holds on each row of the update pair
- That the deleted employer still has its full row data in the change table, even though the row no longer exists in `dbo.employer`

That last one is the whole argument for CDC in a single observation. A timestamp-based extract would have no way of knowing Wealthsimple was ever there.

### Then look at the LSN boundaries

```sql
SELECT CONVERT(VARCHAR(42), sys.fn_cdc_get_min_lsn('dbo_employer'), 1) AS min_lsn,
       CONVERT(VARCHAR(42), sys.fn_cdc_get_max_lsn(), 1)              AS max_lsn;
```

Note the asymmetry: one function takes a capture instance name and the other takes nothing. That is not an inconsistency in the API — it tells you something about what each value means. Work out what.

## Questions to bring back

Answer these in your own words before you read anything else. Paste them with your three scripts and I will review both the SQL and the reasoning.

**1. How many rows are in the change table?**

You ran three statements affecting four rows. Explain the number you actually got, and which statement produced more rows than you might expect.

**2. The update produced two rows. Why would a change feed record the *before* image at all?**

The after image is obviously useful — it is the new state. Make the case for keeping the before image. Think about what a consumer could do with it that it could not do otherwise. There is a good answer involving auditing and a good answer involving replication; either is fine if you can defend it.

**3. Of `min_lsn` and `max_lsn`, which one is your watermark?**

This is the question that matters. If you get this right the next step will feel inevitable rather than arbitrary.

Work through it rather than guessing. Ask yourself: what does each value actually represent? After a successful extract, which one do you store so the next run knows where to resume? And what does it mean if the LSN you stored is now *lower* than `min_lsn`?

That last part is not a trick. It is a real production failure mode with a specific name and a specific recovery procedure, and step 2 builds the code that handles it.

### Optional, if you want to go further

Try this and tell me what you observe:

- Stop SQL Server Agent, make another change to `dbo.employer`, wait 30 seconds, query the change table. Then start Agent again, wait, and query once more. What happened to the change you made while Agent was stopped — was it lost, or delayed?

The answer tells you something important about where CDC actually reads from, and it is a genuinely good thing to have observed first-hand.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Change table exists but stays empty | SQL Server Agent is not running | `SELECT servicename, status_desc FROM sys.dm_server_services;` then start it in Configuration Manager and set it to Automatic |
| `sp_cdc_enable_db` fails outright | Express Edition | CDC is not supported there. Reinstall Developer Edition — there is no workaround |
| Change table is empty for 30+ seconds after a change | Capture job latency, or the job is stopped | `EXEC sys.sp_cdc_help_jobs;` shows both jobs and their settings. The capture job can be started with `sys.sp_cdc_start_job` |
| `Invalid object name 'cdc.dbo_employer_CT'` | Your `@capture_instance` was named differently | Query `cdc.change_tables` to see the real name. This is why naming it explicitly matters |
| Enabling CDC errors on a re-run | Already enabled | Your guard is missing or wrong. Check `cdc.change_tables` joined to `sys.tables` |
| `fn_cdc_get_min_lsn` returns all zeros | No changes captured yet | Make a change, wait, retry |
| Accented names show as `?` | `VARCHAR` instead of `NVARCHAR`, or a missing `N` prefix on the literal | Both the column type and the `N'...'` prefix on the insert matter |

### If you need to start over

Disabling CDC on a table drops its change table and everything in it:

```sql
EXEC sys.sp_cdc_disable_table
     @source_schema = N'dbo',
     @source_name   = N'employer',
     @capture_instance = N'all';
```

That is fine here. Note for later that in production this is destructive and unrecoverable — any change not yet extracted is gone.

### A note on disk

FULL recovery with no log backup means `boreal_ops_log.ldf` grows and never shrinks. On a sandbox with a handful of rows this is irrelevant. If you later load real volume into this database and your disk starts filling, that is the cause, and the fix is a log backup, not a bigger disk.

## While the installer runs — profile the PostgreSQL source

Read-only, zero risk, and it is genuinely the first thing a data engineer does with an unfamiliar source. You cannot design an ingestion pattern for a table you have not looked at.

This is your live job-postings database on the VPS. **Read only.** No `CREATE`, no `ALTER`, and above all no replication slot — a slot with no consumer retains WAL indefinitely and will fill the disk on a box that is also running n8n.

```sql
-- 1. What is in there, and how big
SELECT relname AS table_name,
       n_live_tup AS approx_rows,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM   pg_stat_user_tables
ORDER  BY n_live_tup DESC;

-- 2. Columns and types for the postings table
SELECT column_name, data_type, is_nullable, column_default
FROM   information_schema.columns
WHERE  table_name = '<your postings table>'
ORDER  BY ordinal_position;

-- 3. Is logical replication even possible here?
SHOW wal_level;                       -- need 'logical'; 'replica' is the default
SELECT * FROM pg_replication_slots;   -- should be empty

-- 4. Daily volume - what does an incremental load actually have to handle?
SELECT created_at::date AS day, count(*)
FROM   <your postings table>
WHERE  created_at >= now() - interval '30 days'
GROUP  BY 1 ORDER BY 1;

-- 5. Is there a trigger maintaining updated_at?
SELECT tgname, pg_get_triggerdef(oid)
FROM   pg_trigger
WHERE  tgrelid = '<your postings table>'::regclass AND NOT tgisinternal;
```

### Report back on three things

**Is there an `updated_at`, and is it maintained on every update?** Query 5 answers this. If no trigger keeps it current, and the application does not set it on every write path, then incremental extraction on that column silently misses changes — failure mode 2 from the concepts section. This is the single most important fact to establish about any source, and most people never check.

**What is the natural key?** A unique constraint, or the combination of columns you would deduplicate on. If the same posting can appear twice from two job boards, what makes them the same posting?

**What is the daily row count?** Determines whether this is a two-minute copy or something that needs real design.

Do not build anything against Postgres yet. Profiling is reconnaissance; the ingestion design comes once the landing zone exists.

## Definition of done

Tick these off before moving to step 2.

- [ ] SQL Server Developer Edition installed, Agent running and set to Automatic
- [ ] `boreal` git repo created, with a `.gitignore`
- [ ] `01_database.sql` — creates `boreal_ops`, FULL recovery, re-runnable
- [ ] `02_employer.sql` — `dbo.employer` with both CHECK constraints, re-runnable
- [ ] `03_enable_cdc.sql` — CDC on database and table, re-runnable
- [ ] All three committed separately with real commit messages
- [ ] Experiment run; change table contains rows for the insert, update and delete
- [ ] Both CDC Agent jobs visible in `msdb.dbo.sysjobs`
- [ ] Answers written for the three questions
- [ ] PostgreSQL profiled, with findings on `updated_at`, natural key and daily volume

### What step 2 builds on this

Step 2 is the control framework — the `ctl` schema and the logic that turns "CDC is enabled" into "extraction is safe to retry".

It will cover:

- A source registry, so adding a new source is an `INSERT` rather than a code change
- A watermark table, and the **two-phase commit** that makes extraction crash-safe: pin the ceiling, extract, land, and only then advance the watermark
- Detection and recovery for the expired-LSN case you thought about in question 3
- A run log and file manifest, so every load is auditable and reconciliation balances

The reason step 1 is mostly reading and three short scripts is that step 2 is where the real engineering starts, and it only makes sense if you understand what an LSN is and why it moves. If question 3 is still fuzzy, say so when you report back and we will work through it before moving on — that is not falling behind, it is the correct order to learn this in.
