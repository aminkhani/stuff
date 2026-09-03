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
## 🧩 What is Transport Security?

**Transport security** wraps a plaintext protocol (HTTP, SMTP, the PostgreSQL wire protocol, gRPC) in an encrypted, integrity-protected tunnel, so anyone on the path sees only ciphertext. **TLS (Transport Layer Security)** is that tunnel.

Think of it as a **tamper-evident armored envelope 📦** around every packet: the courier still reads the _address_ (IP, and the hostname via SNI) but not the letter, and cannot swap a page without you noticing.

**SSL vs TLS — the naming reality:** SSL 2.0/3.0 are the dead ancestors (DROWN, POODLE), disabled everywhere. Everything in use today is **TLS**: 1.0/1.1 deprecated by RFC 8996, **1.2** the practical floor, **1.3** (RFC 8446) the target. "SSL" survives only in names — Python's `ssl` module, Nginx's `ssl_certificate`, "SSL certificate" on every CA's pricing page. Nobody ships SSL; everybody says SSL.

---
## 💡 What TLS actually guarantees — and what it does not

| ✅ TLS gives you                                                                | 🚫 TLS does **not** give you                                     |
| ------------------------------------------------------------------------------ | ---------------------------------------------------------------- |
| **Confidentiality** on the wire                                                | **Client identity** — unless you add mTLS                        |
| **Integrity** — AEAD tags catch any modification                               | **Authorization** — who is allowed to do what                    |
| **Server identity** — the peer holds the private key for a name you trust      | **Application security** — SQLi, IDOR, XSS are untouched         |
| **Replay/reorder protection** inside the session                               | **Encryption at rest** — plaintext the moment it lands           |
| **Forward secrecy** — a stolen key can't decrypt captured traffic (always 1.3) | **Metadata privacy** — IPs, sizes, timing, SNI hostname leak     |

> [!IMPORTANT]
> A padlock only means "this connection reached the holder of a certificate for this name". It says nothing about whether that server is honest, patched, or authorized to hold your data. TLS protects the pipe, not the endpoints — in [CIA](./CIA.md) terms it buys **confidentiality + integrity in transit**, and nothing else.

---
## ⚙️ How the handshake works — TLS 1.2 vs 1.3

```text
TLS 1.3 - one round trip
  C -> S : ClientHello (versions, ciphers, SNI, key_share)
  S -> C : ServerHello (key_share) + {Certificate, CertificateVerify, Finished}
  C -> S : {Finished} + {application data}          {...} = already encrypted

TLS 1.2 - two round trips before any application data
  C->S ClientHello            | S->C ServerHello + Certificate + HelloDone
  C->S ClientKeyExchange+CCS+Finished | S->C ChangeCipherSpec + Finished
```

1. **Hello** — the client offers versions, cipher suites and the SNI hostname; in 1.3 it also guesses a **key share** up front, which is what saves the round trip.
2. **Key agreement** — both sides run **(EC)DHE** and derive the same secret; a passive observer cannot compute it.
3. **Server authentication** — the server sends its chain and _signs the handshake transcript_ with the matching private key. This is the only step that binds the crypto to an identity.
4. **Finished** — each side hashes the whole transcript and compares, so a downgrade or an injected message breaks the handshake.

What actually changed in **TLS 1.3**:

- **1-RTT** by default (1.2 needs 2), and the certificate itself is sent **encrypted**.
- **RSA key transport is gone** — **PFS is mandatory**, ECDHE/DHE only, so stealing the server key no longer decrypts captured traffic (which is also why passive TLS-inspection appliances stopped working). Legacy went with it: RC4, 3DES, CBC modes, TLS compression, renegotiation, custom DH groups.
- **0-RTT / early data** lets the client send payload in the first flight — but it is **replayable** and has no forward secrecy. Idempotent reads only, never `POST /transfer`.

---
## 📜 X.509 certificates — what a cert really proves

A certificate is just a **public key + a set of names + validity dates, signed by someone else**. It proves exactly one thing: _a CA was willing to assert that this public key belongs to these names_. The **private key** is the real secret; the cert is public by design.

```text
Root CA (self-signed, shipped in the OS/browser trust store, kept offline)
   |__ signs -> Intermediate CA (online, revocable, rotated)
          |__ signs -> Leaf / end-entity cert (api.example.com)
```

The server must send **leaf + intermediates**. A missing intermediate is the classic "works in Chrome, fails in `curl` and Python" bug — browsers guess, libraries do not. The root is never sent; the client must already trust it.

- **SAN vs CN** — hostname validation reads the **subjectAltName** extension. **CN is deprecated** and ignored by modern clients (Chrome since 58, Go, current OpenSSL). A cert with only a CN fails today.
- **Validation levels & lifetime** — **DV** proves control of the domain (automated, minutes), **OV** adds a light organization check, **EV** adds a heavy legal one but browsers removed the special UI, so EV now buys paperwork rather than security. All public certs are capped at ~398 days and the CA/Browser Forum keeps pushing that down, so **automated renewal is not optional**.
- **Wildcards** — `*.example.com` matches **exactly one label**: `api.example.com` ✅, `a.b.example.com` ❌, bare `example.com` ❌ (it needs its own SAN). And one wildcard key shared across many hosts means one compromise covers all of them.

