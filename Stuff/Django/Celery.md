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
## 🎯 Why a queue and not a thread

A web worker is a slot. While it renders a PDF or calls Stripe it serves nobody else and holds the connection open — four Gunicorn workers plus four thirty-second exports is a site that is down. In-process concurrency ([Futures](../Python/Futures.md)) fixes throughput inside one request and nothing else.

A background job also needs durability, retry and visibility, and a thread has none of them: replace the process during a deploy and the work is gone, with no record that it ever existed.

| Property | Thread / process pool | Task queue |
| --- | --- | --- |
| **Durability** | in RAM; a deploy or crash drops it silently | a message that outlives the process |
| **Back-pressure** | block the request or raise | queue depth is a number you can alert on |
| **Retry** | hand-rolled, impossible once the process dies | backoff, jitter, attempt caps, dead-lettering |
| **Observability** | no id, no state, no history | task id, state, runtime, events, `inspect` |
| **Deploy coupling** | restarting the web tier kills in-flight work | workers drain on their own schedule |
| **Scale, scheduling** | one machine; a `Timer` a restart forgets | more hosts; `eta` and beat live in the broker |

The price is another service to deploy, monitor and secure, plus a serialization boundary. Pay it when losing the work would matter.

> [!tldr] TL;DR
> Send **primary keys, not model instances**, and dispatch from `transaction.on_commit()`. Delivery is **at-least-once**, so every task must be **idempotent** — a design requirement, not a flag. Put a `soft_time_limit` on every task and let `autoretry_for` + `retry_backoff` + `retry_jitter` handle transient failure. **Route slow work to its own queue** so an export cannot starve password-reset mail. `task_serializer = "json"`, never pickle. And `task_always_eager` in tests lies to you.

---
## 🧱 Architecture — who is allowed to lose what

```text
Django view ──.delay(pk)──►   BROKER    ──►   worker pool    ──►  result backend
 (producer)                 (durable)      (prefork/gevent)     (optional, lossy)
```

- **Producer** — a failed publish is a lost job unless the surrounding transaction fails with it.
- **Broker** — the only durable part. A forgotten message raises no exception anywhere, which is why "we already run Redis for the cache" deserves a decision rather than a shrug.
- **Worker** — disposable. It can die mid-task, and an unacknowledged message runs again.
- **Result backend** — convenience. It may lose everything; load-bearing output belongs in your database.

| | **Redis** | **RabbitMQ** | **SQS** |
| --- | --- | --- | --- |
| Durability | best-effort; a failover can drop messages | durable queues, persistent messages, confirms | replicated, effectively lossless |
| Acks | none — emulated with a **visibility timeout** | real per-message ack/nack | visibility timeout plus delete |
| Long tasks | a second copy runs once the timeout passes | safe: the ack is held for the whole task | fine if the timeout beats your worst case |
| Priority | emulated with extra lists; fights prefetch | real priority queues | none — use separate queues |
| Ops cost | near zero if Redis already runs | a cluster, vhosts and exchanges | no servers; IAM instead of passwords |
| Pick when | short tasks, a lost job is survivable | you need guarantees, priority, routing | you are on AWS and want zero ops |

> [!NOTE]
> **Kafka is not a Celery broker**, structurally. Celery needs competing consumers, per-message acks and redelivery of one message; Kafka is a partitioned, replayable log where consumers own partitions and you cannot ack offset 7 while 5 is outstanding. See [Kafka vs RabbitMQ](../SoftwareDesign/KafkaVsRabbitMQ.md).

---
## 🧩 Defining tasks

A task is a function plus a **name**. The name travels in the message; the worker receives a string and looks up a function. Every surprise about task definitions follows from that.

```python
# ---- proj/celery.py ----  import this from proj/__init__.py or nothing registers
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "proj.settings")
app = Celery("proj")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()         # imports tasks.py from every INSTALLED_APPS entry

# ---- settings.py ----  the prefix is stripped: CELERY_BROKER_URL -> broker_url
CELERY_BROKER_URL = "redis://broker:6379/0"    # not the db your cache uses
CELERY_TASK_SERIALIZER = "json"
CELERY_ACCEPT_CONTENT = ["json"]               # refuse pickle even if offered
CELERY_TASK_IGNORE_RESULT = True               # opt in per task
CELERY_TASK_ACKS_LATE = True
CELERY_WORKER_PREFETCH_MULTIPLIER = 1
CELERY_TASK_SOFT_TIME_LIMIT = 540              # raises inside the task
CELERY_TASK_TIME_LIMIT = 600                   # then SIGKILL the child
```

