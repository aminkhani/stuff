# 📨 Quick Reference

> [!tldr] TL;DR
> - **RabbitMQ** = smart broker, simple routing, great for **task queues / job processing** 🐰
> - **Kafka** = distributed log, built for **high-throughput event streaming & replay** 🔥
> - Both solve the same core problem: **decoupling services** so they don't talk to each other directly.

---

## 🤔 Why use a message broker at all?

Instead of Service A calling Service B directly (tight coupling, both must be up), you send a message through a broker:

- ✅ **Decoupling** — producer and consumer don't need to know about each other
- ✅ **Async processing** — don't block the request/response cycle (e.g. send email, generate report)
- ✅ **Load leveling** — absorb traffic spikes, consumers work at their own pace
- ✅ **Reliability** — retry failed jobs, don't lose data if a service is down
- ✅ **Scalability** — add more consumers to process faster

> [!example] Django context
> Instead of `send_email()` blocking your API response, you publish an `"email.send"` event/task and a worker (Celery, or a custom consumer) handles it in the background.

---

## 🐰 RabbitMQ — the smart broker

**What it is:** A traditional **message broker** implementing AMQP. It pushes messages to consumers and tracks delivery.

### Core concepts you MUST know
- 📤 **Producer** — sends messages
- 📥 **Consumer** — receives messages
- 📦 **Queue** — buffer that holds messages until consumed
- 🔀 **Exchange** — receives messages from producers and **routes** them to queues based on rules
  - `direct` — exact routing key match
  - `topic` — pattern matching (`order.*`)
  - `fanout` — broadcast to all bound queues
  - `headers` — route by message headers
- 🔗 **Binding** — the rule connecting an exchange to a queue
- 🔑 **Routing key** — label used to decide where a message goes
- ✅ **ACK / NACK** — consumer confirms message was processed (or not) → enables retry
- 💀 **DLX (Dead Letter Exchange)** — where failed/rejected messages go

### When to use RabbitMQ
- Background job processing (emails, PDF generation, notifications)
- Complex routing logic needed (send to different queues based on type/priority)
- Task queues with **Celery** (very common in Django!)
- Need low-latency, per-message guaranteed delivery
- RPC-style request/reply patterns

---

## 🔥 Kafka — the distributed log

**What it is:** Not really a "queue" — it's a **distributed, append-only log**. Consumers *read* from it, messages aren't removed after consumption.

### Core concepts you MUST know
- 📝 **Topic** — named stream of events (like a table/category)
- 🧩 **Partition** — a topic is split into partitions for parallelism & scale
- 🔢 **Offset** — each message's position in a partition (consumers track their own offset)
- 👥 **Consumer Group** — multiple consumers share the work of a topic (each partition read by only 1 consumer in the group)
- 🖥️ **Broker** — a Kafka server; a cluster has many brokers
- 🔁 **Replication factor** — how many copies of each partition exist (fault tolerance)
- 🗝️ **Producer key** — determines which partition a message lands in (keeps order per key, e.g. per `user_id`)
- ⏳ **Retention** — messages stay for a configured time (hours/days/forever), **not deleted on read** → replay is possible!
- 🧠 **Zookeeper / KRaft** — cluster coordination (modern Kafka uses KRaft, no more Zookeeper)

### When to use Kafka
- High-throughput event streaming (millions of events/sec)
- Event sourcing / audit logs — need to **replay** history
- Multiple independent consumers reading the same stream
- Log aggregation, analytics pipelines, metrics
- Microservices architecture with many services reacting to the same events
- Need strict ordering **within a key** (e.g., all events for one user in order)

---

## ⚖️ Side-by-side comparison

| Aspect | 🐰 RabbitMQ | 🔥 Kafka |
|---|---|---|
| **Model** | Smart broker, push-based | Distributed log, pull-based |
| **Message lifecycle** | Deleted after ACK | Retained per policy, replayable |
| **Throughput** | Good (thousands/sec) | Excellent (millions/sec) |
| **Latency** | Very low, per-message | Low, but batched for throughput |
| **Ordering** | Per-queue | Per-partition (per key) |
| **Routing** | Rich (exchanges/routing keys) | Simple (topic + partition key) |
| **Replay messages** | ❌ No (once consumed, gone) | ✅ Yes (reset offset) |
| **Multiple consumers, same data** | Needs fanout exchange + separate queues | Native via consumer groups |
| **Best for** | Task queues, RPC, job processing | Event streaming, event sourcing, analytics |
| **Django ecosystem** | `Celery` + `pika`/`kombu` | `confluent-kafka-python`, `aiokafka` |
| **Ops complexity** | Simpler to run | Heavier (cluster, partitions, retention tuning) |

---

## 🧭 Decision cheat sheet

```
Do you need to REPLAY old messages / keep full history?
├── YES → Kafka
└── NO
    │
    Do you need complex routing (priority queues, RPC, fanout)?
    ├── YES → RabbitMQ
    └── NO
        │
        Do you need millions of events/sec across many consumers?
        ├── YES → Kafka
        └── NO → RabbitMQ (simpler, good enough)
```

---

## 🔐 Security-relevant notes (since you work in security)

- Both support **TLS** for transport encryption — always enable in production.
- **RabbitMQ**: uses SASL/AMQP auth, supports per-vhost user permissions, plugin for OAuth2.
- **Kafka**: supports **SASL/SCRAM**, **mTLS**, and **ACLs** per topic — lock down who can produce/consume which topics.
- Watch for **unauthenticated management UIs** (RabbitMQ's 15672 port is a common misconfig target).
- Kafka's retention means **sensitive data can persist longer than expected** — think about PII in event payloads and retention policy.

---

## 📚 Quick glossary

| Term | Meaning |
|---|---|
| Producer | Sends messages |
| Consumer | Receives/processes messages |
| Broker | The server(s) running the messaging system |
| Ack | Confirmation a message was handled |
| Idempotency | Processing the same message twice has no bad side-effect (design for this!) |
| Backpressure | What happens when consumers can't keep up with producers |

> [!tip] Rule of thumb
> **"Do I need a to-do list?" → RabbitMQ.**
> **"Do I need a history book?" → Kafka.**
> 