# Files, Directories & Permissions — Theory (Note 5)

How Linux decides whether a process may read, write, enter, or run something — and every tool that shapes that decision: the rwx bits, ownership, umask, special bits, ACLs, file attributes, and mount options.

**Series:**
- Note 1: Shell & Processes
- Note 2: Environment Variables
- Note 3 / 4: Users, Groups & Identity — Theory / Practice
- **Note 5: Files, Directories & Permissions — Theory** ← this note
- Note 6: Files, Directories & Permissions — Practice
- Note 7 / 8: Privilege & Delegation — Theory / Practice

Note 3 answered *who is asking* (a process with a UID, GID, and group list). This note answers *what the file allows*. Note 7 covers the ways to grant extra power on purpose (SUID, sudo, capabilities).

Each phase ends with **"Used in"** (Note 6 steps) and, where relevant, **"How this goes wrong"** for the security track.

---

## Contents

1. Phase 1 — What a file really is: inodes and directory entries
2. Phase 2 — The permission line at a glance
3. Phase 3 — rwx on files vs directories
4. Phase 4 — Which class applies to you
5. Phase 5 — Ownership: who can change what
6. Phase 6 — Changing permissions with `chmod`
7. Phase 7 — Default permissions: umask
8. Phase 8 — Special bits: SUID, SGID, sticky
9. Phase 9 — ACLs: more than one owner and one group
10. Phase 10 — File attributes: immutable and append-only
11. Phase 11 — Mount options: `noexec`, `nosuid`, `nodev`, `ro`
12. Phase 12 — How the kernel makes one decision (full version)
13. Phase 13 — Where things should live, and with what permissions
14. Phase 14 — Debugging "Permission denied": a repeatable method
15. How this goes wrong — overview
16. Summary table
17. Appendix — Command reference

---

# Phase 1 — What a file really is: inodes and directory entries

## 1.1 Two separate parts

A file on disk has two parts:

| Part | Holds |
|---|---|
| **Inode** | The file's metadata: owner UID, group GID, permission bits, size, timestamps, ACLs, attributes, and where the data is on disk |
| **Directory entry** | Just a **name** pointing to an inode number |

The **name is not inside the file**. It's a line in the folder that contains it.

```bash
stat report.txt
```
```
  File: report.txt
  Size: 120        Blocks: 8     IO Block: 4096   regular file
Device: 8,1        Inode: 393812 Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1101/   nimal)   Gid: ( 2001/    devs)
Access: 2026-10-17 09:20:01
Modify: 2026-10-17 09:19:44
Change: 2026-10-17 09:19:44
```

`ls -i` shows inode numbers.

## 1.2 Why this matters for permissions

- **Permissions belong to the inode**, so they're the same no matter which name you use to reach the file.
- **Creating, deleting, and renaming** change the **directory** (they add or remove a name), so they're controlled by the **directory's** permissions, not the file's (Phase 3).
- **Writing to a file** changes the inode's data, so it's controlled by the **file's** permissions.

## 1.3 Hard links and symlinks

| | Hard link | Symbolic link (symlink) |
|---|---|---|
| What | A second name for the **same inode** | A small file containing a **path** to another file |
| Permissions | Shared — it *is* the same file | Its own `lrwxrwxrwx` is ignored; the **target's** permissions apply |
| Deleting the original | Data stays until the last name is removed | Link breaks ("dangling") |
| Across filesystems | No | Yes |

```bash
ln report.txt report-hard.txt     # hard link
ln -s /opt/api/releases/v2 current   # symlink
```

> **How this goes wrong:** in world-writable folders like `/tmp`, an attacker can create a symlink or hard link named like a file a root process will write, making root write somewhere else. Linux protects against this with `fs.protected_symlinks` and `fs.protected_hardlinks` (should be `1`; check with `sysctl fs.protected_symlinks fs.protected_hardlinks`).

**Used in:** Note 6, Steps 2, 3.

---

# Phase 2 — The permission line at a glance

```text
-  rwx  r-x  r-x  +  1   thirana  staff   4096   Oct 17 09:00   script.sh
│  │    │    │    │  │   │        │       │      │              │
│  │    │    │    │  │   │        │       │      │              └─ name (from the directory entry)
│  │    │    │    │  │   │        │       │      └─ last modified time
│  │    │    │    │  │   │        │       └─ size in bytes
│  │    │    │    │  │   │        └─ group owner
│  │    │    │    │  │   └─ user owner
│  │    │    │    │  └─ number of hard links
│  │    │    │    └─ "+" = has an ACL (Phase 9); "." = SELinux label only
│  │    │    └─ "other": everyone who is not the owner and not in the group
│  │    └─ "group": members of the file's group
│  └─ "owner": the user who owns the file
└─ type: - file, d directory, l symlink, c/b device, s socket, p pipe
```

