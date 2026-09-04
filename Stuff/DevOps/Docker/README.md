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
## 🧩 What is Docker?

**Docker** is the tooling to build, ship and run **containers**. A container is **not a small VM** — it is an ordinary **process on the host kernel with a restricted view of the world**. Nothing is virtualised: same kernel, same scheduler, same page cache.

Think of a **hotel room 🏨** instead of a separate house: shared foundation, plumbing and roof (the kernel), but your own door, your own window, and a metered electricity budget (**namespaces** + **cgroups**).

---
## 💡 Why do we need it?
- 📦 **"Works on my machine" becomes true** — the image ships OS libs, interpreter and your wheels as one **immutable, content-addressed** artifact: the exact same digest runs in dev, CI and prod.
- ⚡ **Millisecond startup** and near-zero overhead vs a VM: no guest kernel, no boot.
- 🧱 **Dependency isolation** — two services needing conflicting `libpq`/OpenSSL versions coexist on one host.
- 🚀 **A unit orchestrators understand** — K8s, Nomad and ECS all schedule containers.
---
## ⚙️ How it works

Three unrelated kernel features, glued together by a low-level runtime (`runc`). **Namespaces** decide *what the process can see*:

| Namespace | Isolates | Inside the container |
|---|---|---|
| `pid` | process tree | your app is PID 1; host processes invisible |
| `net` | interfaces, routes, ports, `iptables` | own `eth0`, own port space |
| `mnt` | mount table | own root filesystem |
| `uts` + `ipc` | hostname, SysV/POSIX shared memory | `hostname` returns the container ID; no shared memory with the host |
| `user` | UID/GID mapping | container root ≠ host root (rootless / userns-remap) |

- **cgroups v2** decides *what it may consume*: CPU quota/shares, memory limit (exceeding it is an **OOM-kill**, not swapping), `pids.max`, block-IO weight.
- **overlay2** decides *what it reads and writes*: read-only image layers stacked, plus one thin **writable layer** on top; modifying an existing file **copies it up** into that layer first.
- **Image vs container vs layer**: a **layer** is an immutable tarball of filesystem changes; an **image** is an ordered list of layers + a config JSON (env, entrypoint, user), addressed by a **manifest digest**; a **tag** is a *mutable* label that moves under you; a **container** is image + writable layer + namespaces + cgroups, and it is disposable — `docker rm` destroys the writable layer, which is exactly *why* state belongs in a volume.

**Layer caching:** every `RUN`/`COPY`/`ADD` becomes a layer, reused from cache only if that instruction **and every layer before it** are unchanged. Order **least volatile → most volatile**:
```dockerfile
COPY . .                             # WRONG: any source edit invalidates the line below,
RUN pip install -r requirements.txt  # so every build re-downloads the world
COPY requirements.txt .              # RIGHT: deps stay cached until requirements.txt changes
RUN pip install -r requirements.txt
COPY . .
```
---
## ✅ Dockerfile essentials done right
- **Pin the base by tag *and* digest** — `FROM python:3.12-slim@sha256:...`. Tags are mutable, digests are not.
- **`.dockerignore` first** — `.git/`, `.venv/`, `node_modules/`, `__pycache__/`, `*.sqlite3` bloat the build context, bust the cache, and can leak `.env` into the image.
- **Multi-stage build** — compile in a fat `builder` stage, copy only artifacts into a slim runtime stage; compilers and `-dev` headers never reach production.
- **Non-root `USER`** before `CMD`, with a fixed UID (`useradd -u 10001`) so volume ownership stays stable across rebuilds.
- **`ENTRYPOINT` vs `CMD`** — `ENTRYPOINT` is the fixed program, `CMD` the default arguments a caller can override. Use **exec form** (`["gunicorn", ...]`): **shell form** wraps you in `/bin/sh -c`, which does not forward `SIGTERM`, so graceful shutdown silently breaks and every deploy kills in-flight requests.
- **`HEALTHCHECK`** so the platform can tell *ready* from merely *running*; **`EXPOSE` publishes nothing** — pure metadata, only `-p` / Compose `ports:` binds a host port.
- **PID 1 is special** — it gets no default signal handlers and must reap orphans, or zombies accumulate. Use `--init` (or `ENTRYPOINT ["tini", "--"]`) whenever your process spawns children.
- ⚠️ **Classics to avoid:** `:latest` instead of a digest, `COPY . .` before installing deps, secrets in `ENV`, app state in the writable layer, and no memory limit — so one leaky container OOM-kills the whole host.
---
## 📦 Storage & networking
| Storage type | Lives where | Use it for | Watch out |
|---|---|---|---|
| **Named volume** | Docker-managed (`/var/lib/docker/volumes`) | DB data, media uploads | invisible in your project tree — back it up explicitly |
| **Bind mount** | any host path | live source reload in dev, config files | host UID/GID mismatch, slow on Docker Desktop, full host FS reach |
| **`tmpfs`** | RAM only | `/tmp`, scratch, runtime secrets | counts against the memory limit, vanishes on stop |

