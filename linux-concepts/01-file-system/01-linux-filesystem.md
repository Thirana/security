# Linux Filesystem Structure — for Post-Exploitation

This is a foundation note for the Phantom track. Its job is not to make you memorise a
list of directories. It is to make the layout *make sense*, so that when a later note
says "the SSH keys will be in `~/.ssh`" or "check `/etc/cron.d`", you already understand
why they would be there — and can reason your way to where a file lives even when nobody
tells you.

The note builds up in order: first the single idea that explains the whole layout, then
a high-level tour, then the special "fake" filesystems, and finally the part that
matters most for us — where post-exploitation findings live and the logic that leads you
to them.

---

## The whole tree at a glance

Keep this map handy. Later notes will send you to specific paths; this shows where each
one sits and how they relate. The `[!]` marks a **high-value spot for
post-exploitation** — where secrets, escalation paths, or intel tend to hide. The logic
behind the layout is explained in the phases below; this is just the picture.

```text
/
├── bin/          essential user commands   (symlink to /usr/bin on modern Ubuntu)
├── boot/         kernel image and bootloader
├── dev/          device files (a kernel interface, not real files on disk)
│   └── sda       [!] the raw disk — reading it directly bypasses file permissions
├── etc/          system-wide configuration, as text (the control room)
│   ├── passwd            account list (world-readable)
│   ├── shadow           [!] password hashes (root / shadow group only)
│   ├── sudoers          [!] who may run sudo
│   ├── sudoers.d/       [!] drop-in sudo rules
│   ├── crontab          system-wide cron schedule
│   ├── cron.d/          [!] drop-in cron jobs (run as root)
│   ├── cron.daily/ …    scheduled-script directories (hourly/daily/weekly/monthly)
│   ├── ssh/             SSH server configuration
│   ├── systemd/system/  [!] admin service & timer units (override package ones)
│   ├── fstab            what gets mounted at boot (may leak share credentials)
│   ├── hosts            static hostname-to-IP entries
│   └── resolv.conf      DNS servers
├── home/         normal users' home directories
│   └── <user>/
│       ├── .ssh/            [!] private keys, known_hosts, authorized_keys
│       ├── .aws/            [!] AWS credentials
│       ├── .config/gcloud/  [!] GCP credentials
│       ├── .bash_history    [!] commands typed (sometimes passwords)
│       └── project/.env     [!] app configs with secrets
├── lib/, lib64/  essential shared libraries   (symlink to /usr/lib)
├── media/        auto-mounted removable media
├── mnt/          manual / temporary mount points
├── opt/          [!] hand-installed third-party software (odd SUID or scripts here = finding)
├── proc/         virtual FS: live process & kernel info (generated when you read it)
│   ├── version          kernel version
│   ├── mounts           currently mounted filesystems
│   ├── net/tcp          network connections, readable as a file
│   └── <pid>/
│       ├── environ      [!] a process's environment variables (secrets from launch)
│       └── cmdline      [!] the full command line, including arguments
├── root/         [!] root's home directory (the prize, after escalation)
├── run/          runtime data since boot: PIDs, sockets (lives in memory)
├── sbin/         essential admin/system commands   (symlink to /usr/sbin)
├── srv/          data served by the machine (web / ftp roots, etc.)
├── sys/          virtual FS: kernel & device settings
├── tmp/          [!] world-writable temporary space (working area; dropped tools)
├── usr/          the main program hierarchy (static, shareable)
│   ├── bin/             most user commands (stock SUID here is normal)
│   ├── sbin/            admin/system commands
│   ├── lib/             libraries for /usr programs
│   ├── local/          [!] software installed by hand, outside the package manager
│   │   └── bin/        [!] an odd SUID binary here is a classic finding
│   └── share/           docs, man pages, architecture-independent data
└── var/          variable data that grows and changes during operation
    ├── log/            [!] log files (activity; sometimes leaked secrets)
    ├── spool/
    │   └── cron/crontabs/   [!] per-user cron jobs
    ├── lib/            [!] application state & databases (mysql/, postgresql/)
    ├── www/            [!] web content and app .env files
    ├── backups/        [!] system backups (often unprotected credentials)
    ├── cache/          application cache
    └── tmp/            temporary data that survives a reboot
```

