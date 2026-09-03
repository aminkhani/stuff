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
## 🧩 What is OPA (Open Policy Agent)?

**OPA** is a general-purpose **policy engine**: you hand it a JSON `input`, it evaluates rules written in **Rego** against that input plus a base `data` document, and it returns a JSON decision. That's the whole contract — **policy as code**, served as **decision as a service**. Think of a **judge 👩‍⚖️**: you bring the case file (the facts, as JSON), the judge consults the written law (policy + data) and returns a verdict. The judge never gathers evidence and never carries out the sentence — that part is still your code.
---
## 💡 Why do we need it?

- 🕸️ **Authorization logic scatters** — the same "may this user touch this object" check gets re-implemented in a view, a serializer, a Celery task and the gateway, each subtly different, each drifting, and none of them testable in isolation.
- 🧪 **Nobody can state the rules** — answering "what is our access policy?" by grepping for `if` is not an answer. A declarative policy is one file you unit-test, diff and review in a PR.
- 🔁 **The same rule in several services and languages** — a Go service, a Django app and the kube-apiserver all need "only owners may delete" or "prod pods must set limits". Write it once, enforce it everywhere.
- 📋 **Guardrails belong at the door** — "no image from an untrusted registry", "no `:latest`", "no public bucket" are policy, not code, and should be checked at admission and in CI instead of remembered by reviewers.
---
## ⚙️ How it works — PDP, PEP, `input` and `data`

```
client --> PEP (your app / API gateway / kube-apiserver)
             |  input { user, action, resource }       ^  { "result": true }
             v     anything else must mean DENY -------+
           PDP (OPA) = policy (*.rego) + data (roles, ownership) --> decision log
```

1. The **PEP** (Policy Enforcement Point) is *your* code: it builds `input`, asks, and enforces the answer. The **PDP** (Policy Decision Point) is OPA: it only ever answers → [AccessControl](../../Security/AccessControl.md).
2. **`input`** is the per-request document (who, what action, which object) — build it from server-side facts, session or DB row, never from client claims about itself. **`data`** is the pushed base document (role tables, group membership, allowlists) held in memory, and it arrives as a **bundle**: a signed tarball of policy + data that OPA pulls from HTTP/S3 on an interval (`services` + `bundles` in `opa run -c config.yaml`), so rule changes ship without redeploying a single app.
3. **Decision logs** stream every input/result pair out of OPA — your authorization audit trail, and the reason you can answer "why was this allowed in March?" ([Monitoring & Logging](../Monitoring&Logging/README.md)).
---
## 🛠️ Rego essentials

- A policy is a `package` of **rules**. Every expression in a rule body is **ANDed** and the rule is **undefined** if any of them fails; several rules **sharing a name are ORed**, which is how you bolt on "admins may do anything" without touching the existing rules. Since **OPA 1.0** the `if`/`contains` keywords are mandatory, and `import rego.v1` keeps the same file valid on 0.x too.
- `default allow := false` supplies a value when every `allow` rule is undefined. Without it the PEP gets nothing back, and it must then treat "no answer" as deny.
- `some x in xs` iterates, `contains` builds a partial **set** rule, and comprehensions (`names := [c.name | some c in cs]`) collect values. `not expr` is true when `expr` is *undefined*, which is why `not c.resources.limits.memory` catches an absent field while `!=` does not.
---
## 🚀 Deployment modes & where OPA shows up

**Modes**, pick by latency budget and language: **sidecar / daemon** (`opa run --server`, JSON over HTTP on `:8181`, policy reloads without touching the app) · **Go library** (`github.com/open-policy-agent/opa/rego`, in-process, no network hop) · **WASM / SDK bundle** (compile the policy and evaluate inside a Node, Python or edge runtime) · **CLI in CI** (`conftest`, `opa eval`) where there is no runtime to talk to at all.

