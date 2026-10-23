# Filesystem Structure — Theory (Foundation Note)

Where everything on a Linux system lives, and *why* it lives there — so that when a later note says "the SSH keys are in `~/.ssh`" or "the unit files are in `/etc/systemd/system`", the location already makes sense and you can reason your way to a file you've never been told about.

**Series:**
- Note 0: Introduction & Lab Setup
- **Filesystem Structure — Theory** ← this note
- Filesystem Structure — Practice
- Note 1: Shell & Processes
- Note 2: Environment Variables
- Note 3 / 4: Users, Groups & Identity
- Note 5 / 6: Files, Directories & Permissions
- Note 7 / 8: Privilege & Delegation
- Note 9 / 10: Processes & Signals
- Note 11 / 12: systemd & Services
- Note 13 / 14: Logs, Timers & Resource Limits

This is a **foundation note**, read after the intro and before Note 1. It's about the *shape* of the system: the directory tree, how disks become one tree, and how paths work. It deliberately does **not** cover permissions in depth — that's Note 5 — only *where* things are and why.

The security companion to this note is `../linux-post-exploitation/00-linux-filesystem.md`, which takes the same layout and asks where findings hide. Each phase here ends with **"Used in"** and, where relevant, a short **"How this goes wrong"** pointer to that note.

---

## Contents

1. Phase 1 — The one idea: organised by purpose, not by application
2. Phase 2 — A tour of the root directories
3. Phase 3 — The "fake" filesystems: `/dev`, `/proc`, `/sys`
4. Phase 4 — One tree, many disks: filesystems and mounts
5. Phase 5 — Paths: absolute, relative, `.` and `..`
6. Phase 6 — Where a single application scatters itself
7. How this goes wrong — overview
8. Summary — directory purpose reference
9. Appendix — command reference

---

# Phase 1 — The one idea: organised by purpose, not by application

Windows organises files by **application**: nearly everything for a program sits under one folder in `Program Files`. Linux does the opposite. It organises by **purpose** and by **who manages the file**, and a single application's files are deliberately spread across several directories, each chosen by what *kind* of file it is.

So one web app might have:

- its program in `/usr/bin` (a binary),
- its configuration in `/etc` (settings a human edits),
- its changing data — logs, uploads, a database — in `/var` (data that grows),
- and a developer's copy in `/home/someone` (user files).

Once you see that, the tree becomes predictable. Every directory answers one of a few questions about a file:

| Question about a file | Where it goes |
|---|---|
| Is it a **program** to run? | `/bin`, `/sbin`, `/usr/bin`, `/usr/sbin`, `/usr/local/bin`, `/opt` |
| Is it **configuration** (text a human edits)? | `/etc`, or inside the app's own dir, or a user's home |
| Is it **data that changes** during operation (logs, queues, databases)? | `/var` |
| Does it belong to a **user**? | `/home/<user>`, or `/root` for root |
| Is it **temporary**? | `/tmp`, `/var/tmp`, `/run` |
| Is it a **kernel interface**, not a real file on disk? | `/dev`, `/proc`, `/sys` |

Two deeper distinctions run underneath this and explain oddities you'll meet:

- **Static vs variable.** Programs and libraries don't change during normal use (static). Logs, caches, and queues change constantly (variable). Linux keeps them apart so the static parts can even be mounted read-only. This is why programs live in `/usr` and their logs live in `/var`, never mixed.
- **Who manages it.** Some locations belong to the OS and its package manager (`/usr/bin`, `/lib`). Some belong to the local admin (`/etc`, `/usr/local`, `/opt`). Some belong to ordinary users (`/home`). A file's location tells you who put it there — which is exactly what makes a stray file stand out.

This layout isn't improvised per distro: it's written down in a standard, the **Filesystem Hierarchy Standard (FHS)**. `man 7 hier` is the on-system summary.

**Used in:** every note — each one puts its files where this phase predicts. **How this goes wrong:** the "who manages it" idea is the backbone of the post-ex note — a binary in a human-managed path is a lead; the same binary in `/usr/bin` is stock.

