# Logs, Timers & Resource Limits — Practice (Note 14)

Making the team server's logs reliable, moving scheduled jobs to systemd timers, and putting limits on the services so one bad process can't take down the rest. Each step shows the command, what the result means, and the decision it leads to.

**Theory for this note:** Note 13. Each step points to the phase that explains it ("Why: Note 13, Phase X").

**Continues from:** Note 12 (the rebuilt units). This is the **last** note of the series. It closes the three items Note 12 left open — logs, the cron backup, and resource limits — and ends with a map of all 14 notes.

---

## Contents

- The setup
- Step 0 — Look at the starting state
- Step 1 — Investigating 500 errors at 02:04
- Step 2 — Logs gone after a reboot
- Step 3 — The journal filled the disk
- Step 4 — Moving the backup from cron to a timer
- Step 5 — A missed backup, and a repeating cleanup job
- Step 6 — Workers starving the API of CPU
- Step 7 — A memory leak and the OOM killer
- Step 8 — Capping processes per service
- Step 9 — "Too many open files" (and the `limits.conf` trap)
- Step 10 — Disk space that `du` can't find
- Step 11 — Shipping logs off the server
- Step 12 — Audit: timers, cron, logs, limits
- State of the server after Note 14
- The whole series
- Scenario recap
- Troubleshooting table
- Appendix — Command reference

---

# The setup

Same server as Notes 4–12 (`app-server-01`, Ubuntu 24.04, 2 vCPUs, 4 GB RAM).

| Unit | From |
|---|---|
| `myapps.target` → `api.service`, `worker@1..3.service` | Note 12 |
| `notify-failure@.service` (alerts) | Note 12, Step 5 |
| `nginx.service` + drop-in | Note 12, Step 7 |
| Backup: **cron** → `backup.sh` → `pg_dump` | Notes 2, 6, 10 |

Open items from Note 12: logs, the cron backup, no resource limits.

---

# Step 0 — Look at the starting state

### Logs

```bash
journalctl --disk-usage
journalctl --list-boots | tail -3
systemd-analyze cat-config systemd/journald.conf | grep -v '^#' | grep .
```
```
Archived and active journals take up 1.1G in the file system.

 -2 3f1a... Mon 2026-10-19 08:58:01 +0530  Mon 2026-10-19 23:59:59 +0530
 -1 a812... Tue 2026-10-20 09:00:03 +0530  Wed 2026-10-21 01:29:44 +0530
  0 7c1e... Wed 2026-10-21 03:01:12 +0530  Wed 2026-10-21 08:50:02 +0530

[Journal]
```

**Meaning:** the journal is **persistent** (several boots kept) and uses default limits — nothing set in `[Journal]`. (Why: Note 13, Phase 4.)

### Scheduled jobs

```bash
systemctl list-timers --all --no-pager | head -8
sudo crontab -l -u worker
```
```
NEXT                        LEFT     LAST                        PASSED  UNIT                 ACTIVATES
Wed 2026-10-21 09:12:00     21min    Wed 2026-10-21 08:12:00     38min   sysstat-collect.timer sysstat-collect.service
Thu 2026-10-22 00:00:00     15h      Wed 2026-10-21 00:00:00     8h      logrotate.timer       logrotate.service
...

0 2 * * * ionice -c 3 nice -n 10 /var/lib/worker/backup.sh
```

**Meaning:** system jobs already use timers (logrotate included). Our backup is the odd one out, in cron. (Why: Note 13, Phase 7.)

### Limits

```bash
systemctl show api -p MemoryMax,MemoryHigh,CPUQuotaPerSecUSec,TasksMax,LimitNOFILE
```
```
MemoryMax=infinity
MemoryHigh=infinity
CPUQuotaPerSecUSec=infinity
TasksMax=4558
LimitNOFILE=524288
```

Wait — `LimitNOFILE=524288`? That's the **hard** limit. The soft limit is what applies:

```bash
grep 'open files' /proc/$(systemctl show api -p MainPID --value)/limits
```
```
Max open files            1024                 524288               files
```

**Meaning:** no memory or CPU limits; tasks capped only by the default; **1024** open files (soft). Steps 6–9 deal with each. (Why: Note 13, Phases 10, 11.)

---

# Step 1 — Investigating 500 errors at 02:04

A customer reports errors from the API around 02:00 last night.

### Narrow to the service and time

```bash
journalctl -u api --since "2026-10-21 01:55" --until "02:20" -p err --no-pager
```
```
02:03:12 node[2890]: Error: connect ETIMEDOUT 10.0.0.5:5432
02:03:41 node[2890]: Error: connect ETIMEDOUT 10.0.0.5:5432
...
02:09:58 node[2890]: Error: connect ETIMEDOUT 10.0.0.5:5432
```

**Meaning:** for ~7 minutes, the API couldn't open new database connections. `-p err` hid the normal `info` lines. (Why: Note 13, Phase 3.1.)

### Look at **everything** on the server in that window

