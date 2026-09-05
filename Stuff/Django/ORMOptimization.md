```table-of-contents
title: **Table of Contents**
style: nestedList # TOC style (nestedList|nestedOrderedList|inlineFirstLevel)
minLevel: 0 # Include headings from the specified level
maxLevel: 0 # Include headings up to the specified level
includeLinks: true # Make headings clickable
hideWhenEmpty: false # Hide TOC if no headings are found
debugInConsole: false # Print debug info in Obsidian console
```
---
## 🧭 Three questions, in order

Every ORM performance problem is one of three things, and they take different fixes:

| Question | Symptom | Fix lives in |
| --- | --- | --- |
| **How many queries?** | 1 + N per page; latency scales with row count | the queryset: `select_related`, `prefetch_related`, `annotate` |
| **How much data per query?** | few queries, huge result sets, memory spikes | `only()`, `values()`, `iterator()`, pagination |
| **How fast is one query?** | a single statement dominating the request | indexes and the query plan, not the ORM |

Django's ORM is honest about what it does and quiet about *when*. Nothing in `order.customer.name` says "network round trip", and that ambiguity is the source of nearly every regression in this note.

> [!tldr] TL;DR
> Count queries before tuning them — `assertNumQueries` in tests, the toolbar locally. `select_related` for forward FK, `prefetch_related` for reverse FK and M2M, and any `.filter()` on a prefetched relation throws the prefetch away. Aggregate in SQL with `annotate`, never in a Python loop. Use `update()` with `F()` for read-modify-write instead of `save()`. When the query count is already minimal and it is still slow, the answer is in the **plan**, not in the ORM.

---
## 💤 Laziness, evaluation and the result cache

A `QuerySet` is a query *description*. It reaches the database only when something needs rows:

| Triggers a query | Does not |
| --- | --- |
| iteration, `list()`, `bool()`, `len()` | `filter()`, `exclude()`, `order_by()`, `annotate()` |
| slicing **with** a step (`qs[::2]`) | `qs[5:10]` — that is just `LIMIT/OFFSET` |
| `repr()`, which is why the shell lies to you | binding the queryset to a name |
| `count()`, `exists()`, `aggregate()`, `first()`, `latest()` | `only()`, `defer()`, `select_related()` |
| pickling, template rendering, serialization | `qs.query` — inspect the SQL for free |

Once evaluated, a queryset holds a **result cache** on that instance. Reusing the object is free; re-deriving it is not:

```python
qs = Order.objects.filter(status="paid")

if qs:                      # 1) query runs, result cache filled
    for o in qs:            # 2) free — iterates the cache
        ...
total = len(qs)             # 3) free — len() of the cache

# ❌ two queries: COUNT(*) does not fill the cache
if qs.count():
    rows = list(qs)

# ❌ a new queryset each time, so a new query each time
Order.objects.filter(status="paid").count()
Order.objects.filter(status="paid").first()
```

| Want to know | Use | Why |
| --- | --- | --- |
| "are there any?" | `qs.exists()` | `SELECT 1 … LIMIT 1`; stops at the first row |
| "how many?" | `qs.count()` | `SELECT COUNT(*)`; scans whatever the filter matches |
| "any, and I need the rows anyway" | `if qs:` then iterate | one query, cache reused |
| "how many, and I already have the rows" | `len(qs)` | no query at all |

> [!WARNING]
> `exists()` on an **already evaluated** queryset just checks the cache, but `qs.exists()` *before* iterating is a round trip you did not need. The rule is directional: cheap check first if you might not need the rows, single fetch if you certainly will.

---
## 🔢 The N+1 problem, and the log that proves it

```python
# views.py
orders = Order.objects.filter(status="paid")[:50]
```

```html
<!-- template -->
{% for order in orders %}
  {{ order.customer.name }}      <!-- 1 query per order -->
  {{ order.items.count }}        <!-- 1 more per order -->
{% endfor %}
```