Rule of thumb: **anything you would miss after `docker rm` belongs in a named volume.**

| Network mode | Behaviour | Use |
|---|---|---|
| **`bridge`** (default) | private subnet + NAT, **no** name resolution between containers | legacy standalone containers |
| **user-defined bridge** | same, plus an **embedded DNS server** — containers resolve each other by service name | almost always this; Compose creates one per project |
| **`host`** | no `net` namespace, shares the host stack | latency-sensitive, or you need real client IPs; **no port isolation** |
| **`none`** | loopback only | untrusted batch jobs, pure compute |

> [!WARNING]
> **Publishing a port bypasses UFW.** `-p 5432:5432` installs a `DNAT` rule in Docker's own `iptables` chains, traversed in `nat`/`PREROUTING` — **before** the `INPUT` rules UFW writes. The container is therefore reachable from the whole network while `ufw status` still says `deny (incoming)`. Publish to loopback (`-p 127.0.0.1:5432:5432`) and expose only what the reverse proxy needs: [Nginx](../../SoftwareDesign/WebServer/Nginx.md).

**Compose** describes the whole stack in one YAML on one user-defined network (`docker compose up --build`). It is a dev / single-host tool; for multi-node scheduling go to [Kubernetes](../Kubernetes/README.md), whose node-level runtime is [containerd](../Kubernetes/ClusterInstallation/Containerd/containerd.md).