```bash
journalctl --since "2026-10-21 01:55" --until "02:20" --no-pager -o short-iso | grep -v 'node\['
```
```
2026-10-21T02:00:01+0530 CRON[11021]: (worker) CMD (ionice -c 3 nice -n 10 /var/lib/worker/backup.sh)
2026-10-21T02:11:47 CRON[11020]: (CRON) info (No MTA installed, discarding output)
```

**Meaning:** the backup started at 02:00; errors began at 02:03 and ended around when it finished (02:11). And cron **threw away the backup's output** — we can't see what it did or whether it succeeded. (Why: Note 13, Phase 7.1.)

### Use fields to be precise

```bash
journalctl -u api --since "02:00" --until "02:15" -o json --no-pager \
  | jq -r 'select(.PRIORITY=="3") | .__REALTIME_TIMESTAMP' | wc -l
```
```
42
```

**Meaning:** 42 error entries. `-o json` exposes every field for scripts. (Why: Note 13, Phase 2.)

Search across the whole journal for the same text:

```bash
journalctl --since "7 days ago" -g 'ETIMEDOUT 10.0.0.5' --no-pager | awk '{print $1, $2}' | uniq -c
```
```
     44 Oct 19
     38 Oct 20
     42 Oct 21
```

**Meaning:** it happens **every night**. Not a one-off.

### Conclusion and next steps

The nightly backup puts heavy load on the database, so the API's new connections time out. Two actions:
- Database side: run the dump against a read replica, or tune it (outside this server — it's the database team's area).
- Server side: run the backup as a **timer** so its output, exit status, and duration are recorded, and it can be limited (Step 4).

---

# Step 2 — Logs gone after a reboot

`app-server-02` (built from a minimal image, Note 4, Step 12) crashed overnight. After it rebooted, ops wants to see what happened before the crash.

```bash
journalctl --list-boots
```
```
 0 d41b... Wed 2026-10-21 04:12:30 +0530  Wed 2026-10-21 08:55:10 +0530
```

```bash
journalctl -b -1
```
```
Specifying boot ID or boot offset has no effect, no persistent journal was found.
```

**Meaning:** only the current boot exists. The journal was **volatile** (kept in memory), so the crash evidence was lost at reboot. (Why: Note 13, Phase 4.1.)

```bash
ls -ld /var/log/journal
# ls: cannot access '/var/log/journal': No such file or directory
```

With the default `Storage=auto`, no folder = volatile.

### Fix

Be explicit, so it doesn't depend on a folder existing:

`/etc/systemd/journald.conf.d/10-storage.conf`:

```ini
[Journal]
Storage=persistent
```

```bash
sudo systemctl restart systemd-journald
sudo journalctl --flush           # move what's in memory to disk now
ls -ld /var/log/journal           # created
```

After the next reboot:

```bash
journalctl --list-boots | tail -2      # two boots listed
```

Add this drop-in to the provisioning scripts (Note 4, Step 12; Note 6, Step 12) so every new server gets it.

---

# Step 3 — The journal filled the disk

A disk alert: `/` is 92% full.

### Find what's using it

```bash
df -h /
sudo du -sh /var/log/* 2>/dev/null | sort -h | tail -3
journalctl --disk-usage
```
```
/dev/sda1   39G   36G  3.1G  92% /

512M  /var/log/syslog.1
3.9G  /var/log/journal
...
Archived and active journals take up 3.9G in the file system.
```

**Meaning:** the journal is near its default cap (10% of disk, max 4 GB). That's "normal" for journald — but the disk was already full of other data, and 3.9 GB of logs pushed it over the edge. (Why: Note 13, Phase 4.3.)

### Who is writing so much?

```bash
for u in api worker@1 worker@2 worker@3 nginx; do
  printf '%-10s %s\n' "$u" "$(journalctl -u "$u" --since '1 hour ago' -o cat --no-pager | wc -l)"
done
```
```
api        1203
worker@1   1180
worker@2   612044
worker@3   1166
nginx      9
```

```bash
journalctl -u worker@2 -n 3 -o cat --no-pager
```
```
DEBUG fetched job 81122 payload={"id":81122, ...}
DEBUG poll queue: empty
DEBUG poll queue: empty
```

**Meaning:** `worker@2` is logging **debug** lines in a tight loop. Someone set `LOG_LEVEL=debug` (in `worker.env`, Note 2) while testing and never set it back.

### Lines were also lost

```bash
journalctl -u systemd-journald --since today -g Suppressed --no-pager | tail -2
```
```
systemd-journald[301]: Suppressed 48211 messages from worker@2.service
```

**Meaning:** the flood hit the rate limit (10000 per 30 s), so journald dropped messages — including any **real** errors from worker@2 in that time. (Why: Note 13, Phase 4.4.)

### Fix now

```bash
sudo journalctl --vacuum-size=1G
df -h /          # back to ~83%
```

Set `LOG_LEVEL=info` in `worker.env` and restart the workers.

### Fix permanently

**1. Cap the journal** — `/etc/systemd/journald.conf.d/20-size.conf`:

```ini
[Journal]
SystemMaxUse=1G
SystemKeepFree=5G
MaxRetentionSec=1month
```

