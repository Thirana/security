# Logs, Timers & Resource Limits — Theory (Note 13)

What systemd does **around** services: recording their output (journald), running jobs on a schedule (timers), and controlling how much CPU, memory, and processes they can use (cgroups, limits, and the OOM killer).

**Series:**
- Note 1: Shell & Processes
- Note 2: Environment Variables
- Note 3 / 4: Users, Groups & Identity
- Note 5 / 6: Files, Directories & Permissions
- Note 7 / 8: Privilege & Delegation
- Note 9 / 10: Processes & Signals
- Note 11 / 12: systemd & Services
- **Note 13: Logs, Timers & Resource Limits — Theory** ← this note
- Note 14: Logs, Timers & Resource Limits — Practice

Note 11 covered how systemd starts and stops a service. This note covers the three things it manages while the service runs. The whole-machine view of resources ("why is the server slow?") belongs to the later resources area; here the focus is **per service**.

Each phase ends with **"Used in"** (Note 14 steps) and, where relevant, **"How this goes wrong"** for the security track.

---

## Contents

**Part A — Logs**
1. Phase 1 — How output becomes logs
2. Phase 2 — What a journal entry contains
3. Phase 3 — Reading logs with `journalctl`
4. Phase 4 — Journal storage, size, and rate limits
5. Phase 5 — Text log files and logrotate
6. Phase 6 — Getting logs off the server

**Part B — Timers**
7. Phase 7 — Timers: scheduled jobs the systemd way
8. Phase 8 — Writing schedules: `OnCalendar=` and friends

**Part C — Resource limits**
9. Phase 9 — cgroups: how systemd groups processes
10. Phase 10 — Limiting CPU, memory, processes, and disk I/O
11. Phase 11 — Per-process limits: `ulimit` and `Limit*=`
12. Phase 12 — The OOM killer
13. Phase 13 — Pressure: seeing that limits are being hit

14. How this goes wrong — overview
15. Summary table
16. Appendix — Command reference

---

# Part A — Logs

# Phase 1 — How output becomes logs

## 1.1 Where log lines come from

A running Linux system produces log messages from several sources. On a systemd system, they all go to one place: the **journal**, managed by `systemd-journald`.

```
service stdout/stderr ──┐
syslog() calls ─────────┤
kernel messages ────────┼──► systemd-journald ──► journal files (/var/log/journal)
systemd's own messages ─┤                     └─► (copy) rsyslog ──► /var/log/syslog, auth.log ...
audit events ───────────┘
```

| Source | Example |
|---|---|
| A service's **stdout/stderr** | `console.log("listening on :80")` in the API |
| `syslog()` calls | sshd, sudo, cron |
| The **kernel** | Disk errors, OOM kills (Phase 12), firewall drops |
| **systemd itself** | "Started api.service", "Main process exited" |

## 1.2 Services: just write to stdout

For a systemd service, the default is `StandardOutput=journal` (Note 11, Phase 7.3). So the simplest correct way for an app to log is:

**write lines to stdout/stderr; don't manage log files.**

journald then adds the timestamp, unit name, PID, and user automatically, handles size limits, and makes everything searchable.

## 1.3 Classic text files still exist

On Ubuntu, `rsyslog` also receives a copy of messages and writes the familiar text files: `/var/log/syslog`, `/var/log/auth.log` (Note 3, Phase 8.4), `/var/log/kern.log`. Some apps also write their own files (Note 6, Step 6). Both systems run side by side.

**Used in:** Note 14, Steps 1, 10.

---

# Phase 2 — What a journal entry contains

## 2.1 Fields, not just text

Each journal entry is a set of **fields**. The message is only one of them.

```bash
journalctl -u api -n 1 -o verbose
```
```
Wed 2026-10-21 02:04:11.381 +0530 [s=...;i=...;b=...]
    _SYSTEMD_UNIT=api.service
    _PID=2890
    _UID=901
    _COMM=node
    _EXE=/usr/bin/node
    _BOOT_ID=7c1e...
    _HOSTNAME=app-server-01
    PRIORITY=3
    SYSLOG_IDENTIFIER=node
    MESSAGE=Error: connect ETIMEDOUT 10.0.0.5:5432
```

| Field | Meaning |
|---|---|
| `MESSAGE` | The text |
| `_SYSTEMD_UNIT` | Which unit wrote it |
| `_PID`, `_UID`, `_COMM`, `_EXE` | Which process, user, and program |
| `_BOOT_ID` | Which boot |
| `PRIORITY` | How serious (below) |
| `SYSLOG_IDENTIFIER` | Program tag |

