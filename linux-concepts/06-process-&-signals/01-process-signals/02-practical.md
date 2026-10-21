# Processes & Signals — Practice (Note 10)

Finding, inspecting, controlling, and stopping processes on the team server, step by step. Each step shows the command, what the result means, and the decision it leads to.

**Theory for this note:** Note 9. Each step points to the phase that explains it ("Why: Note 9, Phase X").

**Continues from:** Note 8 (the finished server: accounts, folders, sudo rules, sandboxed services). **Continues into:** Note 12 (writing and managing these services with systemd properly).

---

## Contents

- The setup
- Step 0 — Look around: the process tree on the server
- Step 1 — "The server is slow": find the process
- Step 2 — A long job that died when the SSH session dropped
- Step 3 — The backup script that leaves things behind when stopped
- Step 4 — Restarts that cut the worker off mid-job
- Step 5 — Zombies piling up
- Step 6 — A process `kill -9` can't remove
- Step 7 — The nightly backup slowing down the API
- Step 8 — The worker in a container: PID 1 duties
- Step 9 — A suspicious process: investigate before you kill
- Step 10 — Passwords visible in `ps`
- Scenario recap
- Troubleshooting table
- Appendix — Command reference

---

# The setup

Same server (`app-server-01`, Ubuntu 24.04) as Notes 4–8.

| Process | Runs as | What it is |
|---|---|---|
| `node /opt/api/current/server.js` | `api` | The API, on port 80 (Note 8) |
| `python -m worker` | `worker` | Background jobs: processes uploads, sends emails |
| `backup.sh` → `pg_dump` | `worker` | Nightly database backup (Notes 2 and 6) |

People: `thirana` (admin), `ruwan` and `sachini` (ops), `nimal` (dev), `dilini` (auditor).

---

# Step 0 — Look around: the process tree on the server

### The whole family

```bash
pstree -p | head -20
```
```
systemd(1)─┬─cron(812)
           ├─node(2890)─┬─{node}(2891)
           │            ├─{node}(2892)
           │            └─...
           ├─python(3120)
           ├─sshd(905)───sshd(5011)───sshd(5040)───bash(5041)───pstree(5102)
           ├─systemd-journal(301)
           └─...
```

**Meaning:**
- Everything descends from **systemd (PID 1)**. (Why: Note 9, Phase 1.3.)
- `node(2890)` has children in `{braces}` — those are its **threads**, not separate processes. (Why: Note 9, Phase 1.4.)
- Your own shell is at the end of the `sshd` chain: systemd → sshd → your session → bash.

### The services, with useful columns

```bash
ps -eo pid,ppid,user,stat,etime,%cpu,%mem,cmd | grep -E 'node|python -m worker' | grep -v grep
```
```
 2890     1 api      Ssl  2-03:11:04  0.4  2.4 node /opt/api/current/server.js
 3120     1 worker   Ssl  2-03:11:02  0.1  1.1 python -m worker
```

**Meaning:**
- PPID **1** — systemd started them directly.
- `Ssl` — sleeping (waiting for work), session leader, multi-threaded. Sleeping is the normal state of a healthy service. (Why: Note 9, Phase 4.1.)
- `etime` — running for 2 days, 3 hours.

### One process in detail

```bash
PID=2890
grep -E '^(Name|State|PPid|Uid|Threads|VmRSS)' /proc/$PID/status
sudo ls -l /proc/$PID/exe /proc/$PID/cwd
sudo ls -l /proc/$PID/fd | head -5
```
```
Name:    node
State:   S (sleeping)
PPid:    1
Uid:     901 901 901 901
Threads: 11
VmRSS:   98232 kB

/proc/2890/exe -> /usr/bin/node
/proc/2890/cwd -> /opt/api/releases/2026-10-17-2

0 -> /dev/null
1 -> socket:[48211]
2 -> socket:[48211]
3 -> anon_inode:[eventpoll]
...
```

**Meaning:**
- stdin is `/dev/null` and stdout/stderr are a **socket** — the systemd journal. A service has no terminal. (Why: Note 9, Phase 2.3.)
- `cwd` shows which **release** is actually running — useful after a deploy (Note 6, Step 3).

**Conclusion:** you now know how to read the "normal" picture. Every later step compares against it.

---

# Step 1 — "The server is slow": find the process

