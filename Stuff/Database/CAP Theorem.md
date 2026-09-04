# 🎯 Quick Reference

> [!tldr] TL;DR
> In a **distributed system**, when a **network partition** happens, you can only pick **one**:
> **Consistency** (correct data) OR **Availability** (system keeps responding).
> You can't have both at the same time, during a partition. 🔀

---

## 🤔 Why this exists

Any distributed database runs on multiple nodes across a network. Networks fail — cables get cut, servers can't reach each other. That's a **partition**.

CAP theorem asks: **when nodes can't talk to each other, what do you sacrifice?**

---

## 🧬 The three letters

| Letter | Property | Meaning |
|---|---|---|
| **C** | Consistency | Every read gets the **most recent write** (or an error) — all nodes agree |
| **A** | Availability | Every request gets **a response** (not an error) — even if it's not the latest data |
| **P** | Partition Tolerance | System keeps working even if network communication between nodes **breaks** |

> [!important] The real rule
> **Partition tolerance is not optional.** Networks *will* fail. So realistically you're choosing between:
> **CP** (consistent, but may refuse requests during a partition)
> **AP** (available, but may return stale data during a partition)

---

## ⚖️ CP vs AP — what you actually choose

|                               | 🔒 CP (Consistency + Partition tolerance)                                    | 🌐 AP (Availability + Partition tolerance) |
| ----------------------------- | ---------------------------------------------------------------------------- | ------------------------------------------ |
| **Behavior during partition** | Refuses/blocks requests to stay correct                                      | Keeps responding, may serve stale data     |
| **Guarantees**                | No stale reads, ever                                                         | System never fully down                    |
| **Tradeoff**                  | Some downtime                                                                | Possible inconsistency                     |
| **Good for**                  | Banking, inventory, bookings                                                 | Social feeds, likes/views, caching, DNS    |
| **Examples**                  | MongoDB (default), HBase, Zookeeper, traditional RDBMS with sync replication | Cassandra, DynamoDB, Couchbase, Riak       |

> [!example] Simple analogy
> **CP** = "I'd rather tell you 'try again later' than give you wrong info."
> **AP** = "I'd rather give you *an* answer, even if slightly outdated, than nothing."

---

## 🧠 Common misconceptions

- ❌ "You always pick 2 of 3" — misleading. **P is mandatory** in any real distributed system; the actual choice is C vs A *during a partition only*.
- ❌ "CAP applies all the time" — No. **When there's no partition, you can have both C and A.** CAP tradeoffs only kick in *during* a network failure.
- ❌ "NoSQL = AP, SQL = CP" — not a strict rule. Many systems let you **tune** this (e.g., Cassandra can be configured more toward consistency with quorum reads/writes).

---

## 🔗 Related: PACELC (the more complete picture)

CAP only describes behavior **during a partition**. PACELC extends it:

> **If Partition (P) → choose A or C. Else (E, no partition) → choose Latency (L) or Consistency (C).**

| System | During partition | Normal operation |
|---|---|---|
| DynamoDB, Cassandra | A (available) | L (low latency, eventual consistency) |
| MongoDB, HBase | C (consistent) | C (consistency over latency) |

> [!tip] Why this matters more
> Most of the time your system is **not** partitioned — so the everyday tradeoff is actually **latency vs consistency**, not availability vs consistency. PACELC captures that.

---

## 🐍 Where this shows up in Django/backend work

- **Postgres (single primary)** → effectively CP: if the primary is unreachable, writes fail rather than risk inconsistency
- **Postgres with async replication (read replicas)** → reads from replicas can be **stale** (eventual consistency) — classic AP-ish tradeoff for reads
- **Redis** (single instance) → CP-ish; Redis Cluster can lose writes during partition (favors availability)
- **Celery + broker (RabbitMQ/Kafka)** → task delivery guarantees depend on broker's CAP choice; know if "at-least-once" or "exactly-once" is promised
- **Multi-region deployments** → this is where CAP becomes very real — a user in Europe and one in the US may briefly see different data

---

## 🔐 Security-relevant notes

- **Stale reads (AP systems)** can create security gaps: e.g., a revoked permission or banned user might still appear "active" on a node that hasn't synced yet.
- **Session/auth data** is usually security-critical → lean **CP** (or use a single source of truth like Redis/Postgres) rather than an eventually-consistent store.
- **Rate limiting / fraud detection** across distributed nodes can under-count or over-count during partitions — design for the failure mode you're choosing.
- Systems that choose **A over C** need compensating logic (e.g., re-validate critical actions like payments or permission checks against the source of truth, not a cache).

---

## 🧭 Decision cheat sheet

```
Does stale/wrong data cause real harm (money, security, safety)?
├── YES → lean CP — reject requests rather than risk wrong answers
└── NO
    │
    Does the system need to stay responsive no matter what
    (social feed, analytics, view counts)?
    ├── YES → lean AP — eventual consistency is fine
    └── NO → default to whatever your DB gives you (usually CP-ish)
```

---

## 📚 Quick glossary

| Term | Meaning |
|---|---|
| Partition | Network failure preventing nodes from communicating |
| Consistency | All nodes see the same (latest) data |
| Availability | System always responds (even if data isn't freshest) |
| Eventual consistency | Data converges to correct state *eventually*, not immediately |
| Quorum | Minimum number of nodes that must agree for a read/write to succeed |
| Split-brain | Both sides of a partition think they're the "real" primary |

> [!tip] Rule of thumb
> If it's about **money, security, or uniqueness** (usernames, inventory) → **CP**.
> If it's about **scale and uptime** (feeds, counters, caches) → **AP**.
> 