```sql
-- what the database actually sees: 1 + 50 + 50
SELECT ... FROM orders WHERE status = 'paid' LIMIT 50;
SELECT ... FROM customers WHERE id = 1;
SELECT ... FROM customers WHERE id = 2;   -- ... 48 more
SELECT COUNT(*) FROM items WHERE order_id = 1;
SELECT COUNT(*) FROM items WHERE order_id = 2;   -- ... 48 more
```

Each of those is a round trip plus a parse plus a plan. At 0.4 ms apiece, 101 queries is 40 ms of pure overhead for work the database could have done in one statement. The identical trap in an API lives in serializer fields instead of template tags, with the same cause and the same fix — [DRF Internals](./DRF/DRFInternals.md) shows what it looks like there.

```python
orders = (
    Order.objects.filter(status="paid")
    .select_related("customer")                  # JOIN: no extra query
    .annotate(item_count=Count("items"))         # aggregate: no extra query
)[:50]                                           # one query, total
```

> [!IMPORTANT]
> N+1 is invisible in development. Twenty seeded rows against a database on `localhost` is 21 queries at 0.1 ms — nothing. Fifty rows per page against a managed database 1.5 ms away is 75 ms, and it scales with page size, so the endpoint degrades exactly as the product succeeds.

---
## 🔗 `select_related` versus `prefetch_related`

| | `select_related` | `prefetch_related` |
| --- | --- | --- |
| Mechanism | one `JOIN`, one query | a second query, joined in Python |
| Works on | forward `ForeignKey`, `OneToOneField` | reverse FK, `ManyToManyField`, generic relations — and forward FK too |
| Cost model | wider rows; parent columns duplicated per joined row | one extra query per relation, plus memory for the map |
| Depth | follows `__` chains; `depth=N` follows *everything* N levels down | nest with `__`, or pass a `Prefetch` object |
| Breaks when | the join gets wide enough that the planner mis-estimates | you re-filter the relation per object |
| Null relations | needs a `LEFT OUTER JOIN`, automatic for nullable FKs | unaffected |

```python
# forward FK -> JOIN
Order.objects.select_related("customer", "customer__country")

# reverse FK / M2M -> a second query
Order.objects.prefetch_related("items", "tags")

# both, plus a filtered and column-pruned prefetch
Order.objects.select_related("customer").prefetch_related(
    Prefetch(
        "items",
        queryset=Item.objects.filter(cancelled=False).only("id", "sku", "order_id"),
        to_attr="active_items",        # a plain list attribute, not a manager
    ),
    "items__warehouse",                # nested prefetch through the relation
)
```

`Prefetch` with `to_attr` is what makes a *filtered* relation work at all. Without it, a filter inside the loop builds a new queryset, and a new queryset is a new query:

```python
for order in Order.objects.prefetch_related("items"):
    order.items.all()                              # ✅ served from the cache
    order.items.filter(cancelled=False)            # ❌ new query, once per order
    order.items.count()                            # ❌ new query, once per order
    len(order.items.all())                         # ✅ counts the cache
    len(order.active_items)                        # ✅ when prefetched with to_attr
```

> [!NOTE]
> `only()` inside a `Prefetch` queryset must include the **join key** (`order_id` above). Leave it out and every object re-queries its own row to find it, turning an optimization into a fresh N+1.

---
## ✂️ Fetching fewer columns

| Tool | Returns | Watch out |
| --- | --- | --- |
| `only("a", "b")` | model instances with those fields loaded | touching any other field re-queries **that row** |
| `defer("blob")` | instances with everything except those | the same trap, inverted; the PK is always loaded |
| `values("a", "b")` | dicts | no model instances, so no methods and no `save()` |
| `values_list("a", flat=True)` | a flat list of one column | the cheapest way to get a list of ids |
| `iterator(chunk_size=…)` | a generator, no result cache | prefetching is ignored unless you pass `chunk_size` on Django ≥ 4.1 |
| `count()` / `aggregate()` | numbers | the database does the work and transfers almost nothing |

