# L17 — lm-pivot · The Host You Cannot See

**Act IV · Lateral Movement · T1090**
**Status:** Solved
**Flag:** `bl_86f27e4018015e9dd2004ea8a95e4011`

---

## Challenge brief

> The host you want is not reachable from where you are. The host you own is. Turn the
> box you hold into the route to the box you cannot see.

MITRE tag T1090 is "Proxy" — using a compromised host as a network relay to reach
systems that are not directly accessible from the attacker's position. This is the
opening challenge of Act IV: Lateral Movement.

You land as root on a box that can reach an internal host. The internal host has a
service listening that sends back a proof token when you connect to it. Your job is to
find that host, find that service, connect to it, and write the token to the proof file.

---

## Background: what lateral movement is and why it needs a pivot

When you compromise the first machine in a network — the "foothold" — you typically
land somewhere with limited reach. Firewalls, network segmentation, and routing rules
mean that some hosts are only reachable from inside the network, not from the internet.

The foothold box is inside that network. It can reach hosts that you, sitting at your
attacker machine, cannot. So instead of trying to connect directly to the internal host
(which is impossible from outside), you route your connection through the foothold box.
The foothold becomes a **pivot point** — your traffic goes: attacker → foothold → internal
host.

This is what the challenge description means: "The host you want is not reachable from
where you are. The host you own is." You cannot reach boxB from outside the network. But
the foothold can, and you have root on the foothold.

---

## Docker networking — why the environment looks the way it does

Breachlab's containers run inside Docker. Understanding how Docker handles networking
explains everything we see when we enumerate the box: the IP address pattern, the `.1`
gateway, the DNS names, and why some hosts are reachable while others aren't.

### What Docker bridge networking is

By default, Docker connects containers to a **bridge network** — a virtual network that
exists only on the host machine running Docker. Think of it as a private LAN created in
software. Each container connected to the same bridge can talk to the others directly.
Traffic from outside the host cannot reach the containers unless it's explicitly forwarded
through a port mapping (`-p 8080:80`).

This is exactly the network segmentation that makes lateral movement interesting: the
containers can talk to each other, but you (connecting from outside) can only reach the
foothold container through the port the CTF platform exposed.

### How Docker assigns IP addresses

When Docker creates a bridge network, it assigns a subnet from a private range
(typically `172.17.0.0/16`, `10.0.0.0/8`, or `192.168.0.0/16`). In this level the
subnet was `10.66.118.0/24`.

Within that subnet, Docker follows a consistent pattern:
- **`.1`** — always the bridge gateway address. This is the Docker host's virtual
  interface on the network. It routes traffic between the Docker bridge and the outside
  world. It never runs services that are useful to an attacker.
- **`.2`, `.3`, ...** — containers, assigned in order of creation. The foothold was `.2`
  and boxB was `.3`, meaning the foothold was started first.

This pattern is reliable across all Breachlab levels: when you see `.1`, it's the
gateway — ignore it. Any higher number is a container.

### Docker Compose and service names

Breachlab builds each level as a Docker Compose setup — multiple containers defined in a
`docker-compose.yml` file, each with a service name (`sandbox`, `boxB`, etc.). Docker
Compose creates a custom bridge network named after the project, and assigns each service
a DNS name on that network.

The DNS name format is:

```
<service-name>.<project-name>-net
```

In practice, with the random project IDs Breachlab uses:

```
pc-zVy6N7xZWnwdlJJGkMcsgBRx-boxB.pc-zVy6N7xZWnwdlJJGkMcsgBRx-net
```

Breaking this down:
- `pc-zVy6N7xZWnwdlJJGkMcsgBRx` — the randomly generated project ID for this session
- `boxB` — the Docker Compose service name
- `-net` — the network suffix

When we ran `nslookup 10.66.118.3`, the reverse DNS response confirmed the target was
the `boxB` service. This is useful when you don't know what a host is — the DNS name
often tells you its role.

### Docker's internal DNS resolver

```sh
cat /etc/resolv.conf
```

On every container in a Docker Compose network, the nameserver is `127.0.0.11` —
Docker's embedded DNS resolver. This resolver:

- Answers queries for container service names (`boxB` → `10.66.118.3`)
- Answers reverse queries (IP → Docker DNS name)
- Forwards all other queries to the host's upstream DNS

The `127.0.0.11` address is a Docker implementation detail — it's a special loopback
address that only resolves inside the container's network namespace. You'll see it on
every Breachlab box.

