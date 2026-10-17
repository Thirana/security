# Users, Groups & Identity — Theory (Note 3)

How Linux knows **who** is doing something, where that information is stored, and how accounts are created, changed, locked, and removed properly.

**Series:**
- Note 1: Shell & Processes
- Note 2: Environment Variables
- **Note 3: Users, Groups & Identity — Theory** ← this note
- Note 4: Users, Groups & Identity — Practice
- Note 5 / 6: Files, Directories & Permissions — Theory / Practice
- Note 7 / 8: Privilege & Delegation — Theory / Practice

Every permission check in Notes 5–8 starts with one question: *who is asking?* This note answers that question completely.

Each phase ends with **"Used in"** (where Note 4 applies it) and, where relevant, a **"How this goes wrong"** box — the common misconfiguration, for the security track.

---

## Contents

1. Phase 1 — Identity is numbers
2. Phase 2 — The account database files
3. Phase 3 — Account types and ID ranges
4. Phase 4 — Groups: primary, supplementary, and role groups
5. Phase 5 — Creating and changing accounts
6. Phase 6 — Passwords, aging, locking, and expiry
7. Phase 7 — Process identity and sessions
8. Phase 8 — Switching identity and seeing who did what
9. Phase 9 — Removing accounts and orphaned files
10. Phase 10 — Consistent IDs across servers and containers
11. How this goes wrong — overview
12. Summary table
13. Appendix — Command reference

---

# Phase 1 — Identity is numbers

## 1.1 UID and GID

The kernel doesn't know names. It knows numbers.

| Term | Meaning |
|---|---|
| **UID** (user ID) | A number identifying a user |
| **GID** (group ID) | A number identifying a group |
| **Username / group name** | A human-friendly label, looked up from a file to get the number |

```bash
id
```
```
uid=1000(thirana) gid=1000(thirana) groups=1000(thirana),27(sudo),4(adm)
```

Read it as: user number 1000 (named `thirana`), primary group 1000, and member of groups 27 (`sudo`) and 4 (`adm`).

## 1.2 Files store numbers, not names

When a file is created, the kernel writes the creator's **UID and GID numbers** into the file's metadata. `ls -l` then looks up the names to show you.

```bash
ls -l report.txt      # -rw-r--r-- 1 thirana thirana 120 ... report.txt
ls -n report.txt      # -rw-r--r-- 1 1000    1000    120 ... report.txt
```

`ls -n` shows the raw numbers. This matters in three real situations:

1. **A user is deleted** → their files still have the old UID. `ls -l` shows a bare number, because there's no name for it any more (Phase 9).
2. **A new user gets the same UID** → they silently own all those old files (Phase 9).
3. **Two servers or a container use different numbers for the same name** → files look owned by the wrong person (Phase 10).

## 1.3 UID 0 is root

UID 0 is special. The kernel skips nearly every permission check for it. The **name** `root` doesn't matter — any account with UID 0 is root.

```bash
awk -F: '$3 == 0' /etc/passwd     # list every UID-0 account; should be only root
```

> **How this goes wrong:** an extra UID-0 account (e.g. `backup:x:0:0:...`) is a classic hidden backdoor. It looks like a normal user in lists sorted by name, but has full root power.

**Used in:** Note 4, Step 0 (checking the starting state).

---

# Phase 2 — The account database files

## 2.1 `/etc/passwd` — the accounts

One line per account, 7 fields separated by `:`.

```
nimal:x:1101:1101:Nimal Perera:/home/nimal:/bin/bash
  │   │  │    │       │            │          │
  │   │  │    │       │            │          └─ 7. login shell
  │   │  │    │       │            └─ 6. home folder
  │   │  │    │       └─ 5. comment (full name, "GECOS" field)
  │   │  │    └─ 4. primary GID
  │   │  └─ 3. UID
  │   └─ 2. password placeholder ("x" = real hash is in /etc/shadow)
  └─ 1. username
```

- **World-readable** (`644`), because many programs need to turn names into numbers.
- The **shell** field decides if the account can log in interactively:
  - `/bin/bash`, `/bin/sh`, `/bin/zsh` — real login
  - `/usr/sbin/nologin` — prints "This account is currently not available" and exits
  - `/bin/false` — exits silently

## 2.2 `/etc/shadow` — passwords and aging

One line per account, 9 fields.

