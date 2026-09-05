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
## 🔧 What is a service?

A **service** is a program the machine is supposed to be running *on its own behalf* — not because a human typed a command, but because the host has a job to do: serve HTTP, accept SSH, run a database, ship logs.

That single sentence hides three requirements that make a service different from "a program you started":

- **It must start without a human.** At boot, after a crash, after a deploy.
- **It must be supervised.** Someone has to notice when it dies and decide whether to restart it.
- **It must have a defined relationship to everything else.** The database before the app, the network before the reverse proxy.

On modern Linux, the component that provides all three is **systemd**, and the file that describes one service to it is a `.service` **unit**.

> [!tldr] TL;DR
> A service is a supervised, declaratively-described process. Write a unit file, don't daemonize (run in the foreground and let systemd track you), log to **stdout**, and treat the sandboxing directives (`NoNewPrivileges`, `ProtectSystem`, `SystemCallFilter`) as part of the deployment rather than an optional extra. `systemctl daemon-reload` after every edit.

---
## 👻 What is a daemon?

A **daemon** is a process that runs in the background, detached from any terminal, usually for the lifetime of the machine. The trailing `d` in `sshd`, `systemd`, `journald`, `crond` is the naming convention.

Classically, a program made *itself* a daemon with a well-known ritual:

1. `fork()`, and let the parent exit — the child is orphaned and re-parented to PID 1.
2. `setsid()` — become a session leader with **no controlling terminal**, so a closed SSH session can't kill it.
3. `fork()` a second time, so the process can never re-acquire a terminal.
4. `chdir("/")` so it doesn't pin a filesystem you want to unmount, and reset `umask`.
5. Close/redirect stdin, stdout, stderr, and write a **PID file** so scripts can find it later.

> [!IMPORTANT]
> Under systemd you should do **none** of that. Daemonizing hides the real process from the supervisor: the PID it launched exits immediately, the PID file may be stale or racy, and the actual worker is untracked. Run in the **foreground** as `Type=simple` (or `Type=notify`), write to stdout, and let systemd own backgrounding, tracking and logging. "Daemon" is now a description of the *role*, not a checklist the program implements.

Two consequences worth internalising: a service does not need a terminal, so anything that prompts interactively will hang forever; and the daemon's environment is **not** your login shell's environment — no `.bashrc`, no `PATH` you're used to, no `DISPLAY`.

---
## 🥇 What is the init system?

The init system is **PID 1**: the first user-space process the kernel starts, and the ancestor of everything else. It has two responsibilities the kernel refuses to handle:

- **Bring the system up** to a usable state, in the right order, and take it back down cleanly.
- **Adopt and reap orphans.** When a parent dies, its children are re-parented to PID 1, which must `wait()` on them or the process table fills with zombies.

PID 1 is also special to the kernel: signals without a handler are *ignored*, so PID 1 cannot be casually `SIGKILL`ed, and if it exits the kernel panics.

| | SysV init (historical) | systemd (current default) |
| --- | --- | --- |
| Description of a service | a **shell script** in `/etc/init.d/` implementing `start\|stop\|restart` | a declarative **unit file** with key/value directives |
| Startup order | sequential, by symlink numbering (`S20…`, `S30…`) in runlevels | a **dependency graph**, resolved and started in parallel |
| Finding the process | a PID file the daemon wrote itself | a **cgroup** per service — every child is tracked, nothing escapes |
| Supervision | none; `Restart` is your own `while` loop or a separate tool | built in: `Restart=`, rate limits, watchdogs |
| Logging | whatever the script redirected, plus `syslog` | stdout/stderr captured into the **journal**, indexed by unit |
| On-demand start | `inetd`/`xinetd`, separately configured | **socket, path and timer activation** in the same system |
| Resource limits | `ulimit` in the script, if you remembered | cgroup directives: `MemoryMax=`, `CPUQuota=`, `TasksMax=` |

