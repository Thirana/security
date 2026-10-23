# Introduction to Linux — Theory (Note 0)

What Linux actually is, how it's built, how to find help on the system itself, and how to read the rest of this series.

**Series:**
- **Note 0: Introduction — Theory** ← this note
- Note 0: Lab Setup with Multipass — Practice
- Filesystem Structure — Theory & Practice (foundation notes)
- Note 1: Shell & Processes
- Note 2: Environment Variables
- Note 3 / 4: Users, Groups & Identity
- Note 5 / 6: Files, Directories & Permissions
- Note 7 / 8: Privilege & Delegation
- Note 9 / 10: Processes & Signals
- Note 11 / 12: systemd & Services
- Note 13 / 14: Logs, Timers & Resource Limits

This note doesn't teach commands in depth — every later note does that. Its job is to give you the picture the rest hangs on: what the kernel is, where the line between the kernel and your programs sits, and the one sentence that the whole series keeps coming back to.

Each phase ends with **"Used in"** (where later notes build on it) and, where relevant, **"How this goes wrong"** for the security track.

---

## Contents

1. Phase 1 — What Linux is: kernel, operating system, distribution
2. Phase 2 — Kernel space and user space
3. Phase 3 — "Everything is a file"
4. Phase 4 — The one sentence behind the whole series
5. Phase 5 — Getting help on the system
6. Phase 6 — How to read this series
7. Phase 7 — The running scenario
8. Phase 8 — Why this matters for security
9. How this goes wrong — overview
10. Summary table
11. Appendix — Command reference

---

# Phase 1 — What Linux is: kernel, operating system, distribution

## 1.1 Linux is a kernel

Strictly, **Linux** is only the **kernel**: the program that runs with full control of the hardware and shares it out between everything else. Linus Torvalds started it in 1991; it's released under the GPLv2.

What the kernel does:

| Job | Meaning | Where you meet it |
|---|---|---|
| Processes | Starts, schedules, and stops programs | Note 9 |
| Memory | Gives each process its own private memory | Note 9, Phase 2 |
| Files | Talks to disks and filesystems | Note 5, filesystem note |
| Devices | Drivers for disks, network cards, terminals | `/dev`, filesystem note |
| Networking | TCP/IP, sockets, firewall (netfilter) | `ss`, Note 10 |
| Security | Checks **every** request against user IDs, permissions, capabilities | Notes 3, 5, 7 |

The kernel on its own can't do anything useful for you — there's no shell, no `ls`, no login. Those are ordinary programs.

## 1.2 Kernel + userland = an operating system

Everything that isn't the kernel is **userland** (or user space): the programs and libraries you actually run.

```
┌──────────────────────────────────────────────────────────┐
│  Your apps        node server.js, python -m worker       │
│  Services         sshd, nginx, cron, postgres            │
│  Tools            bash, ls, cp, ps, grep  (GNU coreutils)│  userland
│  Init / manager   systemd (PID 1)                        │
│  C library        glibc — wraps system calls             │
├──────────────────────────────────────────────────────────┤
│  Linux kernel     processes, memory, files, devices, net │  kernel
├──────────────────────────────────────────────────────────┤
│  Hardware         CPU, RAM, disk, network card           │
└──────────────────────────────────────────────────────────┘
```

Many of the basic tools (`bash`, `ls`, `cp`, the C library) come from the **GNU** project (started 1983), which is why some people say "GNU/Linux". The same Linux kernel with a **different** userland gives a very different system:

| System | Kernel | Userland |
|---|---|---|
| Ubuntu, Debian, RHEL | Linux | GNU tools, glibc, systemd |
| Alpine (common in containers) | Linux | BusyBox tools, musl libc, no systemd |
| Android | Linux | Android runtime, Bionic libc |

This is why a script that works on Ubuntu can break in an Alpine container: same kernel, different tools (Note 1, Phase 7 — `sh` vs `bash`).

## 1.3 Distributions

A **distribution** (distro) is a kernel + userland packaged together, plus the things that make it usable:

| A distro provides | Example (Ubuntu) |
|---|---|
| A package manager and repositories | `apt`, `.deb` files, `archive.ubuntu.com` |
| A release cycle and security updates | LTS every two years |
| Defaults: which init, which shell, which firewall | systemd, bash, `ufw` |
| Configuration conventions | `/etc/default/`, `sudo` group |

The two families you'll meet most on servers:

| | Debian family | Red Hat family |
|---|---|---|
| Distros | Debian, **Ubuntu** | RHEL, Fedora, Rocky, AlmaLinux, CentOS Stream |
| Packages | `apt`, `dpkg`, `.deb` | `dnf`, `rpm`, `.rpm` |
| Admin group | `sudo` | `wheel` |
| Main system log file | `/var/log/syslog` | `/var/log/messages` |
| Login/auth log | `/var/log/auth.log` | `/var/log/secure` |
| Mandatory access control | AppArmor | SELinux |
| Firewall front end | `ufw` | `firewalld` |
| SSH service name | `ssh` | `sshd` |
| Apache web server package | `apache2` | `httpd` |

The **concepts** in this series are the same on both. The names and paths in this table are what changes.

## 1.4 Ubuntu releases

This series uses **Ubuntu 24.04 LTS** ("Noble Numbat").

| Term | Meaning |
|---|---|
| Version number | Year.month of release: `24.04` = April 2024 |
| **LTS** | Long Term Support — every two years (April of even years), 5 years of standard security updates (longer with Ubuntu Pro) |
| Interim release | Every six months between LTS releases, supported for 9 months — not for servers |
| Point release | `24.04.1`, `24.04.2`, … — the same release with updates rolled in |

Servers run LTS releases. A server on an LTS release that's past its end of support doesn't get security fixes any more.

## 1.5 Which system am I on?

The first thing to check on any unknown machine:

```bash
cat /etc/os-release     # distro name and version
uname -r                # kernel version
uname -m                # CPU architecture
hostnamectl             # hostname, OS, kernel, and virtualization in one view
```
```
PRETTY_NAME="Ubuntu 24.04.5 LTS"
...
6.8.0-147-generic
aarch64
```

**Meaning:** Ubuntu 24.04, kernel 6.8, on 64-bit ARM (`aarch64`, as on an Apple Silicon Mac). On most cloud servers you'll see `x86_64`. Package names and binaries differ by architecture; commands and concepts don't.

**Used in:** filesystem note (where distro differences show up); every practice note assumes Ubuntu 24.04.

---

# Phase 2 — Kernel space and user space

## 2.1 Two privilege levels

The CPU is always running in one of two **modes**. A common misunderstanding is that only the kernel runs on the CPU and passes apps' work to it. It doesn't: **your apps run directly on the CPU**. When a Python script adds two numbers, the CPU runs those instructions itself, at full speed, with no kernel involved.

That's exactly why user mode exists. Apps run straight on the CPU, so the CPU itself must limit what they can do while they run:

| Mode | Runs | Allowed |
|---|---|---|
| **User mode** (x86 "ring 3", ARM "EL0") | Every app — **including root's** | Maths, loops, comparisons, its **own** memory. That's most of what any program does |
| **Kernel mode** (x86 "ring 0", ARM "EL1") | Only the kernel | Everything: devices, all memory, deciding which memory belongs to whom |

