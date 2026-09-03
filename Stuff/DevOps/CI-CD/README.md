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
## 🧩 What is CI/CD?

Three different ideas sharing one acronym — and interviewers ask precisely because most people blur them:

- **Continuous Integration** — everyone merges into **trunk** at least daily, and **every merge is built and tested automatically**. CI is a *team habit* enforced by a machine, not a YAML file; long-lived branches are the thing it exists to kill.
- **Continuous Delivery** — **every green commit produces a releasable artifact**, and production deploy is *one button a human chooses to press*.
- **Continuous Deployment** — the same, minus the human: green pipeline → **it deploys itself**.

An **airport baggage system 🧳**: CI is every bag going through the same scanner the moment it is checked in; Continuous Delivery is bags queued at the aircraft door, cleared to load; Continuous Deployment is the belt loading them with nobody watching. The distinction is not pedantry — "we do CI/CD" while merging month-old branches is just a build server.
---
## 💡 Why do we need it?
- 🔀 **Small batches beat big ones twice** — merge pain grows with branch age (daily integration keeps conflicts trivial, monthly integration is a project), and a 3-commit deploy is diagnosable where a quarterly 400-commit release is a bisect problem with an audience.
- 🧪 **The machine finds the bug minutes after it is written**, while the author still has the context. Fixing then costs a fraction of fixing after release.
- 📦 **One reproducible artifact** — built once from a known commit, and *that digest* runs in every environment, so "it passed on staging" finally means something ([Docker](../Docker/README.md)).
- 🔐 **A place to enforce policy, and the audit trail that proves it** — lint, tests, dependency/container scanning, signing and `check --deploy` become gates nobody can "forget" ([SCA](../../Hardening/SCA.md), [OPA](../OPA/README.md)), while "who changed what, which tests passed, who approved production" is just pipeline metadata — half of an [SDLC](../../SoftwareDesign/SDLC.md) control set for free.
---
## ⚙️ How it works — pipeline anatomy

```text
trigger          push / merge request / tag / cron / manual / API
 +- pipeline     one run, bound to exactly ONE commit SHA
     +- stage    lint -> test -> build -> scan -> deploy   (a barrier, or a DAG edge)
         +- job  one runner, fresh workspace, parallel with its siblings
             +- step/script    the actual commands
     artifacts   files a job produces that later jobs (or humans) consume
```

**Fail-fast ordering is the whole design** — the cheapest checks that catch the most run first, because every minute of pipeline is multiplied by every push: format/lint/typecheck (seconds; style, undefined names, wrong types) → **unit tests** (1–3 min; logic regressions) → **build the image** (packaging, missing system libs) → **integration/e2e** against a real DB and broker (5–20 min; wiring, migrations, contracts) → **security scans** (SAST, dependencies, container, IaC) → **deploy + smoke test**. A 40-minute suite that ends in a linter error wasted 40 minutes; ordered correctly it fails in 20 seconds.
---
## 🏃 Runners / agents

The machine that executes a job — and both of its axes matter:

- **Hosted vs self-hosted.** Hosted: zero maintenance, per-minute billing, generic hardware, **no route into your private network**. Self-hosted: your hardware, your network, GPUs, cheap at volume — and you now own patching, scaling, isolation and the security of a box that holds deploy credentials.
- **Ephemeral vs long-lived.** Ephemeral = a fresh container/VM per job, destroyed afterwards, and the correct default. Long-lived = "a box with a shell runner": fast, and a slow disaster.

**Why ephemeral matters twice over.** *Hygiene:* a long-lived runner accumulates a global `pip` cache, a stale `node_modules`, a leftover `.env`, a half-migrated database — so the build passes for reasons that are not in the repo, and fails on a clean machine. *Security:* whatever job B leaves on disk, job A can read — tokens, private source, SSH keys. One compromised build on a shared runner reads every other project's secrets, and can persist a backdoor in the runner itself.
---
## 💾 Cache vs artifacts — different jobs, do not mix them