Each triad is **read / write / execute** for one class. A letter = allowed, `-` = denied.

### Octal (numeric) form

| Number | Permission |
|---|---|
| 4 | read `r` |
| 2 | write `w` |
| 1 | execute `x` |

Add them per triad: `rwx` = 7, `r-x` = 5, `rw-` = 6, `r--` = 4.

| Mode | Letters | Typical use |
|---|---|---|
| `755` | `rwxr-xr-x` | Programs, public directories |
| `750` | `rwxr-x---` | Directories for owner + group only |
| `700` | `rwx------` | Private directories (`~/.ssh`) |
| `644` | `rw-r--r--` | Normal readable files |
| `640` | `rw-r-----` | Config/secrets readable by a group |
| `600` | `rw-------` | Private files (keys, secrets owned by one user) |
| `2770` | `rwxrws---` | Shared team directory (SGID, Phase 8) |
| `1777` | `rwxrwxrwt` | `/tmp` (sticky, Phase 8) |

A fourth, leading digit holds the special bits: `4` SUID, `2` SGID, `1` sticky.

**Used in:** every step of Note 6.

---

# Phase 3 — rwx on files vs directories

The same three letters mean very different things on a file and on a directory. Most permission confusion comes from here.

## 3.1 On a file

| Bit | Allows |
|---|---|
| `r` | Read the contents |
| `w` | Change the contents (including emptying it) |
| `x` | Run it as a program |

- `w` on a file does **not** let you delete or rename it — that's the directory's job.
- To **run a script**, you need `x` **and** `r` (the interpreter has to read it). A compiled program needs only `x`.

## 3.2 On a directory

A directory is a list of names. The bits control that list.

| Bit | Allows |
|---|---|
| `r` | **List** the names inside (`ls`) |
| `w` | **Create, delete, rename** entries — only together with `x` |
| `x` | **Enter** it (`cd`) and **reach** a named entry inside ("search" or "traverse") |

## 3.3 Directory combinations

| Mode | What you can do |
|---|---|
| `r--` | See names only. `ls -l` shows `?????????` for details, because you can't reach the entries |
| `--x` | Open a file **if you know its exact name**; can't list |
| `r-x` | List and open — normal read-only access |
| `-wx` | Create/delete files if you know names; can't list (a "drop box") |
| `rwx` | Full control of the list |
| `rw-` | Effectively nothing useful — without `x`, you can't reach entries, so `w` doesn't work |

## 3.4 Every folder on the path needs `x`

To open `/srv/uploads/2026/img.png`, the process needs `x` on **every** directory on the way:

```
/        x?
/srv     x?
/srv/uploads       x?
/srv/uploads/2026  x?
img.png            r?
```

If any one is missing, the answer is "Permission denied" — even if the file itself is `644`.

This also means **a directory protects everything under it**. A `644` file inside a `700` directory is unreadable to others, because they can't get through the directory.

`namei -l` shows the whole path at once:

```bash
namei -l /srv/uploads/2026/img.png
```
```
f: /srv/uploads/2026/img.png
drwxr-xr-x root root    /
drwxr-xr-x root root    srv
drwxrws--- root appdata uploads
drwxr-xr-x api  appdata 2026
-rw-r----- api  appdata img.png
```

## 3.5 Deleting depends on the directory

If you have `w` + `x` on a directory, you can **delete any file in it** — even one you don't own and can't read or write. Deleting is "remove a name from the list", and the list is the directory.

The sticky bit (Phase 8) is the fix for shared directories.

> **How this goes wrong:** a script run by root lives in a directory others can write. They can't edit the script (it's `755 root`), but they can **delete it and put their own** in its place. Always check the directory, not just the file.

**Used in:** Note 6, Steps 2, 5, 7, 9.

---

# Phase 4 — Which class applies to you

## 4.1 First match wins

The kernel picks **one** class and uses only that triad:

1. Is the process's **effective UID** = the file's owner? → use the **owner** triad. Stop.
2. Otherwise, is the file's group in the process's **group list** (primary or supplementary)? → use the **group** triad. Stop.
3. Otherwise → use the **other** triad.

The classes are **not combined**.

## 4.2 Surprising results

```text
----r----- 1 nimal devs ... notes.txt     (mode 040)
```

Nimal is the owner **and** in `devs`. He **can't** read it: the owner triad is checked first and says `---`. Everyone else in `devs` can.

```text
-rw----r-- 1 nimal devs ... file.txt      (mode 604)
```

A `devs` member who isn't nimal **can't** read it (group triad `---`), but a user **not** in `devs` **can** (other `r--`). Rare, but it proves the rule.

