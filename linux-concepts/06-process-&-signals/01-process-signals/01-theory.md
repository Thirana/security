# Processes & Signals — Theory (Note 9)

What a running program is, how to see and inspect it, how processes start, wait, and end, and how you (and the system) talk to them with signals.

**Series:**
- Note 1: Shell & Processes
- Note 2: Environment Variables
- Note 3 / 4: Users, Groups & Identity
- Note 5 / 6: Files, Directories & Permissions
- Note 7 / 8: Privilege & Delegation
- **Note 9: Processes & Signals — Theory** ← this note
- Note 10: Processes & Signals — Practice
- Note 11 / 12: systemd & Services — Theory / Practice
- Note 13 / 14: Logs, Timers & Resource Limits — Theory / Practice

Note 1 explained how the shell **creates** a process (fork + exec). This note is about everything after that: what a process holds, the states it goes through, how it ends, and how to control it.

Each phase ends with **"Used in"** (Note 10 steps) and, where relevant, **"How this goes wrong"** for the security track.

---

## Contents

1. Phase 1 — What a process is
2. Phase 2 — What a process carries: a tour of `/proc`
3. Phase 3 — Seeing processes: `ps`, `pgrep`, `pstree`, `top`
4. Phase 4 — Process states and how a process ends
5. Phase 5 — Zombies, orphans, and PID 1
6. Phase 6 — Signals
7. Phase 7 — Terminals, sessions, and jobs
8. Phase 8 — Handling signals: graceful shutdown
9. Phase 9 — Priority: `nice` and `ionice`
10. Phase 10 — Stuck processes: the `D` state
11. How this goes wrong — overview
12. Summary table
13. Appendix — Command reference

---

# Phase 1 — What a process is

## 1.1 Program vs process

| | Program | Process |
|---|---|---|
| What | A **file** on disk (`/usr/bin/node`) | A **running instance** of a program, in memory |
| How many | One file | Many processes can run the same program at once |
| Lasts | Until deleted | Until it exits or is killed |

Running `node server.js` twice gives **two processes** from **one program**.

## 1.2 PID and PPID

Every process has:

| ID | Meaning |
|---|---|
| **PID** | Process ID — a unique number while it runs |
| **PPID** | Parent process ID — the PID of the process that created it |

```bash
echo $$          # PID of your current shell
echo $PPID       # PID of its parent (e.g. sshd)
```

PIDs are reused after a process ends. The highest possible PID is in `/proc/sys/kernel/pid_max`.

## 1.3 The process tree

Because every process has a parent (Note 1: fork + exec), all processes form a **tree**.

```
systemd (PID 1)
├── sshd
│   └── sshd: thirana
│       └── bash (your shell)
│           └── ps
├── node /opt/api/current/server.js   (the api service)
└── python -m worker                  (the worker service)
```

```bash
pstree -p          # the tree with PIDs
```

Two special PIDs:

| PID | Process | Role |
|---|---|---|
| **1** | `systemd` (the **init** process) | The first user-space process. Starts everything else, adopts orphans (Phase 5) |
| **2** | `kthreadd` | Parent of all **kernel threads** (names in `[brackets]`, e.g. `[kworker/0:1]`) |

## 1.4 Processes and threads

A process can have several **threads** — parallel paths of execution that **share** the process's memory and open files. Node has one main JavaScript thread plus helper threads; a Java server can have hundreds.

```bash
ps -L -p <PID>        # one line per thread
ls /proc/<PID>/task   # one folder per thread
```

For most of this note, "process" is what matters. Threads come up again with CPU usage (Phase 3) and resource limits (Note 13).

**Used in:** Note 10, Steps 0, 1.

---

# Phase 2 — What a process carries: a tour of `/proc`

## 2.1 What a process holds

Earlier notes each covered one part of a process. Together:

| Part | Explained in |
|---|---|
| Program code and memory | This note |
| Identity: UIDs, GIDs, groups | Note 3, Note 7 |
| Environment variables | Note 2 |
| Current directory | Note 1 |
| Open files (including stdin/stdout/stderr) | This note |
| umask | Note 5 |
| Capabilities | Note 7 |
| Resource limits, cgroup | Note 13 |
| Priority (nice value) | Phase 9 |
| State | Phase 4 |

## 2.2 `/proc/<PID>` — the process as files

`/proc` isn't on disk. The kernel shows each running process as a folder, `/proc/<PID>/`, and you read it like files.