Alternatives exist and are worth knowing by name: **OpenRC** (Alpine, Gentoo), **runit**/**s6** (tiny supervision suites, popular in containers), **SysV init** on legacy hosts, and **BusyBox init** in embedded images. The concepts below are systemd-specific, but "declare, supervise, order" is universal.

---
## 🧩 Units in systemd

A **unit** is anything systemd knows how to manage. The suffix picks the type:

| Unit | What it manages | Typical use |
| --- | --- | --- |
| `.service` | a process | the daemon itself |
| `.socket` | a listening socket | **socket activation**: systemd binds the port/path and starts the service on first connection — see [Socket](./Socket.md) and [Unix Domain Sockets](./UnixSockets.md) |
| `.timer` | a schedule | the systemd replacement for cron |
| `.target` | a named group / sync point | `multi-user.target`, `network-online.target` — the successor to runlevels |
| `.mount` / `.automount` | a filesystem | generated from `/etc/fstab` automatically |
| `.path` | filesystem changes | start a unit when a directory becomes non-empty |
| `.device` | a kernel device | ordering against udev, e.g. "after this disk appears" |
| `.slice` / `.scope` | a cgroup of units / of foreign processes | resource control for a whole group |

**Where unit files live — and who wins.** The same unit name may exist in several directories; the *first* match wins, and later ones are shadowed entirely:

| Path | Owner | Precedence |
| --- | --- | --- |
| `/etc/systemd/system/` | you, the admin | highest — overrides the package |
| `/run/systemd/system/` | runtime, volatile | middle |
| `/usr/lib/systemd/system/` | the distro package | lowest — never edit; your changes vanish on upgrade |

Instead of copying a vendor unit, add a **drop-in**: a fragment that is merged on top of the original.

```bash
sudo systemctl edit gunicorn.service   # creates .../gunicorn.service.d/override.conf
sudo systemctl edit --full nginx       # copy the whole vendor unit into /etc for editing
systemctl cat gunicorn.service         # show the effective file + every drop-in, in order
systemctl show gunicorn.service        # every resolved property, including defaults
```

> [!WARNING]
> Two traps that eat an afternoon each. **(1)** systemd caches unit files — after any edit you must run `sudo systemctl daemon-reload`, otherwise you are debugging the previous version. **(2)** In a drop-in, list-valued directives like `ExecStart=` and `Environment=` are *appended*, not replaced. To override them you must first clear the list with an empty assignment:
> ```ini
> [Service]
> ExecStart=
> ExecStart=/usr/local/bin/new-command --flag
> ```

**Instantiated units.** A name with `@` is a template: `getty@tty1.service` comes from `getty@.service`, where `%i` expands to the part after the `@`. One unit file, many instances — handy for per-tenant or per-port workers.

```bash
systemctl enable --now worker@queue-a.service worker@queue-b.service
```

---
## 📄 `.service` units in systemd

A unit file is INI-shaped with three sections: `[Unit]` (identity and dependencies), `[Service]` (how to run it) and `[Install]` (what `systemctl enable` should wire up).

```ini
[Unit]
Description=Items API (Gunicorn)
Documentation=https://internal.docs/items
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=notify
User=items
Group=items
WorkingDirectory=/srv/items
EnvironmentFile=/etc/items/env
ExecStart=/srv/items/.venv/bin/gunicorn items.wsgi:application
ExecReload=/bin/kill -s HUP $MAINPID
Restart=on-failure
RestartSec=2s

[Install]
WantedBy=multi-user.target
```

**`Type=` — how systemd decides you are "up".** Getting this wrong is the most common reason a unit "starts" but nothing works, or a dependent unit starts too early.

| `Type=` | systemd considers the unit started when… | Use for |
| --- | --- | --- |
| `simple` (default) | `fork()`/`exec()` returned — **immediately**, before the port is even bound | simple foreground programs where nothing depends on readiness |
| `exec` | the binary has been successfully executed | same, but catches a bad `ExecStart=` path as a start failure |
| `notify` | the process sends `READY=1` via `sd_notify()` | the honest option: Gunicorn, Nginx (`notify` builds), anything that can signal readiness |
| `forking` | the parent exits and the PID file appears | legacy daemons that insist on daemonizing (`PIDFile=` required) |
| `oneshot` | the command has **finished** | migrations, backups, setup steps (pair with `RemainAfterExit=yes`) |
| `dbus` | the given `BusName=` appears on the bus | classic D-Bus services |
| `idle` | like `simple`, but delayed until other jobs are done | cosmetic — keeps boot output readable |

**Dependencies and ordering are two different axes.** This is the systemd concept people most often get wrong:

| Directive | Meaning |
| --- | --- |
| `Wants=` | pull the other unit in; **if it fails, we still start** — the sane default |
| `Requires=` | pull it in; if it fails or is stopped, we are stopped too |
| `BindsTo=` | stronger `Requires=`: we also stop if the other unit dies unexpectedly (e.g. a device disappears) |
| `Requisite=` | must already be started; do not start it ourselves |
| `PartOf=` | restarting/stopping the other unit propagates to us (not the reverse) |
| `Before=` / `After=` | **ordering only** — no requirement, no pull-in |
| `Conflicts=` | starting us stops the other unit |

> [!IMPORTANT]
> `Requires=postgresql.service` does **not** mean "start after PostgreSQL". It means "PostgreSQL must be running too" — both may start in parallel, and your app will happily race the database. Ordering always needs an explicit `After=`. Write both.

Note also that `After=network.target` only means "the networking *stack* is up", not "I have an IP address". If you bind a specific address, you want `network-online.target` (with `Wants=`, and the distro's wait-online service enabled).

**Running the command.** `ExecStartPre=`, `ExecStart=`, `ExecStartPost=`, `ExecReload=`, `ExecStop=` and `ExecStopPost=` run in that order. Two rules bite everyone once:

- **There is no shell.** `ExecStart=` is `execve()`, so `|`, `>`, `*`, `$(…)` and `~` are literal characters, and the binary needs an **absolute path**. If you truly need shell features, be explicit: `ExecStart=/bin/sh -c 'exec /srv/app/bin/run >> /var/log/app.log'`.
- A leading `-` (`ExecStartPre=-/usr/bin/optional-step`) means "ignore a non-zero exit". Without it, a failing `Pre` step aborts the whole start.

**Restart policy.**

| Directive | Effect |
| --- | --- |
| `Restart=no` (default) | it dies, it stays dead |
| `Restart=on-failure` | non-zero exit, signal, timeout or watchdog — the right default for a service |
| `Restart=always` | also restart after a clean exit; correct for anything that should never stop |
| `RestartSec=` | wait before restarting (default 100 ms — far too fast for a DB-dependent app) |
| `StartLimitIntervalSec=` / `StartLimitBurst=` | give up after N restarts in a window (default 5 in 10 s) |

> [!NOTE]
> `Job for app.service failed … start request repeated too quickly` is the start limiter, not a new bug. The service is crash-looping; read the journal for the *first* failure, fix that, then `systemctl reset-failed app.service`.

**Identity, environment and filesystem.**

| Directive | Why it matters |
| --- | --- |
| `User=` / `Group=` | never run a network service as root; see [Access Control](../Security/AccessControl.md) |
| `DynamicUser=yes` | systemd allocates a transient UID for the service's lifetime — no account to manage or reuse |
| `WorkingDirectory=` | relative paths and `.env` lookups depend on it |
| `Environment=KEY=value` | inline config, visible in `systemctl show` — **not** for secrets |
| `EnvironmentFile=/etc/app/env` | one `KEY=value` per line; this is *not* a shell script — no `export`, no expansion, no command substitution |
| `RuntimeDirectory=app` | creates `/run/app` with the right owner and **removes it on stop**; the clean way to place a Unix socket or PID file |
| `StateDirectory=` / `CacheDirectory=` / `LogsDirectory=` | the same, for `/var/lib`, `/var/cache`, `/var/log` — FHS-correct without `mkdir` in a `Pre` step |

> [!WARNING]
> Secrets do not belong in `Environment=` (world-readable via `systemctl show`) or in a file readable by everyone. Use `LoadCredential=`/`SetCredential=` (systemd ≥ 247), which exposes the value in `$CREDENTIALS_DIRECTORY` readable only by the service, or fetch it at start from [Vault](../DevOps/Vault/README.md). And remember the classic: a secret passed as a **command-line argument** is visible to every user in `ps`.

---
## 🧪 Example — Django + Gunicorn behind a socket

Two units. systemd owns the listening socket, so Nginx can connect to `/run/gunicorn/socket` even while Gunicorn is still booting: the kernel queues the connection instead of refusing it. That is what makes a restart invisible to users.

```ini
# /etc/systemd/system/gunicorn.socket
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn/socket
SocketUser=www-data          # nginx runs as www-data and needs to connect
SocketMode=0660

[Install]
WantedBy=sockets.target
```

```ini
# /etc/systemd/system/gunicorn.service
[Unit]
Description=gunicorn daemon for the items app
Requires=gunicorn.socket
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=notify                  # gunicorn calls sd_notify: READY=1 when workers are up
NotifyAccess=main
User=items
Group=www-data
RuntimeDirectory=gunicorn    # creates /run/gunicorn, cleaned up on stop
WorkingDirectory=/srv/items
EnvironmentFile=/etc/items/env
ExecStart=/srv/items/.venv/bin/gunicorn --workers 3 items.wsgi:application
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed               # SIGTERM the master only; it drains its own workers
TimeoutStopSec=30
Restart=on-failure
RestartSec=2s

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now gunicorn.socket   # enable the SOCKET, not the service
sudo systemctl start gunicorn.service         # or let the first request start it
systemctl status gunicorn.service
```

> [!TIP]
> Gunicorn inherits the socket from systemd (`LISTEN_FDS`), so there is no `--bind` in `ExecStart=`. If you prefer Gunicorn to bind the path itself, drop `Requires=gunicorn.socket` and add `--bind unix:/run/gunicorn/socket` — but then the socket only exists while Gunicorn is alive, and every restart is a brief `502`.

---
## ⏰ Timers instead of cron

A `.timer` starts a `.service` on a schedule. It costs one extra file and buys you journal logs, dependencies, resource limits and the same sandboxing as any other unit.

```ini
# cleanup.timer — pairs with cleanup.service (Type=oneshot)
[Timer]
OnCalendar=daily              # or: Mon *-*-* 03:00:00
Persistent=true               # missed while the host was off? run it at next boot
RandomizedDelaySec=15m        # don't stampede with every other host at 03:00
AccuracySec=1m

[Install]
WantedBy=timers.target
```

```bash
systemctl list-timers --all                          # what runs next, and when
systemd-analyze calendar "Mon *-*-* 03:00:00"        # verify a schedule before trusting it
sudo systemctl start cleanup.service                 # test the job itself, no waiting
```

| | cron | systemd timer |
| --- | --- | --- |
| Output | mailed, or lost | in the journal, `journalctl -u cleanup` |
| Missed runs | gone | replayed with `Persistent=true` |
| Overlap | two copies can run at once | the service is a unit; it won't start twice |
| Environment | a minimal, surprising one | exactly what the unit declares |
| Dependencies / limits | none | `After=`, `MemoryMax=`, sandboxing, the lot |

> [!NOTE]
> For application work — emails, reports, retries — a timer is still the *wrong* layer. It gives you no retries, no visibility and no distribution across hosts. Use Celery or another broker-backed queue; see the durability argument in [Futures](../Python/Futures.md).

---
## 🔐 Hardening a unit

systemd can put a service in a namespace-and-seccomp sandbox with a handful of directives. This is the cheapest security win available on a Linux host: no extra tooling, no application changes, and it directly limits what a compromised process can reach.

```ini
[Service]
# --- privileges ---
NoNewPrivileges=yes                    # setuid binaries can no longer escalate
CapabilityBoundingSet=                 # drop every capability…
AmbientCapabilities=CAP_NET_BIND_SERVICE   # …except what you actually need
PrivateUsers=yes

# --- filesystem ---
ProtectSystem=strict                   # / is read-only, except what you allow below
ProtectHome=yes                        # /home, /root, /run/user are empty
ReadWritePaths=/var/lib/items /run/gunicorn
PrivateTmp=yes                         # its own /tmp — no symlink games between services
PrivateDevices=yes

# --- kernel & processes ---
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectKernelLogs=yes
ProtectControlGroups=yes
ProtectProc=invisible                  # cannot see other users' processes in /proc
RestrictNamespaces=yes
RestrictSUIDSGID=yes
LockPersonality=yes
MemoryDenyWriteExecute=yes             # blocks a whole class of exploit payloads

# --- syscalls & network ---
SystemCallArchitectures=native
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6

# --- resources: a crash-loop shouldn't take the host with it ---
MemoryMax=1G
TasksMax=256
CPUQuota=200%
```

```bash
systemd-analyze security gunicorn.service   # per-directive audit + an exposure score
```

> [!WARNING]
> Apply these incrementally and read the journal after each step. `ProtectSystem=strict` breaks anything that writes outside `ReadWritePaths=`, and `MemoryDenyWriteExecute=yes` breaks JIT runtimes. A sandbox you had to disable in a hurry is worse than one you tightened one directive at a time. The same benchmark-driven mindset applies here as in [SCA](../Hardening/SCA.md).

---
## 📜 Logs — journald

Everything a unit writes to stdout/stderr is captured, tagged with the unit name, PID, UID and boot ID, and stored in the journal. Your application should therefore **log to stdout** and let the platform route it — the same discipline containers force on you.

```bash
journalctl -u gunicorn -f                 # follow
journalctl -u gunicorn --since "10 min ago" -p err
journalctl -u gunicorn -b                 # this boot only  (-b -1 = previous boot)
journalctl -u gunicorn -o json-pretty     # structured fields, for shipping
journalctl --disk-usage ; journalctl --vacuum-time=7d
```

The journal is **volatile by default on some distros**: without `/var/log/journal` (i.e. `Storage=persistent` in `journald.conf`), everything is lost at reboot — exactly when you need last night's logs. Create the directory and restart `systemd-journald`.

---
## 🩺 Debugging checklist

| Symptom | First move |
| --- | --- |
| Edited the unit, nothing changed | `sudo systemctl daemon-reload` |
| Works after `start`, gone after reboot | `enable` is what creates the `*.wants/` symlink; `start` only runs it now — use `enable --now` |
| `status: exited (code=exited, status=203/EXEC)` | bad `ExecStart=` path or a non-executable file |
| `status=200/CHDIR`, `226/NAMESPACE`, `209/STDOUT` | `WorkingDirectory=` missing, or a sandbox directive is denying something |
| "start request repeated too quickly" | crash loop; read the first failure, then `reset-failed` |
| Runs by hand, fails as a unit | environment. No shell, no `PATH`, no `.bashrc`, no TTY — print `env` from a `oneshot` unit to compare |
| Starts before the DB is ready | `After=` is missing (`Requires=` alone does not order) |
| Hangs on stop for 90 s | it ignores `SIGTERM`; fix the signal handler or set `KillMode=`/`TimeoutStopSec=` |
| Nothing in the journal | the app writes to its own log file, or is buffering — set `PYTHONUNBUFFERED=1` |

```bash
systemctl cat app.service            # the effective unit, with drop-ins
systemd-analyze verify app.service   # lint before deploying
systemctl list-dependencies app.service
systemd-analyze blame                # what made boot slow
systemctl --user status              # per-user services; needs `loginctl enable-linger`
```

---
## 🐳 systemd vs containers

The two solve the same problem — "keep this process running, in this environment, with these limits" — at different layers, and they use the same kernel primitives: cgroups for limits, namespaces for isolation.

| | systemd unit | Container ([Docker](../DevOps/Docker/README.md)) |
| --- | --- | --- |
| Declares | how to run a process on **this** host | the filesystem *and* how to run it, anywhere |
| Dependencies | `After=`/`Requires=` between units | `depends_on`, or an orchestrator |
| Isolation | opt-in per directive (`Protect*`, `Private*`) | on by default (own mount/PID/net namespace) |
| Restart | `Restart=on-failure` | `restart: unless-stopped`, or a controller |
| Logs | journald | the container runtime's log driver |
| Scheduling across hosts | none | Kubernetes' job |

Two crossovers worth remembering: **PID 1 in a container** is your application, which usually does *not* reap orphans or forward signals — so a shell wrapper leaves zombies and `docker stop` takes ten seconds. Use `--init` (tini) or handle `SIGTERM` yourself. And in the other direction, when a containerised platform is more than you need, a hardened unit plus a socket is a legitimate deployment: fewer moving parts, and the sandbox is one file.

---
## 🧠 Summary

| Concept | Takeaway |
| --- | --- |
| Service | A supervised process the host runs on its own behalf: auto-start, restart, ordered |
| Daemon | The role, not a ritual — do **not** double-fork under systemd; run in the foreground |
| Init system | PID 1: brings the system up, reaps orphans, cannot be `SIGKILL`ed |
| Units | Declarative files; `/etc` beats `/usr/lib`; extend with drop-ins, never edit vendor units |
| `Type=` | `simple` means "we exec'd it", `notify` means "it said it was ready" — dependents care |
| Dependencies | `Requires=` is a requirement, `After=` is an order. You almost always need both |
| Config & secrets | `EnvironmentFile=` is not a shell script; secrets go in `LoadCredential=`, not `Environment=` |
| Logs | Log to stdout; journald indexes per unit; make storage persistent |
| Hardening | `NoNewPrivileges`, `ProtectSystem=strict`, `SystemCallFilter=@system-service`, then `systemd-analyze security` |
| Timers | Better cron (logs, missed runs, no overlap) — still not a task queue |
| Golden rule | `daemon-reload` after every edit, `enable --now` so it survives a reboot |

---
## 📚 References

- [Linux Yourself / lym](https://lym.readthedocs.io/en/latest/index.html) — services, daemons and init from the ground up
- [Filesystem Hierarchy Standard 3.0](https://refspecs.linuxfoundation.org/FHS_3.0/fhs-3.0.html) — where a service's files belong: `/run`, `/var/lib`, `/etc`, `/usr`
- `man systemd.unit`, `man systemd.service`, `man systemd.exec`, `man systemd.resource-control` — the directive reference, and the only source that matches your installed version