---

# Phase 2 — A tour of the root directories

Here's the whole top level, grouped by the buckets from Phase 1 rather than alphabetically, so the logic stays visible.

### Programs (binaries)

| Path | What lives here | Managed by |
|---|---|---|
| `/bin` | Essential user commands (`ls`, `cat`, `cp`) | OS / package manager |
| `/sbin` | Essential system/admin commands (`mount`, `ip`), usually root-only | OS / package manager |
| `/usr/bin`, `/usr/sbin` | The bulk of installed programs | OS / package manager |
| `/usr/local/bin`, `/usr/local/sbin` | Software installed **by hand**, outside the package manager | The local admin |
| `/opt` | Large self-contained third-party apps, each in its own subfolder | The local admin / vendor |

### Libraries

| Path | What lives here |
|---|---|
| `/lib` (and `/lib64` on x86_64) | Shared libraries the essential binaries need |
| `/usr/lib` | Shared libraries for everything else |

### Configuration

| Path | What lives here |
|---|---|
| `/etc` | **All** system-wide configuration, as editable text. No binaries live here. |

`/etc` is the control room. `/etc/passwd` (accounts), `/etc/ssh/` (SSH config), `/etc/cron.d/` (scheduled jobs), `/etc/sudoers` (who can sudo) all live here. To learn how a box is set up, you read `/etc`.

### Variable data (changes during operation)

| Path | What lives here |
|---|---|
| `/var/log` | Log files |
| `/var/spool` | Queues: cron jobs, mail, print jobs |
| `/var/lib` | Application state and databases (e.g. `/var/lib/postgresql`) |
| `/var/www` | Web content, on many servers |
| `/var/backups` | System backups |
| `/var/cache` | Caches a program can rebuild |
| `/var/tmp` | Temporary files that survive a reboot |

The rule for `/var`: if it's data a running service produces or grows over time, it's here.

### User and temporary space

| Path | What lives here |
|---|---|
| `/home/<user>` | Each normal user's home directory |
| `/root` | Root's home — deliberately **not** in `/home`, so it stays on the root partition and is reachable when `/home` isn't mounted |
| `/tmp` | Temporary files, world-writable, often wiped on reboot |
| `/run` | Runtime data created since boot: PIDs, sockets. Lives in memory, gone on reboot |

### Kernel, boot, and mounts

| Path | What lives here |
|---|---|
| `/boot` | The kernel image and bootloader |
| `/dev` | Device files (Phase 3) |
| `/proc`, `/sys` | Kernel and process information (Phase 3) |
| `/mnt`, `/media` | Mount points for extra or removable filesystems (Phase 4) |
| `/srv` | Data served by the machine for a specific service |

### The `/usr` merge

On modern Ubuntu, `/bin`, `/sbin`, and `/lib` (plus `/lib64` on x86_64) are **symlinks** into `/usr`:

```
/bin  -> usr/bin
/sbin -> usr/sbin
/lib  -> usr/lib
```

The old split of "essential" (`/bin`, `/sbin`) from "the rest" (`/usr`) was historical — the essentials had to sit on a small root partition so the machine could boot and be repaired before `/usr` was mounted. That need is gone, so the directories were merged. The names still work exactly as before, so nothing you type changes.

**Used in:** Note 1 (`PATH` and where programs are found), Note 5 (`/etc` config, `/var` ownership), Notes 11–14 (`/etc/systemd`, `/var/log`, `/var/lib`).

---

# Phase 3 — The "fake" filesystems: `/dev`, `/proc`, `/sys`

Three of those directories don't hold real files on a disk. They're windows into the kernel, presented as files so ordinary tools like `cat` can read them. Understanding this unlocks a lot later, so it's worth a moment.

