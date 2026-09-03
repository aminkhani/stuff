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
## 🧩 What is a Load Balancer?

A **load balancer (LB)** sits in front of a pool of interchangeable backends and decides, per connection or per request, which one serves it. Mechanically it is a **reverse proxy plus a scheduling policy plus a health model** ([ForwardAndReverseProxy](WebServer/ForwardAndReverseProxy.md)).

Think of the **queue manager at a bank** 🏦: everyone comes through one door, and one person sends each customer to whichever teller is free — and quietly stops sending anyone to the teller who just went on break.

---
## 💡 Why do we need it?

- 🟢 **Availability** — a backend dies, the LB stops sending it traffic, users never find out.
- 📈 **Horizontal scale** — add a fourth app server instead of buying a bigger third one.
- 🚀 **Zero-downtime deploys** — drain node 1, upgrade it, put it back, repeat. Blue-green and canary are just weighted pools.
- 🧯 **Overload control** — one place for `maxconn`, queueing and rate limits, so backends degrade instead of tipping over.
- 🔒 **One TLS endpoint** — certs, ciphers and HTTP/2 live at the edge ([TransportSecurity](../Security/TransportSecurity.md)).
- 🔭 **One observability point** — per-upstream latency and error rate, almost for free.

---
## ⚙️ L4 vs L7 — the first decision

| | **L4 (transport)** | **L7 (application)** |
|---|---|---|
| Can see | IP, port, TCP/UDP flags | method, path, headers, cookies, body |
| Balances | **connections** | **individual requests** |
| TLS | passthrough, or terminates blindly | terminates; can re-encrypt to the backend |
| Features | NAT/DSR, any protocol, enormous throughput | routing, rewrite, cache, compression, WAF, retries, header injection |
| Cost | cheapest per byte, near-zero added latency | CPU for TLS + parsing, roughly 0.5–2 ms added |
| Client IP | preserved by DSR, or via **PROXY protocol** | `X-Forwarded-For` / `Forwarded` |
| Typical | AWS NLB, F5, MetalLB, `haproxy mode tcp` | AWS ALB, Nginx, `haproxy mode http`, Envoy |

**Pick L4** for non-HTTP protocols (Postgres, SMTP, game traffic), for end-to-end TLS you must not break, or when you need raw packet throughput. **Pick L7** for anything you want to route, cache, retry or rate limit based on *content*.

---
## 🎲 Balancing algorithms

| Algorithm | Picks | Good for | Watch out |
|---|---|---|---|
| **Round robin** | the next server in order | uniform backends, short requests | a slow backend still gets its full share |
| **Weighted RR** | round robin biased by `weight=` | mixed instance sizes, canary by weight | weights go stale silently as the fleet changes |
| **Least connections** | fewest in-flight connections | requests with very variable duration | a freshly restarted node gets flooded — see slow-start |
| **Least response time / peak-EWMA** | lowest latency moving average | noisy or heterogeneous backends | `least_time` is NGINX **Plus** only; OSS approximates with `random two least_conn;`, Envoy uses P2C least-request, Linkerd uses peak-EWMA |
| **IP hash** | `hash(client IP)` | crude stickiness with no cookie | CGNAT and mobile clients defeat it; Nginx forbids `ip_hash` together with `backup` |
| **Consistent hashing** | `hash(key)` placed on a ring | **cache locality**, sharded caches | needs a stable key: `hash $request_uri consistent;` |

**Why consistent hashing matters:** with `hash(key) % N`, removing one of 5 nodes remaps roughly 80 % of keys — every cache in the fleet misses at the same moment and the origin absorbs the whole load. On a hash ring only about `1/N` of keys move, so losing a node costs a small, survivable miss rate instead of a stampede.

---
## 🩺 Health checks

- **Active** — the LB probes on a timer: HAProxy `check inter 2s fall 3 rise 2`. Finds a dead node with zero user traffic. OSS Nginx has **no** active HTTP check (`health_check` is NGINX Plus).
- **Passive** — the LB infers health from real traffic: Nginx `max_fails=3 fail_timeout=30s`, HAProxy `observe layer7 on-error mark-down`. Free, but the first users pay for the discovery.
- **Shallow** — `/healthz` returns 200 if the process can answer HTTP at all. This is what the LB should use.
- **Deep** — also pings DB, cache and broker. Excellent for deploy gating and alerting, dangerous as an LB probe.