```python
# ❌ one extra query PER ROW: `notes` was deferred, then read
for o in Order.objects.only("id", "reference"):
    print(o.notes)                    # SELECT notes FROM orders WHERE id = ...

# ✅ if you need it, load it
for o in Order.objects.only("id", "reference", "notes"):
    print(o.notes)

# ✅ read-only aggregation: no model instances built at all
Order.objects.filter(status="paid").values("customer_id").annotate(n=Count("id"))

# ✅ a million rows without a million objects in memory
for oid in Order.objects.values_list("id", flat=True).iterator(chunk_size=2000):
    ...
```

`iterator()` trades the result cache for constant memory: rows stream from a server-side cursor instead of materialising at once, which makes it right for exports and backfills and wrong for anything you iterate twice. Two operational details bite. Server-side cursors are **incompatible with a transaction-pooling connection pooler** such as PgBouncer in `transaction` mode, where the cursor vanishes between statements — that is what `DISABLE_SERVER_SIDE_CURSORS = True` exists for, at the cost of buffering client-side. And a long iteration pins one connection for its whole duration, which combines badly with a generous `CONN_MAX_AGE` and a tight `max_connections`.

---
## 📊 Aggregation without lying to yourself

`annotate` adds a column per row; `aggregate` collapses the whole queryset into one dict. Both push arithmetic into SQL, which is the entire point.

```python
# ❌ row multiplication: two multi-valued joins, so each Count is inflated by the other
Order.objects.annotate(items=Count("items"), tags=Count("tags"))

# ✅ correct, at the cost of a sort or hash per aggregate
Order.objects.annotate(
    items=Count("items", distinct=True),
    tags=Count("tags", distinct=True),
)

# ✅ usually better on large tables: no join multiplication to undo
Order.objects.annotate(
    items=Subquery(
        Item.objects.filter(order=OuterRef("pk"))
        .values("order").annotate(n=Count("id")).values("n")
    )
)
```

Row multiplication is the bug that ships, because the numbers stay plausible while being wrong. `distinct=True` fixes it correctly but sorts; a `Subquery` keeps each aggregate independent and tends to plan better past a few hundred thousand rows.

| Instead of | Use | Because |
| --- | --- | --- |
| `order.items.exists()` per row | `annotate(has=Exists(Item.objects.filter(order=OuterRef("pk"))))` | one query, and the planner stops at the first matching row |
| `Count(...) > 0` | `Exists(...)` | counting everything to learn "at least one" is wasted I/O |
| a Python loop to find the latest child | `Subquery(Item.objects.filter(order=OuterRef("pk")).order_by("-created_at").values("id")[:1])` | one query; the `[:1]` is what makes it a scalar subquery |
| `filter()` **after** `annotate()` | `filter()` **before** `annotate()`, when the filter narrows base rows | filtering after becomes `HAVING` on the aggregate; filtering before shrinks what is aggregated |

Filter position is a semantic difference, not only a performance one. `.filter(items__cancelled=False).annotate(n=Count("items"))` counts only the non-cancelled items, while `.annotate(n=Count("items")).filter(items__cancelled=False)` adds a *second* join and counts something else. When in doubt write `Count("items", filter=Q(items__cancelled=False))` and say exactly what you mean.

---
## ✍️ Writes — batches, races and `F()`

```python
# ❌ read-modify-write: two queries and a lost update under concurrency
p = Product.objects.get(pk=1)
p.stock -= 1                        # arithmetic on a value read a moment ago
p.save()                            # UPDATE ... SET stock = 41

# ✅ one statement, computed by the database, no race
Product.objects.filter(pk=1, stock__gte=1).update(stock=F("stock") - 1)
```

`F()` moves the arithmetic into SQL, so two concurrent decrements land on 40 rather than 41. Folding `stock__gte=1` into the filter turns check-then-act into a single atomic statement whose return value — the number of rows updated — tells you whether it applied. When the invariant spans several statements you need a real transaction and often a lock: `select_for_update()` holds row locks for the enclosing `atomic()` block, and the isolation levels and lock-ordering rules behind that are in [Database Transactions](../Database/Database%20Transactions.md).

