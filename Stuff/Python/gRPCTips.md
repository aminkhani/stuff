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
## 🧰 Scope — what this adds to the basics

[gRPC](./gRPC.md) covers the happy path: write a `.proto`, generate stubs, serve, call. That quickstart deliberately takes three shortcuts that must not survive contact with production:

- `add_insecure_port()` — no TLS, no peer identity
- `ThreadPoolExecutor(max_workers=10)` — an arbitrary concurrency limit with no queue bound
- `stub.SayHello(request)` — **no deadline**, so the call can hang forever

This note is the checklist for everything after "hello world": deadlines, retries, schema evolution, error semantics, streaming, transport security, load balancing and observability.

> [!tldr] TL;DR
> Set a **deadline** on every call. Reuse **one channel** per process. Return **specific status codes**, never `INTERNAL` for everything. Never reuse a proto **field number**. Assume an L4 load balancer will pin all your traffic to a single backend.

---
## ⏱️ Deadlines — the habit that prevents outages

A gRPC call with no deadline is a resource leak with a nice API. The client thread, the server thread and the TCP connection all stay pinned until something else breaks.

```python
# Client: a deadline is a timeout in seconds, and it is per call
response = stub.GetItem(req, timeout=2.0)

# Server: the deadline propagated to you — stop early if it is gone
def GetItem(self, request, context):
    if context.time_remaining() < 0.05:        # no point starting
        context.abort(grpc.StatusCode.DEADLINE_EXCEEDED, "no time left")
    ...
    if not context.is_active():                # client hung up mid-work
        return items_pb2.Item()                # bail out, do not commit
```

- Deadlines are **absolute and propagated**: a service that receives a 2 s deadline and calls a third service passes the *remaining* time onward. That is what stops a slow leaf service from parking threads in five services at once.
- Pick the deadline from the caller's own budget (an HTTP request timeout of 5 s ⇒ internal calls get less), not from how long the method usually takes.
- A `DEADLINE_EXCEEDED` on the client says nothing about whether the server applied the change. Every mutating RPC needs an idempotency key if the caller may retry.

> [!WARNING]
> `timeout=` is easy to forget in exactly the code paths that matter — startup checks, health probes, admin scripts. Enforce it in a client interceptor that fills in a default deadline when the caller supplied none.

---
## 🔁 Retries, backoff and idempotency

gRPC has **transparent retries built into the channel** — configured as a service config JSON, not as code in every call site:

```python
SERVICE_CONFIG = {
    "methodConfig": [{
        "name": [{"service": "items.v1.ItemService", "method": "GetItem"}],
        "retryPolicy": {
            "maxAttempts": 4,
            "initialBackoff": "0.1s",
            "maxBackoff": "2s",
            "backoffMultiplier": 2,
            "retryableStatusCodes": ["UNAVAILABLE", "RESOURCE_EXHAUSTED"],
        },
    }],
}
channel = grpc.secure_channel(
    target, creds,
    options=[("grpc.service_config", json.dumps(SERVICE_CONFIG))],
)
```

| Rule | Why |
| --- | --- |
| Retry **only** `UNAVAILABLE` (and `RESOURCE_EXHAUSTED` if the server means "later", not "never") | these mean "the request did not reach or complete", so a retry is safe |
| Never blanket-retry `DEADLINE_EXCEEDED`, `ABORTED`, `INTERNAL` | the work may have already happened — you would double-charge someone |
| Retries live **inside** the deadline | the deadline is the hard budget; attempts stop when it expires, which is the behaviour you want |
| Add jitter and a cap | retries without jitter synchronise clients into a thundering herd against a service that is already sick |
| Make mutations idempotent | an `idempotency_key` field plus a uniqueness constraint turns "did it apply twice?" into a non-question |

> [!NOTE]
> `hedgingPolicy` (send the same request to several backends and take the first answer) is the opposite trade: lower tail latency, more load, and it is only correct for genuinely idempotent reads.

