## Fundamentals
### 🔑 Indexing
- B-tree vs Hash vs GIN/GIST indexes
- when they help vs hurt writes
### 🧩 Normalization vs Denormalization
- 1NF/2NF/3NF
- when to break the rules for performance
### 🔗 Joins
- INNER/LEFT/RIGHT/FULL
- N+1 query problem (huge in Django ORM)
### 📊 Query execution plans
- EXPLAIN ANALYZE
- Reading query costs
## Scaling
### 🪓 Sharding vs Partitioning
- horizontal split of data across nodes/tables
### 🔁 Replication
- primary-replica
- sync vs async
- Read replicas
### ⚖️ Load balancing
- reads/writes
## Consistency & correctness
### 🔒 Locking
- optimistic vs pessimistic locking
### 🏃 Race conditions & idempotency keys
### 📐 Referential integrity
- foreign keys
- Cascades
## Data modeling
### 🗃️ SQL vs NoSQL
- document
- Key-value
- columnar
- graph DBs
- when each fits
### ⏰ Schema migrations
- Django migrations
- zero-downtime migration strategies
## Performance
### ⚡ Caching layers
- query cache
- Redis
- Cache invalidation strategies
### 🐌 Connection pooling
- why Django + Postgres needs PgBouncer at scale
## Security
### 💉 SQL injection
- Parameterized queries
- ORM safety nets and their limits
### 🔐 Encryption at rest vs in transit
### 🕵️ Row-level security / multi-tenancy isolation
### 📝 Audit logging
- tracking who changed what