| Operation | Queries | Signals | `auto_now` | Notes |
| --- | --- | --- | --- | --- |
| `obj.save()` | 1 (+1 `SELECT` when the PK's existence is unknown) | yes | yes | writes every field unless you pass `update_fields=` |
| `qs.update(...)` | 1 | **no** | **no** | set-based: no `save()` override, no `post_save` |
| `bulk_create(objs, batch_size=…)` | 1 per batch | **no** | field defaults only | PKs come back on PostgreSQL, not on every backend |
| `bulk_update(objs, ["f"], batch_size=…)` | 1 per batch, as a large `CASE` | **no** | **no** | needs PKs; huge `CASE` statements get slow, so tune `batch_size` |
| `get_or_create` | 1–3 | on create | yes | races without a unique constraint |
| `update_or_create` | 2–3 | yes | yes | same race; wrap it in `atomic()` |

```python
# get_or_create is SELECT-then-INSERT, so two callers can both miss and both insert.
# The unique constraint is what makes it safe: the ORM catches IntegrityError and
# re-SELECTs. Without the constraint you silently accumulate duplicate rows.
class Membership(models.Model):
    class Meta:
        constraints = [UniqueConstraint(fields=["user", "team"], name="uniq_member")]
```

> [!WARNING]
> `update()`, `bulk_create()` and `bulk_update()` bypass `save()`, the `pre_save`/`post_save` signals and `auto_now`/`auto_now_add`. Anything you rely on in a `save()` override — denormalised counters, search vectors, audit rows, cache invalidation — silently stops happening. Either move that logic into a database constraint or trigger, or do the extra work explicitly after the batch.

---
## 📖 Pagination that stays fast

`LIMIT 20 OFFSET 200000` does not mean "skip 200 000 rows cheaply". The database produces and discards every one of them first, so page 10 000 costs a thousand times page 1 for the same query.

```sql
-- ❌ cost grows linearly with the page number
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 200000;

-- ✅ keyset: the index seeks straight to the position, constant cost
SELECT * FROM orders
WHERE (created_at, id) < ('2026-04-02 10:15:00', 918273)   -- cursor from the last row
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

```python
# the same idea through the ORM
qs = Order.objects.filter(
    Q(created_at__lt=last_created)
    | Q(created_at=last_created, id__lt=last_id)      # tie-break on the PK
).order_by("-created_at", "-id")[:20]
```

Keyset pagination needs an index on the exact tuple you order by and a **unique** tail, so ties cannot make rows repeat or disappear between pages. You give up "jump to page 500", which no real user wants and every scraper does. `COUNT(*)` for a total page count is the other hidden cost — it scans the whole filtered set on every request, so cache it, approximate it from table statistics, or drop it from the UI. Tables where this matters are the ones that only ever grow; archival and rollup strategies for those are in [Managing Ever-Growing Tables](../Database/ManagingEver%E2%80%91GrowingTables.md), and once a single table dominates the schema, [Partitioning](../Database/Partitioning.md) changes which plans are available at all.

---
## 🧮 Indexes, declared from Django

```python
class Order(models.Model):
    organization = models.ForeignKey(Organization, on_delete=models.CASCADE)  # indexed
    status = models.CharField(max_length=16, db_index=True)    # single-column index
    reference = models.CharField(max_length=32)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            # composite: column order matters — equality first, then the sort column
            models.Index(fields=["organization", "-created_at"], name="org_recent_idx"),
            # partial: only the rows anyone queries, so a fraction of the size
            models.Index(fields=["created_at"], name="open_orders_idx",
                         condition=Q(status="open")),
            # functional: makes a case-insensitive lookup indexable
            models.Index(Lower("reference"), name="ref_lower_idx"),
        ]
