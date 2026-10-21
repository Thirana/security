# systemd & Services — Practice (Note 12)

Rebuilding the team server's services properly with systemd, step by step. Each step shows the command, what the result means, and the decision it leads to.

**Theory for this note:** Note 11. Each step points to the phase that explains it ("Why: Note 11, Phase X").

**Continues from:** Note 10 (processes and signals on the same server). **Continues into:** Note 14 (logs, timers, and resource limits for these services).

---

## Contents

- The setup
- Step 0 — Look at the current services
- Step 1 — Rewrite the API unit properly
- Step 2 — The worker didn't come back after a reboot
- Step 3 — The API logs errors at every boot
- Step 4 — How should the worker relate to the API?
- Step 5 — A crash loop that left the API down for two hours
- Step 6 — Restarts that still kill the worker's jobs
- Step 7 — Changing a package's service (nginx) safely
- Step 8 — Three workers from one file
- Step 9 — A developer's own service, without sudo
- Step 10 — Boot got two minutes slower
- Step 11 — Audit the units
- State of the server after Note 12
- Scenario recap
- Troubleshooting table
- Appendix — Command reference

---

# The setup

Same server as Notes 4–10.

| Unit | Current state |
|---|---|
| `api.service` | Written quickly in Note 2, hardening added in Note 8 as `override.conf` |
| `worker.service` | Written quickly in Note 2, hardening added in Note 8 |
| `nginx.service` | Installed from apt; serves the frontend on 443 |

Changes from Notes 9–10 that matter here: the worker now handles `SIGTERM` gracefully (Note 10, Step 4), and jobs like reindexes run with `systemd-run` (Note 10, Step 2).

---

# Step 0 — Look at the current services

```bash
systemctl status api
```
```
● api.service - API
     Loaded: loaded (/etc/systemd/system/api.service; enabled; preset: enabled)
    Drop-In: /etc/systemd/system/api.service.d
             └─override.conf
     Active: active (running) since Tue 2026-10-20 09:12:03 +0530; 7h ago
   Main PID: 2890 (node)
      Tasks: 11 (limit: 4558)
     Memory: 96.0M (peak: 120.4M)
     CGroup: /system.slice/api.service
             └─2890 /usr/bin/node /opt/api/current/server.js
```

**Meaning:** running, enabled at boot, with one drop-in from Note 8. (Why: Note 11, Phase 5.2.)

See the files systemd actually combines:

```bash
systemctl cat api
```
```
# /etc/systemd/system/api.service
[Unit]
Description=API
After=network.target

[Service]
User=api
EnvironmentFile=/etc/myapps/common.env
EnvironmentFile=/etc/myapps/api.env
ExecStart=/usr/bin/node /opt/api/current/server.js
Restart=on-failure
RestartSec=1

[Install]
WantedBy=multi-user.target

# /etc/systemd/system/api.service.d/override.conf
[Service]
NoNewPrivileges=yes
AmbientCapabilities=CAP_NET_BIND_SERVICE
...
```

**Meaning:** the base file is minimal: no `Type=`, no `WorkingDirectory=`, `RestartSec=1`, ordered only after `network.target`. Each of those causes a problem later in this note. (Why: Note 11, Phase 3.3.)

Check for anything broken:

```bash
systemctl list-units --failed
systemctl is-enabled api worker nginx
```
```
0 loaded units listed.
enabled
disabled
enabled
```

**Meaning:** nothing failed now — but **`worker` is disabled**. It's running only because someone started it by hand. Step 2 deals with that.

---

# Step 1 — Rewrite the API unit properly

### Split "what it is" from "how it's locked down"

- `api.service` — what the service is and how it runs.
- `api.service.d/10-hardening.conf` — the security settings from Note 8, kept separate so they're easy to review.

```bash
sudo mv /etc/systemd/system/api.service.d/override.conf \
        /etc/systemd/system/api.service.d/10-hardening.conf
```

### The new base unit

`/etc/systemd/system/api.service`:

```ini
[Unit]
Description=Team API
Documentation=https://git.example.com/team/api
After=network.target

[Service]
Type=exec
User=api
Group=api
WorkingDirectory=/opt/api/current
EnvironmentFile=/etc/myapps/common.env
EnvironmentFile=/etc/myapps/api.env
ExecStart=/usr/bin/node /opt/api/current/server.js
Restart=on-failure
RestartSec=1

[Install]
WantedBy=multi-user.target
```

What changed and why:

