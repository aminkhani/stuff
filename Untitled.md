
# 🗺️ Query Execution Plans — Quick Reference

> [!tldr] TL;DR
> An **execution plan** is the DB's step-by-step strategy for running your query — which scans, which joins, in what order.
> `EXPLAIN` shows you the **plan**. `EXPLAIN ANALYZE` actually **runs it** and shows real timing. This is how you find *why* a query is slow. 🔍

---

## 🤔 Why this matters

You wrote a query. It's slow. **Guessing why is a waste of time.** The query planner already decided exactly how it will fetch your data — `EXPLAIN` just shows you that decision so you can fix the real bottleneck instead of guessing.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 5;
```

---

## 🧬 EXPLAIN vs EXPLAIN ANALYZE

| Command | What it does |
|---|---|
| `EXPLAIN` | Shows the **planned** strategy — estimated costs, no actual execution |
| `EXPLAIN ANALYZE` | **Actually runs** the query and shows real timing + row counts alongside the plan |
| `EXPLAIN ANALYZE, BUFFERS` (Postgres) | Adds cache hit/miss info — how much came from memory vs disk |

> [!warning]
> `EXPLAIN ANALYZE` **actually executes** the query — be careful running it on `DELETE`/`UPDATE` in production (wrap in a transaction and `ROLLBACK` if testing writes).

---

## 🔑 Key plan nodes you MUST recognize

| Node | Meaning | Good or bad? |
|---|---|---|
| 🐌 **Seq Scan** (Sequential Scan) | Reads every row in the table | ⚠️ Bad on large tables — usually means missing index |
| 🎯 **Index Scan** | Uses an index, then fetches matching rows from the table | ✅ Good |
| 🏆 **Index Only Scan** | Uses an index and **never touches the table** (all needed data is in the index) | ✅✅ Best — fastest possible |
| 🔁 **Nested Loop** | For each row in A, scan B looking for a match | OK for small datasets, ⚠️ bad for large ones |
| #️⃣ **Hash Join** | Builds a hash table from one side, probes it with the other | ✅ Good for large unsorted joins |
| 📶 **Merge Join** | Both sides sorted, then merged like a zipper | ✅ Good when data is already sorted/indexed |
| ↕️ **Sort** | Explicit sorting step (for `ORDER BY`, or before a Merge Join) | ⚠️ Watch for `Sort Method: external merge` (disk spill = slow) |
| 🎛️ **Aggregate** | Computing `COUNT`, `SUM`, `AVG`, etc. | Normal for aggregate queries |
| 🚧 **Filter** | Extra `WHERE` condition applied *after* a scan/join (not index-assisted) | ⚠️ If filtering out most rows, an index on that column could help |

---

## 📊 Reading a real plan

```
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 5;

Index Scan using idx_orders_customer_id on orders
  (cost=0.29..8.31 rows=3 width=64)
  (actual time=0.021..0.024 rows=3 loops=1)
  Index Cond: (customer_id = 5)
Planning Time: 0.084 ms
Execution Time: 0.041 ms
```

| Field | Meaning |
|---|---|
| `cost=0.29..8.31` | Estimated cost: startup..total (arbitrary units, not ms) |
| `rows=3` | **Estimated** rows the planner expects |
| `actual time=0.021..0.024` | **Real** time: first row..all rows (in ms) |
| `rows=3 loops=1` | **Actual** rows returned, and how many times this node ran |
| `Planning Time` | Time spent building the plan itself |
| `Execution Time` | Total actual execution time |

> [!tip] The #1 thing to check
> Compare **estimated rows** vs **actual rows**. A huge mismatch means the planner's statistics are stale (`ANALYZE tablename;` to refresh) or the query is written in a way the planner can't estimate well.

---

## 🚩 Red flags to look for

- 🐌 `Seq Scan` on a large table with a `WHERE` clause → missing index
- 🔁 `Nested Loop` with high `loops=` count on a big table → consider an index or restructuring the join
- 💽 `Sort Method: external merge Disk` → sort didn't fit in memory, spilled to disk → increase `work_mem` or reduce result set
- 📉 Estimated rows wildly different from actual rows → stale statistics, run `ANALYZE`
- 🔂 The same table scanned multiple times → maybe a subquery/CTE could be restructured
- ❌ Index exists but planner still does `Seq Scan` → could be low selectivity (planner decided a scan is actually cheaper!) — not always a bug

---

## 🐍 Django — how you actually use this

### Using `.explain()` directly on a QuerySet
```python
qs = Order.objects.filter(customer_id=5)
print(qs.explain())                     # basic plan
print(qs.explain(analyze=True))         # real execution + timing (Postgres/MySQL)
print(qs.explain(verbose=True, analyze=True))
```

### See the raw SQL Django generates (before even running EXPLAIN)
```python
print(qs.query)   # useful to sanity check JOINs/WHERE clauses match what you expect
```

### Common real-world Django scenario
```python
# Suspiciously slow endpoint? Check the plan:
Order.objects.filter(status='pending').select_related('customer').explain(analyze=True)
```
Look for `Seq Scan` on `status` → maybe add:
```python
class Meta:
    indexes = [models.Index(fields=['status'])]
```
Then re-run `.explain(analyze=True)` and confirm it switched to `Index Scan`. 🎯

### Tools to make this easier
- **django-debug-toolbar** — shows every query + a "Explain" button per query, right in the browser
- **django-silk** — profiling + query plan viewer, good for finding N+1s too

---

## 🔐 Security-relevant notes

- Execution plans can **leak schema/data-shape info** — never expose raw `EXPLAIN` output to end users or in public-facing error messages/API responses.
- A query that's cheap on a small table can become a **DoS vector** at scale (e.g., an unindexed search filter an attacker can spam) — check plans on filters exposed to user input, especially search/autocomplete endpoints.
- Watch for **unbounded queries** — `Seq Scan` combined with no `LIMIT` on user-controllable filters can be used to exhaust DB resources (a form of resource-exhaustion/DoS).
- Slow queries in **auth/permission-check code paths** are a subtle risk too — timing differences can sometimes leak information (see timing-attack note in [[Database-Indexing]]).

---

## 🧭 Decision cheat sheet

```
Is a specific query slow?
├── Run EXPLAIN ANALYZE first — don't guess
│
Seq Scan on a WHERE/JOIN column, large table?
├── YES → add an index, re-check plan
│
Estimated rows way off from actual rows?
├── YES → run ANALYZE tablename to refresh planner statistics
│
Sort spilling to disk (external merge)?
├── YES → reduce result set, add index to avoid sort, or tune work_mem
│
Plan looks fine but still slow?
├── Check Execution Time vs Planning Time — maybe it's app-side (N+1, serialization), not the DB
```

---

## 📚 Quick glossary

| Term | Meaning |
|---|---|
| Execution plan | The DB's chosen strategy for running a query |
| `EXPLAIN` | Shows the plan without running it |
| `EXPLAIN ANALYZE` | Runs the query and shows real timing |
| Seq Scan | Full table read, no index used |
| Index Scan | Uses an index to find rows |
| Cost | Planner's internal estimate (not real time) |
| Query planner/optimizer | The DB component that decides the execution strategy |
| Statistics | Planner's knowledge of data distribution, refreshed via `ANALYZE` |

> [!tip] Rule of thumb
> **Never optimize blind.** Run `EXPLAIN ANALYZE` before and after any indexing/query change to confirm it actually helped — plans can surprise you.
> 