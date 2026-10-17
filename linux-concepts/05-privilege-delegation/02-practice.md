# Privilege & Delegation — Practice (Note 8)

Granting the exact extra powers the team server needs, and locking down the services so a compromise reaches almost nothing. Each step shows the command, what the result means, and the decision it leads to.

**Theory for this note:** Note 7. Each step points to the phase that explains it ("Why: Note 7, Phase X").

**Continues from:** Note 6 (the folder layout). This is the **final** note — it closes the three items Note 6 left open and ends with a full server audit.

This note also absorbs the earlier standalone sudo notes: the narrow-rule patterns here are the complete version.

---

## Contents

- The setup
- Step 0 — Look at the starting state
- Step 1 — The API on port 80 without root
- Step 2 — Let `deploy` restart services (and nothing else)
- Step 3 — Ops on-call: broader, but still bounded and logged
- Step 4 — Let developers edit one config, safely
- Step 5 — Lock down the services with systemd
- Step 6 — Full privilege audit
- State of the server after Note 8
- Scenario recap
- Troubleshooting table
- Appendix — Command reference

---

# The setup

The server from Notes 4 and 6. Accounts and folders are done. What's left is **privilege**:

| Need | Who | Current gap |
|---|---|---|
| Bind port 80 | `api` service | Would need root |
| Restart `api` / `worker` after a deploy | `deploy` (CI) | No sudo yet |
| Restart services, read logs, occasional admin | `ruwan`, `sachini` (`ops`) | No sudo yet |
| Edit `worker.env` without root shell | `devs` | No safe way yet |
| Limit what a compromised service can reach | `api`, `worker` | Services run with umask 022 and full FS access |

Admin so far is only `thirana` (group `sudo`).

---

# Step 0 — Look at the starting state

### What can each service already do?

```bash
PID=$(systemctl show api -p MainPID --value)
grep -E '^(Uid|Gid|CapEff|NoNewPrivs)' /proc/$PID/status
```
```
Uid:    901 901 901 901
Gid:    901 901 901 901
CapEff: 0000000000000000
NoNewPrivs:     0
```

**Meaning:** the API runs as `api` with **no capabilities** (good) but `NoNewPrivs: 0` — it could still gain privilege through a SUID binary or file capability. We'll fix that in Step 5. (Why: Note 7, Phase 1 and 10.1.)

### Existing SUID/SGID files (baseline)

```bash
sudo find / -xdev -perm -4000 -type f 2>/dev/null | sort
```
```
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/su
/usr/bin/sudo
/usr/bin/mount
/usr/bin/umount
```

**Meaning:** all expected distro tools (Note 7, Phase 2.3). **Save this list** — Step 6 compares against it to catch anything new.

### Existing capabilities

```bash
sudo getcap -r / 2>/dev/null
```
```
/usr/bin/ping cap_net_raw=ep
```

**Meaning:** only `ping`, as expected.

### Who has sudo

```bash
sudo -l -U thirana | tail -2
getent group sudo
```
```
(ALL : ALL) ALL
sudo:x:27:thirana
```

**Conclusion:** clean baseline. Only `thirana` is admin; services are unprivileged but not yet sandboxed.

---

# Step 1 — The API on port 80 without root

The API should answer on port 80. Ports below 1024 need privilege. We do **not** want to run it as root. (Why: Note 7, Phase 8.1.)

### Wrong options, and why

| Option | Why not |
|---|---|
| `User=root` in the unit | The whole app runs as root — huge blast radius |
| `setcap cap_net_bind_service /usr/bin/node` | Works, but **every** Node program on the box gets the power, and it's lost on the next Node package update (Note 7, Phase 9.2) |

### Right option: grant the capability to the service only

Edit the unit with a drop-in:

```bash
sudo systemctl edit api
```

```ini
[Service]
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
```

- `AmbientCapabilities=` gives the running service exactly this one power, while still running as `api`.
- `CapabilityBoundingSet=` caps the total it could ever hold to just this — a ceiling even a bug can't exceed. (Why: Note 7, Phase 9.3.)

