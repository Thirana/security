# L09 — ca-memory · The Secret That Was Never Written Down

**Act II · Credential Access · T1003**
**Status:** Solved
**Flag:** `bl_97f62d7f62cd47c8260c022ff8410389`

---

## Challenge brief

> This secret was never written to disk. It lives inside a running process. Reach into
> live memory (an environ, a mapped region, a debuggable service) and lift it out.

MITRE tag T1003 is "OS Credential Dumping" — accessing credentials from operating system
structures rather than files on disk. This level specifically covers the `/proc`
filesystem technique: reading a live process's environment variables, which exist only
in memory.

You land as root. There are no config files to read. The credential is held by a running
process and never touched the filesystem.

---

## The core concept: why some secrets only exist in memory

L07 and L08 both involved credentials stored on disk — in config files and in
`/etc/shadow`. Those credentials persist across reboots, survive process restarts, and
can be found by anyone with read access to the right file.

A different class of secrets never touches disk at all. They are injected into a running
process at startup — passed as environment variables by an orchestrator, a secrets
manager, or a container runtime — and exist only for as long as that process lives. When
the process exits, the secret is gone. If you want it, you have to read it from the live
process while it is running.

This is intentional design. The twelve-factor app methodology — a widely adopted pattern
for cloud-native applications — explicitly recommends passing credentials as environment
variables rather than config files, because:

- Environment variables are not stored in the code repository (no accidental git commits
  of secrets)
- They can be injected at runtime by a secrets manager without touching the filesystem
- They are isolated to the process that receives them

The security assumption is: only the process itself can read its own environment. That
assumption is correct for unprivileged users. But it breaks completely when the attacker
has root. Root can read the environment of any process on the box through the `/proc`
filesystem.

---

## The /proc filesystem — what it is and how it works

`/proc` is not a real filesystem stored on disk. It is a virtual filesystem that the
Linux kernel generates on the fly, exposing live kernel data structures as readable
files. Every time you `cat` something in `/proc`, the kernel is constructing the output
from its internal state at that moment — there is no underlying file being read.

Each running process gets its own directory in `/proc` named after its PID:

```
/proc/1/        ← process with PID 1
/proc/7/        ← process with PID 7
/proc/self/     ← always refers to the current process (whoever is reading)
```

Inside each PID directory, several files expose that process's live state:

| File | What it contains |
|---|---|
| `environ` | The environment variables the process was started with, null-byte separated |
| `cmdline` | The full command line including all arguments, null-byte separated |
| `maps` | The process's memory map — every region, its address range, permissions, and what it maps to |
| `mem` | The process's raw memory — readable with the right approach (requires ptrace permission) |
| `status` | Human-readable process status: PID, parent PID, memory usage, UID, GID |
| `fd/` | Directory of all open file descriptors — links to open files, sockets, pipes |
| `exe` | Symlink to the executable the process is running |

For credential harvesting, `environ` and `cmdline` are the primary targets. `maps` and
`mem` are used for deeper memory inspection.

### Why `tr '\0' '\n'` is always needed

The kernel stores environment variables and command-line arguments separated by null
bytes (`\0`) rather than newlines. This is because values themselves can contain
newlines, but cannot contain null bytes. When you `cat /proc/<pid>/environ` directly,
the null bytes display as invisible characters or garbage, and the output is unreadable.

```sh
# raw — unreadable
cat /proc/7/environ

# readable — convert null bytes to newlines
cat /proc/7/environ | tr '\0' '\n'
```

`tr '\0' '\n'` replaces every null byte with a newline, giving you one variable per
line.

### Permissions on /proc files

The permission model is straightforward:
- A process can always read its own `/proc/<pid>/` files
- Root can read any process's `/proc/<pid>/` files
- An unprivileged user can only read their own processes' files — not other users'
  processes

This is why this technique requires root (which earlier levels gave you). Without root,
you can only see your own process's environment — which you already know, because you
set it.

---

## What process environments actually contain

When a process starts, the OS copies the parent's environment and passes it in. The
values stay in that copy for the lifetime of the process. Three categories of things
appear in process environments:

**Standard system variables** — always present, always boring:
- `PATH` — directories to search for executables
- `HOME` — the user's home directory
- `PWD` — current working directory
- `SHLVL` — shell nesting depth
- `HOSTNAME` — the machine name
- `TERM`, `LANG`, `LC_ALL` — terminal and locale settings

