# 👨‍💻 My Learning Journey 🚀

Hey there! This repo is my personal knowledge vault — an Obsidian vault kept under version control, where everything I learn about backend engineering, databases, infrastructure and security gets written down in my own words. All notes live under [`Stuff/`](Stuff).

> **Legend** — every link below points to a note that is actually written and readable · a **🎯 To Learn** checklist marks topics I haven't covered yet, picked for relevance to **Django Backend + Security** roles. Checked items `[x]` are ones a note now covers.

| Area | Written notes |
| --- | --- |
| 🐍 Python & Django | 11 |
| 🛢️ Database | 12 |
| 🏗️ Software Design & Architecture | 11 |
| 🐧 Linux & Networking | 4 |
| 🚀 DevOps | 7 |
| 🔒 Security | 8 |
| 🧠 Large Language Models | 1 |

---

## 🧠 Topics I'm Learning

### 🐍 Python & Django

**Python**

- 🚀 [UV](Stuff/Python/UV.md) — the fast Python package/project manager
- 🔀 [If vs Try](Stuff/Python/IfVsTry.md) — LBYL vs EAFP, and which one to reach for
- 🐞 [Flaky Tests](Stuff/Python/FlakyTest.md) — why tests fail randomly and how to tame them
- 💡 [gRPC](Stuff/Python/gRPC.md) — protobuf, RPC types, when it beats REST
- 🛠️ [gRPC Tips](Stuff/Python/gRPCTips.md) — production practice: deadlines, retries, schema evolution, status codes
- 🧵 [Futures](Stuff/Python/Futures.md) — `concurrent.futures`, thread vs process pools, and the traps

**Django**

- 🔐 [Authentication](Stuff/Django/Authentication.md)
    - 📦 [Djoser](Stuff/Django/DRF/Djoser.md) — ready-made DRF auth endpoints
- 🔑 [Authorization](Stuff/Django/Authorization.md)
    - 🚪 [OAuth](Stuff/Django/OAuth.md)
- 📑 Models
    - ⌨️ [Character Types](Stuff/Django/Models/CharacterTypes.md) — `CharField` vs `TextField`
- 🐘 [Managing Ever‑Growing Tables](Stuff/Database/ManagingEver%E2%80%91GrowingTables.md) — Django + PostgreSQL growth strategies

**🎯 To Learn**

- [ ] Django REST Framework internals (serializers, viewsets, permissions)
- [ ] Django middleware & signals
- [ ] Celery + task queues (async jobs, retries, scheduling)
- [ ] Django ORM query optimization (`select_related`, `prefetch_related`) → groundwork in [Query Execution Plans](Stuff/Database/Query%20Execution%20Plans.md)
- [ ] Async Django (ASGI, Django Channels)
- [ ] Testing: pytest-django, factory_boy, coverage → related: [Flaky Tests](Stuff/Python/FlakyTest.md)
- [ ] Django security settings (CSRF, CORS, `SECURE_*` settings, clickjacking protection)
- [x] Rate limiting & throttling in DRF → [API Security](Stuff/Security/API%20Security.md)

---

### 🛢️ Database

- 🧭 [Database Roadmap](Stuff/Database/README.md) — the study outline this section follows
- 🔑 [Database Indexing](Stuff/Database/Database%20Indexing.md)
- 🗺️ [Query Execution Plans](Stuff/Database/Query%20Execution%20Plans.md) — `EXPLAIN ANALYZE`, reading costs
- 🔗 [SQL Joins](Stuff/Database/SQL%20Joins.md)
- 🧩 [Normalization vs Denormalization](Stuff/Database/Normalization%20vs%20Denormalization.md)
- 💳 [Database Transactions](Stuff/Database/Database%20Transactions.md) — ACID, isolation levels
- 🎯 [CAP Theorem](Stuff/Database/CAP%20Theorem.md)
- 🔢 [Partitioning](Stuff/Database/Partitioning.md)
- 🧼 [VACUUM](Stuff/Database/VACUUM.md)
- 💫 [Fill Factor](Stuff/Database/FillFactor.md)
- 📅 [Materialized View](Stuff/Database/MaterializedView.md)
- 🐘 [Managing Ever‑Growing Tables](Stuff/Database/ManagingEver%E2%80%91GrowingTables.md)

**🎯 To Learn**

- [x] Indexing strategies & `EXPLAIN ANALYZE` → [Database Indexing](Stuff/Database/Database%20Indexing.md) + [Query Execution Plans](Stuff/Database/Query%20Execution%20Plans.md)
- [ ] Replication & failover (streaming replication, read replicas)
- [ ] Redis (caching, sessions, pub/sub)
- [ ] Row-level security & column-level encryption
- [ ] Connection pooling (PgBouncer)
- [ ] Database backup/restore & disaster recovery
- [ ] MongoDB
- [ ] PostgreSQL configuration & tuning

