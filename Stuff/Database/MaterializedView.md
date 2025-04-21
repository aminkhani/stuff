```table-of-contents
title: Table of Contents
style: nestedList # TOC style (nestedList|nestedOrderedList|inlineFirstLevel)
minLevel: 0 # Include headings from the specified level
maxLevel: 0 # Include headings up to the specified level
includeLinks: true # Make headings clickable
hideWhenEmpty: false # Hide TOC if no headings are found
debugInConsole: false # Print debug info in Obsidian console
```
## 🗄️ What Is a Materialized View?

A **Materialized View** stores the result of a query physically on disk, rather than re‑computing it on each access (like a normal view). This lets you:

- 🚀 **Speed up** expensive aggregations or joins
- 🔒 **Snapshot** data at a point in time
- 💾 **Reduce load** on underlying tables
---
## ⚙️ Creating a Materialized View

```sql
CREATE MATERIALIZED VIEW sales_summary_matview AS
SELECT
  region,
  COUNT(*)    AS total_orders,
  SUM(amount) AS total_revenue
FROM orders
GROUP BY region;
```
- 📌 **Name:** `sales_summary_matview`
- 📌 **Query:** Aggregate `orders` by `region`
---
## 🔄 Refreshing the Materialized View

Since the data is “snapshotted,” you must refresh to pick up changes in `orders`:
```sql
-- Non-concurrent refresh (locks the matview)
REFRESH MATERIALIZED VIEW sales_summary_matview;

-- Concurrent refresh (allows SELECTs during refresh)
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary_matview;
```
🔒 **CONCURRENTLY** requires a unique index on the matview.

---
## 🔍 Querying the Materialized View
```sql
SELECT * FROM sales_summary_matview;
```
#### Sample Output

| 🗺️ Region | 📦 Total Orders | 💰 Total Revenue |
| ---------- | --------------- | ---------------- |
| North 🧭   | 1250            | €75 000          |
| South 🌞   | 980             | €58 400          |
| East 🗼    | 1430            | €82 750          |
| West 🏔️   | 1110            | €63 200          |

---
## 🚀 Benefits & Caveats

### Pros
- ⚡ Lightning‑fast reads
- 📉 Reduces repeated computation
### Cons
- ⏳ Needs manual or scheduled refresh
- 🔄 Data can be stale until refreshed
---
## 🛠️ Complete Example

```sql
-- 1. Create base table
CREATE TABLE orders (
  id        SERIAL PRIMARY KEY,
  region    TEXT,
  amount    NUMERIC
);

-- 2. Insert some rows
INSERT INTO orders(region, amount) VALUES
  ('North',  100),
  ('South',  200),
  ('East',   150),
  ('West',   120),
  ('North',  250),
  ('South',  300);

-- 3. Create materialized view
CREATE MATERIALIZED VIEW sales_summary_matview AS
SELECT region, COUNT(*) AS total_orders, SUM(amount) AS total_revenue
FROM orders
GROUP BY region;

-- 4. Query it
SELECT * FROM sales_summary_matview;
```
#### After step 4 you might see

|🗺️ Region|📦 Total Orders|💰 Total Revenue|
|---|---|---|
|North 🧭|2|€350|
|South 🌞|2|€500|
|East 🗼|1|€150|
|West 🏔️|1|€120|

---
## 🗑️ Dropping a Materialized View

```sql
DROP MATERIALIZED VIEW sales_summary_matview;
```
---
Here are three common ways to keep your materialized view fresh on a schedule

## 📅 Scheduling
### 🕒 1. Using the OS Cron Scheduler

The simplest, built‑in way on Linux/Unix:
#### 1. Create a shell script `refresh_mv.sh`
```bash
#!/bin/bash
psql -d mydatabase -c "REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary_matview;"
```
- 🔑 Make sure `sales_summary_matview` has a **unique index** if you’re using `CONCURRENTLY`.
- 🔒 Store credentials securely (e.g. in `.pgpass`).
#### 2. Make it executable

```sql
chmod +x /path/to/refresh_mv.sh
```
#### 3. Edit your crontab:

```bash
crontab -e
```
Add a line to run daily at 2 AM:
```config
0 2 * * * /path/to/refresh_mv.sh >> /var/log/refresh_mv.log 2>&1
```
- 📅 `0 2 * * *` → every day at 02:00
- 📜 Logs go to `/var/log/refresh_mv.log`
---
### 📅 2. Using PostgreSQL’s pg_cron Extension

If you’d rather schedule jobs **inside** Postgres:
#### 1. **Enable pg_cron** in your `postgresql.conf`:
```ini
shared_preload_libraries = 'pg_cron'
```
#### 2. **Restart** Postgres, then in `psql` run:
```sql
CREATE EXTENSION pg_cron;
```
#### 3. Schedule the refresh:
```sql
SELECT cron.schedule(
  'refresh_sales_mv',                     -- job name
  '0 3 * * *',                            -- at 03:00 every day
  $$REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary_matview;$$
);
```
- 🆔 Job “refresh_sales_mv” will fire daily at 03:00.
- ✅ You can list with: `SELECT * FROM cron.job;`

---

### 🐘 3. Using pgAgent (via pgAdmin)

A more full‑featured job scheduler if you already use **pgAdmin**:

#### 1. **Install pgAgent** (package depends on your OS).
#### 2. **In pgAdmin** → create a new **Job**:
- **Steps** → SQL step:
```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary_matview;
```
 
 - **Schedule** → set frequency (e.g. daily at 04:00).
#### 3. **Enable** and **start** the pgAgent service/daemon.

---
### ⚙️ Tips & Caveats

- 🔄 **CONCURRENTLY** lets people keep querying the view during refresh—but **needs** a unique index on all rows.
- ⏱️ Pick a schedule that balances freshness vs. load.
- 📈 Monitor run times (via logs or `cron.job_run_details` in pg_cron) to catch failures.
---
> With one of these in place, your materialized view will stay up‑to‑date automatically—no manual REFRESH required!