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
## 🧩 What is an API Gateway?

An **API gateway** is a single, **policy-aware** entry point for **north-south** API traffic — client → your platform. A [reverse proxy](WebServer/ForwardAndReverseProxy.md) routes bytes; a gateway routes bytes *and* enforces who may send them, how often, and in what shape.

It is the **passport desk at an airport** 🛂: one queue, identity checked once, luggage weighed, and only then are you allowed toward a specific gate — instead of every gate running its own border control.

---
## 💡 What it centralizes

- 🪪 **Authentication** — validate a JWT, API key or client cert once, at the edge.
- 🛂 **Authorization hand-off** — ask a policy engine, then pass a *verified* identity downstream.
- 🚦 **Rate limiting and quotas** — per consumer, per plan, per route; not merely per IP.
- 🔄 **Request / response transformation** — rewrite paths, rename headers, strip fields for legacy clients.
- 📐 **Schema validation** — reject a malformed body before it costs you a worker.
- 🔀 **Protocol translation** — REST ↔ gRPC, HTTP/1.1 ↔ HTTP/2, JSON ↔ protobuf ([gRPC](../Python/gRPC.md)).
- 🧵 **Aggregation / composition** — one client call fanned out to three services (sparingly — see BFF).
- 🔢 **Versioning and traffic shifting** — `/v1` vs `/v2`, canary by weight or header, blue-green cutover.
- 🔑 **API keys, plans, developer portal** — the commercial surface of a public API.
- 🔭 **Observability** — one place that knows latency, error rate and traffic *per consumer*.

---
## ⚙️ How it works

```
       north-south  =  the gateway's job
  internet -> [ LB ] -> [ API GATEWAY ] -> [ orders ]
                         authn, quota,         |
                         validate, route       |   east-west = the mesh's job
                         transform, log        v
                                            [ billing ] <-> [ users ]
```

1. TLS terminates at the load balancer or at the gateway ([LoadBalancer](LoadBalancer.md)).
2. The gateway matches a **route** (host + path + method) to an upstream **service**.
3. It runs a **plugin chain**: authn → authz hand-off → rate limit → schema validation → transform → trace.
4. It forwards upstream, then runs the response half of the chain: mask errors, add headers, record metrics.
5. Whatever it rejects never reaches a worker — that, not elegance, is the capacity argument for a gateway.

---
## 🎭 BFF — and why it is not the same thing

A **Backend for Frontend** is an *application*, not a proxy: one per client type (iOS, web, partner integration). It owns **client-specific** concerns — folding five downstream calls into one payload, trimming fields for a small screen, adapting to a legacy client's shape. The gateway owns **client-agnostic, cross-cutting** policy.

**Rule of thumb:** if the logic differs per client, it belongs in a BFF you deploy like any other service. If it applies identically to every consumer, it belongs in the gateway. Pushing BFF logic into gateway plugins is exactly how a gateway becomes a distributed monolith.

---
## 🛂 PEP vs PDP — where authorization actually lives

- **PEP (Policy Enforcement Point)** — where a decision is *applied*. The gateway is an excellent PEP.
- **PDP (Policy Decision Point)** — where the decision is *made*: OPA, or your own authz service.

The gateway can decide the coarse questions: *is this token valid, is this scope allowed on this route, has this consumer any quota left.* It cannot decide *may user 42 edit invoice 99* — that needs data it does not have. The working pattern is: gateway validates the token → calls a PDP (Envoy `ext_authz` or the APISIX `opa` plugin against an OPA sidecar) → the **service still re-checks object-level permissions with its own data** ([AccessControl](../Security/AccessControl.md), [OPA](../DevOps/OPA/README.md)).

---
## ⚖️ Gateway vs its neighbours

| | What it does | Where it sits | What it costs you |
|---|---|---|---|
| **API gateway** | authn, quotas, transformation, keys, versioning — policy | **north-south**, in front of many APIs | a hop, a control plane, and a team that owns it |
| **Reverse proxy** | TLS, routing, cache, compression, static files | north-south edge | almost nothing — but no per-consumer policy |
| **Load balancer** | spreads traffic over a pool, health checks | in front of one pool | nothing; it has no idea what an API is |
| **Service mesh** | mTLS, retries, circuit breaking on internal calls | **east-west**, a sidecar per pod | one proxy per pod, added latency, large ops surface |
| **BFF** | shapes and aggregates responses for **one** client | behind the gateway, one per frontend | real application code to write, test and deploy |
| **Framework middleware** (DRF) | authn, throttling, validation, in-process | inside each app | nothing to operate — but duplicated in every service |

