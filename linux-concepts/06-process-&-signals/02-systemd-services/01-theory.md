# systemd & Services — Theory (Note 11)

What systemd is, how it starts the system, and how to describe, start, stop, order, restart, and override services with unit files.

**Series:**
- Note 1: Shell & Processes
- Note 2: Environment Variables
- Note 3 / 4: Users, Groups & Identity
- Note 5 / 6: Files, Directories & Permissions
- Note 7 / 8: Privilege & Delegation
- Note 9 / 10: Processes & Signals
- **Note 11: systemd & Services — Theory** ← this note
- Note 12: systemd & Services — Practice
- Note 13 / 14: Logs, Timers & Resource Limits — Theory / Practice

Earlier notes used systemd without explaining it: `EnvironmentFile=` (Note 2), `User=` (Note 4), hardening settings (Note 8), `systemd-run` (Note 10). This note explains the whole thing from the start. Security hardening stays in Note 7 / 8; logs, timers, and resource limits come in Note 13.

Each phase ends with **"Used in"** (Note 12 steps) and, where relevant, **"How this goes wrong"** for the security track.

---

## Contents

1. Phase 1 — What an init system is, and how Linux boots
2. Phase 2 — Units: the things systemd manages
3. Phase 3 — Where unit files live, and which one wins
4. Phase 4 — Anatomy of a unit file
5. Phase 5 — `systemctl` and unit states
6. Phase 6 — Service types: when is a service "started"?
7. Phase 7 — Running the program: `ExecStart` and friends
8. Phase 8 — Dependencies and ordering
9. Phase 9 — Restarts and failure handling
10. Phase 10 — Stopping a service
11. Phase 11 — Targets and the boot process
12. Phase 12 — Template units: many instances from one file
13. Phase 13 — Overrides with drop-ins
14. Phase 14 — User services and transient units
15. How this goes wrong — overview
16. Summary table
17. Appendix — Command reference

---

# Phase 1 — What an init system is, and how Linux boots

## 1.1 The problem

When a server starts, dozens of programs must run: networking, SSH, logging, cron, your API, your worker. Someone has to:

- start them in the right **order**,
- run them as the right **user** with the right **environment**,
- **restart** them if they crash,
- **stop** them cleanly at shutdown,
- record their **output**.

That "someone" is the **init system**. On almost all modern Linux distributions it's **systemd**.

## 1.2 How Linux boots

```
1. Firmware (BIOS/UEFI)     checks hardware, finds the boot disk
2. Bootloader (GRUB)        loads the kernel
3. Kernel                   starts, finds disks and devices
4. initramfs                small temporary system that mounts the real root filesystem
5. systemd (PID 1)          first normal process (Note 9, Phase 1.3)
6. systemd starts units     mounts, networking, logging, sshd, your services ...
7. Login prompt / SSH ready
```

From step 5 on, everything is systemd's job. At shutdown it does the same in reverse.

## 1.3 Measuring boot

```bash
systemd-analyze                 # total boot time, split by stage
systemd-analyze blame           # units sorted by how long they took to start
systemd-analyze critical-chain  # the slowest chain of units that boot waited on
```

```
Startup finished in 2.1s (kernel) + 14.8s (userspace) = 16.9s
multi-user.target reached after 14.6s in userspace.
```

**Used in:** Note 12, Step 10.

---

# Phase 2 — Units: the things systemd manages

## 2.1 What a unit is

A **unit** is anything systemd manages, described by a **unit file**. The file name ends with the unit's **type**.

| Type | Manages | Example |
|---|---|---|
| `.service` | A program / process | `api.service`, `ssh.service` |
| `.timer` | A schedule that starts another unit | `backup.timer` (Note 13) |
| `.target` | A **group** of units / a sync point in boot | `multi-user.target` |
| `.socket` | A socket systemd listens on, starting a service on first connection | `ssh.socket` |
| `.mount` / `.automount` | A mounted filesystem | `srv-uploads.mount` |
| `.path` | Watches a file/folder, starts a unit on changes | `uploads-watch.path` |
| `.slice` | A group for **resource limits** (Note 13) | `system.slice` |
| `.scope` | Processes started outside systemd but tracked by it (e.g. your SSH session) | `session-12.scope` |
| `.device` | A device the kernel reported | `dev-sda.device` |

Most of this note is about **services**, plus **targets**. Timers and slices are in Note 13.

## 2.2 Unit names