- **`@shared_task`, not `@app.task`**, in Django: it binds to whichever app is current, so the module stays reusable and needs no import of your project's app object.
- **`bind=True` gives you `self`** — `self.request.id` for log correlation, `self.request.retries`, `self.retry()`. Most non-trivial tasks want it.
- **The name is the wire contract.** Implicit names are module paths, so moving or renaming the function leaves queued messages that no worker recognises (`Received unregistered task of type`). Set `name=` explicitly and keep a shim under the old name until the queue drains.
- **The signature is a contract too.** A new required positional argument breaks in-flight messages; add keyword arguments with defaults instead.

---
## 📞 Calling tasks

`delay()` is `apply_async()` without the boilerplate. Everything worth configuring lives only on the full form.

| Form / option | What it does |
| --- | --- |
| `t.delay(a, b)` | shorthand for `apply_async(args=(a, b))` |
| `apply_async(args, kwargs)` | the full form; every option below needs it |
| `countdown=30` / `eta=dt` | not before N seconds / not before an absolute aware datetime |
| `expires=300` | discard if not started in time — right for anything worthless when late |
| `queue="slow"` | route explicitly, overriding `task_routes` |
| `priority=0..9` | broker-specific and approximate; on Redis it needs several queues *and* prefetch 1 |
| `headers={...}` | metadata for logs and interceptors, not business arguments |
| `t.apply()` | inline in this process, now — debugging only |

### 🧨 Pass identity, and publish after the commit

```python
# ❌ unserializable, and stale the moment it is sent
send_receipt.delay(order)

# ❌ published inside an open transaction: the worker's own connection cannot
#    see the row yet, and a rollback leaves a job for a row that never existed
with transaction.atomic():
    order = Order.objects.create(...)
    send_receipt.delay(order.pk)

# ✅ identity only, published after the commit that makes the row visible
with transaction.atomic():
    order = Order.objects.create(...)
    transaction.on_commit(partial(send_receipt.delay, order.pk))
```

The broker is faster than your database. A worker picks the message up within milliseconds, opens its own connection and selects a row nothing else can see yet: `DoesNotExist`, or a silent no-op if the task guards with `filter(...).first()`.

Use `partial`, not a lambda — `on_commit(lambda: t.delay(o.pk))` in a loop captures the last `o`. `ATOMIC_REQUESTS = True` puts every view in this situation, and Django's `TestCase` never commits, so callbacks fire only under `captureOnCommitCallbacks()`.

> [!IMPORTANT]
> `on_commit` closes the race, not the durability gap: the commit can succeed and the process die before the callback publishes, leaving a row with no job and no error. When a missed job is a business problem, write the message *inside* the transaction — the transactional-outbox pattern. Isolation rules: [Database Transactions](../Database/Database%20Transactions.md).

---
## 🔁 Delivery semantics — at-least-once

There is no exactly-once mode, there cannot be one across a network, and everything below follows from that.

| Setting | Behaviour | Consequence |
| --- | --- | --- |
| `task_acks_late = False` (default) | acked before the function runs | a worker killed mid-task drops the job |
| `task_acks_late = True` | acked after it returns | a worker killed mid-task means redelivery: it can run twice |
| `task_reject_on_worker_lost = True` | requeue on `SIGKILL`/OOM | a task that OOMs deterministically becomes an infinite redelivery loop |
| `visibility_timeout` (Redis, SQS) | how long a message may sit unacked | effectively **the maximum task lifetime** |

Redis has no ack primitive, so Celery emulates one: a delivered message moves to an unacked set with a `visibility_timeout` deadline (default **3600 s**). When it passes, the same message goes to another worker — while the first is still running it. Your 90-minute export now runs twice, concurrently, with nothing logged.

