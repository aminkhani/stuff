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
## 🧩 What is HashiCorp Vault?

**Vault** is an identity-aware **secrets manager**: one HTTP API that keeps secrets encrypted at rest, **issues short-lived credentials on demand**, and records every access in an **audit log**. Engines, auth methods, secrets and system config are all just **paths** (`secret/myapp/db`, `database/creds/app`) that a policy either allows or denies. Think **hotel front desk 🏨** instead of a shared house key: you show ID, you get a keycard that opens one room and expires at checkout, and the desk keeps a record of every card it ever issued.
---
## 💡 Why do we need it? — secret sprawl & secret zero

- 📄 **`.env` and CI-variable sprawl** — the same DB password sits on six laptops, two CI runners and in a Slack thread. Nobody knows who holds it, so nobody dares rotate it. CI variables are visible to every job, sometimes to fork/PR builds.
- 🧊 **Static credentials** — one leak is a permanent compromise until a human happens to notice.
- 🙈 **A Kubernetes `Secret` is base64, not encryption** — anyone with `get secret` in the namespace (or raw etcd access) reads plaintext.
- 🧾 **No audit trail** — "who read the production DB password last month?" is unanswerable when secrets are files.
- 🥚 **Secret zero** — even *with* a manager, the app needs one credential to reach it. Vault's answer is to make secret zero a **machine identity** (Kubernetes service-account JWT, cloud IAM, single-use AppRole `secret_id`) instead of yet another static password.
---
## ⚙️ How it works

```
auth (approle | k8s | oidc) --> token{ policies, TTL } --> request a path
+----------- BARRIER: everything below is stored encrypted ------------+
| encryption key <-- unwrapped by master key <-- Shamir shares / KMS   |
| mounts:  secret/  database/  pki/  transit/  auth/  sys/             |
+-- persisted to storage backend: Raft (integrated) or Consul ---------+
```

1. **Storage backend** only ever sees ciphertext. **Integrated Storage (Raft)** is today's default: data on each node's own disk, HA through Raft consensus, one system to operate. **Consul** still works, but you then run and back up two clusters.
2. **Barrier** — every write is encrypted with the **encryption key**, which is itself encrypted by the **master key**. A **sealed** Vault holds the ciphertext and no way to read it; it answers nothing except `sys/health` and `sys/seal-status`.
3. **Unseal** rebuilds the master key. `vault operator init` uses **Shamir's Secret Sharing** to split it into N shares with threshold K (default 5/3), so K different people must paste a share after *every* restart. **Auto-unseal** delegates that to cloud KMS, an HSM, or a Transit mount on another Vault — effectively mandatory in production, because pods restart at 03:00.
4. **Root token** comes out of `init`: unlimited, policy-free, no expiry. Save the unseal/recovery shares, then **revoke it** (`vault token revoke <root>`) and log in as a human through OIDC or userpass. Regenerate it only for break-glass with `vault operator generate-root`, which needs a quorum of shares.
5. **Tokens** carry policies and a **TTL**; child tokens die with their parent. **Leases** wrap dynamic secrets — `vault lease renew` extends, `vault lease revoke` kills early, and at expiry **Vault destroys the underlying credential** (drops the DB user, revokes the cert). "My password suddenly stopped working" becomes a normal event your app must handle.
---
## 🗄️ Secrets engines — and why you'd pick each

| Engine | Gives you | Pick it when |
|---|---|---|
| **KV v1** | Flat key/value, no history | You want the simplest possible store and don't mind that `put` overwrites forever |
| **KV v2** | Versioning, soft `delete`/`undelete`, `destroy`, **CAS** writes | Default for hand-managed secrets: you can roll back a bad rotation, and `-cas=` stops two pipelines clobbering each other |
| **database** | A **fresh DB user per request**, dropped again at lease expiry | The single biggest win — no shared app password, and a stolen credential dies within the hour |
| **pki** | Short-lived internal x509 certs from a Vault-hosted CA | You want mTLS between services without the manual CSR ritual → [TransportSecurity](../../Security/TransportSecurity.md) |
| **transit** | Encrypt / decrypt / sign as a service; **the key never leaves Vault** | Column-level encryption where your app only ever holds ciphertext; rotate the key and `rewrap` without a redeploy |
| **ssh** / **totp** | Signed SSH certs or one-time passwords; TOTP codes | Retiring `authorized_keys` sprawl, or sharing a 2FA seed without a phone photo of a QR code |
| **aws / azure / gcp** | Dynamic cloud IAM credentials | A job should get temporary access keys instead of a long-lived IAM user |
---
## 🔑 Auth methods — how you *get* a token