Ops gets an alert: API responses are slow.

### Live view first

```bash
top
```

Press `P` (sort by CPU) and `c` (full commands):

```
top - 14:02:10 up 3 days, load average: 2.15, 1.40, 0.70
Tasks: 143 total,   2 running, 141 sleeping,   0 stopped,   0 zombie
%Cpu(s): 92.0 us,  3.1 sy,  0.0 ni,  4.5 id,  0.2 wa

  PID USER    PR  NI ... S  %CPU %MEM  COMMAND
 5310 nimal   20   0 ... R  99.7  0.3  python3 scripts/regen_thumbs.py --all
 2890 api     20   0 ... S  45.1  2.6  node /opt/api/current/server.js
```

**Meaning:**
- CPU is ~92% busy (`us` = user code), load rising. (Why: Note 9, Phase 3.4.)
- One process, `regen_thumbs.py` started by **nimal**, is using a full core (`R` = running) and taking CPU from the API.

### Why `ps` would have misled you

```bash
ps -eo pid,user,%cpu,cmd --sort=-%cpu | head -3
```
```
 5310 nimal  99.1 python3 scripts/regen_thumbs.py --all
 2890 api     0.4 node /opt/api/current/server.js
```

**Meaning:** in `ps`, the API shows **0.4%** because `%cpu` is averaged over its **whole 3-day life**. `top` shows **right now** (45%). For "what's busy now", use `top`. (Why: Note 9, Phase 3.1.)

### What exactly is it?

```bash
ps -o pid,ppid,user,etime,cmd -p 5310
sudo ls -l /proc/5310/cwd
```
```
 5310  5288 nimal  00:41:12 python3 scripts/regen_thumbs.py --all
/proc/5310/cwd -> /srv/projects/thumbs-tool
```

**Meaning:** a dev's one-off script, running 41 minutes, from the project folder.

### Decide

Two options, in order of politeness:

```bash
sudo renice -n 19 -p 5310        # lower its priority: API gets the CPU first (Step 7 explains)
# or, after talking to nimal:
sudo kill -TERM 5310             # polite stop
```

After `renice`, `top` shows `NI 19` for it and the API's response time recovers, while the script still runs on spare CPU. (Why: Note 9, Phase 9.1.)

**Conclusion:** `top` to find what's busy **now**, `ps -o` and `/proc` to understand it, then the gentlest fix first.

---

# Step 2 — A long job that died when the SSH session dropped

Ruwan starts a one-off search reindex from his laptop, in an SSH session:

```bash
sudo -u worker /opt/worker/venv/bin/python -m worker.reindex
```

40 minutes later his Wi-Fi drops. When he reconnects, the job is gone and the index is half-built.

### What happened

Test it safely with a harmless command. In an SSH session:

```bash
sleep 3000 &
jobs -l
# [1]+  6201 Running    sleep 3000 &
```

Close that SSH window. From **another** session:

```bash
pgrep -a sleep
# (nothing)
```

**Meaning:** when the SSH connection closed, the shell got `SIGHUP` and passed it to all its jobs. The default action of `SIGHUP` is "terminate". (Why: Note 9, Phase 7.4.)

### Rescue for a job that's already running

If you notice **before** disconnecting:

```bash
# job is running in the foreground → press Ctrl+Z
# [1]+  Stopped   python -m worker.reindex
bg %1          # keep running, in the background
disown %1      # remove from the shell's job list → shell won't SIGHUP it
```

Limit: its output still goes to your terminal. When the terminal closes, writes fail and the program may stop with an error. It's a rescue, not a plan. (Why: Note 9, Phase 7.5.)

### The right ways

**Interactive work you want to come back to → `tmux`:**

```bash
tmux new -s reindex
sudo -u worker /opt/worker/venv/bin/python -m worker.reindex
# Ctrl+b then d   → detach; the job keeps running on the server
# later, even from another computer:
tmux attach -t reindex
```

**A real job → run it as a temporary systemd unit:**

```bash
sudo systemd-run --unit=reindex-2026-10-20 \
     --uid=worker --gid=worker \
     -p EnvironmentFile=/etc/myapps/common.env -p EnvironmentFile=/etc/myapps/worker.env \
     /opt/worker/venv/bin/python -m worker.reindex
```
```
Running as unit: reindex-2026-10-20.service
```