Note 3's lesson matters here too: the "group list" is the **process's** list, fixed at login (Note 3, Phase 7.3).

**Used in:** Note 6, Steps 2, 9.

---

# Phase 5 — Ownership: who can change what

## 5.1 The rules

| Action | Who may do it |
|---|---|
| `chown` (change user owner) | **Root only** (`CAP_CHOWN`). You can't give files away or take them |
| `chgrp` (change group) | The file's **owner**, and only to a group **they belong to**; or root |
| `chmod` (change mode) | The file's **owner**, or root. Being able to *write* the file is not enough |
| Write contents | Anyone the mode allows |

```bash
sudo chown nimal file
sudo chown nimal:devs file
chgrp devs file
sudo chown -R api:appdata /srv/uploads
sudo chown -h deploy:deploy current     # change the symlink itself, not its target
```

Safety detail: when a file's owner or group changes, Linux **clears its SUID/SGID bits**, so you can't hand someone a privileged program by accident.

## 5.2 How ownership is set on new files

| How the file appears | Owner | Group | Mode |
|---|---|---|---|
| Created new (`touch`, `>`, app writes it) | Process's effective UID | Process's GID — or the **directory's group** if the directory has SGID | Requested mode minus umask (or default ACL, Phase 9) |
| `cp src dst` (new file) | **Whoever copies** | Same rule as "created new" | Source mode minus umask |
| `cp -p` / `cp -a` | Kept (only root can keep a different owner) | Kept | Kept (`-a` also keeps ACLs) |
| `mv` on the same filesystem | **Kept** | **Kept** | **Kept** — it's only a rename |
| `mv` to another filesystem | Like `cp -a` + delete | | |
| `rsync -a` | Kept by **number** (as root) | Kept by number | Kept (`-A` for ACLs, `-X` for attributes) |
| `tar -x` as root | Kept from the archive | Kept | Kept (`--acls --xattrs` for extras) |

The big one: **`mv` keeps the old owner, group, and mode.** A file moved into a team folder doesn't pick up the folder's group or default ACL — a copied one does. (Note 6, Step 2.)

**Used in:** Note 6, Steps 2, 3, 10.

---

# Phase 6 — Changing permissions with `chmod`

## 6.1 Numeric

```bash
chmod 640 file          # set exactly rw-r-----
chmod 2770 dir          # SGID + rwxrwx---
```

## 6.2 Symbolic

`who` + `op` + `what`:

| Part | Values |
|---|---|
| who | `u` user/owner, `g` group, `o` other, `a` all |
| op | `+` add, `-` remove, `=` set exactly |
| what | `r`, `w`, `x`, `X`, `s` (SUID/SGID), `t` (sticky) |

```bash
chmod u+x deploy.sh           # add execute for owner
chmod g-w,o= file             # remove group write, remove all from other
chmod u=rw,g=r,o= file        # same as 640
chmod g+s dir                 # SGID on a directory
chmod --reference=a.txt b.txt # copy mode from another file
```

Symbolic is safer when you want to change **one** bit without touching the others.

## 6.3 Recursive changes: the capital `X`

```bash
chmod -R 644 /srv/projects        # WRONG: removes x from directories → nobody can enter them
chmod -R 755 /srv/projects        # WRONG: makes every file executable
```

Capital **`X`** means "execute, but only for directories (and files that already have an x somewhere)":

```bash
chmod -R u=rwX,g=rX,o= /opt/api/releases    # dirs 750, files 640 (scripts keep their x)
```

Or treat files and directories separately with `find`:

```bash
find /opt/api/releases -type d -exec chmod 755 {} +
find /opt/api/releases -type f -exec chmod 644 {} +
```

## 6.4 Seeing modes clearly

```bash
stat -c '%a %A %U:%G %n' /etc/myapps/*
```
```
644 -rw-r--r-- root:root /etc/myapps/common.env
640 -rw-r----- root:api /etc/myapps/api.env
```

> **How this goes wrong:** `chmod -R 777` to "fix" a permission error. It makes everything writable and executable by every user and every compromised service on the machine (Note 6, Step 11).

**Used in:** Note 6, Steps 3, 11.

---

# Phase 7 — Default permissions: umask

## 7.1 What it is

When a program creates a file, it asks for a mode — usually `666` for files and `777` for directories. The process's **umask** then **removes** bits from that request.

```
requested  666  rw-rw-rw-
umask      022  ----w--w-   (bits to remove)
result     644  rw-r--r--
```

It is not subtraction; it's "remove these bits". `umask` can only take permissions away, never add them.