- `api.service` — a normal unit.
- `worker@2.service` — an **instance** of the **template** `worker@.service` (Phase 12).
- In commands, `.service` can be left out: `systemctl status api` = `systemctl status api.service`.

**Used in:** throughout Note 12.

---

# Phase 3 — Where unit files live, and which one wins

## 3.1 The three main folders

| Folder | Who puts files there | Priority |
|---|---|---|
| `/etc/systemd/system/` | **You** (the admin) | **Highest** |
| `/run/systemd/system/` | Created at runtime (e.g. `systemd-run`) — gone on reboot | Middle |
| `/usr/lib/systemd/system/` (also `/lib/...` on Ubuntu) | **Packages** (`apt install nginx`) | Lowest |

If the same file name exists in more than one place, **the highest-priority one wins completely**.

## 3.2 Rules that follow

- **Never edit files in `/usr/lib/systemd/system/`.** A package upgrade overwrites them.
- Your own services go in `/etc/systemd/system/`.
- To change a package's unit, use a **drop-in** (Phase 13), not a copy.

## 3.3 Seeing what systemd actually uses

```bash
systemctl cat api            # the file(s) in use, with paths, including drop-ins
systemctl show api           # every setting's final value (long)
systemctl show api -p User,Restart,TimeoutStopUSec
systemd-delta                # every unit that's overridden or extended, and where
```

`systemctl cat` is the first thing to run when a service doesn't behave as its file seems to say.

## 3.4 Reloading after a change

systemd reads unit files into memory. After you create or edit one:

```bash
sudo systemctl daemon-reload
```

Then restart the service if it's running. Forgetting `daemon-reload` gives the warning:

```
Warning: The unit file, source configuration file or drop-ins of api.service changed on disk. Run 'systemctl daemon-reload' to reload units.
```

**Used in:** Note 12, Steps 0, 1, 7.

---

# Phase 4 — Anatomy of a unit file

## 4.1 The three sections

```ini
[Unit]
Description=Team API
Documentation=https://git.example.com/team/api
Wants=network-online.target
After=network-online.target

[Service]
Type=exec
User=api
Group=api
WorkingDirectory=/opt/api/current
EnvironmentFile=/etc/myapps/common.env
EnvironmentFile=/etc/myapps/api.env
ExecStart=/usr/bin/node /opt/api/current/server.js
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

| Section | Says | Main settings |
|---|---|---|
| `[Unit]` | **What it is** and **how it relates** to other units | `Description=`, `Wants=`, `Requires=`, `After=`, `Before=`, `OnFailure=` (Phases 8–9) |
| `[Service]` | **How to run** the program | `Type=`, `ExecStart=`, `User=`, `Environment*=`, `Restart=`, stop settings (Phases 6–10) |
| `[Install]` | **What happens on `enable`** — which target pulls it in at boot | `WantedBy=` (Phase 5.3, 11) |

## 4.2 Syntax rules

- One `Key=value` per line. `#` starts a comment.
- Some settings can appear **many times** and add up (e.g. `EnvironmentFile=`, `ExecStartPre=`, `Wants=`).
- An **empty assignment** (`ExecStart=` with nothing after it) **clears** the list so far. Important for overrides (Phase 13).
- Values are mostly taken literally. No shell (Phase 7).

## 4.3 Specifiers

systemd replaces some `%` codes in unit files:

| Specifier | Becomes |
|---|---|
| `%n` | Full unit name (`worker@2.service`) |
| `%N` | Unit name without the type suffix |
| `%i` | Instance name (`2` in `worker@2`) |
| `%p` | Prefix (`worker` in `worker@2`) |
| `%h` | Home folder of the user running the manager |

**Used in:** Note 12, Steps 1, 5, 8.

---

# Phase 5 — `systemctl` and unit states

## 5.1 Everyday commands

| Command | Effect |
|---|---|
| `systemctl start api` | Start **now** |
| `systemctl stop api` | Stop now |
| `systemctl restart api` | Stop, then start |
| `systemctl reload api` | Ask it to re-read config without stopping (only if it supports it) |
| `systemctl enable api` | Start **automatically at boot** |
| `systemctl disable api` | Don't start at boot |
| `systemctl enable --now api` | Both: start now and at boot |
| `systemctl status api` | Current state + recent logs |
| `systemctl mask api` | Block it from being started at all (links it to `/dev/null`) |
| `systemctl unmask api` | Undo mask |