| | **Cache** | **Artifact** |
|---|---|---|
| Purpose | make a job *faster* | pass output *forward*, or keep it for humans |
| Content | reproducible, disposable (`~/.cache/pip`, `.venv`, `node_modules`) | build output, coverage XML, JUnit report, the image digest |
| Trust | **untrusted** — attacker-writable input if any job can poison it | produced by a known job from a known SHA |
| If it vanishes | the build is slower, still correct | the pipeline is broken |

Never cache what you cannot regenerate, key the cache on the lockfile (`requirements.txt` / `uv.lock`) so a dependency bump invalidates it, and **never cache credentials, `.git/config` or `.netrc`** — a poisoned cache is a supply-chain attack with a friendly name.
---
## 🚦 Environments, approvals & promotion

An **environment** is a named deploy target (`dev`, `staging`, `production`) owning its own variables, URL, approval rules and deployment history; approvals ("required reviewers", "protected environment") are what turn Continuous *Deployment* into Continuous *Delivery* for the one environment that matters. Config comes from the environment, not from the artifact — env vars for boring values, a secret store for the rest ([Vault](../Vault/README.md)).

> [!IMPORTANT]
> **Build once, deploy many.** The artifact is built **exactly once**, from one commit, and the *same immutable digest* is promoted dev → staging → production; only configuration changes between them. If your pipeline rebuilds the image per environment, staging never tested what production runs — a floating base-image tag, a fresh dependency resolution or a different build cache is enough to make them different binaries. Promote a digest, never a `latest` tag ([Docker](../Docker/README.md)).
---
## 🚀 Deployment strategies

| Strategy | Downtime | Rollback speed | Extra infra | DB-migration difficulty |
|---|---|---|---|---|
| **Recreate** (stop all, start new) | ❌ yes, full | redeploy the old version (minutes) | none | 🟢 easiest — nothing else is running, an exclusive window exists |
| **Rolling** (replace N at a time) | none | another rolling update (minutes) | +surge capacity | 🔴 hard — old and new code run **simultaneously**, so every migration must be backward-compatible |
| **Blue-green** (two full stacks, flip the LB) | none | ⚡ instant — flip traffic back | **2× the fleet** | 🔴 hard — both colours share one database |
| **Canary** (1% → 10% → 100%, watch metrics) | none | fast — shift the canary to 0% | small overlap | 🔴 hard, same as rolling, plus you need real SLO metrics to decide |
| **Feature flags** (deploy dark, toggle on) | none | ⚡ instant, **no deploy at all** | flag service | 🟢 decouples schema change from behaviour change |

Verdict: **rolling** is the sane default and what Kubernetes does natively ([Kubernetes](../Kubernetes/README.md)); **blue-green** when you need an instant, provable rollback and can afford double capacity; **canary** when you have the observability to notice the failure ([Monitoring & Logging](../Monitoring&Logging/README.md)); **feature flags** for anything risky, because they let you roll back *behaviour* without touching the artifact. All of the non-recreate options need a [load balancer](../../SoftwareDesign/LoadBalancer.md) doing health-checked draining, or "zero downtime" just means "dropped in-flight requests quietly".
---
## 🗃️ Database migrations in CD — the honest part

Code rolls back in seconds. **Schema does not.** During any zero-downtime deploy, **version N and N−1 of your code talk to the same database at the same time**, so a migration must be compatible with *both*. The discipline is **expand → migrate → contract**, spread over **separate releases**:

1. **Expand** — add the nullable column / new table / new index. Deploy code that **writes both** old and new, still reads old.
2. **Backfill** — copy data in batches, throttled, as a job — not inside a web request and not in one giant transaction.
3. **Switch reads** — a release that reads the new column. Now the old one is unused but still present.
4. **Contract** — *a later release* drops the old column, once no running replica references it and you no longer plan to roll back past step 3.

