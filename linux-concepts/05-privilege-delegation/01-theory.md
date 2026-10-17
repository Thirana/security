# Privilege & Delegation — Theory (Note 7)

How a process gets **more** power than its identity normally allows, and how to grant that power narrowly and on purpose. This is the fourth layer from Note 5's decision model: the ways around the ordinary rules — SUID/SGID, sudo, capabilities — plus the modern way to run a service with *less* power than normal (systemd sandboxing).

**Series:**
- Note 1: Shell & Processes
- Note 2: Environment Variables
- Note 3 / 4: Users, Groups & Identity — Theory / Practice
- Note 5 / 6: Files, Directories & Permissions — Theory / Practice
- **Note 7: Privilege & Delegation — Theory** ← this note
- Note 8: Privilege & Delegation — Practice

Notes 3–6 were about staying inside the rules. This note is about the rules' exceptions: who is allowed to step outside them, by how much, and how to keep that step as small as possible.

Each phase ends with **"Used in"** (Note 8 steps) and a **"How this goes wrong"** box for the security track.

---

## Contents

1. Phase 1 — The three UIDs a process carries
2. Phase 2 — SUID and SGID on programs
3. Phase 3 — Why SUID is dangerous, and shell escapes
4. Phase 4 — sudo: the delegation tool
5. Phase 5 — The sudoers file in depth
6. Phase 6 — sudo's traps: what "narrow" really means
7. Phase 7 — sudoedit, environment, and logging
8. Phase 8 — Capabilities: root split into pieces
9. Phase 9 — Capabilities in practice: files, processes, services
10. Phase 10 — systemd as a privilege and sandbox tool
11. Phase 11 — The full privilege picture
12. How this goes wrong — overview
13. Summary table
14. Appendix — Command reference

---

# Phase 1 — The three UIDs a process carries

Note 3 said a process carries a UID. It actually carries **three** (and a fourth for filesystems). They're what make privilege changes possible.

| UID | Name | Meaning |
|---|---|---|
| **RUID** | Real UID | Who **launched** the process |
| **EUID** | Effective UID | Whose privileges the process **acts with right now** — this is what the kernel checks (Note 5, Phase 12) |
| **SUID** | Saved UID | A **parked** value the process can switch back to |
| (FSUID) | Filesystem UID | Used only for file access; almost always equals EUID |

The same three exist for groups (RGID, EGID, SGID).

## 1.1 Normally they're all equal

When you run `cat`, all three are your UID. Nothing special happens.

```bash
grep -E '^(Uid|Gid)' /proc/self/status
```
```
Uid:  1101  1101  1101  1101      ← real, effective, saved, fs
Gid:  1101  1101  1101  1101
```

## 1.2 They split when privilege is involved

Two things pull them apart:

- A **SUID program** starts with EUID = the file's owner, while RUID stays the launcher. (Phase 2)
- A program **drops privilege on purpose**: a service starts as root (to bind a port, say), then sets its EUID to an unprivileged user, keeping root parked in the saved UID — or dropping it entirely.

The reason for the saved UID: a program can **temporarily** drop to the real user, do the risky part, then raise back to the owner. Well-written SUID programs do exactly this, lowering privilege except for the one moment they need it.

```bash
# see another process's three UIDs
grep -E '^(Uid|Gid)' /proc/<PID>/status
```

> **Why it matters:** "the process is root" is ambiguous. A service showing `Uid: 0 1000 0` is running **as 1000** but can become root again. One showing `Uid: 1000 1000 1000` has dropped root for good. The difference decides how dangerous a compromise is.

**Used in:** Note 8, Steps 1, 5.

---

# Phase 2 — SUID and SGID on programs

## 2.1 What they do

| Bit | On an executable | Effect when run |
|---|---|---|
| **SUID** (`4000`) | `-rwsr-xr-x` | EUID becomes the **file's owner** |
| **SGID** (`2000`) | `-rwxr-sr-x` | EGID becomes the **file's group** |

Because the kernel checks the **effective** ID (Note 5, Phase 12), the program acts with the owner's (or group's) access for as long as it runs. The real UID stays the launcher, so the program can still tell who called it.

## 2.2 The textbook example: `passwd`

```bash
ls -l /usr/bin/passwd
```
```
-rwsr-xr-x 1 root root 59976 ... /usr/bin/passwd
```