| Path | Shows |
|---|---|
| `/proc/PID/status` | Name, state, PPID, UIDs/GIDs, groups, memory, capabilities, threads (easy to read) |
| `/proc/PID/cmdline` | The full command line (parts separated by `\0`) |
| `/proc/PID/environ` | Environment at start (Note 2) |
| `/proc/PID/exe` | Link to the **actual program file** being run |
| `/proc/PID/cwd` | Link to its current directory |
| `/proc/PID/fd/` | One link per **open file**, socket, or pipe |
| `/proc/PID/limits` | Resource limits (Note 13) |
| `/proc/PID/task/` | Its threads |

```bash
PID=$(pgrep -f 'node /opt/api')
tr '\0' ' ' < /proc/$PID/cmdline; echo      # node /opt/api/current/server.js
sudo ls -l /proc/$PID/exe                   # -> /usr/bin/node
sudo ls -l /proc/$PID/cwd                   # -> /opt/api/releases/2026-10-17-2
sudo ls -l /proc/$PID/fd | head             # 0 -> /dev/null, 1 -> socket:[...], ...
grep -E '^(Name|State|PPid|Uid|Threads|VmRSS)' /proc/$PID/status
```

Reading another user's `/proc/PID/environ`, `exe`, `cwd`, or `fd` needs **the same user or root** (Note 2, Phase 6.1).

## 2.3 File descriptors 0, 1, 2

Every process starts with three open files:

| fd | Name | Usually |
|---|---|---|
| 0 | stdin | Keyboard, or `/dev/null` for services |
| 1 | stdout | Terminal, a file, a pipe, or the journal (for services) |
| 2 | stderr | Same choices as stdout |

```bash
ls -l /proc/$$/fd       # your shell: 0, 1, 2 -> /dev/pts/0 (your terminal)
```

This explains a lot of later behaviour: when a terminal closes, fds 0–2 point to something that's gone (Phase 7).

**Used in:** Note 10, Steps 0, 1, 9.

---

# Phase 3 — Seeing processes: `ps`, `pgrep`, `pstree`, `top`

## 3.1 `ps` — a snapshot

`ps` prints the processes at one moment. Two common styles:

```bash
ps aux        # BSD style: every process, with user, CPU, memory
ps -ef        # UNIX style: every process, with PPID
```

The most useful form is **choosing your own columns**:

```bash
ps -eo pid,ppid,user,stat,etime,%cpu,%mem,rss,cmd --sort=-%cpu | head
```

| Column | Meaning |
|---|---|
| `pid` / `ppid` | Process / parent ID |
| `user` | Effective user |
| `stat` | State (Phase 4) plus flags |
| `etime` | Time since it started |
| `%cpu` | CPU use, **averaged over its whole life** (not "right now") |
| `%mem` / `rss` | Share of RAM / actual RAM in use (KB) |
| `cmd` | Full command line |

Other handy forms:

```bash
ps -fp 2890                 # one process, full format
ps -u api                   # processes of one user
ps -f --ppid 2890           # children of a process
ps -o pid,ni,pri,cmd -p 2890   # priority
```

## 3.2 Finding PIDs

```bash
pgrep -a node               # PIDs + command lines matching "node"
pgrep -u worker -a python   # only user worker's
pgrep -f 'server.js'        # match the full command line, not just the name
pidof node                  # PIDs of an exact program name
```

`pgrep` is safer in scripts than `ps | grep` (which also matches the `grep` itself).

## 3.3 `pstree` — the family view

```bash
pstree -p           # whole tree with PIDs
pstree -p 2890      # one subtree
pstree -s -p 2890   # its ancestors up to PID 1
```

## 3.4 `top` / `htop` — live view

```bash
top
```

```
top - 10:05:12 up 3 days,  2:11,  2 users,  load average: 0.42, 0.35, 0.30
Tasks: 141 total,   1 running, 140 sleeping,   0 stopped,   0 zombie
%Cpu(s):  3.1 us,  0.8 sy,  0.0 ni, 95.9 id,  0.1 wa, ...
MiB Mem :   3923.4 total,    812.0 free,   1430.2 used,   1681.2 buff/cache

  PID USER   PR  NI    VIRT    RES   SHR S  %CPU  %MEM     TIME+ COMMAND
 2890 api    20   0 1103420  98232 38120 S   2.3   2.4   1:12.33 node
```