> [!WARNING]
> `visibility_timeout` must exceed the longest lifetime of any message on that broker, scheduled ones included: `countdown=7200` on Redis is redelivered before its ETA and piles up copies of itself. For multi-hour work, chunk it or use RabbitMQ, which holds the ack for the task's lifetime.

### 🧷 Idempotency is a requirement, not a nice-to-have

A task can run twice, so running twice must be harmless. Pick a mechanism per task instead of hoping.

| Technique | Use it when |
| --- | --- |
| Naturally idempotent write | the second run is a no-op: `update(status="sent")`, a `PUT` to a fixed key |
| Unique constraint + `get_or_create` | one row per logical event; let the database reject the duplicate |
| Idempotency key sent downstream | the third party supports one — Stripe and most payment APIs do |
| State guard in a conditional `UPDATE` | the task advances a state machine and exactly one run must win |
| `select_for_update` or an advisory lock | the work cannot be made idempotent and must be serialized |
| Cache mutex (`cache.add`) | best-effort dedupe, where skipping a run is acceptable |

```python
@shared_task(bind=True, max_retries=3)
def send_receipt(self, order_id: int) -> None:
    # 1) claim atomically: one statement, so there is no read-then-write window
    claimed = Order.objects.filter(pk=order_id, receipt="pending").update(
        receipt="sending")
    if not claimed:
        return                   # 2) duplicate delivery — the other run won
    # 3) the downstream call carries its own key, so a redelivery after a
    #    network timeout still results in exactly one email
    mailer.send(order_id, idempotency_key=f"receipt:{order_id}")
    Order.objects.filter(pk=order_id).update(receipt="sent")     # 4) settle
```

`update()` returns the number of matched rows, so of two concurrent runs exactly one gets `1`. `if o.receipt == "pending": o.save()` is a read, a decision and a write with a window both runs walk through. With no state machine to guard, use `transaction.atomic()` plus `select_for_update()` on the row.

---
## 💥 Failure handling — retries, timeouts, dead letters

An uncaught exception marks the task `FAILURE`; raising `Retry` reschedules it. The declarative form covers the common case and gets the arithmetic right.

```python
from celery.exceptions import SoftTimeLimitExceeded

@shared_task(
    bind=True,
    autoretry_for=(RequestException,),  # transient, network-shaped errors only
    retry_backoff=2,                    # 2, 4, 8, 16 ... seconds
    retry_backoff_max=600,
    retry_jitter=True,                  # or every client retries in lockstep
    max_retries=6,
    soft_time_limit=110,
    time_limit=120,
)
def sync_customer(self, customer_id: int) -> None:
    try:
        remote = api.fetch(customer_id, timeout=10)   # always a client timeout
    except ValueError:
        raise                     # malformed on every attempt: fail fast
    except SoftTimeLimitExceeded:
        release_partial_state(customer_id)      # tidy up, then give up
        raise
    Customer.objects.filter(pk=customer_id).update(**remote)
```

- **Retry only what a retry can fix.** `autoretry_for=(Exception,)` retries a `ValidationError` six times with backoff and learns nothing. Timeouts, 502s, lock contention and rate limits qualify; bad input does not.
- **A retry is a new message.** `self.retry(exc=e, countdown=30)` re-publishes with `retries + 1` and raises, so nothing after it runs — and side effects already applied are not undone.
- **Jitter is not optional.** Without it, one downstream outage produces synchronised retry waves at t+2, t+4, t+8 from every worker at once.
- **Decide what "out of retries" means.** The task ends `FAILURE` with `MaxRetriesExceededError`, and if nobody watches failures the job is gone. Land it somewhere: an `on_failure` hook writing a `FailedJob` row, `task_failure` into Sentry, or a RabbitMQ dead-letter exchange.

| | `soft_time_limit` | `time_limit` |
| --- | --- | --- |
| Mechanism | `SIGUSR1`, raised as `SoftTimeLimitExceeded` in your code | `SIGTERM` then `SIGKILL` of the child |
| Your code can | catch, roll back, release the lock, re-raise | nothing — `finally` never runs |
| Outcome | an ordinary exception, so retries and `on_failure` work | the worker logs the loss; redelivery with `acks_late` |
| Rule | set one on **every** task, just under the hard limit | the backstop for code that hangs in a C extension |