## 5.2 Reading `systemctl status`

```bash
systemctl status api
```
```
● api.service - Team API
     Loaded: loaded (/etc/systemd/system/api.service; enabled; preset: enabled)
    Drop-In: /etc/systemd/system/api.service.d
             └─override.conf
     Active: active (running) since Tue 2026-10-20 09:12:03 +0530; 7h ago
   Main PID: 2890 (node)
      Tasks: 11 (limit: 4558)
     Memory: 96.0M (peak: 120.4M)
        CPU: 1min 12.330s
     CGroup: /system.slice/api.service
             └─2890 /usr/bin/node /opt/api/current/server.js

Oct 20 09:12:03 app-server-01 systemd[1]: Started api.service - Team API.
```

| Line | Meaning |
|---|---|
| `●` colour | Green = active, white = inactive, red = failed |
| `Loaded:` | File found and parsed; its **path**; **enabled**/disabled at boot |
| `Drop-In:` | Override files applied on top (Phase 13) |
| `Active:` | Current state and since when |
| `Main PID:` | The process systemd treats as "the service" |
| `Tasks` / `Memory` / `CPU` | Resource use of the whole service (Note 13) |
| `CGroup:` | **Every** process belonging to the service, including children |
| Last lines | The newest log lines from the journal (Note 13) |

## 5.3 Two separate questions: running now? starting at boot?

| Question | Command | Answers |
|---|---|---|
| Is it running **now**? | `systemctl is-active api` | `active`, `inactive`, `failed`, `activating`, `deactivating` |
| Will it start **at boot**? | `systemctl is-enabled api` | `enabled`, `disabled`, `static`, `masked`, `indirect` |

They're independent. A service can be running but disabled (gone after reboot), or enabled but stopped.

What `enable` really does: it reads `[Install] WantedBy=multi-user.target` and creates a **symlink**:

```
Created symlink /etc/systemd/system/multi-user.target.wants/api.service → /etc/systemd/system/api.service.
```

At boot, `multi-user.target` starts everything in its `.wants/` folder.

| `is-enabled` | Meaning |
|---|---|
| `enabled` | Symlinked into a target; starts at boot |
| `disabled` | Has `[Install]`, but not enabled |
| `static` | **No `[Install]` section** — can't be enabled; only started by other units |
| `masked` | Blocked completely |
| `indirect` | Started by another unit that is enabled |

## 5.4 Active states in more detail

| Active / sub-state | Meaning |
|---|---|
| `active (running)` | Process running |
| `active (exited)` | A `oneshot` that finished successfully (Phase 6) |
| `activating (auto-restart)` | Crashed, waiting `RestartSec` before restarting (Phase 9) |
| `inactive (dead)` | Not running |
| `failed` | Ended with an error, or gave up restarting. Stays failed until started again or `reset-failed` |

## 5.5 Listing

```bash
systemctl list-units --type=service              # loaded services and their states
systemctl list-units --failed                    # what's broken
systemctl list-unit-files --type=service         # all service files and enabled state
systemctl list-dependencies api                  # what api pulls in
sudo systemctl reset-failed api                  # clear a failed state
```

**Used in:** Note 12, Steps 0, 2, 5.

---

# Phase 6 — Service types: when is a service "started"?

## 6.1 Why it matters

Other units wait for a service to be **started** (Phase 8). `Type=` tells systemd **when** to consider it started.

| `Type=` | systemd says "started" when… | Use for |
|---|---|---|
| `simple` | It has created the process (before even running the program) | Old default; avoid for new units |
| `exec` | The program has **actually been executed** | **Most apps** (Node, Python). Reports "file not found" as a start failure |
| `notify` | The program **tells** systemd it's ready (`READY=1` via `sd_notify`) | Apps that need time to load before serving — the most accurate |
| `forking` | The starting process forks a child and exits (old daemon style) | Legacy daemons; needs `PIDFile=` |
| `oneshot` | The program has **finished** | Setup tasks, scripts, migrations, backups |
| `dbus` | It has taken a name on D-Bus | Desktop/system services |

## 6.2 The common cases

**A long-running app** (API, worker): `Type=exec`. If the app can signal readiness (Node package `sd-notify`, Python `sdnotify`), `Type=notify` is better — "started" then means "ready to serve".

