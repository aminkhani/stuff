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
## 🧩 What is a Proxy?

A **proxy** terminates one connection and opens a second one on somebody's behalf. Only one distinction really matters:

- **Forward proxy** → acts for the **client**. The client is configured to use it; the destination server never knows it exists. Your **personal assistant** who places the calls for you.
- **Reverse proxy** → acts for the **server**. The client believes the proxy *is* the server; the real backend never sees the client's socket. The **company receptionist** 📞 who answers every call and routes it to the right desk.

Same software, opposite direction of trust. This note extends [WebServer](WebServer.md) and [Nginx](Nginx.md).

---
## 💡 Why do we need it?

**Forward proxy** solves *outbound* problems:

- 🚧 **Egress control / allowlisting** — private-subnet hosts may reach `pypi.org` and your registry, nothing else.
- 🔍 **Corporate filtering + audit** — one place that logs "who fetched what", one place to enforce policy.
- 📦 **Caching package mirrors** — one download of a 300 MB layer serves 50 build agents.
- 🕶️ **Hiding client identity, and reaching the internet with no public route** — the origin sees the proxy's IP, and hosts without a public IP or NAT gateway still get out.

**Reverse proxy** solves *inbound* problems:

- 🔒 **TLS termination** — certs in one place, not in every app ([TransportSecurity](../../Security/TransportSecurity.md)) — plus ⚖️ **load balancing** ([LoadBalancer](../LoadBalancer.md))
- 🗄️ **Caching, compression, static file serving** — never let Django serve `/static/` in production
- 🧭 **Path / host routing** — one public IP, many apps; 🐤 **canary / blue-green** by weight or header
- 🐌 **Request buffering** — absorbs slow uploads and slowloris, so a worker is never blocked reading bytes
- 🛡️ **WAF placement** and 🙈 **hiding internal topology** — the internet learns one hostname, not your subnet map

---
## ⚙️ How it works

```
FORWARD PROXY  (acts for the CLIENT - outbound)
  [ client ] --> [ forward proxy ] --> ( internet ) --> [ any server ]
   configured        allowlist,                          sees the proxy IP,
   to use it         cache, audit log                    never the client

REVERSE PROXY  (acts for the SERVER - inbound)
  [ client ] --> [ reverse proxy ] --> [ app1 ] [ app2 ] [ /static ]
   thinks this       TLS, routing,       private subnet,
   IS the server     cache, WAF          no public IP
```

1. The client opens a TCP (usually TLS) connection **to the proxy**, not to the destination.
2. The proxy reads the request line and headers; a reverse proxy normally **buffers** the whole request first.
3. It makes a decision — *is this destination allowed?* (forward) or *which upstream owns this route?* (reverse).
4. It opens a **second, independent** connection onward, usually from a keep-alive pool, then streams the response back. At that point the original client IP survives **only** in headers — which is why the next section exists.

---
## 🧬 The forwarded-header chain

The backend's socket only shows the proxy's IP, so proxies re-inject the facts they destroyed as headers:

| Header | Status | Carries | Example value |
|--------|--------|---------|---------------|
| `X-Forwarded-For` | de-facto standard | client IP, **appended per hop** | `203.0.113.7, 10.0.1.5` |
| `X-Forwarded-Proto` | de-facto standard | the original scheme | `https` |
| `X-Real-IP` | Nginx convention only | one client IP, no list | `203.0.113.7` |
| `Forwarded` | RFC 7239 — the actual standard | all of the above in one header | `for=203.0.113.7;proto=https` |

> [!WARNING]
> **These are ordinary request headers, so clients can send them.** `curl -H 'X-Forwarded-For: 1.2.3.4'` is a perfectly legal request. They are trustworthy only for hops **you** control, and only if your outermost proxy **overwrites** instead of appending.

**Trusted-proxy counting** is the fix: you know exactly *N* proxies sit in front of you, so you trust the **Nth entry from the right** of `X-Forwarded-For` and discard everything to its left. Get *N* wrong and you get one of two classic bugs:

