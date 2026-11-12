# L14 — de-log-evasion · Leave No Line Behind

**Act III · Persistence and Evasion · T1070.002**
**Status:** Solved
**Flag:** `bl_c9aef3604b1e34744bcdfa86f7ece269`

---

## Challenge brief

> Auditing is watching the paths that matter. Do the thing you need to do, then make
> sure the record never shows it. Selective, quiet, and under the threshold.

MITRE tag T1070.002 is "Indicator Removal: Clear Linux or Mac System Logs" — removing
specific evidence from log files after performing an action, so the record of that
action disappears.

You land as root. The box has a live monitoring agent watching a file. You need to read
that file to get the secret, then remove the access event from the agent's log before
the grader checks it.

---

## Background: what logs are and why attackers clean them

When you do anything on a Linux box — log in, read a file, run a command — the OS
records it somewhere. These records exist so that administrators and security teams can
reconstruct what happened after the fact. For an attacker, those records are evidence.

The goal of log evasion is not to make the box look completely untouched — that is
often impossible. It is to remove the specific records that tie your actions to the
compromise. "The record never shows it" means the entries that would prove you accessed
the file are gone, even if other log entries remain.

This level is about **selective** removal — removing only the evidence of your specific
action, leaving the rest of the log intact so the gap is not obvious.

---

## inotify — how Linux watches files

Before understanding the monitoring agent, you need to understand `inotify` — the
Linux kernel mechanism for watching files and directories.

`inotify` is a kernel subsystem that lets a process register interest in a file or
directory. The kernel then sends notifications to that process whenever a specified
event happens to the watched file. The process never has to poll — it just waits and
the kernel tells it when something happens.

The events you can watch for include:

| Event | Triggered when |
|---|---|
| `IN_ACCESS` | File was read |
| `IN_OPEN` | File was opened |
| `IN_MODIFY` | File content was changed |
| `IN_CREATE` | A file was created inside a watched directory |
| `IN_DELETE` | A file was deleted |
| `IN_ATTRIB` | File metadata changed (permissions, owner, timestamps) |

In this challenge the agent watches for `access` and `open` events on `/opt/crown.key`.
Every time any process opens or reads that file, inotify fires immediately.

### inotifywait — the command-line interface to inotify

`inotifywait` is a command that uses the inotify API and outputs events to stdout as
they happen. The agent runs:

```sh
inotifywait -m -e access -e open \
  --format '{"ts":%T,"ev":"%e","f":"%w"}' \
  --timefmt '%s' \
  /opt/crown.key
```

Breaking down each flag:

- `-m` — monitor mode. Run forever, don't exit after the first event. Without this,
  `inotifywait` exits after catching one event.
- `-e access -e open` — only report these two event types. Access means the file was
  read. Open means a process opened the file (which happens before reading).
- `--format '{"ts":%T,"ev":"%e","f":"%w"}'` — format each event as a JSON object.
  `%T` is the timestamp, `%e` is the event type (`ACCESS` or `OPEN`), `%w` is the
  watched file path.
- `--timefmt '%s'` — format the timestamp as a Unix epoch timestamp (seconds since
  1970). This is what `%T` outputs.
- `/opt/crown.key` — the file to watch.

The output for one open event looks like:

```json
{"ts":1794326384,"ev":"OPEN","f":"/opt/crown.key"}
```

This is **JSONL format** — JSON Lines. One JSON object per line, each line self-contained.
It is a common log format because it is easy to parse line-by-line and each line is a
complete record.

---

## The monitoring agent — how it works

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
  sleep 2