One thing you will notice on the real box: `/bin`, `/sbin`, and `/lib` are symbolic
links pointing into `/usr` (`/bin → /usr/bin`, and so on). Modern distributions merged
them, so the "essential vs the rest" split described below is now historical — but the
directory names still work exactly as before, so nothing you type changes.

---

## Phase 1: The one idea behind the whole layout

Windows organises files by *application*: everything for a program tends to sit under
one folder in `Program Files`. Linux does the opposite. It organises files by **purpose**
and by **who manages them** — and a single application's files are deliberately scattered
across several directories, each chosen by what *kind* of file it is.

So one web app might have:

- its program in `/usr/bin` (a binary),
- its configuration in `/etc` (settings a human edits),
- its changing data — logs, uploads, a database — in `/var` (data that grows and changes),
- and a person's copy of it in `/home/someone` (user files).

Once you see that, the whole tree becomes predictable. Every directory answers one of a
few questions about a file:

| Question about a file | Where it goes |
|---|---|
| Is it a **program** to run? | `/bin`, `/sbin`, `/usr/bin`, `/usr/sbin`, `/usr/local/bin`, `/opt` |
| Is it **configuration** (text a human edits)? | `/etc`, or inside the app's own dir, or a user's home |
| Is it **data that changes** during operation (logs, queues, databases)? | `/var` |
| Does it belong to a **user**? | `/home/<user>`, or `/root` for root |
| Is it **temporary**? | `/tmp`, `/var/tmp`, `/run` |
| Is it a **kernel interface**, not a real file on disk? | `/dev`, `/proc`, `/sys` |

Two more distinctions run underneath this and explain some oddities you will meet:

- **Static vs variable.** Programs and libraries do not change during normal use
  (static). Logs, caches, and spools change constantly (variable). Linux keeps them
  apart so the static parts can even be mounted read-only. This is why programs live in
  `/usr` and their logs live in `/var`, never mixed.
- **Who manages it.** Some locations are owned by the operating system and its package
  manager (`/usr/bin`, `/lib`). Some are for the local administrator (`/etc`,
  `/usr/local`, `/opt`). Some are for ordinary users (`/home`). **This one is the single
  most useful idea for post-exploitation** — we come back to it in Phase 6 — because a
  file's location tells you who put it there, and that tells you whether it is normal or
  suspicious.

---

## Phase 2: A high-level tour of the root directories

Here is the whole top level, grouped by the buckets from Phase 1 rather than
alphabetically, so the logic stays visible.

### Programs (binaries)

| Path | What lives here | Managed by |
|---|---|---|
| `/bin` | Essential user commands (`ls`, `cat`, `cp`) needed even in single-user mode | OS / package manager |
| `/sbin` | Essential system/admin commands (`mount`, `ip`, `fdisk`), usually root-only | OS / package manager |
| `/usr/bin`, `/usr/sbin` | The bulk of installed programs | OS / package manager |
| `/usr/local/bin`, `/usr/local/sbin` | Software installed **by hand**, outside the package manager | The local admin |
| `/opt` | Large self-contained third-party apps, each in its own subfolder | The local admin / vendor |

The split of "essential" (`/bin`, `/sbin`) from "the rest" (`/usr`) is historical: the
essential commands had to be on the small root partition so the system could boot and be
repaired before `/usr` was mounted.

### Libraries

| Path | What lives here |
|---|---|
| `/lib`, `/lib64` | Shared libraries the essential binaries need |
| `/usr/lib` | Shared libraries for everything else |

### Configuration

| Path | What lives here |
|---|---|
| `/etc` | **All** system-wide configuration, as editable text files. No binaries live here. |

`/etc` is the control room of the machine. `/etc/passwd` (accounts), `/etc/ssh/`
(SSH server config), `/etc/cron.d/` (scheduled jobs), `/etc/sudoers` (who can sudo) all
live here. If you want to know how the box is set up, you read `/etc`.