You (a normal user) run `passwd`. It needs to write `/etc/shadow`, which is `640 root:shadow`. SUID makes its EUID root for that write, then it exits and your privilege is gone. This is the mechanism working as intended: a small, carefully written, root-owned program doing **one** narrow job.

## 2.3 Finding SUID/SGID programs

```bash
sudo find / -xdev -perm -4000 -type f 2>/dev/null     # SUID
sudo find / -xdev -perm -2000 -type f 2>/dev/null     # SGID
sudo find / -xdev -perm -6000 -type f -ls 2>/dev/null # both, with details
```

A normal system has a short, known list: `passwd`, `sudo`, `su`, `mount`, `umount`, `ping` (on older systems), `chsh`, `newgrp`, and a few more. **Anything outside that list deserves suspicion** — it's the first place the security track looks.

**Used in:** Note 8, Steps 1, 6.

---

# Phase 3 — Why SUID is dangerous, and shell escapes

## 3.1 The core risk

A SUID-root program runs **everything it does** as root. If the program can be made to do something its author didn't intend — run a command, open a file you name, follow an environment variable — that action happens as root. So a SUID program carries the full burden of never letting the user redirect it.

## 3.2 Shell escapes

Many normal programs can run **another** command from inside themselves. If such a program is SUID-root, that escape hatch becomes a **root shell**.

Classic examples (all as intended features, dangerous only when the program is SUID):

| Program | Escape |
|---|---|
| `vim` / `less` / `more` | `:!sh` or `!sh` runs a shell |
| `find` | `find . -exec /bin/sh \;` |
| `awk` / `env` / `python` / `perl` | run arbitrary code directly |
| `nmap` (old interactive mode) | `!sh` |

If `find` were SUID-root, `find . -exec sh \;` would give a root shell. This is why **you never make an interpreter, editor, pager, or a tool with a shell-out feature SUID.** The project **GTFOBins** catalogues these escapes — the defender uses it to know what *not* to make SUID or allow via sudo.

## 3.3 Other SUID pitfalls

- **Environment** (`LD_PRELOAD`, `LD_LIBRARY_PATH`, `PATH`, `IFS`) — Note 2, Phase 5.5. The kernel strips `LD_*` for SUID programs, and a safe SUID program sets its own PATH. A SUID script that calls `tar` by bare name instead of `/bin/tar` can be hijacked via PATH.
- **SUID shell scripts** — the kernel **ignores** the SUID bit on `#!` scripts on Linux (too racy to secure), so they silently run without privilege. People then "fix" it with a SUID wrapper, which reintroduces the risk.

## 3.4 The modern stance

Because every SUID-root program is a potential full-root hole, the modern preference is:

1. **A capability** instead of SUID when a program needs one specific power (Phase 8).
2. **sudo** with a narrow rule when a *person* needs to run something as root (Phase 4).
3. **A service** (systemd) that holds the privilege, with the user asking it to act, instead of a SUID binary.

> **How this goes wrong:** a custom SUID-root helper that runs a command, or an interpreter/editor left SUID. It turns into a one-line privilege escalation. The whole of Topic 2's "SUID" section is this phase inverted.

**Used in:** Note 8, Steps 1, 6.

---

# Phase 4 — sudo: the delegation tool

## 4.1 What sudo is for

SUID bakes privilege into a *file* permanently. **sudo** grants privilege to a *person* for a *specific command*, governed by a central policy, with every use logged. It's the right tool whenever someone needs to do an admin task without being handed full root.

## 4.2 sudo vs su (recap from Note 3)

| | `su` | `sudo` |
|---|---|---|
| Does | Become another user (new shell) | Run one command as another user |
| Password | **Target's** | **Your own** |
| Governed by | Knowing the target password | The sudoers policy |
| Logged per command | No | **Yes** |
| Scope | Everything | Exactly what the rule allows |

## 4.3 How sudo gets its power

```bash
ls -l /usr/bin/sudo
```
```
-rwsr-xr-x 1 root root ... /usr/bin/sudo
```

sudo is itself **SUID-root** — but it's one carefully audited program (Phase 3's burden, carried well) instead of many. That's the trade: one trusted gatekeeper rather than privilege scattered across binaries.

## 4.4 The basics