---
## 📐 Proto design & schema evolution

The wire format encodes **field numbers**, not names. That single fact drives every rule:

- **Never reuse or renumber a field.** Delete a field and `reserved 4;` (plus `reserved "old_name";`) so nobody re-adds the number later with a different type. An old client writing field 4 as `string` into a new `int64` field 4 produces garbage, not an error.
- **Keep 1–15 for hot fields** — those tags fit in one byte of tag overhead.
- **Every enum needs a zero value**, conventionally `*_UNSPECIFIED`. proto3 has no field presence for scalars by default, so 0 and "absent" are the same value; mark a field `optional` when you must tell them apart.
- **Version the package, not the message**: `package items.v1;`. Breaking changes go to `items.v2` and both are served during migration.
- **Wrap every RPC in its own request/response message** (`GetItemRequest`, not `string`). Adding a field later is then additive instead of a signature change.
- **Use the well-known types** — `google.protobuf.Timestamp`, `Duration`, `FieldMask` for partial updates. Avoid `Any` and `Struct`: they smuggle untyped JSON back into a typed system and defeat the point.
- **Renaming a field is safe on the wire and breaks your code**; changing a number is safe in your code and breaks the wire. Only one of those is recoverable.

---
## 🚦 Status codes and error details

`INTERNAL` for everything is the gRPC equivalent of a blank `500`: the caller cannot tell whether to retry, fix the request, or page someone.

| Code | Means | Client should |
| --- | --- | --- |
| `INVALID_ARGUMENT` | the request is wrong regardless of system state | fix the request; never retry |
| `FAILED_PRECONDITION` | request is fine, **state** is not (mailbox not empty, account not verified) | change state, then retry |
| `ABORTED` | concurrency conflict (optimistic lock, transaction abort) | re-read and retry at a higher level |
| `NOT_FOUND` / `ALREADY_EXISTS` | identity-level outcome | branch, don't retry |
| `UNAUTHENTICATED` | no or bad credentials | refresh the token, then retry once |
| `PERMISSION_DENIED` | authenticated, not allowed | stop — see [Access Control](../Security/AccessControl.md) |
| `RESOURCE_EXHAUSTED` | quota or rate limit | back off and retry with jitter |
| `UNAVAILABLE` | transient: server down, connection dropped | retry with backoff — the only safe default |
| `DEADLINE_EXCEEDED` | budget spent, outcome unknown | retry only if idempotent |
| `UNIMPLEMENTED` | method or version does not exist | a deploy/schema problem, not a runtime one |

Machine-readable detail belongs in the status payload, not in a prose message:

```python
from google.rpc import status_pb2, error_details_pb2, code_pb2
from grpc_status import rpc_status

def CreateItem(self, request, context):
    detail = error_details_pb2.BadRequest(field_violations=[
        error_details_pb2.BadRequest.FieldViolation(
            field="quantity", description="must be > 0")])
    status = status_pb2.Status(
        code=code_pb2.INVALID_ARGUMENT, message="validation failed")
    status.details.add().Pack(detail)
    context.abort_with_status(rpc_status.to_status(status))
```

> [!WARNING]
> The status **message** crosses a trust boundary. `str(exception)` in there leaks table names, file paths and SQL. Log the traceback server-side with a correlation id, and send the caller the id plus a clean message.

---
## 🌊 Streaming — and when not to

| Kind | Shape | Good for |
| --- | --- | --- |
| Unary | 1 → 1 | 95% of RPCs. Start here |
| Server streaming | 1 → N | large result sets, tailing logs, progress updates |
| Client streaming | N → 1 | uploads, batched telemetry |
| Bidirectional | N ↔ N | chat, interactive sessions, long-lived sync |

