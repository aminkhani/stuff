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
## 🎯 Why APIs get their own security topic

A web application ships HTML: the server decides what the user sees, and the browser enforces same-origin rules on top. An API ships **objects and operations**, to clients the server does not control — mobile apps, SPAs, scripts, other services. Three consequences follow, and they explain almost every API breach:

- **Every parameter is attacker-controlled.** `GET /api/invoices/1043` is a request for *someone's* invoice; nothing about it says it is yours.
- **There is no UI to hide behind.** "The button isn't shown for non-admins" is not access control; the endpoint is still there.
- **Clients are not browsers.** Rate limits, CORS and CSRF assumptions built for browsers do nothing against a script with a valid token.

> [!tldr] TL;DR
> Authorize **every object on every request** (BOLA is the #1 API vulnerability), authenticate with short-lived tokens over TLS, bound **everything** the client can grow (page size, upload size, request rate, query depth), treat serializers as an **allowlist** in both directions, and understand that CORS is a browser convenience — not a security control.

---
## 🚨 The OWASP API Security Top 10 (2023) as a checklist

| # | Risk | The one-line test |
| --- | --- | --- |
| API1 | **BOLA** — broken object level authorization | swap an ID for another tenant's ID: does it 200? |
| API2 | Broken authentication | can you brute-force login, reuse a leaked token, or forge a JWT (`alg=none`, weak secret)? |
| API3 | Broken object **property** level authorization | can the client write `is_staff` / read `password_hash` because the serializer took every field? |
| API4 | Unrestricted resource consumption | `?page_size=100000`, a 2 GB upload, 10 000 IDs in one filter |
| API5 | **BFLA** — broken function level authorization | can a normal user call `DELETE /api/users/7` or an admin-only route? |
| API6 | Unrestricted access to sensitive business flows | one script buying every ticket, or 10 000 signups — each request is "valid" |
| API7 | **SSRF** | does the API fetch a URL the client supplied? |
| API8 | Security misconfiguration | `DEBUG=True`, wildcard CORS, stack traces, default credentials |
| API9 | Improper inventory management | the forgotten `v1` and `staging` hosts nobody patches |
| API10 | Unsafe consumption of APIs | you validate *your* clients, then trust a third-party response completely |

Read it as a review checklist, not a taxonomy: five of the ten are authorization or limits, which is where to spend your effort. Threats that are specific to *your* business flows are worth modelling explicitly — see [Threat Modeling](./ThreatModeling.md).

---
## 🧱 Methods

### 🔐 Authentication — proving who is calling

| Mechanism | Good for | Watch out for |
| --- | --- | --- |
| **Session cookie** | first-party browser clients on the same site | needs CSRF protection; set `HttpOnly`, `Secure`, `SameSite=Lax` |
| **API key** | server-to-server, low sensitivity | it identifies the *app*, not a user; store a hash, prefix it so it can be revoked and scanned for in leaks |
| **Bearer JWT** | stateless APIs, short-lived access tokens | cannot be revoked before expiry — keep it minutes, not days; pin `alg`, verify `iss`/`aud`/`exp` |
| **OAuth 2.0 / OIDC** | third-party access, delegated scopes | use authorization code + PKCE; never the implicit flow — see [OAuth](../Django/OAuth.md) |
| **mTLS** | internal service-to-service | certificate lifecycle and rotation; see [Transport Security](./TransportSecurity.md) |

The mechanics of doing this in Django are in [Authentication](../Django/Authentication.md); the rules that generalise:

- **Verify the token's signature *and* its claims.** A syntactically valid JWT proves nothing until you have checked issuer, audience, expiry and algorithm against a fixed list.
- **Store tokens where script can't read them.** For browsers, an `HttpOnly` cookie beats `localStorage`: an XSS bug then steals a session it cannot exfiltrate.
- **Rate-limit and lock out on the auth endpoints specifically.** Credential stuffing is a login-volume problem, not a password-strength problem.
- **Rotate refresh tokens** and treat reuse of an old one as theft: revoke the whole family.
- **Never put secrets in a JWT payload.** It is signed, not encrypted — anyone can Base64-decode it.

### 🔑 Authorization — proving they may touch *this* object

Authentication is one check per request. Authorization is one check per **object and action**, and it is the one people forget.

```python
# ❌ BOLA: authenticated, therefore trusted
invoice = Invoice.objects.get(pk=pk)

# ✅ the object must belong to the caller's tenant — enforced in the query
invoice = get_object_or_404(Invoice, pk=pk, organization=request.user.organization)
```

- Scope the **queryset**, don't filter after fetching: `get_queryset()` returning `Model.objects.filter(owner=self.request.user)` makes the safe path the default one.
- **BFLA** is the same bug at the endpoint level: role checks must live on every route, including the ones only the admin UI links to.
- Unguessable IDs (UUIDs) are **not** authorization. They raise the cost of enumeration and nothing else.
- Pick a model and apply it consistently — RBAC, ABAC or policy-as-code — as laid out in [Access Control](./AccessControl.md) and [Authorization](../Django/Authorization.md).

### 🚦 Rate Limiting

Rate limiting is how an API says "not that fast" instead of falling over. It protects availability (the **A** in the [CIA triad](./CIA.md)), makes brute force and scraping expensive, and caps the cost of any single client. Apply it at three scopes at once:

- **Per Endpoint** — a limit per route, sized to what the route costs. Login, password reset, OTP send, search and report generation deserve far tighter budgets than a cached `GET`.
- **Per User/IP** — a fair-use quota keyed to identity where you have one (user, API key, tenant), falling back to IP for anonymous traffic. Key on the **user first**: one token behind a corporate NAT shouldn't punish everyone at that address.
- **Overall to mitigate DDoS** — a global ceiling and connection limit at the edge, so total load stays inside what the backend can serve regardless of how it is distributed.

**Algorithms**, and what each one actually does:

| Algorithm | Behaviour | Trade-off |
| --- | --- | --- |
| Fixed window | count per calendar minute, reset at the boundary | trivial to implement; allows a 2× burst across the boundary |
| Sliding window (log) | count over the trailing N seconds | accurate, more memory per client |
| **Token bucket** | tokens refill at a steady rate, a request costs one | allows a controlled burst — usually what you want for APIs |
| Leaky bucket (queue) | requests drain at a fixed rate | smooths traffic, adds latency instead of rejecting |
| Concurrency limit | N in-flight requests per client | the right control for expensive endpoints; protects thread pools |

**Where to enforce it:**

| Layer | Sees | Good at |
| --- | --- | --- |
| CDN / WAF | raw IPs, geography, bot signals | volumetric floods, before your bandwidth is spent |
| [API Gateway](../SoftwareDesign/APIGateway.md) / [Nginx](../SoftwareDesign/WebServer/Nginx.md) (`limit_req`) | keys, paths, tokens | per-key quotas across all instances, cheaply |
| Application (DRF throttles) | user, tenant, plan, object | business rules: "5 exports per day per organisation" |

Details that separate a working limiter from a decorative one:

- Respond **`429 Too Many Requests`** with a `Retry-After` header, and expose the budget (`RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`) so well-behaved clients back off instead of hammering.
- Keep counters in a **shared** store (Redis). A per-process in-memory counter multiplied by eight Gunicorn workers is an eight-times-larger limit than you configured.
- **Never trust `X-Forwarded-For`** unless your proxy overwrites it: it is a client-supplied header, so a spoofed value gives an attacker unlimited fresh identities. Configure the trusted proxy count and read the correct position.
- Limits are only half of the story: cost-weight expensive endpoints, and remember that "valid requests, sent 10 000 times" (API6, business-flow abuse) needs business rules — one signup per email, CAPTCHA or proof-of-work on the flows worth abusing.
- Log the rejections. A spike in `429`s or `401`s is one of the highest-signal alerts an API produces.

### 🌍 Cross-Origin Resource Sharing (CORS)

The **same-origin policy** stops script on `evil.com` from *reading* a response from `api.bank.com`. CORS is the mechanism by which a server voluntarily relaxes that rule for chosen origins.

```http
GET /api/me HTTP/1.1
Origin: https://app.example.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Vary: Origin
```

A **preflight** `OPTIONS` request happens first whenever the request isn't "simple" — a custom header like `Authorization`, a JSON content type, or a method other than `GET`/`HEAD`/`POST`. The server answers with `Access-Control-Allow-Methods`, `-Allow-Headers` and `Access-Control-Max-Age`, and only then does the browser send the real request.

> [!IMPORTANT]
> **CORS is not authorization, and it is not a defence against server-side attackers.** It restricts what *browser script on another origin* may read. `curl`, a mobile app or a Python script ignore it completely — they never had a same-origin policy to relax. If an endpoint must be protected, protect it with authentication and authorization; CORS only decides which web front-ends may call it.

The traps, in order of how often they cause an incident:

- **`Access-Control-Allow-Origin: *` together with credentials is invalid** — browsers reject it. The workaround people reach for (reflecting whatever `Origin` was sent, plus `Allow-Credentials: true`) is strictly worse than a wildcard: it turns *every* site into a trusted origin and lets any page make authenticated requests on the user's behalf.
- **Reflecting an origin after a sloppy check.** `origin.endswith("example.com")` matches `notexample.com`, and a regex without anchors matches almost anything. Use an explicit allowlist (`CORS_ALLOWED_ORIGINS`), never `CORS_ALLOW_ALL_ORIGINS = True` in production.
- **Forgetting `Vary: Origin`** when the header is dynamic: a cache (CDN or browser) will happily serve one origin's `Allow-Origin` header to another.
- **Assuming CORS replaces CSRF protection.** A cross-site `POST` with a session cookie is *sent* regardless of CORS — the browser just hides the response. Cookie-authenticated APIs still need CSRF tokens or `SameSite` cookies; token-in-header APIs are immune because the header is not attached automatically.
- Keep the allowlist per environment, and don't let a permissive dev config ship: wildcard CORS in production is textbook API8 (security misconfiguration).

### 🧹 Input — validate as an allowlist

Every field the client can send is part of your attack surface, so define what is *allowed* rather than blocking what looks bad.

- **Serializers are the boundary.** `fields = "__all__"` is mass assignment waiting to happen: today it means the six fields you have, tomorrow it means the `is_staff` column somebody adds. List fields explicitly, and mark derived ones `read_only`.
- **Types and ranges, not just presence.** `page_size=1e9`, a negative quantity, or a 5 000-character name are all "valid strings".
- **SSRF (API7):** any client-supplied URL you fetch must be checked against an allowlist of hosts, resolved and re-checked against private ranges (`169.254.169.254`, `127.0.0.0/8`, `10/8`), and fetched with redirects disabled and a timeout. Never fetch "any URL the user gave us".
- **Uploads:** cap the size, check the real type by **magic bytes** rather than the filename or `Content-Type`, store outside the web root, and never execute or `include` what was uploaded.
- **Injection is still injection.** Parameterised queries and the ORM for SQL; no `eval`/`pickle` on client data; no shell string interpolation.

### 📉 Output — don't over-share

Excessive data exposure is API3's other half, and filtering in the client does not count: the data still crossed the wire.

- Return the fields the endpoint needs. `password`, `token`, internal IDs, other users' emails and audit fields don't belong in a public representation.
- Different audiences get different serializers, rather than one serializer plus `if` statements.
- **Error responses leak too.** A stack trace, an ORM message or a SQL fragment tells an attacker your schema. Return a stable error code plus a correlation ID; log the detail server-side.
- Avoid **user enumeration**: "no such account" versus "wrong password", and a 404-vs-403 difference on objects, both answer questions you didn't mean to answer.

### 🧯 Bound every dimension the client can grow (API4)

| Dimension | Control |
| --- | --- |
| Result size | mandatory pagination with a **maximum** page size, enforced server-side |
| Batch/filter size | cap `?ids=` lists; a 10 000-item fan-out is an OOM — see [Futures](../Python/Futures.md) |
| Request body | a byte limit at the proxy *and* in the app (`DATA_UPLOAD_MAX_MEMORY_SIZE`) |
| Query complexity | for GraphQL: depth and complexity limits; for REST: cap nested expansions |
| Time | timeouts on every downstream call, and a total request timeout |
| Money | quotas on anything that costs per call (SMS, email, LLM tokens, third-party APIs) |

### 🪖 Transport, headers and configuration

- **TLS everywhere, no exceptions**, including internal traffic; HSTS on the public edge. Details and the certificate side in [Transport Security](./TransportSecurity.md).
- Never put a token, key or password in a **query string** — it lands in access logs, proxies and browser history. Headers only.
- Django settings that matter: `DEBUG = False`, a locked `ALLOWED_HOSTS`, `SECURE_SSL_REDIRECT`, `SECURE_HSTS_SECONDS`, `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`, `SECURE_PROXY_SSL_HEADER` behind a proxy, and `X_FRAME_OPTIONS`. `manage.py check --deploy` grades all of it.
- For a JSON API, `Content-Security-Policy: default-src 'none'`, `X-Content-Type-Options: nosniff` and `Cache-Control: no-store` on authenticated responses are cheap and correct.
- **Secrets come from a secret store, not the repo** — see [Vault](../DevOps/Vault/README.md) — and rotation must be possible without a redeploy.

### 🧾 Inventory, versioning and documentation (API9)

The endpoint nobody remembers is the one nobody patches. Keep a generated, authoritative list — an OpenAPI schema produced from the code, not written by hand — and make it part of the pipeline:

- Every host and environment in one inventory: `api`, `api-staging`, `v1` that "no client uses any more", the internal admin API.
- Version deliberately (`/api/v1/`) and **retire** old versions on a schedule; a deprecated version still serving traffic is production.
- Don't publish internal schemas or debug endpoints; turn off interactive docs on public deployments if they expose more than you intend.
- Put the whole surface behind a gateway that does authn, quotas and logging in one place — see [API Gateway](../SoftwareDesign/APIGateway.md) — and keep third-party responses under the same suspicion as client input (API10).

---
## 🧪 Example — a hardened DRF configuration

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",      # deny by default
    ],
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.UserRateThrottle",
        "rest_framework.throttling.AnonRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "anon": "30/hour",
        "user": "1000/day",
        "login": "5/min",                                  # scoped, see below
        "export": "10/day",
    },
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 50,
}
```

```python
# views.py — object scoping, a scoped throttle, explicit fields
from rest_framework import viewsets, serializers
from rest_framework.throttling import ScopedRateThrottle