If an app tries a kernel-mode-only action in user mode (touch a device, read another program's memory), the CPU refuses and hands control to the kernel, which usually kills the app (`Segmentation fault` is one form of this).

### A normal moment

```
CPU in user mode    → runs your app's instructions directly (calculating, looping...)
app needs a file    → makes a system call → CPU switches to kernel mode
CPU in kernel mode  → runs kernel code: checks permissions, reads the disk
done                → CPU switches back to user mode → your app continues
```

It's the **same CPU** the whole time, switching thousands of times a second. The kernel isn't a separate chip. It's code the CPU runs in kernel mode. An app can't switch modes itself: the only ways in are a **system call** (the app asks, Phase 2.2), an **exception** (the app did something it may not do), or an **interrupt** (hardware needs attention), and each one lands at a fixed entry point in the kernel.

So an app never "calls the CPU". Its code simply **is** what the CPU runs in user mode. It calls the **kernel**, and only when it needs something user mode can't do.

### You can see the two modes

CPU time is counted separately for each mode. `time` shows both:

```bash
time python3 -c 'sum(range(10**8))'                        # pure calculation
time dd if=/dev/zero of=/dev/null bs=1M count=20000         # nothing but read/write requests
```
```
real 1.62s   user 1.60s   sys 0.01s       ← python: almost all user mode
real 1.10s   user 0.02s   sys 1.07s       ← dd: almost all kernel mode, doing the reads and writes for it
```

`user` = the CPU running the app's own code; `sys` = the CPU in kernel mode doing work the app asked for. (Your numbers will differ; the split is what matters.) `top` shows the same split as `us` and `sy` (Note 9, Phase 3.4).

### Why this matters

**Root is not kernel mode.** A root process is still a user-mode app that must ask the kernel for everything, and the kernel can still say no (capabilities, sandboxing — Notes 7, 8). The rules hold only while nothing but the kernel gets into kernel mode. A kernel exploit breaks exactly that (Phase 2.5).

## 2.2 System calls: the only door

A user-mode program can't open a file, send a packet, or start a process by itself. It has to **ask the kernel**, through a **system call** (syscall). Linux has a few hundred of them.

| You want to | Syscall(s) |
|---|---|
| Open, read, write a file | `openat`, `read`, `write`, `close` |
| Start a program | `fork`/`clone`, then `execve` (Note 1, Phase 2.2) |
| Change who you are | `setuid`, `setgid` (Note 7) |
| Send a signal | `kill` (Note 9) |
| Listen on a port | `socket`, `bind`, `listen` |

You can watch the requests a program makes with `strace`:

```bash
strace -e trace=openat,read,write cat /etc/hostname
```
```
...                                          ← first, cat's own libraries and locale files
openat(AT_FDCWD, "/etc/hostname", O_RDONLY) = 3
read(3, "app-server-01\n", 131072)          = 14
write(1, "app-server-01\n", 14)             = 14
app-server-01
```

**Meaning:** before doing any work, `cat` opens its shared libraries and locale files, and even that goes through syscalls. Then it asked the kernel to open the file (got file descriptor `3`), read it, and write the bytes to file descriptor `1` (your terminal, Note 9 Phase 2.3). Every command in this series boils down to requests like these.

## 2.3 The kernel checks every request

Each syscall is a request **from a process**, and the kernel decides yes or no:

```
process (running as UID 1101) ──openat("/etc/shadow")──▶ kernel
                                                          │ who is asking?  UID 1101, groups devs, ssh-users
                                                          │ what does the file allow?  root:shadow 640
                                                          ▼
                                                   -1 EACCES  "Permission denied"
```

"Permission denied" isn't the program refusing. It's the kernel refusing the program. The rules it uses are what Notes 3–8 are about:
- **who** the process is (UID, groups — Note 3)
- **what** the file allows (permissions, ACLs — Note 5)
- the **exceptions** (root, SUID, capabilities, sudo — Note 7)

## 2.4 Kernel version and modules

```bash
uname -r                 # running kernel
ls /boot                 # installed kernels (vmlinuz-*, initrd.img-*)
lsmod | head             # loaded kernel modules (drivers)
```

The kernel can load **modules** (drivers, filesystems) while running. Updating the kernel package doesn't change the **running** kernel until you reboot. `uname -r` shows what's running now; `ls /boot` shows what's installed.

## 2.5 Containers share the kernel

A Docker container is **not** a separate kernel. It's ordinary processes on the host's kernel, given:
- **namespaces** — their own view of PIDs, network, mounts, hostnames
- **cgroups** — limits on CPU, memory, processes (Note 13, Phase 9)

`uname -r` inside a container shows the **host's** kernel. This is why a container's PID 1 needs care (Note 9, Phase 5.3) and why the same UID numbers mean the same thing inside and outside (Note 3, Phase 10.3).

> **How this goes wrong:** everything in Phase 2.3 relies on the kernel being correct. A **kernel exploit** skips every check at once — one bug, and a normal user becomes root, or a container process takes over the host. Famous examples: Dirty COW (CVE-2016-5195), Dirty Pipe (CVE-2022-0847). That's why `uname -r` is one of the first commands in enumeration, and why "update and **reboot**" matters on servers.

**Used in:** Note 1 (fork/exec), Note 7 (setuid, capabilities), Note 9 (`/proc`, signals), Note 13 (cgroups).

---

# Phase 3 — "Everything is a file"

## 3.1 The idea

Unix's big design idea: use **one interface** — open, read, write, close — for as many things as possible. Not only documents, but devices, kernel information, and connections between programs too.

| Thing | Looks like | Example |
|---|---|---|
| A document | A regular file | `/etc/hostname` |
| A disk | A device file | `/dev/sda` |
| "Throw it away" | A device file | `/dev/null` |
| Random numbers | A device file | `/dev/urandom` |
| A process's details | Files under `/proc` | `/proc/1234/cmdline` |
| Kernel settings | Files under `/proc/sys` and `/sys` | `/proc/sys/kernel/pid_max` |
| A pipe between programs | A FIFO or an anonymous pipe | `mkfifo /tmp/q` |
| A local connection | A socket file | `/run/systemd/journal/dev-log` |

So the tools you already know work on all of them:

```bash
cat /proc/loadavg                    # read kernel info like a file
cat /proc/sys/kernel/hostname        # a kernel setting, as a file
echo hello > /dev/null               # write to a device
head -c 16 /dev/urandom | xxd        # read from a device
```

## 3.2 File types in `ls -l`

The first character of `ls -l` tells you which kind of "file" it is:

| Char | Type | Example |
|---|---|---|
| `-` | Regular file | `/etc/passwd` |
| `d` | Directory | `/etc` |
| `l` | Symbolic link | `/bin -> usr/bin` |
| `c` | Character device (byte stream) | `/dev/null`, `/dev/tty` |
| `b` | Block device (disk) | `/dev/sda` |
| `p` | Named pipe (FIFO) | made with `mkfifo` |
| `s` | Socket | `/run/systemd/journal/dev-log` |

```bash
ls -l /dev/null /dev/sda /dev/log
```
```
crw-rw-rw- 1 root root 1, 3 ... /dev/null
brw-rw---- 1 root disk 8, 0 ... /dev/sda
lrwxrwxrwx 1 root root   28 ... /dev/log -> /run/systemd/journal/dev-log
```

**Meaning:** devices have **permissions like any file**. `/dev/sda` is readable by group `disk`, so anyone in `disk` can read the raw disk and skip every file permission on it (filesystem note, Phase 3).

## 3.3 Not quite everything

Some things are **not** files: network interfaces (`eth0`, `enp0s1`) have no entry in `/dev` — you reach the network through sockets, and see interfaces through `/sys/class/net/` and `ip`. Processes aren't files either; `/proc/<PID>/` is a **view** of one that the kernel generates when you look at it.

The filesystem note's Phase 3 covers `/dev`, `/proc`, and `/sys` in detail.

> **How this goes wrong:** because so much is a file, **file permissions protect much more than documents**. A device that's too open (`/dev/sda` readable), a socket that's too open (`/var/run/docker.sock` writable), or a `/proc` file that's readable (`/proc/<PID>/environ`) can each hand over things that look unrelated to files.

**Used in:** filesystem note (Phase 3), Note 5 (file types and permissions), Note 9 (Phase 2: `/proc`), Note 4 Step 9 (the Docker socket).

---

# Phase 4 — The one sentence behind the whole series

Put Phases 2 and 3 together and almost everything on a Linux system fits in one sentence:

> **A process, running as some user, asks the kernel to do something to a file — and the kernel checks the rules.**

Each part of that sentence is a part of the series:

| Part of the sentence | Question | Notes |
|---|---|---|
| **A process** | How does a program start, what does it carry, how does it end? | 1, 2, 9, 10 |
| **running as some user** | Who is it? UID, groups | 3, 4 |
| **asks the kernel to do something to a file** | What is a file, a directory, a device? | Filesystem note, 5 |
| **the kernel checks the rules** | rwx, ownership, ACLs, special bits | 5, 6 |
| …and the **exceptions** to the rules | root, SUID, capabilities, sudo | 7, 8 |
| Who **starts** the processes, and keeps them running | systemd, services, boot | 11, 12 |
| What they **leave behind**, when they run, what they're allowed to use | logs, timers, resource limits | 13, 14 |

## 4.1 How the system gets running

A short overview — Note 11 covers it properly:

```
power on
  → firmware (UEFI/BIOS)         finds a bootloader
  → bootloader (GRUB)            loads the kernel and the initramfs
  → kernel                       starts drivers, mounts the root filesystem
  → systemd (PID 1)              the first user-space process; starts everything else
  → services                     sshd, cron, nginx, your apps
  → login                        you SSH in; sshd starts your shell (Note 1)
```

Everything after the kernel is a **process started by another process**. That's why `pstree` always has `systemd` at the top (Note 9, Phase 1.3).

**Used in:** the whole series. When something fails, ask which part of the sentence it is: wrong process? wrong user? wrong file? a rule? an exception?

---

# Phase 5 — Getting help on the system

The notes can't cover every option. The system documents itself, and these tools work offline, on any server.

## 5.1 `man` pages

```bash
man ls                   # the manual for ls
man -k password          # search page names and descriptions (same as: apropos password)
man -f passwd            # one-line summary of every page called passwd (same as: whatis passwd)
```

Inside `man`: `/word` searches, `n` jumps to the next match, `g`/`G` go to the start/end, `q` quits.

## 5.2 Sections

Pages are grouped into numbered **sections**. The same name can be in more than one:

| Section | Contains | Example |
|---|---|---|
| 1 | User commands | `man 1 passwd` — the command that changes your password |
| 2 | System calls (Phase 2.2) | `man 2 openat`, `man 2 kill` |
| 3 | C library functions | `man 3 getpwnam` |
| 4 | Device files | `man 4 null` |
| 5 | **File formats** | `man 5 passwd` — the `/etc/passwd` file; `man 5 sudoers`, `man 5 crontab` |
| 7 | Overviews and concepts | `man 7 signal`, `man 7 capabilities`, `man 7 hier` |
| 8 | Admin commands | `man 8 useradd`, `man 8 sshd` |

```bash
man -f passwd
```
```
passwd (1)           - change user password
passwd (1ssl)        - OpenSSL application commands
passwd (5)           - the password file
```

**Meaning:** `man passwd` opens the first match (the command). For the **file format**, ask for section 5: `man 5 passwd`. When a note writes `sudoers(5)` or `signal(7)`, the number is the section.

Section 7 pages are underrated. `man 7 signal` (Note 9), `man 7 capabilities` (Note 7), and `man 7 hier` (filesystem note) are each a good summary of a whole topic.

## 5.3 Reading a SYNOPSIS

```
cp [OPTION]... [-T] SOURCE DEST
```

| Notation | Meaning |
|---|---|
| `[ ... ]` | Optional |
| `...` | Can be repeated |
| `UPPERCASE` / underlined | Replace with your own value |
| `a\|b` | One or the other |

## 5.4 Other sources

| Tool | Use it for |
|---|---|
| `command --help` | Short list of options for most programs |
| `help cd` | Shell **built-ins** — they have no man page of their own (Note 1, Phase 3) |
| `type cmd` | Is it a built-in, alias, function, or program? (Note 1, Phase 3) |
| `man 5 systemd.service` | Every systemd unit setting (Note 11) |
| `/usr/share/doc/<package>/` | Package README files, changelogs, and example configs |

**Used in:** every note. When a note says "see `sudoers(5)`", this is how you get there.

---

# Phase 6 — How to read this series

## 6.1 The map

| Note | Topic | The question it answers |
|---|---|---|
| **0** | **Introduction & lab** | **What is Linux, and where do I practise?** |
| — | Filesystem structure | Where does everything live, and why? |
| 1 | Shell & Processes | How does a command run? |
| 2 | Environment Variables | What settings does a process carry? |
| 3 / 4 | Users, Groups & Identity | **Who** is a process? |
| 5 / 6 | Files, Directories & Permissions | **What** does a file allow? |
| 7 / 8 | Privilege & Delegation | How do you allow exceptions, narrowly? |
| 9 / 10 | Processes & Signals | What is a running program, and how do you control it? |
| 11 / 12 | systemd & Services | How are services started, ordered, restarted, and stopped? |
| 13 / 14 | Logs, Timers & Resource Limits | Records, schedules, and limits around a running service |

Read them in order. Each depends on the ones before it.

## 6.2 Theory and practice notes

From Note 3 on, topics come in pairs:

| | Theory note (odd number) | Practice note (even number) |
|---|---|---|
| Built from | **Phases** — concepts, in order | **Steps** — one real problem each |
| Each section ends with | **"Used in"** → which practice steps rely on it | **"Why: Note X, Phase Y"** → the theory behind it |
| Security angle | **"How this goes wrong"** boxes | Audit steps, e.g. Note 8 Step 6 |
| Ends with | Summary table, command reference | **State of the server**, scenario recap, troubleshooting table, command reference |

Notes 1 and 2 mix both: concept phases first, then a practical scenario.

A good rhythm: read a theory phase, then do the practice steps that use it, on the lab server.

## 6.3 How commands and output are shown

````
```bash
command --option          # a comment explaining it
```
```
the output you should see
```
````

- Commands and their output are in **separate blocks**. Output is trimmed with `...`.
- **"Meaning:"** after an output explains what to conclude from it.
- `sudo` is always written out. If a command has no `sudo`, run it as yourself.
- A comment like `← good` or `← the problem` points at the line that matters.

## 6.4 Practice notes build one server

All the practice notes work on the **same server**, in order. Note 4 starts where Note 2 left off, Note 6 starts where Note 4 left off, and so on. Each one ends with **"State of the server after Note N"** — that's the starting point of the next one. Doing them out of order means things described as "already there" won't be.

The lab note shows how to snapshot the server after each note, so you can go back.

---

# Phase 7 — The running scenario

Every practice note happens on this server. This is the reference cast; each note introduces its own part in detail.

## 7.1 The server

| | |
|---|---|
| Hostname | `app-server-01` (a second one, `app-server-02`, appears in Note 4 Step 12) |
| OS | Ubuntu 24.04 LTS |
| In the notes | A cloud VM (some steps mention GCP features: OS Login, Cloud Logging) |
| In your lab | A Multipass VM on your Mac (lab note) |
| Your account | `thirana`, UID 1000, the admin |

## 7.2 The apps

| | API | Worker |
|---|---|---|
| Language | Node.js | Python (in a virtualenv) |
| Code | `/opt/api` (later `releases/` + a `current` symlink, Note 6) | `/opt/worker` (`venv/` inside) |
| Runs as | `api` | `worker` |
| Listens on | Port 3000 → port 80 without root (Note 8) | Health check on `HEALTH_PORT` 8001 |
| Jobs | Serves HTTP | Queue jobs, nightly DB backup, reindexing, thumbnails |
| Config | `/etc/myapps/common.env` + `api.env` | `/etc/myapps/common.env` + `worker.env` |
| Database user | `api_rw` (read/write) | `backup_ro` (read-only) |

Later additions: `nginx` in front (Note 12), three workers from one template, `worker@1..3` (Note 12), the backup as a systemd timer (Note 14).

## 7.3 The people

| Account | UID | Role | Groups |
|---|---|---|---|
| `thirana` | 1000 | Platform lead, admin | `sudo`, `adm`, `ssh-users` |
| `nimal` | 1101 | Developer | `devs`, `ssh-users` |
| `sachini` | 1102 | Developer, later on-call | `devs`, `ops`, `ssh-users` |
| `ruwan` | 1103 | Ops on-call | `ops`, `adm`, `ssh-users` |
| `amal` | 1201 | Contractor until 2026-12-31 — offboarded in Note 4 | — (UID retired, never reused) |
| `kavindu` | 1202 | Contractor until 2027-06-30 | `devs`, `ssh-users` |
| `dilini` | 1301 | Auditor, read-only | `adm`, `ssh-users` |
| `deploy` | 1901 | CI/CD pipeline, key-only | `ssh-users` |
| `api` | 901 | Service account, no login | `appdata` |
| `worker` | 902 | Service account, no login | `appdata` |

## 7.4 The groups

| Group | GID | Gives access to |
|---|---|---|
| `devs` | 2001 | Project folder, API logs, editing two config files |
| `ops` | 2002 | Service restarts, all logs, backups |
| `appdata` | 2003 | Uploads shared between `api` and `worker` |
| `ssh-users` | 2004 | Logging in over SSH at all |
| `adm` | 4 | Reading system logs (Ubuntu built-in) |
| `sudo` | 27 | Full admin (Ubuntu built-in) |

## 7.5 How the server grows

| After note | What's been added |
|---|---|
| 1 | Shell setup for `thirana`, Node via nvm, `deploy.sh` |
| 2 | Env files for both apps, first systemd services, the backup cron job |
| 4 | Groups, accounts, SSH restricted to `ssh-users`, no root login |
| 6 | Folder layout, modes, ACLs, protected secrets, logs, backups |
| 8 | Port 80 via a capability, narrow sudo rules, systemd sandboxing |
| 10 | Fixed scripts and shutdown handling, `/proc` hidden from other users |
| 12 | Rewritten units, `myapps.target`, `worker@` templates, nginx override |
| 14 | Persistent journal, timers instead of cron, CPU/memory/task limits, log shipping |

**Used in:** every practice note. Come back here when a note says "nimal" or "`appdata`" and you've forgotten who or what that is.

---

# Phase 8 — Why this matters for security

## 8.1 Same system, two readers

An admin and an attacker read the **same** files, processes, and logs. The admin asks "is this set up right?"; the attacker asks "where was this set up too loosely?". You can't recognise a mistake until you know what the correct setup looks like. That's why these notes build the server **properly**, and point out the common mistakes along the way.

Almost every finding in post-exploitation is one of the mechanisms from this series, set too loosely:

| Mechanism (this series) | Set too loosely, it gives an attacker | Post-exploitation note |
|---|---|---|
| Startup files, `PATH` (Notes 1, 2) | Code that runs when someone logs in; command hijacking | `03-b`, `05-a` |
| Environment, process arguments (Notes 2, 9) | Passwords and tokens in `/proc/*/environ` and `ps` | `04-b` |
| Group membership (Notes 3, 4) | Root through `docker`, `lxd`, `disk` | `03-b` |
| File modes, ACLs (Notes 5, 6) | Readable secrets, writable configs and scripts | `02-a`, `04-a` |
| SUID, capabilities, sudo (Notes 7, 8) | Direct paths to root | `02-b`, `03-a`, `03-b` |
| Unit files, timers, cron (Notes 11–14) | Persistence that survives reboots | `03-a`, `05-a` |
| Logs and audit (Notes 13, 14) | Evidence that can be read, flooded, or deleted | `05-b` |
| Process inspection (Notes 9, 10) | Hiding in plain sight among normal processes | `05-c` |
| The kernel (this note, Phase 2) | Everything at once | `03-b` |

(Notes are in `../linux-post-exploitation/`. Start with `00-linux-permission.md` and `01-enumeration.md`.)

## 8.2 Enumeration is just "Phase 1.5" done thoroughly

The first minutes on an unknown machine are the questions from this note, in order:

```bash
cat /etc/os-release; uname -r     # what system, what kernel? (Phase 1.5, Phase 2.4)
id                                # who am I running as?      (Note 3)
sudo -l                           # what exceptions do I have? (Note 7)
ps -eo user,pid,cmd               # what else is running, as whom? (Note 9)
```

**Used in:** the "How this goes wrong" boxes in every theory note; `linux-post-exploitation/01-enumeration.md`.

---

# How this goes wrong — overview

For the security track: the big-picture mistakes this note sets up.

| Issue | Why it matters | How to check |
|---|---|---|
| Unpatched or unrebooted kernel | A kernel bug skips every permission check | `uname -r` vs `ls /boot`; `apt list --upgradable` |
| End-of-life release | No security fixes at all | `cat /etc/os-release`; compare with Ubuntu's release dates |
| Thinking "root" = "the kernel" | Root is still a process the kernel checks; capabilities and namespaces can limit it (Note 7) | `grep Cap /proc/self/status` |
| Thinking containers are VMs | Containers share the host kernel | `uname -r` inside and outside |
| Device and socket files left open | `/dev/sda`, `docker.sock` act like root | `ls -l /dev/sd* /var/run/docker.sock` |
| Lab habits copied to production | Passwordless sudo users, test keys, open ports (lab note) | Note 8, Step 6 audit |

---

# Summary table

| Concept | Key point |
|---|---|
| Linux | Strictly, the kernel only |
| Userland | Everything else: shell, tools, libraries, systemd, your apps |
| GNU/Linux | Linux kernel + GNU tools; Alpine and Android use the same kernel with different userlands |
| Distribution | Kernel + userland + package manager + release cycle + defaults |
| Debian vs Red Hat family | `apt` vs `dnf`; `sudo` vs `wheel`; `auth.log` vs `secure`; AppArmor vs SELinux |
| Ubuntu LTS | Every two years, 5 years of standard updates; this series uses 24.04 |
| `/etc/os-release`, `uname -r` | Which distro, which kernel |
| Kernel mode vs user mode | Enforced by the CPU; root runs in **user** mode |
| System call | The only way a program gets the kernel to act; watch with `strace` |
| "Permission denied" | The kernel refusing a syscall, based on who the process is |
| Containers | Processes with namespaces and cgroups on the host's kernel |
| Everything is a file | One interface for documents, devices, kernel info, pipes, sockets |
| `ls -l` first character | `-` `d` `l` `c` `b` `p` `s` |
| The one sentence | A process, running as some user, asks the kernel to do something to a file, and the kernel checks the rules |
| Boot | Firmware → GRUB → kernel → systemd (PID 1) → services → login |
| `man N page` | Section 1 commands, 2 syscalls, 5 file formats, 7 overviews, 8 admin |
| `man -k` / `man -f` | Search / one-line summary |
| Theory / practice notes | Phases + "Used in" / Steps + "Why:" + state of the server |

---

# Appendix — Command reference

## Which system is this?

```bash
cat /etc/os-release              # distro and version
uname -r                         # running kernel
uname -m                         # architecture (x86_64, aarch64)
hostnamectl                      # hostname, OS, kernel, virtualization
ls /boot                         # installed kernels
lsmod                            # loaded kernel modules
```

## Kernel and system calls

```bash
strace -e trace=openat,read,write cat /etc/hostname     # watch the syscalls of a command
strace -c ls > /dev/null                                # count syscalls by type
strace -f -p <PID>                                      # attach to a running process (needs permission)
```

## Everything is a file

```bash
ls -l /dev/null /dev/sda /dev/log        # file type is the first character
file /dev/null /etc/passwd /bin          # describe what something is
stat /etc/passwd                         # type, mode, owner, inode, times
cat /proc/loadavg                        # kernel info as a file
ls /sys/class/net                        # network interfaces
```

## Help

```bash
man ls                       # manual page
man 5 passwd                 # a specific section
man -k <word>                # search (apropos)
man -f <name>                # one-line summaries (whatis)
man 7 hier                   # overview of the directory layout
<command> --help             # quick option list
help <builtin>               # shell built-ins
type <name>                  # built-in, alias, function, or file?
ls /usr/share/doc/<package>  # package docs and example configs
```