Point the app at port 80 (via `/etc/myapps/api.env`, Note 2) and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart api
```

### Verify

```bash
PID=$(systemctl show api -p MainPID --value)
getpcaps $PID
```
```
Capabilities for PID 2890: cap_net_bind_service=ep
```

```bash
ss -ltnp | grep ':80 '
```
```
LISTEN 0 511 *:80 *:* users:(("node",pid=2890,...))
```

```bash
grep -E '^Uid' /proc/$PID/status      # Uid: 901 901 901 901  ← still just 'api'
```

**Conclusion:** the API listens on port 80 while running as the unprivileged `api`, with one capability and no more. If it's ever compromised, the attacker gets "can bind low ports", not root. (Why: Note 7, Phase 8.4.)

---

# Step 2 — Let `deploy` restart services (and nothing else)

In Note 6, `deploy` can write releases and swap the `current` symlink, but **restarting the service needs privilege**. CI runs non-interactively, so the rule must be `NOPASSWD` — which means it has to be airtight. (Why: Note 7, Phase 6.5.)

### Write the rule

```bash
sudo visudo -f /etc/sudoers.d/deploy
```

```text
# deploy (CI/CD) may restart ONLY these two services, no password.
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart api, \
                            /usr/bin/systemctl restart worker
```

Why it's safe (the Phase 6.6 test):

- **Exact commands, full path, fixed arguments** — not `systemctl *`, not bare `systemctl`.
- `systemctl restart api` can't run another program or escape to a shell (unlike bare `systemctl`, which opens a pager you can `!sh` from).
- The target `/usr/bin/systemctl` is root-owned and not writable by `deploy`.
- Run-as is `(root)` only.

`visudo` checks syntax on save. Confirm:

```bash
sudo visudo -c
```
```
/etc/sudoers.d/deploy: parsed OK
```

### Test

```bash
sudo -l -U deploy
```
```
User deploy may run the following commands on app-server-01:
    (root) NOPASSWD: /usr/bin/systemctl restart api, /usr/bin/systemctl restart worker
```

As deploy:

```bash
sudo systemctl restart api          # works, no password
sudo systemctl stop api             # Sorry, user deploy is not allowed to execute '/usr/bin/systemctl stop api' as root
sudo systemctl restart nginx        # denied
sudo systemctl                      # denied (bare command not in the rule)
```

**Meaning:** deploy can do exactly the one job and nothing adjacent. Even `stop` is refused. (Why: Note 7, Phase 6.)

### Common trap avoided

If the rule had been `NOPASSWD: /usr/bin/systemctl restart *`, then `sudo systemctl restart "api; bash"`-style tricks or reaching other units could be possible. Exact arguments close that off. (Why: Note 7, Phase 6.1.)

---

# Step 3 — Ops on-call: broader, but still bounded and logged

The `ops` group (ruwan, sachini) needs more than deploy: restart both services, view their status and logs, and reload nginx. They're people, so this rule **requires a password** and is **logged** per command. (Why: Note 7, Phase 4.1, 7.3.)

### Use aliases for readability

```bash
sudo visudo -f /etc/sudoers.d/ops
```

```text
# Command groups
Cmnd_Alias APP_SVC   = /usr/bin/systemctl restart api, \
                       /usr/bin/systemctl restart worker, \
                       /usr/bin/systemctl reload nginx
Cmnd_Alias APP_STATE = /usr/bin/systemctl status api, \
                       /usr/bin/systemctl status worker, \
                       /usr/bin/systemctl is-active *, \
                       /usr/bin/journalctl -u api, \
                       /usr/bin/journalctl -u worker