- **`/dev`** — device files. Each entry stands for a piece of hardware or a kernel device. `/dev/sda` is your first disk *as a raw stream of bytes*; `/dev/null` throws away whatever you write to it; `/dev/urandom` gives random bytes. They have permissions and types like any file (Note 0, Phase 3.2).
- **`/proc`** — one folder per running process (named by PID), plus system-wide kernel info. Nothing here is stored; it's generated the instant you read it:
  - `/proc/<pid>/environ` — the environment a process was started with
  - `/proc/<pid>/cmdline` — its exact command line, arguments included
  - `/proc/version` — the kernel version
  - `/proc/mounts` — what's mounted right now (Phase 4)
- **`/sys`** — a structured view of kernel and device settings (`/sys/class/net/` lists network interfaces). Same idea as `/proc`, less used day to day.

The lesson: these directories aren't storage, they're the kernel talking to you through files. When you want to know what a process is *doing right now*, `/proc` is where you look.

**Used in:** Note 9, Phase 2 (`/proc/<pid>` in depth), Note 0 Phase 3 (everything-is-a-file). **How this goes wrong:** `/proc/<pid>/environ` and `cmdline` leak secrets passed at launch; `/dev/sda` readable by the `disk` group bypasses every file permission. Both are in the post-ex note and Note 10.

---

# Phase 4 — One tree, many disks: filesystems and mounts

Phase 2 showed one tree starting at `/`. But a machine usually has more than one storage area — a disk, a second disk, a USB stick, a network share. How do they all appear under one `/`? This phase builds the answer from the bottom up: what a disk really is, how it becomes files, and how those get joined into one tree. Take it slowly — each layer sits on the one before.

## 4.1 What a disk actually is: a numbered array of blocks

Strip away everything and a disk is just **one huge row of numbered slots**, each the same small size (historically 512 bytes, now usually 4096). These slots are called **sectors** or **blocks**. The disk understands exactly two requests:

- "give me the contents of block number N"
- "write these bytes into block number N"

That's all. A disk has **no idea** what a file is, what a name is, or who owns anything. It's like a warehouse with millions of identical numbered shelves: you can put a box on shelf 814,320 or read what's on it, but the warehouse itself keeps no list of what's where or what belongs to whom.

```
a raw disk = blocks, nothing more
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬──── ...
│  0  │  1  │  2  │  3  │  4  │  5  │  6  │  7  │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴──── ...
   each slot holds a fixed number of bytes; that's the whole machine
```

## 4.2 What a filesystem is: the bookkeeping on top

A **file** — a thing with a name, a size, an owner, and contents that can grow — doesn't exist at the disk level. Something has to *invent* it by keeping a ledger: "the file named `server.js` is owned by `thirana`, is 1013 bytes, and its contents live in blocks 50 and 51." That ledger, plus the rules for keeping it, **is the filesystem**.

So a filesystem is the **bookkeeping layer** written into the blocks themselves. It sets aside some blocks for the ledger (names, owners, sizes, timestamps, and which blocks hold which file — Note 5 calls these records *inodes*) and leaves the rest for file contents. With it, the kernel can turn "open `/opt/api/server.js`" into "read blocks 50 and 51".

**Formatting** a disk means writing a fresh, empty version of this ledger — which is why formatting "erases" everything: the contents blocks are still physically there, but the ledger that named and found them is blank.

Linux supports several filesystem formats — different ledger designs with different trade-offs:

| Filesystem | Where you meet it |
|---|---|
| `ext4` | The Ubuntu default for disks |
| `xfs`, `btrfs` | Other Linux disk formats |
| `vfat` | USB sticks, SD cards, the EFI boot partition (works on every OS) |
| `tmpfs` | A filesystem that lives in **RAM**, not on a disk — `/run` uses it, so it vanishes on reboot |

The key idea: **a disk gives you blocks; a filesystem gives you files.** You always need a filesystem on top of the raw blocks before you can store a single named file.

## 4.3 Partitions: dividing one disk into independent regions

You usually don't put one filesystem across a whole physical disk. First you **partition** it: split that one long row of blocks into a few separate regions, each of which can hold its own filesystem, independently.