- ☸️ **Kubernetes admission control** — **Gatekeeper** runs OPA as a validating webhook: a `ConstraintTemplate` + `Constraint` rejects a Pod with no `resources.limits` before it is ever scheduled ([Kubernetes](../Kubernetes/README.md)).
- 🌐 **API authorization microservice** — one sidecar answers `allow` for every request, so Go, Python and Node services enforce the same rule set instead of three copies of it ([API Gateway](../../SoftwareDesign/APIGateway.md)). And with **partial evaluation** — pre-computing everything that does not depend on `input` — the same policy becomes a **data filter**: "which docs may Alice see" compiles into a SQL `WHERE` clause instead of 40 000 separate decisions.
- 🏗️ **Terraform plans and CI gates** — `terraform show -json plan.out | opa eval -I 'data.terraform.deny'` fails the pipeline on an unencrypted volume or an `0.0.0.0/0` ingress rule; `conftest test` does the same for K8s manifests, Dockerfiles and Compose files before anything reaches a cluster ([CI/CD](../CI-CD/README.md)).
---
## 🧪 Examples

**RBAC + ABAC in one policy** — deny by default, roles from `data`, ownership from `input`:
```rego
package authz
import rego.v1

# data.roles = {"editor": [{"resource": "doc", "action": "read"}, ...], "viewer": [...]}
default allow := false                    # nothing is allowed unless a rule says so

allow if "admin" in input.user.roles      # same-name rules are ORed together
allow if {                                # every expression in a body is ANDed
	some role in input.user.roles
	some perm in data.roles[role]
	perm.resource == input.resource.type
	perm.action == input.action
}
allow if {                                # ABAC: owners may read and delete their own
	input.action in {"read", "delete"}
	input.user.id == input.resource.owner
}
deny_reasons contains msg if {            # a set rule -> readable errors for the PEP
	not allow
	msg := sprintf("%v may not %v %v", [input.user.id, input.action, input.resource.type])
}

# authz_test.rego (same package) -- tests are just rules; `with` swaps the documents
test_owner_can_delete if {
	allow with input as {"user": {"id": "u7"}, "action": "delete", "resource": {"owner": "u7"}}
}
test_stranger_is_denied if {
	not allow with input as {"user": {"id": "u9"}, "action": "delete", "resource": {"owner": "u7"}}
}
```

**Conftest gate** — a Deployment must not run as root and must set memory limits:
```rego
package main                              # conftest looks for deny/violation/warn in "main"
import rego.v1

deny contains msg if {
	input.kind == "Deployment"
	some c in input.spec.template.spec.containers
	not c.securityContext.runAsNonRoot    # "not" also fires when the field is absent
	msg := sprintf("container %q must set securityContext.runAsNonRoot: true", [c.name])
}
deny contains msg if {
	input.kind == "Deployment"
	some c in input.spec.template.spec.containers
	not c.resources.limits.memory
	msg := sprintf("container %q has no memory limit", [c.name])
}
```

```bash
opa fmt --diff . && opa check --strict . && opa test . -v    # the entire CI gate
opa eval -d authz.rego -d data.json -i input.json 'data.authz.allow'   # -> true | false
conftest test k8s/deployment.yaml                            # uses the package main rules
opa run --server --addr 127.0.0.1:8181 authz.rego data.json  # PDP, loopback only
curl -s 127.0.0.1:8181/v1/data/authz/allow -H 'content-type: application/json' \
  -d '{"input":{"user":{"id":"u7","roles":["editor"]},"action":"read",
                "resource":{"type":"doc","owner":"u7"}}}'
# {"result":true}
```
---
## 🚨 When NOT to use it

- ❌ **A single small app** — DRF permissions and `has_object_permission` already do this in-process with the ORM at hand. A PDP would buy you a network hop and a second language.
- ❌ **Relationship graphs** — "which of these 40 000 docs can Alice reach through nested group membership" needs a tuple store built for traversal. That is **OpenFGA / Zanzibar**, not Rego over a pushed `data` blob.
- ❌ **Latency-critical paths where you cannot run in-process** — a remote PDP adds a round trip and a new failure mode per request; use the Go library, WASM, or cache decisions.
- ❌ **As a home for business logic** — pricing, workflow state and field validation are not policy. If a rule needs to *write* something or call another service, it does not belong in Rego.
---
## ⚖️ OPA vs the alternatives