---
## 🧪 Example — key, CSR, inspection, ACME

```bash
# 1) Private key: ECDSA P-256 (smaller + faster than RSA-2048, universally supported)
openssl ecparam -name prime256v1 -genkey -noout -out api.key
chmod 600 api.key
# 2) CSR with a real SAN - modern clients ignore CN entirely
openssl req -new -key api.key -out api.csr \
  -subj "/C=IR/O=Example Ltd/CN=api.example.com" \
  -addext "subjectAltName=DNS:api.example.com,DNS:www.example.com"
# 3) Self-signed, LOCAL DEV ONLY (OpenSSL 3.x keeps the SAN via -copy_extensions)
openssl x509 -req -in api.csr -signkey api.key -days 365 \
  -copy_extensions copyall -out api.crt
# 4) What does it claim, and when does it expire?
openssl x509 -in api.crt -noout -subject -issuer -dates -ext subjectAltName
# 5) Interrogate a live endpoint: chain, negotiated version, cipher
openssl s_client -connect api.example.com:443 -servername api.example.com </dev/null
# 6) Production: ACME / Let's Encrypt - 90-day certs that renew themselves
sudo certbot --nginx -d api.example.com -d www.example.com
sudo certbot renew --dry-run    # prove renewal works BEFORE day 89
```