Follow and check it from any session:

```bash
journalctl -u reindex-2026-10-20 -f
systemctl status reindex-2026-10-20
```

**Meaning:** the job is now a child of **systemd**, not of your shell. It has no terminal, so a dropped connection can't touch it. Output goes to the journal, with timestamps, readable later. It gets the same env files as the worker service (Note 2). (Why: Note 9, Phase 7.5; full systemd detail in Notes 11–13.)

Ruwan's sudo rule (Note 8) doesn't include `systemd-run`, so either thirana runs it, or a narrow rule is added for this specific command.

---

# Step 3 — The backup script that leaves things behind when stopped

The backup script (from Notes 2 and 6), simplified:

```bash
#!/bin/bash
set -euo pipefail
umask 027
dest=/var/backups/appdb/db-$(date +%F).sql
tmp="$dest.partial"
pg_dump > "$tmp"
mv "$tmp" "$dest"
```

(Connection settings come from `PG*` environment variables loaded from `worker.env` — Step 10 explains why not a URL argument.)

One night the database is under maintenance, so ops stops the stuck backup:

```bash
pgrep -af backup.sh
# 7010 /bin/bash /var/lib/worker/backup.sh
sudo kill -TERM 7010
```

### What's left behind

```bash
ps -o pid,ppid,stat,etime,cmd -C pg_dump
ls -la /var/backups/appdb/
```
```
  PID  PPID STAT  ELAPSED CMD
 7011     1 S       12:04 pg_dump

-rw-r----- 1 worker ops  1.2G ... db-2026-10-20.sql.partial
```

**Meaning:**
- `kill` hit only the **bash** process. Its child `pg_dump` kept running and was adopted by PID 1 — an **orphan**. (Why: Note 9, Phase 5.2 and 8.5.)
- The half-written `.partial` file is still there, using disk.

### First attempt: add a trap

```bash
trap 'rm -f "$tmp"; exit 143' TERM
pg_dump > "$tmp"
```

Test again: `kill -TERM <script>`. The trap does **nothing for 20 minutes**, then runs.

**Meaning:** bash runs a trap only **after the foreground command finishes**. The trap was waiting for `pg_dump`. (Why: Note 9, Phase 8.5.)

### Fix: background + `wait`, forward the signal, clean up on any exit

```bash
#!/bin/bash
set -euo pipefail
umask 027

dest=/var/backups/appdb/db-$(date +%F).sql
tmp="$dest.partial"
child=""

cleanup() {
  if [ -n "$child" ]; then kill -TERM "$child" 2>/dev/null || true; fi
  rm -f "$tmp"
}
trap cleanup EXIT
trap 'echo "backup: got SIGTERM, stopping" >&2; exit 143' TERM
trap 'exit 130' INT

pg_dump > "$tmp" &
child=$!
wait "$child"          # wait is interrupted by signals immediately
child=""

mv "$tmp" "$dest"
```

How it works:
- `pg_dump` runs in the background; the script waits with `wait`, which returns **as soon as** a signal arrives.
- The `TERM` trap prints a message and calls `exit 143`.
- `exit` triggers the `EXIT` trap: it stops `pg_dump` and removes the `.partial` file.
- On success, `child` is cleared, so `cleanup` only deletes the (already moved) temp name.

### Test

```bash
sudo -u worker bash /var/lib/worker/backup.sh &
sleep 5
sudo kill -TERM $!
wait $!; echo "exit: $?"
pgrep -a pg_dump
ls /var/backups/appdb/*.partial
```
```
backup: got SIGTERM, stopping
exit: 143
(nothing)
ls: cannot access '/var/backups/appdb/*.partial': No such file or directory
```

**Meaning:** stopped within a second, child gone, no leftovers, exit code 143 (= 128 + 15) tells whoever runs it that it was terminated. (Why: Note 9, Phase 4.3.)

---

# Step 4 — Restarts that cut the worker off mid-job

After each deploy, CI restarts the worker (Note 8, Step 2). Users start reporting thumbnails that are missing or half-generated.

### Look at the worker's last log lines before a restart

```bash
journalctl -u worker --since "10 min ago" | tail -5
```
```
14:20:11 worker[3120]: processing upload img-0042.png
14:20:12 systemd[1]: Stopping worker.service...
14:20:12 systemd[1]: worker.service: Deactivated successfully.
14:20:13 systemd[1]: Started worker.service.
14:20:14 worker[7302]: worker started
```