| Method | Proof of identity | Use for |
|---|---|---|
| **token** | The token itself | Bootstrap, break-glass, CLI after `vault login` |
| **AppRole** | `role_id` (ship it in config) + `secret_id` (the secret half) | **The standard for services** outside Kubernetes and cloud IAM |
| **Kubernetes** / **AWS · Azure IAM** | Pod's service-account JWT verified via the API server, or signed instance/role metadata | Cluster and cloud workloads — there is no secret to distribute at all |
| **JWT / OIDC** | A signed ID token from your IdP | Humans via SSO, plus CI runners using OIDC. Exactly the machinery in [OAuth](../../Django/OAuth.md): Vault is the relying party, validates the ID token, then maps a claim (`groups`, `sub`) onto policies |
| **userpass / LDAP** | Password or directory bind | Humans in small or on-prem setups |

> [!TIP]
> **Response wrapping** solves AppRole's chicken-and-egg problem: a trusted orchestrator requests the `secret_id` with `-wrap-ttl=120s` and receives only a single-use **wrapping token**, which the app exchanges via `vault unwrap`. If anyone intercepts it, the app's unwrap fails loudly — you get a detectable incident instead of a silent leak.
---
## 📜 Policies — default deny, path + capabilities

Capabilities are `create read update delete list sudo deny`, and `deny` beats everything else. A path with no matching rule is **403** — Vault is default-deny. In globs, `+` matches exactly one path segment (`secret/data/+/db`) while `*` is a trailing wildcard for the rest of the path.

```hcl
# policy "myapp" -- anything not listed here is denied
path "secret/data/myapp/*"     { capabilities = ["read"] }
path "secret/metadata/myapp/*" { capabilities = ["read", "list"] }  # KV v2 splits data/ and metadata/
path "database/creds/app"      { capabilities = ["read"] }          # this read mints a new DB user
path "transit/encrypt/pii"     { capabilities = ["update"] }        # encrypting is a write operation
path "secret/data/users/{{identity.entity.name}}/*" {               # templated: own subtree only
  capabilities = ["create", "read", "update", "delete"]
}
```
---
## 🧪 Examples

> [!WARNING]
> `vault server -dev` is **in-memory, auto-unsealed, plain HTTP and root-tokened**, and loses everything on restart. It exists for learning and tests. It is not a "small production" mode — nothing real should ever point at it.

```bash
vault server -dev -dev-root-token-id=root                      # terminal 1
export VAULT_ADDR='http://127.0.0.1:8200' VAULT_TOKEN='root'   # terminal 2
vault status                                                   # Sealed  false
```

**KV v2** — versioned static secrets:
```bash
vault secrets enable -path=secret kv-v2
vault kv put secret/myapp/db user=app password='s3cr3t'
vault kv put -cas=1 secret/myapp/db password='rotated'   # fails unless the current version is 1
vault kv get -field=password secret/myapp/db             # rotated   (-version=1 gets the old one)
```

**Database engine, end to end (PostgreSQL)** — the part worth internalising:
```bash
vault secrets enable database
vault write database/config/postgres \
  plugin_name=postgresql-database-plugin allowed_roles="app" \
  connection_url="postgresql://{{username}}:{{password}}@pg.internal:5432/appdb?sslmode=verify-full" \
  username="vault_root" password="bootstrap-pw"
vault write -f database/rotate-root/postgres      # from now on nobody knows that password
vault write database/roles/app db_name=postgres default_ttl=1h max_ttl=24h \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  revocation_statements="DROP ROLE IF EXISTS \"{{name}}\";"
vault read database/creds/app
# lease_id     database/creds/app/8Kx1pQ...     lease_duration  1h (renewable up to max_ttl)
# username     v-approle-app-7bFq2xE1
# password     A1a-9nR4ndomlyGener4ted
```