### Why containers can't reach each other on port 22 by default

SSH inside a container works differently from SSH on a real machine. For SSH to be
reachable on port 22 inside a container, the container must:
1. Have an SSH daemon (`sshd`) installed and running
2. Have the sshd listening on port 22 (or another port)

Most containers don't run sshd at all — they're designed to run one specific service,
not to be SSH-accessible. This is why `ssh root@10.66.118.3` on port 22 returned
"Connection refused" immediately — there was no sshd running in boxB.

### What isolated means in practice for a CTF attacker

When you land on a Breachlab container:
- You can reach other containers on the same Docker Compose network by IP or DNS name
- You cannot reach containers on different Docker Compose networks (different levels)
- The outside world (internet, your machine) cannot reach containers directly — only
  through what the CTF platform exposed (your SSH port into the foothold)
- The `.1` gateway routes your container's outbound traffic to the internet if needed
  (for DNS lookups, package downloads), but doesn't provide services you can exploit

This is the reason lateral movement requires a pivot: the target exists on the same
internal Docker network as the foothold, but that network is invisible from outside.

---

## Phase 1: network situational awareness

The first thing to do when landing on any new box is map your network position. This
answers three questions: where am I, what subnet am I on, and what other hosts might be
reachable from here.

### Step 1 — Who am I and what IP do I have?

```sh
id
ip addr
```

Result:
```
uid=0(root) ...
inet 10.66.118.2/24 ... eth0
```

**What this tells us:** We are root (good — no escalation needed). We have one network
interface (`eth0`) with IP `10.66.118.2` on the `10.66.118.0/24` subnet. The `/24`
means 256 addresses total — from `10.66.118.0` to `10.66.118.255`. Usable host addresses
are `.1` through `.254`.

**Why we check this first:** The subnet tells us the scan range. Without knowing our IP
and subnet, we don't know which addresses to look for other hosts on. Getting this wrong
means scanning the wrong range entirely.

**Reading CIDR notation:** `10.66.118.2/24` means the first 24 bits are the network
(`10.66.118`) and the last 8 bits are the host (`.2`). A `/24` has 256 total addresses
(2^8), of which 254 are usable hosts.

### Step 2 — Are there additional networks?

```sh
ip route
```

Result:
```
10.66.118.0/24 dev eth0 scope link  src 10.66.118.2
```

**What this tells us:** Only one route — the local subnet. There is no default gateway
and no routes to other subnets. All reachable hosts are on `10.66.118.0/24`.

**Why we check this:** In multi-network Docker environments, containers can be attached
to multiple networks simultaneously. `ip route` would then show multiple subnet entries
or a default gateway routing to a second network. For example:
```
10.66.118.0/24 dev eth0  (this network)
10.66.119.0/24 via 10.66.118.1 dev eth0  (second network via gateway)
```
Here there's only one, so we scan only this subnet.

**What to expect:** In CTF lateral movement challenges you typically see either one
subnet (all hosts on the same bridge network, as here) or two (foothold on one network,
target on another reachable via a gateway). Two subnets usually means you need to use
the foothold as an actual router with SSH tunneling.

### Step 3 — Are there pre-planted hostnames?

```sh
cat /etc/hosts
```

Result: only the box's own hostname (`10.66.118.2`). No other hosts listed.

**What this tells us:** No shortcuts — the challenge hasn't given us the target's
hostname directly. We need to discover it.

**What to expect:** In some CTF levels, `/etc/hosts` is pre-populated with internal
hostnames as a hint. When it is, those entries are your first targets. When it isn't,
discover hosts through scanning.

---

## Phase 2: host discovery

Now that we know the subnet is `10.66.118.0/24`, we need to find which hosts are alive.
The standard tool is a ping sweep — send one ICMP echo to each address and see who
replies.

```sh
for i in $(seq 1 254); do
  ping -c 1 -W 1 10.66.118.$i 2>/dev/null | grep -q "1 packets received" && echo "10.66.118.$i up"
done
```

**What each part does:**
- `seq 1 254` — generates numbers 1 through 254 (all usable host addresses)
- `ping -c 1` — send exactly one packet, then stop
- `-W 1` — wait at most 1 second for a reply
- `grep -q "1 packets received"` — silently check if the reply indicates success
- `&& echo "$ip up"` — only print the IP if ping succeeded

Result:
```
10.66.118.1 up
10.66.118.2 up
10.66.118.3 up
```