### Variable data (changes during operation)

| Path | What lives here |
|---|---|
| `/var/log` | Log files |
| `/var/spool` | Queues: cron jobs (`/var/spool/cron`), mail, print jobs |
| `/var/lib` | Application state and databases (e.g. `/var/lib/mysql`) |
| `/var/www` | Web content, on many servers |
| `/var/backups` | System backups |
| `/var/tmp` | Temporary files that survive a reboot |

The rule for `/var`: if it is data a running service produces or grows over time, it is
here.

### User and temporary space

| Path | What lives here |
|---|---|
| `/home/<user>` | Each normal user's home directory |
| `/root` | Root's home directory — deliberately **not** in `/home`, so it stays on the root partition and is reachable when `/home` is not mounted |
| `/tmp` | Temporary files, world-writable, often wiped on reboot |
| `/run` | Runtime data created since boot: process IDs, sockets. Lives in memory, gone on reboot |

### Kernel, boot, and mounts

| Path | What lives here |
|---|---|
| `/boot` | The kernel image and bootloader |
| `/dev` | Device files (see Phase 3) |
| `/proc`, `/sys` | Kernel and process information (see Phase 3) |
| `/mnt`, `/media` | Mount points for extra or removable filesystems |
| `/srv` | Data served by the machine for a specific service |

---

## Phase 3: The "fake" filesystems — /dev, /proc, /sys

Three of those directories do not hold real files on a disk. They are windows into the
kernel, presented as files so that ordinary tools like `cat` can read them. Understanding
this unlocks several post-exploitation tricks, so it is worth a moment.

- **`/dev`** — device files. Each entry represents a piece of hardware or a kernel
  device. `/dev/sda` is your first disk *as a raw stream of bytes*. This is why the
  `disk` group is dangerous: read `/dev/sda` directly and you bypass every file
  permission on the disk, including `/etc/shadow`.
- **`/proc`** — one folder per running process (named by its process ID), plus
  system-wide kernel info. Nothing here is stored; it is generated on the fly when you
  read it. This is where a lot of live intelligence comes from:
  - `/proc/<pid>/environ` — the environment variables a process was started with (often
    holds secrets passed in at launch).
  - `/proc/<pid>/cmdline` — the exact command line, including arguments (a password
    passed with `-p` shows up here).
  - `/proc/version` — the kernel version.
  - `/proc/net/tcp` — network connections, readable as a file.
- **`/sys`** — a structured view of kernel and device settings. Less used in
  post-exploitation than `/proc`, but the same idea.

The lesson: these directories are not storage, they are the kernel talking to you through
files. When you want to know what a process is *doing right now*, `/proc` is where you
look, and no disk permission stands in your way for your own processes.

---

## Phase 4: Where a single application scatters itself

Because Linux splits by purpose, one service leaves traces in several places at once.
Knowing the pattern means that finding one piece tells you where the others are. A
typical web application on a server:

| Piece | Typical location | Why there |
|---|---|---|
| The program / binary | `/usr/bin` or `/opt/<app>` | it is a program |
| Its config, with DB credentials | `/etc/<app>/` or an `.env` in its own dir | it is configuration |
| Its logs | `/var/log/<app>/` | it is variable data |
| Its database files | `/var/lib/<db>/` | it is variable data |
| Web files it serves | `/var/www/` | served data |
| A developer's checkout of it | `/home/<user>/<app>/` | a user's files |
| Runtime socket / PID | `/run/<app>/` | runtime data |

So if you find the config in `/etc`, you already know to check `/var/log` for what it has
been doing and `/var/lib` or `/var/www` for its data. One find points at the next.

---

## Phase 5: The post-exploitation map — where findings live, and why

This is the part that matters for the track. Using the three buckets from the enumeration
note — escalation paths, credentials, and map/intel — here is where each type of finding
sits and the reasoning that puts it there.

### Escalation paths — "a way to become root"