# ops may run these as root, with a password
%ops ALL=(root) APP_SVC, APP_STATE
```

Note: `journalctl -u api` (with a fixed unit) doesn't open an interactive pager in a way that allows a shell when run this way; still, for safety you can add `--no-pager` in automation. Avoid granting bare `journalctl` or `systemctl`, which **do** open a pager you can escape from. (Why: Note 7, Phase 6.2.)

### Test

```bash
sudo -l -U ruwan
```

As ruwan (after entering his password once):

```bash
sudo systemctl restart worker        # works
sudo systemctl status api            # works
sudo systemctl stop api              # denied — stop isn't in the alias
sudo journalctl -u api               # works
sudo journalctl                      # denied — bare journalctl not allowed
sudo cat /etc/shadow                 # denied
```

### Check the log

```bash
sudo grep ruwan /var/log/auth.log | grep COMMAND | tail -2
```
```
sudo: ruwan : TTY=pts/0 ; PWD=/home/ruwan ; USER=root ; COMMAND=/usr/bin/systemctl restart worker
```

**Meaning:** every privileged action is tied to ruwan, with time and command. That audit trail is the reason to use sudo rather than a shared root password. (Why: Note 7, Phase 7.3.)

---

# Step 4 — Let developers edit one config, safely

Developers occasionally need to change `/etc/myapps/worker.env` (owned `root:worker`, Note 6). The tempting rule is wrong:

```text
%devs ALL=(root) /usr/bin/vim /etc/myapps/worker.env     # DON'T
```

Inside vim, `:!sh` is a **root shell** — full escalation. (Why: Note 7, Phase 6.2.)

### Right way: `sudoedit`

```bash
sudo visudo -f /etc/sudoers.d/devs-config
```

```text
# devs may edit these config files with their own editor, running as themselves.
%devs ALL=(root) sudoedit /etc/myapps/worker.env, \
                 sudoedit /etc/myapps/common.env
```

The developer sets their editor and runs sudoedit:

```bash
export EDITOR=nano          # their own choice
sudoedit /etc/myapps/worker.env
```

What happens (Note 7, Phase 7.1):
1. sudo copies the file to a temp path,
2. opens it with **nano running as the developer**, not root,
3. writes it back as root on save.

There's **no root process to escape from**.

### Test

```bash
sudo -l -U nimal
```
```
(root) sudoedit /etc/myapps/worker.env, sudoedit /etc/myapps/common.env
```

```bash
sudoedit /etc/myapps/worker.env     # opens, edits, saves as root:worker — works
sudoedit /etc/myapps/api.env        # denied — not in the rule
sudo vim /etc/myapps/worker.env     # denied — no vim rule at all
```

Confirm the saved file kept its ownership and mode:

```bash
stat -c '%a %U:%G' /etc/myapps/worker.env
```
```
640 root:worker
```

**Meaning:** developers can edit exactly two files, with any editor, and can't turn it into root access. The file keeps its secure ownership. (Why: Note 7, Phase 7.1.)

### After editing, the change needs a restart

Editing the file doesn't change the running worker (Note 2: env is read at start). ops restart it (Step 3), or the worker is restarted by CI. Developers don't get a restart rule — that stays with ops/deploy.

---

# Step 5 — Lock down the services with systemd

The services run as unprivileged users, but a compromise could still read much of the filesystem, write wherever permissions allow, and try to gain privilege. systemd can shrink all of that to almost nothing — enforcing Note 6's design at the kernel level. (Why: Note 7, Phase 10.)

### Measure first

```bash
systemd-analyze security api
```
```
→ Overall exposure level for api.service: 8.2 UNSAFE 😨
```

**Meaning:** wide open. Each line of the report is a protection that's off. Turn them on one at a time, restarting and testing after each.

### Harden the API unit

```bash
sudo systemctl edit api
```

```ini
[Service]
# --- privilege ---
NoNewPrivileges=yes
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
RestrictSUIDSGID=yes

# --- filesystem (Note 6's layout, enforced) ---
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/var/cache/api /var/log/myapps/api /srv/uploads
StateDirectory=api
CacheDirectory=api
LogsDirectory=myapps/api
UMask=0027