- 🚨 **Spoofing → rate-limit bypass** — you read the *leftmost* value, the attacker rotates it every request, and your per-IP throttle counts one request per "client" forever. The same bug defeats IP allowlists and poisons audit logs.
- 🚨 **Wrong bans / self-DoS** — you read too far right, every request looks like it came from the proxy, and one abuser gets the whole internet blocked (or your admin IP allowlist matches everybody).

---
## 🧪 Example

**Reverse proxy** — Nginx terminating TLS in front of Gunicorn/Django:

```nginx
# /etc/nginx/conf.d/api.conf
upstream django_app {
    server 10.0.1.11:8000;
    server 10.0.1.12:8000;
    keepalive 32;
}
server {
    listen 443 ssl;
    http2 on;                    # nginx >= 1.25.1 (older syntax: listen 443 ssl http2;)
    server_name api.example.com;
    ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    server_tokens off;           # stop advertising "nginx/1.27.0"
    client_max_body_size 10m;
    location /static/ { alias /srv/app/static/; expires 30d; }
    location / {
        proxy_pass         http://django_app;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $remote_addr;   # OVERWRITE - we are the edge
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 5s;
        proxy_read_timeout    60s;
        proxy_buffering       on;   # the default, and it is your slowloris shield
    }
}
```

**Forward proxy** — Squid enforcing an egress allowlist for `10.0.1.0/24`:

```ini
# /etc/squid/squid.conf
http_port 3128
acl internal_net src 10.0.1.0/24
acl allowed_dst  dstdomain .pypi.org .files.pythonhosted.org .docker.io
acl SSL_ports    port 443
acl CONNECT      method CONNECT
http_access deny  !internal_net
http_access deny  CONNECT !SSL_ports       # no CONNECT tunnel to arbitrary ports
http_access allow internal_net allowed_dst
http_access deny  all                      # default-deny, and it must be last
```

```bash
export HTTPS_PROXY=http://10.0.1.5:3128 NO_PROXY=localhost,127.0.0.1,10.0.0.0/8
pip install django    # allowlisted host -> works; anything else -> 403 from Squid
```

> [!NOTE]
> On Windows use `$env:HTTPS_PROXY = "http://10.0.1.5:3128"` in PowerShell — and remember `pip`, `git` and `docker` each keep their own proxy config, so the env var alone is often not enough.

---
## ⚖️ Proxy vs the neighbours

| Thing | Acts for | Where it sits | What it terminates | Typical product |
|-------|----------|---------------|--------------------|-----------------|
| **Forward proxy** | the client | egress edge of your network | TCP/HTTP; TLS only as a `CONNECT` tunnel (or MITM with your own CA) | Squid, tinyproxy, Envoy |
| **Reverse proxy** | the server | ingress edge, in front of apps | TLS + HTTP | Nginx, HAProxy, Caddy, Envoy |
| **NAT gateway** | the client | L3 routing layer | nothing — rewrites IP/port, no URL policy | AWS NAT GW, `iptables MASQUERADE` |
| **VPN** | client *and* network | L3/L4 tunnel | its own crypto, not your app's TLS | WireGuard, OpenVPN, IPsec |
| **Load balancer** | the server | in front of a server pool | an L4 connection, or L7 + TLS | AWS NLB/ALB, HAProxy, MetalLB |
| **API gateway** | server + platform team | in front of many APIs | TLS + HTTP + auth + quota | Kong, APISIX, AWS API Gateway |
| **CDN** | the server, globally | ISP edge, near the user | TLS + HTTP, and caches the response | Cloudflare, Fastly, CloudFront |

**Pick by asking "who am I hiding, and from whom?"** Hiding clients → forward proxy. Hiding servers → reverse proxy. Only address translation → NAT. The whole network rather than one protocol → VPN. Spreading load → [LoadBalancer](../LoadBalancer.md). Per-consumer auth and quotas → [APIGateway](../APIGateway.md). Geography → CDN.

---
## 🚨 When NOT to use it