| Finding | Where it lives | Why there |
|---|---|---|
| SUID binaries (normal) | `/usr/bin`, `/usr/sbin` | distro programs that are meant to be SUID |
| SUID binaries (**suspicious**) | `/usr/local/bin`, `/opt`, `/home/<user>` | a *human* installed it here, not the package manager — so a SUID here was a deliberate act or a mistake |
| Sudo rules | `/etc/sudoers`, `/etc/sudoers.d/` | it is configuration, so it lives in `/etc` |
| Cron jobs (system) | `/etc/crontab`, `/etc/cron.d/`, `/etc/cron.{hourly,daily,weekly,monthly}/` | scheduling config, so `/etc` |
| Cron jobs (per user) | `/var/spool/cron/crontabs/` | a user's changing data, so `/var/spool` |
| Systemd units (admin) | `/etc/systemd/system/` | admin-created config, so `/etc` — and these override package ones |
| Systemd units (package) | `/usr/lib/systemd/system/` | shipped by the OS, so under `/usr` |
| Scripts a root job runs | `/opt`, `/usr/local/bin`, `/home`, sometimes `/tmp` | wherever a human dropped a helper script |

The pattern to carry: **the config that grants power lives in `/etc`; the writable
script or binary that abuses it lives wherever a human put it** — and human-placed things
land in `/opt`, `/usr/local`, `/home`, or `/tmp`, never in the distro's own `/usr/bin`.

### Credentials — "passwords and keys I can read"

| Finding | Where it lives | Why there |
|---|---|---|
| Password hashes | `/etc/shadow` (root/shadow group only) | account config, kept in `/etc` but locked down |
| Account list | `/etc/passwd` (world-readable) | account config that programs must read |
| App credentials / `.env` | `/etc/<app>/`, `/opt/<app>/`, `/var/www/`, `/home/<user>/` | it is configuration, sitting with its app |
| SSH private keys | `/home/<user>/.ssh/`, `/root/.ssh/` | keys belong to a user, so they live in that user's home |
| Shell history | `/home/<user>/.bash_history`, `/root/.bash_history` | a user's own record, so in their home |
| Cloud credentials | `/home/<user>/.aws/`, `/home/<user>/.config/gcloud/` | per-user tool config, so in the home dir |
| Database files | `/var/lib/mysql/`, `/var/lib/postgresql/` | changing service data, so `/var/lib` |
| Backups (often unprotected) | `/var/backups/`, `/opt`, `/tmp`, a user's home | wherever the backup job was told to write |
| Secrets leaking into logs | `/var/log/` | logs are variable data in `/var` |

The pattern: **user secrets are in that user's home; application secrets are with the
application's config; system password hashes are in `/etc/shadow`.** The dotfolders in a
home (`.ssh`, `.aws`, `.config`) are the richest single spot, which is why every note
tells you to read home directories.

### Map and intel — "what is around me"

| Finding | Where it lives | Why there |
|---|---|---|
| Other users on the box | `/etc/passwd` | account config |
| Hostname and hosts file | `/etc/hostname`, `/etc/hosts` | network config in `/etc` |
| DNS servers | `/etc/resolv.conf` | network config |
| Other machines this box trusts | `/home/<user>/.ssh/known_hosts`, `authorized_keys` | SSH relationships live with the user's keys |
| Mounted / network filesystems | `/etc/fstab`, `/proc/mounts` | mount config in `/etc`; live mounts via `/proc` |
| Live processes and their secrets | `/proc/<pid>/environ`, `/proc/<pid>/cmdline` | the kernel's live view, in `/proc` |
| Kernel version | `/proc/version` | kernel info in `/proc` |

---

## Phase 6: The reasoning tool — how to guess where a file is

This is the payoff of the whole note. Two directions of thinking, and both come from the
"who manages it" idea in Phase 1.

**Given a file, decide if it belongs there.** The location tells you who put it there:

| If a file is in... | It was placed by... | So a surprise there means... |
|---|---|---|
| `/usr/bin`, `/usr/sbin`, `/lib` | the OS package manager | expected — boring, this is stock |
| `/etc` | the OS or the admin | config worth reading, but mostly normal |
| `/usr/local`, `/opt` | a human, by hand | **interesting** — someone chose to put this here |
| `/home/<user>` | that user | **interesting** — personal files, keys, history |
| `/tmp`, `/var/tmp` | anyone (world-writable) | **interesting** — often where an attacker or a sloppy script drops things |

The one-line version: **distro-managed locations are expected and boring; human-managed
locations are where mistakes, secrets, and backdoors live.** A SUID binary in `/usr/bin`
is probably fine; the same binary in `/usr/local/bin` or `/opt` is your finding. This is
why the enumeration sweep pays special attention to `/usr/local`, `/opt`, `/home`, and
`/tmp`.

**Given a goal, decide where to look.** Run the file through the Phase 1 questions:

- Need a **password or key**? It is a secret → a user's home (`~/.ssh`, `~/.aws`,
  `.bash_history`) or an app's config (`/etc/<app>`, `/opt/<app>`, `/var/www`).
- Need to know **what runs automatically**? It is scheduling config → `/etc/cron*`,
  `/etc/systemd/system`, `/var/spool/cron`.
- Need to know **what a running process is doing**? It is live kernel info → `/proc`.
- Need to know **what happened on the box**? It is variable data → `/var/log`.
- Need a **writable place to work**? → `/tmp`, `/var/tmp`, or `/dev/shm`.

---

## Summary — directory purpose reference

| Path | Purpose | Managed by | Post-ex value |
|---|---|---|---|
| `/bin`, `/sbin`, `/usr/bin`, `/usr/sbin` | Programs | OS | stock SUID here is normal |
| `/usr/local/bin`, `/opt` | Hand-installed programs | Admin | odd SUID / scripts here = finding |
| `/etc` | System configuration | OS + admin | sudoers, cron, ssh config, `/etc/shadow` |
| `/var/log` | Logs | services | activity, leaked secrets |
| `/var/spool/cron` | Per-user cron | users | scheduled jobs |
| `/var/lib` | App state / databases | services | database files |
| `/var/www` | Web content | admin | app code and `.env` files |
| `/var/backups` | Backups | system | often unprotected credentials |
| `/home/<user>` | User files | users | `.ssh`, `.aws`, history — richest spot |
| `/root` | Root's home | root | the prize, after escalation |
| `/tmp`, `/var/tmp` | Temporary, writable | anyone | working space; dropped tools |
| `/dev` | Device files | kernel | `/dev/sda` = raw disk (disk group) |
| `/proc` | Live process/kernel info | kernel | `environ`, `cmdline`, `version` |
| `/run` | Runtime data | system | sockets, PIDs |

## Summary — where each finding lives

| Looking for... | Look in... |
|---|---|
| Suspicious SUID binaries | `/usr/local/bin`, `/opt`, `/home` |
| Who can sudo | `/etc/sudoers`, `/etc/sudoers.d/` |
| Scheduled root jobs | `/etc/crontab`, `/etc/cron.d/`, `/etc/systemd/system/`, `/var/spool/cron/crontabs/` |
| Password hashes | `/etc/shadow` |
| SSH keys | `/home/<user>/.ssh/`, `/root/.ssh/` |
| Cloud credentials | `~/.aws/`, `~/.config/gcloud/` |
| App credentials | `/etc/<app>/`, `/opt/<app>/`, `/var/www/`, `~/` |
| Databases | `/var/lib/mysql/`, `/var/lib/postgresql/` |
| Live process secrets | `/proc/<pid>/environ`, `/proc/<pid>/cmdline` |
| Other hosts to move to | `~/.ssh/known_hosts`, `~/.ssh/authorized_keys` |
| Network filesystems | `/etc/fstab`, `/proc/mounts` |

**The habit to keep:** a file's location is not random — it is chosen by what kind of
file it is and who manages that location. Distro-managed paths are expected; human-managed
paths (`/etc`, `/opt`, `/usr/local`, `/home`, `/tmp`) are where the findings hide.