- HTTP/2 gives you real **flow control**: a slow consumer applies backpressure through the connection window instead of buffering in RAM. That works only if you actually iterate lazily — materialising the stream into a list throws the benefit away.
- In the **synchronous** Python server, one active RPC occupies one pool thread for its whole lifetime. Ten long-lived streams on a 10-worker pool means zero threads left for unary calls. Long streams belong on `grpc.aio`.
- A stream is not a message queue: it has no durability, no replay and no consumer groups. If a dropped connection means lost data, put a broker behind it — see [Kafka vs RabbitMQ](../SoftwareDesign/KafkaVsRabbitMQ.md).
- Keep streams alive deliberately: idle proxies and NAT devices kill quiet connections, so enable keepalive pings (`grpc.keepalive_time_ms`) and allow them server-side, or your "long-lived" stream dies every few minutes.

---
## 🧵 Python runtime specifics

| Topic | What to do |
| --- | --- |
| **Channel reuse** | Create **one channel per target per process** and share it — channels are thread-safe and manage their own connection pool. A channel per call means a TCP + TLS handshake per call |
| **Thread pool sizing** | The sync server dedicates one thread per in-flight RPC. Size it to your downstream capacity, not to CPU count — see [Futures](./Futures.md) |
| **Queue bound** | `grpc.server(..., maximum_concurrent_rpcs=N)` rejects overflow with `RESOURCE_EXHAUSTED` instead of queueing without limit. An unbounded queue is how a latency spike becomes an OOM |
| **Async server** | `grpc.aio.server()` for many concurrent or long-lived streams. Do **not** call blocking code (the Django ORM, `requests`) inside it without `run_in_executor` |
| **Fork safety** | A channel created **before** `fork()` is not usable in the child. Under Gunicorn/uWSGI, build channels lazily on first use in the worker, never at import time |
| **Message size** | Receive is capped at 4 MB by default. Raise `grpc.max_receive_message_length` deliberately — the cap is also a DoS control. Anything genuinely large should stream in chunks |
| **Compression** | `compression=grpc.Compression.Gzip` per channel or per call: a clear win for large text payloads, wasted CPU for small binary ones |
| **Generated code** | `*_pb2.py` files are build artifacts. Generate them in CI (or `make proto`) and keep the `.proto` as the source of truth |

> [!IMPORTANT]
> If a servicer method touches the Django ORM, it is running in a gRPC pool thread with its own DB connection. Call `django.db.close_old_connections()` at the start and end of the handler, or the process slowly accumulates dead connections until PostgreSQL refuses new ones.

---
## 🔐 Security

- **TLS is not optional.** `add_insecure_port` belongs in tests. Use `server.add_secure_port(addr, grpc.ssl_server_credentials(...))`, and for service-to-service traffic require client certs (`require_client_auth=True`) — the mTLS trade-offs are in [Transport Security](../Security/TransportSecurity.md).
- **Token auth rides in metadata, over TLS.** `grpc.composite_channel_credentials(channel_creds, call_creds)` refuses to attach call credentials to an insecure channel on purpose: a bearer token on a plaintext channel is a token you have given away.
- **Authenticate in an interceptor, authorize in the handler.** A `ServerInterceptor` is the right place to validate the token once; per-object checks still belong next to the data, exactly as with [Authorization](../Django/Authorization.md).
- **Metadata is not a safe place for secrets you didn't intend to send.** Interceptor logging that dumps all metadata will happily write `authorization` headers into your logs.
- **Turn reflection off in production.** `grpc_reflection` is a schema dump for anyone who can reach the port — great for `grpcurl` in dev, free reconnaissance in prod.
- **Message and stream limits are security controls**: `max_receive_message_length`, `maximum_concurrent_rpcs`, `grpc.max_concurrent_streams`, plus a rate limit at the proxy. Protobuf parsing of a hostile 200 MB message is a CPU denial-of-service.
- **Do not expose gRPC straight to the internet** without a gateway in front doing authn, quotas and inventory — see [API Gateway](../SoftwareDesign/APIGateway.md).

---
## 🕸️ Load balancing & the edge

