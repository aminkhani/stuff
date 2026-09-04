# 🧩 Quick Reference

> [!tldr] TL;DR
> **Normalization** = split data into related tables to **eliminate duplication** and keep it consistent. ✂️
> **Denormalization** = intentionally **duplicate data** to make reads faster, at the cost of write complexity. 📦
> Most real apps are **mostly normalized with a few deliberate denormalized shortcuts.**

---

## 🤔 The core problem normalization solves

Imagine one big flat table:

| order_id | customer_name | customer_email | product | price |
|---|---|---|---|---|
| 1 | John Doe | john@x.com | Laptop | 999 |
| 2 | John Doe | john@x.com | Mouse | 20 |
| 3 | John Doe | john@x.com | Keyboard | 50 |

Problems:
- 🔁 **Update anomaly** — John changes his email → must update **every row**, easy to miss one → inconsistent data
- 🗑️ **Delete anomaly** — delete his only order → you lose his customer info entirely
- ➕ **Insert anomaly** — can't add a new customer without an order (no order_id to attach them to)

**Normalization fixes this** by splitting into `customers` and `orders` tables, linked by a foreign key.

---

## 🧬 Normal Forms — what you MUST know

| Form | Rule | Fixes |
|---|---|---|
| **1NF** | Each column holds **atomic** (single) values — no lists/arrays in a cell | "Multi-valued" columns like `"tags: a,b,c"` |
| **2NF** | 1NF + every non-key column depends on the **whole** primary key (not part of it) | Partial dependency (relevant mainly for composite keys) |
| **3NF** | 2NF + no column depends on another **non-key** column (only on the key) | Transitive dependency, e.g. storing `city` and `zip_code` where `city` is derivable from `zip_code` |

> [!example] 3NF example
> ❌ `orders(id, customer_id, customer_city)` — `customer_city` depends on `customer_id`, not on `orders.id` → violates 3NF
> ✅ Move `city` into the `customers` table instead.

> [!tip] Practical rule
> In real backend work you rarely formally prove "is this 3NF?" You just apply the intuition: **"does this column truly belong to this table, or does it belong to a related one?"**

---

## 📦 Denormalization — doing it on purpose

Sometimes normalized data means **too many JOINs** for a hot-path query. Denormalization intentionally duplicates data to avoid that.

### Common denormalization techniques
- 🧮 **Cached aggregate** — store `order_count` on the `Customer` model instead of `COUNT()`-ing orders every time
- 📋 **Duplicated field** — store `product_name` directly on `OrderItem` (so it's preserved even if the product is renamed/deleted later — also fixes a *real* business need, not just performance!)
- 🗂️ **Flattened JSON blob** — store a whole nested structure in one JSONB column instead of 5 related tables
- 🏗️ **Read-optimized copy** — a separate reporting table/materialized view built from normalized source tables

---

## ⚖️ Side-by-side comparison

| | 🧩 Normalization | 📦 Denormalization |
|---|---|---|
| **Goal** | Eliminate duplicate/redundant data | Optimize for fast reads |
| **Writes** | Simple, one place to update | Complex — must update all copies |
| **Reads** | Slower (needs JOINs) | Fast (data already together) |
| **Data integrity** | High — single source of truth | Risk of **inconsistent copies** |
| **Storage** | Efficient (no duplication) | More storage used |
| **Best for** | Transactional systems (OLTP), correctness-critical data | Read-heavy systems, reporting, analytics, caching layers |

---

## 🐍 Django — how this shows up in practice

### Normalized (default Django way)
```python
class Customer(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField()

class Order(models.Model):
    customer = models.ForeignKey(Customer, on_delete=models.CASCADE)
    # customer_name/email NOT stored here — fetched via customer.name
```

### Denormalized — duplicated snapshot field
```python
class OrderItem(models.Model):
    order = models.ForeignKey(Order, on_delete=models.CASCADE)
    product = models.ForeignKey(Product, on_delete=models.SET_NULL, null=True)
    product_name = models.CharField(max_length=200)  # 👈 snapshot, survives product rename/delete
    price_at_purchase = models.DecimalField(max_digits=10, decimal_places=2)  # 👈 price can change later!
```
> This is actually a **best practice**, not a hack — an order should show what was purchased *at that time*, not today's price/name.

### Denormalized — cached counter (needs care!)
```python
class Customer(models.Model):
    name = models.CharField(max_length=100)
    order_count = models.PositiveIntegerField(default=0)  # 👈 denormalized cache

# Must keep in sync manually, e.g. via signals:
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=Order)
def update_order_count(sender, instance, created, **kwargs):
    if created:
        instance.customer.order_count = F('order_count') + 1
        instance.customer.save(update_fields=['order_count'])
```
> [!warning]
> Every denormalized field is a **new source of truth to keep in sync**. If a signal fails silently, or a bulk update skips signals (`bulk_update`, raw SQL), the counter **drifts out of sync**. Always ask: *"can I just annotate/aggregate instead?"*

### Often a better alternative: annotate instead of denormalize
```python
from django.db.models import Count
Customer.objects.annotate(order_count=Count('order'))
```
Computed on read, always accurate, no sync bugs — costs a bit more query time. Use this **first**, denormalize only when profiling proves it's needed.

---

## 🔐 Security-relevant notes

- Denormalized/duplicated PII (e.g. email copied into 5 tables) means **more places to secure, redact, and delete** — GDPR "right to be forgotten" gets harder the more you've duplicated personal data.
- Snapshot fields (like `product_name` at time of order) are **good for audit trails** — they preserve historical truth even if the source record is later altered or deleted maliciously.
- Cached/denormalized permission flags (e.g. `is_admin` copied onto a session or JWT) can become **stale** — if permissions are revoked, a denormalized copy might still grant access until it's refreshed. Prefer checking the source of truth for sensitive authorization decisions.
- Over-normalized permission logic spread across many JOINs can be **slow enough to encourage shortcuts** (e.g. skipping checks) — balance correctness with query performance carefully in auth-critical paths.

---

## 🧭 Decision cheat sheet

```
Is this data changing often and needs one accurate source of truth?
├── YES → normalize it
└── NO
    │
    Is a query on this data slow due to expensive JOINs/aggregates,
    AND you've confirmed it with profiling (not a guess)?
    ├── YES → consider denormalizing (or annotate/cache instead)
    └── NO → keep it normalized, don't optimize prematurely

Does the record need to preserve a HISTORICAL snapshot
(price/name at time of purchase, audit trail)?
├── YES → denormalize deliberately (this is correct design, not a hack)
└── NO → reference the live related table
```

---

## 📚 Quick glossary

| Term | Meaning |
|---|---|
| Normalization | Splitting data to remove redundancy |
| Denormalization | Duplicating data intentionally for read performance |
| Normal Form (1NF/2NF/3NF) | Formal rules describing "how normalized" a schema is |
| Update/Delete/Insert anomaly | Bugs caused by redundant, unnormalized data |
| Materialized view | A denormalized, pre-computed query result stored as a table |
| Snapshot field | Denormalized copy preserving data "as it was" at a point in time |
| Single source of truth | The one place data should be considered authoritative |

> [!tip] Rule of thumb
> **Normalize by default. Denormalize deliberately, with a reason, and track what falls out of sync.**
> 