```bash
sudo cmd                 # run cmd as root
sudo -u api cmd          # run cmd as api (used all through Notes 4 and 6)
sudo -l                  # list what you may run
sudo -k                  # forget the cached timestamp (next sudo asks again)
sudo -v                  # refresh the timestamp
```

After a successful password, sudo caches it for a few minutes (per terminal) so you aren't asked on every command.

**Used in:** Note 8, Steps 2, 3, 4.

---

# Phase 5 — The sudoers file in depth

## 5.1 Where the policy lives

| File | Purpose |
|---|---|
| `/etc/sudoers` | Main policy. **Edit only with `visudo`** |
| `/etc/sudoers.d/*` | Drop-in files (same syntax). Edit with `visudo -f /etc/sudoers.d/name` |

`visudo` checks the syntax **before** saving. A broken sudoers file can lock everyone out of root, so a bad save that slipped through is a real emergency. `visudo -c` checks all files.

Drop-ins are preferred: one file per purpose, easy to add and remove, survives package upgrades.

## 5.2 A rule, field by field

```text
ruwan   ALL = (root)      /usr/bin/systemctl restart api
  │      │     │           └─ command(s) allowed (full paths)
  │      │     └─ "run-as": which user (and :group) it may run as
  │      └─ host the rule applies to (ALL on a single server)
  └─ who: a user, or %group, or a User_Alias
```

Read: "ruwan may run exactly `systemctl restart api` as root."

## 5.3 Tags

Placed before the command:

| Tag | Effect |
|---|---|
| `NOPASSWD:` | Don't prompt for a password for these commands |
| `PASSWD:` | Require a password (the default) |
| `NOEXEC:` | Stop the command from running **other** programs (a guard against shell escapes, where supported) |
| `SETENV:` | Allow the caller to pass environment variables (`sudo VAR=x` / `sudo -E`) |

## 5.4 Grant to groups, and use aliases

The same lesson as Note 3: give rules to **groups**, not individuals.

```text
# A group rule
%ops    ALL = (root) /usr/bin/systemctl restart api, /usr/bin/systemctl restart worker

# Aliases keep big policies readable
User_Alias  ADMINS   = thirana, ruwan
Cmnd_Alias  SERVICES = /usr/bin/systemctl restart api, /usr/bin/systemctl restart worker
Runas_Alias DEPLOYER = deploy

ADMINS  ALL = (ALL) ALL
%devs   ALL = (DEPLOYER) SERVICES
```

## 5.5 The default admin rule

On Ubuntu, `/etc/sudoers` contains:

```text
%sudo   ALL=(ALL:ALL) ALL
```

"Anyone in group `sudo` may run anything as anyone." That's why adding a user to `sudo` (Note 3) gives full root. RHEL uses `%wheel` the same way.

## 5.6 Reading what applies

```bash
sudo -l            # yourself
sudo -l -U ruwan   # another user (as admin)
```
```
User ruwan may run the following commands on app-server-01:
    (root) /usr/bin/systemctl restart api
    (root) /usr/bin/systemctl restart worker
```

**Used in:** Note 8, Steps 2, 3, 4, 6.

---

# Phase 6 — sudo's traps: what "narrow" really means

A rule that *looks* narrow can still be full root. These are the mistakes the security track hunts for.

## 6.1 A wildcard that's wider than it looks

```text
ruwan ALL=(root) /usr/bin/systemctl restart *
```

`*` matches spaces, so `systemctl restart nginx; bash` or `systemctl restart -- --root ...` tricks are possible, and `restart sshd@...` or other units may be reachable. Prefer **exact** commands, or no wildcard at all. If you must, match a tight pattern and know what it allows.

## 6.2 Editors, pagers, interpreters = full root

```text
ruwan ALL=(root) /usr/bin/vim /etc/myapps/api.env     # DON'T
```

Phase 3's shell escape: inside `vim`, `:!sh` is a root shell. The same for `less`, `more`, `nano` (with its spell/execute features), `awk`, `python`, `find`, `tar --to-command`, `systemctl` (it uses a pager — `!sh` from it), and many others. For editing files, use **`sudoedit`** (Phase 7).

## 6.3 Writable command = full root

If the allowed command (or a directory on its path, Note 5) is writable by the user, they replace it with their own script. **The sudo rule is only as safe as the file's permissions.** A rule pointing at `/home/ruwan/bin/deploy.sh` is a root shell if ruwan can edit that script.