- ❌ **One more hop in front of a single internal service** — a reverse proxy for one internal API that already speaks TLS only buys you latency, a second timeout budget, and another thing to misconfigure.
- ❌ **An open forward proxy — never.** An unauthenticated proxy reachable from the internet is found within hours and used for credential stuffing, spam and scanning **from your IP**. You get the abuse reports and the blocklist entry, not the attacker.
- ❌ **Long-lived WebSocket / gRPC streams on a default config** — the default 60 s `proxy_read_timeout` kills idle sockets mid-stream, and gRPC needs end-to-end HTTP/2, so use `grpc_pass` rather than `proxy_pass` ([gRPCTips](../../Python/gRPCTips.md)).

---
## 🔐 Security notes & production hardening

- 🔑 **Never run an open proxy.** Bind the forward proxy to the private interface, keep `http_access deny all` as the final rule, and require authentication if it is reachable outside the subnet.
- 🧹 **Strip inbound `X-Forwarded-*` at the edge.** Overwrite with `$remote_addr` / `$scheme`; use `$proxy_add_x_forwarded_for` only on *inner* hops, never on the outermost one. `Forwarded` and `X-Real-IP` get the same treatment.
- 🏷️ **Validate `Host`.** Nginx `server_name` plus a `default_server` that returns `444`, and Django `ALLOWED_HOSTS`. Without it, Host-header poisoning turns into password-reset links pointing at the attacker's domain.
- ⏱️ **Bound everything:** `client_max_body_size`, `client_body_timeout`, `proxy_connect_timeout`, `limit_req_zone`, `limit_conn_zone`. An unbounded proxy is a free amplifier.
- 🙈 **Do not leak upstream internals:** `server_tokens off;`, `proxy_intercept_errors on;` with your own `error_page`, and hide upstream `Server` / `X-Powered-By` (`proxy_hide_header`). A raw Gunicorn traceback page is a free architecture diagram.
- 🏰 **Put it in the DMZ** — proxy in the DMZ, app and DB in the private zone, one-way firewall rules ([DMZ](../../Linux/DMZ.md)). The proxy is also **not** an authorization boundary: it can be bypassed if anything else can route to the app, so authorize in the service too.
- 🕳️ **SSRF cuts both ways:** when your own app fetches user-supplied URLs *through* a proxy, that proxy's allowlist becomes your SSRF control. Without one, `http://169.254.169.254/` is a single user-supplied URL away.

---
## 🐍 Django / Backend tie-in

- **`SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')`** makes `request.is_secure()` believe that header. Safe **only** if the proxy always overwrites it and nothing can reach Gunicorn directly — otherwise a client sends `X-Forwarded-Proto: https` over plain HTTP and your redirect, secure cookies and HSTS logic all silently disengage.
- **`ALLOWED_HOSTS`** must list real hostnames, not `['*']`, because the proxy passes `Host` straight through. Set `USE_X_FORWARDED_HOST = True` only when the edge sets `X-Forwarded-Host` itself; otherwise leave it off and forward the real `Host`.
- **Real client IP for throttling:** `REMOTE_ADDR` is the proxy, and DRF's `AnonRateThrottle` keys on it — so behind a proxy every anonymous user shares one bucket. Fix it with middleware that reads `X-Forwarded-For` **by trusted-proxy count**, never `split(',')[0]` ([Authentication](../../Django/Authentication.md)).

> [!IMPORTANT]
> Gunicorn has its own `forwarded_allow_ips` (default `127.0.0.1`). If Nginx runs on a different host or container, Gunicorn ignores its `X-Forwarded-*` headers entirely until you set it — the usual cause of "Django keeps redirecting to `http://`".

---
## 🧠 Summary

| Concept | Takeaway |
|---------|----------|
| Forward vs reverse | Forward acts for the **client**, reverse acts for the **server**. Everything else follows from that. |
| `X-Forwarded-*` | Client-supplied, therefore spoofable. Overwrite at the edge; trust by hop count. |
| Reverse-proxy wins | TLS, load balancing, cache, static files, buffering, WAF and routing, all in one place |
| Never | An open forward proxy — or treating any proxy as your only authorization layer |

---