| Line | Why |
|---|---|
| `Type=exec` | "Started" means node was actually executed. A missing binary now shows as a start failure. (Note 11, Phase 6) |
| `Group=api` | Explicit, not left to defaults |
| `WorkingDirectory=/opt/api/current` | Relative paths in the app resolve inside the current release. The symlink is resolved at each start, so after a deploy swap (Note 6, Step 3), a restart picks up the new release |

(`After=` and `RestartSec=1` stay as they are for now. Steps 3 and 5 show why they need to change.)

### Two settings promised in Note 10

Add to `10-hardening.conf`:

```ini
[Service]
# Note 10, Step 9: block running programs from places the API can write
NoExecPaths=/dev/shm /tmp /var/tmp /srv/uploads /var/cache/api /var/lib/api
# Note 10, Step 10: the API can't see other processes in /proc
ProtectProc=invisible
```

- `NoExecPaths=` — even if an attacker writes a file into one of these folders, the service can't run it. The miner in Note 10, Step 9 ran from `/dev/shm`.
- `ProtectProc=invisible` — inside this service, `/proc` shows only the service's own processes.

(`MemoryDenyWriteExecute=` is another common hardening setting, but **Node's JIT compiler needs it off** — turning it on breaks Node.)

### Check, load, apply

```bash
sudo systemd-analyze verify /etc/systemd/system/api.service
sudo systemctl daemon-reload
sudo systemctl restart api
systemctl status api --no-pager | head -5
```

**Meaning:** `verify` reports unknown settings and typos before they cause a failed start. `daemon-reload` makes systemd read the new files. (Why: Note 11, Phase 3.4.)

Test the new protections from inside the service's view:

```bash
sudo systemd-run --pipe --wait -p User=api -p ProtectProc=invisible ps aux | wc -l
```

A handful of lines (only its own processes), instead of 140+.

---

# Step 2 — The worker didn't come back after a reboot

After a planned reboot for kernel updates, uploads stop being processed.

```bash
systemctl status worker
```
```
○ worker.service - Worker
     Loaded: loaded (/etc/systemd/system/worker.service; disabled; preset: enabled)
     Active: inactive (dead)
```

**Meaning:** `disabled` — nothing starts it at boot. Before the reboot it was running only because someone had used `systemctl start` after creating it, never `enable`. (Why: Note 11, Phase 5.3.)

### Fix

```bash
sudo systemctl enable --now worker
```
```
Created symlink /etc/systemd/system/multi-user.target.wants/worker.service → /etc/systemd/system/worker.service.
```

**Meaning:** `enable` created the symlink that makes `multi-user.target` start it at boot; `--now` also starts it right away.

### Related trap: units that can't be enabled

Ruwan writes a small cleanup unit and tries to enable it:

```bash
sudo systemctl enable uploads-cleanup
```
```
The unit files have no installation config (WantedBy=, RequiredBy=, UpheldBy=,
Also=, or Alias= settings in the [Install] section, and DefaultInstance= for
template units). This means they are not meant to be enabled or disabled ...
```

```bash
systemctl is-enabled uploads-cleanup
# static
```

**Meaning:** no `[Install]` section → `static` → can't be enabled. That's correct for a unit that's meant to be **started by something else** (a timer, Note 13–14). It's a bug only if you expected it to start at boot.

**Rule:** after creating any unit, check **both** `is-active` and `is-enabled`.

---

# Step 3 — The API logs errors at every boot

After each reboot, the API works — but the logs show errors at startup.

```bash
journalctl -b -u api --no-pager | head -8
```
```
09:00:04 systemd[1]: Started api.service - Team API.
09:00:05 node[611]: Error: connect ENETUNREACH 10.0.0.5:5432
09:00:05 systemd[1]: api.service: Main process exited, code=exited, status=1/FAILURE
09:00:06 systemd[1]: api.service: Scheduled restart job, restart counter is at 1.
09:00:06 node[690]: Error: connect ENETUNREACH 10.0.0.5:5432
...
09:00:09 node[722]: listening on :80
```

(`-b` means "this boot only".)

**Meaning:** the API started before the network was ready, couldn't reach the database, crashed, and was restarted a few times until the network came up. It "worked" only because of `Restart=`.

### Why — check the ordering

```bash
systemctl show api -p After -p Wants
systemd-analyze critical-chain api.service
```
```
After=network.target system.slice basic.target sysinit.target ...
Wants=

api.service @3.912s
└─network.target @3.901s
  └─systemd-networkd.service @3.402s +480ms
    ...
```

```bash
systemd-analyze critical-chain network-online.target
```
```
network-online.target @7.215s
└─systemd-networkd-wait-online.service @3.903s +3.310s
```

**Meaning:** the API is ordered after `network.target` (3.9 s), which only means "network management has started". The network is actually **up** at `network-online.target` (7.2 s). The API started ~3 seconds too early. (Why: Note 11, Phase 8.4.)

### Fix

In `api.service`:

```ini
[Unit]
Wants=network-online.target
After=network-online.target
```

Both lines: `Wants=` pulls the target into the boot, `After=` waits for it. (Why: Note 11, Phase 8.3.)

```bash
sudo systemctl daemon-reload
```

After the next reboot:

```bash
journalctl -b -u api --no-pager | head -3
```
```
09:00:08 systemd[1]: Started api.service - Team API.
09:00:09 node[640]: listening on :80
```

### Also fix it in the app

Ordering only helps **at boot**. The database can restart at any time later. The API should **retry** its database connection with a delay instead of crashing — then systemd ordering is just a nice-to-have.

---

# Step 4 — How should the worker relate to the API?

The worker calls the API (`API_BASE_URL`, Note 2). A developer added this to `worker.service` "so the API is always there":

```ini
Requires=api.service
After=api.service
```

### What it actually does

During a deploy, CI restarts the API (Note 8, Step 2):

```bash
sudo systemctl restart api
journalctl -u worker --since "1 min ago" --no-pager | grep -E 'Stopping|Started'
```
```
16:05:02 systemd[1]: Stopping worker.service - Worker...
16:05:04 systemd[1]: Started worker.service - Worker.
```

**Meaning:** `Requires=` means "if the API is stopped or restarted, stop/restart the worker too". Every API deploy now also restarts the worker, interrupting its jobs. (Why: Note 11, Phase 8.1.)

### Decide what you really want

| Want | Setting |
|---|---|
| Starting the worker should also start the API | `Wants=api.service` |
| If both start together, API goes first | `After=api.service` |
| API restarts should **not** touch the worker | Not `Requires=` |

The worker already retries API calls, so a brief API restart is fine. Use:

```ini
Wants=api.service
After=api.service
```

### Group them: one target for the team's apps

Ops often want "restart all our apps" as one action. Create a target:

`/etc/systemd/system/myapps.target`:

```ini
[Unit]
Description=Team applications

[Install]
WantedBy=multi-user.target
```

In **both** `api.service` and `worker.service`:

```ini
[Unit]
PartOf=myapps.target

[Install]
WantedBy=myapps.target
```

- `PartOf=myapps.target` — stopping or restarting the target does the same to the service.
- `WantedBy=myapps.target` — enabling the service links it into the target, and the target is linked into `multi-user.target`.

Because `[Install]` changed, re-run enable so the symlinks move:

```bash
sudo systemctl daemon-reload
sudo systemctl reenable api worker
sudo systemctl enable myapps.target
systemctl list-dependencies myapps.target
```
```
myapps.target
● ├─api.service
● └─worker.service
```

Now:

```bash
sudo systemctl restart myapps.target     # restarts both
```

**Update the sudo rules (Note 8):** ops can be given `/usr/bin/systemctl restart myapps.target` instead of listing every service. Keep exact commands — no wildcards. (Why: Note 7, Phase 6.1.)

---

# Step 5 — A crash loop that left the API down for two hours

A deploy ships a release that's missing a required env variable. The API exits with code 1 immediately. Two hours later, a customer reports the site is down.

```bash
systemctl status api
```
```
× api.service - Team API
     Active: failed (Result: exit-code) since Tue 2026-10-20 14:02:11 +0530; 2h 3min ago
   Main PID: 9012 (code=exited, status=1/FAILURE)

14:02:10 node[9012]: Error: SESSION_SECRET is not set
14:02:11 systemd[1]: api.service: Scheduled restart job, restart counter is at 5.
14:02:11 systemd[1]: api.service: Start request repeated too quickly.
14:02:11 systemd[1]: api.service: Failed with result 'exit-code'.
```

**Meaning:**
- `Restart=on-failure` restarted it, but with `RestartSec=1` there were 5 starts within 10 seconds.
- That hit the default **start limit** (5 starts per 10 s), so systemd **gave up** and marked it `failed`.
- Nothing told anyone. (Why: Note 11, Phase 9.2.)

### Restore service now

Roll back to the previous release (Note 6, Step 3), then clear the failed state:

```bash
sudo -u deploy bash -c 'ln -s /opt/api/releases/2026-10-17-2 /opt/api/current.new && mv -T /opt/api/current.new /opt/api/current'
sudo systemctl reset-failed api
sudo systemctl start api
```

### Decide a restart policy

Two different situations need two answers:
- A **bad release** fails forever — retrying won't fix it; a person must act.
- A **temporary problem** (database restarting) fixes itself — keep retrying, but slowly.

Policy: retry with growing waits, and if it still can't stay up after ~10 minutes, stop and **alert**.

In `api.service`:

```ini
[Unit]
StartLimitIntervalSec=10min
StartLimitBurst=10
OnFailure=notify-failure@%n.service

[Service]
Restart=on-failure
RestartSec=2
RestartSteps=5
RestartMaxDelaySec=60
```

- Waits grow from 2 s to 60 s (systemd 254+; check `systemctl --version`). (Why: Note 11, Phase 9.3.)
- 10 starts in 10 minutes → give up → `failed` → `OnFailure=` starts the alert unit. (Why: Note 11, Phase 9.4.)

### The alert unit — a template

`/etc/systemd/system/notify-failure@.service`:

```ini
[Unit]
Description=Send alert: %i failed

[Service]
Type=oneshot
DynamicUser=yes
SupplementaryGroups=systemd-journal
EnvironmentFile=/etc/myapps/alerts.env
ExecStart=/usr/local/bin/notify-failure %i
```

- `%i` is the instance name — here, the failed unit's name (`api.service`), passed in by `OnFailure=notify-failure@%n.service`. (Why: Note 11, Phase 12.)
- `DynamicUser=yes` — runs as a temporary user, not root. `systemd-journal` group lets it read the logs. (Why: Note 7, Phase 10.1.)
- `alerts.env` holds `ALERT_WEBHOOK_URL=...` (`600 root:root` — systemd reads it as root, Note 6, Step 4).

`/usr/local/bin/notify-failure` (`755 root:root`):

```bash
#!/bin/bash
set -euo pipefail
unit="$1"
text="$(hostname): ${unit} FAILED
$(journalctl -u "$unit" -n 15 --no-pager -o short-iso)"
curl -fsS -X POST -H 'Content-Type: application/json' \
     --data "$(jq -n --arg t "$text" '{text: $t}')" \
     "$ALERT_WEBHOOK_URL"
```

(Needs `jq`: `sudo apt install jq`. The webhook can be Slack, Google Chat, or similar.)

### Test the alert without breaking anything

```bash
sudo systemctl daemon-reload
sudo systemctl start notify-failure@api.service
journalctl -u notify-failure@api.service -n 5 --no-pager
```

The team channel gets the message with the last 15 log lines. Add the same `OnFailure=` line to `worker.service`.

**Conclusion:** a broken release now produces an alert within ~10 minutes instead of silent downtime, and temporary problems retry slowly instead of hammering the database.

---

# Step 6 — Restarts that still kill the worker's jobs

Note 10, Step 4 added a graceful `SIGTERM` handler. But some thumbnail jobs take up to 3 minutes, and those still fail during deploys.

```bash
journalctl -u worker --since "15:00" --no-pager | grep -v 'processing'
```
```
15:10:00 worker[7410]: SIGTERM received: finishing current job, then exiting
15:10:00 worker[7410]: convert failed: killed by signal 15
15:11:30 systemd[1]: worker.service: State 'stop-sigterm' timed out. Killing.
15:11:30 systemd[1]: worker.service: Killing process 7410 (python) with signal SIGKILL.
15:11:30 systemd[1]: worker.service: Main process exited, code=killed, status=9/KILL
15:11:30 systemd[1]: worker.service: Failed with result 'timeout'.
```

Two separate problems:

**1. `convert failed: killed by signal 15`** — the child process got `SIGTERM` at the same moment as the worker. The default `KillMode=control-group` signals **every** process in the service at once. The worker wanted to let its current `convert` finish, but the converter was already killed. (Why: Note 11, Phase 10.3.)

**2. `stop-sigterm timed out` after 90 s** — the default `TimeoutStopSec=90` is shorter than the longest job. (Why: Note 11, Phase 10.2.)

### Fix

In `worker.service`:

```ini
[Service]
KillMode=mixed
TimeoutStopSec=5min
```

- `KillMode=mixed` — `SIGTERM` goes **only to the main worker process**. It finishes its current job (including the converter), then exits. If the timeout still hits, `SIGKILL` goes to everything left.
- `TimeoutStopSec=5min` — longer than the longest job, with margin.

```bash
sudo systemctl daemon-reload
sudo systemctl restart worker
journalctl -u worker -n 4 --no-pager
```
```
15:30:01 worker[8120]: SIGTERM received: finishing current job, then exiting
15:32:14 worker[8120]: done img-0213.png
15:32:14 worker[8120]: worker stopped cleanly
15:32:15 systemd[1]: Started worker.service - Worker.
```

**Trade-off:** a longer stop timeout means deploys and shutdowns can take up to 5 minutes. That's the cost of not losing work. Keep jobs short where possible.

---

# Step 7 — Changing a package's service (nginx) safely

nginx (from apt) needs three changes:
1. Restart automatically if it crashes (the package unit doesn't).
2. Allow more open files (lots of connections). (Resource limits in detail: Note 13.)
3. Use the team's config file `/etc/nginx/team.conf` instead of the default.

### Never edit the package file

```bash
systemctl cat nginx | head -2
# # /usr/lib/systemd/system/nginx.service
```

That file is replaced on every nginx upgrade. Use a drop-in. (Why: Note 11, Phase 3.2 and 13.)

### First attempt

```bash
sudo systemctl edit nginx
```

```ini
[Service]
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
ExecStart=/usr/sbin/nginx -c /etc/nginx/team.conf -g 'daemon on; master_process on;'
```

Save (systemctl edit reloads automatically) and restart:

```bash
sudo systemctl restart nginx
```
```
Failed to restart nginx.service: Unit nginx.service has a bad unit file setting.
```
```bash
journalctl -u nginx -n 2 --no-pager
```
```
nginx.service: Service has more than one ExecStart= setting, which is only allowed for Type=oneshot services. Refusing.
```

**Meaning:** `ExecStart=` is a **list**. The drop-in **added** a second command instead of replacing the first. (Why: Note 11, Phase 13.3.)

### Look at everything that runs nginx

```bash
systemctl cat nginx | grep -E '^Exec'
```
```
ExecStartPre=/usr/sbin/nginx -t -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx.pid
```

**Meaning:** three lines start nginx with the default config: the config **test** (`ExecStartPre`), the **start**, and the **reload**. Changing only `ExecStart` would test and reload the wrong config.

### Fixed drop-in

```ini
[Service]
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

ExecStartPre=
ExecStartPre=/usr/sbin/nginx -t -q -c /etc/nginx/team.conf -g 'daemon on; master_process on;'
ExecStart=
ExecStart=/usr/sbin/nginx -c /etc/nginx/team.conf -g 'daemon on; master_process on;'
ExecReload=
ExecReload=/usr/sbin/nginx -c /etc/nginx/team.conf -g 'daemon on; master_process on;' -s reload
```

Each empty `Key=` clears the package's value first.

```bash
sudo systemctl restart nginx
systemctl status nginx --no-pager | head -6
systemd-delta --type=extended
```
```
     Drop-In: /etc/systemd/system/nginx.service.d
             └─override.conf
     Active: active (running) ...

[EXTENDED]   /usr/lib/systemd/system/nginx.service → /etc/systemd/system/nginx.service.d/override.conf
[EXTENDED]   /etc/systemd/system/api.service → /etc/systemd/system/api.service.d/10-hardening.conf
```

**Meaning:** nginx runs with the team config; `systemd-delta` lists every unit you've extended, so nothing is hidden. Future nginx upgrades keep the drop-in.

**Lesson:** when you change **how a program starts**, check **every** `Exec*` line that runs it.

---

# Step 8 — Three workers from one file

The upload queue is growing. The team wants three workers.

### Convert the worker to a template

```bash
sudo systemctl disable --now worker
sudo mv /etc/systemd/system/worker.service /etc/systemd/system/worker@.service
sudo mv /etc/systemd/system/worker.service.d /etc/systemd/system/worker@.service.d
```

Edit `/etc/systemd/system/worker@.service`:

```ini
[Unit]
Description=Worker %i
Wants=api.service
After=api.service
PartOf=myapps.target
OnFailure=notify-failure@%n.service

[Service]
Type=exec
User=worker
Group=worker
WorkingDirectory=/var/lib/worker
EnvironmentFile=/etc/myapps/common.env
EnvironmentFile=/etc/myapps/worker.env
Environment=WORKER_ID=%i
ExecStart=/opt/worker/venv/bin/python -m worker
Restart=on-failure
RestartSec=2
KillMode=mixed
TimeoutStopSec=5min

[Install]
WantedBy=myapps.target
```

`%i` becomes `1`, `2`, `3`. `WORKER_ID` lets each instance name itself in logs. (Why: Note 11, Phase 12.)

The `StateDirectory=worker` from Note 8's hardening is shared by all instances — they all use `/var/lib/worker`, which is fine because they only read config there. If each needed its own folder, use `StateDirectory=worker/%i`.

### Start them

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now worker@1 worker@2 worker@3
systemctl list-units 'worker@*' --no-pager
```
```
UNIT              LOAD   ACTIVE SUB     DESCRIPTION
worker@1.service  loaded active running Worker 1
worker@2.service  loaded active running Worker 2
worker@3.service  loaded active running Worker 3
```

Each is its own service:

```bash
journalctl -u worker@2 -n 3 --no-pager
sudo systemctl restart worker@2          # only that one
sudo systemctl restart myapps.target     # api + all three workers
```

### Update anything that named `worker.service`

- Note 8's sudo rules (`systemctl restart worker`) → replace with `restart myapps.target` or the exact instance names. **Don't** use `worker@*` — a wildcard in sudo is wider than it looks. (Why: Note 7, Phase 6.1.)
- Note 10's `systemd-run` reindex used `worker`'s env files — unchanged.
- Monitoring and scripts that check `systemctl is-active worker` → check `worker@1 worker@2 worker@3`, or the target.

---

# Step 9 — A developer's own service, without sudo

Nimal wants his thumbnail tool's file watcher (`/srv/projects/thumbs-tool/watch.py`) running continuously while he develops it. He has no sudo for this — and shouldn't need it.

### A user service

As nimal, `~/.config/systemd/user/thumbs-watch.service`:

```ini
[Unit]
Description=Thumbnail tool watcher (dev)

[Service]
Type=exec
WorkingDirectory=/srv/projects/thumbs-tool
ExecStart=/usr/bin/python3 /srv/projects/thumbs-tool/watch.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now thumbs-watch
systemctl --user status thumbs-watch
journalctl --user -u thumbs-watch -f
```

**Meaning:** runs as nimal, managed by **his own** systemd manager, no sudo. `default.target` is the user manager's normal target. (Why: Note 11, Phase 14.1.)

### Problem: it stops when he logs out

Next morning it's not running. It stopped at about the time his last SSH session ended.

**Meaning:** a user's systemd manager runs only while they have a session. When the last session ends, it stops their services.

### Decision: lingering

The admin can let it keep running:

```bash
sudo loginctl enable-linger nimal
ls /var/lib/systemd/linger/
# nimal
```

Now his user manager starts at boot and runs without a login.

**Decide on purpose.** Lingering means a user can keep processes running permanently, starting at boot, without anyone logged in. That's useful for developers — and also a classic **persistence** method for attackers (Note 11, "How this goes wrong"). Allow it per person, record why, and include `/var/lib/systemd/linger/` in the audit (Step 11).

---

# Step 10 — Boot got two minutes slower

After Step 3 (waiting for `network-online.target`), a reboot takes more than two minutes before the API answers.

```bash
systemd-analyze
systemd-analyze blame | head -3
```
```
Startup finished in 2.1s (kernel) + 2min 4.3s (userspace) = 2min 6.4s

2min 0.112s systemd-networkd-wait-online.service
     1.420s cloud-init.service
      812ms snapd.service
```

**Meaning:** `systemd-networkd-wait-online` waited **two minutes** (its default timeout). Before Step 3, nothing waited for it, so the problem was hidden. (Why: Note 11, Phase 1.3.)

### What is it waiting for?

```bash
networkctl list
```
```
IDX LINK  TYPE     OPERATIONAL  SETUP
  1 lo    loopback carrier      unmanaged
  2 ens4  ether    routable     configured
  3 ens5  ether    no-carrier   configuring
```

**Meaning:** `ens5` — a second network card added for a test last month — has no connection, so it never becomes "configured". By default, wait-online waits for **every** managed interface.

### Fix

Tell netplan that interface isn't required for "online". In `/etc/netplan/60-ens5.yaml`:

```yaml
network:
  version: 2
  ethernets:
    ens5:
      dhcp4: true
      optional: true
```

```bash
sudo netplan apply
```

(Or remove `ens5` if it's no longer needed.)

Next boot:

```bash
systemd-analyze
```
```
Startup finished in 2.1s (kernel) + 15.2s (userspace) = 17.3s
```

**Conclusion:** adding a correct dependency can expose a slow unit that was always there. `systemd-analyze blame` and `critical-chain` find it quickly.

---

# Step 11 — Audit the units

Unit files decide what root starts, as whom. Add these checks to the Note 8 audit script.

```bash
echo "=== Unit files or drop-ins writable by non-root (expect: none) ==="
sudo find /etc/systemd /usr/lib/systemd /run/systemd -type f \
     \( -name '*.service' -o -name '*.target' -o -name '*.timer' -o -name '*.conf' \) \
     -perm /022 2>/dev/null

echo "=== Programs started by running services, and who can change them ==="
for u in $(systemctl list-units --type=service --state=running --no-legend | awk '{print $1}'); do
  systemctl show "$u" -p ExecStart --value | grep -o 'path=[^ ;]*' | cut -d= -f2
done | sort -u | xargs -r stat -c '%U:%G %a %n'
# expect root-owned, not group/other-writable. Also check scripts passed as arguments.

echo "=== Services running as root ==="
for u in $(systemctl list-units --type=service --state=running --no-legend | awk '{print $1}'); do
  user=$(systemctl show "$u" -p User --value)
  [ -z "$user" ] && echo "root  $u"
done

echo "=== Overridden / extended units (expect: only ones we made) ==="
systemd-delta --type=extended,overridden --no-pager

echo "=== Masked units (expect: none important) ==="
systemctl list-unit-files --state=masked --no-pager --no-legend

echo "=== Users with lingering services ==="
ls /var/lib/systemd/linger/ 2>/dev/null

echo "=== Failed units ==="
systemctl list-units --failed --no-legend
```

### Reading it

| Check | Investigate if… |
|---|---|
| Writable unit files | Any result — someone could choose what root runs |
| ExecStart programs | Not `root`-owned, or mode allows group/other write |
| Services as root | An app service appears (should have `User=`) |
| `systemd-delta` | A drop-in you didn't create — e.g. an extra `ExecStartPre=` on a normal service |
| Masked units | A security service (logging, firewall) blocked |
| Lingering | A user not on the approved list |
| Failed units | Anything — and why its alert didn't fire |

---

# State of the server after Note 12

### Units

| Unit | Purpose |
|---|---|
| `myapps.target` | Groups the team's apps; enabled in `multi-user.target` |
| `api.service` + `10-hardening.conf` | API; waits for network; backoff restarts; alerts on failure |
| `worker@1..3.service` + hardening drop-in | Three workers; graceful stop up to 5 min |
| `notify-failure@.service` | Sends an alert with recent logs for any failed unit |
| `nginx.service` + `override.conf` | Package unit with restart, file limit, team config |
| `thumbs-watch.service` (nimal, user) | Dev tool; lingering approved |

### Final `api.service`

```ini
[Unit]
Description=Team API
Documentation=https://git.example.com/team/api
Wants=network-online.target
After=network-online.target
PartOf=myapps.target
StartLimitIntervalSec=10min
StartLimitBurst=10
OnFailure=notify-failure@%n.service

[Service]
Type=exec
User=api
Group=api
WorkingDirectory=/opt/api/current
EnvironmentFile=/etc/myapps/common.env
EnvironmentFile=/etc/myapps/api.env
ExecStart=/usr/bin/node /opt/api/current/server.js
Restart=on-failure
RestartSec=2
RestartSteps=5
RestartMaxDelaySec=60

[Install]
WantedBy=myapps.target
```

**Still open for Note 14:**
- Logs: where they're kept, how long, and how to search them well.
- The backup still runs from cron — moving it to a systemd timer.
- No CPU or memory limits yet — a runaway worker can starve the API.

---

# Scenario recap

| Step | What happened | Concept (Note 11) |
|---|---|---|
| 0 | Read status and the combined unit | `status`, `cat`, `is-enabled` (3, 5) |
| 1 | Rewrote the API unit; split hardening into a drop-in | `Type=exec`, `WorkingDirectory`, `verify`, `daemon-reload` (3, 4, 6, 7) |
| 2 | Worker missing after reboot; a "static" unit | `start` vs `enable`; `[Install]` (5) |
| 3 | API started before the network | `network-online.target`, `Wants=` + `After=` (8) |
| 4 | API restarts also restarted the worker | `Requires=` vs `Wants=`; `PartOf=` + target (8, 11) |
| 5 | Crash loop → silent `failed` | Start limit, backoff, `OnFailure=` template alert (9, 12) |
| 6 | Long jobs killed on stop | `KillMode=mixed`, `TimeoutStopSec=` (10) |
| 7 | nginx drop-in refused | Clearing list settings; check every `Exec*` (13) |
| 8 | Three workers from one file | Templates, `%i`, instances (12) |
| 9 | Dev service without sudo | User services, lingering (14) |
| 10 | Boot 2 min slower | `systemd-analyze blame`, wait-online (1) |
| 11 | Unit audit | Security overview |

---

# Troubleshooting table

| Symptom | Likely cause | Check | Fix |
|---|---|---|---|
| Edited a unit, nothing changed | No `daemon-reload` / no restart | `systemctl status` warning | `daemon-reload`, then restart |
| Service behaves differently from its file | A drop-in overrides it | `systemctl cat UNIT` | Edit or remove the drop-in |
| Service not running after reboot | Started but never enabled | `systemctl is-enabled` | `enable --now` |
| `enable` says "no installation config" | No `[Install]` section (`static`) | `systemctl is-enabled` | Add `[Install]`, or start it from another unit |
| Errors at boot, works later | Started before network/DB | `journalctl -b -u UNIT`, `critical-chain` | `Wants=` + `After=network-online.target`; app retries |
| Restarting A also restarts B | B has `Requires=A` / `BindsTo=A` / `PartOf=A` | `systemctl show B -p Requires,BindsTo,PartOf` | Use `Wants=` |
| "Start request repeated too quickly" | Hit the start limit | `systemctl status` | Fix cause, `reset-failed`, set `RestartSec`/limits on purpose |
| Unit failed, nobody knew | No alerting | — | `OnFailure=` alert unit |
| "State 'stop-sigterm' timed out" | Shutdown slower than `TimeoutStopSec` | Journal | Raise timeout / shorten jobs |
| Children killed before the main process can finish them | `KillMode=control-group` | Journal shows child killed by signal 15 | `KillMode=mixed` |
| "more than one ExecStart= setting" | Drop-in added instead of replaced | Journal | `ExecStart=` empty line first |
| Override ignored after package upgrade | You edited `/usr/lib/...` | `systemctl cat` path | Use a drop-in in `/etc` |
| Template instance won't enable | No `[Install]` in the template | `systemctl cat name@.service` | Add `WantedBy=` |
| User service stops at logout | No lingering | `ls /var/lib/systemd/linger` | `loginctl enable-linger USER` (decide on purpose) |
| `systemctl --user` fails via `sudo -u` | No user session bus | — | SSH in as the user |
| Slow boot | One unit waiting on a timeout | `systemd-analyze blame` / `critical-chain` | Fix the slow unit (e.g. `optional: true` interface) |

---

# Appendix — Command reference

## Inspect

| Command | Purpose |
|---|---|
| `systemctl status UNIT --no-pager` | State and recent logs |
| `systemctl cat UNIT` | Combined files |
| `systemctl show UNIT -p After,Wants,Requires,PartOf` | Relations |
| `systemctl show UNIT -p NRestarts,Restart,TimeoutStopUSec,KillMode` | Restart/stop settings |
| `systemctl is-active UNIT` / `is-enabled UNIT` | Now / at boot |
| `systemctl list-units --failed` | Broken units |
| `systemctl list-units 'worker@*'` | Template instances |
| `systemctl list-dependencies TARGET` | What a target pulls in |
| `systemd-delta --type=extended,overridden` | All local overrides |
| `journalctl -b -u UNIT` | This boot's logs for a unit |

## Change

| Command | Purpose |
|---|---|
| `systemd-analyze verify FILE` | Check a unit before loading |
| `systemctl daemon-reload` | Load changed files |
| `systemctl enable --now UNIT` | Start now and at boot |
| `systemctl reenable UNIT` | Redo enable after `[Install]` changed |
| `systemctl reset-failed UNIT` | Clear failed state |
| `systemctl edit UNIT` | Drop-in for a package unit |
| `systemctl restart myapps.target` | Restart a group (with `PartOf=`) |

## Boot

| Command | Purpose |
|---|---|
| `systemd-analyze` | Boot time |
| `systemd-analyze blame` | Slowest units |
| `systemd-analyze critical-chain [UNIT]` | What a unit waited on |
| `networkctl list` | Interface states (wait-online) |

## User services

| Command | Purpose |
|---|---|
| `systemctl --user daemon-reload / enable --now / status UNIT` | Manage own services |
| `journalctl --user -u UNIT` | Own service logs |
| `loginctl enable-linger USER` / `disable-linger USER` | Allow / stop running without login |
| `ls /var/lib/systemd/linger/` | Who has lingering |

## Patterns used in this note

| Pattern | Purpose |
|---|---|
| `Wants=` + `After=network-online.target` | Start after the network is up |
| `Wants=` + `After=api.service` | Soft dependency with order |
| `PartOf=myapps.target` + `WantedBy=myapps.target` | Group control |
| `StartLimitIntervalSec=` / `StartLimitBurst=` + `OnFailure=notify-failure@%n.service` | Give up and alert |
| `RestartSec=` / `RestartSteps=` / `RestartMaxDelaySec=` | Backoff |
| `KillMode=mixed` + `TimeoutStopSec=` | Graceful stop of a parent with children |
| `Key=` empty line in drop-in | Clear a list before replacing |
| `name@.service` + `%i` | Template instances |
| `NoExecPaths=` / `ProtectProc=invisible` | Block running files from writable dirs / hide other processes |