**Meaning:** "processing img-0042" and then **nothing** from the worker — no "done", no "stopping". It died within a second of the stop request.

### Confirm with a manual signal

```bash
WPID=$(pgrep -f 'python -m worker$')
sudo kill -TERM $WPID
journalctl -u worker -n 3
```

Same result. **Meaning:** the worker has **no SIGTERM handler**, so Python uses the default action: **terminate immediately**, in the middle of a job. (Why: Note 9, Phase 6.1 and 8.3.)

### Fix: handle SIGTERM in the worker

```python
import signal, logging

log = logging.getLogger("worker")
stopping = False

def on_term(signum, frame):
    global stopping
    stopping = True
    log.info("SIGTERM received: finishing current job, then exiting")

signal.signal(signal.SIGTERM, on_term)

while not stopping:
    job = queue.get(timeout=5)       # returns every 5 s even with no work, so the flag is checked
    if job:
        process(job)                 # always completes

log.info("worker stopped cleanly")
```

### Test again

```bash
sudo systemctl restart worker
journalctl -u worker -n 5
```
```
14:41:02 worker[7302]: processing upload img-0051.png
14:41:03 worker[7302]: SIGTERM received: finishing current job, then exiting
14:41:05 worker[7302]: done img-0051.png
14:41:05 worker[7302]: worker stopped cleanly
14:41:06 worker[7410]: worker started
```

**Meaning:** the current job finished, then the worker exited on its own. (Why: Note 9, Phase 8.1.)

### The same for the API

Without a handler, Node also exits instantly on `SIGTERM`, dropping in-flight requests (clients see "connection reset" during deploys). Add the `server.close()` handler from Note 9, Phase 8.2, with a safety timeout.

### One limit to remember

systemd waits only a limited time after `SIGTERM` (90 seconds by default) before sending `SIGKILL`. If a job can take longer, the handler isn't enough — that setting is in Note 12.

---

# Step 5 — Zombies piling up

### The signal

```bash
top | head -3
```
```
Tasks: 361 total,   1 running, 142 sleeping,   0 stopped, 218 zombie
```

**Meaning:** 218 zombies — and growing. A few briefly is normal; a growing number is a bug. (Why: Note 9, Phase 5.1.)

### Whose are they?

```bash
ps -eo pid,ppid,stat,etime,cmd | awk '$3 ~ /^Z/' | head -3
```
```
 8810  7410 Z     01:02:11 [convert] <defunct>
 8823  7410 Z     01:01:58 [convert] <defunct>
 8840  7410 Z     01:01:40 [convert] <defunct>
```

```bash
ps -o pid,user,cmd -p 7410
```
```
 7410 worker python -m worker
```

**Meaning:** all zombies are finished `convert` processes (ImageMagick, used for thumbnails), and their **parent is the worker** (7410). The worker starts converters but never collects their exit status.

### Try to kill one

```bash
sudo kill -9 8810
ps -o pid,stat -p 8810
```
```
 8810 Z
```

**Meaning:** you can't kill a zombie — it's already dead. Only its **parent** can clear it, by calling `wait()`. (Why: Note 9, Phase 5.1.)

### Find the cause in the code

```python
# keep a list so we can limit how many run at once
self.running.append(subprocess.Popen(["convert", src, dst]))
```

The code keeps `Popen` objects in a list to count them, but **never calls `.poll()` or `.wait()`** on them. Finished converters stay as zombies.

### Fix

Short term — restart the parent:

```bash
sudo systemctl restart worker
top | head -3      # zombie count drops to 0
```

**Meaning:** when the worker exits, its zombie children are adopted by PID 1, which reaps them immediately. (Why: Note 9, Phase 5.3.)

Real fix — collect children:

```python
# simplest: wait for each conversion (with a time limit)
subprocess.run(["convert", src, dst], check=True, timeout=120)

# or, if running several in parallel, reap finished ones regularly:
self.running = [p for p in self.running if p.poll() is None]
```

### Why it mattered

Each zombie holds a PID. With enough of them, the worker hits its **task limit** (Note 13) or the system's PID limit, and then **nothing new can start** — not even the converters it needs.

---

# Step 6 — A process `kill -9` can't remove