| umask | New file | New directory | Typical use |
|---|---|---|---|
| `022` | `644` | `755` | System default; others can read |
| `002` | `664` | `775` | Team work with user private groups |
| `027` | `640` | `750` | Services; others get nothing |
| `077` | `600` | `700` | Very private (root's scripts, secrets) |
| `007` | `660` | `770` | Shared service data; others get nothing |

```bash
umask          # 0002
umask -S       # u=rwx,g=rwx,o=rx
umask 027      # change for this shell and its children
```

Programs can ask for **narrower** modes on purpose — e.g. `ssh-keygen` creates private keys as `600` whatever your umask is.

## 7.2 umask is per process and inherited

Like the environment (Note 2), umask is part of the process and is **copied to children** on fork/exec. Set it once in the parent, and everything started from there uses it.

## 7.3 Where umask comes from

| Context | Source |
|---|---|
| Login sessions | `/etc/login.defs` `UMASK` (applied by `pam_umask`), then shell startup files |
| Ubuntu users with user private groups | With `USERGROUPS_ENAB yes`, group bits are made equal to owner bits, so `022` becomes **`002`** for normal users. Check yours with `umask` |
| Your shell | `umask` in `~/.bashrc` / `~/.profile` |
| systemd services | `UMask=` in the unit; default **`0022`** |
| cron jobs | Inherited from cron (usually `022`); set it in the script |
| Docker containers | Usually `022`; set in the entrypoint |

Why `002` is safe for normal users: their private group contains only themselves (Note 3, Phase 4.3), so "group writable" means "writable only by me" — until the file lands in a **shared** group's folder, which is exactly where you want group write.

> **How this goes wrong:** services with umask `022` writing secrets, uploads, or backups that become readable by every local user. Set `UMask=0027` (or narrower) for services that handle private data.

**Used in:** Note 6, Steps 0, 2, 5, 7.

---

# Phase 8 — Special bits: SUID, SGID, sticky

## 8.1 Overview

| Bit | Octal | On a file | On a directory |
|---|---|---|---|
| **SUID** | `4000` | Program runs with the **file owner's** UID | (ignored on Linux) |
| **SGID** | `2000` | Program runs with the **file's group** | New files and folders **inherit the directory's group**; new subfolders also get SGID |
| **Sticky** | `1000` | (ignored on Linux) | Only the **file's owner**, the **directory's owner**, or root can delete/rename entries |

How they look:

| You see | Where | Means |
|---|---|---|
| `-rwsr-xr-x` | owner `x` slot | SUID + execute |
| `drwxrws---` | group `x` slot | SGID + execute |
| `drwxrwxrwt` | other `x` slot | Sticky + execute |
| `S` / `T` (capital) | same slots | Bit set but `x` is **off** — usually a mistake |

```bash
chmod u+s file   # SUID      chmod 4755 file
chmod g+s dir    # SGID      chmod 2770 dir
chmod +t dir     # sticky    chmod 1777 dir
```

SUID and SGID **on files** change the identity a program runs with. They're part of privilege, so the details (and how they're abused) are in **Note 7**.

## 8.2 SGID on directories — the team folder tool

Without SGID, a file gets the creator's **primary** group (their private group). Teammates in `devs` can't use it.

```
/srv/projects   drwxrwx--- root devs
nimal creates notes.md  →  -rw-rw-r-- nimal nimal   ← group is "nimal", not "devs"
```

With SGID:

```
/srv/projects   drwxrws--- root devs
nimal creates notes.md  →  -rw-rw-r-- nimal devs    ← inherits "devs"
nimal creates sub/      →  drwxrwsr-x nimal devs    ← subfolder also gets SGID
```

**SGID fixes the group, not the group's permission bits.** If the creator's umask is `022`, the file is `644` — group can read but not edit. You still need umask `002` or a default ACL (Phase 9). Note 6, Step 2 shows both problems.

## 8.3 Sticky bit — shared folders where people shouldn't delete each other's files

```bash
ls -ld /tmp
# drwxrwxrwt 18 root root ... /tmp
```

Everyone can create files in `/tmp` (`rwx` for other), but the `t` means you can only delete **your own**. Without it, Phase 3.5 would let any user delete anyone's temp files.

Use sticky on any directory that many users write to but shouldn't clean up for each other.

> **How this goes wrong:** a world-writable directory **without** sticky (`drwxrwxrwx`) — anyone can delete or replace anyone's files. Find them: `find / -xdev -type d -perm -0002 ! -perm -1000`.

**Used in:** Note 6, Steps 2, 5, 7.

---

# Phase 9 — ACLs: more than one owner and one group

## 9.1 Why ACLs exist

Normal permissions give you **one** owner, **one** group, and everyone else. Real needs often look like:

- `api` writes, `devs` read, `adm` read, nobody else → two groups need access
- One contractor needs read access to one folder, without joining a whole group

