# L00 · Act I · Privilege Escalation — Know Your Ground

**MITRE tag:** T1082 — System Information Discovery  
**Challenge type:** Local enumeration / leaked-secret discovery  
**Flag:** `bl_e4b030f48d738c0f4ed1297971ea47e9`  
**SSH:** `pe-recon@204.168.229.209 -p 2224`

---

## Challenge brief

> *"You land as a low privilege user with no map. Enumerate the host: SUID bits, sudo rules, capabilities, cron, writable paths. One asset on this box was left the way it never should have been."*

The goal is to find a credential or secret that was left world-readable — something that should have been protected but wasn't. Run `verify` after writing the recovered value to `/tmp/.pe-recon-proof`.

---

## Initial hypothesis

The phrase "left the way it never should have been" points to a misconfigured file — something with permissions that are too open. In a real engagement this usually means a config file, a `.env` file, or a credential file that should be readable only by root or a service account, but was accidentally left world-readable.

The standard sweep for this kind of find: check the user's own files and shell history first (fastest), then look at non-standard directories on the filesystem, then check SUID/capabilities, then do a broad find for interesting filenames.

---

## Enumeration log

### Step 1 — Who are we?

```bash
whoami
id
```

**Result:**
```
operator
uid=1000(operator) gid=1000(operator) groups=1000(operator)
```

**Conclusion:** We are a plain low-privilege user. No wheel group, no docker group, no special privileges. The container is Alpine Linux (hostname looked like `de366dbbec12` — a Docker container ID).

---

### Step 2 — Is sudo available?

```bash
sudo -l
```

**Result:** `sudo: not found`

**Conclusion:** No sudo at all — not just restricted, not installed. This is common in minimal Docker containers. Any privilege escalation path here won't go through sudo.

---

### Step 3 — Check environment variables

```bash
env
```

**Result:** Only standard variables: `HOSTNAME`, `HOME=/home/operator`, `PATH`, `TERM`, `LANG`. No tokens, no passwords, no API keys.

**Conclusion:** Dead end. No secrets leaked into the process environment for this user.

---

### Step 4 — Check PID 1's environment (container init)

In Docker containers, secrets are sometimes injected via `docker run -e SECRET=...`, which means they appear in PID 1's environment even if not in the current shell's env.

```bash
cat /proc/1/environ | tr '\0' '\n'
```

**Result:** `Permission denied`

**Conclusion:** Dead end. We are not root, so we can't read `/proc/1/environ`. This is a common block on Docker challenges — the technique only works if you have root or if the container is running with relaxed proc settings.

> **Note on the syntax:** `/proc/1/environ` stores variables as null-byte separated strings. The `tr '\0' '\n'` converts null bytes to newlines to make it readable. Use single quotes around `\0` and `\n` — backticks here will break the shell.

---

### Step 5 — Check shell history

This is worth checking early because operators sometimes accidentally type credentials directly into the shell, and those end up in history.

```bash
cat ~/.ash_history
cat ~/.bash_history
```

**Result:** `.ash_history` existed (98 bytes) but contained only the commands we ran ourselves during this session — `whoami`, `id`, `sudo -l`, `env`, etc. `.bash_history` was empty.

**Conclusion:** Dead end. The history only reflects our own session. This is an Alpine Linux container so the shell is `ash` (busybox), not bash — `.ash_history` is the relevant file here, not `.bash_history`.

> **Tip:** Before running `cat .ash_history` make sure you are in the home directory or use the full path `~/.ash_history`. Running it from `/` will fail with "no such file" because it looks for `./ash_history` relative to your current directory.

---

### Step 6 — Survey the filesystem for non-standard directories

Standard Linux directories (`/bin`, `/etc`, `/lib`, `/usr`, `/var`) are not interesting on their own. What matters is what's non-standard — anything that looks like an application, a service config, or a CI/CD artifact.

```bash
ls /
ls /opt 2>/dev/null
ls /app 2>/dev/null
ls /srv 2>/dev/null
ls /var/
```

**Result:**
- `/opt` → `acme-ci` ← **non-standard, suspicious**
- `/app` → empty
- `/srv` → empty
- `/var` → standard subdirectories only

**Conclusion:** `/opt/acme-ci` is the only non-standard directory. The name suggests a CI/CD system (Continuous Integration for a company called "Acme"). CI systems often store registry tokens, deploy keys, or API credentials on disk to authenticate against image registries or artifact stores. This is the primary lead.

---

### Step 7 — SUID binary sweep

SUID binaries run as their owner (usually root) regardless of who executes them. Non-standard SUID binaries are a privilege escalation vector — if the binary has a flaw, you can use it to get root access.

```bash
find / -perm -4000 2>/dev/null
```

