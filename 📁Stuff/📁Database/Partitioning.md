## 🧱 What is Partitioning in Databases?

**Partitioning** is the process of **splitting a large table into smaller, more manageable pieces** called **partitions**—while keeping them logically as **one single table** to the application.

You can think of it like cutting a **giant pizza 🍕** into slices—you still see _one pizza_, but each piece is easier to handle!

---
### 💡 Why Partition Data?

- 📈 **Improve query performance**
- 💾 **Manage large datasets** more efficiently
- 📦 **Distribute load** across multiple disks/servers
- 🚀 **Speed up maintenance** like backups and deletes
- 🔍 **Faster search** in specific segments
---
## 📊 When Should You Use Partitioning?

Use partitioning when:

- Your **tables have millions (or billions) of rows** 🏗️
- Most queries focus on **a subset of the data** (e.g., by date, region, or user)
- You want to **archive old data** or drop old chunks quickly 📦🗑️
- You're managing **time-series**, logs, or event data 🕒
---
## 🧩 Types of Partitioning

Let’s break down the main types:
### 1. 📆 **Range Partitioning**

Split by ranges of values (e.g., dates, ID ranges)
**Example:**
- Jan–Mar → Partition 1
- Apr–Jun → Partition 2
- Jul–Sep → Partition 3

```sql
PARTITION BY RANGE (created_at)
```

🧠 Use it when:
- You have **time-based data** like logs, orders, events
- You often query recent or specific date ranges
### 2. 🔢 **List Partitioning**

Split by **specific values**
**Example:**
- Region = ‘US’ → Partition A
- Region = ‘EU’ → Partition B
- Region = ‘ASIA’ → Partition C

```sql
PARTITION BY LIST (region)
```

🧠 Use it when:
- You group data by **discrete categories**, like region, language, or plan type

### 3. 🎲 **Hash Partitioning**

Distributes rows using a **hash function** evenly
**Example:**
- Hash(user_id) % 4 → splits into 4 partitions

```sql
PARTITION BY HASH (user_id)
```

🧠 Use it when:
- You want **even distribution**
- Your queries are **spread across all values**, not just one range
---
### 4. 🧬 **Composite Partitioning** (Hybrid)

Mix two methods like **range + hash** for more control.
**Example:**
- First by `created_at` (range) → then by `user_id` (hash) inside each range

🧠 Use it when:
- You need both **query optimization** and **even load distribution**
---
## ⚙️ Partitioning vs Sharding

|Concept|Emoji|What it means|
|---|---|---|
|**Partitioning**|🗂️|Splitting data _within_ a single database|
|**Sharding**|🌍|Splitting data _across multiple databases/servers_|

> 🔥 Pro tip: Partitioning can be done inside a single Postgres/MySQL instance. Sharding is more complex and involves scaling horizontally.
---
## 🚀 Benefits of Partitioning

✅ Faster queries on subsets  
✅ Better I/O performance  
✅ Easier to archive/drop old data  
✅ Improved maintenance (analyze, vacuum, backup)  
✅ Can support **partition pruning** (engine ignores irrelevant partitions)

---
## 🚨 When NOT to Use Partitioning

❌ Your table is **small** (under 1M rows)  
❌ Your queries always **touch all data**  
❌ You have **complex joins** that may span many partitions  
❌ Partitioning is overkill without a clear pattern to split on

---
## 🛠️ Partitioning in PostgreSQL

```sql
CREATE TABLE logs (
  id SERIAL,
  message TEXT,
  created_at DATE
) PARTITION BY RANGE (created_at);
```
Then create partitions:
```sql
CREATE TABLE logs_2024 PARTITION OF logs
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```
PostgreSQL supports:
- Range 📆
- List 🔢
- Hash 🎲
---
## 🧪 Real-Life Use Cases

|Use Case|Partition Type|Emoji|
|---|---|---|
|Web server logs|Range by date|📅📁|
|E-commerce orders by country|List|🌍🛒|
|Analytics by user ID|Hash|👤📊|
|IoT sensor data|Range + Hash|📡🧬|

---
## 🧠 Summary

|Concept|Emoji|Takeaway|
|---|---|---|
|Partitioning|🧩|Breaking large tables into pieces for performance|
|Types|📅🔢🎲|Range, List, Hash (and combos!)|
|When to use|🚀|Big tables, time-series, regional data|
|Benefits|⚡|Speed, maintainability, scalability|

---
> Let’s build a **Django example with PostgreSQL partitioning** from scratch! 🐍
> 
> We'll focus on **Range Partitioning by date** (great for logs, orders, events, etc.)

## 🧪 Scenario: Log Table Partitioned by `created_at`

We’ll create a `Log` model, where logs are **partitioned by month** based on their creation date.

---
### ✅ Step 1: Set Up Django & PostgreSQL

Make sure PostgreSQL is installed and Django is using it:

```bash
pip install psycopg2-binary
```
**`settings.py`**:
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'partition_demo',
        'USER': 'postgres',
        'PASSWORD': 'yourpassword',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
```
### ✅ Step 2: Create Your `Log` Model 🧩

```python
# models.py
from django.db import models

class Log(models.Model):
    message = models.TextField()
    created_at = models.DateTimeField()

    class Meta:
        managed = False  # We'll handle table creation manually
        db_table = 'log'
```

> ⚠️ `managed = False` because we’ll use raw SQL to define the partitions.

### ✅ Step 3: Create Partitioned Table in Raw SQL 🏗️

Create a SQL file: `sql/partition_setup.sql`

```sql
CREATE TABLE log (
    id SERIAL PRIMARY KEY,
    message TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

-- Partition for April 2025
CREATE TABLE log_2025_04 PARTITION OF log
FOR VALUES FROM ('2025-04-01') TO ('2025-05-01');

-- Partition for May 2025
CREATE TABLE log_2025_05 PARTITION OF log
FOR VALUES FROM ('2025-05-01') TO ('2025-06-01');
```
Run it in PostgreSQL:
```bash
psql -U postgres -d partition_demo -f sql/partition_setup.sql
```
### ✅ Step 4: Create Migration (Only for Django Awareness)

```bash
python manage.py makemigrations yourapp
```
Manually edit the migration file to **skip table creation**:
```python
operations = [
    migrations.RunSQL("SELECT 1;", reverse_sql="SELECT 1;"),
]
```
### ✅ Step 5: Use It in Django Code 🔧

You can now insert logs normally, and PostgreSQL will route them to the correct partition!

```python
from yourapp.models import Log
from datetime import datetime

Log.objects.create(message="Partitioned log!", created_at=datetime(2025, 4, 15))
```
### ✅ Step 6: Query Logs 🚀

Django doesn’t care that the table is partitioned—it just queries the main table:

```python
Log.objects.filter(created_at__gte="2025-04-01", created_at__lt="2025-05-01")
```

>⚡ PostgreSQL **partition pruning** will optimize this automatically!
---
### 🎯 BONUS: Automatically Create New Partitions

You can write a Django `management command` or a `cron job` to:
1. Check for the next month
2. Create a new partition dynamically with `CREATE TABLE ... PARTITION OF`
---
### 🧠 Summary

|Step|What You Did|Emoji|
|---|---|---|
|1|Set up Django + Postgres|⚙️🐘|
|2|Defined `Log` model|🧩📝|
|3|Created partitioned table in SQL|📄🪄|
|4|Inserted/query data|📥🔍|
|5|PostgreSQL handled partitions|🪄⚡|

---