What that did inside Postgres: Vault connected as `vault_root`, ran your `creation_statements` to **`CREATE ROLE` a brand-new user** with a generated password and `VALID UNTIL` the lease end, granted it exactly those privileges, and handed you the pair. At expiry — or on `vault lease revoke <lease_id>` — it reconnects and runs `revocation_statements` to `DROP ROLE`. Two replicas get two *different* users, so the audit log shows which credential leaked, and revoking one doesn't disturb the other.

**AppRole login** — how a service actually obtains a token:
```bash
vault auth enable approle
vault write auth/approle/role/myapp token_policies="myapp" token_ttl=20m token_max_ttl=1h secret_id_num_uses=1
vault read  auth/approle/role/myapp/role-id        # role_id    6a1f...
vault write -f auth/approle/role/myapp/secret-id   # secret_id  9d2c...  (add -wrap-ttl=120s)
vault write auth/approle/login role_id=6a1f... secret_id=9d2c...
# auth.client_token   hvs.CAESIJ...   -> send it as the X-Vault-Token header
```
---
## 🚨 When NOT to use it

- ❌ **A solo project or a single VM** — SOPS + age, or your cloud's secret manager, is ten minutes of work instead of a cluster to babysit.
- ❌ **Nobody owns the operational burden** — unseal, HA quorum, upgrades, Raft snapshots. An unmaintained Vault is a single point of failure in front of *every* service.
- ❌ **As a general-purpose database or blob store** — it's a secret store: no rich queries, every read is an audited encrypted round trip. Credentials and keys, not archives, dumps or images.
- ❌ **Per-request secret reads with no caching** — you have just put Vault on your p99 latency path and invented your own rate limit.
---
## ⚖️ Vault vs the alternatives

| Option | Dynamic creds | Encryption as a service | Audit | Ops cost | On-prem / air-gapped | Pick it when |
|---|---|---|---|---|---|---|
| **Vault** | ✅ DB, cloud, SSH, PKI | ✅ Transit | ✅ per request | 🔴 High: seal, HA, backups, upgrades | ✅ | Many services, real rotation requirements, mixed or on-prem infra |
| **AWS Secrets Manager / Azure Key Vault** | 🟡 Rotation via Lambda/policy | 🟡 Separate KMS service | ✅ CloudTrail | 🟢 None (managed) | ❌ | You are all-in on one cloud and want zero operational load |
| **Kubernetes `Secret`** | ❌ | ❌ | 🟡 API audit only | 🟢 Built in | ✅ | Only as a *delivery* mechanism, fed by a real store or the CSI driver |
| **SOPS + age / Sealed Secrets** | ❌ | ❌ | 🟡 Git history | 🟢 Low | ✅ | GitOps and small teams: encrypted secrets live in the repo, decrypted at deploy |
| **Infisical / Doppler** | 🟡 Limited | ❌ | ✅ | 🟢 Low (SaaS) | 🟡 Self-host | You want a good UI and `.env` sync without running Vault yourself |
| **plain `.env`** | ❌ | ❌ | ❌ | 🟢 Zero | ✅ | Local development, and nothing else |

The honest verdict: Vault is the most capable option here and by a wide margin the **most operationally heavy**. Dynamic credentials, Transit and internal PKI are things nothing else on this list does as well — but you pay for them with unseal procedures, HA quorum, version upgrades and rehearsed Raft snapshot restores. If dynamic creds and encryption-as-a-service are not on your roadmap, a managed secret manager or SOPS is the better engineering trade.
---
## 🔐 Security notes & production hardening

