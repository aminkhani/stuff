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
## 🔮 What is a Future?

A **Future** is a handle to a result that does not exist yet. You hand work to something else, you immediately get back a receipt, and later you cash the receipt in for either a value or an exception.

`concurrent.futures` (stdlib, since 3.2) gives you two halves of that idea:

- an **Executor** — a pool of workers plus a queue of pending work (`ThreadPoolExecutor`, `ProcessPoolExecutor`)
- a **`Future`** — the receipt for one submitted callable

You never construct a `Future` yourself; the executor returns one from `submit()`. Your code keeps running while a worker executes the callable, and the future is where the outcome eventually lands.

> [!tldr] TL;DR
> `submit()` returns instantly with a `Future`. `future.result()` blocks until the work finishes, then **returns the value or re-raises the exception** that happened inside the worker. Threads for I/O, processes for CPU, and always pass a `timeout`.

---
## 💡 Future vs thread vs coroutine vs `asyncio.Future`

| | What it is | Who runs it | How you get the result |
| --- | --- | --- | --- |
| `threading.Thread` | one OS thread you start and join | the OS scheduler | you don't — you write results into shared state yourself |
| `concurrent.futures.Future` | a **result container** with a state machine | a worker in an executor pool | `f.result()` / `f.exception()`, blocking |
| `asyncio.Task` / coroutine | a function that can suspend at `await` | the event loop, **one thread** | `await coro`, cooperatively |
| `asyncio.Future` | the same idea, loop-aware | the event loop | `await fut` — **never** call `.result()` and block a loop |

> [!IMPORTANT]
> The two `Future` classes are different types with a deliberately similar API. `concurrent.futures.Future` is **thread-safe and blocking**; `asyncio.Future` is **not thread-safe and must not be blocked on**. Bridge them with `asyncio.wrap_future()` (concurrent → asyncio) or `loop.run_in_executor()` (which hands you an awaitable backed by a thread pool).

---
## ⚙️ The state machine

```
PENDING ──submit──▶ RUNNING ──▶ FINISHED   (result or exception set)
   │
   └──cancel()──▶ CANCELLED
```

- `cancel()` succeeds **only** while the future is `PENDING`. Once a worker picked the callable up, cancellation is impossible — there is no way to interrupt a running Python function from outside.
- `result(timeout=None)` blocks. On timeout it raises `TimeoutError` **without stopping the work** — the worker keeps going, you simply stopped waiting.
- `exception()` returns the raised exception instead of re-raising it.
- `add_done_callback(fn)` runs `fn(future)` in whichever thread finished the work (or immediately, in the calling thread, if it is already done). Callbacks that raise are logged and swallowed.

---
## 🧵 ThreadPool vs ProcessPool — the GIL decides

| | `ThreadPoolExecutor` | `ProcessPoolExecutor` |
| --- | --- | --- |
| Good for | **I/O-bound** work: HTTP calls, DB queries, disk, `subprocess` | **CPU-bound** work: parsing, compression, image/crypto math |
| Why | the GIL is released around I/O syscalls, so threads overlap | each process has its **own** interpreter and its own GIL |
| Cost per worker | a few KB of stack, cheap | a full interpreter: tens of MB and a startup delay |
| Arguments & results | passed by reference, zero copy | **pickled** and shipped over a pipe — must be picklable |
| Shared state | the same objects; you own the locking | none; use `multiprocessing` primitives or a queue/DB |
| Failure mode | one task can corrupt shared state | a segfaulting worker raises `BrokenProcessPool` for everyone |
| Default `max_workers` | `min(32, os.cpu_count() + 4)` | `os.cpu_count()` (and it is capped at 61 on Windows) |

> [!NOTE]
> Free-threaded CPython (3.13+, the `t` builds) removes the GIL and makes threads genuinely parallel for CPU work — but only if every C extension you use supports it. Until your whole dependency tree is ready, keep treating "CPU-bound ⇒ processes" as the rule.

> [!WARNING]
> On Linux, `fork` has historically been the default process start method and it is unsafe in a process that already has threads (locks can be inherited mid-hold → a child that deadlocks) or an open DB connection (two processes sharing one socket). Be explicit:
> `ProcessPoolExecutor(mp_context=multiprocessing.get_context("spawn"))`. `spawn` re-imports your module, so everything at import time must be side-effect free and guarded by `if __name__ == "__main__":`.

---
## 🧪 Example — submit, map, as_completed, timeouts, cancellation