done
```

The agent does three things:

1. **Clears the alert log** (`': > "$ALERT"'`) — truncates `agent.jsonl` to zero bytes
   at startup. The `: >` idiom is a shell way to empty a file: `:` is a no-op command
   that succeeds, and `>` redirects its (empty) output to the file, truncating it.

2. **Starts `inotifywait` in the background** (`&`) watching `$MON_WATCH`
   (which is `/opt/crown.key`) and appending (`>>`) every event to `$ALERT`
   (`/var/log/agent.jsonl`).

3. **Writes a heartbeat** (`date +%s > "$HB"`) to `/run/agent.hb` every 2 seconds.
   The grader checks this file to confirm the agent is still alive — if the heartbeat
   stops being updated, the grader knows the agent was killed. This prevents the simple
   "just kill the agent" approach.

The agent comment in the script says: "fully in-box and evadable." The challenge
designers are telling you that the monitoring is intentionally bypassable — this is a
learning exercise in how userspace monitoring agents have limits that kernel-level
auditing does not.

---

## Why this monitoring agent is evadable

`inotifywait` delivers events asynchronously. When you open `/opt/crown.key`, the kernel
fires the inotify notification and `inotifywait` receives it. But there is a small
delay between when `inotifywait` receives the event and when it writes the JSONL line
to the log file.

More importantly: the log file is a plain text file at `/var/log/agent.jsonl`. The
agent writes to it, but it has no integrity protection — no hash, no seal, no secondary
check. As root, you can edit the file after the agent writes to it. The agent will never
know, because it only appends to the file; it never reads back what it wrote.

This is the fundamental weakness of userspace monitoring agents:
- They write evidence to a file
- Root can edit that file
- The agent has no way to detect or prevent this

Contrast this with **auditd** (the kernel audit framework) and **Falco** (a kernel-level
security tool). These operate at the kernel level — they intercept system calls before
they even reach userspace. Even root cannot edit their records after the fact without
either corrupting the log (which is itself a finding) or having access to the audit
daemon's own keys. The agent comment explicitly says: "Not auditd/Falco" — the
challenge is teaching the difference between weak userspace monitoring and stronger
kernel-level monitoring.

---

## The solve: read, then clean

The sequence is:

1. Read `/opt/crown.key` — this triggers `inotifywait`, which writes an `OPEN` and
   possibly an `ACCESS` entry to `agent.jsonl`.
2. Immediately remove those entries from `agent.jsonl` using `sed -i`.
3. Write the secret to the proof file.
4. Run `verify` — the grader checks `agent.jsonl` and sees no access events.

The timing matters but is not tight. `inotifywait` is fast — entries appear in the log
within milliseconds. But `sed -i` runs in milliseconds too. As long as you clean the
log before calling `verify`, the grader sees nothing.

---

## Enumeration log

### Step 1 — Run verify to understand the grader

```sh
verify
```

```
Not solved yet. the crown secret was not recovered into the proof
Write the value you recover to /tmp/.de-log-proof, then run verify again.
```

Proof file: `/tmp/.de-log-proof`. The grader wants a "crown secret" — something we
need to find, not just a log to erase. Two requirements: find the secret AND clean the
evidence.

---

### Step 2 — Understand what's running

```sh
ps aux
```

Three things stand out:

```
root   7  ... /bin/sh /usr/local/bin/monitor-agent.sh
root   8  ... inotifywait -m -e access -e open ... /opt/crown.key
```

A monitoring agent and an `inotifywait` process watching `/opt/crown.key`. The log:

```sh
ls /var/log/
```

```
agent.jsonl
```

Only one log file. This is the agent's alert stream.

---

### Step 3 — Read the agent script

```sh
cat /usr/local/bin/monitor-agent.sh
```

Reveals: the agent uses a heartbeat at `/run/agent.hb` (so killing it would be
detected), appends events to `/var/log/agent.jsonl`, and watches `$MON_WATCH`. Since
`inotifywait` already shows `/opt/crown.key` in `ps aux`, that is the watched file.

Also check the current log state:

```sh
cat /var/log/agent.jsonl
```

```json
{"ts":1794326384,"ev":"OPEN","f":"/opt/crown.key"}
```

An `OPEN` event already exists — probably from the grader's initial test. This confirms
the format and that the log is active.

---

### Step 4 — Read the secret and immediately clean the log

```sh
cat /opt/crown.key
```

```
2c9c698c9742f2161c737af0442b0bc5
```

The moment `cat` opened the file, `inotifywait` fired and appended a new `OPEN` entry
to `agent.jsonl`. Clean it immediately:

```sh
sed -i '/crown\.key/d' /var/log/agent.jsonl
cat /var/log/agent.jsonl
```

Empty. All entries mentioning `crown.key` are gone — the original one from before and
the new one from our `cat` command.

**What `sed -i '/crown\.key/d'` does:**

- `sed -i` — edit the file in place (overwrite the original, not a copy)
- `/crown\.key/` — match any line containing `crown.key` (the `\.` escapes the dot so
  it matches a literal dot, not "any character")
- `d` — delete the matched line

This is the standard text log editing technique. Because `agent.jsonl` is plain text
(JSONL), one line = one event, so deleting matching lines removes those events cleanly.

---

### Step 5 — Write proof and verify

```sh
printf '2c9c698c9742f2161c737af0442b0bc5' > /tmp/.de-log-proof && verify
```

**Level solved.**

---

## What the vulnerability demonstrates

The monitoring agent represents a common real-world pattern: a userspace daemon that
writes security events to a log file. It is better than nothing — it catches events in
real time and would catch an attacker who does not know to clean up. But it has a
fundamental limit: the log file it writes to is on the same filesystem, owned by root.
An attacker who already has root can edit that file.

**What the agent does well:**
- Catches access events in real time with no polling delay
- Heartbeat confirms it is running (killing it would be detected)
- Logging to a separate file means events are captured even if the box is noisy

**What the agent cannot defend against:**
- Root editing the log file after the fact
- An attacker who disables inotify watches before accessing the file (by remounting
  the filesystem with `nosymfollow` or other techniques)
- An attacker who reads the file through a path inotify is not watching (a hardlink,
  a bind mount, or reading from the block device directly)

**What would stop this attack:**
- Forwarding logs to a remote syslog server in real time — the attacker cannot edit
  logs that have already left the box
- Using kernel-level auditing (auditd or eBPF-based tools like Falco) — these intercept
  syscalls before userspace, and their records cannot be edited without corrupting the
  audit log itself, which is itself a detectable finding
- Making the log file append-only with `chattr +a /var/log/agent.jsonl` — even root
  cannot overwrite or truncate an append-only file, though root can remove the attribute

---

## The heartbeat — why killing the agent would fail

The agent writes `date +%s` (the current Unix timestamp) to `/run/agent.hb` every 2
seconds. The grader checks this file — if the timestamp is stale (older than a few
seconds), it knows the agent was killed and would likely fail the level.

Killing PID 7 (the agent shell) would stop the heartbeat writes within 2 seconds. The
grader would detect a stale heartbeat and know something tampered with the monitoring.

Killing PID 8 (`inotifywait`) would stop new events from being recorded, but the
heartbeat would continue. This is a subtler approach — the grader might still detect
that inotifywait is no longer running. The challenge notes: "fully in-box and evadable"
but this refers to log editing, not to killing processes.

The correct approach — editing the log after the fact — avoids both problems. The agent
keeps running, the heartbeat stays fresh, but the evidence of access is gone.

---

## Dead ends

There are no major dead ends in this level — the solve path was direct once the agent
script was read. The key was reading `monitor-agent.sh` first to understand what was
being watched and where the log was written, rather than blindly trying things.

---

## Comparison: userspace monitoring vs kernel-level monitoring

| Property | Userspace agent (this challenge) | auditd / Falco (kernel-level) |
|---|---|---|
| How it works | Process registers inotify watches, writes to a log file | Kernel intercepts syscalls and writes to audit ring buffer |
| Can root edit the log? | Yes — it is a file on the filesystem | No — the ring buffer is in kernel memory; editing log files leaves the kernel copy |
| Can the agent be killed? | Yes — grader may detect via heartbeat | The kernel audit subsystem cannot be killed without root changing the audit rules, which itself creates an audit record |
| Can events be missed? | Yes — if the log file is edited after the fact | Very difficult — events are recorded at the syscall level before userspace sees them |
| Detects file reads? | Yes via inotify | Yes via `-w file -p r` audit rules |
| Works in containers? | Yes — inotify works in container namespaces | No for kernel auditd (not namespaced, would see all container events) — Falco with eBPF has better container support |
| Real-world examples | Simple SIEM agents, file integrity monitors with no remote logging | Linux auditd, Falco, eBPF-based detection tools |

The challenge comment in `monitor-agent.sh` explicitly notes: "Not auditd/Falco: the
kernel audit subsystem is not namespaced, so an in-container kernel monitor would read
host-wide events and break per-session isolation." This is a real Docker/container
limitation — kernel audit events are not namespaced, so you cannot run per-container
kernel auditing without affecting the whole host.

---

## Carryover findings

None. The level is self-contained.

---

## Command reference

| Command | What it does |
|---|---|
| `ps aux` | List running processes — reveals the monitoring agent and inotifywait |
| `cat /usr/local/bin/monitor-agent.sh` | Read the agent script to understand what is watched and where logs go |
| `ls /var/log/` | Find the agent's log file |
| `cat /var/log/agent.jsonl` | Read current log state before acting |
| `cat /opt/crown.key` | Read the watched file — triggers an inotify event |
| `sed -i '/crown\.key/d' /var/log/agent.jsonl` | Remove all log lines mentioning the watched file |
| `cat /var/log/agent.jsonl` | Verify the log is clean |
| `printf 'VALUE' > /tmp/.de-log-proof && verify` | Write proof and verify |
| `inotifywait -m -e access -e open --format '...' --timefmt '%s' FILE` | Monitor a file for access/open events (how the agent works) |
| `: > /path/to/file` | Truncate a file to zero bytes (shell idiom) |
| `chattr +a /var/log/file` | Make a log file append-only — even root cannot overwrite it |
| `ausearch -k rule-key` | Search kernel audit log — not editable from userspace |