**Injected runtime credentials** — the interesting ones:
- Database passwords: `DB_PASSWORD=...`, `DATABASE_URL=postgres://user:pass@host/db`
- API tokens: `API_TOKEN=...`, `STRIPE_SECRET_KEY=sk_live_...`
- Encryption keys: `ENCRYPTION_KEY=...`, `SECRET_KEY_BASE=...`
- Service credentials: `REDIS_PASSWORD=...`, `AGENT_TOKEN=...`
- Cloud credentials: `AWS_ACCESS_KEY_ID=...`, `AWS_SECRET_ACCESS_KEY=...`

**Agent and session variables** — useful for lateral movement:
- `SSH_AUTH_SOCK` — path to an SSH agent socket (covered in topic 4B)
- `DBUS_SESSION_BUS_ADDRESS` — inter-process communication socket

The filter when sweeping is simple: ignore the standard variables, look for anything
with a value that resembles a secret — a long hex string, a bearer token prefix, a
variable name containing `PASSWORD`, `SECRET`, `KEY`, `TOKEN`, `API`, or `AUTH`.

---

## The sweep approach — when you don't know which process holds the secret

In this level there were only two processes and we checked both individually. On a real
box with dozens of running services, you sweep all of them at once:

```sh
for pid in $(ps -eo pid --no-headers); do
    cat /proc/$pid/environ 2>/dev/null | tr '\0' '\n' | \
    grep -iE 'pass|secret|key|token|api|credential|auth' | \
    sed "s/^/[PID $pid] /"
done
```

What each part does:
- `ps -eo pid --no-headers` — list all PIDs, no header line
- `cat /proc/$pid/environ 2>/dev/null` — read each process's environment, silencing
  permission errors (which appear for kernel threads and some system processes even as
  root)
- `tr '\0' '\n'` — make the output readable
- `grep -iE 'pass|secret|key|token|api|credential|auth'` — filter to only interesting
  lines
- `sed "s/^/[PID $pid] /"` — prefix each hit with the PID so you know where it came
  from

This produces a clean list of credential-like values across all running processes,
labelled by PID. Then for any interesting PID, read its full environment:

```sh
cat /proc/<pid>/environ | tr '\0' '\n'
```

---

## The cmdline technique — credentials passed as arguments

Some tools accept credentials as command-line arguments rather than environment
variables — the MySQL `-p` flag, Redis `--requirepass`, some deployment scripts. These
appear in `/proc/<pid>/cmdline`:

```sh
cat /proc/<pid>/cmdline | tr '\0' '\n'
```

This prints each argument on its own line. A process started as:

```sh
mysql -u webapp -pS3cr3t-DB-2024 -h db.internal mydb
```

Would show:

```
mysql
-u
webapp
-pS3cr3t-DB-2024
-h
db.internal
mydb
```

The password is `-pS3cr3t-DB-2024` — the `-p` flag and password written together with
no space, which is the MySQL CLI convention. The important difference from shell history:
`/proc/cmdline` shows commands that are *currently running*, not commands that were run
in the past. A long-running background process may have been started months ago, its
history entry long rotated out, and the credential is still readable here.

Sweep all cmdlines for inline credentials:

```sh
for pid in $(ps -eo pid --no-headers); do
    cmdline=$(cat /proc/$pid/cmdline 2>/dev/null | tr '\0' ' ')
    echo "$cmdline" | grep -iE '\-p[^ ]+|--pass|--secret|--token|--key' && \
    echo "  ^ PID $pid"
done
```

---

## Deeper: reading heap memory with /proc/maps and /proc/mem

`/proc/<pid>/environ` captures the startup environment. But a process may fetch
additional credentials *after* startup — from a secrets manager, over an API, from a
database — and those live in the process's heap memory. They never appear in `environ`
because they were not part of the startup environment.

To read heap memory, you first need to know where the heap is:

```sh
cat /proc/<pid>/maps
```

This prints the process's memory map. Each line is one region:

```
address-range         perms  offset  dev   inode  pathname
7f8b4c000000-7f8b4c200000 rw-p 00000000 00:00 0     [heap]
7ffd4a000000-7ffd4a200000 rw-p 00000000 00:00 0     [stack]
```

The `[heap]` region is where dynamic memory lives — strings, buffers, connection pools.
`rw-p` means readable and writable. The two hex values are the start and end addresses.

To read that memory, you need a debugger. On boxes where `gdb` is available:

```sh
gdb -p <pid>
```

Inside gdb:

```
(gdb) dump memory /tmp/heap.bin 0x7f8b4c000000 0x7f8b4c200000
(gdb) detach
(gdb) quit
```

Then search the dump for readable strings:

```sh
strings /tmp/heap.bin | grep -iE 'pass|secret|key|token|bearer|api'
rm /tmp/heap.bin
```

