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
## 🧩 What is GitLab CI?

**GitLab CI/CD** is the pipeline engine built into GitLab itself: one file at the repo root — **`.gitlab-ci.yml`** — plus **runners** that poll the server for jobs and report back. Nothing else to install, and the container registry, environments, merge-request gates, review apps and security dashboards are *the same product*. That integration, not the YAML, is the actual advantage over bolting a separate CI tool onto a forge. Think of a **conveyor belt built into the factory 🏭** rather than a machine wheeled in beside it — the QA station and the loading bay already know about each other. See [CI/CD](./README.md) for the concepts this note assumes.

---
## ⚙️ Anatomy of `.gitlab-ci.yml`

Everything is a **job** — a top-level key with a `script`. Jobs are grouped into `stages`, and these keywords do the work:

| Key | What it does | Gotcha |
|---|---|---|
| `stages:` | ordered stage names; jobs in one stage run in parallel, the next stage waits for all of them | a stage is a **barrier** — one slow job blocks everything behind it |
| `script:` / `before_script:` / `after_script:` | the commands; `before_script` is prepended to `script`, `after_script` runs even on failure | `after_script` runs in a **separate shell**, so it cannot see variables exported by `script` |
| `image:` / `services:` | the container the job body runs in, plus side-car containers on its network (Postgres, Redis) | the **shell** executor ignores `image:`; services are reachable by image name or `alias`, **never `localhost`** — they are separate containers |
| `rules:` | per-job conditions: `if`, `changes`, `exists`, `when`, `allow_failure` | **first match wins**, and no match means the job is not created at all |
| `needs:` | turns the pipeline into a **DAG** — start as soon as *these* jobs finish, ignoring stage order | it also narrows artifact download to exactly those jobs |
| `cache:` vs `artifacts:` | cache = a keyed path restored to make jobs faster; artifacts = files published for later stages and humans | caches are shared and mutable, so treat them as untrusted; add `dependencies: []` to a job that needs no artifacts, or it drags every earlier one into its workspace |
| `extends:` / `include:` | inherit from a hidden job (`.template`) / pull YAML from another file, project or URL | `include:remote:` is a live dependency on someone else's server — prefer `include:project` with a `ref` |
| `workflow:` | pipeline-level rules; the supported way to stop branch+MR **duplicate pipelines** | without it you pay twice for every push to an open MR |
| `environment:` + `when: manual` | names a deploy target (history, `url`, `deployment_tier`, `on_stop`) and puts a play button in front of it | scope production variables to the environment; a manual job in an *earlier* stage blocks later stages unless `allow_failure: true` |
| `trigger:` | starts a **child pipeline** (`include:`) or a **multi-project** pipeline (`project:`) | child pipelines keep a monorepo's config readable and its graph small; multi-project chains a downstream repo |

**Why `rules:` replaced `only`/`except`:** the old pair could not express AND/OR logic, could not see *why* the pipeline started (`CI_PIPELINE_SOURCE`), and reliably produced duplicate branch+MR pipelines. `only/except` still works but receives no new features — never mix both in one job. Also worth wiring in: `tags:` so a job only runs on chosen runners, `parallel: 5` with `CI_NODE_INDEX`/`CI_NODE_TOTAL` to split a slow suite, `parallel: matrix:` for version fan-out, `interruptible: true` so a new push cancels superseded runs, and **review apps** — a dynamic `environment: { name: review/$CI_COMMIT_REF_SLUG, on_stop: stop_review }` that deploys one throwaway instance per merge request.

---
## 🏃 Runner executors & the docker-in-docker problem

| Executor | How a job runs | Notes |
|---|---|---|
| **shell** | directly on the runner host, as the `gitlab-runner` user | fastest to set up, least isolated: no clean workspace, global tool installs, jobs can read each other's leftovers |
| **docker** | one container per job from `image:`, with `services:` as siblings | the sane default; **docker autoscaler** (successor to `docker+machine`) extends it by provisioning a fresh cloud VM per job — ephemeral by construction, pay-per-job |
| **kubernetes** | every job becomes a Pod (build + helper + service containers) | you already run K8s and want scale-to-zero ([Kubernetes](../Kubernetes/README.md)) |

**The docker-in-docker trap.** Building an image inside a docker-executor job classically means attaching the `docker:dind` service, which requires `privileged = true` in the runner's `config.toml`. **That single flag removes the isolation you were relying on — a privileged container is effectively host root**, so any job on that runner can mount the host disk, read every other project's build workspace and escape to the node. Mounting `/var/run/docker.sock` is the same hole with fewer steps ([Docker](../Docker/README.md)). Use a **daemonless builder** instead:

- **Kaniko** — `gcr.io/kaniko-project/executor:debug` builds and pushes a Dockerfile with no daemon and no privileges. Simplest drop-in, and what the example below uses.
- **BuildKit rootless** — `moby/buildkit:rootless` with `buildctl-daemonless.sh`, or `docker buildx` pointed at a remote BuildKit instance: better caching and multi-arch support. If dind is genuinely unavoidable, quarantine it on a **dedicated, tagged runner** used only by trusted projects, never shared with public repositories.
---
## 🔑 Variables, precedence & tokens

Precedence, highest first: **manual-run / trigger / schedule variables → project → group → instance → `dotenv` artifact from an earlier job → job-level `variables:` → global `variables:` → predefined `CI_*`**. Anything defined in the YAML therefore loses to a UI-defined variable of the same name, which is how an operator overrides a default without editing the repo. The predefined set you will use constantly: `CI_COMMIT_SHA`, `CI_COMMIT_REF_NAME`/`_SLUG`, `CI_DEFAULT_BRANCH`, `CI_PIPELINE_SOURCE`, `CI_MERGE_REQUEST_IID`, `CI_REGISTRY_IMAGE`, `CI_ENVIRONMENT_SLUG`, `CI_PROJECT_DIR`.

- **Masked** — the value is filtered out of job logs. It must be a single line, at least 8 characters, from a restricted character set, so many real secrets need base64 first. Masking is cosmetic: `set -x` or an `echo` through `base64` puts it straight back in the log.
- **Protected** — exposed **only** to jobs on protected branches/tags. This is the real control: it is what keeps a production credential out of every feature branch and fork pipeline.
- **File type** — GitLab writes the value to a temp file and sets the variable to that *path*. The correct way to ship a kubeconfig, a service-account JSON or a PEM chain.
- **`CI_JOB_TOKEN`** — minted per job, valid only while the job runs; used to clone other repos, pull from the project registry and call the API as the job. Its **allowlist** ("limit access to this project") decides which *other* projects may use a token to reach yours — keep it restricted, or a job in a low-trust project inherits your access.
---
## 🧪 Example — a real Django pipeline

```yaml
stages: [lint, test, build, deploy]

default:
  image: python:3.12-slim
  before_script: [pip install -r requirements.txt -r requirements-dev.txt]
  cache: { key: { files: [requirements.txt] }, paths: [.cache/pip] }   # dep bump = new cache

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"   # must be inside the workspace to be cacheable
  POSTGRES_DB: app_test                         # these three configure the service container
  POSTGRES_USER: app
  POSTGRES_PASSWORD: app                        # test-only value; real secrets are CI variables
  DATABASE_URL: "postgres://app:app@postgres:5432/app_test"   # host = the service alias
  DJANGO_SETTINGS_MODULE: config.settings.ci

lint:
  stage: lint
  script:
    - ruff check . && ruff format --check .
    - python manage.py makemigrations --check --dry-run   # missing migration = red pipeline

test:
  stage: test
  services: [{ name: postgres:16, alias: postgres }]
  script:
    - python manage.py check --deploy
    - pytest -q --cov=apps --cov-report=xml --junitxml=report.xml
  coverage: '/^TOTAL.+?(\d+\%)$/'
  artifacts:
    when: always                                # keep the reports even when tests fail
    reports:
      junit: report.xml
      coverage_report: { coverage_format: cobertura, path: coverage.xml }

build:                                          # daemonless: no dind, no privileged runner
  stage: build
  needs: [lint, test]
  image: { name: gcr.io/kaniko-project/executor:debug, entrypoint: [""] }
  before_script: []                             # nothing to pip install here
  script:
    - /kaniko/executor --context "$CI_PROJECT_DIR" --dockerfile Dockerfile
      --destination "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
  rules: [{ if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH }]

deploy:
  stage: deploy
  needs: [build]
  tags: [trusted-prod]                          # production runs only on a runner you control
  environment: { name: production, url: https://shop.example.com, deployment_tier: production }
  id_tokens: { VAULT_ID_TOKEN: { aud: https://vault.example.com } }   # OIDC, no stored secret
  before_script: []
  script: ["./scripts/deploy.sh $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"]   # migrate, then roll out
  rules: [{ if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH, when: manual }]
```
---
## ⚖️ GitLab CI vs GitHub Actions