# --- kernel / network ---
ProtectKernelModules=yes
ProtectKernelTunables=yes
ProtectControlGroups=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
SystemCallFilter=@system-service
SystemCallErrorNumber=EPERM
```

What the key lines do (Note 7, Phase 10.2):

- `NoNewPrivileges=yes` — the service can never gain privilege; combined with `RestrictSUIDSGID=yes`, SUID binaries and file capabilities simply stop working for it.
- `ProtectSystem=strict` — the **entire filesystem is read-only** to the API, except the three `ReadWritePaths`. Even if a permission somewhere were wrong, the write is blocked.
- `ReadWritePaths=` — exactly the folders from Note 6 the API writes: its cache, its logs, uploads.
- `UMask=0027` — the extra layer promised in Note 6, Step 5: the API's new files are never readable by "other".

### Interaction to watch: capabilities + NoNewPrivileges

`AmbientCapabilities=CAP_NET_BIND_SERVICE` still works with `NoNewPrivileges=yes`, because systemd grants the capability at start — the service isn't *gaining* it later. But the capability must also be inside `CapabilityBoundingSet=`, or it's dropped. We set both (Step 1), so port 80 keeps working.

### Restart and test the real workflow

```bash
sudo systemctl daemon-reload
sudo systemctl restart api
systemctl status api            # active (running)?
```

Then exercise the app: hit port 80, upload a file (writes `/srv/uploads`), check logs appear. If something breaks, the report names the cause — usually a path that needs adding to `ReadWritePaths`.

```bash
sudo journalctl -u api -n 30
```

A typical failure and fix:

```
Error: EACCES: permission denied, open '/var/lib/api/sessions.db'
```
→ add `/var/lib/api` to `ReadWritePaths` (or rely on `StateDirectory=api`, which creates and allows `/var/lib/api`). Here `StateDirectory=api` already covers it, so no change needed.

### Confirm the lockdown

```bash
PID=$(systemctl show api -p MainPID --value)
grep -E '^(CapEff|NoNewPrivs)' /proc/$PID/status
```
```
CapEff: 0000000000000400       ← only cap_net_bind_service
NoNewPrivs:     1
```

Prove the filesystem is read-only to the service:

```bash
sudo -u api touch /etc/test                 # Permission denied (ordinary: not owner)
sudo systemd-run -p ProtectSystem=strict --uid=api touch /etc/test   # Read-only file system
```

Re-score:

```bash
systemd-analyze security api
```
```
→ Overall exposure level for api.service: 2.9 OK 🙂
```

### Do the same for the worker

The worker has no port to bind, so it needs **no capabilities at all** — set the bounding set empty:

```ini
[Service]
NoNewPrivileges=yes
CapabilityBoundingSet=
RestrictSUIDSGID=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/var/lib/worker /var/log/myapps/api /var/backups/appdb /srv/uploads
StateDirectory=worker
UMask=0027
ProtectKernelModules=yes
ProtectKernelTunables=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
SystemCallFilter=@system-service
```

`CapabilityBoundingSet=` (empty) means the worker can hold **no** capability ever — the strongest setting, right for a background job. (Why: Note 7, Phase 9.3.)

**Conclusion:** both services now run as unprivileged users, with only the capabilities they need, unable to gain privilege, and able to write only a handful of named folders. A compromise of either reaches almost nothing. This is Note 6's design turned into kernel-enforced rules. (Why: Note 7, Phase 10.2.)

---

# Step 6 — Full privilege audit

One script to check every privilege mechanism from Notes 3–7. Each check should print nothing or only the known-good baseline. This is what the security track runs first on any server.

`/root/provision/audit-privilege.sh`:

```bash
#!/bin/bash
# Review every privilege mechanism. Run as root. Investigate anything unexpected.
echo "=== 1. UID 0 accounts (expect: root only) ==="
awk -F: '$3 == 0 {print $1}' /etc/passwd

echo "=== 2. SUID/SGID files vs baseline ==="
find / -xdev -perm -6000 -type f 2>/dev/null | sort > /tmp/suid.now
diff /root/provision/suid-baseline.txt /tmp/suid.now && echo "  (no changes)"

echo "=== 3. File capabilities (expect: /usr/bin/ping only) ==="
getcap -r / 2>/dev/null

echo "=== 4. All sudo rules ==="
for u in $(getent passwd | awk -F: '$3>=1000 && $3<60000 {print $1}') root; do
  rules=$(sudo -l -U "$u" 2>/dev/null | grep -A50 'may run' | grep -v 'may run')
  [ -n "$rules" ] && echo "-- $u --" && echo "$rules"
done

echo "=== 5. Dangerous sudo patterns ==="
grep -rE '(\(ALL\).*ALL|NOPASSWD:.*ALL|/vim|/vi |/nano|/less|/more|/man|/python|/perl|/awk|/find|ALL *$|\*)' \
  /etc/sudoers /etc/sudoers.d/ 2>/dev/null | grep -v '^\s*#'