> [!WARNING]
> `except Exception: pass` in a task is worse than a crash: the message is acked, the task reports `SUCCESS`, the dashboard is green and the work never happened. Time limits also depend on signals and the prefork pool — with `--pool=threads` there is nothing to signal, so every I/O call needs its own timeout.

---
## ⚙️ The worker execution model

A `celery worker` process is a supervisor dispatching into a pool, and the pool decides what work it is good at.

| Pool | Runs tasks in | Right for | Watch out for |
| --- | --- | --- | --- |
| `prefork` (default) | N forked children, one task each | CPU work, the ORM, C extensions | memory × N, a DB connection per child |
| `--pool=threads` | a thread pool in one process | I/O at modest fan-out | the GIL; no signal-based time limits |
| `--pool=gevent` / `eventlet` | green threads, hundreds per process | HTTP, webhooks, mail | patch before any socket import; one blocking C call stalls everything |
| `--pool=solo` | the main thread | debugging | never production |

| Flag / setting | Get it wrong and |
| --- | --- |
| `--concurrency=N` | prefork memory exhaustion, N idle DB connections per host |
| `worker_prefetch_multiplier` | the default 4 lets one child hoard four 20-minute jobs while siblings idle — set **1** |
| `--max-tasks-per-child=N` | a leaky library grows until the OOM killer intervenes |
| `--max-memory-per-child=KB` | the same failure, measured instead of guessed |
| `--queues=a,b` | one pool means the slowest task type sets everyone's latency |

One shared queue is the most common capacity bug here: a password reset queued behind 400 CSV exports arrives forty minutes late.

```python
# settings.py — separate the fast lane from the slow lane
CELERY_TASK_ROUTES = {
    "accounts.send_email":      {"queue": "fast"},
    "billing.generate_invoice": {"queue": "slow"},
    "reports.*":                {"queue": "slow"},   # glob patterns work
}
CELERY_TASK_DEFAULT_QUEUE = "fast"
```

```bash
# two deployments, sized independently — that is the entire point
celery -A proj worker -Q fast -c 8 --prefetch-multiplier=4 -n fast@%h
celery -A proj worker -Q slow -c 2 --prefetch-multiplier=1 -n slow@%h \
       --max-tasks-per-child=50 --soft-time-limit=1800
```

Split by **latency requirement**, not by Django app: `fast`, `slow` and perhaps `beat` covers most systems. A worker started `-Q fast,slow` is back to one shared pool, so the split only helps when the deployments are separate.

---
## ⏰ Scheduling — beat and the exactly-one-scheduler rule

Beat does not run tasks. It is a producer with a clock: it publishes on a schedule and ordinary workers consume.

```python
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    "expire-trials": {
        "task": "accounts.expire_trials",
        "schedule": crontab(hour=3, minute=15),   # 03:15 in CELERY_TIMEZONE
        "options": {"queue": "slow", "expires": 3600},
    },
    "poll-webhooks": {
        "task": "integrations.poll",
        "schedule": 300.0,                        # every five minutes
        "options": {"expires": 240},   # skip, don't pile up, if workers lag
    },
}
CELERY_TIMEZONE = "Europe/Berlin"     # crontab() is interpreted in this zone
```

> [!IMPORTANT]
> **Exactly one beat process may run at a time.** It has no leader election and no locking, so two replicas publish every scheduled task twice, every night, silently. Pin the deployment to one replica, and defend the task itself with a state guard if double-firing would be damaging during a rollout.

- **The schedule file is state.** `--schedule=/var/lib/celery/beat-schedule` remembers the last run of each entry; on an ephemeral filesystem it is wiped every deploy, so interval entries fire again on each restart.
- **Beat does not catch up.** Down from 02:00 to 04:00, a `crontab(hour=3)` entry simply never happens. Work that must not be skipped needs a task that asks "what is still unprocessed?".
- **The clock is a publish time, not a start time**, which is why `expires` belongs on periodic tasks. `DatabaseScheduler` moves the schedule into the admin — right for operations staff, and also a way for one careless row to fire a task every second.