---
## 🧪 Example
### (a) Multi-stage Dockerfile — Django + gunicorn, non-root
```dockerfile
# pin the digest with: docker buildx imagetools inspect python:3.12-slim
FROM python:3.12-slim@sha256:<digest> AS builder
RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential libpq-dev && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY requirements.txt .
RUN pip wheel --wheel-dir /wheels -r requirements.txt

FROM python:3.12-slim@sha256:<digest>
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
RUN apt-get update && apt-get install -y --no-install-recommends libpq5 curl \
 && rm -rf /var/lib/apt/lists/* && useradd -u 10001 -r -s /usr/sbin/nologin app
WORKDIR /app
COPY --from=builder /wheels /wheels
COPY requirements.txt .
RUN pip install --no-index --find-links=/wheels -r requirements.txt && rm -rf /wheels
COPY --chown=10001:10001 . .
RUN python manage.py collectstatic --noinput
USER 10001
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -fsS http://127.0.0.1:8000/healthz/ || exit 1
ENTRYPOINT ["gunicorn", "config.wsgi:application"]
CMD ["--bind=0.0.0.0:8000", "--workers=3", "--timeout=60", "--access-logfile=-"]
```
### (b) `docker-compose.yml` — web + postgres with a real readiness gate
```yaml
services:
  db:
    image: postgres:16-alpine
    env_file: .env                  # POSTGRES_DB/_USER/_PASSWORD, gitignored
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U shop -d shop"], interval: 5s, retries: 10 }
  web:
    build: .
    env_file: .env
    ports: ["127.0.0.1:8000:8000"]  # loopback only; Nginx terminates TLS in front
    depends_on: { db: { condition: service_healthy } }   # waits for the healthcheck, not "started"
volumes:
  pgdata:
```
### (c) The CLI you actually use
```bash
docker build -t shop/web:1.4.2 .                             # BuildKit by default
docker run --rm -it --entrypoint /bin/bash shop/web:1.4.2     # throwaway shell, nothing persists
docker exec -it -u 0 shop-web-1 bash                          # root shell in a RUNNING container
docker logs -f --tail 100 shop-web-1
docker stats                                 # live CPU/mem/net: first stop for "it feels slow"
docker image history shop/web:1.4.2          # why is this image 1.2 GB? size per instruction
docker inspect shop-web-1 --format '{{json .HostConfig.Mounts}}'
docker system prune -af --volumes            # DANGER: --volumes deletes unused named volumes
```
---
## ⚖️ Docker vs Podman vs containerd vs LXC vs VMs
| Option | Process model | Rootless | Pick it when |
|---|---|---|---|
| **Docker Engine** | central `dockerd`, root by default | opt-in rootless mode | default dev experience, Compose, largest ecosystem |
| **Podman** | daemonless fork/exec, rootless-first | native | no root daemon allowed; systemd integration; pod semantics |
| **containerd + `nerdctl`** | the runtime K8s really uses; Docker-compatible CLI | yes (via rootlesskit) | debugging a K8s node, minimal host — [containerd](../Kubernetes/ClusterInstallation/Containerd/containerd.md) |
| **LXC / LXD** | *system* containers: full init, many services, feels like a VM | yes | you want a long-lived lightweight machine, not one process |
| **VM (KVM / Hyper-V)** | own kernel, hardware-enforced boundary | n/a | untrusted multi-tenant code, a different kernel/OS, strict compliance |
| **buildah / kaniko / BuildKit** | builds images with **no** Docker daemon | yes | building inside CI or K8s, where mounting the socket is unacceptable ([CI/CD](../CI-CD/README.md)) |

---
## 🚨 When NOT to use it
- ❌ **A stateful production database "just in a container"** with no deliberate storage design — volume driver, fsync semantics, tested restore, major-version upgrade path. Containers are fine; a casual bind mount on laptop-grade storage is not.
- ❌ **As process isolation for untrusted code.** A container shares the host kernel: **one kernel privilege-escalation bug and the isolation is gone — a container is not a security boundary.** For real tenant isolation use a sandboxed runtime or microVM: **gVisor**, **Kata Containers**, **Firecracker**.
- ❌ **One container running several services under `supervisord`** — you lose per-process restart, per-process limits, clean stdout logging and independent scaling. One concern per container; group them with Compose or a K8s Pod.
- ❌ **Desktop GUI apps** (X11/Wayland sockets, GPU passthrough, audio = more plumbing than the packaging saves) and **anything that only works with `--privileged` or a host kernel module** — that is the platform telling you to use a VM.
---
## 🔐 Security notes & production hardening
**Never:**
- 🚫 **Mount `/var/run/docker.sock` into a container.** Socket access **is root on the host** — it can launch a privileged container that mounts `/`. That includes "just for CI"; use kaniko/buildah instead.
- 🚫 `--privileged` (disables capability, seccomp, AppArmor and device restrictions in one flag), or `--pid=host` / `--net=host` without a written reason.
- 🚫 Secrets in `ARG` or `ENV`. **Build args and env vars are stored in image metadata** — anyone who can pull the image reads them back with `docker image history` / `docker inspect`. Rotate anything already baked in, then use a BuildKit secret mount:
```dockerfile
# docker build --secret id=pip_token,src=./pip_token.txt -t shop/web:1.4.2 .
RUN --mount=type=secret,id=pip_token \
    pip install --index-url "https://$(cat /run/secrets/pip_token)@pypi.internal/simple" -r requirements.txt
```
**Always** — runtime secrets come from a store, never the image ([Vault](../Vault/README.md)):