## 6.4 `(ALL)` run-as and missing restrictions

```text
ruwan ALL=(ALL) /usr/bin/systemctl restart api
```

`(ALL)` lets him run it as **any** user including root — usually you want `(root)` only. And a command with no argument restriction (`/usr/bin/systemctl` alone) allows `systemctl` to do **anything**: `stop`, `mask`, `edit`, start a root shell via its pager.

## 6.5 `NOPASSWD` trade-off

`NOPASSWD` is convenient and necessary for automation (the `deploy` user, cron). But it means anyone who gets that user's session runs the command with no second check. Use it only for **specific, safe, non-escaping** commands — never for `ALL`.

## 6.6 The test for a good rule

- Exact command, **full path**, **specific arguments**.
- The command **cannot run another program** (no shell, editor, pager, interpreter, or `--exec`-style option).
- The command and its whole path are **writable only by root**.
- Run-as is the **narrowest** user needed, usually `(root)` or a service account.
- Given to a **group/role**, not scattered across people.
- `NOPASSWD` only where automation needs it, and only on safe commands.

> **How this goes wrong:** every item above, inverted. `sudo -l` showing a wildcard, an editor, `(ALL) ALL`, or a user-writable target is an immediate escalation finding.

**Used in:** Note 8, Steps 2, 3, 4, 6.

---

# Phase 7 — sudoedit, environment, and logging

## 7.1 Editing files the safe way: `sudoedit`

```text
%ops ALL=(root) sudoedit /etc/myapps/worker.env
```

The user runs `sudoedit /etc/myapps/worker.env` (or `sudo -e ...`). sudoedit:

1. Copies the file to a temp location,
2. Opens it with the user's **own** editor running as **the user** (not root),
3. Copies it back as root when they save.

So there's **no root editor to escape from**. This is the correct way to let someone edit a root-owned file. Never grant `sudo vim <file>`.

## 7.2 Environment handling

sudo **resets the environment** by default (`env_reset`), keeping only a safe list and using `secure_path` as PATH (Note 2, Phase 5). This blocks `LD_PRELOAD`/`PATH` attacks. The relevant settings:

```text
Defaults env_reset
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults env_keep += "LANG LC_*"
```

`SETENV:` on a rule, or `sudo -E`, lets the caller pass variables through — give it only to trusted rules, never to a limited one, or you reopen the `LD_*` door.

## 7.3 Logging and auditing

- Every sudo use is logged to `/var/log/auth.log` (Ubuntu) / `journalctl`: who, when, which command, from where.
- `Defaults log_output` records full session output to `/var/log/sudo-io/` for high-value rules (`LOG_OUTPUT` tag per command).
- Failed attempts are logged too — useful signal.

```bash
sudo grep sudo /var/log/auth.log | tail
journalctl _COMM=sudo --since today
```

This audit trail is sudo's main advantage over `su` and over SUID binaries: every privileged action is tied to a person.

**Used in:** Note 8, Steps 4, 6.

---

# Phase 8 — Capabilities: root split into pieces

## 8.1 The problem capabilities solve

Root is all-or-nothing. But a web server needs just **one** root power: bind port 80 (ports under 1024 are privileged). Making it SUID-root hands it **every** power to use one. That's the opposite of least privilege.

**Capabilities** divide root's authority into ~40 independent powers. Grant exactly the one needed, nothing more.

## 8.2 Ones you'll meet

| Capability | Grants |
|---|---|
| `CAP_NET_BIND_SERVICE` | Bind ports below 1024 |
| `CAP_NET_RAW` | Raw sockets (what `ping` uses) |
| `CAP_NET_ADMIN` | Configure networking (interfaces, firewall) |
| `CAP_CHOWN` | Change file ownership |
| `CAP_DAC_OVERRIDE` | Bypass file r/w/x checks (Note 5, Phase 12) |
| `CAP_DAC_READ_SEARCH` | Bypass read/search checks |
| `CAP_SETUID` / `CAP_SETGID` | Change the process's UID / GID |
| `CAP_KILL` | Send signals to any process |
| `CAP_SYS_ADMIN` | A huge catch-all, close to full root — avoid relying on it |
| `CAP_SYS_PTRACE` | Inspect/attach to other processes' memory |