The worker also generates thumbnails from a shared NFS folder (`/mnt/shared`, a Filestore share). One afternoon, thumbnail jobs stop. Ops tries to restart the worker, and `systemctl stop worker` hangs.

### Look at the state

```bash
ps -eo pid,stat,etime,wchan:28,cmd | awk '$2 ~ /^D/'
```
```
 9120 D    00:18:30 folio_wait_bit_common        python -m worker.thumbs
 9133 D    00:12:02 folio_wait_bit_common        ls /mnt/shared
```

**Meaning:**
- `D` = **uninterruptible sleep**: waiting inside the kernel for storage. (Why: Note 9, Phase 10.1.)
- Even ruwan's `ls` of the folder is stuck the same way.

### Try SIGKILL

```bash
sudo kill -9 9120
sleep 2; ps -o pid,stat -p 9120
```
```
 9120 D
```

**Meaning:** the signal can't be delivered until the process returns from the kernel, and the read it's waiting on never finishes. Killing is the wrong tool here. (Why: Note 9, Phase 10.2.)

### Find the layer underneath

```bash
findmnt -T /mnt/shared
sudo dmesg -T | tail -3
```
```
TARGET      SOURCE              FSTYPE OPTIONS
/mnt/shared 10.0.0.9:/share     nfs4   rw,relatime,hard,...

[Tue Oct 20 15:02:14 2026] nfs: server 10.0.0.9 not responding, still trying
```

**Meaning:** the NFS server isn't answering. With a `hard` mount, the kernel retries forever, so every process reading the share waits in `D`. A firewall rule change earlier that day had blocked the NFS port.

### Fix

Restore the connectivity (revert the firewall rule). Within seconds:

```bash
sudo dmesg -T | tail -1
# nfs: server 10.0.0.9 OK
ps -o pid,stat -p 9120
# (gone — the pending SIGKILL was delivered once the read returned)
```

**Conclusion:** a process stuck in `D` points you to **storage or network**, not to the process. Note 9, Phase 10 has the background; the network-debugging area will cover tracing the blocked port.

---

# Step 7 — The nightly backup slowing down the API

The API's response time spikes every night around 02:00.

### Watch during the backup

```bash
top
```
```
%Cpu(s): 48.5 us,  4.0 sy,  0.0 ni, 22.1 id, 25.4 wa

  PID USER    PR  NI  S  %CPU  COMMAND
10210 worker  20   0  R  98.0  gzip -9
10209 worker  20   0  S  12.3  pg_dump
 2890 api     20   0  S  30.1  node /opt/api/current/server.js
```

**Meaning:**
- The backup now pipes `pg_dump | gzip -9`. `gzip -9` (maximum compression) uses a whole core.
- `wa` 25% — a lot of time waiting for **disk**, from writing the backup file.
- On this 2-CPU server, the API competes with both. (Why: Note 9, Phase 3.4.)

### Lower the backup's priority right now

```bash
sudo renice -n 10 -p 10209 10210
ps -o pid,ni,cmd -p 10209,10210
```
```
  PID  NI CMD
10209  10 pg_dump
10210  10 gzip -9
```

**Meaning:** when the CPU is busy, the scheduler now prefers the API (NI 0) over the backup (NI 10). The backup still uses CPU nobody else wants. (Why: Note 9, Phase 9.1.)

### Make it permanent in the script

```bash
pg_dump | ionice -c 3 nice -n 10 gzip -6 > "$tmp" &
```

Also start the whole script with lower priority from wherever it's scheduled:

```bash
ionice -c 3 nice -n 10 /var/lib/worker/backup.sh
```

- `nice -n 10` — lower CPU priority. The `worker` user can do this without sudo (making itself **nicer** is always allowed).
- `ionice -c 3` — disk access only when nobody else needs it.
- `gzip -6` — the default level: much less CPU, slightly bigger file.

**Next night:** the API's latency stays flat; the backup takes a few minutes longer.

**Limit:** `nice` and `ionice` are **hints**. For a guaranteed cap (e.g. "never more than half a core"), use cgroup limits — Note 13.

---

# Step 8 — The worker in a container: PID 1 duties

The team tests running the worker in Docker (Note 6, Step 10).