> [!WARNING]
> **The deep health-check cascade.** Every backend shares one database. The DB stalls for 5 s, all deep checks fail **simultaneously**, the LB marks 100 % of the pool down and returns 503 to everyone — including requests that were cache hits and never needed the DB. The probes also pile extra queries onto the already-struggling DB. Keep the LB probe shallow, expose dependencies on a separate `/readyz` for deploys and alerts, and prefer fail-open behaviour (Envoy's `healthy_panic_threshold`, default 50 %: below that it ignores health status and spreads load across everything).

---
## 🧷 Stickiness, TLS, draining, retries, slow-start

- **Sticky sessions (affinity)** — cookie-based (HAProxy `cookie SRV insert indirect nocache`) or `ip_hash`. It works, and it is a **design smell**: load stops being even, draining a node logs its users out, autoscaling buys you less, and a single crash becomes user-visible. Fix the cause instead — move session state to Redis or signed cookies (Django: `SESSION_ENGINE = "django.contrib.sessions.backends.cache"`) and stay stateless. Keep affinity only for legacy apps you cannot change, or for consistent-hash cache routing where locality *is* the point.
- **TLS termination vs passthrough** — **terminate** when you want L7 features and central cert management; **re-encrypt** (bridging) when the internal hop crosses untrusted network; **passthrough** only when the app itself must see the client certificate, or policy forbids decrypting anywhere else. Passthrough forces L4, so there is nowhere to write `X-Forwarded-For` — use **PROXY protocol** ([TransportSecurity](../Security/TransportSecurity.md)).
- **Draining / graceful shutdown** — (1) mark the server `drain` so no new connections arrive, (2) let in-flight requests finish (deregistration delay), (3) `SIGTERM` and let Gunicorn retire its workers, (4) only then `SIGKILL`. LB membership is *eventually consistent*, so add a short pre-stop sleep or you will serve 502s for a few seconds after every deploy ([Service](../Linux/Service.md)).
- **Timeouts and retries** — keep `connect` low (1–5 s) and `read` as long as your slowest legitimate endpoint. Retry only **idempotent** requests: Nginx deliberately will not retry POST/PATCH/LOCK unless you add `non_idempotent` to `proxy_next_upstream`. Always add **jitter** and a **retry budget** (cap retries at a few percent of traffic) — a flat retry policy against a struggling backend triples its load and converts a blip into a metastable outage.
- **Slow-start** — a node rejoining the pool has cold caches, an empty DB connection pool and imports to warm. Ramp its weight: HAProxy `slowstart 30s`, NGINX Plus `slow_start=30s`, Envoy `slow_start_config`. OSS Nginx has none, so ramp weight manually or accept the latency spike.
- **The LB is itself a SPOF** — one HAProxy box is one outage. Run an active/passive pair sharing a floating **VIP** over `keepalived`/VRRP, or go active/active with **BGP anycast** and ECMP. DNS failover works but is bounded by TTL and resolver caching, so it is minutes, not seconds. Managed cloud LBs already do this for you.

---
## 🧪 Example

**Nginx (L7, passive health checks)** — extends [Nginx](WebServer/Nginx.md):

```nginx
# /etc/nginx/conf.d/app.conf
upstream app_pool {
    least_conn;                                    # or: hash $request_uri consistent;
    server 10.0.1.11:8000 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:8000 max_fails=3 fail_timeout=30s;
    server 10.0.1.13:8000 backup;                  # used only if both above are down
    keepalive 32;
}
server {
    listen 443 ssl;
    server_name app.example.com;
    ssl_protocols TLSv1.2 TLSv1.3;
    location / {
        proxy_pass http://app_pool;
        proxy_next_upstream       error timeout http_502 http_503;
        proxy_next_upstream_tries 2;
        proxy_connect_timeout 2s;
        proxy_read_timeout   30s;
        proxy_set_header Host            $host;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```

**HAProxy (L7, active checks + slow-start + stats bound to loopback):**

```ini
# /etc/haproxy/haproxy.cfg   (HAProxy 2.2+)
global
    stats socket /run/haproxy/admin.sock mode 660 level admin
defaults
    mode    http
    timeout connect 5s
    timeout client  30s
    timeout server  30s
    retries 2
    option  redispatch

frontend fe_https
    bind :443 ssl crt /etc/haproxy/certs/app.pem
    default_backend be_app

backend be_app
    balance leastconn
    option httpchk
    http-check send meth GET uri /healthz/
    http-check expect status 200
    default-server inter 2s fall 3 rise 2 slowstart 30s maxconn 200
    server app1 10.0.1.11:8000 check
    server app2 10.0.1.12:8000 check
listen stats
    bind 127.0.0.1:8404        # never plain :8404 - that is a public admin panel
    stats enable
    stats uri /stats
```

```bash
# take a node out of rotation without a reload (runtime API)
echo "set server be_app/app1 state drain" | socat stdio /run/haproxy/admin.sock
```

---
## ⚖️ Load balancing: all the options

| Option | Layer | Runs where | Strength | Pick it when |
|---|---|---|---|---|
| **Cloud / hardware L4** (AWS NLB, F5) | L4 | managed or appliance | static IP, millions of flows, TLS passthrough, HA built in | you need raw throughput or non-HTTP protocols |
| **Nginx / HAProxy** | L7 (or L4) | your VM or container | routing, cache, ACLs, rich stats, free | you want full control and can run the HA pair yourself |
| **Envoy** | L4 + L7 | edge **or** sidecar | dynamic xDS config, native HTTP/2 + gRPC, outlier detection, retry budgets | config must change at runtime, or gRPC is first-class |
| **DNS round robin** | DNS | nowhere | free, global, zero infrastructure | crude geo/site spread only — no health awareness, TTL caching skews it |
| **BGP anycast** | network | routers | global failover, absorbs DDoS, no single site | you own an ASN and need multi-region entry |
| **Client-side LB** (gRPC `round_robin`) | in the client | the caller's process | no extra hop, per-**request** balancing over HTTP/2 | gRPC or a service you control on both ends ([gRPC](../Python/gRPC.md)) |
| **Service mesh sidecar** (Istio, Linkerd) | L7 east-west | one proxy per pod | mTLS, retries, circuit breaking on every internal call | many services talking to each other and you accept the ops cost |
| **K8s `type: LoadBalancer`** / MetalLB | L4 | cloud controller, or MetalLB speaker | one line of YAML; MetalLB gives bare metal real external IPs | in-cluster workloads — [K8s LoadBalancer + MetalLB](../DevOps/Kubernetes/Networking&Services/LoadBalancer/README.md) |
| **K8s Ingress / Gateway API** | L7 | controller pods | host/path routing and TLS for many Services behind one LB | you would otherwise pay for one cloud LB per Service — [Ingress](../DevOps/Kubernetes/Networking&Services/AdvancedNetworking/Ingress/README.md) |

---
## 🚨 When NOT to use it

- ❌ **A single-instance app.** One backend behind an LB is a second thing to break and a second timeout to tune. Add it when you add the second instance.
- ❌ **When the database is the real bottleneck.** More app replicas behind an LB means *more* connections to the same Postgres, and you hit pool exhaustion faster. Fix the DB first — pooling, read replicas, indexes, caching.
- ❌ **When you need per-request policy** — token validation, quotas per consumer, request transformation. That is an [APIGateway](APIGateway.md), not a load balancer.
- ❌ **gRPC / HTTP-2 behind an L4 LB.** L4 balances *connections*, and gRPC keeps one long-lived HTTP/2 connection, so **every RPC lands on whichever backend won the initial coin flip** — traffic is not balanced at all, and new pods stay idle. Use client-side balancing over a headless Service, or an L7 proxy that actually terminates HTTP/2 (Envoy, `grpc_pass`).

---
## 🔐 Security notes & production hardening

- 🔒 **Terminate TLS properly:** `TLSv1.2`/`1.3` only, a modern cipher list, OCSP stapling, HSTS at the edge, and `HTTP/2`. The LB is where cert rotation happens, so automate it.
- 🚪 **Lock down the admin surface.** HAProxy `listen stats` → bind to `127.0.0.1` or a management VLAN, add `stats auth`; the runtime socket is `mode 660 level admin`, so it is root-equivalent. Nginx `stub_status` → `allow 127.0.0.1; deny all;`. An exposed LB stats page is a free inventory of your backends.
- 🚦 **Rate and connection limits at the edge:** Nginx `limit_req_zone` + `limit_conn_zone`; HAProxy stick-tables with `tcp-request connection reject if { src_conn_rate(st) gt 100 }`, plus `maxconn` per server so backpressure lands on the LB queue, not on Gunicorn.
- 🪪 **PROXY protocol vs `X-Forwarded-For`:** in `mode tcp`/passthrough there is no header to write, so use PROXY protocol (`server ... send-proxy`, Nginx `listen 443 proxy_protocol` + `set_real_ip_from <lb-ip>; real_ip_header proxy_protocol;`). ⚠️ A listener with `proxy_protocol` enabled **trusts whatever source IP the peer claims** — firewall it to the LB only.
- 🩺 **The health endpoint is public by necessity.** The LB cannot log in, so `/healthz/` must be unauthenticated — which means it must return nothing but `200` and a fixed body. No version string, no git SHA, no DB hostname, no migration state, no dependency list. Keep it off the access log and rate-limit it.
- 📊 **Observe per upstream, not just in aggregate:** 5xx rate and p95/p99 latency *per server*, active connections, health-check flaps, retry rate. One bad node hides completely in a fleet average ([Monitoring&Logging](../DevOps/Monitoring&Logging/README.md)).
- 🚧 **The LB is not an authorization boundary.** Anything that can route to a backend directly — another pod, a debug port, a misapplied security group — bypasses every rule you wrote here. Authenticate and authorize **in the service too** ([AccessControl](../Security/AccessControl.md)).

---
## 🧠 Summary

| Concept | Takeaway |
|---|---|
| L4 vs L7 | L4 balances connections and is blind; L7 balances requests and can read them |
| Algorithm | `least_conn` is the sane default; consistent hashing when cache locality matters |
| Health checks | Keep the LB's probe **shallow** — deep checks fail the whole fleet at once |
| Stickiness | A smell. Externalise session state and stay stateless |
| Retries | Idempotent only, with jitter and a budget, or you amplify the outage |
| The LB itself | A SPOF until you add a VIP pair or anycast — and never your only authz layer |

---