| Mechanism | Best at | Weak at |
| --- | --- | --- |
| **Celery beat** | schedules that fan out over a fleet, with retries and routing | needs broker, workers and beat healthy; a single point of scheduling |
| **cron** | one host, one command, zero infrastructure | no locking so runs overlap, no retries, a log file for visibility |
| **systemd timer** | `OnCalendar=`, `Persistent=true` catch-up, journald, cgroup limits | per-host, and you build "did it succeed" yourself — [Service](../Linux/Service.md) |

Workers are long-running daemons, so on a VM they belong under a supervisor rather than a `nohup`:

```ini
# /etc/systemd/system/celery-slow.service
[Service]
Type=simple
User=app
EnvironmentFile=/etc/app/celery.env
WorkingDirectory=/srv/app
ExecStart=/srv/app/.venv/bin/celery -A proj worker -Q slow -c 2 -n slow@%%h
Restart=always
KillSignal=SIGTERM     # warm shutdown: stop consuming, finish in-flight tasks
TimeoutStopSec=300     # ...but do not wait forever
KillMode=mixed         # SIGKILL whatever survives the timeout
```

`SIGTERM` is the warm shutdown: stop consuming, finish what you hold. A second `SIGTERM` abandons in-flight tasks — redelivery with `acks_late`, loss without. Keep `TimeoutStopSec` above your longest task.

---
## 🔗 Canvas — composing work

A **signature** is a task call captured as data: name, arguments, options. Everything else in Canvas is built from it.

| Primitive | Shape | Use for |
| --- | --- | --- |
| `task.s()` / `.si()` | one deferred call | the building block; `.si()` ignores the previous result |
| `chain(a.s(), b.s())` | sequential, result piped forward | pipelines where step 2 consumes step 1 |
| `group(a.s(i) for i in ids)` | parallel fan-out | "process these 500 rows" |
| `chord(group(...), cb.s())` | fan-out, then one callback with every result | aggregating a batch — **needs a result backend** |
| `chunks(items, n)` | one group split into batches | 50 000 items as 500 tasks of 100 |

```python
from celery import chain, chord, group

chord(
    group(fetch_region.s(r) for r in regions),  # header: runs in parallel
    assemble_report.s(report_id),               # body: runs when all finish
).apply_async()

# immutable links: upload and notify ignore the previous return value
chain(render.s(doc_id), upload.si(doc_id), notify.si(doc_id)).apply_async()
```

> [!WARNING]
> **Never call `.get()` on a result inside a task.** On prefork it deadlocks by construction: the parent occupies a child, and the subtask it waits for can never be scheduled. Return `self.replace(signature)` instead — Celery substitutes the new `group` or `chain` into the workflow and the parent finishes. Building a `group` inside a task and waiting on it is the same mistake in a different hat.

---
## 📦 Results — when a backend earns its cost

Most tasks are fire-and-forget, and for those a result backend is an extra write per task, an extra read per poll, and one more store to expire. You need one for `chord`, for reading `AsyncResult` outside the worker, and for progress a UI polls; you do not need one when the task's real output is a row it wrote itself.

- `task_ignore_result = True` globally, `ignore_result=False` on the few that need one, and always set `result_expires` — a backend without expiry is an unbounded store, which on Redis is a memory incident.
- **Not the Django database at volume.** `django-celery-results` writes a row per task and pollers read it repeatedly: a write-heavy queue inside the database you were trying to protect.
- `AsyncResult(id).get()` in a web request is a synchronous wait on a background job, with an extra network hop. Poll from the client or push over a websocket.

---
## 🐍 Django-specific operational details

### 🔌 Stale database connections

Celery's Django fixup calls `close_old_connections()` around each task, so the ordinary case is handled. The exceptions break: a task that sits in an HTTP call for ten minutes and then writes, or threads you spawn yourself.

```python
@shared_task(soft_time_limit=1700, time_limit=1800)
def nightly_reconcile() -> None:
    close_old_connections()      # discard anything that died while idle
    slow_external_sync()         # minutes with no database traffic
    close_old_connections()      # by now the connection may be dead
    write_results()
```

`CONN_MAX_AGE` needs care under prefork: every child keeps its own connection, so `-c 8` on four hosts is 32 mostly-idle connections, and an hourly task holds one the server may already have dropped — `InterfaceError: connection already closed`, once, then fine. Keep it at `0` for workers, or let PgBouncer own pooling.