> [!WARNING]
> **Never ship a destructive migration in the same release as the code that stops using the column.** The rolling deploy hasn't finished: old pods still `SELECT dropped_column` and return 500s, and your rollback path is now gone because the old code cannot run against the new schema. Renames are two releases (add + backfill + drop), never one `ALTER TABLE ... RENAME`.

Operationally: run `migrate` as a **one-shot job before the new code rolls out** (safe, because the migration is backward-compatible by construction), set a short `lock_timeout` so a blocked `ALTER TABLE` fails the deploy instead of queueing every query behind an `ACCESS EXCLUSIVE` lock, and build indexes with `CREATE INDEX CONCURRENTLY`. See also [Partitioning](../../Database/Partitioning.md) for the large-table cases where an online change is the only option.
---
## 📈 DORA metrics — is the pipeline actually working?

Two speed measures and two stability measures — use them as diagnostics, not targets (pipeline duration is your lever on lead time, rollback design your lever on recovery). The research point is that **good teams improve both halves together**; treating them as a trade-off is the anti-pattern:

| Metric | Question | Elite ballpark |
|---|---|---|
| **Deployment frequency** | how often do you ship to production? | on demand, multiple times a day |
| **Lead time for changes** | commit → running in production | under a day |
| **Change failure rate** | % of deploys causing degraded service | ~5–15% |
| **Failed-deployment recovery time** (formerly MTTR) | how fast do you recover? | under an hour |
---
## 🔁 GitOps — the pull-based alternative

Classic CD is **push**: the pipeline holds cluster credentials and runs `kubectl apply` / `helm upgrade` from outside. **GitOps** inverts it — desired state lives in a Git repo and an **in-cluster agent** (**Argo CD**, **Flux**) continuously **pulls** and reconciles reality toward it.

| | **Push CD** | **GitOps (pull)** |
|---|---|---|
| Who holds cluster creds | the CI runner — an internet-facing system | **nobody outside**: the agent has them, inbound API access stays closed |
| Drift | persists silently until the next deploy | detected and **self-healed**; rollback is `git revert` on the manifest repo |
| Cost | trivial to set up | another controller, and secrets need Sealed/External Secrets or [Vault](../Vault/README.md) |

The clean split: **CI builds, tests and pushes the image, then commits the new digest into the manifests repo; CD is the reconciler.** Keep manifests in a separate repo or path so an image bump does not retrigger the build.
---
## ⚖️ Tooling comparison

| Tool | Hosting model | Config | Ecosystem | Secrets / OIDC | Self-host cost | Pick it when |
|---|---|---|---|---|---|---|
| **[GitLab CI](./GitLabCI.md)** | SaaS *or* fully self-managed, same product | `.gitlab-ci.yml` + `include`/`extends` | built-in registry, environments, review apps, scanners | masked + **protected** variables, `id_tokens:` for OIDC | GitLab server + runners — heavy, but one system | your code is on GitLab, or you need CI + registry + reviews on-prem |
| **[GitHub Actions](./GitHubActions.md)** | SaaS-first (GHES for on-prem) | `.github/workflows/*.yml` | **largest** marketplace — and the largest supply-chain surface | environment secrets, first-class OIDC to AWS/GCP/Azure | runners are free software; isolation is your problem | your code is on GitHub; OSS; matrix testing across versions |
| **[Jenkins](./Jenkins.md)** | self-hosted only | Groovy `Jenkinsfile` (declarative) | thousands of plugins: its superpower and its CVE feed | Credentials plugin binding; OIDC only via plugins | **highest** — patch monthly, own the controller | on-prem, air-gapped, exotic hardware, 20 years of existing jobs |
| **Argo CD / Flux** | runs **inside** the Kubernetes cluster | Git-stored manifests / Helm / Kustomize | K8s-native only — **not** a build system | External Secrets / Sealed Secrets | one controller per cluster | you deploy to K8s and want pull-based, self-healing delivery |
| **CircleCI / Buildkite** | CircleCI: SaaS + orbs · Buildkite: SaaS control plane, **your** agents | `.circleci/config.yml` / `pipeline.yml` | smaller, focused, fast | OIDC supported | Buildkite: you run only the agents | you want hosted UX but your source and compute stay on your hardware (Buildkite) |