---

### 🏗️ Software Design & Architecture

- 🖥️ [Web Server](Stuff/SoftwareDesign/WebServer/WebServer.md)
    - 🚦 [Nginx](Stuff/SoftwareDesign/WebServer/Nginx.md)
    - 🔁 [Forward & Reverse Proxy](Stuff/SoftwareDesign/WebServer/ForwardAndReverseProxy.md)
- ⚖️ [Load Balancer](Stuff/SoftwareDesign/LoadBalancer.md)
- 🚪 [API Gateway](Stuff/SoftwareDesign/APIGateway.md)
- 📨 [Kafka vs RabbitMQ](Stuff/SoftwareDesign/KafkaVsRabbitMQ.md)
- 🧩 [Design Walkthrough](Stuff/SoftwareDesign/DesignWalkthrough.md) — how to attack a design problem step by step
- 📺 [System Design Interview Guide](Stuff/SoftwareDesign/System%20Design%20YouTube.md) — full framework, with YouTube as the running example
- 📐 [SOLID](Stuff/SoftwareDesign/SOLID.md)
- 💼 [SDLC](Stuff/SoftwareDesign/SDLC.md)
- 🗒️ [User Story](Stuff/SoftwareDesign/Scrum/UserStory.md) — Scrum

**🎯 To Learn**

- [x] Message queues (RabbitMQ, Kafka) & event-driven design → [Kafka vs RabbitMQ](Stuff/SoftwareDesign/KafkaVsRabbitMQ.md)
- [ ] Microservices vs. monolith trade-offs
- [ ] Caching strategies (write-through, cache-aside, invalidation)
- [ ] Idempotency & retry-safe API design → groundwork in [gRPC Tips](Stuff/Python/gRPCTips.md)
- [ ] Domain-Driven Design basics
- [ ] Circuit breaker / bulkhead resilience patterns

---

### 🐧 Linux & Networking

- 🔌 [Socket](Stuff/Linux/Socket.md)
- 🧷 [UNIX Domain Sockets](Stuff/Linux/UnixSockets.md)
- 🌐 [DMZ](Stuff/Linux/DMZ.md) — demilitarized zone, network segmentation
- 🛠️ [Service](Stuff/Linux/Service.md) — services, daemons, init systems, systemd units, timers and sandboxing

**🎯 To Learn**

- [ ] TCP/IP fundamentals & the OSI model
- [ ] Firewalls & `iptables`/`nftables`
- [ ] VPNs & network segmentation → related: [DMZ](Stuff/Linux/DMZ.md)
- [x] TLS handshake deep dive → [Transport Security](Stuff/Security/TransportSecurity.md)
- [ ] Linux hardening (SELinux/AppArmor, file permissions, user isolation) → started in [SCA](Stuff/Hardening/SCA.md) + unit sandboxing in [Service](Stuff/Linux/Service.md)
- [ ] Magic bytes / file-type detection

---

### 🚀 DevOps

- 🐳 [Docker](Stuff/DevOps/Docker/README.md)
- ⚓ [Kubernetes](Stuff/DevOps/Kubernetes/README.md) — its own roadmap, plus install guides, workloads, networking and backup notes inside that folder
- 🧪 [CI/CD](Stuff/DevOps/CI-CD/README.md)
    - 🦊 [GitLab CI](Stuff/DevOps/CI-CD/GitLabCI.md)
- 🔄 [Git](Stuff/DevOps/Git/README.md)
- 🔒 [HashiCorp Vault](Stuff/DevOps/Vault/README.md)
- 📜 [Open Policy Agent (OPA)](Stuff/DevOps/OPA/README.md)

**🎯 To Learn**

- [ ] GitHub Actions & Jenkins (CI/CD beyond GitLab)
- [ ] Infrastructure as Code (Terraform, Ansible)
- [ ] Container security (image scanning, distroless images, least-privilege containers)
- [ ] Cloud fundamentals (AWS/GCP/Azure IAM, networking, managed DBs)
- [ ] Zero-downtime deployments (blue-green, canary, rolling)
- [ ] Monitoring, centralized logging & tracing (ELK/EFK, Prometheus, OpenTelemetry)
- [ ] Secrets rotation & management at scale → foundation in [Vault](Stuff/DevOps/Vault/README.md)

---

### 🔒 Security