### 🧪 Testing, and why `task_always_eager` lies

`CELERY_TASK_ALWAYS_EAGER = True` makes `.delay()` call the function inline. It is the quickest way to a green suite and a reliable way to ship a broken task, because it removes exactly the parts that fail in production.

| In eager mode | In production |
| --- | --- |
| the function gets the objects you passed | arguments are JSON round-tripped: a `Decimal` or an instance now raises |
| it runs in the test's transaction and sees uncommitted rows | the worker sees only committed data — the `on_commit` bug is invisible |
| exceptions land in your assertion | exceptions become retries, backoff, and a `FAILURE` nobody reads |
| no retries, time limits, prefetch or routing | all of them apply, each with its own failure mode |

```python
# 1) unit-test the work with no Celery involved
def test_generate_invoice(order):
    generate_invoice(order.pk)
    order.refresh_from_db()
    assert order.invoice_number

# 2) test that the view dispatches, after commit, with the right argument
def test_view_queues_invoice(client, django_capture_on_commit_callbacks):
    with django_capture_on_commit_callbacks(execute=True):
        with patch("billing.tasks.generate_invoice.delay") as delay:
            client.post("/orders/", data)
    delay.assert_called_once_with(Order.objects.get().pk)
```

Tasks carry hidden state across a process boundary, which makes them a classic source of order-dependent failures — see [Flaky Tests](../Python/FlakyTest.md). `pytest-celery` starts a real broker and worker for a handful of integration tests; not for every task.

### 📈 Monitoring — what to alert on

A queue fails quietly: a fleet that has stopped consuming looks exactly like a fleet with nothing to do.

```bash
celery -A proj inspect active      # running right now, per worker
celery -A proj inspect reserved    # prefetched but not started yet
celery -A proj status              # which workers are alive at all
redis-cli -n 0 llen slow           # raw queue depth — the number that matters
```

| Signal | Why to page on it |
| --- | --- |
| **Queue depth** and its slope | the best single indicator: consumption is losing to production |
| **Publish → start** latency, separate from runtime | queue wait and execution time break differently |
| **Failure and retry rate** per task | a retry storm is a dependency outage announcing itself early |
| **Consumer count** per queue | zero consumers on a live queue is an outage with no errors |
| **Age of last success** per beat entry | a dead scheduler is silent by definition |

Flower gives a dashboard and REST API over task events (`worker_send_task_events`); beyond a small deployment, export the same events to Prometheus. It has no real authentication, so it is never a public port.

---
## 🔐 Security

- **Never enable pickle.** A broker that accepts pickled messages is a remote-code-execution channel for anything able to publish. `task_serializer = "json"` is half of it; `accept_content = ["json"]` is the half that makes workers *refuse* pickle.
- **Keep the broker internal.** Redis has no authentication by default: `requirepass` or ACLs, a private bind address, `rediss://` across untrusted networks. An exposed broker lets an attacker enqueue any registered task with any arguments — code execution without needing pickle at all.
- **Broker credentials live in a URL**, so they surface in `ps`, container inspect output and tracebacks. Load them from a secret manager — see [Vault](../DevOps/Vault/README.md) — and make rotation possible without a rebuild.
- **Never take a task name from user input.** `app.send_task(request.data["task"])` is arbitrary code execution with extra steps.
- **Arguments get logged** into the broker, the result backend, Flower and your log pipeline. Pass an id, not a token or a card number.
- **Authorize before you enqueue.** A task has no `request.user`, so the decision travels as data — and gets re-checked at execution time, because the message may run after access was revoked.

---
## 🤔 Should this be a task at all?

A queue bought to save 20 ms is a net loss: the broker round-trip costs milliseconds, and around it you own a serialization boundary, a deployment, a dashboard and a pager.

| The work is | Where it belongs |
| --- | --- |
| under ~100 ms and needed for the response | inline, in the request |
| slow because a query is slow | fixed, not moved — [ORM Optimization](./ORMOptimization.md). A 40-second query is still 40 seconds where nobody is watching |
| slow because the same value is recomputed on every read | cached or precomputed — [Caching](../SoftwareDesign/Caching.md) |
| slow, external and retryable: mail, payments, PDFs, imports | a task |
| scheduled, or required to survive a deploy | a task |
| fan-out over I/O *inside* one request the user is waiting on | a thread pool — [Futures](../Python/Futures.md) |