```
nimal:$y$j9T$abc...$xyz...:20378:0:90:7:14:20453:
  │          │               │   │  │ │  │    │
  │          │               │   │  │ │  │    └─ 8. account expiry date
  │          │               │   │  │ │  └─ 7. inactive days after password expires
  │          │               │   │  │ └─ 6. warning days before password expires
  │          │               │   │  └─ 5. max days a password is valid
  │          │               │   └─ 4. min days between password changes
  │          │               └─ 3. last password change
  │          └─ 2. password hash
  └─ 1. username
```

Dates are **days since 1 Jan 1970**. Use `chage -l user` to see them as real dates.

The **hash field** tells you a lot:

| Hash field | Meaning |
|---|---|
| `$y$...` | yescrypt hash (default on new Ubuntu/Debian) |
| `$6$...` | SHA-512 hash |
| `!` or `*` alone | No password set — password login impossible |
| `!$y$...` | A real hash with `!` in front = **password locked** (Phase 6) |
| *(empty)* | **No password needed at all** — very dangerous |

- **Readable only by root and the `shadow` group** (`640 root:shadow`).
- Splitting accounts (public) from hashes (private) is itself a security design. Long ago, hashes were in `/etc/passwd`, readable by everyone.

## 2.3 `/etc/group` — the groups

```
devs:x:2001:nimal,sachini
  │  │  │      │
  │  │  │      └─ 4. supplementary members (comma-separated)
  │  │  └─ 3. GID
  │  └─ 2. password placeholder
  └─ 1. group name
```

**Important:** this member list shows only **supplementary** members. A user whose **primary** group is `devs` (field 4 in `/etc/passwd`) is also in it, but does **not** appear here.

## 2.4 `/etc/gshadow`

Group passwords and group administrators. Rarely used today. Readable only by root and `shadow`.

## 2.5 Don't read the files directly — use `getent`

On many servers, accounts don't come only from these files. They can also come from LDAP, Active Directory (via SSSD), or a cloud identity service (e.g. **GCP OS Login**). `/etc/nsswitch.conf` decides the sources.

`getent` asks through all configured sources, the same way the system does:

```bash
getent passwd nimal      # nimal's account line
getent group devs        # the devs group
getent passwd            # all accounts from all sources
sudo getent shadow nimal # shadow entry (root only)
```

`grep nimal /etc/passwd` may miss users from LDAP or OS Login. `getent` won't.

## 2.6 Editing safely

Never hand-edit these files with a normal editor while the system is running. Use:

- The tools: `useradd`, `usermod`, `groupadd`, `gpasswd`, `passwd`, `chage` (Phase 5–6)
- If you really must edit by hand: `sudo vipw` and `sudo vigr` (they lock the files and check them)
- Consistency checks: `sudo pwck -r` and `sudo grpck -r` (read-only check)

> **How this goes wrong:**
> - `/etc/passwd` or `/etc/shadow` writable by non-root → anyone can add a UID-0 user or remove root's password. Check: `ls -l /etc/passwd /etc/shadow /etc/group`.
> - Empty hash field → login with no password. Check: `sudo awk -F: '$2 == ""' /etc/shadow`.

**Used in:** Note 4, Steps 0, 3, 7.

---

# Phase 3 — Account types and ID ranges

## 3.1 Types of accounts

| Type | Purpose | Typical UID | Shell | Password |
|---|---|---|---|---|
| **root** | Full admin | 0 | `/bin/bash` | Usually locked; use sudo instead |
| **System / service** | Runs one service (e.g. `www-data`, `postgres`, your `api`) | 1–999 | `/usr/sbin/nologin` | None |
| **Human** | A real person | 1000+ | `/bin/bash` | Optional (SSH keys preferred) |
| **Machine login** | A non-person that must log in (CI/CD `deploy`) | 1000+ (often a set range) | `/bin/bash` | None, key only |
| **nobody** | "No one"; used for unmapped IDs and some low-trust work | 65534 | `nologin` | None |

## 3.2 Why service accounts exist

Each service runs as **its own limited user** instead of root. If the service is broken into, the attacker gets only what that user can touch — not the whole server. This is the **least privilege** idea, and it's why Note 2 ran the API as `api` and the worker as `worker`.

Two service accounts should **not share** a user. Shared users mean shared damage and no way to tell their actions apart.

## 3.3 Where the ranges come from: `/etc/login.defs`