```python
import time
from concurrent.futures import (
    ThreadPoolExecutor, ProcessPoolExecutor,
    as_completed, wait, FIRST_EXCEPTION, TimeoutError,
)

import httpx

URLS = [f"https://example.com/api/items/{i}" for i in range(1, 51)]

def fetch(url: str) -> tuple[str, int]:
    # 1) I/O-bound: the GIL is released while the socket waits, so threads overlap
    r = httpx.get(url, timeout=5.0)          # per-task timeout, always
    r.raise_for_status()
    return url, len(r.content)

# 2) A pool is a context manager: __exit__ calls shutdown(wait=True) and blocks
with ThreadPoolExecutor(max_workers=8, thread_name_prefix="fetch") as pool:
    futures = {pool.submit(fetch, u): u for u in URLS}   # map future -> input

    # 3) as_completed yields in COMPLETION order, not submission order
    for fut in as_completed(futures, timeout=60):
        url = futures[fut]
        try:
            _, size = fut.result()           # re-raises whatever fetch() raised
        except httpx.HTTPError as exc:
            print(f"failed {url}: {exc!r}")  # one bad task must not kill the batch
        else:
            print(f"ok {url}: {size} bytes")
```

```python
# 4) map(): same signature as builtin map, but results come back IN ORDER,
#    and the exception is raised when you ITERATE past the failing item.
with ThreadPoolExecutor(max_workers=8) as pool:
    for url, size in pool.map(fetch, URLS, timeout=60):
        ...

# 5) wait(): stop the batch as soon as anything blows up
with ThreadPoolExecutor(max_workers=8) as pool:
    futures = [pool.submit(fetch, u) for u in URLS]
    done, not_done = wait(futures, timeout=30, return_when=FIRST_EXCEPTION)
    for fut in not_done:
        fut.cancel()                      # only cancels the ones still PENDING

# 6) Timeout ≠ stop. The work continues; you just stopped waiting for it.
with ThreadPoolExecutor() as pool:
    fut = pool.submit(time.sleep, 30)
    try:
        fut.result(timeout=1)
    except TimeoutError:
        fut.cancel()                      # returns False — it is already RUNNING
    # ...and __exit__ still blocks ~29s waiting for that worker

# 7) Give up on pending work at shutdown (3.9+)
pool = ThreadPoolExecutor(max_workers=4)
pool.submit(time.sleep, 30)
pool.shutdown(wait=False, cancel_futures=True)

# 8) CPU-bound: processes, explicit start method, module-level function
def crunch(n: int) -> int:
    return sum(i * i for i in range(n))

if __name__ == "__main__":               # required with spawn
    import multiprocessing as mp
    ctx = mp.get_context("spawn")
    with ProcessPoolExecutor(max_workers=4, mp_context=ctx) as pool:
        print(sum(pool.map(crunch, [10_000_000] * 4, chunksize=1)))
```

> [!TIP]
> With `ProcessPoolExecutor.map`, a large `chunksize` is the difference between real speedup and pure IPC overhead: each item costs a pickle round-trip, so batch small tasks (`chunksize=100`) and keep `chunksize=1` only for long ones.

---
## 🚨 The traps

| Trap | What actually happens | Fix |
| --- | --- | --- |
| **Silently swallowed exceptions** | if you never call `result()`/`exception()`, the traceback is stored in the future and nobody ever sees it | always consume every future, even in fire-and-forget loops |
| **`map()` hides the failure** | the exception surfaces only when iteration reaches that item, and the rest is abandoned | use `submit` + `as_completed` when partial failure is normal |
| **`with` block hangs** | `__exit__` = `shutdown(wait=True)`; a stuck task blocks the whole program (and `atexit` at interpreter shutdown) | timeouts *inside* the task, not just on `result()` |
| **`cancel()` does nothing** | a `RUNNING` task cannot be interrupted | make tasks short, or have them poll a `threading.Event` |
| **Pool inside pool** | a worker that submits to its own pool and waits can deadlock the pool with no free worker left | separate pools, or restructure to one level |
| **Unbounded `submit()`** | the work queue is unbounded: 1M submits = 1M queued items in RAM | chunk the input, or bound with a semaphore |
| **Non-picklable payload** | lambdas, closures, DB connections, sockets → `PicklingError` in a process pool | pass plain data, build heavy objects inside the worker |
| **`BrokenProcessPool`** | one worker dies (segfault, OOM-kill) and **every** pending future fails | isolate risky C extensions, cap memory, recreate the pool |
| **Lost logging context** | `contextvars`, request IDs and Django's current-language state do not follow a `submit()` | capture what you need and pass it explicitly |

> [!WARNING]
> A pool is not a rate limiter for the thing you are calling. Eight workers hammering one API is eight concurrent requests forever — add real backoff and honour `429`/`Retry-After`, or you will get your own IP throttled. Related: fan-out like this is a classic source of order-dependent, timing-dependent test failures — see [Flaky Tests](./FlakyTest.md).