- **Never `-dev`.** Run a real config file: `storage "raft"` plus a `listener "tcp"` on `:8200` with `tls_cert_file` / `tls_key_file` (a real cert, not self-signed) and `tls_min_version = "tls12"`. Keep it on a private subnet — Vault should never have a public listener ([DMZ](../../Linux/DMZ.md)).
- **Auto-unseal** through cloud KMS, an HSM, or a Transit mount on a second Vault. Split the **Shamir / recovery shares across different people**, stored offline — never all five in one password-manager entry.
- **Revoke the initial root token** once setup is done. `vault operator generate-root` is break-glass only, and it needs a quorum of shares, which is exactly the point.
- **Enable an audit device before go-live**: `vault audit enable file file_path=/vault/logs/audit.log` (mode `0600`, or use `syslog`). Vault **refuses requests when all audit devices fail** — fail-closed by design, so monitor that disk.
- **Least privilege per app**: one policy per service, no wildcard `secret/*` reads, no shared policies. Short `token_ttl` and `default_ttl` with *actual* renewal, instead of quietly setting `720h`.
- **`disable_mlock`** — Vault normally locks memory so secrets cannot be swapped to disk. Disabling it (often required in containers/K8s) means swap may contain plaintext; if you must disable it, disable swap on the host as well.
- **Back up Raft**: `vault operator raft snapshot save vault.snap` on a schedule, encrypted at rest, with recovery/unseal keys stored **separately** from the snapshot — together they are a complete compromise. Rehearse the restore before you need it.
- **Rotate the DB root credential** with `vault write -f database/rotate-root/postgres` so that no human, including you, knows it. Monitor seal status, unsealed-node count, token/lease counts and audit failures → [Monitoring & Logging](../Monitoring&Logging/README.md).

> [!NOTE]
> On Windows, `vault.exe` is fine for learning — set `$env:VAULT_ADDR="https://127.0.0.1:8200"` in PowerShell before any command. There is no systemd to supervise it, so for anything resembling a real setup run Vault in a container or under WSL2 ([Docker](../Docker/README.md)).
---
## 🐍 Django / Backend tie-in

Use the **`hvac`** client, authenticate with **AppRole at startup** (never a static `VAULT_TOKEN` in `settings.py`), and **cache what you fetch for the lifetime of the lease** — Vault is not something you call per request.
```python
import hvac, os
from django.db import connections

def fetch_db_creds():                      # call once at startup, then cache the lease
    c = hvac.Client(url=os.environ["VAULT_ADDR"])           # TLS is verified by default
    c.auth.approle.login(role_id=os.environ["VAULT_ROLE_ID"], secret_id=os.environ["VAULT_SECRET_ID"])
    lease = c.secrets.database.generate_credentials(name="app")
    return lease["data"], lease["lease_id"]  # renew: c.sys.renew_lease(lease_id, increment=3600)

def rebind(creds):                         # renewal failed -> swap in fresh credentials
    d = connections["default"].settings_dict
    d["USER"], d["PASSWORD"] = creds["username"], creds["password"]
    connections["default"].close()         # the next query reconnects as the new DB user
```

Dynamic DB credentials fight with persistent connections: with `CONN_MAX_AGE` set, a pooled connection can outlive the credential that opened it, and an external pooler (pgbouncer) will happily keep a dead user's session. Keep `CONN_MAX_AGE` **well below the lease TTL**, renew the lease from a background thread, and treat `OperationalError: password authentication failed` as "re-fetch and `close()`", not as a crash. For a sensitive model field, **Transit** means Django only ever stores ciphertext — `c.secrets.transit.encrypt_data(name="pii", plaintext=b64)` in `save()`, `decrypt_data` on read: the key stays in Vault, and rotating it never rewrites your rows (that's what `rewrap` is for).
---
## 🧠 Summary

| Concept | Takeaway |
|---|---|
| **Seal / unseal** | Storage is worthless without the master key. Auto-unseal in production; Shamir shares split across humans |
| **Everything is a path** | One API, one policy language, default deny — and `deny` always wins |
| **Leases** | A dynamic secret has an expiry Vault actively *enforces*: renew it, or handle the failure |
| **Dynamic credentials** | Per-app, per-request DB users are the reason to run Vault at all |
| **The real cost** | Operational weight: HA quorum, upgrades, Raft snapshots, break-glass drills |