```bash
sudo systemctl restart systemd-journald
```

With logs shipped off the server (Step 11), a smaller local journal is fine.

**2. Stop debug lines at the source**, whatever the app's setting — in `worker@.service`:

```ini
[Service]
LogLevelMax=info
```

Even if someone sets debug again, debug lines are dropped before they're stored. (Why: Note 13, Phase 4.4.)

(For `LogLevelMax=` to recognise debug lines, the app must mark them — e.g. with a `<7>` prefix or a journal logging handler, Note 13, Phase 2.2. Otherwise all stdout lines arrive as `info`.)

---

# Step 4 — Moving the backup from cron to a timer

Step 1 showed cron hides the backup's output. Also from earlier notes: cron's tiny environment (Note 2), and `ionice`/`nice` written into the cron line by hand (Note 10, Step 7).

### What cron recorded

```bash
grep -E 'CRON.*(backup|MTA)' /var/log/syslog | tail -2
```
```
Oct 21 02:00:01 app-server-01 CRON[11021]: (worker) CMD (ionice -c 3 nice -n 10 /var/lib/worker/backup.sh)
Oct 21 02:11:47 app-server-01 CRON[11020]: (CRON) info (No MTA installed, discarding output)
```

That's all. No exit code, no errors, no duration. If the backup had failed for a week, nobody would know.

### Move the script to a root-owned place

The script currently lives in `/var/lib/worker`, which `worker` can write. The unit audit (Note 12, Step 11) wants programs started by units to be root-owned:

```bash
sudo install -d -m 755 /usr/local/lib/myapps
sudo install -m 755 -o root -g root /var/lib/worker/backup.sh /usr/local/lib/myapps/backup.sh
```

(The script keeps its `umask 027`, `trap`, and background-plus-`wait` pattern from Note 10, Step 3.)

### The service

`/etc/systemd/system/backup.service`:

```ini
[Unit]
Description=Nightly database backup
Wants=network-online.target
After=network-online.target
OnFailure=notify-failure@%n.service

[Service]
Type=oneshot
User=worker
Group=worker
EnvironmentFile=/etc/myapps/common.env
EnvironmentFile=/etc/myapps/worker.env
ExecStart=/usr/local/lib/myapps/backup.sh
TimeoutStartSec=2h

# priority (replaces nice/ionice in the cron line — Note 10, Step 7)
Nice=10
IOSchedulingClass=idle

# limits and sandbox
CPUQuota=50%
MemoryMax=512M
UMask=0027
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/var/backups/appdb
```

Notes on the choices:

| Setting | Why |
|---|---|
| `Type=oneshot` | A job that runs and ends (Note 11, Phase 6) |
| **No `[Install]`** | Started only by the timer (`static`) |
| `TimeoutStartSec=2h` | For `oneshot`, there's **no start timeout by default** — a stuck `pg_dump` would run forever. Cap it |
| `Nice=`, `IOSchedulingClass=` | The unit versions of `nice`/`ionice` (Note 13, Phase 10.5) |
| `CPUQuota=50%` | Hard cap: half a core for compression, whatever happens |
| `OnFailure=` | Alert if it fails (Note 12, Step 5) |
| Sandbox | It can write only the backup folder (Note 8) |

### The timer

`/etc/systemd/system/backup.timer`:

```ini
[Unit]
Description=Nightly database backup schedule

[Timer]
OnCalendar=*-*-* 02:00:00 Asia/Colombo
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
```

Check the schedule before trusting it:

```bash
timedatectl | grep 'Time zone'
systemd-analyze calendar --iterations=2 '*-*-* 02:00:00 Asia/Colombo'
```
```
                Time zone: Etc/UTC (UTC, +0000)
    Next elapse: Thu 2026-10-22 02:00:00 +0530
       (in UTC): Wed 2026-10-21 20:30:00 UTC
```

**Meaning:** the server runs in **UTC**. Writing the time zone in the schedule keeps the job at 02:00 Colombo time regardless. (Why: Note 13, Phase 8.1.)

### Enable, test, remove cron

```bash
sudo systemd-analyze verify /etc/systemd/system/backup.service /etc/systemd/system/backup.timer
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
```

Run it once now, without waiting for 02:00:

```bash
sudo systemctl start --no-block backup.service
journalctl -u backup.service -f
```
```
09:12:03 systemd[1]: Starting backup.service - Nightly database backup...
09:23:41 systemd[1]: backup.service: Deactivated successfully.
09:23:41 systemd[1]: Finished backup.service - Nightly database backup.
09:23:41 systemd[1]: backup.service: Consumed 9min 2.110s CPU time.
```

**Meaning:** start, finish, success, and CPU time — all recorded. (`--no-block` returns immediately; without it, `systemctl start` waits for a `oneshot` to finish.)

Test a failure on purpose — set a wrong `PGPASSWORD` temporarily:

```bash
sudo systemctl start backup.service
systemctl status backup.service --no-pager | head -4
```
```
× backup.service - Nightly database backup
     Active: failed (Result: exit-code) since ...
    Process: 12210 ExecStart=/usr/local/lib/myapps/backup.sh (code=exited, status=1/FAILURE)
```

The alert from `notify-failure@backup.service` arrives with the `pg_dump` error lines. Restore the password.

Remove the cron entry:

```bash
sudo crontab -l -u worker | grep -v backup.sh | sudo crontab -u worker -
sudo crontab -l -u worker        # gone
```

```bash
systemctl list-timers backup.timer --no-pager
```
```
NEXT                        LEFT LAST                        PASSED UNIT         ACTIVATES
Thu 2026-10-22 02:06:41 ... 16h  Wed 2026-10-21 09:12:03 ... 11min  backup.timer backup.service
```

**Conclusion:** same job, but now visible, alerting, limited, and sandboxed.

---

# Step 5 — A missed backup, and a repeating cleanup job

### The missed run

The server was shut down from 01:30 to 03:00 for a disk resize. With cron, the 02:00 backup would simply not have run that day.

```bash
journalctl -u backup.service -b --no-pager | head -2
```
```
03:01:40 systemd[1]: Starting backup.service - Nightly database backup...
```

**Meaning:** `Persistent=true` remembered that the 02:00 run was missed, and ran it right after boot. (Why: Note 13, Phase 7.3.)

### A job that repeats with a gap

Note 12, Step 2 created `uploads-cleanup.service` (a `static` oneshot that deletes processed uploads older than 7 days). Now schedule it: 10 minutes after boot, then **15 minutes after each run finishes**.

`/etc/systemd/system/uploads-cleanup.timer`:

```ini
[Unit]
Description=Clean up processed uploads regularly

[Timer]
OnBootSec=10min
OnUnitInactiveSec=15min

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now uploads-cleanup.timer
systemctl list-timers uploads-cleanup.timer --no-pager
```

**Why `OnUnitInactiveSec=` and not "every 15 minutes":** if a cleanup run ever takes 20 minutes (a huge backlog), the next run starts 15 minutes **after it finishes**, never on top of it. And even with `OnCalendar=*:0/15`, systemd won't start a second copy of a service that's still running — unlike cron. (Why: Note 13, Phases 7.2, 8.2.)

---

# Step 6 — Workers starving the API of CPU

At 11:00 a customer uploads 5000 high-resolution images. The API becomes slow.

```bash
systemd-cgtop -n 1
```
```
Control Group                       Tasks   %CPU   Memory
/                                     201  199.2     2.9G
/system.slice                         120  196.8     2.1G
/system.slice/worker@1.service          6   64.1   310.2M
/system.slice/worker@2.service          6   63.8   402.9M
/system.slice/worker@3.service          6   62.5   298.0M
/system.slice/api.service              11    5.9   110.4M
```

**Meaning:** on this 2-CPU server (200% total), three workers take ~190%. The API gets what's left. `systemd-cgtop` shows usage **per service**, which `top` can't. (Why: Note 13, Phase 9.3.)

### Fix: a slice with a combined cap for all workers

`/etc/systemd/system/workers.slice`:

```ini
[Unit]
Description=All worker instances

[Slice]
CPUQuota=100%
CPUWeight=50
```

In `worker@.service`:

```ini
[Service]
Slice=workers.slice
```

And give the API a larger share under contention, in `api.service`:

```ini
[Service]
CPUWeight=200
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapps.target
systemd-cgls -u workers.slice --no-pager
```
```
workers.slice
├─worker@1.service
├─worker@2.service
└─worker@3.service
```

| Setting | Effect |
|---|---|
| `CPUQuota=100%` on the slice | All workers **together** can use at most one core — however many instances run |
| `CPUWeight=50` / `CPUWeight=200` | When CPU is busy, the API gets 4× the share of the workers |

(Why: Note 13, Phases 10.1, 10.6.)

### Verify during the next big batch

```bash
systemd-cgtop -n 1 | grep -E 'workers|api'
cat /sys/fs/cgroup/workers.slice/cpu.stat | grep throttled
```
```
/workers.slice                         18   99.6     1.0G
/system.slice/api.service              11   38.4   112.1M

nr_throttled 1820
throttled_usec 451200133
```

**Meaning:** workers hold at one core; the cap is being hit (`nr_throttled` rising) — so it's doing its job. The batch takes longer, and the API stays fast. That's the trade-off you chose.

---

# Step 7 — A memory leak and the OOM killer

One morning: the API was unreachable for ~20 minutes overnight, and `worker@2` restarted.

### What happened

```bash
journalctl -k --since yesterday -g 'killed process' --no-pager
```
```
03:41:09 kernel: Out of memory: Killed process 8120 (python) total-vm:3611204kB, anon-rss:2931440kB, ...
```

```bash
journalctl -u 'worker@2' --since "03:40" --until "03:43" --no-pager
```
```
03:41:09 systemd[1]: worker@2.service: A process of this unit has been killed by the OOM killer.
03:41:09 systemd[1]: worker@2.service: Main process exited, code=killed, status=9/KILL
03:41:09 systemd[1]: worker@2.service: Failed with result 'oom-kill'.
03:41:11 systemd[1]: worker@2.service: Scheduled restart job, restart counter is at 1.
```