**ACLs (Access Control Lists)** add extra entries for **named users** and **named groups** on top of the normal bits. They're supported on ext4, XFS, and most Linux filesystems by default.

## 9.2 Reading an ACL

```bash
getfacl /var/log/myapps/api
```
```
# file: var/log/myapps/api
# owner: api
# group: adm
# flags: -s-
user::rwx
group::r-x
group:devs:r-x
mask::r-x
other::---
default:user::rwx
default:group::r-x
default:group:devs:r-x
default:mask::r-x
default:other::---
```

| Entry | Meaning |
|---|---|
| `user::` | The owner (same as owner triad) |
| `user:name:` | A **named user** |
| `group::` | The owning group (same as group triad) |
| `group:name:` | A **named group** |
| `mask::` | **Upper limit** for named users, the owning group, and named groups |
| `other::` | Everyone else (same as other triad) |
| `default:...` | On directories: the ACL **new files and folders inside will get** |
| `flags: -s-` | SGID is set |

`ls -l` shows a `+`:

```
drwxr-s---+ 2 api adm 4096 ... api
```

## 9.3 Setting ACLs

```bash
setfacl -m g:devs:rX dir                 # add/modify an entry (-m = modify)
setfacl -m u:kavindu:rX dir              # a named user
setfacl -x g:devs dir                    # remove one entry
setfacl -b dir                           # remove the whole ACL
setfacl -R -m g:devs:rX dir              # recursively, for existing files
setfacl -m d:g:devs:rX dir               # default ACL (d: prefix) for new files
setfacl -k dir                           # remove the default ACL only
```

Capital `X` works here too (execute only for directories).

**Typical pattern** for a shared folder — fix existing content and future content:

```bash
setfacl -R -m g:devs:rwX /srv/projects     # existing files and folders
setfacl -R -m d:g:devs:rwX /srv/projects   # default for new ones (-R applies to subfolders)
```

## 9.4 The mask — the confusing part

The **mask** limits every entry in the "group class": named users, the owning group, and named groups. The **effective** permission is the entry **AND** the mask.

```
group:devs:rwx      #effective:r-x     ← devs asked rwx, mask allows only r-x
mask::r-x
```

Two rules that surprise people:

1. **When a file has an ACL, the "group" slot in `ls -l` shows the mask**, not the owning group's permission.
2. **`chmod g=...` on a file with an ACL changes the mask.** So `chmod 700 dir` sets the mask to `---`, which **silently removes access** for every named user and group, without deleting the entries.

```bash
chmod 750 dir        # mask becomes r-x → devs effectively r-x
chmod 700 dir        # mask becomes --- → devs effectively nothing
getfacl dir          # shows "#effective:---" next to group:devs
```

`setfacl -m` recalculates the mask automatically to cover all entries (unless you use `-n`).

## 9.5 Default ACLs replace umask

When a directory has a default ACL, files created inside get:

**requested mode AND the default ACL** — the process's **umask is not used**.

That's why default ACLs are the most reliable way to keep a shared folder group-writable: it works even if one user or app has umask `022`. (`mv` still keeps the old ACL — Phase 5.2.)

## 9.6 Keeping ACLs when copying and backing up

| Tool | Keeps ACLs with |
|---|---|
| `cp` | `-a` or `--preserve=all` |
| `rsync` | `-A` (and `-X` for other extended attributes) |
| `tar` | `--acls --xattrs` (when creating **and** extracting) |
| Backup of ACLs only | `getfacl -R /srv > acls.txt` and later `setfacl --restore=acls.txt` |

## 9.7 Updated decision rule with ACLs

1. Owner? → `user::` entry. Stop.
2. A **named user** entry matches? → that entry **AND mask**. Stop.
3. Any matching group — owning group or named groups? → allowed if **any** matching entry (AND mask) grants the access; denied otherwise. Stop.
4. → `other::` entry.

> **How this goes wrong:**
> - Forgotten named-user entries for people who left (`getfacl -R` audits them).
> - A `chmod` later silently disabling the ACL through the mask — or a looser mask granting more than intended.
> - Backups restored without `--acls`, quietly dropping access rules.

**Used in:** Note 6, Steps 2, 5, 6, 9.

---

# Phase 10 — File attributes: immutable and append-only

Attributes are flags stored in the inode, separate from permissions. They apply **even to root**.

| Attribute | Effect |
|---|---|
| `i` (immutable) | Can't be changed, deleted, renamed, or linked to — by anyone |
| `a` (append-only) | Can only be added to (`>>`); can't be truncated, overwritten, deleted, or renamed |

```bash
sudo chattr +i /etc/myapps/common.env    # freeze
sudo chattr -i /etc/myapps/common.env    # unfreeze
sudo chattr +a /var/log/myapps/api/audit.log
lsattr /var/log/myapps/api/audit.log
# -----a--------e------- /var/log/myapps/api/audit.log
```