**Result:**
```
/usr/bin/passwd
/usr/bin/backup       ← non-standard
/usr/bin/chsh
/usr/bin/chage
/usr/bin/chfn
/usr/bin/expiry
/usr/bin/gpasswd
```

**Conclusion:** Most of these are standard Alpine binaries. `/usr/bin/backup` is not — it's a custom SUID binary that doesn't exist in stock Alpine. It was clearly planted. This is not needed to solve Level 0 but is almost certainly the attack surface for a future privilege escalation level. **Noted for later.**

---

### Step 8 — Capabilities sweep

Linux capabilities are a more granular alternative to SUID. A binary with `cap_setuid` can change its own UID without being SUID. Worth checking.

```bash
getcap -r / 2>/dev/null
```

**Result:** Nothing.

**Conclusion:** No capability-based escalation paths.

---

### Step 9 — Check /etc/passwd for other users and context

```bash
cat /etc/passwd
```

**Result:** Only `operator` (uid=1000) is a non-system user. Root is there but has no password entry accessible to us.

**Conclusion:** Single-user box. No other interactive accounts to target for lateral movement at this level.

---

### Step 10 — Investigate /opt/acme-ci

Back to the primary lead.

```bash
ls -la /opt/acme-ci/
```

**Result:**
```
total 12
drwxr-xr-x 2 root root 4096 Oct 16 16:16 .
drwxr-xr-x 1 root root 4096 Oct 16 16:16 ..
-rw-r--r-- 1 root root   32 Oct 16 16:16 .registry-token
```

**Conclusion:** There is one file: `.registry-token`. It is owned by root but the permissions are `rw-r--r--` — world-readable. Anyone on the box can read it. The filename tells us exactly what it is: a token to authenticate against a container image registry (Docker Hub, ECR, GCR, etc.).

---

### Step 11 — Read the token (and the glob trap)

First attempt using a glob:

```bash
cat /opt/acme-ci/*
```

**Result:** Nothing printed.

**Why:** Shell globs (`*`) do not match filenames that start with a dot. `.registry-token` is a hidden file, so `*` skips it entirely. This is a classic mistake and a common CTF trick.

The fix — name the file explicitly:

```bash
cat /opt/acme-ci/.registry-token
```

**Result:** `833e6b30227916b6d3f2a9ebe770ffd9`

---

### Step 12 — Submit the proof and verify

```bash
echo '833e6b30227916b6d3f2a9ebe770ffd9' > /tmp/.pe-recon-proof
verify
```

**Result:**
```
Verifying...
Level solved: pe-recon

Your flag:
bl_e4b030f48d738c0f4ed1297971ea47e9
```

---

## What the vulnerability was

A CI/CD system (`acme-ci`) stored a container registry authentication token in a file at `/opt/acme-ci/.registry-token`. The file was owned by root but had world-readable permissions (`-rw-r--r--`). Any user on the box — including an attacker with only a low-privilege shell — could read it.

In a real environment this token would let an attacker authenticate to the container registry, pull private images (which may contain source code, other secrets, or base images for production systems), and potentially push malicious images to be deployed.

The misconfiguration: the file should have been `chmod 600` (readable only by root) or stored in a secrets manager rather than on disk at all.

---

## Carryover findings

| Finding | Location | Relevance |
|---|---|---|
| Custom SUID binary | `/usr/bin/backup` | Not standard Alpine. Likely a planted privilege escalation vector for a future level. |

---

## Dead ends summary

| What we tried | Why it failed |
|---|---|
| `env` | No secrets in current user's environment |
| `/proc/1/environ` | Permission denied — requires root or relaxed proc settings |
| `~/.ash_history` | Only contained our own session commands, nothing pre-existing |
| `~/.bash_history` | Empty (Alpine uses ash, not bash) |
| `cat /opt/acme-ci/*` | Glob skips dotfiles — missed `.registry-token` |

---

## Command reference

```bash
# Who are we
whoami
id

# Sudo rules (if sudo is installed)
sudo -l

# Current user's environment variables
env

# PID 1 environment (Docker secrets injection) — requires root
cat /proc/1/environ | tr '\0' '\n'

# Shell history (Alpine uses ash)
cat ~/.ash_history
cat ~/.bash_history

# Filesystem survey — look for non-standard directories
ls /opt /app /srv /var 2>/dev/null

# SUID binary sweep
find / -perm -4000 2>/dev/null

# Capabilities sweep
getcap -r / 2>/dev/null

# User list
cat /etc/passwd

# Files owned by current user
find / -user operator -type f 2>/dev/null | grep -v proc

# List directory including hidden files
ls -la /opt/acme-ci/

# Read a hidden file (glob won't work — name it explicitly)
cat /opt/acme-ci/.registry-token

# Submit proof
echo 'TOKEN_VALUE' > /tmp/.pe-recon-proof
verify
```