Root is now defined as **holding the full capability set**. "Root bypasses checks" = "root has every capability."

## 8.3 How a process gets a capability

Two routes:

1. **File capabilities** — stored on the executable (like SUID, but one specific power). When run, the process gains that capability. No root owner needed.
2. **Ambient/inherited, or granted by a launcher** — systemd can start a service with specific capabilities (`AmbientCapabilities=`), which is cleaner than putting them on the file.

## 8.4 The capability sets (brief)

A file capability is written like `cap_net_bind_service+ep`:

| Letter | Set | Meaning |
|---|---|---|
| `p` | Permitted | The process is **allowed** to use it |
| `e` | Effective | It's **active** right away |
| `i` | Inheritable | Passed to programs this one `exec`s (rarely needed) |

`+ep` (permitted + effective) is the usual combination for "this program has and uses this power."

> **Why prefer capabilities over SUID:** a SUID-root binary that's exploitable gives **all** of root. A binary with only `cap_net_bind_service` that's exploitable gives only the ability to bind low ports. The blast radius is tiny by comparison.

**Used in:** Note 8, Step 1.

---

# Phase 9 — Capabilities in practice: files, processes, services

## 9.1 Inspecting

```bash
getcap /usr/bin/ping
```
```
/usr/bin/ping cap_net_raw=ep
```

```bash
getcap -r / 2>/dev/null        # EVERY file on the system with capabilities
```

This recursive scan is essential for auditing — capabilities **do not show in `ls -l`** (Note 5). A file granting `cap_setuid` or `cap_dac_override` is as powerful as SUID-root and far easier to miss.

Check a running process:

```bash
grep Cap /proc/<PID>/status       # hex bitmaps
getpcaps <PID>                    # human-readable
```

## 9.2 Setting and removing on a file

```bash
sudo setcap 'cap_net_bind_service=+ep' /usr/bin/node   # grant
getcap /usr/bin/node                                   # verify
sudo setcap -r /usr/bin/node                           # remove all
```

Caveat: file capabilities are lost when the file is replaced (a package update, a new deploy). For a service, granting via **systemd** is more durable than `setcap` on the binary (Phase 10).

## 9.3 Granting to a service (preferred)

In a unit file:

```ini
[Service]
User=api
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
```

- `AmbientCapabilities=` **gives** the service exactly this power while running as the unprivileged `api` user.
- `CapabilityBoundingSet=` **caps** the total set it could ever hold — a ceiling, so even a bug can't gain more. Setting it to just what's needed is the hardening move.

This beats `setcap` on the binary: it survives redeploys, is visible in the unit, and pairs with the sandboxing in Phase 10.

> **How this goes wrong:** over-broad capabilities. `cap_sys_admin`, `cap_dac_override`, `cap_setuid` on a file or service are effectively root. A hidden file capability (found only by `getcap -r /`) is a classic persistence trick.

**Used in:** Note 8, Steps 1, 5, 6.

---

# Phase 10 — systemd as a privilege and sandbox tool

Notes 2, 4, and 6 used systemd to run services with the right user, environment, and directories. It's also the main tool for giving a service **less** than its user would normally have — shrinking what a compromise can reach.

## 10.1 Identity and least privilege

| Setting | Effect |
|---|---|
| `User=` / `Group=` | Run as a specific account (Note 2) |
| `DynamicUser=yes` | systemd creates a **temporary** user for the service, gone when it stops — no permanent account to attack |
| `SupplementaryGroups=` | Extra groups (e.g. `appdata`) without editing `/etc/group` |
| `AmbientCapabilities=` / `CapabilityBoundingSet=` | Grant/limit capabilities (Phase 9) |
| `NoNewPrivileges=yes` | The service (and its children) can **never gain privilege** — SUID and file capabilities stop working for it. A strong, cheap win |

## 10.2 Filesystem limits (enforce Note 6's design)

