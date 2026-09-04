# 🔗 Quick Reference

> [!tldr] TL;DR
> A **JOIN** combines rows from two (or more) tables based on a related column.
> This is the direct payoff of **normalization** — you split data apart, then JOIN it back together when you need it. 🧩➡️🔗

---

## 🤔 Why joins exist

You normalized your data (see [[Normalization-vs-Denormalization]]) — `customers` and `orders` are separate tables. But you need to answer: *"show me each order with the customer's name."* That requires **combining** rows from both tables → a JOIN.

```
customers                    orders
+----+-------+               +----+--------------+--------+
| id | name  |               | id | customer_id  | total  |
+----+-------+               +----+--------------+--------+
| 1  | John  |               | 1  | 1            | 100    |
| 2  | Amy   |               | 2  | 1            | 50     |
+----+-------+               | 3  | 3            | 20     |  ← customer_id 3 doesn't exist!
                              +----+--------------+--------+
```

---

## 🔑 Join types you MUST know

### 🟦 INNER JOIN — only matching rows
```sql
SELECT o.id, c.name, o.total
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id;
```
Returns orders 1 and 2 only. **Order 3 is excluded** (no matching customer).

| Keep in mind | |
|---|---|
| Default join type | `JOIN` alone means `INNER JOIN` |
| Rows without a match on **either side** are dropped | |

---

### 🟩 LEFT JOIN (LEFT OUTER JOIN) — all from left, matched or NULL from right
```sql
SELECT c.name, o.total
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id;
```
Returns **all customers**, even Amy who has no orders — `o.total` is `NULL` for her.

> [!example] Most common join in real apps
> "Show all users, and their profile picture **if they have one**" → LEFT JOIN.

---

### 🟨 RIGHT JOIN (RIGHT OUTER JOIN) — mirror of LEFT JOIN
```sql
SELECT c.name, o.total
FROM customers c
RIGHT JOIN orders o ON o.customer_id = c.id;
```
Returns **all orders**, even order 3 with the missing customer — `c.name` is `NULL`.
> Rarely used in practice — most people just flip the table order and use LEFT JOIN instead. Good to recognize, rarely write it yourself.

---

### 🟥 FULL OUTER JOIN — everything from both sides
```sql
SELECT c.name, o.total
FROM customers c
FULL OUTER JOIN orders o ON o.customer_id = c.id;
```
Returns all customers AND all orders, `NULL` filled in wherever there's no match on either side. (Not supported in MySQL — emulate with `UNION` of LEFT + RIGHT.)

---

### ⬛ CROSS JOIN — every combination (cartesian product)
```sql
SELECT c.name, p.product_name
FROM colors c
CROSS JOIN sizes p;
```
No `ON` condition — every row from table A paired with every row from table B. `5 colors × 3 sizes = 15 rows`. Used for generating combinations (e.g., product variants), rarely for typical queries — can accidentally explode row counts if used by mistake!

---

### 🔁 SELF JOIN — a table joined to itself
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```
Used for hierarchical data (employee → manager, category → parent category).

---

## 🖼️ Visual summary

```
INNER JOIN        LEFT JOIN         RIGHT JOIN        FULL OUTER JOIN
   A ∩ B            A + (A∩B)         B + (A∩B)          A ∪ B

  ⚪️⚫️              ⚪️⚫️               ⚪️⚫️                ⚪️⚫️
   ⬛️                🟢⬛️               ⬛️🟡                🟢⬛️🟡
  (only overlap)   (all of A)        (all of B)         (everything)
```

---

## ⚠️ The N+1 problem — the #1 join-related bug in Django

```python
# ❌ BAD — 1 query for orders, then N queries (one per order) for customer
orders = Order.objects.all()
for order in orders:
    print(order.customer.name)   # hits DB every loop iteration!
```

```python
# ✅ GOOD — select_related does a SQL JOIN, fetches everything in 1 query
orders = Order.objects.select_related('customer')
for order in orders:
    print(order.customer.name)   # no extra query
```

| Django ORM method | SQL equivalent | Use for |
|---|---|---|
| `select_related('field')` | `INNER/LEFT JOIN` | ForeignKey / OneToOne (single related object) |
| `prefetch_related('field')` | separate query + Python-side join | ManyToMany / reverse FK (multiple related objects) |

> [!warning] Rule of thumb
> **ForeignKey/OneToOne → `select_related`. ManyToMany/reverse FK → `prefetch_related`.** Getting this wrong is the most common Django performance bug.

---

## 🐍 Django ORM → JOIN mapping

```python
# INNER JOIN — only orders that have a customer
Order.objects.select_related('customer').filter(customer__isnull=False)

# LEFT JOIN — all orders, customer may be null
Order.objects.select_related('customer')  # LEFT JOIN by default for nullable FK

# Filtering across a join
Order.objects.filter(customer__name='John')  # JOIN + WHERE, one query

# Explicit annotate/aggregate across a join
from django.db.models import Count
Customer.objects.annotate(order_count=Count('order'))
```

> Django uses `LEFT JOIN` automatically when the FK is **nullable**, and `INNER JOIN` when it's required — this is often invisible to you, but shows up in `.query` or `EXPLAIN`.

---

## 🐌 Performance notes

- Joining on a **non-indexed** column = slow (see [[Database-Indexing]]) — always index FK columns (Django does this automatically for `ForeignKey`)
- Joining many large tables together = expensive — check `EXPLAIN ANALYZE` for `Nested Loop` vs `Hash Join` vs `Merge Join` strategies
- Too many joins in one query can hurt readability and performance — sometimes 2 simpler queries + app-side merge is faster than 1 giant join
- `prefetch_related` avoids row multiplication issues that can happen with `select_related` across ManyToMany relationships

---

## 🔐 Security-relevant notes

- **Tenant isolation bugs** often hide in joins — forgetting to filter by `tenant_id`/`owner_id` in a joined query can leak **cross-tenant data**.
- JOIN conditions built from **unsanitized user input** (raw SQL) are an injection risk — always use the ORM or parameterized queries, never string-format table/column names from user input.
- Overly permissive JOINs in reporting/admin queries can accidentally expose **more data than intended** (e.g., a LEFT JOIN pulling in soft-deleted or unauthorized records) — always double check `WHERE` scoping alongside the join.

---

## 🧭 Decision cheat sheet

```
Do you need only rows that exist in BOTH tables?
├── YES → INNER JOIN
└── NO
    │
    Do you need ALL rows from one table, with related data if it exists?
    ├── YES → LEFT JOIN (most common in real apps)
    └── NO
        │
        Do you need everything from both, matched or not?
        ├── YES → FULL OUTER JOIN
        └── NO → CROSS JOIN (all combinations) or SELF JOIN (hierarchy)
```

---

## 📚 Quick glossary

| Term | Meaning |
|---|---|
| Join | Combining rows from 2+ tables based on a related column |
| Join condition (`ON`) | The rule matching rows between tables |
| N+1 problem | 1 query + N extra queries in a loop — the #1 ORM performance bug |
| `select_related` | Django: SQL JOIN for single related objects (FK/O2O) |
| `prefetch_related` | Django: separate query + merge for multiple related objects (M2M/reverse FK) |
| Cartesian product | Every row of A paired with every row of B (CROSS JOIN) |

> [!tip] Rule of thumb
> Default to **LEFT JOIN** when unsure — it never *loses* rows. Use **INNER JOIN** only when you're sure every row *must* have a match. Always index your join columns.
> 