```bash
grep -E '^(UID|GID|SYS_UID|SYS_GID)_(MIN|MAX)|^UMASK|^HOME_MODE|^USERGROUPS_ENAB|^ENCRYPT_METHOD|^PASS_' /etc/login.defs
```
```
PASS_MAX_DAYS   99999
PASS_MIN_DAYS   0
PASS_WARN_AGE   7
UID_MIN          1000
UID_MAX         60000
SYS_UID_MIN       100
SYS_UID_MAX       999
GID_MIN          1000
GID_MAX         60000
UMASK           022
HOME_MODE       0750
USERGROUPS_ENAB yes
ENCRYPT_METHOD  YESCRYPT
```

| Setting | Effect |
|---|---|
| `UID_MIN` / `UID_MAX` | Range `useradd` uses for normal accounts |
| `SYS_UID_MIN` / `SYS_UID_MAX` | Range used with `useradd --system` |
| `HOME_MODE` | Permissions of new home folders (`0750` on newer Ubuntu; older ones used `0755`) |
| `UMASK` | Default permission mask (Note 5) |
| `USERGROUPS_ENAB yes` | Each new user gets their own group with the same name (Phase 4.3) |
| `PASS_*` | Default password aging for **new** accounts |
| `ENCRYPT_METHOD` | Hash algorithm for new passwords |

Values differ a little between distributions. Always check your own server.

## 3.4 `/etc/skel` — the starter home folder

When a home folder is created with `useradd -m`, the contents of `/etc/skel` are copied into it.

```bash
ls -la /etc/skel
```
```
.bash_logout  .bashrc  .profile
```

Put company defaults here (a standard `.bashrc`, a README) so every new user gets them. Changing `/etc/skel` doesn't affect existing users.

> **How this goes wrong:** service accounts with a real shell and a password. If the password leaks or is guessed, someone can log in interactively as the service. Check: `getent passwd | awk -F: '$3 < 1000 && $7 !~ /(nologin|false)$/'`.

**Used in:** Note 4, Steps 0, 3, 5, 6.

---

# Phase 4 — Groups: primary, supplementary, and role groups

## 4.1 Primary group

Every user has exactly **one primary group** (the GID in `/etc/passwd`).

When the user creates a file, the file's group is set to their **current primary group** (unless the folder has the SGID bit — Note 5).

## 4.2 Supplementary groups

A user can be in **many** other groups. All of them count when the kernel checks group permission on a file.

```bash
id nimal
# uid=1101(nimal) gid=1101(nimal) groups=1101(nimal),2001(devs),2004(ssh-users)
#                 └ primary        └ all groups, including primary
```

## 4.3 User private groups (UPG)

With `USERGROUPS_ENAB yes`, each new user gets a group with the **same name and usually the same number** (`nimal` user → `nimal` group). Only that user is in it.

Why this design:
- A user's personal files are group-owned by a group containing only them, so "group" permission on their files doesn't accidentally share with anyone.
- This makes it safe to use a more open umask like `002` for team work (Note 5).

## 4.4 Role groups — grant access to groups, not people

The industry pattern:

```
people  →  role groups  →  access
nimal   ─┐
sachini ─┴→  devs   →  read/write /srv/projects, read app logs
ruwan   ───→  ops   →  restart services (sudo rule), read all logs
```

Rules:
- **Give access to groups, then put people in groups.** Never give a file or sudo rule to a single person if a role fits.
- When someone joins, add them to the right groups. When they change role or leave, remove them. Nothing else needs to change.
- Name groups by **role or resource** (`devs`, `ops`, `appdata`, `ssh-users`), not by person.

## 4.5 Special groups on Ubuntu/Debian

Some built-in groups give real power. Know them before adding anyone.

| Group | Members can | Risk level |
|---|---|---|
| `sudo` (Ubuntu) / `wheel` (RHEL) | Run anything as root (with default sudoers) | **Root** |
| `docker` | Control Docker, e.g. start a container that mounts the host's `/` as root | **Root-equivalent** |
| `lxd` | Same idea with LXD containers | **Root-equivalent** |
| `disk` | Read/write raw disks directly (bypass all file permissions) | **Root-equivalent** |
| `shadow` | Read `/etc/shadow` (password hashes) | High |
| `adm` | Read most logs in `/var/log` (may contain sensitive data) | Medium |
| `systemd-journal` | Read the full system journal (`journalctl`) | Medium |
| `www-data` | Act as the web server user | Depends on what it owns |

```bash
getent group sudo docker lxd disk shadow adm
```