**A script that runs and ends** (backup, migration): `Type=oneshot`.
- Add `RemainAfterExit=yes` if, after it finishes, the unit should still count as "active" (e.g. a setup step others depend on).

**Main PID:** for `exec`, `simple`, `notify`, it's the process from `ExecStart`. For `forking`, systemd needs `PIDFile=` to find it.

**Used in:** Note 12, Steps 1, 3.

---

# Phase 7 — Running the program: `ExecStart` and friends

## 7.1 The commands

| Setting | When it runs |
|---|---|
| `ExecStartPre=` | Before the main program (can be repeated) |
| `ExecStart=` | **The main program** (one, except for `oneshot`) |
| `ExecStartPost=` | After the main program is started |
| `ExecReload=` | On `systemctl reload` |
| `ExecStop=` | On stop — **optional**; without it, systemd just sends signals (Phase 10) |
| `ExecStopPost=` | After the service has stopped (even after a crash) |

Prefixes on the path:

| Prefix | Meaning |
|---|---|
| `-` | Ignore failure of this command (`ExecStartPre=-/usr/bin/rm -f /run/x.lock`) |
| `+` | Run this one command with **full privileges** (ignores `User=` and sandboxing) |

## 7.2 No shell

`ExecStart` is **not** run by a shell (Note 8, Step 5). These don't work directly:

```ini
ExecStart=source venv/bin/activate && python -m worker   # WRONG: source, && are shell syntax
ExecStart=node server.js > /var/log/api.log              # WRONG: > is shell syntax
ExecStart=node server.js                                  # WRONG-ish: relative program name
```

Rules:
- Use the **full path** to the program.
- If you truly need shell features: `ExecStart=/bin/sh -c 'cmd1 && cmd2'` — but prefer a proper script file.
- systemd does its own simple variable expansion: `${PORT}` and `$PORT` from `Environment=`/`EnvironmentFile=` work in `ExecStart` arguments.

## 7.3 The run environment

| Setting | Effect | Note |
|---|---|---|
| `User=` / `Group=` | Run as this identity | Notes 3, 4 |
| `SupplementaryGroups=` | Extra groups | Note 7 |
| `WorkingDirectory=` | Current folder | Note 1 |
| `Environment=` / `EnvironmentFile=` | Env vars | Note 2 |
| `UMask=` | Default file mode | Note 5 |
| `StandardOutput=` / `StandardError=` | Where output goes; default **journal** | Note 13 |
| `ExecReload=/bin/kill -HUP $MAINPID` | Common reload pattern | Note 9, SIGHUP |

**Used in:** Note 12, Steps 1, 7.

---

# Phase 8 — Dependencies and ordering

This is the most misunderstood part of systemd. There are **two separate questions**:

1. **Dependency** — "if I start A, should B also be started?"
2. **Ordering** — "if both are starting, which goes first?"

systemd keeps them **independent**.

## 8.1 Dependency settings (in `[Unit]`)