| Option | Language | Primary domain | Relationship data | Curve | Pick it when |
|---|---|---|---|---|---|
| **OPA / Rego** | Rego — declarative, datalog-flavoured | Anything JSON-in / JSON-out: K8s, APIs, CI, Terraform | 🟡 only what you push into `data`, no graph traversal | 🔴 Steep — budget real time for Rego | You need one policy language across several domains and services |
| **Kyverno** | YAML | Kubernetes admission only | ❌ | 🟢 Low | Pure K8s policy: the pragmatic default, no new language to learn |
| **Kubernetes RBAC** | YAML verbs on resources | Who may call the K8s API | ❌ | 🟢 Low | Coarse API access — it cannot inspect the object's *contents* |
| **Casbin** | Model + policy files (CSV/DB) | In-app RBAC/ABAC, many languages | 🟡 via adapters | 🟢 Low | Library-level authz inside one app, with no PDP to operate |
| **Cedar** | Cedar DSL, typed and analysable | App / API authorization | 🟡 one-hop entity graph | 🟡 Medium | You want provable, typed app authz (AWS Verified Permissions on AWS) |
| **OpenFGA / Zanzibar** | Relationship tuples + schema DSL | ReBAC: sharing, nesting, org trees | ✅ core feature, graph queries | 🟡 Medium | "Which objects can user X see", and inherited permissions |
| **Hand-rolled `if`s** | Your own language | Whatever you remembered to check | 🟢 it is just your DB | 🟢 None | One small app and one team — until the rules drift and nobody can audit them |
---
## 🔐 Security notes & production hardening

> [!IMPORTANT]
> **Fail closed.** A PDP timeout, a 5xx, an unparseable body or an undefined result must all mean **deny**. Set an explicit client timeout in the tens of milliseconds, keep `default allow := false` in the policy, and make the PEP's error path return "no" — an authorization system that fails open is not an authorization system.

- **Sign and verify bundles** (`opa build --signing-key`, run with `--verification-key`) and serve them over HTTPS: whoever controls the bundle server controls your authorization. `opa run --server` itself has **no authentication and no authorization by default**, and its Policy API can *replace* your rules — bind it to `127.0.0.1` as a sidecar, or start it with `--authentication=token --authorization=basic`. Never expose `:8181`.
- **No secrets in policy or in `data`** — bundles are copied, cached and logged. Decision logs contain the entire `input`, which means PII: apply decision-log **masking** and treat the log sink as a sensitive system ([Monitoring & Logging](../Monitoring&Logging/README.md)).
- **Policy is code** — git, PR review, and `opa fmt --diff` + `opa check --strict` + `opa test` on every commit ([CI/CD](../CI-CD/README.md)). Bound query complexity too: nested iteration over a large `data` document is combinatorial, so keep `data` small, prefer equality on fields OPA can index, and set a query timeout.
- **The PEP is still yours to get right** — a policy engine cannot fix a check you forgot to make. Every entry point must ask, and `input` must be assembled from server-side facts (session, DB row), never from what the client claims about itself.
---
## 🐍 Django / Backend tie-in

```python
# permissions.py -- a DRF object permission that asks a local OPA sidecar
import requests
from rest_framework.permissions import BasePermission

class OPAPermission(BasePermission):
    def has_object_permission(self, request, view, obj):
        payload = {"input": {"user": {"id": str(request.user.id), "roles": [g.name for g in request.user.groups.all()]},
                             "action": request.method.lower(), "resource": {"type": obj._meta.model_name, "owner": str(obj.owner_id)}}}
        try:
            r = requests.post("http://127.0.0.1:8181/v1/data/authz/allow", json=payload, timeout=0.25)
            return r.status_code == 200 and r.json().get("result") is True
        except (requests.RequestException, ValueError):
            return False       # timeout, 5xx or garbage all mean deny, never allow-on-error
```

The honest take: for **one** Django app this is a worse `has_object_permission` — you have added a network hop, a second language and a new failure mode to express a rule the ORM could check directly. OPA earns its place when the *same* rules must hold in more than one place: a Go worker and Django enforcing identical ownership rules, the [API Gateway](../../SoftwareDesign/APIGateway.md) rejecting requests before they reach any service, and Gatekeeper applying the same policy repo in-cluster. Then Rego is the single source of truth and DRF just calls it — with a cache and a hard timeout, because every view now depends on it.
---
## 🧠 Summary

| Concept | Takeaway |
|---|---|
| **PDP vs PEP** | OPA decides, your code enforces — and must fail closed whenever OPA does not answer |
| **`input` vs `data`** | Per-request facts vs the pushed base document (bundles); policy touches nothing else |
| **Rego** | Rule body = AND, same-name rules = OR, `default allow := false`, `not` also catches undefined |
| **Right tool?** | Several enforcement points → OPA · one Django app → DRF permissions · relationship graphs → OpenFGA |