- Only root (with `CAP_LINUX_IMMUTABLE`) can set or remove them.
- Root can still remove the attribute and then change the file — so it guards against **mistakes and normal processes**, not against a determined root attacker.
- Append-only logs break log rotation (rotation renames/truncates). You need to remove the attribute during rotation, or rotate differently.
- Supported on ext4 and XFS (XFS supports `i` and `a`, not every attribute).

> **How this goes wrong:** attackers sometimes set `+i` on their own backdoor so admins can't delete it — "Operation not permitted" as root is the clue. Check with `lsattr`.

**Used in:** Note 6, Steps 6, 7.

---

# Phase 11 — Mount options: `noexec`, `nosuid`, `nodev`, `ro`

A filesystem can be mounted with options that override what the files say:

| Option | Effect |
|---|---|
| `ro` | Read-only: no writes at all |
| `noexec` | Files can't be **run directly** (`./script` → Permission denied) |
| `nosuid` | SUID/SGID bits on files are ignored |
| `nodev` | Device files are ignored |

```bash
findmnt -o TARGET,SOURCE,OPTIONS /tmp
```

Common hardening: mount `/tmp`, `/var/tmp`, `/dev/shm`, and upload folders with `noexec,nosuid,nodev`.

**Limitation:** `noexec` only blocks **direct** execution. An interpreter can still read and run the file:

```bash
./x.sh        # Permission denied
bash x.sh     # runs
```

It's a useful extra layer (stops dropped binaries from running directly), not a complete defence.

A folder that isn't its own filesystem can get these options with a **bind mount** of itself (Note 6, Step 11).

**Used in:** Note 6, Step 11.

---

# Phase 12 — How the kernel makes one decision (full version)

When a process tries to open, create, delete, or run something:

```
1. Walk the path
   └─ need x (search) on every directory on the way          → else EACCES

2. Filesystem and attribute checks
   ├─ writing on a read-only mount (ro)?                     → EROFS
   ├─ executing on a noexec mount?                            → EACCES
   └─ changing an immutable (+i) / non-append to +a file?    → EPERM ("Operation not permitted")

3. Bypass for privileged processes
   ├─ CAP_DAC_OVERRIDE (root has it): skip read/write checks, and
   │   execute checks on directories. A FILE still needs at least
   │   one x bit set somewhere to be executed.
   └─ CAP_DAC_READ_SEARCH: skip read checks on files and
       read/search checks on directories

4. Discretionary check (owner/group/other + ACL)
   ├─ owner?            → owner entry
   ├─ named user?       → entry AND mask
   ├─ any group match?  → any (entry AND mask) that allows
   └─ else              → other
   For delete/rename: check the DIRECTORY (w+x), plus sticky rule

5. Mandatory access control (if enabled): AppArmor / SELinux
   └─ can deny even when everything above allowed            → EACCES, logged by the kernel
```

| Error | Usually means |
|---|---|
| `Permission denied` (EACCES) | Permission bits, ACL, path traversal, `noexec`, or AppArmor/SELinux |
| `Operation not permitted` (EPERM) | Not allowed **at all** for you: `chown` as non-root, immutable/append-only attribute, `chmod` on a file you don't own |
| `Read-only file system` (EROFS) | `ro` mount |

Four layers, each answering a different question:

| Layer | Question |
|---|---|
| Identity (Note 3) | Who is the process? |
| Ownership + bits + ACL (this note) | What does the file allow that identity? |
| Attributes + mount options (this note) | Does the file or filesystem forbid it regardless? |
| Privilege (Note 7) | Does the process have a way around the rules (root, capabilities, SUID, sudo)? |

---

# Phase 13 — Where things should live, and with what permissions

## 13.1 Standard locations

| Path | Normal mode / owner | Notes |
|---|---|---|
| `/` , `/usr`, `/etc` | `755 root:root` | Only root writes |
| `/etc/passwd`, `/etc/group` | `644 root:root` | Public |
| `/etc/shadow` | `640 root:shadow` | Private |
| `/home` | `755 root:root` | |
| `/home/user` | `750 user:user` (newer Ubuntu) | Private by default |
| `~/.ssh` | `700 user:user` | sshd refuses looser |
| `~/.ssh/authorized_keys` | `600` (or `644`) `user:user` | Must not be group/other writable |
| Private keys (`id_ed25519`) | `600` | ssh refuses looser |
| `/tmp`, `/var/tmp` | `1777 root:root` | Sticky |
| `/var/log` | `root`-owned, written by services | Many log files `640 syslog:adm` |

## 13.2 Layout for your own applications

