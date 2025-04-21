## Managing Ever‑Growing Tables in Django & PostgreSQL

When a table just keeps growing without bound, you’ll eventually run into performance, storage, and maintenance headaches. In PostgreSQL + Django the most common "policies" or patterns to adopt are:

---
## 1. Data‑Retention / Archival Policy

Decide how long you really need "ho" rows in your main table. Everything older than that gets **archived** or **deleted** on a regular cadence.

- **Retention window**  
    E.g. "keep 3 months of logs in `events`, archive older to `events_archive`"
- **Automate via cron / Django management command**

```bash
# daily-cron.sh
./manage.py archive_old_events --older-than=90
```

```python
# your_app/management/commands/archive_old_events.py
from django.core.management.base import BaseCommand
from django.utils import timezone
from your_app.models import Event, ArchivedEvent

class Command(BaseCommand):
    help = "Move old events to archive table"
    def add_arguments(self, parser):
        parser.add_argument('--older-than', type=int, default=90)
    def handle(self, *args, **opts):
        cutoff = timezone.now() - timezone.timedelta(days=opts['older_than'])
        old = Event.objects.filter(created__lt=cutoff)
        # bulk create into archive
        ArchivedEvent.objects.bulk_create(
            old.values_list(
                'field1','field2',…,'created'
            )
        )
        old.delete()
        self.stdout.write(f"Archived {old.count()} events")
```

---
## 2. [Table Partitioning](/Stuff/Database/Partitioning)

Split the big table into many smaller child tables by range (e.g. date) or list (e.g. customer_id). PostgreSQL’s **declarative partitioning** will then:

- Only scan the relevant partition(s)
- Let you **drop** an old partition instantly (very fast!)
- Keep per-partition indexes small
### Example:
```sql
-- Main “mother” table:
CREATE TABLE events (
  id        bigserial PRIMARY KEY,
  created   timestamptz NOT NULL,
  user_id   int        NOT NULL,
  payload   jsonb
) PARTITION BY RANGE (created);

-- Monthly partitions:
CREATE TABLE events_2025_04
  PARTITION OF events
  FOR VALUES FROM ('2025-04-01') TO ('2025-05-01');
CREATE TABLE events_2025_05
  PARTITION OF events
  FOR VALUES FROM ('2025-05-01') TO ('2025-06-01');
```
Then, dropping all April data is:
```sql
DROP TABLE events_2025_04;
```

In Django you can:
1. **Run raw SQL** in a migration to create each new month’s partition.
2. Use a helper library like **pg_partman** (via the `django-pg-partman` wrapper) to automate new‐partition creation and retention.

---
## 3. Index & [Vacuum](/Stuff/Database/VACUUM) Maintenance

As tables grow, index bloat and dead tuples creep up.
- **Tune autovacuum** thresholds so it kicks in early on large tables.
- Periodically run manual `VACUUM (VERBOSE, ANALYZE) events;` or even `VACUUM FULL` during a maintenance window.
- Consider a **lower fillfactor** (e.g. 70–80%) if you do a lot of in‑place updates:

```sql
ALTER TABLE events SET (fillfactor = 80);
```

---
## 4. Pagination / Query‑Patterns

Large OFFSETs kill performance. Use **keyset pagination** (a.k.a. cursor pagination) in Django:

```python
# Instead of: objs = Event.objects.order_by('-created')[offset:offset+limit]
# Do:
qs = Event.objects.filter(created__lt=last_seen).order_by('-created')[:limit]
```

---
## 5. Horizontal Scaling / Sharding

If you’re truly at multi‑hundreds‑million rows and need “unlimited” scale:

- **Shard by a high‑cardinality key** (e.g. user_id modulo N).
- Use **Citus** or roll your own sharding logic in Django.
---
### Putting It All Together

1. **Define your data‑lifecycle** (e.g. 90‑day retention).
2. **Partition** by time, automate new partitions & old‑partition drops.
3. **Archive** or **purge** old data via a nightly Django command.
4. **Tune** your autovacuum and indexes.
5. **Adopt** keyset pagination for user‑facing queries.

With these in place, your “ever‑growing” table will stay performant, manageable, and predictable.

---