- 🌀 [CIA Triad](Stuff/Security/CIA.md) — confidentiality, integrity, availability
- 🧩 [Access Control](Stuff/Security/AccessControl.md) — authentication vs authorization, DAC, MAC, RBAC, ABAC, PBAC
- 🔐 [Transport Security](Stuff/Security/TransportSecurity.md) — SSL/TLS, certificates, mTLS
- 🧨 [Threat Modeling](Stuff/Security/ThreatModeling.md) — STRIDE, DREAD, PASTA
- 📋 [NIST Cybersecurity Framework](Stuff/Security/Cybersecutity%20Framework.md)
- 📜 [ISO/IEC 27001](Stuff/Security/ISO27001.md) — ISMS, controls, certification
- 🛡️ [SCA — Security Configuration Assessment](Stuff/Hardening/SCA.md) — CIS benchmarks, Wazuh SCA module
- 🛡️ [API Security](Stuff/Security/API%20Security.md) — OWASP API Top 10, BOLA/BFLA, rate limiting, CORS

**🎯 To Learn**

- [ ] OWASP Top 10 (in depth, with Django-specific mitigations)
- [ ] Secure coding practices & code review checklists
- [ ] SAST/DAST tools (Bandit, Semgrep, OWASP ZAP)
- [ ] Software Composition Analysis — dependency/CVE scanning (distinct from the configuration-assessment SCA note above)
- [x] API security in depth (JWT best practices, OAuth2 scopes, API keys, mTLS between services) → [API Security](Stuff/Security/API%20Security.md)
- [ ] Encryption fundamentals (symmetric vs. asymmetric, hashing, salting, key management/KMS)
- [ ] Security headers (CSP, HSTS, X-Frame-Options)
- [ ] WAF concepts & configuration
- [ ] SIEM basics & log correlation
- [ ] Incident response process
- [ ] Compliance basics (GDPR, SOC 2) → related: [ISO/IEC 27001](Stuff/Security/ISO27001.md)
- [ ] Zero Trust Architecture

---

### 🧠 Large Language Models

- ⚙️ [MCP](Stuff/LLM/MCP.md) — Model Context Protocol

**🎯 To Learn**

- [ ] RAG (Retrieval-Augmented Generation) fundamentals
- [ ] LLM/prompt injection security & sandboxing tool calls
- [ ] Embeddings & vector databases (pgvector, FAISS)

---

## 🗂️ Full Index

Everything in the vault, folder by folder, so nothing gets lost. Kubernetes is a single entry on purpose — that folder carries its own roadmap and 23 notes of its own.

<details>
<summary><b>Stuff/Database</b> — 12 notes</summary>

- [README](Stuff/Database/README.md) — database study roadmap (fundamentals → scaling → consistency → modeling)
- [CAP Theorem](Stuff/Database/CAP%20Theorem.md)
- [Database Indexing](Stuff/Database/Database%20Indexing.md)
- [Database Transactions](Stuff/Database/Database%20Transactions.md)
- [FillFactor](Stuff/Database/FillFactor.md) — PostgreSQL fill factor and I/O performance
- [ManagingEver‑GrowingTables](Stuff/Database/ManagingEver%E2%80%91GrowingTables.md) — Django & PostgreSQL
- [MaterializedView](Stuff/Database/MaterializedView.md)
- [Normalization vs Denormalization](Stuff/Database/Normalization%20vs%20Denormalization.md)
- [Partitioning](Stuff/Database/Partitioning.md)
- [Query Execution Plans](Stuff/Database/Query%20Execution%20Plans.md)
- [SQL Joins](Stuff/Database/SQL%20Joins.md)
- [VACUUM](Stuff/Database/VACUUM.md)

</details>

<details>
<summary><b>Stuff/DevOps</b> — 6 notes + the Kubernetes folder</summary>

- [CI-CD/README](Stuff/DevOps/CI-CD/README.md) — what CI/CD is, pipelines, stages
- [CI-CD/GitLabCI](Stuff/DevOps/CI-CD/GitLabCI.md)
- [Docker/README](Stuff/DevOps/Docker/README.md)
- [Git/README](Stuff/DevOps/Git/README.md)
- [OPA/README](Stuff/DevOps/OPA/README.md) — Open Policy Agent, Rego, policy as code
- [Vault/README](Stuff/DevOps/Vault/README.md) — HashiCorp Vault
- [Kubernetes/README](Stuff/DevOps/Kubernetes/README.md) — entry point for the Kubernetes area (install guides, core concepts, workloads, networking, backup)

</details>

<details>
<summary><b>Stuff/Django</b> — 5 notes</summary>