| Part | Meaning |
|---|---|
| `load average` | Average number of processes running or waiting, over 1/5/15 min (full meaning in the resources area later) |
| `Tasks` | Count by state — watch the **zombie** number |
| `%Cpu`: `us` / `sy` / `ni` / `id` / `wa` | User code / kernel / niced processes / idle / **waiting for disk** |
| `PR` / `NI` | Kernel priority / nice value (Phase 9) |
| `RES` | RAM actually used |
| `S` | State |
| `%CPU` | CPU use **right now** (unlike `ps`) |

Useful keys in `top`:

| Key | Action |
|---|---|
| `P` / `M` | Sort by CPU / memory |
| `c` | Show full command lines |
| `1` | Show each CPU core |
| `H` | Show threads |
| `u` | Filter by user |
| `k` | Send a signal to a PID |
| `r` | Renice a PID |
| `q` | Quit |

`htop` is a friendlier version (colour, scrolling, tree view with `F5`). Install with `sudo apt install htop`.

**Used in:** Note 10, Steps 0, 1, 7.

---

# Phase 4 — Process states and how a process ends

## 4.1 The states

The `S` / `stat` column shows one letter:

| Letter | State | Meaning |
|---|---|---|
| `R` | Running | Using a CPU, or ready and waiting for one |
| `S` | Sleeping | Waiting for something (a network request, a timer, input). **Most processes, most of the time** |
| `D` | Uninterruptible sleep | Waiting inside the kernel, usually for disk or network storage. Can't be interrupted (Phase 10) |
| `T` | Stopped | Paused by a signal (`Ctrl+Z`, `SIGSTOP`) |
| `t` | Traced | Paused by a debugger |
| `Z` | Zombie | Finished, but parent hasn't collected its exit status (Phase 5) |
| `I` | Idle | An idle kernel thread |

Extra characters after the letter in `ps`:

| Char | Meaning |
|---|---|
| `s` | Session leader (e.g. a login shell) |
| `+` | In the foreground of a terminal |
| `l` | Has several threads |
| `<` / `N` | High / low priority |

Example: `Ssl` = sleeping, session leader, multi-threaded. `R+` = running in the foreground.

## 4.2 The lifecycle

```
         fork + exec (Note 1)
              │
              ▼
        ┌──► R (running) ◄──────┐
        │     │                 │
        │     ├── waits ──► S / D ── event done ──┘
        │     │
        │     ├── SIGSTOP / Ctrl+Z ──► T ── SIGCONT ──┘
        │     │
        │     └── exits (returns, exit(), or killed by a signal)
        │                 │
        │                 ▼
        │            Z (zombie: exit status kept for the parent)
        │                 │
        │           parent calls wait()
        │                 │
        │                 ▼
        │              gone (PID free)
```

## 4.3 Exit codes

When a process ends, it leaves an **exit code** (0–255) for its parent:

| Code | Meaning |
|---|---|
| `0` | Success |
| `1`–`125` | Failure (meaning depends on the program) |
| `126` | Found, but not executable (Note 5) |
| `127` | Command not found |
| `128 + N` | **Killed by signal N** (shell convention) |

```bash
sleep 100 &
kill -TERM $!
wait $!; echo $?      # 143  = 128 + 15 (SIGTERM)
```

Common ones you'll see in logs: **130** (Ctrl+C, SIGINT), **137** (SIGKILL — often the OOM killer, Note 13), **143** (SIGTERM).

## 4.4 Waiting for a child

A parent **collects** a child's exit code with the `wait()` system call. In bash:

```bash
long_task &        # start in background; $! = its PID
pid=$!
...
wait $pid          # pause until it ends
echo "exit code: $?"
```

**Used in:** Note 10, Steps 3, 5, 6.

---

# Phase 5 — Zombies, orphans, and PID 1

## 5.1 Zombies

A **zombie** is a process that has **already finished**, but whose parent hasn't called `wait()` yet. The kernel keeps a tiny entry (PID + exit code) so the parent can collect it.

```bash
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /Z/'
```
```
 4410  3120 Z    [convert] <defunct>
```

Facts:
- A zombie uses **no CPU and no memory**, only a **PID slot**.
- You **can't kill a zombie** — it's already dead. `kill -9` does nothing.
- A few zombies briefly are normal. **Growing** numbers mean the **parent has a bug**: it starts children and never waits for them.
- Too many can exhaust PIDs (`pid_max`) or the service's task limit (Note 13), and then nothing new can start.

