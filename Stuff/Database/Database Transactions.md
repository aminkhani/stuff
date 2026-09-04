# 💳 Quick Reference

> [!tldr] TL;DR
> A **transaction** groups multiple database operations into ONE all-or-nothing unit.
> Either **everything succeeds** ✅ or **everything is rolled back** ❌ — no half-finished state.

---

## 🤔 Why transactions matter

Imagine a bank transfer: subtract $100 from Account A, add $100 to Account B.

```
1. UPDATE account_a SET balance = balance - 100
2. 💥 SERVER CRASHES HERE
3. UPDATE account_b SET balance = balance + 100   ← never runs
```

Without a transaction: **$100 vanished**. With a transaction: step 1 gets **rolled back** automatically. 🔒

---

## 🧬 ACID — the properties that define a transaction

| Letter | Property | Meaning |
|---|---|---|
| **A** | Atomicity | All operations succeed, or none do (all-or-nothing) |
| **C** | Consistency | DB moves from one valid state to another (constraints, FKs always hold) |
| **I** | Isolation | Concurrent transactions don't interfere with each other |
| **D** | Durability | Once committed, data survives crashes/power loss |

> [!example] Mnemonic
> **A**ll or nothing, **C**onstraints hold, **I**solated from others, **D**urable forever.

---

## 🔑 Core concepts you MUST know

- 🟢 **BEGIN / START TRANSACTION** — open a transaction block
- ✅ **COMMIT** — save all changes permanently
- ❌ **ROLLBACK** — undo all changes since BEGIN
- 📍 **SAVEPOINT** — a checkpoint inside a transaction you can roll back to *without* undoing the whole thing
- 🔒 **Locking** — DB locks rows/tables so two transactions don't corrupt the same data
- 🏁 **Isolation Level** — how much one transaction can "see" of another's uncommitted work
- ⚔️ **Deadlock** — two transactions wait on each other's locks forever → DB kills one

---

## 🎚️ Isolation levels (from loose → strict)

| Level | Prevents | Allows (problem) |
|---|---|---|
| **Read Uncommitted** | Nothing | 🚫 Dirty reads (see uncommitted data from others) |
| **Read Committed** | Dirty reads | 🚫 Non-repeatable reads (data changes between two reads in same tx) |
| **Repeatable Read** | Dirty + non-repeatable reads | 🚫 Phantom reads (new rows appear on re-query) |
| **Serializable** | Everything | Slowest — transactions run as if fully sequential |

> [!tip] Defaults
> **PostgreSQL** default = `Read Committed`
> **MySQL (InnoDB)** default = `Repeatable Read`
> You rarely need to change this — but **know it exists** when debugging weird concurrency bugs.

---

## ⚔️ Common concurrency problems

- 👻 **Dirty Read** — reading another transaction's uncommitted (possibly rolled-back) changes
- 🔁 **Non-repeatable Read** — same query, two different results, within one transaction
- 👻 **Phantom Read** — a re-run query returns new rows that weren't there before
- 🏃 **Race Condition** — two transactions read-modify-write the same row simultaneously, one overwrites the other
- ⚔️ **Deadlock** — Tx1 locks Row A waits for Row B, Tx2 locks Row B waits for Row A → stuck forever

---

## 🐍 Django transactions — what you'll actually use

### `atomic()` — the main tool
```python
from django.db import transaction

with transaction.atomic():
    order.save()
    payment.save()
    # if any exception is raised here, BOTH are rolled back
```

As a decorator:
```python
@transaction.atomic
def place_order(request):
    ...
```

### 🔒 `select_for_update()` — row locking to prevent race conditions
```python
with transaction.atomic():
    account = Account.objects.select_for_update().get(id=1)
    account.balance -= 100
    account.save()
```
> Locks the row until the transaction commits — another request trying to `select_for_update()` the same row **waits**.

### 📍 Savepoints (nested atomic blocks)
```python
with transaction.atomic():
    order.save()
    try:
        with transaction.atomic():  # creates a SAVEPOINT
            risky_operation()
    except SomeError:
        pass  # only the inner block rolls back, outer continues
```

### ⚙️ Other useful bits
- `transaction.on_commit(callback)` — run code (e.g. send email) **only after** commit succeeds — avoids side effects on rollback
- `TestCase` wraps each test in a transaction + rollback automatically (fast tests)
- `settings.py` → `ATOMIC_REQUESTS = True` wraps every view in a transaction (use carefully, hurts performance if overused)

---

## 🔐 Security-relevant notes

- Transactions prevent **inconsistent state exploitation** — e.g., a crash mid-payment leaving a user "paid" without a debit.
- Use `select_for_update()` for **balance/credit/inventory** updates to prevent **race condition exploits** (e.g., double-spending via concurrent requests).
- Never trust "check-then-act" without a lock: checking `if balance >= amount` then updating separately is a classic **TOCTOU (time-of-check to time-of-use)** vulnerability.
- Long-held locks = potential **DoS vector** — a poorly scoped transaction can lock rows and stall other requests.
- Rolled-back transactions should **not leak** partial data through logs, cache, or side effects (email already sent, etc.) — use `on_commit()`.

---

## 🧭 Decision cheat sheet

```
Are you doing multiple related writes that must succeed/fail together?
├── YES → wrap in transaction.atomic()
└── NO → single write, transaction not usually needed

Are multiple requests likely to update the SAME row concurrently
(balance, stock count, seat booking)?
├── YES → use select_for_update() inside atomic()
└── NO → plain atomic() is enough

Need a side effect (email, webhook) tied to a DB write?
├── YES → use transaction.on_commit()
└── NO → normal code is fine
```

---

## 📚 Quick glossary

| Term | Meaning |
|---|---|
| Transaction | Group of ops treated as one unit |
| Commit | Make changes permanent |
| Rollback | Undo changes |
| Savepoint | Partial checkpoint within a transaction |
| Isolation level | How much concurrent transactions can see of each other |
| Deadlock | Circular lock wait between transactions |
| Race condition | Concurrent read-modify-write causing lost updates |
| Idempotency | Safe to retry the same operation without side effects |

> [!tip] Rule of thumb
> If money, inventory, or a count is involved — **always** wrap it in `transaction.atomic()` and consider `select_for_update()`.
> 