```bash
docker run -d --name worker --user 902:902 --group-add 2003 myorg/worker
docker exec worker ps -o pid,ppid,stat,cmd
```
```
PID PPID STAT CMD
  1    0 Ssl  python -m worker
 25    1 Z    [convert] <defunct>
...
```

**Meaning:** inside the container, the **worker itself is PID 1**. PID 1 has two extra jobs (Note 9, Phase 5.3):
1. **Reap orphans.** If a converter starts its own helper and exits, that helper is re-parented to PID 1 — the worker — which doesn't expect it and never reaps it.
2. **Signals.** PID 1 only receives signals it has handlers for. The Step 4 handler covers `SIGTERM`, but nothing else.

### Fix: a tiny init as PID 1

```bash
docker run -d --init --name worker --user 902:902 --group-add 2003 myorg/worker
docker exec worker ps -o pid,ppid,cmd
```
```
PID PPID CMD
  1    0 /sbin/docker-init -- python -m worker
  7    1 python -m worker
```

**Meaning:** `--init` puts `tini` at PID 1. It forwards signals to the worker and reaps any orphan. The worker goes back to being a normal process. In Compose: `init: true`.

---

# Step 9 — A suspicious process: investigate before you kill

During the morning check, `top` shows:

```
  PID USER  PR  NI  S  %CPU  COMMAND
11402 api   20   0  R  98.6  [kworker/1:3]
```

A kernel worker thread, owned by `api`, using a full core. That doesn't look right.

### Check 1 — is it really a kernel thread?

```bash
ps -o pid,ppid,user,stat,lstart,cmd -p 11402
```
```
  PID  PPID USER  STAT  STARTED                  CMD
11402     1 api   R     Tue Oct 20 03:12:44 2026 [kworker/1:3]
```

Compare with a real one:

```bash
ps -o pid,ppid,user,cmd -p $(pgrep -x 'kworker/0:1' | head -1)
```
```
   19     2 root  [kworker/0:1]
```

**Meaning:** real kernel threads are children of **PID 2 (kthreadd)** and owned by **root**. This one has PPID 1 and user `api`. The name in brackets is a **disguise** — a process can set its own name to anything. (Why: Note 9, "How this goes wrong".)

### Check 2 — what's actually running?

```bash
sudo ls -l /proc/11402/exe /proc/11402/cwd
```
```
/proc/11402/exe -> /dev/shm/.k (deleted)
/proc/11402/cwd -> /dev/shm
```

**Meaning:** the real program ran from `/dev/shm` (memory-backed, often writable) and its file was **deleted** after starting — both common malware signs. Real kernel threads have **no** `exe` link at all.

### Check 3 — is it talking to anything?

```bash
sudo ss -tnp | grep 11402
```
```
ESTAB 0 0 10.0.0.4:51022 203.0.113.50:3333 users:(("[kworker/1:3]",pid=11402,fd=7))
```

**Meaning:** a connection to an outside address on port 3333 — typical of a crypto-mining pool. Combined with 100% CPU: very likely a miner, started through a weakness in the API.

### Respond in the right order

**1. Freeze it, don't kill it yet.** Killing destroys the evidence in `/proc`.

```bash
sudo kill -STOP 11402          # pauses it: no CPU, no network activity, evidence intact
```

**2. Collect evidence** (into a root-only folder):

```bash
sudo install -d -m 700 /root/ir/11402
cd /root/ir/11402
sudo cp /proc/11402/exe ./program.bin                   # works even though the file was deleted
sudo sh -c 'tr "\0" " " < /proc/11402/cmdline > cmdline.txt'
sudo sh -c 'tr "\0" "\n" < /proc/11402/environ > environ.txt'
sudo ls -l /proc/11402/fd > fds.txt
sudo cat /proc/11402/maps > maps.txt
sudo ss -tnp | grep 11402 > net.txt
ps -o pid,ppid,user,lstart,cmd -p 11402 > ps.txt
sha256sum program.bin > program.sha256
```

**3. Stop it.**

```bash
sudo kill -9 11402
```

**4. Find how it got in and close it:**
- API logs around **03:12** (the start time from `lstart`): `journalctl -u api --since "03:00" --until "03:20"`.
- `environ.txt` shows it ran with the **API's environment** — so assume the API's secrets (`api.env`, Note 2) are exposed and **rotate them**.
- Check the API's systemd hardening (Note 8, Step 5) — e.g. whether the service could write and execute in `/dev/shm`. Note 12 adds settings that block running programs from writable paths.
- Look for persistence: unexpected cron entries, systemd units, SSH keys (Notes 4, 7, 8 audits).