Fields starting with `_` are added **by journald** and can't be faked by the app. That makes them trustworthy for investigations.

## 2.2 Priority levels

| Number | Name | Use |
|---|---|---|
| 0 | `emerg` | System unusable |
| 1 | `alert` | Act immediately |
| 2 | `crit` | Critical |
| 3 | `err` | Errors |
| 4 | `warning` | Warnings |
| 5 | `notice` | Normal but significant |
| 6 | `info` | Normal information (default for stdout) |
| 7 | `debug` | Debug detail |

By default, stdout lines are `info` and stderr lines are also `info` unless the app marks them. An app can set the level of a line by starting it with `<N>`:

```
<3>database connection failed       ← stored as err
<4>slow query: 2.3s                 ← stored as warning
```

(Python's `systemd.journal` handler and Node libraries like `pino` with a systemd transport can do this for you.)

**Used in:** Note 14, Step 1.

---

# Phase 3 — Reading logs with `journalctl`

## 3.1 Choosing what to see

| Option | Shows |
|---|---|
| `-u api` | One unit (repeat for several; globs work: `-u 'worker@*'`) |
| `-b` / `-b -1` | This boot / previous boot |
| `--list-boots` | All boots stored |
| `--since "2026-10-21 02:00" --until "02:15"` | A time window |
| `--since "1 hour ago"` / `--since today` | Relative times |
| `-p err` | Priority `err` and **more serious** |
| `-p warning..err` | A range |
| `-k` | Kernel messages only (like `dmesg`) |
| `-g 'timeout'` | Message matches a pattern (case-insensitive if all lowercase) |
| `_UID=901` / `_PID=2890` / `_COMM=sshd` | Any field match |

Combining:
- **Different fields** are combined with AND: `-u api -p err --since today`.
- The **same field** repeated is OR: `-u api -u nginx`.

## 3.2 Choosing how to see it

| Option | Effect |
|---|---|
| `-f` | Follow new lines (like `tail -f`) |
| `-n 50` | Last 50 lines |
| `-r` | Newest first |
| `-o short-iso` | ISO timestamps (good for sharing) |
| `-o json` / `-o json-pretty` | All fields, for scripts |
| `-o cat` | Message text only |
| `--no-pager` | Plain output, for pipes and scripts |
| `-x` | Add explanation text for systemd messages |

## 3.3 Who can read what

- Every user can read **their own** processes' logs.
- Members of `adm` or `systemd-journal` (Note 3, Phase 4.5) and root can read **everything**.

**Used in:** Note 14, Step 1 and throughout.

---

# Phase 4 — Journal storage, size, and rate limits

## 4.1 Volatile or persistent

| Mode | Where | After reboot |
|---|---|---|
| **Volatile** | `/run/log/journal` (memory) | **Gone** |
| **Persistent** | `/var/log/journal` (disk) | Kept |

The default, `Storage=auto`, means: **persistent if `/var/log/journal` exists**, otherwise volatile. Ubuntu normally creates that folder, but some minimal images don't. Check:

```bash
journalctl --list-boots | head      # only one boot listed = probably volatile
ls -ld /var/log/journal
```

## 4.2 Configuration

journald reads `/etc/systemd/journald.conf` and drop-ins in `/etc/systemd/journald.conf.d/*.conf`. Use a drop-in, then `sudo systemctl restart systemd-journald`.

```bash
systemd-analyze cat-config systemd/journald.conf     # the settings in effect
```

## 4.3 Size limits

| Setting | Default | Meaning |
|---|---|---|
| `SystemMaxUse=` | 10% of the filesystem, max 4 GB | Total disk the persistent journal may use |
| `SystemKeepFree=` | 15% of the filesystem, max 4 GB | Always leave this much free |
| `SystemMaxFileSize=` | 1/8 of `SystemMaxUse` | Size of each journal file before rotating |
| `MaxRetentionSec=` | none | Delete entries older than this |
| `RuntimeMaxUse=` | 10% of `/run` | Same, for volatile storage |

Old entries are deleted automatically when limits are reached. Manual cleanup:

```bash
journalctl --disk-usage
sudo journalctl --vacuum-size=500M      # shrink to 500 MB
sudo journalctl --vacuum-time=14d       # delete entries older than 14 days
```

## 4.4 Rate limits

To stop one noisy service flooding the journal, journald limits each service:

| Setting | Default |
|---|---|
| `RateLimitIntervalSec=` | 30 s |
| `RateLimitBurst=` | 10000 messages per interval |

Over the limit, messages are **dropped**, and journald writes:

```
systemd-journald[301]: Suppressed 48211 messages from worker@2.service
```

Per-service overrides go in the **unit**:

| Unit setting | Effect |
|---|---|
| `LogRateLimitIntervalSec=` / `LogRateLimitBurst=` | Per-service rate limit |
| `LogLevelMax=info` | Drop messages less important than this (e.g. all `debug`) before they're stored |

> **How this goes wrong:** a flood of junk messages can make journald drop the lines that matter (an attacker can do this on purpose), and a small journal can rotate away the evidence of an incident before anyone looks. Size limits and log shipping (Phase 6) are the defence.

**Used in:** Note 14, Steps 2, 3.

---

# Phase 5 — Text log files and logrotate

When an app writes its own log **files** (Note 6, Step 6), something must stop them growing forever. That's **logrotate**, run daily by `logrotate.timer` on Ubuntu.

## 5.1 Two ways to rotate

| Method | What happens | Catch |
|---|---|---|
| `create` (default) | Rename `app.log` → `app.log.1`, create a new empty `app.log` | The app still has the **old file open** and keeps writing to `app.log.1` — unless it's told to reopen (`postrotate` sends a signal, e.g. `SIGUSR1`) |
| `copytruncate` | Copy `app.log` to `app.log.1`, then empty `app.log` in place | Works with apps that can't reopen, but **lines written during the copy are lost** |

## 5.2 The deleted-but-open file

If an app keeps writing to a rotated file, and logrotate later **compresses and deletes** it, the app still holds it open. The file has no name, but **its disk space isn't freed** while it's open. `du` can't see it; `df` can.

```bash
sudo lsof +L1        # open files that have been deleted (link count < 1)
```

The space is released only when the app closes the file (reopen signal or restart).

## 5.3 Useful commands

```bash
sudo logrotate -d /etc/logrotate.d/myapps     # dry run: show what would happen
sudo logrotate -f /etc/logrotate.d/myapps     # force a rotation now
```

**Simplest option:** log to stdout and let journald handle storage and rotation (Phase 1.2).

**Used in:** Note 14, Step 10.

---

# Phase 6 — Getting logs off the server

Logs kept only on the server have two problems:

1. If the server **dies** or its disk is lost, the logs go with it.
2. An **attacker with root** can delete or edit them (Note 10, Step 9). Logs already shipped elsewhere can't be changed from the server.

Common ways to ship them:

| Option | Notes |
|---|---|
| **Cloud agent** — e.g. Google Cloud **Ops Agent** → Cloud Logging | Reads the journal and files; searchable across all servers; retention and alerting in the cloud |
| rsyslog forwarding | Classic; sends syslog to a central server |
| `systemd-journal-upload` / `-remote` | Journal-native forwarding |
| Other agents (Fluent Bit, Vector, Datadog, etc.) | Many destinations |

Ship at least: system and auth logs, the journal for your services, and the kernel log.

**Used in:** Note 14, Step 11.

---

# Part B — Timers

# Phase 7 — Timers: scheduled jobs the systemd way

## 7.1 cron's weak points

Note 2 (Step 8) and Note 10 (Step 3) ran the backup from **cron**. cron works, but:

| Problem | Effect |
|---|---|
| Tiny environment | Scripts fail differently than by hand (Note 2) |
| Output is emailed — often to nowhere | On servers without mail: `(CRON) info (No MTA installed, discarding output)`. **Errors vanish** |
| No catch-up | If the server is off at 02:00, the job doesn't run that day |
| Overlaps | If a run takes longer than the interval, a second copy starts |
| No limits or sandbox | The job can use all CPU/memory, see the whole filesystem |
| No dependencies | Can't say "after the network is up" or "not while X runs" |

## 7.2 How a timer works

A timer is **two units** with the same name:

```
backup.timer   — WHEN to run
backup.service — WHAT to run (Type=oneshot)
```

When the timer fires, systemd **starts the service**. So the job gets everything a service has:

- logs in the journal, with exit status (Phase 3)
- environment files, `User=`, sandboxing (Notes 2, 7, 11)
- `Nice=`, `IOSchedulingClass=` (Note 9's `nice`/`ionice`), and hard resource limits (Phase 10)
- `OnFailure=` alerts (Note 11)
- **no overlaps**: if the service is still running when the timer fires again, it isn't started a second time

## 7.3 The timer unit

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Nightly database backup

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
```

| Setting | Meaning |
|---|---|
| `OnCalendar=` | Calendar schedule (Phase 8) |
| `Persistent=true` | If a run was **missed** (server off), run it as soon as possible after boot |
| `RandomizedDelaySec=` | Add a random delay up to this — spreads load when many servers share a schedule |
| `AccuracySec=` | How precise the start time must be (default 1 min) |
| `Unit=` | Start a differently named service (default: same name) |
| `WantedBy=timers.target` | **Enable the timer**, not the service |

The matching service has `Type=oneshot` and usually **no `[Install]`** section (it's `static` — started only by the timer, Note 12, Step 2).

## 7.4 Managing timers

```bash
sudo systemctl enable --now backup.timer     # schedule it
systemctl list-timers --all                  # NEXT / LEFT / LAST / PASSED for every timer
sudo systemctl start backup.service          # run the job now, by hand
journalctl -u backup.service                 # every run's output
```

One-off timers without files:

```bash
sudo systemd-run --on-active=30min /usr/local/bin/cleanup.sh           # in 30 minutes
sudo systemd-run --on-calendar='2026-10-25 03:00' /usr/local/bin/x.sh  # at a time
```

**Used in:** Note 14, Steps 4, 5.

---

# Phase 8 — Writing schedules: `OnCalendar=` and friends

## 8.1 Calendar format

```
DayOfWeek Year-Month-Day Hour:Minute:Second [TimeZone]
```

`*` = any. Lists with `,`, ranges with `..`, steps with `/`.

| Expression | Means |
|---|---|
| `daily` / `hourly` / `weekly` / `monthly` | Shortcuts (midnight / top of hour / Monday 00:00 / 1st 00:00) |
| `*-*-* 02:00:00` | Every day at 02:00 |
| `Mon..Fri 09:00` | Weekdays at 09:00 |
| `*:0/15` | Every 15 minutes |
| `*-*-01 03:30` | 1st of each month at 03:30 |
| `Sat,Sun *-*-* 04:00` | Weekends at 04:00 |
| `*-*-* 02:00:00 Asia/Colombo` | 02:00 Colombo time, whatever the server's time zone |

Without a time zone, the **server's local time zone** is used. Cloud VMs are often in **UTC** — check with `timedatectl`.

Test any expression:

```bash
systemd-analyze calendar --iterations=3 '*-*-* 02:00:00 Asia/Colombo'
```
```
  Original form: *-*-* 02:00:00 Asia/Colombo
Normalized form: *-*-* 02:00:00 Asia/Colombo
    Next elapse: Thu 2026-10-22 02:00:00 +0530
       (in UTC): Wed 2026-10-21 20:30:00 UTC
       From now: 17h left
       Iteration #2: Fri 2026-10-23 02:00:00 +0530
       ...
```

## 8.2 Relative (monotonic) timers

| Setting | Runs |
|---|---|
| `OnBootSec=5min` | 5 minutes after boot |
| `OnUnitActiveSec=15min` | 15 minutes after the service last **started** |
| `OnUnitInactiveSec=15min` | 15 minutes after the service last **finished** — a gap between runs, never overlapping |

`OnBootSec=` + `OnUnitInactiveSec=` together = "start soon after boot, then repeat with a gap" — good for cleanup jobs.

## 8.3 cron vs timer, side by side

| | cron | systemd timer |
|---|---|---|
| Schedule | `0 2 * * *` | `OnCalendar=*-*-* 02:00:00` |
| Output | Email (often lost) | Journal |
| Missed run | Skipped | `Persistent=true` catches up |
| Overlap | Possible | Prevented |
| Env / user / sandbox / limits | Minimal | Full service settings |
| See next run | Hard | `systemctl list-timers` |
| Failure alert | No | `OnFailure=` |

**Used in:** Note 14, Steps 4, 5.

---

# Part C — Resource limits

# Phase 9 — cgroups: how systemd groups processes

## 9.1 What a cgroup is

A **control group (cgroup)** is a kernel feature that:

1. **groups** processes together,
2. **measures** what the group uses (CPU, memory, I/O, number of processes),
3. **limits** it.

Every process belongs to exactly one cgroup, and **its children start in the same cgroup**. That's how systemd knows every process of a service, even ones it didn't start directly (Note 11, Phase 5.2's `CGroup:` line, and `KillMode=`).

Modern Ubuntu uses **cgroup v2**: one tree under `/sys/fs/cgroup/`.

## 9.2 The tree systemd builds

```
/ (root)
├── system.slice          ← system services
│   ├── api.service
│   ├── worker@1.service
│   ├── nginx.service
│   └── ssh.service
├── user.slice            ← logged-in users
│   └── user-1101.slice
│       └── session-12.scope   (nimal's SSH session)
└── machine.slice         ← VMs / some containers
```

| Unit type | Role in the tree |
|---|---|
| **slice** | A folder in the tree that groups other units. Limits on a slice apply to **everything inside it combined** |
| **service** | A service's processes |
| **scope** | Processes started outside systemd (an SSH session) |

You can create your own slice and put services in it with `Slice=` (Phase 10.6).

## 9.3 Looking at it

```bash
systemd-cgls                         # the tree with processes
systemd-cgls -u api.service          # one unit
systemd-cgtop                        # live CPU/memory/tasks per cgroup — like top, per service
cat /proc/2890/cgroup                # 0::/system.slice/api.service
ls /sys/fs/cgroup/system.slice/api.service/
```

The kernel's files for one service:

| File | Shows |
|---|---|
| `memory.current` / `memory.max` | Memory used / hard limit |
| `memory.events` | Counts of `high`, `max`, `oom`, **`oom_kill`** events |
| `cpu.stat` | CPU time used, and `nr_throttled` / `throttled_usec` (quota hits) |
| `pids.current` / `pids.max` | Tasks now / limit |

**Used in:** Note 14, Steps 6, 7, 8.

---

# Phase 10 — Limiting CPU, memory, processes, and disk I/O

These are **unit settings** (in `[Service]`, or in a `.slice`). systemd writes them into the cgroup files for you.

## 10.1 CPU

| Setting | Meaning |
|---|---|
| `CPUWeight=` | **Share** when CPUs are busy. Default 100, range 1–10000. A service with 200 gets twice the CPU of one with 100 — **only under contention** |
| `CPUQuota=` | **Hard cap**, as a percentage of **one CPU**. `50%` = half a core, `200%` = two cores. Applies even when the CPU is idle |

Compare with Note 9's `nice`: nice is a per-process hint; `CPUWeight=` is the same idea for a whole service; `CPUQuota=` is a guarantee that can't be exceeded.

## 10.2 Memory

| Setting | Meaning |
|---|---|
| `MemoryHigh=` | **Soft limit.** Above it, the kernel reclaims the service's memory aggressively and **slows it down**. Nothing is killed |
| `MemoryMax=` | **Hard limit.** If the service can't stay under it, the kernel's OOM killer **kills a process in that service** (Phase 12) |
| `MemorySwapMax=` | Limit swap use |
| `MemoryLow=` / `MemoryMin=` | **Protect** a service's memory from being reclaimed for others |

Typical pattern: `MemoryHigh=` a bit below `MemoryMax=`, so the service slows down (a warning sign you can see) before it's killed.

## 10.3 Processes / threads

| Setting | Meaning |
|---|---|
| `TasksMax=` | Max **processes + threads** in the service. Default ~15% of the system limit (the "limit: 4558" in `systemctl status`) |

When reached, new processes/threads fail with `Resource temporarily unavailable` (EAGAIN). This stops fork bombs and runaway child creation (Note 10, Step 5's zombie growth).

## 10.4 Disk I/O

| Setting | Meaning |
|---|---|
| `IOWeight=` | Share of disk when busy (like `CPUWeight=`) |
| `IOReadBandwidthMax=/dev/sda 20M` | Hard cap on read speed for a device |
| `IOWriteBandwidthMax=/dev/sda 20M` | Hard cap on write speed |

How well I/O limits work depends on the disk and filesystem; test on your setup.

## 10.5 Priority settings (Note 9 equivalents)

| Unit setting | Same as |
|---|---|
| `Nice=10` | `nice -n 10` |
| `IOSchedulingClass=idle` | `ionice -c 3` |
| `CPUSchedulingPolicy=idle` | Run only when nothing else wants CPU |

## 10.6 Slices: limits for a group

```ini
# /etc/systemd/system/workers.slice
[Unit]
Description=All worker instances

[Slice]
CPUQuota=100%
MemoryMax=2G
```

```ini
# in worker@.service
[Service]
Slice=workers.slice
```

Now **all workers together** can use at most one core and 2 GB, however many instances run.

## 10.7 Changing limits

```bash
sudo systemctl set-property api MemoryMax=800M            # live, and saved (in /etc/systemd/system.control/)
sudo systemctl set-property --runtime api CPUQuota=50%    # live, until reboot
systemctl show api -p MemoryMax,MemoryHigh,CPUQuotaPerSecUSec,TasksMax
systemctl show api -p MemoryCurrent,MemoryPeak,TasksCurrent
```

For a one-off command:

```bash
sudo systemd-run --wait -p MemoryMax=500M -p CPUQuota=50% /usr/local/bin/heavy-report.sh
```

**Used in:** Note 14, Steps 6, 7, 8.

---

# Phase 11 — Per-process limits: `ulimit` and `Limit*=`

## 11.1 rlimits

Older than cgroups, **rlimits** are limits on **each process**, inherited by children (like umask and environment).

| Limit | `ulimit` flag | Unit setting | Typical problem when too low |
|---|---|---|---|
| Open files | `-n` | `LimitNOFILE=` | `EMFILE: Too many open files` (each socket is a file) |
| Processes per user | `-u` | `LimitNPROC=` | Can't fork |
| Core dump size | `-c` | `LimitCORE=` | No crash dumps (often wanted off) |
| Stack size | `-s` | `LimitSTACK=` | Deep recursion crashes |

Each has a **soft** limit (what applies now) and a **hard** limit (the most the soft limit can be raised to without privilege).

```bash
ulimit -a          # all limits of your shell
ulimit -n          # soft open files
ulimit -Hn         # hard open files
cat /proc/2890/limits    # a running process's limits
```

## 11.2 The trap: `limits.conf` doesn't apply to services

`/etc/security/limits.conf` is applied by **PAM at login** (`pam_limits`). It affects **login sessions** (SSH, `su`), **not systemd services** — those never go through a PAM login.

| For | Set limits in |
|---|---|
| Users' shells | `/etc/security/limits.conf` or `limits.d/` |
| A systemd service | `LimitNOFILE=` etc. in the unit |
| All services by default | `DefaultLimitNOFILE=` in `/etc/systemd/system.conf` |

systemd's default for services is `LimitNOFILE=1024:524288` (soft 1024, hard 524288). Busy network services (nginx, Node APIs) usually need the soft limit raised.

**Used in:** Note 14, Step 9.

---

# Phase 12 — The OOM killer

## 12.1 What it is

When memory runs out and can't be reclaimed, the kernel's **OOM (Out Of Memory) killer** picks a process and kills it with **SIGKILL** to free memory. No cleanup, no warning to the process (Note 9: exit 137 = 128 + 9).

## 12.2 Two kinds of OOM

| | Global OOM | cgroup OOM |
|---|---|---|
| Trigger | The **whole system** is out of memory | A service went over its **`MemoryMax=`** |
| Victim | Any process on the system | Only a process **inside that cgroup** |
| Kernel log | `Out of memory: Killed process 8120 (python) ...` | `Memory cgroup out of memory: Killed process 8120 (python) ...` |

The second is what you want: **a misbehaving service kills only itself**, instead of the kernel choosing something on the whole machine — possibly your API or sshd.

## 12.3 How the victim is chosen

Each process has an **OOM score**, mostly based on how much memory it uses, adjusted by `oom_score_adj` (−1000 to 1000):

```bash
cat /proc/2890/oom_score        # higher = more likely to be killed
cat /proc/2890/oom_score_adj    # 0 = no adjustment
```

| `oom_score_adj` | Effect |
|---|---|
| `-1000` | Never killed |
| negative | Less likely |
| `0` | Normal |
| positive | More likely |

Unit setting: `OOMScoreAdjust=-500` makes a service a less likely victim in a **global** OOM.

## 12.4 What systemd reports

```
worker@2.service: A process of this unit has been killed by the OOM killer.
worker@2.service: Main process exited, code=killed, status=9/KILL
worker@2.service: Failed with result 'oom-kill'.
```

`OOMPolicy=` decides what systemd does when **any** process in the unit is OOM-killed:

| `OOMPolicy=` | Effect |
|---|---|
| `stop` (default) | Stop the **whole unit** (then `Restart=` may restart it) |
| `continue` | Leave the unit running — the main process deals with its lost child |
| `kill` | Kill all remaining processes in the unit |

`continue` suits a parent like the worker: if one `convert` child is killed for a huge image, the worker can mark that job failed and keep going.

## 12.5 Finding OOM kills

```bash
journalctl -k -g 'killed process'                           # kernel's record
journalctl -u 'worker@*' -g 'OOM killer'                    # systemd's record
cat /sys/fs/cgroup/system.slice/worker@2.service/memory.events   # oom_kill counter
```

Ubuntu may also run **systemd-oomd**, a user-space killer that acts **earlier**, based on memory **pressure** (Phase 13), mainly on desktops. If it kills something, its own log says so: `journalctl -u systemd-oomd`.

> **How this goes wrong:** a service with no `MemoryMax=` can push the whole server into a global OOM, and the kernel may kill something important instead. Limits turn a server-wide outage into a single restarted service.

**Used in:** Note 14, Step 7.

---

# Phase 13 — Pressure: seeing that limits are being hit

Limits are working when they're **sometimes reached**. You can see how often with **PSI (Pressure Stall Information)**: the share of time tasks were **waiting** for a resource.

```bash
cat /proc/pressure/memory           # whole system
cat /sys/fs/cgroup/system.slice/api.service/memory.pressure   # one service
```
```
some avg10=4.20 avg60=2.10 avg300=0.90 total=1934221
full avg10=0.00 avg60=0.00 avg300=0.00 total=12011
```

| Line | Meaning |
|---|---|
| `some` | % of time **at least one** task was stalled waiting for this resource |
| `full` | % of time **all** tasks were stalled — real slowdown |
| `avg10/60/300` | Averages over 10 s, 1 min, 5 min |

Same for `cpu.pressure` and `io.pressure`. Rising `memory` pressure for a service with `MemoryHigh=` means it's being throttled; `cpu.stat`'s `nr_throttled` rising means `CPUQuota=` is being hit.

The whole-machine use of PSI belongs to the resources area.

**Used in:** Note 14, Steps 6, 7.

---

# How this goes wrong — overview

| Issue | Why it matters | How to check |
|---|---|---|
| Logs only on the server | Lost with the disk; root attacker can delete them | Is a shipping agent running? (Phase 6) |
| Volatile journal | Every reboot erases evidence | `journalctl --list-boots` |
| Journal too small / short retention | Incident evidence rotated away | `journalctl --disk-usage`, `--list-boots` dates |
| Rate-limit suppression | Real errors dropped during floods (can be forced on purpose) | `journalctl -u systemd-journald -g Suppressed` |
| Secrets in logs | Anyone in `adm` reads them (Note 2) | Search for tokens, `password=`, connection strings |
| Unchecked cron and timers | Classic **persistence** for attackers | `systemctl list-timers --all`, all crontabs, `/etc/cron.*` |
| No resource limits | One bug or attack starves everything (DoS) | `systemctl show -p MemoryMax,TasksMax,CPUQuotaPerSecUSec` |
| No `TasksMax` on user sessions | Fork bomb by a logged-in user | `systemctl show user-1101.slice -p TasksMax` |
| Disk filled by logs | Services fail to write; can be triggered on purpose | `df`, `journalctl --disk-usage`, `lsof +L1` |

---

# Summary table

| Concept | Key point |
|---|---|
| journald | Collects stdout/stderr, syslog, kernel, systemd messages |
| Apps under systemd | Log to stdout; journald adds time, unit, PID, user |
| rsyslog | Ubuntu also writes text files (`syslog`, `auth.log`) |
| Journal fields | `_SYSTEMD_UNIT`, `_PID`, `_UID`... added by journald (trustworthy) |
| Priority | 0 emerg ... 3 err ... 6 info, 7 debug; `<3>` prefix sets it |
| `journalctl` | `-u`, `-b`, `--since/--until`, `-p`, `-k`, `-g`, `-f`, `-o json` |
| Storage | Persistent only if `/var/log/journal` exists (with `Storage=auto`) |
| Size | `SystemMaxUse`, `SystemKeepFree`, `MaxRetentionSec`; `--vacuum-*` |
| Rate limits | 10000 msgs / 30 s per service; "Suppressed N messages" |
| `LogLevelMax=` / `LogRateLimit*=` | Per-unit filtering and limits |
| logrotate | `create` + reopen signal, or `copytruncate` (may lose lines) |
| Deleted-but-open file | Space not freed until closed; `lsof +L1` |
| Ship logs | Survive disk loss and root attackers |
| Timer | `.timer` (when) + `.service` (what, `oneshot`); enable the timer |
| Timer benefits | Journal logs, catch-up, no overlap, env/sandbox/limits, alerts |
| `OnCalendar=` | `*-*-* 02:00:00 [TZ]`; test with `systemd-analyze calendar` |
| `OnUnitInactiveSec=` | Repeat with a gap after each finish |
| cgroup | Kernel grouping of processes for measuring and limiting |
| Slice | Group of units; limits apply to all of them combined |
| `CPUWeight=` / `CPUQuota=` | Share under contention / hard cap (% of one CPU) |
| `MemoryHigh=` / `MemoryMax=` | Throttle / OOM-kill inside the service |
| `TasksMax=` | Cap processes + threads (stops fork bombs) |
| `Nice=` / `IOSchedulingClass=` | Unit versions of `nice` / `ionice` |
| `set-property` | Change limits live |
| rlimits | Per-process; `ulimit`, `/proc/PID/limits`, `Limit*=` in units |
| `limits.conf` | Login sessions only — **not** systemd services |
| OOM killer | SIGKILL (exit 137); global vs cgroup OOM |
| `OOMScoreAdjust=` / `OOMPolicy=` | Protect a service / what happens to the unit after an OOM kill |
| PSI | `some`/`full` stall %; shows limits being hit |

---

# Appendix — Command reference

## Logs

| Command | Purpose |
|---|---|
| `journalctl -u UNIT [-u UNIT2]` | One or more units (globs ok) |
| `journalctl -b` / `-b -1` / `--list-boots` | This boot / previous / list |
| `journalctl --since "..." --until "..."` | Time window |
| `journalctl -p err` / `-p warning..err` | By priority |
| `journalctl -k` | Kernel |
| `journalctl -g PATTERN` | Search messages |
| `journalctl _UID=N` / `_PID=N` / `_COMM=name` | By field |
| `journalctl -f` / `-n N` / `-r` | Follow / last N / newest first |
| `journalctl -o short-iso` / `json-pretty` / `verbose` / `cat` | Output formats |
| `journalctl --user -u UNIT` | User service logs |
| `journalctl --disk-usage` | Journal size |
| `journalctl --vacuum-size=500M` / `--vacuum-time=14d` | Shrink |
| `systemd-analyze cat-config systemd/journald.conf` | Effective journald config |
| `systemctl restart systemd-journald` | Apply journald changes |
| `logrotate -d FILE` / `-f FILE` | Dry run / force |
| `lsof +L1` | Deleted-but-open files |

## Timers

| Command | Purpose |
|---|---|
| `systemctl list-timers --all` | All timers, next and last runs |
| `systemctl enable --now NAME.timer` | Schedule |
| `systemctl start NAME.service` | Run the job now |
| `systemd-analyze calendar --iterations=N 'EXPR'` | Test a schedule |
| `systemd-run --on-active=30min CMD` / `--on-calendar='...' CMD` | One-off timers |
| `timedatectl` | Server time zone |

## cgroups and limits

| Command | Purpose |
|---|---|
| `systemd-cgls` / `systemd-cgls -u UNIT` | Tree |
| `systemd-cgtop` | Live per-service usage |
| `cat /proc/PID/cgroup` | A process's cgroup |
| `cat /sys/fs/cgroup/.../memory.events` / `cpu.stat` / `pids.current` | Kernel counters |
| `systemctl show UNIT -p MemoryCurrent,MemoryPeak,TasksCurrent` | Current use |
| `systemctl show UNIT -p MemoryMax,MemoryHigh,CPUQuotaPerSecUSec,TasksMax` | Limits |
| `systemctl set-property [--runtime] UNIT Key=Val` | Change limits live |
| `systemd-run --wait -p MemoryMax=... -p CPUQuota=... CMD` | One-off with limits |
| `ulimit -a` / `ulimit -n` / `ulimit -Hn` | Shell rlimits |
| `cat /proc/PID/limits` | Process rlimits |

## OOM and pressure

| Command | Purpose |
|---|---|
| `journalctl -k -g 'killed process'` | Kernel OOM kills |
| `journalctl -u UNIT -g 'OOM killer'` | systemd's record |
| `cat /proc/PID/oom_score` / `oom_score_adj` | Victim likelihood |
| `cat /proc/pressure/{cpu,memory,io}` | System pressure |
| `cat /sys/fs/cgroup/.../memory.pressure` | Service pressure |

## Key unit settings

| Setting | Purpose |
|---|---|
| `StandardOutput=` / `StandardError=` | Where output goes (default journal) |
| `LogLevelMax=` / `LogRateLimitIntervalSec=` / `LogRateLimitBurst=` | Per-unit log control |
| `OnCalendar=` / `Persistent=` / `RandomizedDelaySec=` / `OnBootSec=` / `OnUnitInactiveSec=` | Timer schedule |
| `Slice=` | Put a service in a slice |
| `CPUWeight=` / `CPUQuota=` | CPU |
| `MemoryHigh=` / `MemoryMax=` / `MemoryLow=` | Memory |
| `TasksMax=` | Processes + threads |
| `IOWeight=` / `IOReadBandwidthMax=` / `IOWriteBandwidthMax=` | Disk |
| `Nice=` / `IOSchedulingClass=` | Priority |
| `LimitNOFILE=` / `LimitNPROC=` / `LimitCORE=` | rlimits |
| `OOMScoreAdjust=` / `OOMPolicy=` | OOM behaviour |