> **How this goes wrong:** adding a developer to `docker` "so they don't need sudo" actually gives them root. Treat membership in root-equivalent groups exactly like sudo access.

**Used in:** Note 4, Steps 1, 2, 9, 10.

---

# Phase 5 — Creating and changing accounts

## 5.1 `useradd` vs `adduser`

| | `useradd` | `adduser` (Debian/Ubuntu) |
|---|---|---|
| What | The low-level tool, same on all distros | A friendly wrapper around `useradd` |
| Style | Non-interactive; does exactly what options say | Interactive; asks questions, sets password |
| Defaults | Minimal — **no home folder unless `-m`**, shell `/bin/sh` unless `-s` | Creates home, uses `/bin/bash`, sensible defaults |
| Best for | Scripts, automation, exact control | Quick manual creation |

On RHEL/Fedora, `adduser` is just another name for `useradd`.

Rule: **in scripts, use `useradd` with every option spelled out.** That way the result doesn't depend on hidden defaults.

## 5.2 `useradd` options

| Option | Meaning |
|---|---|
| `-m` | Create the home folder (copies `/etc/skel`) |
| `-M` | Do **not** create a home folder |
| `-d /path` | Home folder path |
| `-s /bin/bash` | Login shell |
| `-u 1101` | Fixed UID |
| `-g group` | Primary group (must exist) |
| `-G a,b` | Supplementary groups |
| `-U` / `-N` | Create / don't create a user private group |
| `-c "Full Name"` | Comment |
| `-e 2026-12-31` | Account expiry date |
| `-r` / `--system` | System account (system UID range, no aging, no home by default) |

Other defaults come from `/etc/default/useradd`:

```bash
useradd -D       # show defaults
```

## 5.3 `usermod` — change an existing account