Rule of thumb: **use whatever your forge already provides** — the integration (statuses, MR gates, registry, environments) is most of the value. Reach for Jenkins when the constraint is physical, and add Argo/Flux *next to* your CI rather than instead of it.
---
## 🚨 When NOT to use it / limits
- ❌ **Continuous *Deployment* with a thin test suite or no rollback path.** Auto-deploy amplifies whatever quality you already have. Without a tested rollback (previous digest still pullable, migrations reversible), it is an automated outage generator — stay on Continuous *Delivery* with a human gate until the safety net exists.
- ❌ **A pipeline built on flaky tests.** Two red builds that pass on re-run and the team learns to click "retry" — at which point the gate is decorative and a real regression ships. Quarantine and fix them; never `pytest --reruns` your way out ([Flaky Tests](../../Python/FlakyTest.md)).
- ❌ **A pipeline nobody waits for.** Past ~10 minutes people push and context-switch, so failures land after the author has moved on. Parallelise, split by test type, cache dependencies, and run the 40-minute e2e suite nightly or pre-merge — not on every commit.
- ❌ **400 lines of shell inside YAML.** Vendor YAML is untestable, unrunnable locally, un-lintable and un-portable. Put the logic in `scripts/deploy.sh` or a `Makefile` **that the repo owns and any developer can run**; the pipeline should only decide *when* to call it. It is also the cheapest migration insurance you will ever buy.
---
## 🔐 Security notes & production hardening

Your CI system is the **most privileged machine you own**: read access to every repository, the credentials to production, and it executes arbitrary code from every branch. That is why supply-chain attackers target the pipeline before the application — and why "it's only build tooling" is how organisations get owned.

| Control | Concretely |
|---|---|
| **Secrets never in YAML** | the pipeline definition lives in the repo — use the platform's secret store, mark variables **masked**, and scope them to **protected branches / environments** so a feature branch can never read a production key |
| **OIDC federation, not static cloud keys** | the runner requests a short-lived, workload-bound **JWT**; AWS/GCP/Azure (or [Vault](../Vault/README.md)) verifies it and returns a credential valid for minutes. Nothing to rotate, nothing to steal. **Constrain the subject claim** (`repo:org/app:ref:refs/heads/main`) — a trust policy matching `repo:org/*` lets *any* repo in the org assume your production role |
| **Least-privilege job token** | the built-in token defaults to far more than a test job needs: make it read-only per job, grant write only to the one job that pushes |
| **Pin third parties by digest** | `FROM python:3.12-slim@sha256:...`; third-party actions/templates pinned to a **full commit SHA**, never `@v4` — a tag is mutable, and a compromised upstream repoints it straight at your runner |
| **Masking is a display filter, not a control** | `echo "$SECRET" \| base64`, `set -x`, or the value inside a JSON blob defeats it. Assume that whatever a job can read, a job can exfiltrate |
| **Never run untrusted PR code with secrets attached** | fork/MR pipelines get no secrets, no write token and no deploy stage. Anything else is running a stranger's `Makefile` with your production credentials in the environment |
| **Hosted, ephemeral runners for public repos** | a self-hosted runner serving public PRs is a takeover vector — a stranger's branch becomes code execution inside your network, on a box holding the previous job's leftovers |
| **Sign and attest what you ship** | `cosign sign` the image **by digest**, generate an **SBOM** per build (`syft`, `docker buildx build --sbom=true`) and a **provenance** attestation binding commit → pipeline → digest. The **SLSA** levels are graded strengths of exactly that claim ([SCA](../../Hardening/SCA.md)) |
| **Verify at deploy time** | signatures are worthless unless something *refuses* unsigned images — admission policy in the cluster ([OPA](../OPA/README.md)), pulling over TLS from a registry you control ([Transport Security](../../Security/TransportSecurity.md)) |
| **Scans are gates, not reports** | SAST + `pip-audit` + `trivy image` + IaC scanning, failing the pipeline on *new* criticals with a documented exception path. A dashboard nobody blocks on changes nothing |
| **Protect the pipeline definition itself** | `.gitlab-ci.yml` / `.github/workflows/**` / `Jenkinsfile` under **CODEOWNERS** with required review — whoever can edit the pipeline can print every secret. Audit who may approve production, forbid self-approval, and keep deploy logs immutable ([Monitoring & Logging](../Monitoring&Logging/README.md)) |
---
## 🐍 Django / Backend tie-in