```
one physical disk /dev/sda, split into partitions:
┌───────────────────────────┬──────────────┬──────────────┐
│  /dev/sda1  (11G, ext4)    │ sda16 (boot) │ sda15 (EFI)  │
│  → becomes the / tree      │  ext4        │  vfat        │
└───────────────────────────┴──────────────┴──────────────┘
```

Why split at all? A few common reasons:
- **Different jobs need different formats.** The EFI firmware that starts the machine can only read `vfat`, so `/boot/efi` is a small `vfat` partition, while the main system is `ext4`.
- **Isolation.** If logs in one partition fill it completely, the others still have room (Note 14).
- **Different rules per region.** One partition can be mounted read-only or `noexec` (4.5) without affecting the rest.

The partitions show up as `/dev/sda1`, `/dev/sda2`, and so on — the disk is `sda`, each partition a number on it. Each one is now a blank region ready for a filesystem.

## 4.4 Mounting: joining filesystems into one tree

Now the payoff. You have, say, three partitions, each with its own filesystem. Windows would give each a drive letter (`C:`, `D:`, `E:`) — three separate trees. **Linux has no drive letters.** It has exactly **one** tree, starting at `/`, and you graft each filesystem onto a **directory** inside it. That attaching is called **mounting**, and the directory you attach to is the **mount point**.

Once a filesystem is mounted on a directory, that directory *becomes* that filesystem — opening anything underneath it reads from that filesystem's blocks.

```
one tree, assembled from separate filesystems:

/                    ← sda1's filesystem is mounted here (the "root filesystem")
├── etc/             ─┐
├── home/            ─┤ all still on sda1
├── opt/             ─┘
├── boot/            ← sda16's filesystem is mounted here
│   └── efi/         ← sda15's (vfat) is mounted here
└── mnt/
    └── usb/         ← plug in a stick, mount it here, and this dir becomes the stick
```

So when you `cd /boot/efi`, you've silently crossed from `sda1`'s filesystem onto `sda15`'s — same seamless tree, different physical storage underneath. Run `mount /dev/sdb1 /mnt/usb` and writing to `/mnt/usb/file` writes to the stick; `umount` it and `/mnt/usb` is just an empty directory again.

One tree. The question is never "which drive?" but "which filesystem is mounted at this point?".

## 4.5 Seeing what's mounted

`lsblk` ("list block devices") shows the whole picture from 4.1–4.4 at once — the physical disk, the partitions on it, and where each is mounted:

```bash
lsblk
```
```
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda       8:0    0   12G  0 disk
├─sda1    8:1    0   11G  0 part /
├─sda15   8:15   0   99M  0 part /boot/efi
└─sda16 259:0    0  923M  0 part /boot
```

Read it top-down: one disk `sda` (12G), split into three partitions, and each partition's filesystem mounted at a point in the one tree — `sda1` at `/`, the others at `/boot` and `/boot/efi`. That's 4.1 (disk) → 4.3 (partitions) → 4.4 (mounts) in four lines.

More views of the same thing:

```bash
df -h                    # mounted filesystems and how full each is
findmnt                  # the full mount tree (clearest view)
mount | column -t        # every mount with its options (ro, noexec, ...)
cat /proc/mounts         # the same, straight from the kernel (Phase 3)
```

Mount **options** matter later: a filesystem can be mounted read-only (`ro`), or so that programs on it can't run (`noexec`) or gain privilege (`nosuid`) — Note 6 uses these to lock down an uploads folder.

## 4.6 Mounts at boot: `/etc/fstab`

Mounts that should exist on every boot are listed in `/etc/fstab` (it's configuration, so it's in `/etc`). Each line is: what to mount, where, the filesystem type, and the options.

```
# <device>              <mount point>  <type>  <options>        <dump> <pass>
LABEL=cloudimg-rootfs   /              ext4    defaults          0      1
LABEL=UEFI              /boot/efi      vfat    umask=0077        0      1
```

Devices are named by **LABEL** or **UUID** rather than `/dev/sda1`, because device names (`sda`, `sdb`) can change between boots while a label or UUID doesn't. (`lsblk -f` shows each partition's label and UUID.)

