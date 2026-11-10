# L15 — de-process-hide · The Process That Is Not There

**Act III · Persistence and Evasion · T1036.004**
**Status:** Solved
**Flag:** `bl_7e9c8420c62556b62eb0a787244ca3b2`

---

## Challenge brief

> An enumerator is about to walk this box looking for you: a process, a port, a file.
> Run what you need to run and make sure none of it appears when the box is asked what
> it is running.

MITRE tag T1036.004 is "Masquerade Task or Service" — running a process in a way that
it either does not appear in process listings or appears as something innocent.

You land as root. A monitoring agent is running that hunts for processes by cmdline
signature. You need to run a process that continuously writes to the proof file, while
keeping it out of the agent's detection.

---

## Background: how ps and pgrep actually work

To understand how to evade process detection, you first need to understand how detection
tools find processes.

### What /proc is

`/proc` is a virtual filesystem maintained by the Linux kernel. It is not stored on
disk — the kernel generates its contents on the fly whenever you read from it. Every
running process gets a directory at `/proc/<pid>/`. Inside that directory:

| File | Contents |
|---|---|
| `/proc/<pid>/status` | Human-readable process status: name, PID, state, memory usage, UID |
| `/proc/<pid>/cmdline` | The full command line the process was started with, null-byte separated |
| `/proc/<pid>/comm` | The process name — just the executable name, up to 15 characters |
| `/proc/<pid>/environ` | The process's environment variables at startup, null-byte separated |
| `/proc/<pid>/exe` | Symlink to the executable binary |
| `/proc/<pid>/fd/` | Directory of open file descriptors |

### How ps reads process information

`ps aux` does not have a special kernel interface. It works entirely through `/proc`:

1. Opens `/proc` as a directory
2. Lists all numeric subdirectories (each is a PID)
3. For each PID, reads `/proc/<pid>/status` and `/proc/<pid>/cmdline`
4. Formats and prints the results

This means `ps` is just a tool that reads files. It has no special privilege or kernel
connection beyond what any file read gives it.

### How pgrep -f works

`pgrep` finds processes by matching against their attributes. The `-f` flag specifically
means: match against the full command line (from `/proc/<pid>/cmdline`), not just the
process name.

```sh
pgrep -f "phantom-beacon"
```

This reads `/proc/<pid>/cmdline` for every running process and returns the PIDs of any
process whose full command line contains the string `phantom-beacon`. Without `-f`,
pgrep would only match against the process name in `/proc/<pid>/comm` (the short name).

**What is in cmdline?** When a process starts, its full invocation is stored in
`/proc/<pid>/cmdline`. For example:

```sh
while true; do sleep 1; done &
```

This shell loop's cmdline in `/proc` would look like `/bin/sh` (since it's running as a
shell script). But if you run:

```sh
phantom-beacon --write /tmp/proof
```

The cmdline would contain `phantom-beacon`, and `pgrep -f phantom-beacon` would find it.

The key distinction: **cmdline is set at process start and reflects how the process was
invoked, not what it is doing.** An anonymous shell loop (`while true; do ...; done`)
running inside a `/bin/sh` process has `/bin/sh` as its cmdline. The loop itself is not
a separate process — it runs inside the shell process.

---

## The monitoring agent — what it hunts for

```sh
cat /usr/local/bin/monitor-agent.sh
```

The relevant part:

```sh
MON_PROC_SIG=phantom-beacon

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

Every 2 seconds, the agent runs `pgrep -f phantom-beacon`. If any process has
`phantom-beacon` in its full cmdline, the agent logs a `PROC` alert to `agent.jsonl`:

```json
{"ts":1794327100,"ev":"PROC","pid":1234}
```

The grader reads `agent.jsonl`. If it sees `PROC` alerts, it knows the agent detected
something. So our process must not have `phantom-beacon` in its cmdline — otherwise the
agent finds it and reports it.

The agent also uses a heartbeat — writes the current timestamp to `/run/agent.hb` every
2 seconds. The grader checks the heartbeat is fresh. This prevents the simple "just kill
the agent" approach.

---

## What the grader actually checks

```sh
verify
```

```
Not solved yet. no live process is re-writing the objective value
Write the value you recover to /tmp/.de-hide-proof, then run verify again.
```

"No live process is re-writing the objective value" — the grader is checking that:

1. `/tmp/.de-hide-proof` exists and contains the correct value
2. That value is being **continuously updated** by a live process — not just written
   once and left

The grader verifies this by checking the file is being updated during the verification
window (while it's doing the `Verifying.....` dots). A single write before `verify`
would not be enough — the file needs to be actively written by a running process.

The two requirements are therefore:

- **Grader requirement:** a live process continuously writing the secret to the proof
  file
- **Agent requirement:** that process must not have `phantom-beacon` in its cmdline

These two requirements are independent. The grader checks the proof file; the agent
checks process cmdlines. We just need a process that satisfies both — write to the
proof file, but don't be named `phantom-beacon`.

---

## The beacon secret

```sh
find /opt -type f 2>/dev/null
```

```
/opt/.beacon-secret
```

```sh
cat /opt/.beacon-secret
```

```
f04ba40d40af649fac2253a0bbef857d
```

This is the value our process needs to continuously write to `/tmp/.de-hide-proof`.

---

## The solve: an anonymous shell loop

The simplest process that satisfies both requirements is a shell `while` loop running in
the background:

```sh
while true; do printf 'f04ba40d40af649fac2253a0bbef857d' > /tmp/.de-hide-proof; sleep 1; done &
```

**Why this works against the agent:** This loop runs inside the current shell (`/bin/sh`).
It is not a separate process with its own cmdline entry — it is code executing within
the shell process. The shell's cmdline in `/proc/<pid>/cmdline` is `/bin/sh -l` (the
login shell). There is no `phantom-beacon` anywhere in that cmdline. `pgrep -f
phantom-beacon` finds nothing, and the agent logs nothing.

**Why this satisfies the grader:** The loop runs every 1 second, writing the secret to
the proof file. During the grader's verification window, the file is being continuously
updated. The grader confirms a live process is writing it.

**Breaking down the command:**
- `while true; do ... done` — a shell loop that runs forever
- `printf 'f04ba40d40af649fac2253a0bbef857d'` — write the secret without a trailing
  newline (consistent with how graders typically compare values)
- `> /tmp/.de-hide-proof` — overwrite the proof file each iteration
- `sleep 1` — wait 1 second between writes (keeps the CPU usage near zero)
- `&` — run the whole loop in the background, returning control to the shell immediately

After running this, the shell prompt returns immediately while the loop continues
running in the background.

---

## Why the LD_PRELOAD approach wasn't needed

The topic note covers a full userland rootkit — compiling a `.so` file that hooks
`readdir64` to filter out process entries from `/proc`, making a process truly invisible
to `ps`. This level was initially assumed to need that approach because of the process
hiding angle.

But the actual requirement was simpler: don't have `phantom-beacon` in the cmdline. The
agent uses `pgrep -f`, which checks cmdlines. It does not check whether a process
exists in `ps aux` or enumerate `/proc` directly. An anonymous shell loop with no
suspicious cmdline passes this check trivially.

**When the LD_PRELOAD rootkit would be needed:** if the agent (or the grader) checked
whether a specific binary was running by scanning `ps` or `/proc` directly — looking for
any process, regardless of cmdline. Or if the challenge required hiding from a
human who would run `ps aux` and investigate. In this challenge, the detection surface
was specifically `pgrep -f phantom-beacon`, so the simplest evasion was just not using
that string.

This is a real-world lesson: the right level of evasion depends on what the detection
mechanism actually checks. Over-engineering (a full LD_PRELOAD rootkit) wastes time and
introduces failure points when a simpler approach works.

---

## How pgrep -f is used in real monitoring

`pgrep -f` with a signature string is a common pattern in real monitoring scripts,
EDR (endpoint detection and response) agents, and simple IDS setups. For example:

- A security team might run `pgrep -f 'nc -e'` to hunt for netcat reverse shells
- A WAF might watch for `pgrep -f 'python.*SimpleHTTPServer'` to detect attackers
  serving payloads
- A monitoring agent might scan for `pgrep -f 'msfconsole'` or `pgrep -f 'beacon'`

The evasion in each case is the same: rename or wrap your tool so its cmdline does not
contain the signature. Common real-world techniques:

**1. Use a shell wrapper.** Instead of running `nc -e /bin/sh`, run a shell loop that
does the same work without the `nc -e` signature in the cmdline. The loop's parent
process (`/bin/sh`) appears innocent.

**2. Rename the binary.** Copy `nc` to `/tmp/.update` and run it from there. `pgrep -f
'nc -e'` won't match, but `pgrep -f '.update'` would. Choose a name that blends in:
`/tmp/kworker`, `/usr/lib/systemd/.helper`.

**3. Use exec -a to set argv[0].** Shell's `exec -a name command` starts a command with
a custom `argv[0]`, which is what appears as the process name in `ps` and in
`/proc/<pid>/comm`. The cmdline in `/proc/<pid>/cmdline` still shows the actual binary
path, but `comm` (what `ps` shows in the name column) shows your chosen name.

**4. Encode or alias the signature.** If the detection hunts for `phantom-beacon`, run
`ph${sep}antom-be${sep}acon` with a variable expansion that collapses to the full name
at runtime but never appears as a literal string in the cmdline. (This works against
simple string-match detectors; more sophisticated ones check the actual binary on disk.)

**5. Use a cron job or systemd timer.** A cron entry runs as a child of the cron daemon.
`pgrep -f` sees the cron daemon's cmdline, not the actual command being run. The command
runs briefly, finishes, and disappears from the process table.

---

## ps aux — what the columns mean

Since `ps aux` appears throughout this track, it is worth explaining fully:

```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           7  0.0  0.0   1628   960 ?        S    16:12   0:00 /bin/sh /usr/local/bin/monitor-agent.sh
```

| Column | Meaning |
|---|---|
| `USER` | The user running the process |
| `PID` | Process ID — the unique number the kernel assigns |
| `%CPU` | CPU usage percentage averaged over the last minute |
| `%MEM` | Memory usage as a percentage of total RAM |
| `VSZ` | Virtual memory size in KB — all memory the process has mapped, including shared libraries |
| `RSS` | Resident set size in KB — actual physical RAM currently in use |
| `TTY` | The terminal this process is attached to. `?` means no terminal (background daemon) |
| `STAT` | Process state: `S` = sleeping, `R` = running, `Z` = zombie, `s` = session leader, `+` = foreground |
| `START` | When the process started |
| `TIME` | Total CPU time consumed (not wall clock time) |
| `COMMAND` | The command line (`a` flag) or just the name (`-e` flag) |

The `a` in `ps aux` means show all processes (not just those for the current user). The
`u` means show the user-oriented format (all the columns above). The `x` means include
processes with no terminal (`TTY=?`).

`ps -ef` is an alternative format that shows PPID (parent PID) instead of memory info,
which is useful for seeing which process spawned which.

---

## Enumeration log

### Step 1 — Run verify to understand the grader

```sh
verify
```

```
Not solved yet. no live process is re-writing the objective value
Write the value you recover to /tmp/.de-hide-proof, then run verify again.
```

Key phrase: "live process is re-writing" — not just a file that exists, but a process
actively updating it. Proof file: `/tmp/.de-hide-proof`.

---

### Step 2 — Understand the monitoring agent

```sh
cat /usr/local/bin/monitor-agent.sh
cat /proc/7/environ | tr '\0' '\n' | grep MON
```

```
MON_PROC_SIG=phantom-beacon
MON_HB=/run/agent.hb
MON_ALERT=/var/log/agent.jsonl
```

The agent hunts for processes with `phantom-beacon` in their cmdline. It has a
heartbeat — cannot be killed. The alert log is `/var/log/agent.jsonl`.

---

### Step 3 — Find the secret

```sh
find /opt -type f 2>/dev/null
cat /opt/.beacon-secret
```

```
f04ba40d40af649fac2253a0bbef857d
```

---

### Step 4 — Start the hidden process

```sh
while true; do printf 'f04ba40d40af649fac2253a0bbef857d' > /tmp/.de-hide-proof; sleep 1; done &
```

This loop runs inside the current shell. Its parent process's cmdline is `/bin/sh -l`.
No `phantom-beacon` string appears anywhere. The agent's `pgrep -f phantom-beacon`
returns nothing.

```sh
verify
```

**Level solved.**

---

## What the challenge demonstrates

The challenge title "Not on the List" and the MITRE tag T1036.004 (Masquerade Task or
Service) are about running something that evades process-based detection. The lesson is
that detection quality is only as good as what it checks:

- An agent that hunts `phantom-beacon` in cmdlines misses any process that does the
  same work without that string
- A human running `ps aux` would see the shell loop — it's not invisible, it just
  doesn't match the signature
- Stronger detection would enumerate all processes, check what each one does, and flag
  anything that writes to sensitive proof paths — not just string-match on cmdlines

In a real penetration test, this maps to: know what your target's EDR signatures look
for, and structure your tools so none of their cmdlines, file names, or network patterns
match those signatures. The simplest version is just not naming your reverse shell `nc`
or not storing your payload as `meterpreter.exe`.

---

## Dead ends

### Assuming LD_PRELOAD rootkit was needed

The level title and topic note pointed toward a full LD_PRELOAD process-hiding rootkit.
No `gcc` on the box confirmed this was not the path. The actual requirement was just
"don't have `phantom-beacon` in the cmdline" — which an anonymous shell loop satisfies
without any C code.

### Overthinking the hiding requirement

The grader does not run `ps aux` and check whether our process appears. It checks the
proof file is being updated and checks `agent.jsonl` for `PROC` alerts. The agent only
generates those alerts via `pgrep -f phantom-beacon`. Our shell loop is completely
invisible to that check.

---

## Carryover findings

None. The level is self-contained.

---

## Command reference

| Command | What it does |
|---|---|
| `cat /proc/<pid>/cmdline \| tr '\0' '\n'` | Read a process's full command line — what pgrep -f matches against |
| `cat /proc/<pid>/environ \| tr '\0' '\n' \| grep MON` | Read the monitoring agent's environment to find what it hunts for |
| `pgrep -f "string"` | Find PIDs of any process with "string" in its full cmdline |
| `pgrep -f "string" -l` | Same but also print the process name alongside the PID |
| `find /opt -type f 2>/dev/null` | Find the secret file on disk |
| `while true; do printf 'VALUE' > /tmp/proof; sleep 1; done &` | Background loop continuously writing to proof file — no suspicious cmdline |
| `ps aux` | List all processes with full details — shows PID, user, CPU, memory, cmdline |
| `cat /var/log/agent.jsonl` | Read the monitor agent's alert log — PROC entries mean detection |
| `printf 'VALUE' > /tmp/.de-hide-proof && verify` | Write proof and verify |