**What this tells us:** Three hosts alive:
- `.1` — the Docker bridge gateway (explained above — always present, never a target)
- `.2` — us (the foothold)
- `.3` — the other container (boxB), our target

**Also useful — DNS reverse lookup:**

```sh
nslookup 10.66.118.3
```

Result:
```
name = pc-zVy6N7xZWnwdlJJGkMcsgBRx-boxB.pc-zVy6N7xZWnwdlJJGkMcsgBRx-net
```

The DNS name tells us this is the `boxB` Docker Compose service — confirming our target
and its role. The Docker embedded DNS at `127.0.0.11` resolved the reverse lookup.

---

## Phase 3: port scanning boxB

Now that we have boxB's IP (`10.66.118.3`), we need to find what services it's running.
This is where things went wrong initially, and the lesson is important.

### The failed approach: /dev/tcp shell trick

The standard busybox/ash way to check a port without nmap is:

```sh
timeout 1 sh -c 'echo >/dev/tcp/10.66.118.3/22' 2>/dev/null && echo "open"
```

This opens `/dev/tcp/<host>/<port>` as a file, which triggers a TCP connection attempt.
If the connection succeeds, the redirect succeeds. If it fails, the redirect fails.

We ran this for ports 1–2000 and got nothing. The command completed in about 1 second —
suspiciously fast. For 2000 ports with a 1-second timeout each, it should take much
longer if ports were actually timing out. The fast completion meant every port returned
"connection refused" immediately.

**Why this failed:** `/dev/tcp` is a **bash built-in** — it's implemented directly in
bash's code, not as a real file. When bash sees `/dev/tcp/host/port`, it intercepts the
file open and performs a TCP connection instead. Busybox ash is not bash. It has a
partial implementation with known gaps — particularly around certain port ranges and
connection states. On this box it silently failed on port 9000 (the actual open port),
reporting it as closed.

**The lesson:** `/dev/tcp` is a useful quick check for a few specific well-known ports.
Never rely on it for comprehensive scanning. Use nmap when it's available.

### The correct approach: nmap

The full filesystem listing showed nmap was installed at `/usr/bin/nmap`. This was the
key finding — checking what tools are available before choosing a scanning approach
should come before spending time on shell primitives.

```sh
nmap -p- --min-rate 1000 10.66.118.3
```

**What each flag does:**
- `-p-` — scan all 65535 TCP ports. Without this, nmap only scans its top 1000 most
  common ports. Using `-p-` ensures nothing is missed regardless of which port the
  service runs on.
- `--min-rate 1000` — send at least 1000 packets per second. Safe to use on a local
  Docker network where latency is near zero and there is no IDS to evade.

Result:
```
PORT     STATE SERVICE
9000/tcp open  cslistener
```

One port open — 9000. `cslistener` is nmap's generic label for port 9000 (it doesn't
know the actual protocol from the port number alone).

### Identifying and using the service on port 9000

We know port 9000 is open but not what protocol it speaks. The identification approach
goes from most common to most generic:

**Try HTTP first** — the most common protocol on non-standard ports:
```sh
curl -s http://10.66.118.3:9000/
```
No output. Not HTTP.

**Try SSH** — also common on non-standard ports (2222, 8022, 9022, etc.):
```sh
ssh -o StrictHostKeyChecking=no -p 9000 root@10.66.118.3
```
"Connection closed" immediately. Real SSH prints a host key fingerprint warning before
closing — this closed without that, so it's not SSH.

**Try raw TCP with nc** — when no known protocol matches, connect raw and read whatever
the service sends:
```sh
nc 10.66.118.3 9000
```
Result:
```
74033642e0747328eaad024710ed5f1e
```

The service sent a 32-character hex token and closed. This is the classic CTF
proof-delivery pattern: a minimal TCP service that accepts any connection, sends the
token, and disconnects immediately.

---

## Enumeration log

### Step 1 — Network situational awareness

```sh
id && ip addr && ip route && cat /etc/hosts
```

Result: root on `10.66.118.2/24`, single subnet, no other routes, no pre-planted hosts.

**Decision:** Scan the full `/24` subnet for live hosts.

---

### Step 2 — Host discovery via ping sweep

```sh
for i in $(seq 1 254); do ping -c 1 -W 1 10.66.118.$i 2>/dev/null | grep -q "1 packets received" && echo "10.66.118.$i up"; done
```