**When to use this vs environ.** `environ` is fast, targeted, and low-risk. Heap
dumping is slower, produces large noisy output, and — importantly — pauses the target
process while gdb is attached. On a real box, pausing a live service causes timeouts
and may trigger alerts. Use heap dumping only when:

- The process fetches credentials after startup (secrets manager, API calls)
- You suspect a credential exists in memory but it is not showing up in `environ`
- You need session tokens or decrypted keys that were only ever in memory

On Alpine Linux specifically, `gdb` may not be installed (`busybox` does not include
it). The challenge description mentioned it as a technique — but the actual secret here
was in `environ`, so `gdb` was not needed.

---

## Why this technique matters in real environments

Modern infrastructure passes secrets to containers and processes through environment
variables by design — Docker's `-e` flag, Kubernetes secrets mounted as env vars,
AWS ECS task definitions, GCP Cloud Run service configuration. The assumption is that
the container or process boundary protects the values.

That assumption holds when the attacker does not have root on the host. It breaks when
they do. A container runtime with root access to the host can read any container's
`/proc/<pid>/environ` from outside the container, without ever entering it. The
"isolation" the container provides does not protect environment variables from a
compromised host.

This is one reason why "root on a container host" is a much more serious finding than
"root inside a container" — the host has a `/proc` view of every container running
on it.

---

## Enumeration log

### Step 1 — List running processes

```sh
ps aux
```

Result: two `sleep infinity` processes (PID 1 and PID 7), our shell (PID 36), and `ps`
itself. Standard `sleep` processes are often used as placeholders in CTF containers to
carry environment variables — they run indefinitely, hold whatever env was injected, and
do nothing else.

**Conclusion:** Check both sleep processes' environments.

---

### Step 2 — Read both process environments

```sh
cat /proc/1/environ | tr '\0' '\n'
cat /proc/7/environ | tr '\0' '\n'
```

PID 1 — only standard system variables: `HOSTNAME`, `PWD`, `HOME`, `SHLVL`, `PATH`.
Nothing interesting.

PID 7:

```
HOSTNAME=6a4823e41d73
AGENT_TOKEN=e97f7a35721160090ca7844cd13e7843
PWD=/
HOME=/root
SHLVL=0
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

`AGENT_TOKEN=e97f7a35721160090ca7844cd13e7843` — a credential that was injected into
this process at startup and exists nowhere on disk.

---

### Step 3 — Write proof and verify

```sh
verify
```

Grader told us the proof file: `/tmp/.ca-memory-proof`.

```sh
printf 'e97f7a35721160090ca7844cd13e7843' > /tmp/.ca-memory-proof && verify
```

**Level solved.**

---

## What the vulnerability was

The `AGENT_TOKEN` was injected as an environment variable into PID 7 at container
startup. This is standard practice for containerized applications. The vulnerability is
not in the injection mechanism — it is in having root on the host. With root, `/proc`
is an open window into every process's memory.

The challenge title captures it: "the secret that was never written down." It was never
in a file. But it was in `/proc`.

---

## Dead ends

None this level. The path was direct: list processes → read environ → find credential
→ verify. The level was designed to teach the technique clearly rather than hide the
target.

The challenge description mentioned three paths — environ, mapped regions, and a
debuggable service — but all three ultimately reach the same place (process memory). On
this box, `environ` was sufficient. On a harder variant, the secret might only exist in
the heap and would require the `maps` + `gdb` approach.

---

## Carryover findings

None. The token was self-contained.

---

## Command reference

| Command | What it does |
|---|---|
| `ps aux` | List all running processes with PID, user, and command |
| `cat /proc/<pid>/environ \| tr '\0' '\n'` | Read a process's environment variables, one per line |
| `cat /proc/<pid>/cmdline \| tr '\0' '\n'` | Read a process's command line arguments, one per line |
| `for pid in $(ps -eo pid --no-headers); do cat /proc/$pid/environ 2>/dev/null \| tr '\0' '\n' \| grep -iE 'pass\|secret\|key\|token\|api' \| sed "s/^/[PID $pid] /"; done` | Sweep all process environments for credential keywords |
| `cat /proc/<pid>/maps` | Show a process's memory map — find heap/stack region addresses |
| `gdb -p <pid>` | Attach debugger to a live process for memory inspection |
| `dump memory /tmp/heap.bin 0xSTART 0xEND` | (inside gdb) Dump a memory region to a file |
| `strings /tmp/heap.bin \| grep -iE 'pass\|secret\|key\|token'` | Search a memory dump for readable credential strings |
| `printf 'VALUE' > /tmp/.ca-memory-proof && verify` | Write proof and verify |