> [!NOTE]
> On Windows run these from **Git Bash** (`openssl` ships with Git for Windows) — the `\` continuations and `</dev/null` are POSIX-only, and `chmod 600` is a no-op on NTFS.

---
## 🤝 mTLS — when the client must prove itself too

In one-way TLS only the server authenticates. With **mTLS (mutual TLS)** the server sends a `CertificateRequest`, and the client presents its own certificate and signs the transcript too — so the connection now carries a **cryptographic client identity**, usually a service rather than a human.

- **Service-to-service** on a private network: no shared bearer secret sitting in an env var. Same for admin planes — `etcd`, the Kubernetes API server, [Vault](../DevOps/Vault/README.md).
- **Kubernetes service mesh** (Istio, Linkerd): sidecars issue, rotate and validate short-lived certs for you. This is the only place mTLS is close to free.
- **Bank / PSD2 / partner APIs** whose contract mandates a client certificate.

The cost is **not the crypto, it is the PKI lifecycle** — issue a cert per client, distribute it safely, rotate before expiry, revoke on compromise, and debug `alert bad certificate` with no useful error text. Hand-rolled mTLS across 40 services with 1-year certs is a scheduled outage. Without automation, prefer one-way TLS plus a strong bearer token.

> [!WARNING]
> mTLS authenticates, it does not authorize. A valid client cert tells you _which_ service is calling, never _what it may do_ — map the cert Subject/SAN to a role and enforce policy above it (see [Access Control](./AccessControl.md)).

---
## ⚖️ One-way TLS vs mTLS vs VPN vs SSH tunnel vs app-layer signing

| Approach                                | Authenticates          | Scope                       | Ops cost                    | Pick it when                                                                     |
| --------------------------------------- | ---------------------- | --------------------------- | --------------------------- | -------------------------------------------------------------------------------- |
| **One-way TLS**                         | server only            | per connection              | low (ACME automates it)     | Anything public-facing. This is the default.                                      |
| **mTLS**                                | server **and** client  | per connection              | high without a mesh/CA      | Service-to-service, mesh, partner APIs that mandate client certs                  |
| **VPN** (WireGuard, IPsec)              | the device/peer        | the whole network (L3)      | medium + routing/DNS        | You need many ports or non-TLS protocols on a private network                      |
| **SSH tunnel / port-forward**           | the human's key        | one port, ad hoc            | trivial but manual          | Debugging, one-off DB access from your laptop                                     |
| **App-layer signing** (JWS, HMAC hooks) | the **message**        | per message, survives hops  | medium (key distribution)   | You need non-repudiation, or integrity that outlives the connection (webhooks)     |

These compose rather than compete: a webhook can be HMAC-signed **inside** TLS, and a mesh runs mTLS **inside** a VPN. Rule of thumb — **TLS for the channel, signing for the message, VPN for the network, SSH for humans.**

---
## 🔀 Termination vs passthrough vs re-encrypt

| Mode                            | TLS ends at           | Backend receives      | Trade-off                                                                       |
| ------------------------------- | --------------------- | --------------------- | ------------------------------------------------------------------------------- |
| **Termination** (offload)       | the LB / Nginx        | plaintext HTTP        | One cert to renew, full L7 routing, WAF, caching — but a plaintext internal hop and the backend must trust forwarded headers |
| **Passthrough** (TCP/L4)        | the app itself        | real TLS + client cert| True end-to-end, mTLS reaches the app — but the LB is blind: no L7 routing, no header rewriting, no caching |
| **Re-encrypt** (bridging)       | LB, then a new session| TLS from the LB       | L7 features **and** an encrypted internal hop — but two cert sets and double handshake cost |

Terminate for ordinary web traffic; passthrough when the app must see the client certificate itself; re-encrypt when a control says "encrypted in transit everywhere". See [Load Balancer](../SoftwareDesign/LoadBalancer.md) and [Nginx](../SoftwareDesign/WebServer/Nginx.md).

---
## 🚨 When NOT to use it / where it stops helping

- ❌ **As your only control, or as protection at rest** — TLS plus a vulnerable endpoint is just an encrypted exploit, and once the handshake ends the payload is plaintext in RAM, logs and the DB.
- ❌ **0-RTT on state-changing requests** — replayable by design. And ❌ **a wildcard for everything**: one key, one blast radius; use per-service certs where you can.
- ❌ **mTLS without an automated CA, or self-signed certs in production** — manual distribution fails at renewal rather than at rollout, and self-signed certs end with someone disabling verification, which is the habit that actually kills you. Run an internal CA.
- ❌ **TLS instead of network segmentation** — a reachable admin port is still reachable; put it behind a [DMZ](../Linux/DMZ.md) or private subnet.
- ⚠️ Ancient clients (old Android, Java 7, embedded devices) may not speak TLS 1.2+. That is a client-upgrade problem, not a reason to re-enable TLS 1.0.

---
## 🔐 Security notes & production hardening

- **Protocols/ciphers:** `ssl_protocols TLSv1.2 TLSv1.3;` — SSLv3 and TLS 1.0/1.1 off. Ban **RC4, 3DES, NULL and export** suites; prefer AEAD (`AES-GCM`, `CHACHA20-POLY1305`) with ECDHE.
- **Private keys:** `chmod 600`, owned by root and readable only by the service user. **Never in git**, never baked into a Docker image layer — add `*.key` and `*.pem` to `.gitignore` on day one.
- **HSTS:** `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`. Ramp `max-age` up gradually, and treat **preload as one-way** — once on the browser list every subdomain must stay HTTPS, and removal takes months.
- **Expiry + revocation:** enable `ssl_stapling on; ssl_stapling_verify on;` so clients don't phone the CA and leak their browsing. Then remember that **an expired cert is the classic outage** — alert at 30/14/7 days from an *external* probe, never from the host that owns the cert, and keep `certbot renew --dry-run` in a timer.
- **Never weaken the client side:** `verify=False`, `curl -k`, `InsecureSkipVerify`, `NODE_TLS_REJECT_UNAUTHORIZED=0` reduce TLS to obfuscation — install your internal CA bundle instead. And note **SNI is plaintext** even in TLS 1.3, so "we use HTTPS" ≠ "nobody sees which hosts we reach".
- **Verify, don't assume:** `nmap --script ssl-enum-ciphers -p 443 api.example.com`, `testssl.sh`, or SSL Labs for public hosts.

---
## 🐍 Django / Backend tie-in

```python
# settings.py - production only; these break plain-http local dev
SECURE_SSL_REDIRECT = True                  # 301 http -> https
SECURE_HSTS_SECONDS = 31536000              # start at 3600, then ramp up
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True                  # only if EVERY subdomain is HTTPS forever
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")  # ONLY behind a proxy
```

**The `SECURE_PROXY_SSL_HEADER` trap:** Django now believes _any_ request carrying `X-Forwarded-Proto: https`. If a client can reach Gunicorn directly, or the proxy _forwards_ that header instead of overwriting it, an attacker simply sets it — `request.is_secure()` returns `True`, `SECURE_SSL_REDIRECT` stops redirecting, and "secure" cookies go out in plaintext. Fix both ends: `proxy_set_header X-Forwarded-Proto $scheme;` in [Nginx](../SoftwareDesign/WebServer/Nginx.md) (overwrite, never pass through) **and** bind Gunicorn to `127.0.0.1` or a UNIX socket.

Outbound calls: `requests` verifies by default, so keep `verify=True`; use `verify="/etc/ssl/certs/internal-ca.pem"` for an internal CA and `cert=("client.crt", "client.key")` for mTLS. Finally, `python manage.py check --deploy` flags every missing `SECURE_*` setting — run it in CI, not on the day of the audit.

---
## 🧠 Summary

| Concept              | Takeaway                                                                                   |
| -------------------- | ------------------------------------------------------------------------------------------ |
| **SSL vs TLS**       | SSL is dead; you always mean **TLS 1.2/1.3** even when the docs say "SSL"                   |
| **What TLS proves**  | The peer holds the key for a name — *not* client identity, *not* authorization              |
| **TLS 1.3**          | 1-RTT, encrypted certificate, PFS mandatory, no RSA key transport; 0-RTT is replayable       |
| **Certificates**     | Public key + SANs signed by a CA; root → intermediate → leaf, and you must ship intermediates|
| **mTLS**             | Adds a client identity; the hard part is cert lifecycle — automate it or skip it             |
| **Biggest real risk**| An expired certificate, not a weak cipher. Monitor expiry from outside.                      |