| Control | Flag / setting |
|---|---|
| Least privilege | `--cap-drop=ALL --cap-add=NET_BIND_SERVICE`; `USER 10001` in the image **and** `--user 10001:10001` at runtime |
| Block escalation, immutable rootfs | `--security-opt no-new-privileges:true --read-only --tmpfs /tmp:rw,noexec,nosuid,size=64m` |
| Contain resource-exhaustion DoS | `--memory=512m --cpus=1.5 --pids-limit=200` |
| Shrink host-root blast radius | rootless Docker, or `"userns-remap": "default"` in `/etc/docker/daemon.json` |
| Bound log growth | `--log-opt max-size=10m --log-opt max-file=3` ([Monitoring & Logging](../Monitoring&Logging/README.md)) |
| Supply chain | `trivy image shop/web:1.4.2`, `docker buildx build --sbom=true` ([SCA](../../Hardening/SCA.md)); pull over TLS from a registry you control and verify signatures with cosign ([Transport Security](../../Security/TransportSecurity.md)) |

**Base image tradeoffs:** `-slim` (Debian) is the right default for Python. **Distroless** drops the shell and package manager — great attack-surface reduction, painful debugging (use ephemeral debug containers). **Alpine** links **musl**, so `pip` finds no `manylinux` wheels and compiles from source (slow builds, needs a toolchain), and musl's resolver is famously strict about large DNS responses and search-domain handling. For Django, stay on `slim`. Inside Kubernetes the same controls live in `securityContext` — `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities.drop: [ALL]` — and should be **enforced** by an admission policy engine, not merely documented ([OPA](../OPA/README.md)).

> [!NOTE]
> **Windows:** Docker Desktop runs a Linux VM on the **WSL2** backend — keep the repo *inside* the WSL2 filesystem, because bind mounts crossing `/mnt/c` are dramatically slower. And when Git checks out `entrypoint.sh` with **CRLF**, the container dies with `exec /entrypoint.sh: no such file or directory` or `exec format error`, because the trailing `\r` becomes part of the interpreter path. Fix it in `.gitattributes` (`*.sh text eol=lf`) — see [Git](../Git/README.md).
---
## 🐍 Django / Backend tie-in
- Run `collectstatic` at **build** time, not in the entrypoint — the image stays immutable and `--read-only` actually works. And **migrations are not a container-start step** when you run N replicas: they race. Run them as a one-shot job (`docker compose run --rm web python manage.py migrate`, or a K8s `Job`/init container) gated in the pipeline.
- Behind a proxy set `SECURE_PROXY_SSL_HEADER` and gunicorn's `--forwarded-allow-ips`, or Django builds `http://` URLs and redirect-loops. Keep `DEBUG=False` and an explicit `ALLOWED_HOSTS`, both from env.
- Point `HEALTHCHECK` at a cheap `/healthz/` view that does **not** touch the DB, and keep a separate deep readiness endpoint that does. Gunicorn workers ≈ `2 × CPU + 1`, but size it against `--cpus`: the container sees the *host's* CPU count, not its cgroup quota.
---
## 🧠 Summary
| Concept | Takeaway |
|---|---|
| Container | A host process with namespaces (view) + cgroups (limits) + overlayfs (files) |
| Image | Immutable stack of content-addressed layers — pin by digest, never `:latest`; order instructions least → most volatile so deps stay cached |
| Multi-stage | Build tools stay in the builder; the runtime image stays small and non-root |
| State | Named volumes only; the writable layer dies with the container |
| Networking | User-defined bridge for DNS; `-p` bypasses UFW, so bind to loopback |
| Isolation | Not a kernel security boundary — gVisor/Kata/microVM for untrusted code |
| Hardening | non-root, `cap-drop=ALL`, `--read-only`, `no-new-privileges`, limits, no `docker.sock` |
