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
## 🧩 What is Threat Modeling?

**Threat modeling** is a structured design activity: draw what you are building, then reason about how an attacker abuses it — *before* the code exists. It is the cheapest security control you own, because it removes design flaws that no scanner, WAF or pentest can retrofit.

Think of it as a **fire drill on the blueprint 🏗️**: you don't wait for a fire to discover that the emergency exit opens inward. Pentesting finds bugs in what you built; threat modeling finds out that you are building the wrong thing.

---
## 💡 Why do it, and where it fits in the SDLC

- **Design flaws are invisible to tools.** "Any authenticated user can read any invoice" is a perfectly valid HTTP 200 — SAST and DAST see nothing wrong.
- **Cost.** Fixing a misplaced trust boundary in a design doc is a conversation; fixing it after 40 endpoints depend on it is a migration.
- **Shared mental model.** Devs, ops and security argue over the same diagram instead of three different assumptions.
- **Compliance leverage.** It is the artifact auditors ask for as proof of a secure SDLC — see [ISO 27001](./ISO27001.md) and the *Identify* function in [NIST CSF](./Cybersecutity%20Framework.md).

In the [SDLC](../SoftwareDesign/SDLC.md) it lives in **design**, after requirements and before implementation — and then again on every architecture change (new datastore, new third party, new boundary). Not once per project: once per **change that moves data across a boundary**.

---
## ⚙️ How it works — Shostack's four questions

| Question                                    | What you actually produce                             |
| ------------------------------------------- | ----------------------------------------------------- |
| 1. **What are we building?**                | A **DFD** with trust boundaries drawn on it            |
| 2. **What can go wrong?**                   | A threat list — STRIDE, per element                    |
| 3. **What are we going to do about it?**    | Mitigations, each one a real ticket                    |
| 4. **Did we do a good enough job?**         | Review, test the mitigations, retro the model itself   |

A **DFD (Data Flow Diagram)** has only five symbols, and that austerity is the point — it is about *data movement*, not classes: **external entity**, **process**, **data store**, **data flow**, and the **trust boundary** (a dashed line wherever the level of trust changes — internet→DMZ, DMZ→private subnet, your code→a third party, user space→kernel).

> [!TIP]
> Threats concentrate on **flows that cross a trust boundary**. A diagram with no boundaries drawn is a sketch, not a threat model — draw the dashed lines first and half the analysis falls out for free.

```text
       Internet (untrusted)         :  Private subnet (trusted)
  [Mobile app] --HTTPS--> ( Nginx )-:-> ( Django + Gunicorn ) --> [[ PostgreSQL ]]
  [Attacker] ------------^          :            |
                                    :            +--> ( Celery ) --> [[ Redis ]]
                                    :            +--HTTPS--> { Stripe API }

  [] external entity  () process  [[]] data store  {} third party  : trust boundary
```

---
## 🔤 STRIDE — answering "what can go wrong?"

Walk **every element** of the DFD and ask all six questions. STRIDE is a checklist for **threat identification** and nothing more — it deliberately does not rank anything.

| Threat                     | Property violated | Concrete example                                                                                          | Mitigation                                                                                              |
| -------------------------- | ----------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| **S**poofing               | Authentication    | Replayed long-lived JWT; forged `X-Forwarded-For`; look-alike domain; a webhook that only *claims* to be Stripe | Short-lived tokens + refresh rotation, verify signature/`aud`/`iss`, mTLS, HMAC-signed webhooks           |
| **T**ampering              | Integrity         | Client posts `{"role": "admin"}`; MITM on plain HTTP; an audit table anyone can `UPDATE`                    | [TLS](./TransportSecurity.md) everywhere, explicit serializer `fields`, server-side validation, append-only logs |
| **R**epudiation            | Non-repudiation   | User denies cancelling a paid order; nobody can say who deleted the record                                 | Immutable audit trail with actor + IP + request-id; no shared admin accounts                             |
| **I**nformation disclosure | Confidentiality   | `DEBUG=True` traceback exposing settings; serializer leaking `password_hash`; world-readable bucket        | `DEBUG=False`, field allow-lists, encryption at rest, least-privilege DB user                            |
| **D**enial of service      | Availability      | `?page_size=100000`; ReDoS in a validator; unbounded upload; a report endpoint with no timeout             | Pagination caps, throttling, statement + request timeouts, upload limits, WAF                            |
| **E**levation of privilege | Authorization     | IDOR/BOLA on `/api/orders/42/`; `is_staff` settable through the API; SSTI → RCE                            | Object-level permissions, deny-by-default, queryset filtering — see [Access Control](./AccessControl.md) |

---
## 📊 Ranking risk — DREAD, and why CVSS replaced it

**DREAD** scores a threat 1–10 on five axes and averages them:

`Risk = (Damage + Reproducibility + Exploitability + Affected users + Discoverability) / 5`