**Verdict:** north-south policy → gateway. East-west reliability → mesh. Spreading load → LB. Client-shaped payloads → BFF. One service, few consumers → middleware, and stop there.

---
## 🛠️ Picking a product

| Product | Config model | Extensibility | Pick when |
|---|---|---|---|
| **Kong Gateway 3.x** | Postgres-backed **or** DB-less `kong.yaml`; CRDs via its Ingress Controller | Lua plugins (PDK), WASM, external plugins in Go/Python/JS | you want a mature plugin catalogue plus a declarative file that lives in Git |
| **Apache APISIX** | etcd-backed, hot-reloaded through the Admin API; standalone YAML too | Lua plugins, plugin runners (Go/Python/Java), WASM | you need OSS OIDC/OPA plugins and config changes with no restart |
| **Traefik** | K8s CRDs, Docker labels, or a file provider — auto-discovers | middlewares, a narrower plugin system (Yaegi/WASM) | Docker Compose or K8s and you value zero-config discovery over deep policy |
| **Envoy** | xDS from a control plane; raw YAML is verbose on purpose | HTTP filters, Lua, WASM, `ext_authz` | you already run a control plane or mesh, or need surgical control |
| **Nginx + Lua** (OpenResty) | plain `nginx.conf` plus Lua | anything you are willing to write and test yourself | you already run [Nginx](WebServer/Nginx.md) and need two small policies, not a platform |
| **AWS API Gateway** | fully managed, defined in IaC | Lambda authorizers, usage plans, WAF integration | serverless backends and you want zero gateway to operate |
| **Kubernetes Gateway API** | CRDs, `gateway.networking.k8s.io/v1` | portable core plus implementation-specific policy attachment | vendor-neutral, RBAC-friendly K8s routing — the successor to [Ingress](../DevOps/Kubernetes/Networking&Services/AdvancedNetworking/Ingress/README.md) |

---
## 🧪 Example

Kong Gateway 3.x in **DB-less** mode — one file, version-controlled, no database:

```yaml
# kong.yaml
_format_version: "3.0"

services:
  - name: orders-api
    url: http://orders.internal:8000
    connect_timeout: 2000          # ms
    read_timeout: 30000
    routes:
      - name: orders-v1
        paths: ["/v1/orders"]
        methods: ["GET", "POST"]
        strip_path: false          # upstream still sees /v1/orders

consumers:
  - username: mobile-app
    keyauth_credentials:
      - key: "{vault://env/MOBILE_APP_KEY}"   # never a literal secret in Git

plugins:
  - name: key-auth
    service: orders-api
    config:
      key_names: ["apikey"]
  - name: rate-limiting
    service: orders-api
    config:
      minute: 60
      policy: local              # use "redis" once you run more than one node
  - name: request-size-limiting
    service: orders-api
    config:
      allowed_payload_size: 8    # MB
```

```bash
docker run -d --name kong \
  -e KONG_DATABASE=off \
  -e KONG_DECLARATIVE_CONFIG=/kong/kong.yaml \
  -e KONG_ADMIN_LISTEN=127.0.0.1:8001 \
  -v "$PWD/kong.yaml:/kong/kong.yaml:ro" \
  -p 8000:8000 kong:3.6

curl -i http://localhost:8000/v1/orders                  # 401, no key
curl -i http://localhost:8000/v1/orders -H 'apikey: ...' # 200, and 61st call in a
                                                         # minute returns 429
```
## 🚨 When NOT to use it

- ❌ **A single monolith.** Your reverse proxy plus DRF middleware already does authn, throttling and validation. A gateway adds a hop, a config language and an on-call rotation for zero new capability.
- ❌ **Business logic in the gateway.** Routing on order state, computing a discount in a Lua plugin, stitching a domain object together — none of it is covered by your test suite, none of it deploys with your app, and debugging it means reading somebody else's proxy logs.
- ❌ **A gateway team as a queue.** If every route change waits on one platform team's review, delivery slows across the whole org. Give teams self-service config (CRDs, per-team files in Git) with guardrails instead.
- ❌ **The distributed-monolith smell.** If a service is only usable through gateway-specific rewriting and aggregation, the gateway has become hidden coupling — you can no longer call, test or move that service without it.
- ❌ **Internal service-to-service calls.** `orders → gateway → billing` adds two hops and two failure modes to every internal request. East-west traffic belongs to direct calls or a mesh.

