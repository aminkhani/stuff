# 🔑 Quick Reference

> [!tldr] TL;DR
> An index is a **shortcut data structure** that lets the DB find rows **without scanning the whole table**.
> Speeds up **reads** 🚀, but costs extra space and slows down **writes** ✍️ (every insert/update must update the index too).

---

## 🤔 Why indexes matter

Without an index, `WHERE email = 'x@y.com'` means the DB checks **every single row** — a **full table scan**. On 10 rows, fine. On 10 million rows, painfully slow.

An index is like a **book's index page** — instead of reading every page to find "Kafka," you jump straight to page 42. 📖

---

## 🧬 How it works (conceptually)

Most indexes use a **B-Tree** (balanced tree) structure:

```
             [50]
           /      \
       [25]        [75]
      /    \       /    \
   [10]  [30]   [60]   [90]
```

- Sorted structure → DB can binary-search instead of scanning linearly
- Lookup goes from **O(n)** → **O(log n)**
- The index stores the column value + a **pointer** to the actual row on disk

---

## 🔑 Index types you MUST know

| Type | Best for | Notes |
|---|---|---|
| 🌳 **B-Tree** (default) | Equality (`=`) and range queries (`<`, `>`, `BETWEEN`) | Default index type in Postgres/MySQL — covers 90% of cases |
| #️⃣ **Hash** | Equality only (`=`) | Faster than B-Tree for exact match, useless for ranges |
| 📚 **GIN** (Generalized Inverted Index) | Arrays, JSONB, full-text search | Postgres-specific, great for `@>`, `?`, text search |
| 🗺️ **GiST** | Geospatial data, ranges, nearest-neighbor | Used by PostGIS |
| 🔀 **Composite (multi-column)** | Queries filtering on multiple columns together | Column **order matters** (see below) |
| 🎯 **Partial** | Queries that always filter a subset (e.g. `WHERE active = true`) | Smaller, faster index, ignores irrelevant rows |
| 🔗 **Unique** | Enforcing uniqueness + speeding lookup | Automatically created for `PRIMARY KEY` / `UNIQUE` |
| 📇 **Covering index** | Query that only needs indexed columns | DB never touches the actual table row ("index-only scan") |

---

## ⚠️ The tradeoff — indexes aren't free

| ✅ Pros | ❌ Cons |
|---|---|
| Much faster `SELECT` / `WHERE` / `ORDER BY` / `JOIN` | Slower `INSERT` / `UPDATE` / `DELETE` (index must update too) |
| Enforces uniqueness | Takes extra **disk space** |
| Speeds up sorting (`ORDER BY` on indexed column) | Too many indexes = diminishing returns + maintenance overhead |

> [!warning] Common mistake
> Indexing **every column** "just in case." This bloats writes and disk usage for indexes that are never used. Index based on **actual query patterns**, not guesses.

---

## 🧩 Composite index — column order matters!

```sql
CREATE INDEX idx_user_status ON orders (user_id, status);
```

This index helps:
- ✅ `WHERE user_id = 5` (uses leftmost column)
- ✅ `WHERE user_id = 5 AND status = 'paid'` (uses both)
- ❌ `WHERE status = 'paid'` (does NOT use this index — status isn't leftmost!)

> [!tip] Rule
> Put the **most selective / most commonly filtered-alone column first**. Think of it like a phone book sorted by (last name, first name) — you can't jump to "all Johns" efficiently.

---

## 🐍 Django — how you actually apply this

### Basic index
```python
class Order(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)  # FK auto-indexed
    status = models.CharField(max_length=20, db_index=True)   # simple index
```

### Composite index (Meta.indexes — preferred, modern way)
```python
class Order(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    status = models.CharField(max_length=20)

    class Meta:
        indexes = [
            models.Index(fields=['user', 'status']),
        ]
```

### Partial index (Postgres)
```python
class Meta:
    indexes = [
        models.Index(
            fields=['user'],
            condition=models.Q(status='active'),
            name='idx_active_orders',
        ),
    ]
```

### Unique constraint (also an index)
```python
class Meta:
    constraints = [
        models.UniqueConstraint(fields=['email'], name='unique_email'),
    ]
```

### 🔍 Finding what needs an index
```python
# In dev, use django-debug-toolbar or:
Order.objects.filter(status='paid').explain(analyze=True)
```
```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'paid';
```
Look for `Seq Scan` (bad, no index used) vs `Index Scan` / `Index Only Scan` (good). 🎯

---

## 🚫 When an index WON'T help

- Query uses a function on the column: `WHERE UPPER(email) = 'X'` (unless you make a **functional/expression index**)
- Column has **low cardinality** (e.g. `is_active` boolean on its own — only 2 possible values, scan might be faster anyway)
- `LIKE '%something%'` (leading wildcard) — B-Tree can't help; needs trigram (`pg_trgm`) or full-text search index
- Table is small — full scan is already fast, index adds overhead for nothing
- Query returns most of the table anyway (planner will often ignore the index and scan)

---

## 🔐 Security-relevant notes

- Indexes on **sensitive columns** (email, SSN-like data) speed up lookups but the index itself **also stores the raw value** — encrypting a column doesn't help if the index leaks patterns.
- **Timing attacks**: indexed vs non-indexed lookups have different response times — in rare cases this can leak whether a record exists (e.g., email enumeration via response time).
- Don't forget indexes on **foreign keys used in permission checks** (e.g., `tenant_id`, `owner_id`) — missing these can turn authorization queries into slow full scans, a potential DoS angle under load.
- Audit/log tables grow huge — index the columns used for **security audits** (timestamp, user_id, action) or investigations become painfully slow.

---

## 🧭 Decision cheat sheet

```
Is this column frequently used in WHERE, JOIN, or ORDER BY?
├── NO → don't index
└── YES
    │
    Is it low-cardinality (few unique values, e.g. boolean)?
    ├── YES → consider partial index instead of full index
    └── NO
        │
        Used with other columns together in queries?
        ├── YES → composite index (most selective column first)
        └── NO → simple single-column B-Tree index
```

---

## 📚 Quick glossary

| Term | Meaning |
|---|---|
| Index | Data structure that speeds up lookups |
| Full table scan | Reading every row — what indexes help you avoid |
| Cardinality | Number of unique values in a column (high = good for indexing) |
| Selectivity | How much an index narrows down results |
| Composite index | Index on multiple columns |
| Covering index | Index containing all columns a query needs (no table lookup needed) |
| `EXPLAIN ANALYZE` | Command to see how the DB actually executes a query |

> [!tip] Rule of thumb
> Index columns you **filter, join, or sort on frequently** — not columns you just display. Always verify with `EXPLAIN ANALYZE`, don't guess.
> 