Its weakness is fatal in practice: **the numbers are subjective and not reproducible**. Two engineers score the same IDOR 4.2 and 8.6, no axis has a defined scale, and *Discoverability* actively rewards security-by-obscurity — a bug is not less dangerous because it is well hidden. Microsoft itself moved off it.

**CVSS v3.1/v4.0** is the modern answer for concrete vulnerabilities: a defined metric vector (`AV`/`AC`/`PR`/`UI`/`S`/`C`/`I`/`A`) yielding a reproducible string such as `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N` → 6.5, plus temporal and environmental modifiers. It is the language CVEs, scanners and [SCA](../Hardening/SCA.md) tooling already speak, so it plugs straight into remediation SLAs.

> [!IMPORTANT]
> At **design** time there is no exploit to score, so CVSS does not fit either. Use a plain **Likelihood × Impact** matrix for design threats, CVSS for findings from scanners and pentests, and DREAD only in a small team that has already agreed what each digit means.

---
## 🍝 PASTA — the risk-centric 7-stage process

**PASTA** (Process for Attack Simulation and Threat Analysis) is not a checklist but a **whole methodology**: it starts from business impact and ends with a prioritised countermeasure roadmap.

1. **Define business objectives** — what the application is worth, plus compliance drivers.
2. **Define the technical scope** — infrastructure, dependencies, network, actors.
3. **Application decomposition** — DFDs, data classification, trust boundaries.
4. **Threat analysis** — real threat intelligence: who actually attacks this sector, and how.
5. **Vulnerability & weakness analysis** — map known weaknesses (CWEs) onto components.
6. **Attack modelling** — attack trees; simulate the paths an attacker would chain together.
7. **Risk & impact analysis** — residual risk, countermeasures, roadmap with owners.

Stages 4 and 6 are exactly what STRIDE lacks: **evidence-driven** threats and *chained* attack paths rather than a per-element checklist. The cost is equally real — PASTA is weeks of work with business stakeholders, so it suits a regulated program, not a sprint.

---
## 🧪 Example — model a Django REST API as code (pytm)

Threat modelling **as code** means the model lives in git beside the app, diffs during review, and regenerates in CI instead of rotting as a PDF.

```python
# tm.py  -  OWASP pytm
from pytm import TM, Actor, Server, Datastore, Dataflow, Boundary

tm = TM("Orders API")
inet, dmz, priv = Boundary("Internet"), Boundary("DMZ"), Boundary("Private subnet")

client = Actor("Mobile client");         client.inBoundary = inet
web = Server("Nginx (TLS termination)"); web.inBoundary = dmz
api = Server("Django + Gunicorn");       api.inBoundary = priv
db = Datastore("PostgreSQL");            db.inBoundary = priv

Dataflow(client, web, "HTTPS POST /api/v1/orders (JWT)")
Dataflow(web, api, "proxy_pass 127.0.0.1:8000 - crosses DMZ -> private")
Dataflow(api, db, "psycopg over TLS: SELECT/INSERT orders")

tm.process()
```

```bash
pip install pytm                                   # graphviz needed for --dfd
python tm.py --dfd | dot -Tpng -o docs/dfd.png     # regenerate the diagram
python tm.py --report docs/template.md > docs/threats.md
```

The threats this exact diagram surfaces, and the mitigation that really closes each one:

| #     | Threat (STRIDE)                                                                    | Boundary crossed  | Concrete mitigation                                                                                    |
| ----- | ---------------------------------------------------------------------------------- | ----------------- | ------------------------------------------------------------------------------------------------------ |
| **1** | **S** — a long-lived JWT lifted from mobile storage is replayed                     | Internet → DMZ    | 5–15 min access token, rotating refresh tokens, `SIMPLE_JWT` blacklist, bind the refresh to a device     |
| **2** | **E** — `GET /api/v1/orders/42/` returns another tenant's order (IDOR / BOLA)       | DMZ → private     | `get_queryset().filter(user=request.user)` **and** `has_object_permission`; never trust the URL pk       |
| **3** | **T** — client posts `{"is_staff": true}` to the profile serializer                | DMZ → private     | Explicit `fields` + `read_only_fields`; never `fields = "__all__"`                                       |
| **4** | **I** — `DEBUG=True` on staging returns settings and SQL inside the traceback       | Internet → DMZ    | `DEBUG=False`, tight `ALLOWED_HOSTS`, `manage.py check --deploy` gating the pipeline                     |
| **5** | **D** — `?page_size=100000` on the list endpoint pins the database                  | DMZ → private     | DRF `PAGE_SIZE` + `max_page_size`, `DEFAULT_THROTTLE_RATES`, `statement_timeout` on the DB role          |
| **6** | **S/R** — a forged "Stripe" webhook marks an order paid, leaving no trace           | third party → private | Verify the `Stripe-Signature` HMAC, idempotency keys, one append-only audit row per state change     |

Five of the six sit on a **boundary crossing** — and #2 is precisely the class of bug no scanner will ever report, because it is a valid `200 OK`.

