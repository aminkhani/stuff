```table-of-contents
title: Table of Contents
style: nestedList # TOC style (nestedList|nestedOrderedList|inlineFirstLevel)
minLevel: 0 # Include headings from the specified level
maxLevel: 0 # Include headings up to the specified level
includeLinks: true # Make headings clickable
hideWhenEmpty: false # Hide TOC if no headings are found
debugInConsole: false # Print debug info in Obsidian console
```

## TextField()

In Django’s model layer, `TextField` is the “unconstrained” text column type: it’s used whenever you need to store arbitrarily long strings (e.g. blog posts, comments, descriptions) rather than short snippets (where you’d use `CharField`).

### 📝 Defining a `TextField` in Your Django Model

```python
from django.db import models

class Article(models.Model):
    title      = models.CharField(max_length=200)
    body       = models.TextField()            # “unlimited” text
    summary    = models.TextField(blank=True)  # optional text
    updated_at = models.DateTimeField(auto_now=True)
```

- No `max_length` needed (ignored by most backends if present).
- Defaults: `blank=False, null=False` (empty strings, not NULL). Use `null=True` if you want SQL `NULL`.
---
### 🔗 Database Column Type Mapping

Django’s backends map `TextField` to the large‑text type for each database:

|Backend|Column Type|
|---|---|
|PostgreSQL|`TEXT`|
|MySQL/Maria|`LONGTEXT` (or `TEXT`)|
|SQLite|`TEXT`|
|Oracle|`CLOB`|

These mappings live in each backend’s `DatabaseOperations.db_type()`.

---
### 🔄 ORM‑Level Conversion

- **`get_prep_value(self, value)`**  
    Ensures your Python `str` is safe for the DB driver.
- **`from_db_value(self, value, …)`**  
    (Django ≥1.8) Converts DB text back into a Python `str`.
- **`value_to_string(self, obj)`**  
    Used when serializing (e.g. fixtures).
---
### 🔍 Querying & Lookups

Standard string lookups apply:

```python
# exact, contains, startswith, regex…
Article.objects.filter(body__icontains="django")
```

> [!WARNING] Indexes on `TEXT` are limited—use `CharField` or PostgreSQL full‑text search (`SearchVector`) if you need speed.

---
### 🚀 Performance Considerations

- **Storage**: Large texts often live “out of row” (BLOB/CLOB), impacting I/O.
- **Memory**: Fetching huge text fields can spike Python’s memory usage.
- **Indexing**: Limited support—consider alternate strategies for fast lookup.
---
### 📋 Lifecycle Summary

1. **Model**: You declare `body = models.TextField()`.
2. **Migration**: Django emits `ALTER TABLE … ADD COLUMN body TEXT;`.
3. **Save**: Python `str` → `get_prep_value()` → SQL.
4. **Load**: SQL text → `from_db_value()` → Python `str`.
5. **Query**: Lookups like `__contains` → SQL `LIKE '%…%'` or FTS.
---
### 🐘 Inspecting Column Types in `psql`

**Describe the table**

```sql
-- basic listing:
\d your_table_name

-- with extras (storage, stats…):
\d+ your_table_name
```
Shows each column’s name, type, modifiers, defaults, and more.

---
## CharField()

> [!IMPORTANT] Here’s how `character varying(n)` (aka `varchar(n)`) and `character(n)` (aka `char(n)`) differ in PostgreSQL:

### 📦  Storage & Padding

- **`varchar(n)`**
    - Variable‑length string, up to _n_ bytes.        
    - No automatic padding—what you store is what you get back.
    - Example: storing `"Hi"` returns `"Hi"`.
- **`char(n)`**
    - Fixed‑length string, always exactly _n_ bytes.
    - Shorter values are right‑padded with spaces on storage **and** retrieval.
    - Example: storing `"Hi"` returns `"Hi   "` (with trailing spaces).
---
### 🚨 Length Enforcement

- **Both** will ❌ error if you try to store a string longer than _n_.
- **`char(n)`** pads shorter inputs; **`varchar(n)`** does **not**.
```sql
-- With n = 5:
INSERT INTO t(col) VALUES ('HelloWorld');  -- ❌ too long!
INSERT INTO t(col) VALUES ('Hi');
SELECT '"' || col || '"' FROM t;
-- varchar(5):  "Hi"  
-- char(5):     "Hi   "  (with 3 trailing spaces) ✨
```
---
### 🔍  Comparison Semantics

- PostgreSQL follows the SQL spec: **ignores trailing spaces** when comparing both types.

```sql
'abc'::char(5) = 'abc'::varchar(5);   -- true ✅  
'abc  '::char(5) = 'abc';             -- true ✅  
```

- But when you `SELECT …`, those trailing spaces **show up** from `char(n)` (unless you `TRIM()` them).
---
### ⚙️ Performance & Use‑Cases

- **`varchar(n)`** is your go‑to when you need a maximum length 📏.
- **`char(n)`** is for **fixed‑width** needs (legacy protocols, aligned dumps) 🗜️.

> [!IMPORTANT] In practice, there’s **no significant perf diff**—both store strings efficiently under the hood.

---
### 🛠️ Best Practice

- Use **`varchar(n)`** (or even plain `text` if you don’t need an upper limit) ✅.
- Reserve **`char(n)`** for very specific fixed‑width requirements only 🏷️.
---
### 🗂️  Quick Reference Table

| 🔖 Feature             | **`varchar(n)`**        | **`char(n)`**                      |
| ---------------------- | ----------------------- | ---------------------------------- |
| 🔢 Length enforcement  | Error if > _n_          | Error if > _n_                     |
| 📦 Storage length      | Actual string length    | Always exactly _n_ (spaces padded) |
| ➡️ Trailing spaces     | Preserved as‑is         | Always present up to _n_           |
| ⚖️ Comparison behavior | Ignores trailing spaces | Ignores trailing spaces            |
| 🎯 Typical use case    | Variable‑length text    | Fixed‑width/legacy formats         |

---