Result: `.1` (Docker gateway — not a target), `.2` (us), `.3` (boxB — the target).

**Decision:** Target `10.66.118.3`. Find its open ports.

---

### Step 3 — Port scan with /dev/tcp (failed)

Swept ports 1–2000 using the `/dev/tcp` shell trick. All returned "closed" in ~1 second
— too fast, and the approach silently failed on port 9000. Extended the sweep to
2001–65535 and cancelled after 4 minutes with nothing found.

**Decision:** `/dev/tcp` is unreliable here. Check what tools are available.

---

### Step 4 — Inventory tools

```sh
find / -not -path '/proc/*' -not -path '/sys/*' -type f 2>/dev/null
```

Found `/usr/bin/nmap` in the listing.

**Decision:** Use nmap — it is reliable, comprehensive, and fast on a local network.

---

### Step 5 — Full port scan with nmap

```sh
nmap -p- --min-rate 1000 10.66.118.3
```

Result: port 9000 open.

**Decision:** Connect to port 9000 to discover the service.

---

### Step 6 — Identify and use the service

```sh
curl -s http://10.66.118.3:9000/      # no response — not HTTP
ssh -p 9000 root@10.66.118.3          # closed immediately — not SSH
nc 10.66.118.3 9000                   # returns token: 74033642e0747328eaad024710ed5f1e
```

**Decision:** Raw TCP service returning a hex token. This is the proof value.

---

### Step 7 — Write proof and verify

```sh
printf '74033642e0747328eaad024710ed5f1e' > /tmp/.lm-pivot-proof && verify
```

**Level solved.**

---

## What the challenge demonstrates

This level teaches the first step of lateral movement in a Docker environment: using the
network position of the foothold container to reach another container that is invisible
from outside the Docker network.

In a real network the equivalent would be: an attacker SSHes into an internet-facing
web server (the foothold), then uses that server's internal network access to reach a
database, an internal API, or a development host that has no external firewall rules
allowing inbound connections.

The pivot in this level was simple — the service on boxB was directly reachable from the
foothold without any tunneling needed, because they were on the same Docker bridge
network. In later levels (L18, L19), the pivot involves SSH tunneling, credential reuse,
and multi-hop chains where each host is a stepping stone to the next.

---

## Dead ends

### /dev/tcp port sweep (1–9000)

Swept ports 1–2000 and partially 2001–65535. All returned "closed." The busybox ash
`/dev/tcp` implementation silently failed on port 9000. Spent significant time here
before switching to nmap.

**Lesson:** Check for nmap first. `/dev/tcp` is not a reliable port scanner.

### Gateway (.1) enumeration

Checked the Docker gateway for services. Nothing listening beyond Docker's internal
routing. The gateway is infrastructure, not a target.

### HTTP and SSH on port 9000

Both are reasonable first guesses for an unknown port. HTTP gave no response, SSH
closed immediately. `nc` for raw TCP inspection was the correct next step.

---

## Carryover findings

None. The level is self-contained.

---

## Command reference

| Command | What it does |
|---|---|
| `ip addr` | Show all network interfaces and IPs — read the subnet from the CIDR notation |
| `ip route` | Show routing table — reveals additional subnets or confirms only one network |
| `cat /etc/hosts` | Check for pre-planted internal hostnames |
| `cat /etc/resolv.conf` | Shows `127.0.0.11` on Docker containers — Docker's embedded DNS |
| `nslookup <ip>` | Reverse DNS lookup — Docker DNS returns service names like `boxB` |
| `for i in $(seq 1 254); do ping -c 1 -W 1 10.x.x.$i 2>/dev/null \| grep -q "1 packets received" && echo "$i up"; done` | Ping sweep to find live hosts on a /24 subnet |
| `find /usr/bin /usr/local/bin -type f 2>/dev/null` | Inventory available tools — check for nmap before resorting to /dev/tcp |
| `nmap -p- --min-rate 1000 <ip>` | Full TCP port scan of all 65535 ports — reliable and fast on LAN |
| `nmap -p <port> <ip>` | Scan a specific port |
| `curl -s http://<ip>:<port>/` | Try HTTP on an unknown port — most common protocol first |
| `ssh -o StrictHostKeyChecking=no -p <port> root@<ip>` | Try SSH on a non-standard port |
| `nc <ip> <port>` | Raw TCP connection — shows whatever the service sends on connect |
| `printf 'VALUE' > /tmp/.lm-pivot-proof && verify` | Write proof and verify |