**Conclusion:** name, owner, parent, `exe`, `cwd`, and network connections together tell you what a process really is. **Freeze → collect → kill → investigate**, never kill first.

---

# Step 10 — Passwords visible in `ps`

While reviewing processes, the auditor (`dilini`, no special access) runs:

```bash
ps -eo user,cmd | grep pg_dump
```
```
worker  pg_dump postgres://backup_ro:BkPass@10.0.0.5:5432/appdb
```

**Meaning:** an earlier version of the backup script passed the database URL **as an argument**. Every user on the server can read every process's command line. (Why: Note 2, Phase 6.1; Note 9, "How this goes wrong".)

### Fix 1 — keep secrets out of arguments

PostgreSQL tools read connection settings from environment variables. Put them in `worker.env` (Note 6: `640 root:worker`):

```
PGHOST=10.0.0.5
PGPORT=5432
PGDATABASE=appdb
PGUSER=backup_ro
PGPASSWORD=BkPass
```

The script then runs plain `pg_dump`, as in Step 3. Now:

```bash
ps -eo user,cmd | grep pg_dump
# worker  pg_dump
```

The password is in the process's **environment**, which only `worker` and root can read (`/proc/PID/environ`).

### Fix 2 — hide other users' processes

Even without secrets, seeing every process (names, arguments, users) helps an attacker. Mount `/proc` so users only see **their own** processes:

```bash
sudo mount -o remount,hidepid=invisible /proc
```

As dilini:

```bash
ps aux | wc -l       # only her own processes now
```

As root, `ps aux` still shows everything, so admin tools and the Note 8 audit script keep working.

To make it permanent, add to `/etc/fstab`:

```
proc  /proc  proc  defaults,hidepid=invisible  0  0
```

Test after enabling: a few tools expect to see all processes. If a monitoring agent runs as its own user and needs the full view, the `gid=` mount option lets one group see everything.

For a single service, systemd can do the same per unit with `ProtectProc=invisible` (Note 12).

**Conclusion:** never put secrets in command lines; and limit who can see processes at all.

---

# Scenario recap

| Step | What happened | Concept (Note 9) |
|---|---|---|
| 0 | Read the normal process tree and one service in detail | PID/PPID, tree, threads, `/proc`, fds (1, 2, 3) |
| 1 | Slow server traced to a dev script | `top` vs `ps` CPU, `renice` (3, 9) |
| 2 | Long job died with the SSH session | SIGHUP on logout; `tmux`, `systemd-run` (7) |
| 3 | Killed backup left an orphan and a partial file | Orphans; trap timing; background + `wait`; exit 143 (4, 5, 8) |
| 4 | Worker died mid-job on restart | Default SIGTERM action; graceful handler (6, 8) |
| 5 | Zombies growing | Parent must reap; can't kill a zombie (5) |
| 6 | Process immune to `kill -9` | `D` state; fix the storage (10) |
| 7 | Backup starving the API | `nice`, `ionice` (9) |
| 8 | Worker as PID 1 in a container | PID 1 duties; `--init` (5) |
| 9 | Disguised miner | Fake names, deleted exe, freeze → collect → kill (security) |
| 10 | Password visible in `ps` | Secrets in env not args; `hidepid` (security) |

---

# Troubleshooting table