---
## ⚖️ STRIDE vs DREAD vs PASTA vs LINDDUN vs Attack Trees vs tooling

| Method                                       | The question it answers                            | Output                                       | Effort | Pick it when                                                        |
| -------------------------------------------- | -------------------------------------------------- | -------------------------------------------- | ------ | ------------------------------------------------------------------- |
| **STRIDE**                                   | *What can go wrong?* — **identification**          | Categorised threat list, per element          | Low    | Design review of a feature or service. The default starting point.   |
| **DREAD**                                    | *Which one matters most?* — subjective **ranking**  | A single 1–10 number                          | Low    | Small team with an agreed scale, fast triage. Otherwise skip it.      |
| **CVSS**                                     | *How severe is this concrete vulnerability?*        | Reproducible vector + 0.0–10.0                | Low    | Scanner/pentest findings, CVEs, remediation SLAs                     |
| **PASTA**                                    | *What is the business risk, end to end?* — process  | Risk-based countermeasure roadmap             | High   | Regulated org, large program, business sign-off required             |
| **LINDDUN**                                  | *What can go wrong for **privacy**?*                | Privacy threats: Linkability, Identifiability, Non-repudiation, Detectability, Disclosure, Unawareness, Non-compliance | Medium | PII/GDPR-heavy systems; complements STRIDE, never replaces it |
| **Attack Trees**                             | *How exactly could an attacker reach goal X?*       | AND/OR tree of concrete attack paths          | Medium | Deep-diving one high-value asset STRIDE already flagged              |
| **Tooling** (Threat Dragon, IriusRisk, pytm) | Not a method — somewhere to *keep* the model         | Versioned diagram + generated threat list     | Low    | You want the model in git and regenerated in CI                      |

**STRIDE and DREAD answer different questions.** STRIDE *finds* threats, DREAD/CVSS *rank* them, PASTA wraps both inside a business process. A healthy default: STRIDE for identification, Likelihood × Impact (or CVSS once code exists) for ranking.

---
## 🚨 When NOT to do it (or when it turns into theatre)

- ❌ **On a throwaway prototype or an internal script** with no users and no data — and equally ❌ **as a 60-page PDF signed off once**: a model nobody revisits is wrong the day after the next architecture change, and worse, it *feels* like coverage.
- ❌ **Boiling the ocean.** One session over a whole monolith yields 200 undifferentiated threats nobody triages. Scope it to one service, one feature, one boundary.
- ❌ **Threats without an owner.** A threat that never becomes a ticket is a note, and notes do not ship mitigations. Equally: the model predicts, it does not verify — you still need tests, SAST/DAST, [SCA](../Hardening/SCA.md) and a pentest.
- ⚠️ **As a security-team-only exercise.** Without the engineers who wrote the code you will model an architecture that does not exist.

---
## 🔐 Making the model survive contact with production

- **Keep it in the repo, next to the code** — `docs/threat-model.md` plus `tm.py`, reviewed in the same PR as the design. A model in a wiki nobody opens is already dead.
- **Trigger, don't schedule:** re-run on a new datastore, a new third-party integration, a new trust boundary, an auth change. "Every quarter" is how you get a stale model.
- **Every accepted threat becomes a ticket** with an owner and a severity, linked back to the model; every *rejected* threat gets one line explaining why. That record is exactly what an auditor wants.
- **Write abuse cases, not only user stories.** Next to "as a user I can export my invoices", add "as a competitor I export *someone else's* invoices" — see [User Story](../SoftwareDesign/Scrum/UserStory.md).
- **Use attacker personas with different capability** — a curious logged-in customer, a script kiddie with `ffuf`, an insider with DB read access, a funded competitor — and **verify each mitigation exists** as something you can point at: a setting, a test, a policy in [OPA](../DevOps/OPA/README.md), a permission class. Then feed the results into the *Identify/Protect* functions of [NIST CSF](./Cybersecutity%20Framework.md) and your [CIA](./CIA.md) priorities.

---
## ⚠️ Common mistakes

- Drawing an **architecture diagram** (boxes and logos) instead of a **data-flow diagram** with boundaries.
- Modelling only the happy path — the attacker is not in your user table, so put them on the diagram explicitly.
- Confusing **assumptions** with controls: "the internal network is trusted" is an assumption, and it is usually the finding itself.

---
## 🧠 Summary

| Concept                 | Takeaway                                                                            |
| ----------------------- | ----------------------------------------------------------------------------------- |
| **Threat modeling**     | Draw it, break it on paper, fix the design before the code exists                    |
| **Four questions**      | What are we building → what can go wrong → what do we do → did it work                |
| **DFD**                 | Five symbols; all the value is in the **trust boundaries** you bother to draw         |
| **STRIDE vs DREAD**     | STRIDE *identifies*, DREAD/CVSS *rank*. Different questions — never a substitute      |
| **Living document**     | In git, retriggered by architecture change, every threat linked to a ticket            |