```

| Declaration | Produces | Use for |
| --- | --- | --- |
| `db_index=True` | one B-tree on one column | a single hot equality or range filter |
| `Meta.indexes` with several fields | one composite index | `WHERE a = ? ORDER BY b`; leftmost-prefix rules apply |
| `Index(condition=Q(...))` | a partial index | a skewed column where you only query one slice |
| `Index(Lower("f"))` | a functional index | `filter(f__iexact=…)`, which no plain index can serve |
| `UniqueConstraint` | a unique index | correctness first, and the index comes free |
| `GinIndex`, `BTreeGinIndex` | PostgreSQL-specific structures | `JSONField` containment, trigram and full-text search |

A `ForeignKey` is indexed automatically and an M2M through-table indexes both sides. What Django cannot do is make an index *usable*: a function applied to the column (`WHERE DATE(created_at) = …`), a leading wildcard (`LIKE '%x'`), an `OR` across unrelated columns, or a type mismatch between column and parameter all force a scan no matter what you declared. Which structure wins for which access pattern is in [Database Indexing](../Database/Database%20Indexing.md); whether *your* query uses one is only answerable from the plan.

---
## 📏 Measuring instead of guessing

| Tool | Answers | Caveat |
| --- | --- | --- |
| `django-debug-toolbar` | how many queries this page ran, duplicates highlighted | dev only; it distorts timings and never ships |
| `connection.queries` | the SQL and its duration | **only populated when `DEBUG=True`**, and it grows unbounded |
| `CaptureQueriesContext` | the SQL for one block, with `DEBUG` either way | the honest programmatic version of the above |
| `assertNumQueries(n)` | a hard regression guard | pin it on list views and serializers |
| `qs.explain(analyze=True)` | the real plan with real row counts | `ANALYZE` **executes** the statement — never on a write outside a rollback |
| `log_min_duration_statement` | which statements are slow in production | the only measurement made against real data and a real cache |
| `pg_stat_statements` | aggregate cost per normalised query | finds the cheap query that runs 40 000 times |

```python
from django.db import connection
from django.test.utils import CaptureQueriesContext

with CaptureQueriesContext(connection) as ctx:
    render_dashboard(user)

print(len(ctx.captured_queries))                  # the number that matters
for q in ctx.captured_queries:
    print(f"{float(q['time']) * 1000:6.1f} ms  {q['sql'][:120]}")