HTTP/2 multiplexes every RPC of a channel onto **one long-lived TCP connection**. An L4 load balancer therefore balances *connections*, not calls: the first backend it picks gets all your traffic, and a freshly scaled-up pod receives nothing.

| Option | How it balances | Notes |
| --- | --- | --- |
| L7 proxy (Envoy, Nginx `grpc_pass`, HAProxy in HTTP/2 mode) | per request | the usual answer at the edge — also does TLS and quotas ([Load Balancer](../SoftwareDesign/LoadBalancer.md)) |
| Client-side LB (`dns:///svc:50051` + `round_robin`) | per request, in the client | needs DNS that returns **all** backend addresses (a headless service), and clients that re-resolve |
| Service mesh sidecar | per request, transparently | mTLS and retries for free, one more proxy per pod to operate |
| `max_connection_age_ms` on the server | forces periodic reconnects | the cheap fix that lets any LB rebalance eventually |
| Unix domain socket (`unix:///run/app.sock`) | n/a — same host | sidecar or local supervisor; see [Unix Domain Sockets](../Linux/UnixSockets.md) |

Browsers cannot speak gRPC directly: they need **gRPC-Web** plus a translating proxy, or a JSON/REST gateway generated from the same protos.

---
## 🩺 Health checks & observability

- Implement the standard health service (`grpc_health_v1`) and point your orchestrator's probe at it — a TCP check only proves the port is open, not that the servicer works.
- Interceptors are the one place to add metrics, tracing and structured logs for every RPC: record **method, status code, peer and latency**. Log the code, not a stringified exception.
- Set `GRPC_VERBOSITY=debug` and `GRPC_TRACE=http,call_error,connectivity_state` when a connection misbehaves; it prints the handshake and connectivity transitions that the Python API hides.
- Watch `channel.subscribe(...)` connectivity state in long-lived clients: a channel stuck in `TRANSIENT_FAILURE` fails calls instantly and looks like an application bug.

---
## 🧪 Example — hardened server and channel

```python
# server.py — TLS, bounded concurrency, health, interceptors, graceful stop
import signal
from concurrent import futures
import grpc
from grpc_health.v1 import health, health_pb2, health_pb2_grpc

def build_server() -> grpc.Server:
    with open("/etc/certs/tls.key", "rb") as k, open("/etc/certs/tls.crt", "rb") as c:
        key, crt = k.read(), c.read()
    with open("/etc/certs/ca.crt", "rb") as f:
        ca = f.read()

    creds = grpc.ssl_server_credentials(
        [(key, crt)], root_certificates=ca, require_client_auth=True)  # mTLS

    server = grpc.server(
        futures.ThreadPoolExecutor(max_workers=16, thread_name_prefix="rpc"),
        maximum_concurrent_rpcs=64,                    # shed load, don't queue it
        interceptors=[AuthInterceptor(), MetricsInterceptor()],
        options=[
            ("grpc.max_receive_message_length", 4 * 1024 * 1024),
            ("grpc.keepalive_time_ms", 30_000),
            ("grpc.max_connection_age_ms", 300_000),   # let the LB rebalance
        ],
    )
    ...
    return server
```

```python
def serve() -> None:
    server = build_server()

    # Health service: report NOT_SERVING before the port closes so the LB
    # drains traffic *before* in-flight RPCs are cut off.
    health_servicer = health.HealthServicer(
        experimental_non_blocking=True,
        experimental_thread_pool=futures.ThreadPoolExecutor(max_workers=1),
    )
    health_pb2_grpc.add_HealthServicer_to_server(health_servicer, server)
    health_servicer.set("items.v1.ItemService", health_pb2.HealthCheckResponse.SERVING)

    server.start()

    def _drain(signum, frame):
        health_servicer.enter_graceful_shutdown()      # probes start failing
        server.stop(grace=10).wait()                   # finish in-flight RPCs

    signal.signal(signal.SIGTERM, _drain)              # SIGTERM = `docker stop`
    server.wait_for_termination()
```