**Meaning:**
- `Out of memory` (no "cgroup") — a **global** OOM: the **whole server** ran out of memory. (Why: Note 13, Phase 12.2.)
- The kernel chose the biggest process — the leaking `worker@2`, at 2.9 GB of 4 GB.
- But before that kill, the server spent ~20 minutes almost out of memory, constantly reclaiming — everything, including the API, slowed to a crawl.

Check the pressure history the next time it grows (Note 13, Phase 13):

```bash
cat /sys/fs/cgroup/workers.slice/worker@2.service/memory.pressure
```

### Fix: contain it inside the worker

Per instance, in `worker@.service`:

```ini
[Service]
MemoryHigh=500M
MemoryMax=700M
OOMPolicy=continue
```

And for all workers together, in `workers.slice`:

```ini
[Slice]
MemoryMax=1800M
```

And protect the API in a **global** OOM, in `api.service`:

```ini
[Service]
OOMScoreAdjust=-500
```

| Setting | Effect |
|---|---|
| `MemoryHigh=500M` | Above 500 MB, the worker is slowed — a visible warning |
| `MemoryMax=700M` | Above 700 MB, the kernel kills a process **inside this worker only** |
| Slice `MemoryMax=1800M` | All workers together never take more than 1.8 GB — the API and system keep the rest |
| `OOMPolicy=continue` | If a **child** (`convert` on a huge image) is OOM-killed, the worker keeps running and marks that job failed. If the **main** worker process is killed, the unit still exits and `Restart=` brings it back |
| `OOMScoreAdjust=-500` | In a global OOM, the API is a much less likely victim |

(Why: Note 13, Phases 10.2, 12.3, 12.4.)

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapps.target
systemctl show worker@2 -p MemoryHigh,MemoryMax,OOMPolicy
```

### Next time the leak grows

```bash
journalctl -k -g 'killed process' --since today --no-pager
cat /sys/fs/cgroup/workers.slice/worker@2.service/memory.events
```
```
14:10:33 kernel: Memory cgroup out of memory: Killed process 9921 (python) total-vm:1102331kB, anon-rss:698120kB, ...

low 0
high 2210
max 41
oom 1
oom_kill 1
```

**Meaning:** `Memory cgroup out of memory` — the kill stayed **inside** `worker@2`. `high 2210` shows it was throttled for a while first. The API and other workers weren't touched; `worker@2` restarted in 2 seconds.

The leak itself still has to be fixed in the code — limits only stop it from becoming an outage.

---

# Step 8 — Capping processes per service

Note 10, Step 5 fixed the zombie bug in the code. A cap makes sure a similar bug can't exhaust the system's PIDs.

In `worker@.service`:

```ini
[Service]
TasksMax=100
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart myapps.target
systemctl status worker@1 --no-pager | grep Tasks
```
```
      Tasks: 6 (limit: 100)