```bash
#!/usr/bin/env bash
# scripts/ci.sh - the pipeline body: identical on a laptop and on a runner
set -euo pipefail
export DJANGO_SETTINGS_MODULE=config.settings.ci
export DATABASE_URL="postgres://app:app@127.0.0.1:5432/app_test"
ruff check . && ruff format --check .                 # or: black --check . && isort --check-only .
mypy config apps
python manage.py makemigrations --check --dry-run     # exits 1 if a model change has no migration
python manage.py check --deploy                       # needs DEBUG=False settings, or it passes blind
pytest -x -q --cov=apps --cov-report=xml              # coverage.xml becomes the artifact
pip-audit -r requirements.txt                         # known CVEs in the dependency tree
docker build -t "$IMAGE:$GIT_SHA" .
trivy image --exit-code 1 --severity HIGH,CRITICAL "$IMAGE:$GIT_SHA"
docker push "$IMAGE:$GIT_SHA"                         # then promote THIS tag/digest, never rebuild
```

- **`makemigrations --check --dry-run` is the highest-value one-line gate in a Django pipeline** — it catches "model edited, migration forgotten", a bug that is invisible locally (your dev DB already has the column) and fatal in production.
- **Order of operations at deploy:** push image → run `migrate` **once** as a one-shot job/`Job` on the *new* image → then roll out replicas. Never migrate from the container entrypoint: N replicas start simultaneously and race. `collectstatic` belongs to the image build ([Docker](../Docker/README.md)).
- **Test against real Postgres, not SQLite** — constraint timing, `JSONField`/array behaviour, `select_for_update` and migration SQL all differ, so a green SQLite suite proves little. Run it as a pipeline **service** container, and keep the suite deterministic ([Flaky Tests](../../Python/FlakyTest.md)).


> [!NOTE]
> **Windows:** a shell script authored on Windows and committed with **CRLF** dies on a Linux runner with `$'\r': command not found` or a silently mangled last argument. Commit `*.sh text eol=lf` in `.gitattributes` ([Git](../Git/README.md)) — the runner will not fix it for you.
---
## 🧠 Summary

| Concept | Takeaway |
|---|---|
| CI vs CD vs CD | Integrate daily and test every merge → every green commit is *releasable* → it deploys itself |
| Pipeline design | trigger → stage → job → step; cheapest checks first, one commit SHA per run, logic in scripts the repo owns |
| Artifacts | Build **once**, promote the same digest across environments; cache for speed, artifacts for correctness |
| Deploying | Rolling by default, blue-green for instant rollback, canary with real metrics, flags to decouple risk |
| Migrations | Expand → backfill → switch → contract, across **separate releases**; never destructive in the same release |
| Security | Ephemeral least-privilege runners, OIDC over static keys, pin by digest, no secrets for fork PRs, sign + verify, CODEOWNERS on the pipeline |
| Outcome | DORA's four keys, and GitOps when you want pull-based, self-healing delivery |