```python
# client.py — one channel per process, TLS, default deadline, retries
import grpc, json

class DeadlineInterceptor(grpc.UnaryUnaryClientInterceptor):
    """Fill in a deadline whenever a caller forgot one."""
    def __init__(self, default_timeout: float = 2.0):
        self._t = default_timeout

    def intercept_unary_unary(self, continuation, details, request):
        if details.timeout is None:
            details = details._replace(timeout=self._t)
        return continuation(details, request)

def build_channel(target: str) -> grpc.Channel:
    creds = grpc.ssl_channel_credentials(root_certificates=CA)
    call_creds = grpc.access_token_call_credentials(get_token())
    channel = grpc.secure_channel(
        target,                                        # "dns:///items:50051"
        grpc.composite_channel_credentials(creds, call_creds),
        options=[
            ("grpc.service_config", json.dumps(SERVICE_CONFIG)),
            ("grpc.lb_policy_name", "round_robin"),
            ("grpc.keepalive_time_ms", 30_000),
        ],
    )
    return grpc.intercept_channel(channel, DeadlineInterceptor())
```

> [!TIP]
> Build the channel **lazily** (a module-level `@lru_cache` factory called on first use), not at import time. That single change is what keeps the same code correct under Gunicorn's pre-fork model.

---
## 🚨 Common mistakes

| Mistake | What it costs you |
| --- | --- |
| No deadline on a call | a hung leaf service parks threads all the way up the call chain until every service is out of workers |
| A new channel per request | a TCP + TLS handshake per call, and connection churn that looks like packet loss |
| `INTERNAL` (or `UNKNOWN`) for every failure | callers cannot tell "fix your request" from "retry in a second", so they retry everything |
| `str(exc)` in the status message | table names, paths and SQL handed to whoever called you |
| Retrying non-idempotent RPCs | duplicate charges, duplicate emails, duplicate rows |
| Reusing a proto field number | silent data corruption — the wire has no names to disagree about |
| L4 load balancer in front of gRPC | one backend takes all traffic; new replicas idle |
| Long-lived streams on the sync server | one thread per stream; the pool starves and unary calls time out |
| `add_insecure_port` "just for now" | plaintext tokens, no peer identity, and it always survives to production |
| Reflection left enabled | your full API surface published to anyone who can reach the port |
| Committing `*_pb2.py` and editing by hand | generated code drifts from the `.proto` that is supposed to be the contract |

---
## 🧠 Summary

| Area | Takeaway |
| --- | --- |
| Deadlines | Every call gets one, they propagate, and a client interceptor enforces the default |
| Retries | Service config in the channel, `UNAVAILABLE` only, jitter, and idempotency keys for mutations |
| Protos | Field numbers are the contract: `reserved` on delete, never renumber, version the package |
| Errors | Pick the code the caller can act on; machine-readable detail in the status payload, secrets nowhere |
| Streaming | Real backpressure, but one thread per stream on the sync server — and never a substitute for a queue |
| Runtime | One channel per process, bounded pool, `maximum_concurrent_rpcs`, lazy channels under fork |
| Security | mTLS by default, authn in an interceptor, authz next to the data, limits as controls, reflection off |
| Edge | HTTP/2 needs an L7 proxy or client-side LB; browsers need gRPC-Web |
| Operations | Standard health service, interceptor metrics, and `GRPC_TRACE` when the transport misbehaves |

---
## 📚 References

- [gRPC Python docs](https://grpc.github.io/grpc/python/) · [performance best practices](https://grpc.io/docs/guides/performance/)
- [Protobuf style guide](https://protobuf.dev/programming-guides/style/) · [proto3 language guide](https://protobuf.dev/programming-guides/proto3/)
- [Google API design guide — errors](https://google.aip.dev/193) · [status code reference](https://grpc.io/docs/guides/status-codes/)