| | **GitLab CI** | **GitHub Actions** |
|---|---|---|
| Self-hosting | first-class — the same product runs on your hardware, runners in several executor flavours | GHES is a separate, expensive product; self-hosted *runners* are easy, the server is not |
| DAG & reuse | `needs:` DAG, `extends`, `include`, parent-child and multi-project pipelines | `needs:`, reusable workflows, composite actions, and a huge marketplace |
| Registry / environments | container registry, environments and review apps built in | GHCR and environments built in; review-app equivalents are DIY |
| Ecosystem | templates and CI components; verbose config, but fewer moving parts to audit | thousands of actions and more expression syntax — enormous leverage, enormous supply-chain surface |

Verdict: **use whichever forge hosts your code**. Choose GitLab CI deliberately when you need on-prem CI *plus* registry and reviews in one auditable system; choose Actions when the marketplace and matrix testing are worth the pinning discipline ([GitHub Actions](./GitHubActions.md)).

---
## 🚨 When NOT to use it / limits
- ❌ **Your code lives on GitHub.** Mirroring a repo into GitLab just for CI means two sources of truth, two permission models and confusing MR statuses. Use the forge's own runner.
- ❌ **A tiny script or a one-command build** — a `Makefile` plus a `pre-commit` hook gives most of the value with none of the YAML ([Git](../Git/README.md)).
- ❌ **Complex, long-lived deploy orchestration** (multi-cluster rollout, drift correction, dependency-ordered releases). A pipeline is a one-shot script; that job belongs to Argo CD/Flux or Argo Workflows. Heavy build fan-out is the other limit: shared runners are metered and modest, and a self-hosted fleet becomes yours to secure.
---
## 🔐 Security notes & production hardening
- 🛡️ **Protected branches + protected variables are one control.** Mark `main` protected, mark every production variable **protected**, and a merge request — including one from a fork — simply cannot read it. Without this, any contributor who can run a pipeline can `env` your deploy key.
- 🚫 **Never `privileged = true`** in `config.toml`, and never bind-mount the Docker socket — that is host root for every job on the runner; build with Kaniko or rootless BuildKit. Then **tag your runners and tag your production jobs**: an untagged deploy job will happily execute on a shared or community runner, while `tags: [trusted-prod]` runs only where you decided.
- 🏷️ **Restrict the `CI_JOB_TOKEN` allowlist.** Default-open token access lets a job in another project act with your project's permissions; keep the inbound allowlist minimal and revisit it when repos are added.
- 🔑 **`id_tokens:` for OIDC instead of stored credentials.** GitLab signs a short-lived JWT per job; [Vault](../Vault/README.md) (JWT auth) or AWS/GCP/Azure verifies it and hands back a minutes-long credential. Bind the role to the exact claims — `project_path`, `ref`, `ref_protected: "true"` — or any branch in any project of that instance can assume it.
- 🏷️ **Tag your runners and tag your production jobs.** An untagged deploy job will happily execute on a shared or community runner; a `tags: [trusted-prod]` job runs only where you decided.
- 🧹 **Do not cache credentials.** `.git/config`, `.netrc`, `~/.docker/config.json` and `~/.aws` must never appear in `cache:paths` — caches are shared and writable by other jobs, so a poisoned cache is a supply-chain compromise. And own the pipeline definition: `CODEOWNERS` on `.gitlab-ci.yml` and every `include`d template, pinned to a tag/SHA rather than a moving branch, with approval required on protected environments so nobody merges themselves into production.
---
## 🐍 Django / Backend tie-in
- **Service hostnames, not `localhost`.** The job and `postgres:16` are sibling containers, so `DATABASE_URL` must point at the alias (`@postgres:5432`). This is the single most common "works locally, fails in CI" bug in a Django pipeline.
- **Review apps are excellent with Django** — deploy one instance per merge request, template `ALLOWED_HOSTS` from `CI_ENVIRONMENT_SLUG`, and give it an `on_stop` job that tears down the app *and* drops its database.
- **Migrations belong in `deploy.sh`, not inline in a job's `script:`** — run `migrate` once against the new image, then roll out, and keep the whole sequence runnable by a human during an incident ([CI/CD](./README.md)).
---
## 🧠 Summary

| Concept | Takeaway |
|---|---|
| Model | One `.gitlab-ci.yml` + runners; jobs in stages, or a `needs:` DAG when order actually matters |
| Reuse | `extends` + `include` + hidden `.templates`, and `workflow:` to kill duplicate pipelines |
| Services | Side-car containers reached by **alias**, never `localhost` |
| Building images | Kaniko or rootless BuildKit — `privileged` dind is host root |
| Secrets & deploys | Masked is cosmetic, **protected** is the control, `id_tokens:` beats stored keys; deploy via `environment:` + `when: manual` on a tagged trusted runner, promoting `$CI_COMMIT_SHA` |