The key principle: **an app should not be able to change its own code or config.** If it's broken into, the attacker shouldn't be able to plant a permanent backdoor in the code or edit its own limits.

| What | Where | Owner | App's access |
|---|---|---|---|
| Code | `/opt/<app>` (or `/usr/local/lib/<app>`) | root or `deploy` | **Read only** |
| Config | `/etc/<app>` | `root:<app>` `640` | **Read only** |
| Secrets | `/etc/<app>` or a secret manager | `root:<app>` `640` / `600` | **Read only** |
| State / data | `/var/lib/<app>` | `<app>` `750` | Read/write |
| Cache | `/var/cache/<app>` | `<app>` | Read/write, safe to delete |
| Logs | `/var/log/<app>` (or journald) | `<app>` | Write (append) |
| Runtime (sockets, PID files) | `/run/<app>` | `<app>` | Read/write, cleared on boot |
| Shared data | `/srv/<name>` | group-owned | By group / ACL |

systemd can create the data, cache, log, and runtime folders for you with correct owners (`StateDirectory=`, `CacheDirectory=`, `LogsDirectory=`, `RuntimeDirectory=`) — Note 8.

**Used in:** Note 6, Steps 1, 3, 4.

---

# Phase 14 — Debugging "Permission denied": a repeatable method

Work through these in order. Each step either finds the cause or rules a layer out.

| # | Question | Command | What you learn |
|---|---|---|---|
| 1 | **Who** is the process, really? | `id` (yourself) / `grep -E '^(Uid\|Gid\|Groups)' /proc/PID/status` (a service) | UID and the **actual** group list (stale groups? Note 3, Phase 7.3) |
| 2 | Can it **get through the path**? | `namei -l /full/path/file` | A directory without `x` for that identity |
| 3 | What do the **bits and owner** say? | `stat -c '%a %A %U:%G %n' file` | Which class applies (Phase 4) and its bits |
| 4 | Is there an **ACL**? | `getfacl file` (look for `#effective`) | Named entries, mask limits |
| 5 | For create/delete: what about the **directory**? | `stat` + `getfacl` on the parent; check for sticky | Phase 3.5 / 8.3 |
| 6 | Any **attributes**? | `lsattr file` | `i` or `a` → EPERM even for root |
| 7 | Any **mount options**? | `findmnt -T file` | `ro`, `noexec`, `nosuid` |
| 8 | **AppArmor/SELinux**? | `sudo journalctl -k \| grep -i -E 'apparmor="DENIED"\|avc:'` | MAC denial |
| 9 | **Test** as that identity | `sudo -u api test -r file && echo yes \|\| echo no` (also `-w`, `-x`) | Confirms the fix |

Never fix by widening permissions until you know **which** layer is denying. "Make it 777" hides the real problem and creates a new one.

**Used in:** Note 6, Step 9 (full walkthrough) and throughout.

---

# How this goes wrong — overview

| Misconfiguration | Why it's dangerous | How to find it |
|---|---|---|
| World-writable files | Anyone (or any compromised service) can change them | `find / -xdev -type f -perm -0002` |
| World-writable dirs without sticky | Anyone can delete/replace others' files | `find / -xdev -type d -perm -0002 ! -perm -1000` |
| Root-run scripts or their directories writable by others | Replace the script → runs as root | Check the script **and** every directory above it |
| Writable systemd units, cron files, `/etc/profile.d/` | Code that root or others will run | `find /etc/systemd /etc/cron* /etc/profile.d -perm /022` |
| App can write its own code/config | Persistent backdoor after compromise | `sudo -u app test -w /opt/app/... && echo WRITABLE` |
| Secrets `644` | Every local user reads them | `stat` on secret files; check umask of the writer |
| Services with umask `022` writing private data | New files readable by all | Check `UMask=`; check new files' modes |
| Loose `~/.ssh` or keys | sshd/ssh refuse — or worse, someone edits `authorized_keys` | `stat ~/.ssh ~/.ssh/authorized_keys` |
| `chmod -R 777` | Everything wide open and executable | `find -perm -0002` |
| ACL mask / forgotten ACL entries | Hidden access, or access silently removed | `getfacl -R` |
| Immutable files you didn't set | Attacker persistence | `lsattr -R` on suspicious paths |
| `/tmp`, uploads without `noexec,nosuid,nodev` | Dropped binaries run directly | `findmnt -o TARGET,OPTIONS` |
| `protected_symlinks/hardlinks` off | Link attacks in shared dirs | `sysctl fs.protected_symlinks fs.protected_hardlinks` |

---

# Summary table