---
## 🔐 Security notes & production hardening

- 🪪 **Validate the whole JWT, not just the signature:** pin `alg` (reject `none` and RS↔HS confusion), verify `iss`, `aud` and `exp`/`nbf` with bounded clock skew, cache JWKS and handle rotation. A signature-only check happily accepts a valid token minted for a different audience.
- 🛡️ **Defence in depth:** re-check authorization in the service. Anything that can reach the upstream directly — another pod, a debug port, a stale security group — skips every rule you wrote at the edge.
- 🔗 **Never blindly trust gateway-injected identity headers** (`X-Consumer-ID`, `X-User-Id`). They mean something only if the edge **strips** them from inbound requests *and* upstreams accept connections **only** from the gateway — enforced with mTLS plus network policy, not with a comment in the config.
- 🙈 **Mask internal errors:** map upstream 5xx to a generic body. No stack traces, no upstream hostnames, no ORM messages.
- 🚦 **Limit per consumer, not only per IP:** quotas, `request-size-limiting`, upstream timeouts, and a WAF for the OWASP basics. Behind CGNAT, per-IP limits punish entire networks instead of the abuser.
- 🔐 **Secrets by reference, never inline:** `{vault://...}` or APISIX secret refs backed by [Vault](../DevOps/Vault/README.md). Gateway config lives in Git and gets pasted into tickets.
- ⏱️ **Budget the latency, remove the SPOF:** every plugin costs milliseconds, and the gateway now sits on the critical path for 100 % of traffic. Run at least two replicas behind an LB, keep its health probe shallow, roll config out gradually, and watch p99 per route ([Monitoring&Logging](../DevOps/Monitoring&Logging/README.md)).

> [!IMPORTANT]
> **Never expose the admin API.** Kong's Admin API on `0.0.0.0:8001` is unauthenticated remote configuration: anyone who reaches it can add a route that proxies anywhere, or swap a plugin. It is a recurring source of real breaches. Bind it to `127.0.0.1` or to a management network with mTLS and RBAC — and for APISIX, change the default `admin_key` and restrict `allow_admin` to your own CIDRs.

---
## 🐍 Django / Backend tie-in

- **DRF throttling vs gateway rate limiting** — the gateway does the cheap coarse work (per-consumer and per-IP limits) *before* a Gunicorn worker is ever occupied, which is what actually protects capacity. DRF's `ScopedRateThrottle` handles the **business** quota that needs app data — "10 exports per day per organisation". Keep both; never try to express a business rule in Lua.
- **Where to terminate auth** — the gateway proves the token is authentic, but Django still needs a `request.user`. Re-validating the JWT locally (Simple JWT's `JWTAuthentication`) is the safe default: cheap, self-contained, and still correct if somebody bypasses the gateway. Trust a signed identity header only over mTLS. Object-level checks (`has_object_permission`) never leave Django ([Authorization](../Django/Authorization.md), [OAuth](../Django/OAuth.md)).
- **`ALLOWED_HOSTS` and forwarded headers** — the gateway usually rewrites `Host`, so either add its hostname to `ALLOWED_HOSTS` or configure it to preserve the original. Every `X-Forwarded-*` rule from [ForwardAndReverseProxy](WebServer/ForwardAndReverseProxy.md) still applies, with one more trusted hop to count.

---
## 🧠 Summary

| Concept         | Takeaway                                                                                       |
| --------------- | ---------------------------------------------------------------------------------------------- |
| What it is      | A policy-aware **north-south** entry point: authn, quotas, validation, transformation, routing |
| Gateway vs mesh | North-south policy vs east-west reliability — different problems, different products           |
| BFF             | Client-specific shaping is an app you deploy, not a plugin in the gateway                      |
| PEP vs PDP      | The gateway enforces coarse decisions; the service still authorizes its own objects            |
| Never           | Business logic in the gateway, a public admin API, or blind trust in injected identity headers |

---