**Fix:** make the parent wait for its children (a code fix). Short-term: restart the **parent** — its zombies get adopted by PID 1, which reaps them immediately.

## 5.2 Orphans

An **orphan** is a process whose **parent died first**. It keeps running. The kernel gives it a new parent:

- PID 1 (systemd), or
- the nearest **subreaper** — a process that asked to adopt orphans in its subtree (systemd's per-user manager does this).

```bash
ps -o pid,ppid,cmd -p <PID>       # PPID 1 = adopted
```

Orphans are often **unintended leftovers**: you killed a script, but the long-running command it started is still going (Note 10, Step 3).

## 5.3 Why PID 1 is special

| Duty | Why it matters |
|---|---|
| **Adopts orphans and reaps them** | Otherwise every orphan would become a permanent zombie when it ended |
| **Ignores signals it hasn't asked to handle** | The kernel protects PID 1: a signal with a default "terminate" action is **not delivered** unless PID 1 has a handler. Even `SIGKILL` from inside its own container is ignored |
| **If it dies, the system (or container) ends** | On a host: kernel panic. In a container: the container stops |

This is why containers need care (Note 1, Step 7):
- If your app is PID 1 in a container and has no `SIGTERM` handler, `docker stop` is ignored until the 10-second SIGKILL.
- If your app is PID 1 and starts child processes, **it must reap zombies** — a job normal apps never expect to do.
- Fix both with a tiny init: `docker run --init` (uses `tini`) or `ENTRYPOINT ["tini", "--", ...]`.

**Used in:** Note 10, Steps 3, 5, 8.

---

# Phase 6 — Signals

## 6.1 What a signal is

A **signal** is a small numbered message sent to a process to tell it something happened: "please stop", "stop now", "the terminal closed", "pause".

The process can react in one of three ways:

| Reaction | Meaning |
|---|---|
| **Default action** | What the kernel does if the program didn't say otherwise (terminate, stop, ignore, or terminate + core dump) |
| **Catch** | Run the program's own handler function (e.g. clean up, then exit) |
| **Ignore** | Do nothing |

Two signals **can't be caught or ignored**: `SIGKILL` and `SIGSTOP`. The kernel handles them directly.

## 6.2 Signals you need to know

| Signal | No. | Default | Sent by / used for |
|---|---|---|---|
| `SIGHUP` | 1 | Terminate | Terminal/SSH closed; by convention, "**reload config**" for daemons (e.g. nginx) |
| `SIGINT` | 2 | Terminate | **Ctrl+C** in a terminal |
| `SIGQUIT` | 3 | Terminate + core dump | **Ctrl+\\** |
| `SIGKILL` | 9 | Terminate | "Die **now**" — can't be caught, no cleanup |
| `SIGUSR1` / `SIGUSR2` | 10 / 12 | Terminate | Program-defined (e.g. reopen log files) |
| `SIGSEGV` | 11 | Terminate + core | Program bug: bad memory access |
| `SIGPIPE` | 13 | Terminate | Writing to a pipe/socket nobody reads (`cmd \| head`) |
| `SIGTERM` | 15 | Terminate | "**Please stop**" — the normal, polite stop (`kill`, `systemctl stop`, `docker stop`) |
| `SIGCHLD` | 17 | Ignore | Sent to a parent when a child ends |
| `SIGCONT` | 18 | Continue | Resume a stopped process |
| `SIGSTOP` | 19 | Stop | Pause — can't be caught |
| `SIGTSTP` | 20 | Stop | **Ctrl+Z** (can be caught) |

(Numbers are for Linux on x86/ARM. Use names in commands — they're the same everywhere.)

```bash
kill -l          # list all signals
```

## 6.3 SIGTERM vs SIGKILL — the most important pair

| | `SIGTERM` (15) | `SIGKILL` (9) |
|---|---|---|
| Meaning | "Please stop" | "Stop now" |
| Program can react | Yes: finish work, save, close connections, delete temp files | **No** — killed by the kernel instantly |
| Result | Clean shutdown (if the program handles it) | Half-written files, lost in-progress work, leftover temp files and locks |
| Use | **Always first** | Only if SIGTERM didn't work after a reasonable wait |

The right sequence, which systemd and Docker both follow: **SIGTERM → wait → SIGKILL** (Note 11 shows systemd's version).

## 6.4 Sending signals

```bash
kill 2890               # SIGTERM (the default)
kill -TERM 2890         # same, explicit
kill -9 2890            # SIGKILL
kill -HUP 2890          # reload
kill -STOP 2890 / kill -CONT 2890   # pause / resume

pkill -TERM -u worker python    # by name and user
pkill -f 'server.js'            # by full command line
killall node                    # every process named exactly "node" (careful)

kill -0 2890 && echo alive      # signal 0: sends nothing, just checks it exists and you're allowed
```

Sending to a **process group** (Phase 7) — the leading `-` means "group":

```bash
kill -TERM -- -3120     # every process in group 3120
```

## 6.5 Who can signal whom

- You can signal **your own** processes.
- **Root** (or `CAP_KILL`, Note 7) can signal any process.
- Otherwise: `kill: (2890) - Operation not permitted`.

So a developer can't stop the `api` service by hand — they'd need a sudo rule (Note 8).

**Used in:** Note 10, Steps 2, 3, 4, 9.

---

# Phase 7 — Terminals, sessions, and jobs

## 7.1 Sessions and process groups

When you log in, your shell starts a **session**. Every command line you run becomes a **process group** (a pipeline like `cat f | grep x` is one group with two processes).

- One group at a time is in the **foreground** — it gets your keyboard input and keyboard signals.
- Others are in the **background**.

```bash
ps -o pid,pgid,sid,stat,cmd --forest -t $(tty)
```

## 7.2 Keyboard signals

| Keys | Signal | Sent to |
|---|---|---|
| `Ctrl+C` | `SIGINT` | Every process in the **foreground group** |
| `Ctrl+\` | `SIGQUIT` | Foreground group |
| `Ctrl+Z` | `SIGTSTP` | Foreground group (pauses it) |

## 7.3 Job control

The shell calls each process group a **job**.

```bash
./reindex.sh &          # start in the background → [1] 5120
jobs -l                 # list jobs: [1]+ 5120 Running ./reindex.sh &
fg %1                   # bring job 1 to the foreground
# Ctrl+Z                → [1]+ Stopped ./reindex.sh
bg %1                   # continue it in the background
kill %1                 # SIGTERM to job 1
```

A background job that tries to **read** from the terminal gets stopped automatically.

## 7.4 What happens when the terminal closes

When your SSH connection drops or you close the window:

1. The terminal sends `SIGHUP` ("hang up") to your shell.
2. Bash then sends `SIGHUP` to **all its jobs**, running and stopped.
3. Their default action: **terminate**.

So a long job started in an SSH session **dies when your laptop sleeps or the Wi-Fi drops**. Even if it ignored SIGHUP, its stdout/stderr point at a terminal that no longer exists, and the next write can kill it with `SIGPIPE` / an error.

## 7.5 Keeping a job alive after logout

| Method | How | Good for | Downside |
|---|---|---|---|
| `nohup cmd &` | Ignore SIGHUP; output to `nohup.out` | Quick one-offs | No easy way to come back to it |
| `cmd &` then `disown` | Remove the job from the shell's list, so the shell won't SIGHUP it | Forgot to use nohup | Output still goes to the dead terminal — redirect it |
| `setsid cmd` | Start in a brand-new session, no terminal | Detached background work | Same |
| **`tmux` / `screen`** | Run inside a session that stays on the server; detach and reattach later | **Interactive work** you want to come back to | Must remember to start tmux first |
| **`systemd-run`** | Run as a temporary systemd unit, with logs in the journal | **Real jobs** (migrations, reindexes) | Covered properly in Notes 11–13 |

```bash
nohup ./reindex.sh > /tmp/reindex.log 2>&1 &
tmux new -s reindex          # then run the job; Ctrl+b d to detach; tmux attach -t reindex
sudo systemd-run --unit=reindex --uid=worker /opt/worker/reindex.sh
journalctl -u reindex -f     # follow its output
```

> **How this goes wrong:** attackers use the same tools (`nohup`, `setsid`, `disown`) to keep a backdoor running after they disconnect. A long-running process with **PPID 1**, no terminal, running from `/tmp` or `/dev/shm`, is a classic sign.

**Used in:** Note 10, Step 2.

---

# Phase 8 — Handling signals: graceful shutdown

## 8.1 The graceful shutdown pattern

When a service gets `SIGTERM`, a well-behaved one:

1. **Stops accepting** new work (close the listening socket, stop pulling from the queue).
2. **Finishes in-flight work** (current requests, the current job).
3. **Cleans up** (flush logs, close DB connections, remove temp files and locks).
4. **Exits** with code 0 — **before** the stop timeout, or it will be SIGKILLed anyway.

## 8.2 In Node.js

```js
const server = app.listen(port);

process.on('SIGTERM', () => {
  console.log('SIGTERM: stopping');
  server.close(() => {              // stop new connections, wait for open ones
    db.end().then(() => process.exit(0));
  });
  setTimeout(() => process.exit(1), 25000).unref();   // safety net: don't hang forever
});
```

Without a handler, Node's default for `SIGTERM` is to **exit immediately**, dropping open requests.

## 8.3 In Python

```python
import signal

stopping = False

def on_term(signum, frame):
    global stopping
    stopping = True          # just set a flag; the main loop checks it

signal.signal(signal.SIGTERM, on_term)

while not stopping:
    job = queue.get(timeout=5)
    if job:
        process(job)         # current job always finishes
# loop ended → clean up and exit normally
```

Keep the handler **tiny** (set a flag). Do the real work in the main loop.

## 8.4 In shell scripts: `trap`

`trap` runs a command when the script gets a signal or exits:

```bash
#!/bin/bash
tmp=$(mktemp)

cleanup() {
  rm -f "$tmp"
}
trap cleanup EXIT              # always runs when the script exits, for any reason
trap 'exit 143' TERM           # on SIGTERM, exit (which then triggers the EXIT trap)
trap 'exit 130' INT            # on Ctrl+C
```

`EXIT` is the most useful trap: it runs on normal exit, on error (`set -e`), and after the TERM/INT traps call `exit`.

## 8.5 Two shell traps that catch people

**1. A trap waits for the foreground command to finish.**

```bash
trap 'echo got TERM; exit 143' TERM
pg_dump ... > dump.sql        # runs 20 minutes in the foreground
```

If you send `SIGTERM` to the script, bash notes it but **runs the trap only after `pg_dump` finishes** — 20 minutes later. Fix: run the command in the background and `wait`; `wait` is interrupted by signals immediately:

```bash
pg_dump ... > dump.sql &
child=$!
trap 'kill -TERM $child 2>/dev/null; exit 143' TERM
wait $child
```

**2. Killing a script doesn't kill what it started.**

`kill <script PID>` signals only the bash process. Its children (e.g. `pg_dump`) keep running as **orphans** (Phase 5.2). Either forward the signal to the child (as above), or signal the whole **process group**:

```bash
kill -TERM -- -<PGID>         # the script and everything it started
```

## 8.6 Limiting run time: `timeout`

```bash
timeout 30m ./reindex.sh              # SIGTERM after 30 minutes
timeout -k 1m 30m ./reindex.sh        # ...and SIGKILL 1 minute later if still running
echo $?                               # 124 if it timed out
```

**Used in:** Note 10, Steps 3, 4.

---

# Phase 9 — Priority: `nice` and `ionice`

## 9.1 Nice values

When more processes want CPU than there are cores, the kernel's scheduler decides who runs. The **nice value** is a hint:

| Nice | Meaning |
|---|---|
| `-20` | Highest priority ("not nice" to others) |
| `0` | Default |
| `19` | Lowest priority ("very nice": gives way to others) |

`top` shows it as `NI`, and `PR` (= 20 + NI for normal processes).

Nice only matters **when the CPU is busy**. On an idle server, a nice-19 process still gets all the CPU it wants.

```bash
nice -n 10 ./backup.sh           # start with lower priority
renice -n 15 -p 5120             # change a running process
ps -o pid,ni,cmd -p 5120         # check
```

Rules:
- Normal users can only make their processes **nicer** (raise the number), never less nice.
- Lowering the number (higher priority) needs root.

## 9.2 Disk priority: `ionice`

Nice affects **CPU** only. A backup can hog the **disk** instead. `ionice` sets disk priority:

| Class | Meaning |
|---|---|
| `-c 1` (realtime) | Always first (root only, dangerous) |
| `-c 2 -n 0..7` (best-effort) | Normal, with levels (0 = highest) |
| `-c 3` (idle) | Only gets the disk when nobody else wants it |

```bash
ionice -c 3 nice -n 10 ./backup.sh     # low CPU and idle disk priority
```

How much effect `ionice` has depends on the disk scheduler and storage type; on some cloud disks it's limited. For hard limits, use cgroups (Note 13).

**Used in:** Note 10, Step 7.

---

# Phase 10 — Stuck processes: the `D` state

## 10.1 What `D` means

`D` (uninterruptible sleep) means the process is **inside the kernel**, waiting for something that must not be interrupted half-way — almost always **storage I/O**: a slow or failing disk, a hung network filesystem (NFS, `gcsfuse` buckets), or a stuck device.

## 10.2 Why `kill -9` doesn't work

Signals are delivered when a process returns from the kernel to its own code. A process in `D` hasn't returned, so **even SIGKILL waits** until the I/O finishes or fails. If the I/O never completes, the process stays forever — sometimes until reboot.

Some kernel waits are marked "killable": they also show as `D`, but do react to `SIGKILL`. Many are not. So the practical rule is: **if a process is still in `D` after `kill -9`, the cause is underneath it, and killing won't help.**

## 10.3 Finding out what it's waiting for

```bash
ps -eo pid,stat,wchan:32,cmd | awk '$2 ~ /D/'
```
```
 6012 D    folio_wait_bit_common    python -m worker.thumbs
```

`wchan` is the kernel function it's waiting in — a clue to the layer (here, waiting for a page of file data to be read from storage). Root can see more:

```bash
sudo cat /proc/6012/stack
```

The fix is **the thing it's waiting on**: the storage, mount, or network. Processes in `D` also count in the **load average**, so a hung mount can show high load with an idle CPU (resources area).

**Used in:** Note 10, Step 6.

---

# How this goes wrong — overview

For the security track: what normal process behaviour looks like when someone abuses it.

| Issue | Why it matters | How to check |
|---|---|---|
| Secrets in command lines | Any user can read every process's arguments with `ps` (Note 2) | `ps -eo user,cmd`; look for `-p`, `--password`, tokens |
| `/proc` visible to everyone | Users can see all processes and arguments | Mount `/proc` with `hidepid=invisible` (Note 10, Step 10) |
| Disguised process names | `argv[0]` can be anything: a backdoor named `[kworker/0:2]` or `sshd` | Real kernel threads have **PPID 2** and **no `exe` link**; compare `/proc/PID/exe` with the name |
| Deleted executables | Malware deletes its file after starting | `ls -l /proc/*/exe 2>/dev/null \| grep deleted` |
| Running from writable/temp places | Dropped tools often run from `/tmp`, `/dev/shm`, `/var/tmp` | `/proc/PID/exe` and `/proc/PID/cwd` |
| Detached long-running processes | `nohup`/`setsid` backdoors survive logout | PPID 1, no TTY, odd user, unexpected network connections (`ss -tnp`) |
| Unexpected network listeners | Bind shells, miners' pools | `sudo ss -tulpn`, `sudo ss -tnp` |
| High CPU from an unknown process | Crypto miners | `top`, then the checks above |
| Fork bombs | Create processes until the system can't start any more | Prevent with task limits (Note 13) |
| Killing others' processes | Normally `Operation not permitted`; only root/`CAP_KILL` | Who has `CAP_KILL` or broad sudo (Note 7) |

When investigating a suspicious process, **collect evidence before killing it** — its file, command line, environment, open files, and connections disappear when it dies (Note 10, Step 9).

---

# Summary table

| Concept | Key point |
|---|---|
| Program vs process | File on disk vs running instance |
| PID / PPID | Process ID / parent's ID; all processes form a tree under PID 1 |
| PID 1 / PID 2 | systemd (init, adopts orphans) / kthreadd (kernel threads) |
| Threads | Share the process's memory; `ps -L`, `top` + `H` |
| `/proc/PID/` | `status`, `cmdline`, `environ`, `exe`, `cwd`, `fd/`, `limits` |
| fds 0/1/2 | stdin / stdout / stderr |
| `ps -eo ...` | Snapshot with chosen columns; `%cpu` is lifetime average |
| `pgrep -a` / `pstree -p` / `top` | Find / family tree / live view |
| States | `R` running, `S` sleeping, `D` stuck in kernel I/O, `T` stopped, `Z` zombie |
| Exit codes | 0 OK; 126/127; 128+N = killed by signal N (130, 137, 143) |
| Zombie | Finished, not yet reaped; can't be killed; fix the parent |
| Orphan | Parent died; adopted by PID 1 or a subreaper; keeps running |
| PID 1 | Reaps orphans; ignores unhandled signals; containers need `--init` |
| Signals | Default / catch / ignore; SIGKILL and SIGSTOP can't be caught |
| SIGTERM vs SIGKILL | Polite stop with cleanup vs instant kill; always TERM first |
| SIGHUP | Terminal closed — or "reload" for daemons |
| Who can signal | Own processes; root/`CAP_KILL` for others |
| Jobs | `&`, `Ctrl+Z`, `jobs`, `fg`, `bg`, `kill %1` |
| Logout | SIGHUP kills your jobs; use `tmux` or `systemd-run` for long work |
| Graceful shutdown | Stop new work → finish current → clean up → exit before timeout |
| `trap` | `EXIT` for cleanup; traps wait for foreground commands; children aren't killed with the script |
| `timeout` | Limit run time; exit code 124 |
| `nice` / `ionice` | CPU / disk priority; matters only under contention |
| `D` state | Waiting in kernel I/O; even `kill -9` waits; fix the storage |

---

# Appendix — Command reference

## Viewing

| Command | Purpose |
|---|---|
| `echo $$` / `echo $PPID` | Current shell PID / its parent |
| `ps aux` / `ps -ef` | All processes |
| `ps -eo pid,ppid,user,stat,etime,%cpu,%mem,rss,cmd --sort=-%cpu` | Custom columns, sorted |
| `ps -fp PID` / `ps -u user` / `ps -f --ppid PID` | One process / a user's / children |
| `ps -L -p PID` | Threads |
| `pgrep -a name` / `pgrep -f pattern` / `pidof prog` | Find PIDs |
| `pstree -p` / `pstree -s -p PID` | Tree / ancestors |
| `top` (`P`, `M`, `c`, `1`, `H`, `k`, `r`) | Live view and keys |
| `htop` | Friendlier live view |

## Inspecting one process

| Command | Purpose |
|---|---|
| `grep -E '^(Name\|State\|PPid\|Uid\|Threads\|VmRSS)' /proc/PID/status` | Summary |
| `tr '\0' ' ' < /proc/PID/cmdline` | Full command line |
| `sudo ls -l /proc/PID/exe /proc/PID/cwd` | Real program and working directory |
| `sudo ls -l /proc/PID/fd` / `sudo lsof -p PID` | Open files and sockets |
| `ps -o pid,stat,wchan:32,cmd -p PID` | What a stuck process waits on |
| `sudo ss -tnp \| grep PID` | Its network connections |

## Signals

| Command | Purpose |
|---|---|
| `kill PID` / `kill -TERM PID` | Polite stop |
| `kill -9 PID` / `kill -KILL PID` | Forced stop (last resort) |
| `kill -HUP PID` | Reload (daemons) |
| `kill -STOP PID` / `kill -CONT PID` | Pause / resume |
| `kill -TERM -- -PGID` | Whole process group |
| `kill -0 PID` | Exists and allowed? |
| `pkill -u user name` / `pkill -f pattern` | By name/user / command line |
| `kill -l` | List signals |

## Jobs and detaching

| Command | Purpose |
|---|---|
| `cmd &` / `jobs -l` | Background / list jobs |
| `fg %N` / `bg %N` / `kill %N` | Foreground / continue in background / signal a job |
| `Ctrl+C` / `Ctrl+Z` / `Ctrl+\` | SIGINT / SIGTSTP / SIGQUIT |
| `nohup cmd > log 2>&1 &` | Survive logout |
| `disown %N` | Detach an already-running job |
| `setsid cmd` | New session |
| `tmux new -s name` / `tmux attach -t name` / `Ctrl+b d` | Persistent terminal session |
| `systemd-run --unit=name --uid=user cmd` | Run as a temporary unit with logs |

## Scripts

| Command | Purpose |
|---|---|
| `trap cleanup EXIT` | Cleanup on any exit |
| `trap 'exit 143' TERM` / `trap 'exit 130' INT` | Handle stop / Ctrl+C |
| `cmd & child=$!; wait $child` | Background + wait (signals handled immediately) |
| `timeout -k 1m 30m cmd` | Time limit with forced kill |

## Priority

| Command | Purpose |
|---|---|
| `nice -n 10 cmd` | Start with lower CPU priority |
| `renice -n 15 -p PID` | Change a running process |
| `ionice -c 3 cmd` | Idle disk priority |
| `ps -o pid,ni,pri,cmd -p PID` | Check priority |