```

**Meaning:** normal use is 6 (the worker, its threads, a converter); 100 leaves plenty of room. If a bug starts children without limit, it hits 100 and gets:

```
OSError: [Errno 11] Resource temporarily unavailable
```

— an error **in that worker only**, instead of the whole server being unable to start processes. (Why: Note 13, Phase 10.3.)

### Users too

The same idea protects against a runaway script (or fork bomb) from a logged-in user:

```bash
systemctl show user-1101.slice -p TasksMax
```
```
TasksMax=10813
```

That's systemd's default per-user cap. It's already there; just know where it comes from.

---

# Step 9 — "Too many open files" (and the `limits.conf` trap)

During a traffic spike, the API logs:

```
Error: EMFILE: too many open files, accept
```

### Check the real limit and use

```bash
PID=$(systemctl show api -p MainPID --value)
grep 'open files' /proc/$PID/limits
sudo ls /proc/$PID/fd | wc -l
```
```
Max open files            1024                 524288               files
1024
```

**Meaning:** every connection is a file descriptor. The API hit its **soft** limit of 1024. (Why: Note 13, Phase 11.1.)

### The trap

Someone had already "fixed" this last month, in `/etc/security/limits.conf`:

```
*   soft   nofile   65535
```

And it seems to work:

```bash
ulimit -n        # in an SSH session
# 65535
```

But the API still has 1024.

**Meaning:** `limits.conf` is applied by PAM at **login**. SSH sessions get it; **systemd services never log in**, so they don't. (Why: Note 13, Phase 11.2.)

### Fix in the unit

In `api.service`:

```ini
[Service]
LimitNOFILE=65535
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart api
grep 'open files' /proc/$(systemctl show api -p MainPID --value)/limits
```
```
Max open files            65535                65535                files
```

(nginx already got this in Note 12, Step 7.) Leave `limits.conf` for interactive users only.

---

# Step 10 — Disk space that `du` can't find

A disk alert again — but this time, the numbers don't match:

```bash
df -h /
sudo du -shx / 2>/dev/null
```
```
/dev/sda1   39G   31G  6.1G  84% /
24G     /
```

**Meaning:** `df` (the filesystem's count) says 31 GB used; `du` (adding up visible files) finds only 24 GB. **7 GB is used by files that have no name** — deleted, but still open. (Why: Note 13, Phase 5.2.)

```bash
sudo lsof +L1
```
```
COMMAND  PID USER  FD   TYPE DEVICE   SIZE/OFF NLINK  NODE NAME
node    2890  api  21w   REG    8,1 7489203311     0 41022 /var/log/myapps/api/app.log.1 (deleted)
```

**Meaning:**
- logrotate renamed `app.log` → `app.log.1` and created a new `app.log` (Note 6, Step 6).
- The API **never reopened** its log file, so it kept writing to `app.log.1`.
- Days later, logrotate compressed and **deleted** `app.log.1`. The API still has it open, so the space isn't freed — and it keeps growing.

### Free it now

```bash
sudo systemctl restart api      # closes the file → space released
df -h /                         # back to 24G used
```

### Fix permanently

Options, best first:

1. **Log to stdout only** and let journald store and rotate (Note 13, Phase 1.2). Remove the file logging from the API. Logs still reach developers and auditors through `journalctl` (`adm` group; give `devs` access with the `systemd-journal` group or by shipping logs, Step 11).
2. If file logs are required: make the app **reopen** its file on a signal, and add to the logrotate rule:
   ```
   postrotate
       systemctl kill -s USR1 api.service
   endscript
   ```
3. `copytruncate` — works without app changes, but can lose lines during rotation.

The team chooses option 1 for `app.log`. The append-only `audit.trail` (Note 6, Step 6) stays as a file, rotated monthly, with the API reopening it on `SIGUSR1`.

---

# Step 11 — Shipping logs off the server

Two reasons from this series:
- If `app-server-02`'s disk had failed (Step 2), even a persistent journal would be gone.
- The attacker in Note 10, Step 9 ran as `api`. With root, an attacker could delete local logs.

The servers run on GCP, so use the **Google Cloud Ops Agent**, which sends logs to **Cloud Logging**. (Why: Note 13, Phase 6.)

### Install

```bash
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install
systemctl status google-cloud-ops-agent --no-pager | head -3
```

### Configure: the journal and the audit trail

`/etc/google-cloud-ops-agent/config.yaml`:

```yaml
logging:
  receivers:
    myapps_journal:
      type: systemd_journald
    myapps_audit:
      type: files
      include_paths:
        - /var/log/myapps/api/audit.trail
  service:
    pipelines:
      myapps:
        receivers: [myapps_journal, myapps_audit]
```

```bash
sudo systemctl restart google-cloud-ops-agent
```

(The agent's default pipeline already sends `/var/log/syslog` and related files. Check the current Ops Agent documentation for receiver options — they evolve.)

### Permissions

The VM's service account needs permission to write logs (the **Logs Writer** role, `roles/logging.logWriter`). Grant only that — not broad editor roles.

### Check

In Cloud Logging, filter on the VM and `jsonPayload._SYSTEMD_UNIT="api.service"` — the journal fields from Note 13, Phase 2 come through as structured fields.

**Conclusion:** logs now survive the server, can't be edited from it, and are searchable across `app-server-01` and `-02` in one place. The local journal (Step 3) can stay small.

---

# Step 12 — Audit: timers, cron, logs, limits

Add these to the audit scripts from Notes 8 and 12.

```bash
echo "=== All timers (expect: system timers + backup + uploads-cleanup) ==="
systemctl list-timers --all --no-pager

echo "=== User crontabs (expect: none after Step 4) ==="
for u in $(getent passwd | cut -d: -f1); do
  c=$(sudo crontab -l -u "$u" 2>/dev/null | grep -Ev '^\s*(#|$)')
  [ -n "$c" ] && echo "-- $u --" && echo "$c"
done

echo "=== System cron folders (expect: package files only) ==="
ls -la /etc/crontab /etc/cron.d /etc/cron.hourly /etc/cron.daily /etc/cron.weekly /etc/cron.monthly

echo "=== Journal: storage, size, oldest entry ==="
journalctl --disk-usage
journalctl --list-boots --no-pager | head -1

echo "=== Messages suppressed by rate limits (last 7 days) ==="
journalctl -u systemd-journald --since "7 days ago" -g Suppressed --no-pager | tail -5

echo "=== OOM kills (last 7 days) ==="
journalctl -k --since "7 days ago" -g 'killed process' --no-pager

echo "=== Limits on our services ==="
for u in api worker@1 worker@2 worker@3 backup; do
  systemctl show "$u" -p Id -p MemoryMax -p TasksMax -p CPUQuotaPerSecUSec -p LimitNOFILESoft --value | paste -sd ' '
done

echo "=== Deleted but open files ==="
sudo lsof +L1 2>/dev/null