| Command | Effect |
|---|---|
| `usermod -aG devs nimal` | **Add** nimal to `devs` (keep other groups) |
| `usermod -G devs nimal` | **Replace** all supplementary groups with only `devs` |
| `usermod -s /usr/sbin/nologin api` | Change shell |
| `usermod -d /var/lib/api -m api` | Change home (and `-m` moves the contents) |
| `usermod -l newname oldname` | Rename the account (UID unchanged) |
| `usermod -u 901 api` | Change UID (files in home are updated; others aren't) |
| `usermod -c "New comment" user` | Change comment |
| `usermod -L` / `-U` | Lock / unlock password (Phase 6) |
| `usermod -e 2026-12-31` | Set account expiry |

> **How this goes wrong:** `usermod -G` without `-a` silently **removes the user from every other group**, including `sudo` — you can lock yourself out of admin. Always use **`-aG`** to add.

## 5.4 Group commands

| Command | Effect |
|---|---|
| `groupadd -g 2001 devs` | Create a group with a fixed GID |
| `groupadd --system appdata` | Create a system group |
| `groupmod -n newname oldname` | Rename a group |
| `groupmod -g 2005 devs` | Change GID (file ownership isn't updated) |
| `groupdel devs` | Delete a group (fails if it's someone's primary group) |
| `gpasswd -a nimal devs` | Add one user to a group |
| `gpasswd -d nimal devs` | Remove one user from a group |
| `gpasswd -M a,b,c devs` | Set the full member list |
| `deluser nimal devs` | Debian/Ubuntu: remove from group |

`gpasswd -a/-d` is a safe way to add or remove one membership without touching others.

## 5.5 Deleting

| Command | Effect |
|---|---|
| `userdel nimal` | Remove the account; home folder and files stay |
| `userdel -r nimal` | Also remove home folder and mail spool |
| `deluser --remove-home nimal` | Debian/Ubuntu equivalent |

Neither removes files the user owns **outside** their home. See Phase 9.

**Used in:** Note 4, Steps 2–8.

---

# Phase 6 — Passwords, aging, locking, and expiry

## 6.1 Setting and checking passwords

```bash
sudo passwd nimal          # set nimal's password
passwd                     # change your own
sudo passwd -S nimal       # status
```
```
nimal P 2026-10-17 0 90 7 14
```

| Field | Meaning |
|---|---|
| `P` / `L` / `NP` | Has a usable **P**assword / password **L**ocked / **N**o **P**assword |
| date | Last change |
| `0 90 7 14` | Min days, max days, warning days, inactive days |

`L` also appears for accounts that never had a password (`!` in shadow). That's normal for key-only users and service accounts.

## 6.2 Password aging with `chage`

```bash
sudo chage -l nimal
```
```
Last password change                    : Oct 17, 2026
Password expires                        : Jan 15, 2027
Password inactive                       : Jan 29, 2027
Account expires                         : never
Minimum number of days between password change  : 0
Maximum number of days between password change  : 90
Number of days of warning before password expires: 7
```

| Command | Effect |
|---|---|
| `chage -M 90 user` | Password valid for max 90 days |
| `chage -m 1 user` | Must wait 1 day between changes |
| `chage -W 7 user` | Warn 7 days before expiry |
| `chage -I 14 user` | Lock account 14 days after password expires if not changed |
| `chage -d 0 user` | **Force password change at next login** |
| `chage -E 2026-12-31 user` | **Account** expires on that date |

## 6.3 Two different kinds of expiry

| | Password expiry | Account expiry |
|---|---|---|
| Set by | `chage -M`, `-d` | `chage -E`, `useradd -e`, `usermod -e` |
| What happens | User must **change** the password to continue | User **cannot log in at all** |
| Affects SSH key login? | No | **Yes** |
| Use for | Password rotation policy | Contractors, temporary access, offboarding |

## 6.4 Locking: the trap

```bash
sudo passwd -l amal        # or: usermod -L amal
```

This puts `!` in front of the hash. It blocks **password** logins only.

**It does not block SSH key logins.** A locked user with an `authorized_keys` file can still SSH in.

To truly stop an account from logging in:

| Method | Blocks password | Blocks SSH key | Notes |
|---|---|---|---|
| `passwd -l` / `usermod -L` | Yes | **No** | Password only |
| `usermod -e 1` (expire account) | Yes | **Yes** | Recommended; reversible with `usermod -e ""` |
| `usermod -s /usr/sbin/nologin` | Shell login yes | Shell yes, but not port forwarding or scp-like use | Partial |
| Remove from `ssh-users` (with `AllowGroups`) | Yes (SSH) | **Yes** | Needs sshd set up for it |
| Remove `~/.ssh/authorized_keys` | — | Yes | Keys only |

Use `usermod -e 1` (sets expiry to 2 Jan 1970). Avoid `chage -E 0`: the value 0 is unclear in the shadow format and some tools read it as "never expires".

Best practice for offboarding: **expire + lock + remove from access groups + end sessions** (Note 4, Step 11).

## 6.5 Password policy (PAM)

Rules like minimum length and complexity, and lockout after failed attempts, come from **PAM** (Pluggable Authentication Modules), the login framework:

| Need | Where |
|---|---|
| Password strength | `pam_pwquality`, settings in `/etc/security/pwquality.conf` |
| Lock after failed attempts | `pam_faillock`, `/etc/security/faillock.conf`; check with `faillock --user name` |
| Aging defaults for new users | `/etc/login.defs` (`PASS_MAX_DAYS`, etc.) |

On servers, the stronger policy is usually **no passwords for SSH at all** (keys only, `PasswordAuthentication no` in `sshd_config`). Passwords then only matter for local console and `sudo`.

**Used in:** Note 4, Steps 3, 7, 11.

---

# Phase 7 — Process identity and sessions

## 7.1 Every process carries an identity

Users don't do things. **Processes** do — and each process carries:

- a **UID** (actually several — real, effective, saved; Note 7 explains them)
- a **primary GID**
- a **list of supplementary GIDs**

```bash
grep -E '^(Uid|Gid|Groups)' /proc/self/status
```
```
Uid:    1101    1101    1101    1101      ← real, effective, saved, filesystem
Gid:    1101    1101    1101    1101
Groups: 1101 2001 2004
```

```bash
ps -o pid,user,group,supgrp,cmd -p $$     # same thing via ps
```

The kernel checks permissions against **these process values**, not against `/etc/passwd`.

## 7.2 How a login sets them

```
sshd / login
  │ 1. check password or key (PAM)
  │ 2. look up user → UID, primary GID, group list (from /etc/passwd, /etc/group)
  │ 3. set the process's group list, GID, then UID
  ▼
your shell (bash)   ← now has your identity
  │ fork + exec (Note 1)
  ▼
every program you run   ← inherits the same identity
```

## 7.3 Group changes need a new login

The group list is **copied into the process at login**. Changing `/etc/group` afterwards does not update processes that are already running.

```bash
sudo usermod -aG ops sachini       # sachini is logged in right now

# in sachini's existing session:
id              # groups: sachini, devs      ← her process (old list)
id sachini      # groups: sachini, devs, ops ← the database (new list)
```

`id` with no name shows **your current process**. `id <name>` shows **what the database says**. When they differ, the session is stale.

Ways to get the new group:

| Method | Effect |
|---|---|
| Log out and log back in | Clean, recommended |
| `newgrp ops` | Starts a **new shell** with `ops` as primary group |
| `sg ops -c 'command'` | Runs one command with the `ops` group |
| `su - $USER` | New login shell (asks your password) |

The same applies to **services**: after adding `api` to a group, **restart the service**. The running process still has the old list.

Also: new SSH connections may reuse an existing connection if SSH multiplexing (`ControlMaster`) is on — then "log in again" doesn't actually start a new session.

## 7.4 Files take the process's identity

A new file gets:
- owner = the process's **effective UID**
- group = the process's **effective GID** (primary group), or the folder's group if the folder has SGID (Note 5)

That's why a file created by the API service is owned by `api`, even if *you* started the service.

**Used in:** Note 4, Steps 5, 8.

---

# Phase 8 — Switching identity and seeing who did what

## 8.1 Switching user

| Command | Result | Password asked |
|---|---|---|
| `su nimal` | Shell as nimal, keeps much of your environment | nimal's |
| `su - nimal` | **Login** shell as nimal, clean environment | nimal's |
| `sudo -u nimal cmd` | Run one command as nimal | **your own** (if allowed) |
| `sudo -u nimal -i` | Login shell as nimal | your own |
| `sudo -i` | Root login shell | your own |
| `runuser -u api -- cmd` | Run as `api` from a root script (no PAM password prompt) | none (root only) |

## 8.2 Service accounts and `nologin`

```bash
sudo su - api
# This account is currently not available.
```

`nologin` blocks shells started **as a login** (SSH, `su`). But root can still run commands as the service user:

```bash
sudo -u api whoami                       # api
sudo -u api cat /etc/myapps/api.env      # test what api can read
sudo -u api bash                         # an interactive shell as api, for debugging
```

This is the standard way to **test permissions as a service** (Note 6 uses it a lot).

## 8.3 Root login

Best practice on servers:
- **Lock root's password** and don't log in as root. People use their own accounts plus `sudo`, so every action is tied to a person.
- In `/etc/ssh/sshd_config`: `PermitRootLogin no`.
- **No shared accounts** ("admin" used by three people). Shared accounts make logs useless.

## 8.4 Who is logged in, who did what

| Command / file | Shows |
|---|---|
| `who` / `w` | Who is logged in now, and what they're running |
| `last` | Login history (`/var/log/wtmp`) |
| `lastb` | Failed login attempts (root) |
| `lastlog` | Last login per user (may be replaced by `lastlog2` on the newest releases) |
| `/var/log/auth.log` | SSH logins, sudo use, `su`, account changes (Ubuntu) |
| `journalctl -u ssh` | SSH service logs |
| `loginctl list-sessions` | Active sessions (systemd) |

**Used in:** Note 4, Steps 5, 9, 11.

---

# Phase 9 — Removing accounts and orphaned files

## 9.1 What `userdel` leaves behind

`userdel -r` removes the account, home folder, and mail. It does **not** remove:
- files the user created elsewhere (`/srv/projects`, `/tmp`, `/opt/...`)
- their crontab (on some systems it does; check)
- running processes
- sudo rules that name them directly
- their SSH keys copied into other accounts (e.g. into `deploy`'s `authorized_keys`)

## 9.2 Orphaned files

Files whose UID no longer has a name:

```bash
ls -l /srv/projects/old.txt
# -rw-rw-r-- 1 1201 devs 512 ... old.txt      ← bare number: owner deleted
```

Find them:

```bash
sudo find / -xdev \( -nouser -o -nogroup \) -ls 2>/dev/null
```

## 9.3 UID reuse

`useradd` normally picks the **next free number after the highest in use**. If the deleted user had the highest UID, the next new user gets **the same UID** — and silently becomes the owner of all the orphaned files.

```
amal (UID 1201) deleted, leaves /srv/projects/old.txt (owner 1201)
new user kavindu created → gets UID 1201 → now owns old.txt
```

That's both a privacy problem and a security problem.

## 9.4 Safe offboarding order

1. **Disable** — expire + lock (Phase 6.4), remove from access groups
2. **Stop** — end sessions and processes, remove crontab
3. **Find** — everything they own on the system
4. **Hand over** — reassign needed files to a team account or group
5. **Archive** — copy the home folder somewhere safe if policy needs it
6. **Delete later** — after a waiting period, `userdel -r`, then check `-nouser`
7. **Don't reuse** — use fixed UIDs per person and never give a removed UID to someone new

> **How this goes wrong:** forgotten accounts of people who left (still able to SSH with a key), and orphaned files picked up by a reused UID. Check regularly: `last`, `lastlog`, and `find -nouser`.

**Used in:** Note 4, Steps 11, 12.

---

# Phase 10 — Consistent IDs across servers and containers

## 10.1 The problem

Because files store numbers (Phase 1.2), **the same name with different numbers** on two machines causes wrong ownership:

```
server-01: api = UID 998      server-02: api = UID 997
backup made on 01, restored on 02  →  files owned by 998 = someone else (or nobody) on 02
```

This also affects NFS shares, `rsync -a`, tar archives, and disks moved between servers.

## 10.2 Fixes

| Approach | How |
|---|---|
| **Fixed IDs** | Always create accounts with `-u` / `-g`, from a written plan or a provisioning script |
| **Configuration management** | Ansible, cloud-init, or a script that creates the same accounts the same way everywhere |
| **Central identity** | LDAP / SSSD, or **GCP OS Login** for human users on Compute Engine (accounts and sudo come from Google identities and IAM roles) |
| **Copy by name** | `rsync --usermap` / `--groupmap`, or `tar --same-owner` with matching names |

## 10.3 Containers share UID numbers with the host

A container doesn't have its "own" users in the kernel's eyes. A process running as UID 1000 inside a container is **UID 1000 on the host** (unless user namespaces are used).

Consequences:
- **Root in a container is UID 0 on the host.** Docker limits what it can do, but escaping or mounting host folders gives real root. That's why the `docker` group is root-equivalent (Phase 4.5).
- **Bind mounts** show host files with the container's names for those numbers, which may be different or missing (Note 6 has a full example).
- Best practice: run container apps as a **non-root fixed UID** (`USER 10001` in the Dockerfile, or `docker run --user`).
- **Rootless containers** map container UIDs to an unprivileged range on the host, defined in `/etc/subuid` and `/etc/subgid`.

**Used in:** Note 4, Step 13; Note 6 (bind mounts).

---

# How this goes wrong — overview

A quick list for the security track. Each one is a correct mechanism set too loosely.

| Misconfiguration | Why it's dangerous | How to check |
|---|---|---|
| Extra UID 0 account | Hidden full root | `awk -F: '$3==0' /etc/passwd` |
| Empty password hash | Login with no password | `sudo awk -F: '$2==""' /etc/shadow` |
| Writable `/etc/passwd`, `/etc/shadow`, `/etc/group` | Anyone can create root | `ls -l /etc/passwd /etc/shadow /etc/group` |
| Service accounts with real shells + passwords | Interactive login as a service | `getent passwd \| awk -F: '$3<1000 && $7!~/(nologin\|false)$/'` |
| Shared human accounts | No accountability | Review `last`, interview teams |
| People in `sudo`, `docker`, `lxd`, `disk` without need | Root or root-equivalent | `getent group sudo docker lxd disk` |
| `usermod -G` without `-a` | Removes groups, sometimes admin lockout | Use `-aG` / `gpasswd -a` |
| "Locked" with `passwd -l` but has SSH keys | Still able to log in | `passwd -S` + check `~/.ssh/authorized_keys` + `chage -l` |
| Accounts of people who left | Old keys still work | `lastlog`, HR list vs `getent passwd` |
| Orphaned files + UID reuse | New user inherits old data | `find / -xdev -nouser` |
| Different IDs across servers/containers | Wrong ownership after restore/mount | `ls -n`, compare `getent passwd` |

---

# Summary table

| Concept | Key point |
|---|---|
| UID / GID | The kernel uses numbers; names are labels |
| Files store numbers | Deleted users → bare numbers; reused UIDs inherit files |
| UID 0 | Root, whatever the name |
| `/etc/passwd` | Accounts (public) |
| `/etc/shadow` | Hashes + aging (root/shadow only) |
| `/etc/group` | Groups + supplementary members (primary members not listed) |
| `getent` | Correct way to look up accounts from all sources |
| `vipw` / `vigr` / `pwck` / `grpck` | Safe manual edit / consistency check |
| Account types | root, system/service (nologin), human, machine login, nobody |
| `/etc/login.defs`, `/etc/skel` | Defaults for new accounts and homes |
| Primary vs supplementary groups | New files use the primary group; all groups count for access |
| User private groups | Each user has their own group |
| Role groups | Grant to groups, put people in groups |
| Special groups | `sudo`, `docker`, `lxd`, `disk` = root or equivalent |
| `useradd` vs `adduser` | Scripts: `useradd` with all options; manual: `adduser` |
| `usermod -aG` | Add to a group (`-G` alone replaces!) |
| `passwd -l` | Locks password only, **not SSH keys** |
| `usermod -e 1` | Expires the account — blocks all logins |
| Password vs account expiry | Change password vs no login at all |
| Process identity | UID + GID + group list, set at login, inherited |
| Group change | Needs new login (or `newgrp`, service restart) |
| `id` vs `id user` | Current process vs database |
| `sudo -u svc cmd` | Test as a nologin service account |
| Offboarding | Disable → stop → find → hand over → archive → delete later |
| Consistent IDs | Fixed `-u`/`-g`, provisioning scripts, central identity |
| Containers | Same UID numbers as the host; container root = host root |

---

# Appendix — Command reference

## Looking up identity

| Command | Purpose |
|---|---|
| `id` / `id user` | Current process identity / database identity |
| `whoami` | Current effective username |
| `groups user` | Groups of a user |
| `getent passwd user` | Account line (all sources) |
| `getent group name` | Group line (all sources) |
| `sudo getent shadow user` | Shadow line |
| `ls -n` | Owners as numbers |
| `grep -E '^(Uid\|Gid\|Groups)' /proc/self/status` | Process credentials |
| `ps -o pid,user,group,supgrp,cmd -p PID` | Identity of a process |

## Creating and changing

| Command | Purpose |
|---|---|
| `useradd -m -s /bin/bash -u UID -c "Name" -G g1,g2 user` | Human account |
| `useradd --system -u UID -g grp -d /var/lib/svc -M -s /usr/sbin/nologin svc` | Service account |
| `useradd -D` | Show `useradd` defaults |
| `adduser user` | Interactive (Debian/Ubuntu) |
| `usermod -aG group user` | Add to group |
| `usermod -s SHELL` / `-d DIR -m` / `-l NEW` / `-u UID` | Change shell / home / name / UID |
| `groupadd -g GID name` / `groupadd --system name` | Create group |
| `groupmod -n NEW OLD` / `-g GID` | Rename / renumber group |
| `gpasswd -a user group` / `-d user group` | Add / remove one member |
| `userdel -r user` / `groupdel group` | Delete |

## Passwords and expiry

| Command | Purpose |
|---|---|
| `passwd user` | Set password |
| `passwd -S user` | Status |
| `passwd -l` / `-u user` | Lock / unlock password |
| `chage -l user` | Show aging |
| `chage -M 90 -W 7 -I 14 user` | Aging policy |
| `chage -d 0 user` | Force change at next login |
| `usermod -e YYYY-MM-DD user` / `chage -E YYYY-MM-DD user` | Account expiry date |
| `usermod -e 1 user` | Expire now (blocks key logins too) |
| `usermod -e "" user` | Remove expiry |
| `faillock --user user` | Failed-login lock status |

## Switching and sessions

| Command | Purpose |
|---|---|
| `su - user` | Login shell as user (their password) |
| `sudo -u user cmd` / `sudo -u user -i` | Run as user / login shell as user |
| `runuser -u user -- cmd` | Run as user from root |
| `newgrp group` / `sg group -c cmd` | New shell / one command with a group |
| `who`, `w`, `last`, `lastb`, `lastlog` | Sessions and history |
| `loginctl list-sessions` / `terminate-user user` | systemd sessions |

## Checks

| Command | Purpose |
|---|---|
| `awk -F: '$3==0' /etc/passwd` | UID 0 accounts |
| `sudo awk -F: '$2==""' /etc/shadow` | Empty passwords |
| `sudo pwck -r` / `sudo grpck -r` | Consistency check |
| `getent group sudo docker lxd disk adm` | Powerful group members |
| `sudo find / -xdev \( -nouser -o -nogroup \)` | Orphaned files |
| `grep -E '^(UID\|SYS_UID\|HOME_MODE\|UMASK)' /etc/login.defs` | Account defaults |