| Setting | Effect |
|---|---|
| `ProtectSystem=strict` | The **whole filesystem is read-only** to the service, except what you allow |
| `ReadWritePaths=` | The few paths it may write (e.g. `/var/lib/api`, `/srv/uploads`) |
| `ReadOnlyPaths=` | Explicitly read-only paths |
| `ProtectHome=yes` | `/home`, `/root`, `/run/user` are hidden |
| `PrivateTmp=yes` | A private `/tmp` — can't see or tamper with others' temp files |
| `StateDirectory=` / `CacheDirectory=` / `LogsDirectory=` / `RuntimeDirectory=` | systemd creates `/var/lib/<x>`, `/var/cache/<x>`, `/var/log/<x>`, `/run/<x>` with the right owner and mode automatically (Note 6, Steps 3, 6) |
| `UMask=0027` | Default umask for the service's files (Note 6, Steps 5, 7) |

This makes "the app can't modify its own code or config" (Note 6) a kernel-enforced fact, not just a permissions convention. Even if the file permissions were wrong, `ProtectSystem=strict` would still stop the write.

## 10.3 Kernel/system limits

| Setting | Effect |
|---|---|
| `ProtectKernelModules=` / `ProtectKernelTunables=` | Can't load modules or change kernel settings |
| `RestrictAddressFamilies=` | Limit socket types (e.g. only `AF_INET AF_INET6`) |
| `SystemCallFilter=` | Allow only certain syscalls (seccomp) |
| `PrivateDevices=yes` | No access to real devices |
| `RestrictSUIDSGID=yes` | The service can't create SUID/SGID files |

## 10.4 Measuring it

```bash
systemd-analyze security api.service
```

Gives each setting a score and an overall exposure rating. It's a checklist: turn on protections until the score is good and the service still works.

> **Why this is the modern answer:** instead of granting privilege (SUID, broad sudo) and hoping the program is careful, you run the program with the **minimum** it needs and let the kernel enforce the box. A compromise then reaches almost nothing.

**Used in:** Note 8, Step 5.

---

# Phase 11 — The full privilege picture

Putting Notes 3–7 together, a permission decision has four layers, and privilege is the fourth:

```
1. Identity (Note 3)       Who is the process? (RUID/EUID + groups)
2. File rules (Note 5)     What do owner/group/other + ACL allow the EUID?
3. File/FS guards (Note 5) Do attributes (+i/+a) or mount options (ro/noexec) forbid it?
4. Privilege (this note)   Does the process have a way around the rules?
      ├─ EUID 0 (root)                → bypasses almost everything
      ├─ a capability (CAP_DAC_*, …)  → bypasses a specific check
      ├─ SUID/SGID on the program     → runs with the owner's/group's EUID
      └─ sudo                          → a person runs a command as another user, per policy
```

The three ways to **grant** privilege, compared:

| Mechanism | Granted to | Scope | Audited | Preferred when |
|---|---|---|---|---|
| **SUID/SGID** | A program (everyone who runs it) | The owner's full identity | No | Almost never now; a few system tools |
| **Capability** | A program or service | One specific power | Via `getcap`/unit | A program needs one root power |
| **sudo** | A person/group | Specific commands as another user | **Yes, per command** | A person needs to run admin tasks |

And the way to **reduce** privilege: systemd sandboxing (Phase 10) — run with less than the account normally has.

The guiding rule across all of them: **least privilege** — the smallest power, for the shortest time, given to the narrowest subject, and logged. Every finding in Topic 2 is this rule broken somewhere.

---

# How this goes wrong — overview

| Misconfiguration | Why it's dangerous | How to find it |
|---|---|---|
| Unexpected SUID/SGID binary | Runs as owner (often root) | `find / -xdev -perm -6000 -type f` vs a known-good list |
| Interpreter/editor/pager SUID | Instant root shell | Same scan; cross-check GTFOBins |
| Custom SUID helper that runs commands | Escalation | Review every non-standard SUID file |
| sudo rule with `*`, `(ALL)`, or no args | Wider than it looks | `sudo -l`, read `/etc/sudoers*` |
| sudo to an editor/pager/interpreter | Shell escape to root | `sudo -l`; look for vim/less/python/etc. |
| sudo target writable by the user | Replace it → root | Check the command's and its path's permissions |
| `NOPASSWD: ALL` | Session = root, no check | `sudo -l` |
| Over-broad capability (`cap_sys_admin`, `cap_dac_override`, `cap_setuid`) | Effectively root | `getcap -r /` |
| Hidden file capability | Persistence, missed by `ls` | `getcap -r /` |
| Service runs as root needlessly | Big blast radius | `systemctl status`, `/proc/PID/status` Uid |
| No `NoNewPrivileges`/sandboxing on exposed services | Compromise reaches whole FS | `systemd-analyze security` |
| `SETENV`/`-E` on a limited rule | `LD_PRELOAD` → root | Read the rule |