---
## ⚖️ `concurrent.futures` vs the alternatives

| Approach | Concurrency model | Survives a restart? | Reach for it when |
| --- | --- | --- | --- |
| `concurrent.futures` pool | OS threads / processes, blocking API | ❌ in-memory only | fan-out inside **one** synchronous call: 50 HTTP calls, 4 CPU crunches |
| `threading` directly | raw threads | ❌ | long-lived background loops, not request-shaped work |
| `asyncio` | one thread, cooperative | ❌ | thousands of concurrent sockets, and your libraries are async |
| `asyncio.to_thread` / `run_in_executor` | thread pool behind an event loop | ❌ | a blocking call (DB driver, `requests`) inside async code |
| **Celery / RQ / Dramatiq** | separate worker processes + broker | ✅ retries, scheduling, visibility | anything that must not be lost: emails, payments, reports, ETL |
| `multiprocessing.Pool` | processes, older API | ❌ | legacy code; `ProcessPoolExecutor` is the modern face of it |

> [!IMPORTANT]
> The dividing line is durability, not speed. If losing the work would be a bug — a receipt not sent, a payment not captured — it belongs in a queue with retries, not in a pool that dies with the process handling the request.

---
## 🔐 Security & production notes

- **Pools are a resource budget.** `max_workers` bounds your own concurrency, but the queue behind it is unbounded — a user-controlled fan-out (`?ids=` with 10 000 entries) turns into 10 000 queued tasks and an OOM. Cap the input size *before* you submit; that cap is a security control, not a nicety.
- **Timeouts are mandatory at both levels.** A timeout on `result()` protects the caller; a timeout inside the task (socket/DB) protects the worker. Only the second one actually frees a worker slot.
- **Threads share everything, including secrets.** A token pulled from [Vault](../DevOps/Vault/README.md) and stored in a module-level global is visible to every task in the process. Pass credentials as arguments with the narrowest lifetime you can manage.
- **Process pools leak more than you think.** With `fork`, the child inherits the parent's whole address space (in-memory secrets included). Passing a secret as a subprocess **argument** makes it visible in `ps`/`/proc` — use the environment or a pipe instead.
- **A worker crash is a denial-of-service primitive.** If untrusted input can reach a C extension that segfaults, one request kills the pool and `BrokenProcessPool` fails every concurrent job. Validate first, isolate the risky parser, and recreate the pool on failure.
- **Log the outcome, not just the launch.** `future.exception()` is where the traceback hides; a pool with no consumer of results is a pool with invisible errors, which is how "it works, sometimes" incidents start.

---
## 🐍 Django / backend tie-in

| Situation | What to do |
| --- | --- |
| Fan-out to several internal APIs in one view | `ThreadPoolExecutor` is fine — bound the list, set per-call timeouts, keep the total under your own request timeout |
| ORM queries inside pool threads | each thread opens **its own** DB connection; call `django.db.close_old_connections()` at the start and end of the task or you leak connections until the pool is exhausted |
| `ATOMIC_REQUESTS` + threads | the request's transaction belongs to the request thread only; work in a pool thread commits **separately** and will not roll back with the view |
| CPU work in a web worker | don't. A `ProcessPoolExecutor` under Gunicorn multiplies memory per worker and competes with request handling — push it to Celery |
| Blocking call inside an async view | `await asyncio.to_thread(blocking_fn, ...)`, or `sync_to_async(fn, thread_sensitive=True)` so Django reuses the right thread for ORM work |
| Emails, webhooks, reports | Celery, not a pool. A pool dies with the process; a queue retries |

> [!WARNING]
> `sync_to_async(..., thread_sensitive=True)` (the default) runs everything in **one** shared executor thread — correct for the ORM, but it serialises your "parallel" calls. Use `thread_sensitive=False` only for code that touches no per-thread Django state.

---
## 🧠 Summary

| Concept | Takeaway |
| --- | --- |
| What a Future is | A thread-safe result container: `PENDING → RUNNING → FINISHED/CANCELLED` |
| Getting the value | `result()` blocks and re-raises the worker's exception — always consume it |
| Threads vs processes | I/O-bound → threads (GIL is released); CPU-bound → processes (own GIL, pickled payloads) |
| Cancellation | Only works while `PENDING`; a running Python function cannot be interrupted |
| Timeouts | `result(timeout=...)` stops *waiting*, never the work — put a timeout in the task too |
| Biggest trap | An unread future: the exception is stored, logged nowhere, and the bug looks intermittent |
| Not a queue | No durability, no retries, no visibility — if losing the task is a bug, use Celery |