| Symptom | Likely cause | Check | Fix |
|---|---|---|---|
| Server slow, unsure why | One process hogging CPU | `top`, sort with `P` | `renice`, or stop it politely |
| `ps` shows low CPU for a busy process | `ps %cpu` is a lifetime average | Use `top` | — |
| Job died after closing SSH | SIGHUP to the shell's jobs | Was it started in a plain SSH shell? | `tmux`, `systemd-run`, or `nohup` |
| Killed a script, but its work continues | Child orphaned to PID 1 | `ps -o pid,ppid,cmd -C <child>` | Forward the signal; kill the process group |
| Trap runs much later than the signal | Trap waits for foreground command | Is the long command in the foreground? | Run in background + `wait` |
| Service loses work on restart | No SIGTERM handler | Last log lines before "Stopping" | Add a graceful shutdown handler |
| Growing zombie count | Parent never calls `wait()` | `ps ... \| awk '$3 ~ /^Z/'` → PPID | Fix the parent; restart it short-term |
| `kill -9` has no effect | Process in `D` state | `ps -o stat,wchan`, `dmesg`, `findmnt` | Fix the storage/network underneath |
| `kill: Operation not permitted` | Not your process | `ps -o user -p PID` | Ask the owner, or use an allowed sudo rule |
| Container ignores `docker stop` | App is PID 1 without a handler | `docker exec ... ps` | Handler + `--init` |
| Zombies inside a container | PID 1 doesn't reap orphans | `docker exec ... ps` | `--init` / `init: true` |
| Process named like a kernel thread, but odd | Disguised malware | PPID, user, `/proc/PID/exe` | Freeze, collect, kill, investigate |
| Secrets readable via `ps` | Passed as arguments | `ps -eo user,cmd` | Env vars / config files; `hidepid` |
| A batch job (e.g. backup) slows other services | Competing for CPU/disk | `top` `wa`, NI column | `nice`, `ionice`; cgroups (Note 13) |

---

# Appendix — Command reference

## Find and inspect

| Command | Purpose |
|---|---|
| `pstree -p` | Whole process tree |
| `ps -eo pid,ppid,user,stat,etime,%cpu,%mem,cmd` | Main columns |
| `ps -o pid,ppid,user,lstart,cmd -p PID` | Exact start time |
| `top` → `P`, `M`, `c`, `H`, `k`, `r` | Live view |
| `pgrep -af pattern` | Find by command line |
| `grep -E '^(Name\|State\|PPid\|Uid\|Threads\|VmRSS)' /proc/PID/status` | Summary |
| `sudo ls -l /proc/PID/exe /proc/PID/cwd /proc/PID/fd` | Real program, directory, open files |
| `sudo ss -tnp \| grep PID` | Network connections of a process |
| `ps -eo pid,stat,wchan:28,cmd \| awk '$2 ~ /^D/'` | Stuck processes |
| `ps -eo pid,ppid,stat,cmd \| awk '$3 ~ /^Z/'` | Zombies and their parents |

## Control

| Command | Purpose |
|---|---|
| `kill -TERM PID` → wait → `kill -9 PID` | Correct stop order |
| `kill -STOP PID` / `kill -CONT PID` | Freeze / resume |
| `kill -TERM -- -PGID` | Stop a whole process group |
| `renice -n 10 -p PID` | Lower priority of a running process |
| `ionice -c 3 nice -n 10 cmd` | Start with low CPU and disk priority |

## Long jobs

| Command | Purpose |
|---|---|
| `Ctrl+Z`, `bg %1`, `disown %1` | Rescue a running foreground job |
| `tmux new -s name` / `Ctrl+b d` / `tmux attach -t name` | Persistent interactive session |
| `sudo systemd-run --unit=NAME --uid=U --gid=G -p EnvironmentFile=... cmd` | Run as a temporary unit |
| `journalctl -u NAME -f` / `systemctl status NAME` | Follow / check it |

## Scripts

| Pattern | Purpose |
|---|---|
| `trap cleanup EXIT` | Always clean up |
| `trap 'exit 143' TERM` | Exit on SIGTERM (triggers EXIT trap) |
| `cmd & child=$!; wait "$child"` | Signals handled immediately |
| `kill -TERM "$child"` in cleanup | Don't leave orphans |

## Investigation (security)

| Command | Purpose |
|---|---|
| `kill -STOP PID` | Freeze, keep evidence |
| `sudo cp /proc/PID/exe file` | Copy the program, even if deleted |
| `/proc/PID/cmdline`, `environ`, `maps`, `fd/` | Capture state |
| `ls -l /proc/*/exe 2>/dev/null \| grep deleted` | Find processes with deleted programs |
| `sudo ss -tulpn` / `sudo ss -tnp` | Listeners / connections |

## Containers and `/proc`

| Command | Purpose |
|---|---|
| `docker run --init ...` / Compose `init: true` | tini as PID 1 |
| `docker exec CONTAINER ps -o pid,ppid,stat,cmd` | Processes inside |
| `mount -o remount,hidepid=invisible /proc` | Users see only their own processes |
| `proc /proc proc defaults,hidepid=invisible 0 0` | fstab line |