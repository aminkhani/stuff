## 🧭 The Two Files at a Glance

|File|Controls|Analogy|
|---|---|---|
|⚙️ `postgresql.conf`|Server behavior: memory, logging, connections, replication|The engine settings 🔧|
|🚪 `pg_hba.conf`|Who can connect, from where, using which auth method|The bouncer at the door 🕴️|

Both live in the **data directory** by default (find it with `SHOW config_file;` and `SHOW hba_file;` inside `psql`), though packaged installs (e.g. Debian/Ubuntu `apt` PostgreSQL) often put them in `/etc/postgresql/<version>/main/`.

Changes to most `postgresql.conf` settings need a **reload** (`SELECT pg_reload_conf();` or `systemctl reload postgresql`); a few (like `shared_buffers`) need a full **restart**. `pg_hba.conf` changes only need a **reload**.

---
## ⚙️ postgresql.conf

### 🔌 Connection Settings

```ini
listen_addresses = 'localhost'   # or specific IPs, or '*' for all (⚠️ see security section)
port = 5432
max_connections = 100
```

> [!warning] `listen_addresses = '*'` Only bind to all interfaces if you truly need remote connections AND have `pg_hba.conf` + firewall locked down. Otherwise keep it to `localhost` or specific private IPs.

### 🧠 Memory Settings

```ini
shared_buffers = 256MB        # ~25% of system RAM, needs RESTART to change
work_mem = 4MB                # per sort/hash operation, per connection
maintenance_work_mem = 64MB   # for VACUUM, CREATE INDEX, etc.
effective_cache_size = 768MB  # hint to planner, not actual allocation
```

> [!tip] Sizing rule of thumb `shared_buffers` ≈ 25% of RAM. `effective_cache_size` ≈ 50–75% of RAM (tells the planner how much OS cache is likely available).

### 📝 Logging Settings

```ini
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_min_duration_statement = 500   # log queries slower than 500ms
log_connections = on
log_disconnections = on
log_line_prefix = '%m [%p] %u@%d '  # timestamp, pid, user@db
```

### 🔄 WAL / Replication (relevant for backups & HA)

```ini
wal_level = replica          # 'replica' or 'logical' for logical replication
max_wal_senders = 5
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'
```

### 🧵 Autovacuum (keeps table bloat under control)

```ini
autovacuum = on
autovacuum_naptime = 1min
autovacuum_vacuum_scale_factor = 0.2
```

> [!warning] Never disable autovacuum in production Turning it off "temporarily" is a common cause of runaway table bloat and transaction ID wraparound emergencies later.

---
## 🚪 pg_hba.conf

HBA = **Host-Based Authentication**. Each line is a rule; PostgreSQL evaluates them **top to bottom** and uses the **first match**.

### 📐 Line Format

```
TYPE      DATABASE   USER     ADDRESS         METHOD
```

```ini
# TYPE   DATABASE  USER      ADDRESS          METHOD
local    all       postgres                   peer
local    all       all                        scram-sha-256
host     all       all       127.0.0.1/32     scram-sha-256
host     all       all       ::1/128          scram-sha-256
host     mydb      app_user  10.0.0.0/24      scram-sha-256
hostssl  mydb      app_user  0.0.0.0/0        scram-sha-256
```

### 🏷️ TYPE column

- `local` → Unix domain socket (no network, same machine only)
- `host` → TCP/IP, SSL or non-SSL
- `hostssl` → TCP/IP, **SSL required** ✅
- `hostnossl` → TCP/IP, SSL forbidden (rarely used deliberately)

### 🎯 DATABASE / USER columns

- `all` → matches everything
- Specific name → `mydb`, `app_user`
- Comma-separated list or `@file` for groups: `+admins`

### 🌍 ADDRESS column

- CIDR notation: `10.0.0.0/24`, `192.168.1.5/32`
- `0.0.0.0/0` → **any IPv4 address** (⚠️ use with `hostssl` + strong auth only, ideally never in production without a VPN/firewall in front)
- `samehost` / `samenet` → dynamic matches

---
## 🔐 Authentication Methods in pg_hba

|Method|What it does|Security|
|---|---|---|
|`trust`|No password at all|🔴 Never use except local dev on a trusted single-user machine|
|`peer`|Matches OS username to Postgres username (local only)|🟢 Good for local admin/service accounts|
|`password`|Plaintext password over the wire|🔴 Only acceptable inside `hostssl` + trusted network|
|`md5`|Legacy hashed password|🟡 Deprecated — vulnerable to replay in some scenarios, prefer scram-sha-256|
|`scram-sha-256`|Modern salted challenge-response hash|🟢 **Recommended default** for password auth|
|`cert`|Client SSL certificate required|🟢 Strongest — good for service-to-service/machine auth|
|`ldap` / `radius` / `gss` (Kerberos)|External identity provider|🟢 Good for enterprise SSO/centralized auth|