---

# Summary table

| Concept | Key point |
|---|---|
| RUID / EUID / SUID | Launcher / acting-as / parked; kernel checks **effective** |
| Dropping privilege | Service starts root, sets EUID to a user, parks or drops root |
| SUID / SGID on a program | Runs with the file's owner / group |
| `passwd` | The correct SUID example — one narrow job |
| Shell escape | A SUID program that can run another command = root shell |
| SUID scripts | Ignored by the kernel on Linux |
| sudo | Person runs a command as another user, per policy, **logged** |
| sudo is SUID-root | One audited gatekeeper instead of many binaries |
| sudoers | `who host=(run-as) commands`; edit with `visudo`; drop-ins in `/etc/sudoers.d` |
| `%sudo ALL=(ALL:ALL) ALL` | Why the `sudo` group = full root |
| Narrow rule | Exact command, full path, fixed args, non-escaping, root-owned, group-granted |
| sudoedit | Edit root files with **no root editor to escape** |
| env_reset / secure_path | sudo blocks `LD_*`/`PATH` attacks |
| Capabilities | ~40 slices of root; grant exactly one |
| File caps | `getcap -r /`; invisible to `ls` |
| systemd caps | `AmbientCapabilities=` grant, `CapabilityBoundingSet=` cap |
| systemd sandbox | `NoNewPrivileges`, `ProtectSystem=strict`, `ReadWritePaths`, `DynamicUser`, `*Directory=` |
| `systemd-analyze security` | Scores a service's exposure |
| Least privilege | Smallest power, shortest time, narrowest subject, logged |

---

# Appendix — Command reference

## Process identity

| Command | Purpose |
|---|---|
| `grep -E '^(Uid\|Gid)' /proc/PID/status` | Real/effective/saved IDs |
| `getpcaps PID` / `grep Cap /proc/PID/status` | A process's capabilities |
| `id` / `whoami` | Current identity |

## SUID / SGID

| Command | Purpose |
|---|---|
| `chmod u+s file` / `chmod 4755 file` | Set SUID |
| `chmod g+s file` / `chmod 2755 file` | Set SGID on a program |
| `find / -xdev -perm -4000 -type f 2>/dev/null` | Find SUID |
| `find / -xdev -perm -2000 -type f 2>/dev/null` | Find SGID |
| `find / -xdev -perm -6000 -type f -ls 2>/dev/null` | Both, detailed |

## sudo

| Command | Purpose |
|---|---|
| `sudo cmd` / `sudo -u user cmd` | Run as root / as a user |
| `sudo -l` / `sudo -l -U user` | List rules for you / another user |
| `sudo -k` / `sudo -v` | Clear / refresh the cached timestamp |
| `sudoedit file` / `sudo -e file` | Edit a root file safely |
| `visudo` / `visudo -f /etc/sudoers.d/name` | Edit policy safely |
| `visudo -c` | Check all sudoers files |
| `grep sudo /var/log/auth.log` / `journalctl _COMM=sudo` | Audit use |

## Capabilities

| Command | Purpose |
|---|---|
| `getcap file` / `getcap -r / 2>/dev/null` | Show file caps / audit all |
| `setcap 'cap_net_bind_service=+ep' file` | Grant a capability |
| `setcap -r file` | Remove all capabilities |

## systemd privilege & sandbox

| Setting / command | Purpose |
|---|---|
| `User=` / `Group=` / `SupplementaryGroups=` | Run-as identity |
| `DynamicUser=yes` | Temporary service user |
| `AmbientCapabilities=` / `CapabilityBoundingSet=` | Grant / cap capabilities |
| `NoNewPrivileges=yes` | Block all privilege gain |
| `ProtectSystem=strict` + `ReadWritePaths=` | Read-only FS except named paths |
| `ProtectHome=yes` / `PrivateTmp=yes` | Hide homes / private /tmp |
| `StateDirectory=` / `CacheDirectory=` / `LogsDirectory=` / `RuntimeDirectory=` | Auto-created owned dirs |
| `UMask=0027` | Service umask |
| `systemd-analyze security SERVICE` | Exposure score |