| Setting | Effect |
|---|---|
| `Wants=B` | Starting A also **starts B**. If B fails, A **still starts** (soft) |
| `Requires=B` | Starting A also starts B. If B **fails to start**, A doesn't start. If B is **stopped/restarted**, A is **stopped/restarted too** (hard) |
| `BindsTo=B` | Like `Requires=`, and A also stops if B stops **for any reason** (even crashing) |
| `PartOf=B` | Stopping or restarting **B** also stops/restarts **A** (one direction; doesn't start A) |
| `Requisite=B` | A starts only if B is **already** running; doesn't start B |
| `Conflicts=B` | Starting A stops B, and the reverse |

Default choice: **`Wants=`**. Use `Requires=` only when A truly can't work without B *and* you want A stopped when B is stopped.

## 8.2 Ordering settings (in `[Unit]`)

| Setting | Effect |
|---|---|
| `After=B` | If both are starting, start A **after B is started** (Phase 6 defines "started"). At shutdown, stop A **before** B |
| `Before=B` | The reverse |

## 8.3 They're separate — the classic mistake

```ini
Requires=postgresql.service       # dependency only
```

Without `After=`, both start **at the same time**. A may start before the database is up. You almost always want both:

```ini
Wants=postgresql.service
After=postgresql.service
```

And the other way: `After=B` alone does **not** start B. It only orders them *if* both are being started.

## 8.4 Network: `network.target` vs `network-online.target`

| Target | Means |
|---|---|
| `network.target` | Network **management** has started. Says **nothing** about having an IP or a route. Used for ordering shutdown |
| `network-online.target` | The network is **configured and up** (as decided by a "wait-online" service, e.g. `systemd-networkd-wait-online` on Ubuntu Server) |

A service that connects to something at startup (a database on another host) needs:

```ini
Wants=network-online.target
After=network-online.target
```

Both lines are needed: `Wants=` to pull the target in, `After=` to wait for it.

Even then, **apps should retry** connections. The database can restart at any time after boot; ordering only helps at startup.

## 8.5 Default dependencies

Unless `DefaultDependencies=no`, every service automatically gets:
- `After=` basic system setup (`sysinit.target`, `basic.target`) — so mounts and logging are ready,
- `Conflicts=shutdown.target` and `Before=shutdown.target` — so it's stopped at shutdown.

That's why simple units work without listing these.

```bash
systemctl list-dependencies api             # what api wants/requires
systemctl list-dependencies --after api     # what api is ordered after
systemd-analyze critical-chain api.service  # what api waited on at boot, with times
```

**Used in:** Note 12, Steps 3, 4.

---

# Phase 9 — Restarts and failure handling

## 9.1 `Restart=`

| Value | Restarts when the process… |
|---|---|
| `no` (default) | Never |
| `on-failure` | Exits with a non-zero code, is killed by a signal, times out, or watchdog fails — **the usual choice** |
| `on-abnormal` | Killed by a signal / timeout / watchdog, but **not** a non-zero exit |
| `on-abort` | Killed by an uncaught signal only |
| `on-success` | Exits cleanly (code 0) |
| `always` | Any exit — even a clean one |

A manual `systemctl stop` never triggers a restart.

`RestartSec=5` — wait 5 seconds before restarting (default 100 ms, which can hammer a struggling database).

## 9.2 Start rate limiting — "start request repeated too quickly"

To stop a broken service restarting forever, systemd limits **how many starts** happen in a time window:

| Setting (in `[Unit]`) | Default | Meaning |
|---|---|---|
| `StartLimitIntervalSec=` | 10 s | The window |
| `StartLimitBurst=` | 5 | Max starts in the window |

If the limit is hit, systemd **gives up**: the unit goes to `failed` with:

```
api.service: Start request repeated too quickly.
api.service: Failed with result 'exit-code'.
```

It stays down until a person runs `systemctl start` (or `reset-failed` then start). With `RestartSec=5`, five starts take 20+ seconds, so the default 10-second window is never hit — the service retries forever. Pick values on purpose (Note 12, Step 5).

## 9.3 Backoff (systemd 254 and newer)

Increase the wait after each failed restart:

```ini
RestartSec=2
RestartSteps=5
RestartMaxDelaySec=60
```

Waits grow from 2 s to 60 s over 5 restarts. Check your version with `systemctl --version` (Ubuntu 24.04 has 255).

## 9.4 Reacting to failure: `OnFailure=`

```ini
[Unit]
OnFailure=notify-failure@%n.service
```

When this unit enters `failed`, systemd starts the named unit — usually a small `oneshot` that sends an alert. `%n` passes the failed unit's name (Phase 4.3, Phase 12).

## 9.5 What counts as success

```ini
SuccessExitStatus=143          # treat "killed by SIGTERM" as a clean exit
```

Useful for apps that exit with 143 when stopped (Note 9, Phase 4.3), so a normal stop isn't logged as a failure.

```bash
systemctl show api -p NRestarts     # how many times systemd restarted it
```

**Used in:** Note 12, Step 5.

---

# Phase 10 — Stopping a service

## 10.1 What `systemctl stop` does

This is Note 9's **SIGTERM → wait → SIGKILL** pattern, done by systemd:

```
1. Run ExecStop= (if set)
2. Send KillSignal (default SIGTERM) — to whom depends on KillMode
3. Wait up to TimeoutStopSec (default 90 s)
4. Still running? Send SIGKILL to everything left in the service
```

## 10.2 Settings

| Setting | Default | Meaning |
|---|---|---|
| `KillSignal=` | `SIGTERM` | The polite signal |
| `TimeoutStopSec=` | 90 s | How long to wait for a clean exit |
| `KillMode=` | `control-group` | Who gets the signal (below) |
| `SendSIGKILL=` | yes | Whether to force-kill after the timeout |
| `TimeoutStartSec=` | 90 s | How long starting may take before it's a failure |

## 10.3 `KillMode=`

Remember from `systemctl status`: the service's **cgroup** holds **all** its processes, including children (Phase 5.2).

| `KillMode=` | `SIGTERM` goes to | At timeout, `SIGKILL` goes to |
|---|---|---|
| `control-group` (default) | **Every** process in the service, at the same time | Everything left |
| `mixed` | **Only the main process** | Everything left |
| `process` | Only the main process | Only the main process (children may survive — avoid) |
| `none` | Nobody (only `ExecStop=` runs) | Nobody — avoid |

`mixed` is the right choice when the main process manages children and wants to **finish them itself** (the worker and its `convert` processes, Note 12, Step 6). With `control-group`, children are signalled at the same moment and may die mid-job.

## 10.4 What the log says when the timeout hits

```
worker.service: State 'stop-sigterm' timed out. Killing.
worker.service: Killing process 3120 (python) with signal SIGKILL.
worker.service: Main process exited, code=killed, status=9/KILL
worker.service: Failed with result 'timeout'.
```

That means the graceful shutdown (Note 9, Phase 8) took longer than `TimeoutStopSec`.

**Used in:** Note 12, Step 6.

---

# Phase 11 — Targets and the boot process

## 11.1 What a target is

A **target** is a unit that does nothing itself. It **groups** other units and marks a **point in boot** ("everything needed for X is ready").

| Target | Meaning |
|---|---|
| `multi-user.target` | Full system, no graphical desktop — **the normal server state** |
| `graphical.target` | `multi-user` + desktop |
| `network-online.target` | Network configured (Phase 8.4) |
| `rescue.target` | Single-user repair mode, basic system |
| `emergency.target` | Minimal shell, root filesystem possibly read-only |
| `default.target` | Link to whichever target boot aims for |

```bash
systemctl get-default                   # multi-user.target on a server
sudo systemctl isolate rescue.target    # switch now (careful: stops most services)
```

## 11.2 `WantedBy=multi-user.target`

Read it as: "**when enabled**, add me to the list of units `multi-user.target` wants." Booting into `multi-user.target` then starts the service.

## 11.3 Your own target to group services

```ini
# /etc/systemd/system/myapps.target
[Unit]
Description=Team applications
Wants=api.service worker.service

[Install]
WantedBy=multi-user.target
```

With `PartOf=myapps.target` in each service, one command controls the group:

```bash
sudo systemctl restart myapps.target      # restarts api and worker
sudo systemctl stop myapps.target         # stops both
```

**Used in:** Note 12, Steps 4, 8.

---

# Phase 12 — Template units: many instances from one file

A **template** has `@` in its name and no instance: `worker@.service`. Starting `worker@1.service`, `worker@2.service` creates **instances**, each using the same file with `%i` replaced.

```ini
# /etc/systemd/system/worker@.service
[Unit]
Description=Worker %i
PartOf=myapps.target

[Service]
Type=exec
User=worker
Environment=WORKER_ID=%i
ExecStart=/opt/worker/venv/bin/python -m worker

[Install]
WantedBy=myapps.target
```

```bash
sudo systemctl enable --now worker@1 worker@2 worker@3
systemctl list-units 'worker@*'
```

Each instance is a **separate service**: its own processes, logs (`journalctl -u worker@2`), restarts, and status.

Templates are common for: several copies of a worker, one service per site/customer, and alert units like `notify-failure@%n.service` (Phase 9.4).

**Used in:** Note 12, Steps 5, 8.

---

# Phase 13 — Overrides with drop-ins

## 13.1 What a drop-in is

A **drop-in** is a small file that **adds to or changes** a unit without copying it:

```
/etc/systemd/system/nginx.service.d/override.conf
```

systemd reads the main unit, then every `*.conf` in `<unit>.d/` folders (in name order). Later settings override earlier ones.

## 13.2 Creating them

```bash
sudo systemctl edit nginx            # opens an editor for override.conf; reloads on save
sudo systemctl edit --full nginx     # copies the WHOLE unit to /etc — avoid unless needed
sudo systemctl revert nginx          # remove your drop-ins (and /etc copies of vendor units)
```

`systemctl edit` is the safe default: the package's file stays untouched and keeps getting upgrades; your change lives in `/etc`.

## 13.3 Lists vs single values

- **Single values** (`Restart=`, `User=`, `TimeoutStopSec=`): the drop-in value **replaces** the original.
- **Lists** (`ExecStart=`, `Environment=`, `ExecStartPre=`): the drop-in value is **added**. To replace, first **clear** with an empty assignment:

```ini
[Service]
ExecStart=
ExecStart=/usr/sbin/nginx -g 'daemon off;' -c /etc/nginx/custom.conf
```

Without the empty line you get:

```
nginx.service: Service has more than one ExecStart= setting, which is only allowed for Type=oneshot services. Refusing.
```

## 13.4 Finding overrides

```bash
systemctl cat nginx                 # main file + every drop-in, with paths
systemd-delta --type=extended       # all units with drop-ins
systemd-delta --type=overridden     # /etc copies replacing vendor files
```

**Used in:** Note 12, Steps 7, 11.

---

# Phase 14 — User services and transient units

## 14.1 User services

Besides the **system** manager (PID 1), each logged-in user gets their **own** systemd manager. Users can run services **without sudo**:

| | System services | User services |
|---|---|---|
| Unit files | `/etc/systemd/system/` | `~/.config/systemd/user/` |
| Command | `sudo systemctl ...` | `systemctl --user ...` (no sudo) |
| Runs as | Any `User=` | Always that user |
| Logs | `journalctl -u name` | `journalctl --user -u name` |
| Starts | At boot | When the user logs in; stops when their last session ends |

To keep a user's services running when they're **not logged in** (and start them at boot), an admin enables **lingering**:

```bash
sudo loginctl enable-linger nimal
ls /var/lib/systemd/linger/          # users with lingering on
```

User services need a proper login session (SSH in as the user); `sudo -u nimal systemctl --user` usually fails because there's no user session bus.

## 14.2 Transient units

`systemd-run` (Note 10, Step 2) creates a **temporary** unit in `/run/systemd/transient/` without writing a file. It gets logs, cgroup tracking, and can take any setting with `-p`:

```bash
sudo systemd-run --unit=reindex --uid=worker -p TimeoutStopSec=300 /opt/worker/reindex.sh
```

It disappears after finishing (a failed one stays visible until `reset-failed`).

**Used in:** Note 12, Step 9.

---

# How this goes wrong — overview

Unit files decide **what root starts, as whom, and with what**. That makes them a prime target.

| Misconfiguration | Why it's dangerous | How to find it |
|---|---|---|
| Unit files or drop-ins writable by non-root | Anyone who can edit them chooses what root runs at boot/restart | `find /etc/systemd /usr/lib/systemd -type f -perm /022` |
| `ExecStart` program or script writable by non-root | Replace the program → runs as the service user (or root) | Check owner/mode of every `ExecStart` path and script argument |
| Script in a writable folder (`/tmp`, a home dir) | Same, via the folder (Note 5, Phase 3.5) | `namei -l` on the path |
| Service running as root without need | A bug = root | `systemctl show -p User` (empty = root) |
| sudo rules for `systemctl edit`, `daemon-reload`, `link`, `start *` | `edit` opens an editor as root and creates root-run config; `link` adds a unit from anywhere | `sudo -l` (Note 7, Phase 6) |
| Hidden drop-ins | An attacker adds `ExecStartPre=` to a normal service — survives package upgrades | `systemd-delta --type=extended`, `systemctl cat` |
| Lingering user services | User-level persistence that runs at boot without login | `ls /var/lib/systemd/linger`, `~/.config/systemd/user/` |
| Unexpected timers | Scheduled persistence (Note 13) | `systemctl list-timers --all` |
| Masked security services | Logging or firewall quietly blocked | `systemctl list-unit-files --state=masked` |

---

# Summary table

| Concept | Key point |
|---|---|
| Init system | Starts, orders, restarts, and stops everything; systemd is PID 1 |
| Boot | Firmware → GRUB → kernel → initramfs → systemd → units |
| Unit | Anything systemd manages; type = file suffix |
| Unit locations | `/etc` (yours, wins) > `/run` > `/usr/lib` (packages, never edit) |
| `daemon-reload` | Needed after editing unit files |
| `systemctl cat` / `show` | The files actually used / final values |
| Sections | `[Unit]` relations, `[Service]` how to run, `[Install]` what enable does |
| Empty assignment | `Key=` clears a list |
| `start` vs `enable` | Now vs at boot (symlink into `target.wants`) |
| `static` | No `[Install]`; can't be enabled |
| `Type=` | When "started": `exec` for apps, `notify` for ready-signalling, `oneshot` for scripts |
| `ExecStart` | Full path, no shell; `-` ignore failure, `+` full privileges |
| `Wants=` / `Requires=` | Soft / hard dependency |
| `After=` / `Before=` | Ordering only — separate from dependency |
| `network-online.target` | Use `Wants=` + `After=`; apps should still retry |
| `PartOf=` | Restart/stop follows another unit (groups) |
| `Restart=on-failure` + `RestartSec=` | The usual restart policy |
| Start limit | 5 starts / 10 s default → "repeated too quickly" → failed |
| `OnFailure=` | Start an alert unit when this one fails |
| Stop | `KillSignal` → `TimeoutStopSec` (90 s) → SIGKILL |
| `KillMode=mixed` | SIGTERM to main only; children handled by the app |
| Targets | Groups / boot points; `multi-user.target` = normal server |
| Templates | `name@.service` + `%i` → many instances |
| Drop-ins | `systemctl edit`; clear lists with `Key=` before replacing |
| User services | `systemctl --user`; lingering keeps them running |
| Transient units | `systemd-run`; no file |

---

# Appendix — Command reference

## State and inspection

| Command | Purpose |
|---|---|
| `systemctl status UNIT` | State, PID, cgroup, recent logs |
| `systemctl is-active UNIT` / `is-enabled UNIT` / `is-failed UNIT` | Quick checks |
| `systemctl cat UNIT` | Files in use, with drop-ins |
| `systemctl show UNIT -p Prop1,Prop2` | Final property values |
| `systemctl list-units --type=service` / `--failed` | Loaded / failed units |
| `systemctl list-unit-files --type=service` | All files and enable state |
| `systemctl list-dependencies UNIT` / `--after UNIT` | Dependencies / ordering |
| `systemd-delta --type=extended` | Units with drop-ins |
| `systemctl --version` | systemd version |

## Control

| Command | Purpose |
|---|---|
| `systemctl start / stop / restart / reload UNIT` | Now |
| `systemctl enable / disable UNIT` | At boot |
| `systemctl enable --now UNIT` | Both |
| `systemctl mask / unmask UNIT` | Block / unblock |
| `systemctl daemon-reload` | Reload unit files |
| `systemctl reset-failed UNIT` | Clear failed state |
| `systemctl isolate TARGET` | Switch to a target |
| `systemctl get-default` / `set-default TARGET` | Boot target |

## Editing

| Command | Purpose |
|---|---|
| `systemctl edit UNIT` | Create/edit a drop-in |
| `systemctl edit --full UNIT` | Copy whole unit to `/etc` |
| `systemctl revert UNIT` | Remove local changes |
| `systemd-analyze verify FILE` | Check a unit file for errors |

## Boot

| Command | Purpose |
|---|---|
| `systemd-analyze` | Boot time |
| `systemd-analyze blame` | Slowest units |
| `systemd-analyze critical-chain [UNIT]` | Chain boot waited on |

## User and transient

| Command | Purpose |
|---|---|
| `systemctl --user ...` | Manage your own services |
| `loginctl enable-linger USER` / `disable-linger` | Keep user services running without login |
| `systemd-run --unit=NAME [--uid=U] [-p Key=Val] CMD` | Transient unit |

## Key unit settings

| Setting | Section | Purpose |
|---|---|---|
| `Description=` | Unit | Human name |
| `Wants=` / `Requires=` / `BindsTo=` / `PartOf=` | Unit | Dependencies |
| `After=` / `Before=` | Unit | Ordering |
| `OnFailure=` | Unit | Alert unit |
| `StartLimitIntervalSec=` / `StartLimitBurst=` | Unit | Restart rate limit |
| `Type=` | Service | Start readiness |
| `ExecStart=` / `ExecStartPre=` / `ExecReload=` / `ExecStop=` | Service | Commands |
| `User=` / `Group=` / `WorkingDirectory=` | Service | Run context |
| `Environment=` / `EnvironmentFile=` | Service | Env vars |
| `Restart=` / `RestartSec=` / `RestartSteps=` / `RestartMaxDelaySec=` | Service | Restart policy |
| `KillSignal=` / `KillMode=` / `TimeoutStopSec=` / `TimeoutStartSec=` | Service | Stop/start behaviour |
| `SuccessExitStatus=` | Service | Extra "OK" exit codes |
| `WantedBy=` | Install | What `enable` links into |