| Concept | Key point |
|---|---|
| Inode vs name | Metadata (owner, mode, ACL) is in the inode; the name is in the directory |
| Hard link / symlink | Same inode / path to a target (target's permissions apply) |
| File `r w x` | Read / change contents / run |
| Directory `r w x` | List / create-delete-rename (needs x) / enter and reach |
| Path traversal | `x` on every directory on the path; directories protect what's below |
| Deleting | Controlled by the directory, not the file |
| Class selection | Owner → group → other; first match wins, not combined |
| `chown` / `chgrp` / `chmod` | Root / owner to own groups / owner or root |
| `mv` vs `cp` | `mv` keeps owner, group, mode, ACL; `cp` makes a new file |
| Capital `X` | Execute for directories only — safe recursive changes |
| umask | Removes bits from new files; per process, inherited; services default `022` |
| SGID directory | New files inherit the directory's group (not its permission bits) |
| Sticky | Only owners delete their own files in a shared directory |
| ACL | Named users/groups; mask limits them; `+` in `ls -l` |
| `chmod` + ACL | Changes the mask — can silently remove ACL access |
| Default ACL | Inherited by new files; replaces umask |
| `chattr +i` / `+a` | Immutable / append-only, even for root |
| Mount options | `ro`, `noexec`, `nosuid`, `nodev` override file bits |
| Kernel order | Path → mount/attributes → capabilities → owner/ACL/group/other → AppArmor/SELinux |
| App layout | Code and config read-only to the app; data/cache/logs/runtime writable |
| Debugging | `id` → `namei -l` → `stat` → `getfacl` → parent dir → `lsattr` → `findmnt` → MAC → test with `sudo -u` |

---

# Appendix — Command reference

## Inspecting

| Command | Purpose |
|---|---|
| `ls -l` / `ls -ld dir` / `ls -n` | Mode, owner, group / the directory itself / numeric IDs |
| `ls -i` | Inode numbers |
| `stat file` | Full metadata |
| `stat -c '%a %A %U:%G %n' files` | Compact mode + owner view |
| `namei -l /path` | Permissions along the whole path |
| `getfacl file` / `getfacl -R dir` | ACLs |
| `lsattr file` / `lsattr -d dir` | Attributes |
| `findmnt -T path` / `findmnt -o TARGET,OPTIONS` | Mount and its options |
| `umask` / `umask -S` | Current umask |
| `sudo -u user test -r/-w/-x path && echo yes` | Test access as another user |

## Changing ownership and mode

| Command | Purpose |
|---|---|
| `chown user:group file` / `-R` / `-h` | Change owner / recursive / symlink itself |
| `chgrp group file` | Change group |
| `chmod 640 file` / `chmod g+w,o= file` | Numeric / symbolic |
| `chmod -R u=rwX,g=rX,o= dir` | Safe recursive |
| `find dir -type d -exec chmod 755 {} +` | Directories only |
| `find dir -type f -exec chmod 644 {} +` | Files only |
| `chmod g+s dir` / `chmod +t dir` | SGID / sticky |
| `install -d -m 750 -o u -g g dir` | Create directory with owner and mode |
| `install -m 640 -o root -g app src dst` | Copy a file with owner and mode |

## umask

| Command | Purpose |
|---|---|
| `umask 027` | Set for this shell and children |
| `UMask=0027` | Set for a systemd service |

## ACLs

| Command | Purpose |
|---|---|
| `setfacl -m g:grp:rX path` | Add/modify a group entry |
| `setfacl -m u:user:rwX path` | Add/modify a user entry |
| `setfacl -m d:g:grp:rwX dir` | Default ACL for new content |
| `setfacl -R -m ...` | Recursive |
| `setfacl -x g:grp path` / `-b` / `-k` | Remove entry / all / default only |
| `getfacl -R dir > acls.txt` / `setfacl --restore=acls.txt` | Back up / restore ACLs |
| `cp -a`, `rsync -aAX`, `tar --acls --xattrs` | Keep ACLs when copying |

## Attributes and mounts

| Command | Purpose |
|---|---|
| `chattr +i` / `-i` | Immutable on / off |
| `chattr +a` / `-a` | Append-only on / off |
| `mount -o remount,noexec,nosuid,nodev /mnt` | Change mount options |
| `/etc/fstab` options column | Persistent mount options |

## Audits

| Command | Purpose |
|---|---|
| `find / -xdev -type f -perm -0002` | World-writable files |
| `find / -xdev -type d -perm -0002 ! -perm -1000` | World-writable dirs without sticky |
| `find / -xdev \( -nouser -o -nogroup \)` | Orphaned files |
| `find /etc -xdev -perm /022 -type f` | Group/other-writable files in `/etc` |
| `sysctl fs.protected_symlinks fs.protected_hardlinks` | Link protections |
| `sudo journalctl -k \| grep -i apparmor` | AppArmor denials |