echo "=== 6. Services running as root ==="
systemctl show '*.service' -p MainPID -p User --value 2>/dev/null >/dev/null
for svc in $(systemctl list-units --type=service --state=running --no-legend | awk '{print $1}'); do
  u=$(systemctl show "$svc" -p User --value)
  echo "${u:-root}  $svc"
done | grep -E '^(root|)  ' | sort

echo "=== 7. Service hardening scores (our apps) ==="
for svc in api worker; do
  printf '%s: ' "$svc"; systemd-analyze security "$svc" 2>/dev/null | tail -1
done

echo "=== 8. World-writable and orphaned (from Note 6) ==="
find / -xdev -type f -perm -0002 2>/dev/null
find / -xdev \( -nouser -o -nogroup \) 2>/dev/null
```

Create the SUID baseline once, from the known-good Step 0 list:

```bash
sudo find / -xdev -perm -6000 -type f 2>/dev/null | sort | sudo tee /root/provision/suid-baseline.txt
```

Run it:

```bash
sudo bash /root/provision/audit-privilege.sh
```

### Reading the output

| Section | Good result | Investigate if… |
|---|---|---|
| 1 | Only `root` | Any other name → hidden root account (Note 3) |
| 2 | "(no changes)" | A new SUID file appears → why? who put it? (Note 7, Phase 3) |
| 3 | `/usr/bin/ping` only | Any file with `cap_setuid`, `cap_dac_override`, `cap_sys_admin` (Note 7, Phase 9) |
| 4 | The rules from Steps 2–4 | Any rule you didn't write |
| 5 | Empty | A match = a wildcard, editor, or `ALL` in a rule (Note 7, Phase 6) |
| 6 | Only system services as root | An app service as root that shouldn't be |
| 7 | `api`/`worker` OK | A regression after a unit change |
| 8 | Empty | World-writable or orphaned files (Note 6) |

**Conclusion:** this one script covers identity (Note 3), permissions (Note 5), and privilege (Note 7). Run it after any change and on a schedule. A clean run means the four layers from Note 7, Phase 11 are all narrow.

---

# State of the server after Note 8

The build is complete. Final privilege picture:

| Subject | Privilege | Granted by |
|---|---|---|
| `api` service | Runs as `api`; `cap_net_bind_service` only; no-new-privs; FS read-only except cache/logs/uploads | Capability + systemd sandbox |
| `worker` service | Runs as `worker`; **no** capabilities; no-new-privs; FS read-only except its dirs | systemd sandbox |
| `deploy` | `systemctl restart api/worker` only, no password | Narrow sudo rule |
| `%ops` | Restart services, reload nginx, view status/logs, with password, logged | sudo rule with aliases |
| `%devs` | `sudoedit` two config files | sudoedit rule |
| `thirana` | Full admin | group `sudo` |
| Everyone else | No privilege | — |

Every grant is the smallest that works, given to a role, and (for people) logged.

### The whole series

| Note | Layer |
|---|---|
| 1 Shell | How commands run |
| 2 Environment | What processes carry |
| 3 / 4 Users & Groups | **Who** a process is |
| 5 / 6 Files & Permissions | **What** a file allows |
| 7 / 8 Privilege | The ways **around** the rules, granted narrowly |

Together they're the full access model: identity decides which rules apply, the rules decide what's allowed, and privilege is the controlled set of exceptions. For the security track (Topic 2), every finding is one of these set too loosely — and now the correct, narrow version of each is written down.

---

# Scenario recap

| Step | What happened | Concept (Note 7) |
|---|---|---|
| 0 | Baseline: service caps, SUID list, sudo | RUID/EUID, SUID audit (1, 2) |
| 1 | API on port 80 without root | Capability via systemd, not SUID or `setcap` on the binary (8, 9.3) |
| 2 | deploy restarts services, NOPASSWD | Narrow rule: exact command, no escape, root-owned target (6) |
| 3 | ops: more commands, password, logged | Group rule, aliases, audit trail (4, 5, 7.3) |
| 4 | devs edit a config safely | sudoedit, never `sudo vim` (6.2, 7.1) |
| 5 | Services sandboxed | NoNewPrivileges, ProtectSystem, ReadWritePaths, caps (10) |
| 6 | One audit script for everything | The full privilege picture (11) |

---

# Troubleshooting table

| Symptom | Likely cause | Check | Fix |
|---|---|---|---|
| Service can't bind port 80 after hardening | Capability not in the bounding set | `getpcaps PID` | Add it to both `AmbientCapabilities=` and `CapabilityBoundingSet=` |
| `setcap` capability vanished | File replaced by update/deploy | `getcap file` | Grant via systemd `AmbientCapabilities=` instead |
| Service fails after `ProtectSystem=strict` | A write path isn't allowed | `journalctl -u svc` for EACCES/"Read-only file system" | Add the path to `ReadWritePaths=` or use `StateDirectory=` |
| `sudo` rule "parsed OK" but command denied | Argument/path doesn't match exactly | `sudo -l -U user` | Make the rule's command match the real invocation (full path, args) |
| `visudo` won't save | Syntax error | message from `visudo` | Fix the line; never bypass with a plain editor |
| sudo to a service works but also gives a shell | Command opens a pager/editor | Try the command, press `!`/`:!` | Use a fixed sub-command (`status X`), add `--no-pager`, or avoid it |
| `sudo -E` / env var ignored | `env_reset` (expected) | `sudo -l` for `SETENV` | Pass explicitly; grant `SETENV` only to trusted rules |
| `NoNewPrivileges` broke sudo/ping inside a service | Those need to gain privilege | `/proc/PID/status` NoNewPrivs | Expected; don't call SUID tools from a hardened service |
| Capability on a file but process doesn't have it | `NoNewPrivileges` or not in bounding set | `getpcaps PID` | Grant via the unit; check bounding set |
| New SUID file in audit | Package, or something planted | `dpkg -S file` / check mtime, owner | Verify it's legitimate; remove if not |
| `systemd-analyze security` score worse after change | A protection got turned off | compare unit to Step 5 | Restore the setting |

---

# Appendix — Command reference

## Baseline & audit

| Command | Purpose |
|---|---|
| `find / -xdev -perm -6000 -type f 2>/dev/null \| sort` | SUID+SGID list (save as baseline) |
| `diff baseline.txt <(find ... )` | Detect new SUID/SGID files |
| `getcap -r / 2>/dev/null` | All file capabilities |
| `awk -F: '$3==0' /etc/passwd` | UID 0 accounts |
| `sudo -l -U user` | A user's sudo rules |
| `systemd-analyze security SERVICE` | Service exposure score |
| `getpcaps PID` / `grep -E '^(CapEff\|NoNewPrivs)' /proc/PID/status` | A process's live privilege |

## Capabilities

| Command | Purpose |
|---|---|
| `setcap 'cap_net_bind_service=+ep' file` | Grant on a file (avoid for services) |
| `setcap -r file` | Remove file capabilities |
| `AmbientCapabilities=CAP_...` | Grant to a service |
| `CapabilityBoundingSet=CAP_...` / `=` (empty) | Cap / forbid all capabilities |

## sudo policy

| Command / line | Purpose |
|---|---|
| `visudo -f /etc/sudoers.d/name` | Edit a drop-in safely |
| `visudo -c` | Check all sudoers files |
| `user ALL=(root) NOPASSWD: /full/path arg` | Narrow automation rule |
| `%group ALL=(root) Cmnd_Alias` | Group rule with an alias |
| `%group ALL=(root) sudoedit /path` | Safe file editing |
| `Cmnd_Alias NAME = cmd1, cmd2` | Group commands |
| `sudo -l` / `sudo -k` | List rules / clear timestamp |
| `grep sudo /var/log/auth.log` | Audit usage |

## systemd hardening

| Setting | Purpose |
|---|---|
| `NoNewPrivileges=yes` | Block all privilege gain |
| `RestrictSUIDSGID=yes` | Can't create SUID/SGID files |
| `ProtectSystem=strict` + `ReadWritePaths=` | Read-only FS except named paths |
| `ProtectHome=yes` / `PrivateTmp=yes` | Hide homes / private tmp |
| `StateDirectory=` / `CacheDirectory=` / `LogsDirectory=` | Auto-created owned dirs |
| `UMask=0027` | Service file default mode |
| `RestrictAddressFamilies=` / `SystemCallFilter=@system-service` | Limit sockets / syscalls |
| `systemctl edit SERVICE` / `daemon-reload` | Add a drop-in / reload units |