- [Authentication](Stuff/Django/Authentication.md)
- [Authorization](Stuff/Django/Authorization.md)
- [OAuth](Stuff/Django/OAuth.md)
- [DRF/Djoser](Stuff/Django/DRF/Djoser.md)
- [Models/CharacterTypes](Stuff/Django/Models/CharacterTypes.md)

</details>

<details>
<summary><b>Stuff/Hardening</b> — 1 note</summary>

- [SCA](Stuff/Hardening/SCA.md) — Security Configuration Assessment: CIS benchmarks, Wazuh SCA, CIS-CAT

</details>

<details>
<summary><b>Stuff/LLM</b> — 1 note</summary>

- [MCP](Stuff/LLM/MCP.md) — Model Context Protocol

</details>

<details>
<summary><b>Stuff/Linux</b> — 4 notes</summary>

- [Socket](Stuff/Linux/Socket.md)
- [UnixSockets](Stuff/Linux/UnixSockets.md)
- [DMZ](Stuff/Linux/DMZ.md)
- [Service](Stuff/Linux/Service.md) — services, daemons, init systems, systemd units, timers, sandboxing

</details>

<details>
<summary><b>Stuff/Python</b> — 6 notes</summary>

- [UV](Stuff/Python/UV.md)
- [IfVsTry](Stuff/Python/IfVsTry.md)
- [FlakyTest](Stuff/Python/FlakyTest.md)
- [gRPC](Stuff/Python/gRPC.md)
- [gRPCTips](Stuff/Python/gRPCTips.md) — deadlines, retries, proto evolution, status codes, hardening
- [Futures](Stuff/Python/Futures.md) — `concurrent.futures`: pools, cancellation, timeouts, traps

</details>

<details>
<summary><b>Stuff/Security</b> — 7 notes</summary>

- [AccessControl](Stuff/Security/AccessControl.md)
- [CIA](Stuff/Security/CIA.md)
- [TransportSecurity](Stuff/Security/TransportSecurity.md)
- [ThreatModeling](Stuff/Security/ThreatModeling.md)
- [ISO27001](Stuff/Security/ISO27001.md)
- [Cybersecutity Framework](Stuff/Security/Cybersecutity%20Framework.md) — NIST CSF (filename typo kept so existing links don't break)
- [API Security](Stuff/Security/API%20Security.md) — OWASP API Top 10, authz, rate limiting, CORS, DRF hardening

</details>

<details>
<summary><b>Stuff/SoftwareDesign</b> — 12 notes</summary>

- [WebServer/WebServer](Stuff/SoftwareDesign/WebServer/WebServer.md)
- [WebServer/Nginx](Stuff/SoftwareDesign/WebServer/Nginx.md)
- [WebServer/ForwardAndReverseProxy](Stuff/SoftwareDesign/WebServer/ForwardAndReverseProxy.md)
- [LoadBalancer](Stuff/SoftwareDesign/LoadBalancer.md)
- [APIGateway](Stuff/SoftwareDesign/APIGateway.md)
- [KafkaVsRabbitMQ](Stuff/SoftwareDesign/KafkaVsRabbitMQ.md)
- [DesignWalkthrough](Stuff/SoftwareDesign/DesignWalkthrough.md)
- [SOLID](Stuff/SoftwareDesign/SOLID.md)
- [SDLC](Stuff/SoftwareDesign/SDLC.md)
- [Scrum/UserStory](Stuff/SoftwareDesign/Scrum/UserStory.md)
- [System Design YouTube](Stuff/SoftwareDesign/System%20Design%20YouTube.md) — complete system design interview guide
- [README1](Stuff/SoftwareDesign/README1.md) — ⚠️ byte-for-byte copy of the [Database roadmap](Stuff/Database/README.md), most likely a stray file worth deleting

</details>

<details>
<summary><b>Repo root</b> — no notes any more, just the paperwork</summary>

- [LICENSE](LICENSE) — MIT

</details>

---

## 🧰 About this vault

Notes are plain Markdown written in Obsidian and pushed here, so they read the same in the app and on GitHub — every link above is relative for that reason. `Stuff/` holds the knowledge base; each area is a folder, and larger areas (Database, Kubernetes) carry their own `README.md` roadmap. Every note listed above is actually written; the **🎯 To Learn** checklists are the honest to-do list of what is still missing.

## 🔗 Connect with Me

- 🐙 GitHub: [aminkhani](https://github.com/aminkhani)
- 💼 LinkedIn: [aminkhani-ai](https://linkedin.com/in/aminkhani-ai)
- 📬 Email: aminkhani2010@gmail.com

---

🚀 Thanks for stopping by! Let's keep learning, building, and sharing knowledge!