> [!tip] Always prefer `scram-sha-256` over `md5` Set `password_encryption = scram-sha-256` in `postgresql.conf` so newly created/changed passwords use the stronger hash.

---
## 🛡️ Security Best Practices

### 1. 🔒 Never use `trust` in production

`trust` means "if you can connect, you're in" — no password check at all.
### 2. 🌐 Don't expose Postgres directly to the internet

- Bind `listen_addresses` to private/internal IPs only
- Put it behind a VPN, SSH tunnel, or private VPC network
- If remote access is unavoidable, force `hostssl` + `scram-sha-256` + IP allow-listing
### 3. 🔑 Enforce SSL/TLS

```ini
# postgresql.conf
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
```

```ini
# pg_hba.conf — reject non-SSL connections entirely
hostssl all all 0.0.0.0/0 scram-sha-256
```
### 4. 👤 Principle of least privilege

- Don't let application users connect as `postgres` (the superuser)
- Create scoped roles per app/service:

```sql
CREATE ROLE app_user WITH LOGIN PASSWORD 'x' NOSUPERUSER NOCREATEDB NOCREATEROLE;
GRANT CONNECT ON DATABASE mydb TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
```
### 5. 🎯 Be as specific as possible in pg_hba rules

Prefer:

```ini
host  mydb  app_user  10.0.5.0/24  scram-sha-256
```
over:
```ini
host  all   all       0.0.0.0/0    scram-sha-256
```

Narrow database, user, and address on every line — never rely on a single broad catch-all rule.
### 6. 📝 Log connections and slow queries

Already shown above (`log_connections`, `log_min_duration_statement`) — critical for detecting brute-force attempts or abnormal access patterns.
### 7. 🚫 Rate-limit and firewall at the network layer too

`pg_hba.conf` doesn't rate-limit failed logins — pair it with `fail2ban`, a security group/firewall, or a connection pooler (`pgbouncer`) with its own limits.
### 8. 🔁 Rotate credentials & use connection poolers for secrets hygiene

Use a secrets manager (Vault, AWS Secrets Manager, etc.) rather than plaintext passwords in `.env` files committed anywhere.
### 9. 🧱 Separate roles for humans vs services

Human DBA logins → `cert` or SSO (`ldap`/`gss`); service/app accounts → `scram-sha-256` + certificate pinning where possible.

---
## 📊 PostGIS-Specific Notes

- PostGIS doesn't change `pg_hba.conf` behavior — it's just an extension inside a normal database, so all the same auth rules apply per-database.
- Spatial queries (ST_Intersects, ST_Union, etc. used in merge/split/bulk geometry operations) can be memory- and CPU-heavy — tune `work_mem` up for connections/roles that run heavy spatial joins:

```sql
ALTER ROLE gis_worker SET work_mem = '64MB';
```

- If GeoServer connects as a separate service account from your Django app, give it its **own** `pg_hba.conf` line and role, scoped only to the schemas/tables it actually needs — don't reuse the Django app's superuser-ish credentials.
- Large geometry bulk updates benefit from a higher `maintenance_work_mem` during index rebuilds (`REINDEX`, `CREATE INDEX CONCURRENTLY` on spatial indexes).

---
## ⚠️ Common Pitfalls

- 🚨 Leaving the default `trust` rule for `local all` on a multi-user server — any local OS user can then connect as any Postgres role.
- 🚨 Editing `pg_hba.conf` but forgetting to `SELECT pg_reload_conf();` — old rules stay active until reload.
- 🚨 Using `0.0.0.0/0` "temporarily" for debugging and forgetting to remove it.
- 🚨 Setting `shared_buffers` too high (>40% RAM) — can starve OS page cache and hurt performance.
- 🚨 Forgetting rule order matters — a broad `all/all` rule placed before a specific one will shadow it since Postgres uses **first match wins**.
- 🚨 Using `md5` because "it's what the tutorial showed" — it's legacy; use `scram-sha-256`.

---
## ✅ Best Practices Checklist

- [ ] `listen_addresses` restricted to necessary interfaces only
- [ ] No `trust` rules outside isolated local dev
- [ ] `scram-sha-256` used everywhere (not `md5`, not `password`)
- [ ] `hostssl` enforced for any non-local connection
- [ ] `pg_hba.conf` rules as narrow as possible (specific DB, user, CIDR — not `all`/`all`/`0.0.0.0/0`)
- [ ] Application roles are non-superuser, least-privilege
- [ ] Separate roles per service (Django app vs GeoServer vs admin)
- [ ] `log_connections`, `log_disconnections`, `log_min_duration_statement` enabled
- [ ] Autovacuum left on and tuned, not disabled
- [ ] Secrets stored in a secrets manager, not committed config files
- [ ] Firewall/security groups restrict port 5432 regardless of `pg_hba.conf`