echo "=== Log shipping agent running? ==="
systemctl is-active google-cloud-ops-agent
```

### Reading it

| Check | Investigate if… |
|---|---|
| Timers | A timer you don't recognise — scheduled **persistence** is a classic attacker technique |
| Crontabs / cron folders | Any user crontab, or a file in `/etc/cron.d` not from a package (`dpkg -S file`) |
| Journal | Only one boot listed; oldest entry very recent |
| Suppressed | Any service being rate-limited — lost lines |
| OOM kills | Any — and whether it was global or cgroup |
| Limits | `infinity` on our app services |
| Deleted-open files | Large sizes |
| Agent | Not `active` — logs aren't leaving the server |

---

# State of the server after Note 14

### Logs

| Item | Setting |
|---|---|
| Journal | Persistent; max 1 GB; keep 5 GB free; 1 month retention |
| Shipping | Ops Agent → Cloud Logging (journal + audit trail) |
| API logs | stdout → journal; audit trail as append-only file |
| Workers | `LogLevelMax=info` |

### Scheduled jobs

| Timer | Runs |
|---|---|
| `backup.timer` | 02:00 Asia/Colombo, catches up if missed, random delay ≤ 10 min |
| `uploads-cleanup.timer` | 10 min after boot, then 15 min after each run ends |
| cron | No team jobs left |

### Resource limits

| Unit | CPU | Memory | Tasks | Open files | OOM |
|---|---|---|---|---|---|
| `api.service` | `CPUWeight=200` | — | default | 65535 | `OOMScoreAdjust=-500` |
| `workers.slice` (all workers) | `CPUQuota=100%`, `CPUWeight=50` | `MemoryMax=1800M` | — | — | — |
| `worker@N.service` | (from slice) | `MemoryHigh=500M`, `MemoryMax=700M` | 100 | default | `OOMPolicy=continue` |
| `backup.service` | `CPUQuota=50%`, `Nice=10`, idle I/O | `MemoryMax=512M` | default | default | — |

The server is now complete: accounts (Note 4), files (Note 6), privilege (Note 8), processes (Note 10), services (Note 12), and logs, schedules, and limits (this note).

---

# The whole series

| Note | Topic | The question it answers |
|---|---|---|
| 1 | Shell & Processes | How does a command run? |
| 2 | Environment Variables | What settings does a process carry? |
| 3 / 4 | Users, Groups & Identity | **Who** is a process? |
| 5 / 6 | Files, Directories & Permissions | **What** does a file allow? |
| 7 / 8 | Privilege & Delegation | How do you allow exceptions, narrowly? |
| 9 / 10 | Processes & Signals | What is a running program, and how do you control it? |
| 11 / 12 | systemd & Services | How are services started, ordered, restarted, and stopped? |
| 13 / 14 | Logs, Timers & Resource Limits | What happens around a running service: records, schedules, and limits |

Each practice note built on the same server, so the final state of every note is the starting point of the next.

---

# Scenario recap

| Step | What happened | Concept (Note 13) |
|---|---|---|
| 0 | Checked journal, timers, limits | Storage, timers, cgroup limits, rlimits (4, 7, 10, 11) |
| 1 | Traced nightly API errors to the backup | `journalctl` filters, fields, `-g` (2, 3) |
| 2 | No logs from before a crash | Volatile vs persistent journal (4.1) |
| 3 | Journal filled the disk; lines suppressed | Size limits, rate limits, `LogLevelMax=` (4.3, 4.4) |
| 4 | Backup moved from cron to a timer | Timer + oneshot service, `Persistent=`, time zone (7, 8) |
| 5 | Missed run caught up; cleanup with a gap | `Persistent=`, `OnUnitInactiveSec=` (7.3, 8.2) |
| 6 | Workers starved the API | `systemd-cgtop`, slice `CPUQuota=`, `CPUWeight=` (9, 10) |
| 7 | Global OOM from a leak | `MemoryHigh/Max=`, `OOMPolicy=`, `OOMScoreAdjust=` (10, 12) |
| 8 | Capped processes | `TasksMax=` (10.3) |
| 9 | EMFILE despite `limits.conf` | `LimitNOFILE=`; PAM limits don't apply to services (11) |
| 10 | `df` vs `du` mismatch | Deleted-but-open file, logrotate, log to stdout (5) |
| 11 | Logs shipped to Cloud Logging | Off-server logs (6) |
| 12 | Audit of timers, cron, logs, limits | Security overview |

---

# Troubleshooting table

| Symptom | Likely cause | Check | Fix |
|---|---|---|---|
| Can't see logs from before a reboot | Volatile journal | `journalctl --list-boots` | `Storage=persistent` |
| Disk full of logs | Noisy service / no journal cap | `journalctl --disk-usage`, per-unit line counts | Vacuum; `SystemMaxUse=`; `LogLevelMax=`; fix the app |
| "Suppressed N messages" | Rate limit hit | `journalctl -u systemd-journald -g Suppressed` | Fix the flood; `LogRateLimit*=` if needed |
| Cron job failed silently | Output discarded (no mail) | `grep CRON /var/log/syslog` | Move to a timer |
| Timer never ran | Service enabled instead of timer, or bad schedule | `systemctl list-timers --all`, `systemd-analyze calendar` | `enable --now X.timer` |
| Job ran at the wrong hour | Server in UTC | `timedatectl` | Add the time zone to `OnCalendar=` |
| Missed run after downtime | No `Persistent=true` | `list-timers` LAST | Add `Persistent=true` |
| Oneshot job runs forever | No start timeout for `oneshot` | `systemctl status` duration | `TimeoutStartSec=` |
| One service slows the others | No CPU limits | `systemd-cgtop` | `CPUWeight=`, `CPUQuota=`, slices |
| Global OOM, wrong process killed | No `MemoryMax=` | `journalctl -k -g 'killed process'` (no "cgroup") | `MemoryMax=` per service/slice; `OOMScoreAdjust=` |
| Service restarts with `'oom-kill'` | Hit its own `MemoryMax=` | `memory.events` | Fix the leak; raise limit if genuinely needed |
| Whole worker stops when a child is OOM-killed | `OOMPolicy=stop` (default) | Journal | `OOMPolicy=continue` + handle failed jobs |
| `Resource temporarily unavailable` on fork | `TasksMax=` reached | `systemctl status` Tasks line | Find the leak of processes/threads; raise only if justified |
| `EMFILE: too many open files` | Soft `LimitNOFILE` 1024 | `/proc/PID/limits`, `ls /proc/PID/fd \| wc -l` | `LimitNOFILE=` in the unit (not `limits.conf`) |
| `df` much higher than `du` | Deleted-but-open files | `lsof +L1` | Restart/reopen; fix rotation; log to stdout |
| Logs missing in Cloud Logging | Agent down or no permission | `systemctl status google-cloud-ops-agent` | Fix config; grant `roles/logging.logWriter` |

---

# Appendix — Command reference

## Logs

| Command | Purpose |
|---|---|
| `journalctl -u UNIT --since "..." --until "..." -p err` | Errors in a window |
| `journalctl --since "..." --until "..." -o short-iso` | Everything in a window |
| `journalctl -g PATTERN --since "7 days ago"` | Search |
| `journalctl -u UNIT -o json \| jq ...` | Script on fields |
| `journalctl -k -g 'killed process'` | OOM kills |
| `journalctl -u systemd-journald -g Suppressed` | Rate-limit drops |
| `journalctl --list-boots` / `-b -1` | Boots / previous boot |
| `journalctl --disk-usage` / `--vacuum-size=1G` / `--flush` | Size / shrink / write to disk |
| `systemd-analyze cat-config systemd/journald.conf` | Effective config |
| `lsof +L1` | Deleted-but-open files |

## Timers

| Command | Purpose |
|---|---|
| `systemctl list-timers --all` | All timers |
| `systemctl enable --now NAME.timer` | Schedule |
| `systemctl start --no-block NAME.service` | Run now without waiting |
| `systemd-analyze calendar --iterations=N 'EXPR'` | Test a schedule |
| `systemd-analyze verify FILES` | Check units |
| `timedatectl` | Time zone |
| `crontab -l -u USER` / `crontab -r -u USER` | List / remove crontab |

## Limits

| Command | Purpose |
|---|---|
| `systemd-cgtop -n 1` | Usage per service |
| `systemd-cgls -u UNIT` | Tree for a unit/slice |
| `systemctl show UNIT -p MemoryMax,MemoryHigh,TasksMax,CPUQuotaPerSecUSec` | Limits |
| `systemctl show UNIT -p MemoryCurrent,MemoryPeak,TasksCurrent` | Use |
| `systemctl set-property [--runtime] UNIT Key=Val` | Change live |
| `cat /sys/fs/cgroup/<path>/memory.events` / `cpu.stat` / `memory.pressure` | Kernel counters |
| `cat /proc/PID/limits` / `ls /proc/PID/fd \| wc -l` | rlimits / open files |
| `ulimit -n` | Shell limit (logins only) |

## Unit snippets used in this note

| Snippet | Purpose |
|---|---|
| `Storage=persistent`, `SystemMaxUse=1G`, `SystemKeepFree=5G`, `MaxRetentionSec=1month` | journald drop-ins |
| `LogLevelMax=info` | Drop debug lines for a unit |
| `Type=oneshot` + `TimeoutStartSec=2h` + no `[Install]` | A scheduled job's service |
| `OnCalendar=*-*-* 02:00:00 Asia/Colombo` + `Persistent=true` + `RandomizedDelaySec=10min` | Nightly timer |
| `OnBootSec=10min` + `OnUnitInactiveSec=15min` | Repeat with a gap |
| `Nice=10` + `IOSchedulingClass=idle` | Low priority |
| `Slice=workers.slice` + `[Slice] CPUQuota=100%` | Shared CPU cap |
| `MemoryHigh=` + `MemoryMax=` + `OOMPolicy=continue` | Contain a leak |
| `OOMScoreAdjust=-500` | Protect from global OOM |
| `TasksMax=100` | Cap processes/threads |
| `LimitNOFILE=65535` | Open files for a service |