```

```python
print(Order.objects.filter(status="open").explain(analyze=True, buffers=True))
```

A query *count* is a code review problem; a query *duration* is a plan problem, and conflating them wastes days. A sequential scan over 500 rows is fine, the same plan over 50 million is an outage, and only `EXPLAIN (ANALYZE, BUFFERS)` against production-shaped data tells you which one you have. Estimated versus actual rows differing by orders of magnitude is the highest-signal line in any plan — [Query Execution Plans](../Database/Query%20Execution%20Plans.md) covers how to read one.

> [!TIP]
> Optimize against production-*shaped* data, not merely production-sized data. Skew matters more than volume: a status column that is 98% `closed` makes the planner's choice for `status="open"` completely different from what a uniformly seeded dev database suggests.

---
## 🚪 When the answer is not the ORM

Some queries are already as good as they can be and still too slow. The fix is then architectural:

| Situation | Move to | Why |
| --- | --- | --- |
| The same expensive read, over and over | a cache keyed on the inputs — see [Caching](../SoftwareDesign/Caching.md) | the cheapest query is the one you never send |
| An expensive write path inside the request | a background task — see [Celery](./Celery.md) | nobody needs to watch a spinner while counters rebuild |
| A heavy aggregation that tolerates staleness | a materialized view on a refresh schedule — see [Materialized View](../Database/MaterializedView.md) | pay once per refresh instead of once per request |
| Something the ORM cannot express | `raw()` or `connection.cursor()` with **parameters** | window functions, CTEs, `DISTINCT ON` — real SQL, safely bound |
| Reporting competing with transactional load | a read replica or a separate warehouse | analytics and OLTP want different indexes and different plans |
| Many independent slow calls in one request | concurrency, carefully — see [Futures](../Python/Futures.md) | a thread per query is a connection per thread; it hides an N+1 rather than fixing it and drains the pool |

```python
# raw SQL is fine; string interpolation is not
Order.objects.raw(
    "SELECT id, reference FROM orders WHERE organization_id = %s AND total > %s",
    [org_id, threshold],                  # ✅ bound parameters, never an f-string
)
```

---
## 🚨 Common mistakes

| Mistake | Why it hurts | Fix |
| --- | --- | --- |
| Touching a relation inside a loop | 1 + N queries; latency scales with page size | `select_related` / `prefetch_related` / `annotate` |
| `.filter()` or `.count()` on a prefetched relation | the prefetch is discarded and the query runs per object | `Prefetch(queryset=…, to_attr=…)`, then read the attribute |
| `prefetch_related` for a forward FK | a second query where a `JOIN` would do | `select_related` |
| `select_related` across many tables | a wide join the planner mis-estimates | split it and prefetch the long tail |
| `qs.count()` then `list(qs)` | two queries for one answer | `if qs:` and reuse the result cache |
| Re-deriving `Model.objects.filter(...)` | a fresh queryset means a fresh query, every time | bind it to a name and reuse it |
| `only()` then reading a deferred field | one extra query **per row** | include the field, or switch to `values()` |
| `Count()` over two multi-valued joins | row multiplication — plausible, wrong numbers | `distinct=True`, `filter=Q(...)`, or `Subquery` |
| `exists()` per row in a loop | N queries to compute N booleans | `annotate(Exists(… OuterRef("pk")))` |
| Read-modify-write through `save()` | lost updates under concurrency | `update(field=F("field") - 1)` in one statement |
| `bulk_create` where signals were doing work | counters, search vectors and audit rows quietly stop | do it explicitly, or push it into the database |
| `get_or_create` with no unique constraint | duplicate rows under concurrency | add a `UniqueConstraint` |
| `LIMIT/OFFSET` on deep pages | cost grows linearly with the page number | keyset pagination on an indexed tuple |
| Pagination with no unique tie-break | rows repeat or vanish between pages | append the primary key to the ordering |
| Adding indexes by intuition | write amplification with no read benefit | `EXPLAIN (ANALYZE, BUFFERS)` before and after |
| Optimizing against 50 dev rows | the plan flips at real volume and real skew | measure on production-shaped data |

---
## 🧠 Summary

| Area | Takeaway |
| --- | --- |
| Laziness | A queryset is a description; evaluation fills a result cache you should reuse, not rebuild |
| Counting | `exists()` to ask, `count()` to count, `len()` when you already hold the rows |
| N+1 | The default failure mode of every ORM: invisible locally, linear in page size |
| Joining | `select_related` for forward FK, `prefetch_related` for reverse FK and M2M, `Prefetch` when it needs filtering |
| Columns | `only`/`defer` help until you touch the deferred field; `values` and `iterator` for read-only bulk |
| Aggregation | Push it into SQL, watch for row multiplication across joins, prefer `Exists`/`Subquery` to per-row queries |
| Writes | `F()` with `update()` for atomic arithmetic, bulk helpers for volume — and both skip your signals |
| Pagination | Keyset on an indexed, unique-tailed tuple; `OFFSET` is a tax that grows with depth |
| Indexes | Declare them in `Meta`, order the columns deliberately, and confirm from the plan that they are used |
| Measuring | Query counts in tests, `EXPLAIN (ANALYZE)` for duration, slow-query logs in production |
| Escalation | Cache it, defer it to a task, materialize it, or write the SQL — with bound parameters |

---
## 📚 References

- [Django — database access optimization](https://docs.djangoproject.com/en/stable/topics/db/optimization/) · [QuerySet API reference](https://docs.djangoproject.com/en/stable/ref/models/querysets/)
- [Django — aggregation](https://docs.djangoproject.com/en/stable/topics/db/aggregation/) · [query expressions](https://docs.djangoproject.com/en/stable/ref/models/expressions/) · [conditional expressions](https://docs.djangoproject.com/en/stable/ref/models/conditional-expressions/)
- [Django — model index reference](https://docs.djangoproject.com/en/stable/ref/models/indexes/) · [constraints](https://docs.djangoproject.com/en/stable/ref/models/constraints/) · [transactions](https://docs.djangoproject.com/en/stable/topics/db/transactions/)
- [PostgreSQL — using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html) · [index types](https://www.postgresql.org/docs/current/indexes-types.html) · [pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)
- [Use The Index, Luke — no offset](https://use-the-index-luke.com/no-offset)
- [django-debug-toolbar documentation](https://django-debug-toolbar.readthedocs.io/en/latest/)