**Used in:** Note 6 (`noexec`/`nosuid` mount options, the Docker bind mount), Note 14 (disk filling up — a full filesystem is one mount). **How this goes wrong:** `/etc/fstab` can hold credentials for network shares, and `df`/`mount` reveal other filesystems worth exploring — see the post-ex note.

---

# Phase 5 — Paths: absolute, relative, `.` and `..`

A **path** is how you name a file's place in the tree. Getting this straight prevents a whole class of "file not found" and "why did it run *that*" confusion.

## 5.1 Absolute vs relative

| Kind | Starts with | Meaning | Example |
|---|---|---|---|
| **Absolute** | `/` | From the root of the tree. Means the same thing from anywhere | `/etc/ssh/sshd_config` |
| **Relative** | anything else | From your **current directory** (`pwd`) | `ssh/sshd_config` (if you're in `/etc`) |

```bash
pwd                      # where am I now?
cd /etc                  # absolute: always goes to the same place
cd ssh                   # relative: /etc + ssh = /etc/ssh
```

## 5.2 The special names

| Name | Means |
|---|---|
| `.` | The current directory |
| `..` | The parent directory |
| `~` | Your home directory (`/home/thirana`); a shell shorthand, not a real entry |
| `-` | With `cd`: the previous directory you were in |

```bash
cd /var/log
cd ..                    # now in /var
cat ./syslog             # ./ means "in this directory" — rarely needed, but explicit
cd -                     # back to /var/log
```

`.` and `..` are **real directory entries** present in every directory (you'll see them with `ls -a`), not shell tricks.

## 5.3 Why `./` matters for running programs

To run a program, the shell searches the directories in `$PATH` (Note 1). Your current directory is **not** on `$PATH`, so to run a script sitting right next to you, you must name it with a path:

```bash
./deploy.sh              # "the deploy.sh in this directory"
deploy.sh                # "command not found" — the shell searched $PATH, not here
```

**Used in:** Note 1, Phase 6 (`./script` vs `script`, the shebang), Note 1 Phase 2 (how `$PATH` is searched). **How this goes wrong:** if `.` ever *is* on root's `$PATH`, or a program uses a relative path for a helper, an attacker can drop a lookalike in the working directory — PATH hijacking, in the post-ex note and Note 7.

---

# Phase 6 — Where a single application scatters itself

Because Linux splits by purpose, one service leaves traces in several places at once. Knowing the pattern means finding one piece tells you where the others are. A typical web application:

| Piece | Typical location | Why there |
|---|---|---|
| The program / binary | `/usr/bin` or `/opt/<app>` | it's a program |
| Its config, with DB credentials | `/etc/<app>/` or an `.env` in its own dir | it's configuration |
| Its logs | `/var/log/<app>/` | it's variable data |
| Its database files | `/var/lib/<db>/` | it's variable data |
| Web files it serves | `/var/www/` | served data |
| A developer's checkout | `/home/<user>/<app>/` | a user's files |
| Runtime socket / PID | `/run/<app>/` | runtime data |

So if you find the config in `/etc`, you already know to check `/var/log` for what it's been doing and `/var/lib` or `/var/www` for its data. One find points at the next.

This is exactly how our scenario's apps are laid out: the API's code in `/opt/api`, its config in `/etc/myapps/`, its logs via the journal and `/var/log/myapps/`, uploads in `/srv/uploads`. The practice note traces one app across the tree.

**Used in:** Note 2 (env files in `/etc/myapps`), Note 6 (the full folder layout), Notes 12–14 (where services, logs, and backups live).

> **Inodes, links, and permissions** are the other half of "what a file is" — the metadata behind each name, hard vs symbolic links, and the `rwx` bits. That's **Note 5**, which goes deep on it. This note stops at *where* files are; Note 5 covers *what a file is and who may touch it*.

---

# How this goes wrong — overview

A short bridge to `../linux-post-exploitation/00-linux-filesystem.md`, which treats the whole layout as a map of where findings hide. The one idea to carry across:

> **Distro-managed locations are expected and boring; human-managed locations (`/etc`, `/opt`, `/usr/local`, `/home`, `/tmp`) are where mistakes, secrets, and backdoors live.**

| Layout fact (this note) | Why it matters once someone's on the box |
|---|---|
| Who manages each path (Phase 1) | A SUID binary in `/usr/bin` is stock; in `/opt` or a home it's a lead |
| `/proc` is the kernel's live view (Phase 3) | `environ` and `cmdline` leak secrets no file on disk holds |
| `/dev/sda` is the raw disk (Phase 3) | Read access bypasses every file permission |
| Other filesystems are mounted (Phase 4) | `df`/`mount`/`fstab` reveal shares and disks to explore |
| Current dir isn't on `$PATH` (Phase 5) | …unless someone added it — then a dropped lookalike runs |

---

# Summary — directory purpose reference

| Path | Purpose | Managed by |
|---|---|---|
| `/bin`, `/sbin`, `/usr/bin`, `/usr/sbin` | Programs | OS |
| `/usr/local/bin`, `/opt` | Hand-installed programs | Admin |
| `/etc` | System configuration (text) | OS + admin |
| `/var/log` | Logs | services |
| `/var/spool` | Queues (cron, mail) | services |
| `/var/lib` | App state / databases | services |
| `/var/www` | Web content | admin |
| `/var/backups` | Backups | system |
| `/home/<user>` | User files | users |
| `/root` | Root's home | root |
| `/tmp`, `/var/tmp` | Temporary, writable | anyone |
| `/boot` | Kernel and bootloader | OS |
| `/dev` | Device files | kernel |
| `/proc`, `/sys` | Live process/kernel info | kernel |
| `/run` | Runtime data (PIDs, sockets) | system |
| `/mnt`, `/media` | Mount points | admin / system |

| Concept | Key point |
|---|---|
| Organised by purpose | Program / config / variable data / user / temporary / kernel interface |
| Static vs variable | `/usr` doesn't change; `/var` does — kept apart |
| Who manages it | OS / admin / user — tells you who placed a file |
| FHS | The standard behind the layout; `man 7 hier` |
| `/usr` merge | `/bin`, `/sbin`, `/lib` are symlinks into `/usr` |
| Fake filesystems | `/dev`, `/proc`, `/sys` are kernel windows, not disk storage |
| Filesystem | On-disk format turning blocks into files (`ext4`, `tmpfs`, …) |
| Mount | Grafting a filesystem onto a directory; no drive letters |
| `/etc/fstab` | Mounts set up at every boot, by UUID |
| Absolute vs relative path | From `/` vs from the current directory |
| `.` `..` `~` `-` | This dir / parent / home / previous dir |
| `./program` | Needed because the current dir isn't on `$PATH` |

---

# Appendix — command reference

## Looking around the tree

```bash
ls -l /                        # the top-level directories
ls -la ~                       # a home directory, including dotfiles
tree -L 2 /etc                 # two levels deep (install: sudo apt install tree)
file /bin /etc/passwd /dev/null   # what kind of thing is this?
stat /etc/passwd               # size, owner, timestamps, inode
man 7 hier                     # the on-system description of the layout
```

## Disks and mounts

```bash
lsblk                          # disks and partitions as a tree
df -h                          # mounted filesystems and free space
findmnt                        # the full mount tree
mount | column -t              # every mount and its options
cat /etc/fstab                 # mounts set up at boot
cat /proc/mounts               # live mounts, from the kernel
```

## Paths

```bash
pwd                            # current directory
cd /etc && cd ssh              # absolute then relative
cd ..                          # up one level
cd -                           # back to the previous directory
realpath ./file                # resolve a relative path to an absolute one
readlink -f /bin               # follow symlinks to the real target
```

## The fake filesystems

```bash
cat /proc/version              # kernel version
cat /proc/mounts               # current mounts
ls /proc/$$                    # your shell's own process directory
ls /sys/class/net              # network interfaces
ls -l /dev/null /dev/sda       # device files and their types
```