class InvoiceSerializer(serializers.ModelSerializer):
    class Meta:
        model = Invoice
        fields = ["id", "number", "amount", "status", "created_at"]   # never "__all__"
        read_only_fields = ["id", "status", "created_at"]

class InvoiceViewSet(viewsets.ModelViewSet):
    serializer_class = InvoiceSerializer
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = "export"                    # per-endpoint budget

    def get_queryset(self):
        # the tenant filter lives here, so no action can forget it
        return Invoice.objects.filter(organization=self.request.user.organization)

    def perform_create(self, serializer):
        # owner comes from the token, never from the request body
        serializer.save(organization=self.request.user.organization)
```

```python
# CORS: an explicit allowlist, and caching that respects Origin
CORS_ALLOWED_ORIGINS = ["https://app.example.com"]     # never CORS_ALLOW_ALL_ORIGINS
CORS_ALLOW_CREDENTIALS = True
CORS_PREFLIGHT_MAX_AGE = 3600

# Bound the request itself
DATA_UPLOAD_MAX_MEMORY_SIZE = 5 * 1024 * 1024          # 5 MB
DATA_UPLOAD_MAX_NUMBER_FIELDS = 1000
```

> [!WARNING]
> DRF's throttles use the cache, so with `LocMemCache` each Gunicorn worker keeps its own counter and your "5/min" login limit is really 5 per minute *per process*. Point the cache at Redis before you rely on any of these numbers.

---
## 🩺 Testing and monitoring

- **Test authorization like a feature.** For every endpoint, a test where user B requests user A's object and expects 404/403. That single pattern catches BOLA regressions forever.
- **Fuzz against the schema.** `schemathesis` drives your OpenAPI spec and finds 500s and validation gaps; OWASP ZAP or Burp cover the HTTP layer; `bandit`/`semgrep` cover the code.
- **Log security events with intent**: authentication outcome, subject, endpoint, object ID, decision and rate-limit rejections. Alert on `401`/`403`/`429` spikes, one token touching many object IDs (enumeration), and traffic from one key across many accounts.
- **Never log the credential itself** — tokens, cookies and `Authorization` headers get redacted before they reach the log pipeline.

---
## 🚨 Common mistakes

| Mistake | Why it hurts |
| --- | --- |
| "Authenticated, therefore authorized" | BOLA: any user reads any object by changing an ID |
| Hiding an endpoint instead of protecting it | BFLA: the admin route works for everyone who guesses it |
| `fields = "__all__"` | clients write columns you never meant to expose, forever after every migration |
| Rate limiting on IP only | one NAT punishes real users, and one attacker rotates addresses |
| In-memory throttle counters | the limit silently multiplies by the number of workers |
| Trusting `X-Forwarded-For` | unlimited spoofed identities against every per-IP control |
| Reflecting `Origin` with credentials | every website becomes a trusted front-end for your API |
| Treating CORS as access control | non-browser clients ignore it entirely |
| Tokens in URLs or logs | credentials preserved in access logs, proxies and history |
| Verbose errors | your schema, paths and SQL, handed over for free |
| An unversioned, uninventoried surface | the endpoint nobody owns is the one that never gets patched |

---
## 🧠 Summary

| Area | Takeaway |
| --- | --- |
| Authentication | Short-lived tokens over TLS, claims verified, auth endpoints rate-limited separately |
| Authorization | Per object, per action, enforced in the queryset — the #1 API vulnerability class |
| Rate limiting | Per endpoint **and** per user/IP **and** globally; token bucket, shared counters, `429` + `Retry-After` |
| CORS | A browser relaxation of same-origin — an allowlist, `Vary: Origin`, never wildcard with credentials, never a substitute for authz |
| Input | Explicit allowlists; SSRF, uploads and injection are the sharp edges |
| Output | Serialize deliberately; errors carry a correlation ID, not a stack trace |
| Limits | Page size, body size, batch size, timeouts, cost quotas — all server-side |
| Configuration | `DEBUG=False`, secure cookies, HSTS, secrets in a store, `check --deploy` in CI |
| Inventory | Generated OpenAPI, versions retired on a schedule, one gateway in front |
| Verification | An authorization test per endpoint, schema fuzzing, and alerts on `401`/`403`/`429` |

---
## 📚 References

- [OWASP API Security Top 10 (2023)](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [OWASP REST Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html) · [Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [MDN — CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) · [Django deployment checklist](https://docs.djangoproject.com/en/stable/howto/deployment/checklist/) · [DRF throttling](https://www.django-rest-framework.org/api-guide/throttling/)