The shape that pays for the queue: validate, write a `pending` row, enqueue after commit, return `202 Accepted` with a URL to poll. The state stays inspectable in your own database, which puts weight on the view and serializer layer — see [DRF Internals](./DRF/DRFInternals.md).

---
## 🚨 Common mistakes

| Mistake | Why it hurts | Fix |
| --- | --- | --- |
| Passing a model instance | unserializable, or stale by the time it runs | pass the pk and re-fetch |
| `.delay()` inside `atomic()` | the worker cannot see the row; a rollback orphans the job | `on_commit(partial(...))` |
| Assuming one delivery | duplicate emails, double charges | a state guard or an idempotency key |
| No `soft_time_limit` | one hung call occupies a child forever | a soft limit on every task |
| `autoretry_for=(Exception,)` | permanent errors burn six attempts and hide the cause | retry transient types only |
| Default `prefetch_multiplier` | one child hoards four long jobs while siblings idle | `1` on the slow queue |
| One queue for everything | a password reset waits behind 400 exports | `task_routes`, separate deployments |
| Two beat replicas | every periodic task fires twice, silently | exactly one scheduler |
| `visibility_timeout` below runtime | Redis redelivers work that is still running | raise it, or use RabbitMQ |
| `.get()` inside a task | deadlocks the prefork pool | `self.replace()` or a chord |
| `except Exception: pass` | acked, `SUCCESS`, green dashboard, no work done | let it raise, or retry |
| Testing with `task_always_eager` | hides serialization, commit, retry and routing bugs | call the function directly |
| Pickle, or a public broker | remote code execution for anyone who can publish | `accept_content = ["json"]` |

---
## 🧠 Summary

| Question | The short answer |
| --- | --- |
| **When is a queue worth it?** | when the work is slow *and* losing it would matter |
| **What crosses the wire?** | JSON: ids and scalars, never instances |
| **When do I publish?** | in `transaction.on_commit()`, always |
| **How often does it run?** | at least once — design for two |
| **What makes a rerun safe?** | a conditional `UPDATE`, a unique constraint, an idempotency key |
| **Which failures retry?** | transient ones, with backoff, jitter and a cap |
| **What stops a hang?** | `soft_time_limit` everywhere, `time_limit` as backstop |
| **Which pool?** | prefork for CPU and the ORM, gevent for I/O fan-out |
| **Why separate queues?** | so slow work cannot set the latency of fast work |
| **Which broker?** | RabbitMQ for guarantees, Redis for short and cheap, SQS for zero ops |
| **Result backend?** | only for chords, and state you have nowhere better to keep |
| **What do I alert on?** | queue depth, publish-to-start latency, retry rate, consumers |
| **Never enable** | pickle, a public broker, a task name from user input |

---
## 📚 References

- [Celery — Tasks](https://docs.celeryq.dev/en/stable/userguide/tasks.html) — names, binding, retries, time limits, idempotency guidance.
- [Celery — Calling tasks](https://docs.celeryq.dev/en/stable/userguide/calling.html) and [Workers](https://docs.celeryq.dev/en/stable/userguide/workers.html) — `apply_async` options, pools, prefetch, remote control.
- [Celery — Periodic tasks](https://docs.celeryq.dev/en/stable/userguide/periodic-tasks.html) and [Canvas](https://docs.celeryq.dev/en/stable/userguide/canvas.html) — beat, `crontab()`, signatures, chords.
- [Django — `transaction.on_commit()`](https://docs.djangoproject.com/en/stable/topics/db/transactions/#performing-actions-after-commit) — the commit hook and testing it.
- [RabbitMQ — Reliability guide](https://www.rabbitmq.com/docs/reliability) — acknowledgements, redelivery, publisher confirms.
- [AWS — Transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html) — closing the publish-after-commit gap.
- [Azure Architecture Center — Retry pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry) — backoff, jitter and when not to retry.












