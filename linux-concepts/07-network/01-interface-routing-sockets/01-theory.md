# Interfaces, Routing & Sockets — Theory (Note 15)

How a Linux server connects to a network: the interfaces that carry packets, the addresses on them, the routing table that decides where packets go, and the sockets and ports that connect the network to your programs.

**Series:**
- Notes 1–2: Shell, Environment
- Notes 3–8: Users, Files & Permissions, Privilege
- Notes 9–14: Processes, systemd, Logs/Timers/Limits
- **Note 15: Interfaces, Routing & Sockets — Theory** ← this note
- Note 16: Interfaces, Routing & Sockets — Practice
- Note 17 / 18: DNS & Host Firewall — Theory / Practice
- Note 19 / 20: SSH & Network Debugging — Theory / Practice

This is the **inside-the-server** side of networking. The cloud side (VPCs, cloud firewall rules, load balancers) decides whether a packet reaches the VM. This note covers what the VM itself does with it. Examples use a GCP Compute Engine VM, because some things look different there than in textbooks.

Each phase ends with **"Used in"** (Note 16 steps) and, where relevant, **"How this goes wrong"** for the security track.

---

## Contents

1. Phase 1 — The big picture: following one request
2. Phase 2 — Interfaces: where packets enter and leave
3. Phase 3 — IP addresses on an interface
4. Phase 4 — How the address gets there: DHCP, netplan, networkd
5. Phase 5 — Neighbours: from IP address to MAC address
6. Phase 6 — Routing: deciding where a packet goes
7. Phase 7 — Forwarding, NAT, and network namespaces
8. Phase 8 — Ports and the two transport protocols
9. Phase 9 — Sockets: how programs use the network
10. Phase 10 — The life of a TCP connection
11. Phase 11 — What the connection errors really mean
12. Phase 12 — Reading `ss` like a pro
13. Phase 13 — Reachability tools: ping, traceroute, MTU
14. Phase 14 — Kernel network settings (sysctl)
15. How this goes wrong — overview
16. Summary table
17. Appendix — Command reference

---

# Phase 1 — The big picture: following one request

## 1.1 One request, start to finish

A user opens the site. Their browser connects to the API on our server (`app-server-01`) on port 80. Here's everything that happens, in order:

```
user's laptop
   │  "connect to 192.0.2.10 port 80"
   ▼
internet
   ▼
Google's edge ─ external IP 192.0.2.10 is translated to the VM's internal IP 10.0.0.4
   ▼
VPC firewall rules (cloud side)                    ← allowed?
   ▼
VM's network interface ens4 (10.0.0.4)             ← Phase 2, 3
   ▼
host firewall (nftables / ufw)                     ← allowed?  (Note 17)
   ▼
kernel: is any socket LISTENing on port 80?         ← Phase 9
   ▼
node (api service, PID 2890) accepts the connection ← Phase 9, 10
   │
   │  reply
   ▼
kernel: routing table — which way out?              ← Phase 6
   ▼
ens4 → gateway 10.0.0.1 → Google edge → internet → laptop
```

## 1.2 The layers, mapped to Linux

Networking is usually taught as **layers**. Each layer has a Linux piece and a command to look at it:

| Layer | Question it answers | Linux piece | Main command |
|---|---|---|---|
| **Link** | How do bytes reach the next machine on the same network? | Interface, MAC address | `ip link`, `ip neigh` |
| **Network (IP)** | How does a packet find the right machine anywhere? | IP address, routing table | `ip addr`, `ip route` |
| **Transport (TCP/UDP)** | Which program on that machine, and is delivery reliable? | Port, socket | `ss` |
| **Application** | What do the bytes mean? | HTTP, PostgreSQL, SSH | `curl`, `psql`, `ssh` |

## 1.3 The five questions

When something "can't connect", the cause is almost always one of these. They're the structure of this note and the order of debugging:

| # | Question | Covered in |
|---|---|---|
| 1 | Does the server have a working **interface and address**? | Phases 2–5 |
| 2 | Is there a **route** to the other side — and back? | Phases 6–7 |
| 3 | Is a program **listening** (or connecting) on the right **address and port**? | Phases 8–12 |
| 4 | Is a **firewall** dropping or rejecting it (cloud or host)? | Note 17 |
| 5 | Does the **name** resolve to the right IP? | Note 17 |

Keep this table in mind. Every practice step in Note 16 answers one of these questions.

---

# Phase 2 — Interfaces: where packets enter and leave

## 2.1 What an interface is

An **interface** is a point where the kernel sends and receives packets. Some are real (or virtual) network cards; some exist only in software.

| Interface | What it is |
|---|---|
| `lo` | **Loopback** — a fake network inside the machine. Traffic to `127.0.0.1` never leaves the server |
| `ens4` (GCP), `eth0`, `enp0s3` | The VM's network card. On GCP it's a virtual card (gVNIC or virtio) |
| `docker0` | A software **bridge** Docker creates — a virtual switch for containers |
| `veth...` | One end of a **virtual cable**; the other end is inside a container (Phase 7) |
| `wg0`, `tun0` | VPN tunnels |

Names like `ens4` are "predictable names" based on the hardware slot, so they don't change between reboots.

## 2.2 Looking at interfaces

```bash
ip -br link        # brief: one line per interface
```
```
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
ens4             UP             42:01:0a:00:00:04 <BROADCAST,MULTICAST,UP,LOWER_UP>
docker0          DOWN           02:42:8f:1c:2a:b1 <NO-CARRIER,BROADCAST,MULTICAST,UP>
```

```bash
ip link show ens4  # full detail
```
```
2: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 42:01:0a:00:00:04 brd ff:ff:ff:ff:ff:ff
```

| Part | Meaning |
|---|---|
| `UP` | The interface is **enabled** by the admin/system |
| `LOWER_UP` | The "cable" is **connected** (there's a carrier). `UP` without `LOWER_UP` = enabled but nothing plugged in |
| `NO-CARRIER` | Enabled, but no link (e.g. `docker0` with no containers) |
| `mtu 1460` | **Maximum packet size** this interface sends (Phase 13) |
| `link/ether 42:01:...` | The **MAC address** — the interface's hardware address on the local network |
| `state UP` | Overall state |

GCP detail: VM MAC addresses start with `42:01`, and the rest is the internal IP in hex: `0a:00:00:04` = `10.0.0.4`.

## 2.3 Counters: is the interface dropping things?

```bash
ip -s link show ens4
```
```
    RX:  bytes  packets errors dropped  missed   mcast
    9182736251  8123311      0       0       0       0
    TX:  bytes  packets errors dropped carrier collsns
    4123998120  6012881      0       0       0       0
```

Rising `errors` or `dropped` points to a problem **below** IP: a bad link, a full queue, or a driver issue. On a healthy cloud VM these stay at or near 0.

**Used in:** Note 16, Steps 0, 7, 9.

---

# Phase 3 — IP addresses on an interface

## 3.1 What's on the VM

```bash
ip -br addr
```
```
lo               UNKNOWN        127.0.0.1/8 ::1/128
ens4             UP             10.0.0.4/32 fe80::4001:aff:fe00:4/64
docker0          DOWN           172.17.0.1/16
```

```bash
ip addr show ens4
```
```
2: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 ...
    link/ether 42:01:0a:00:00:04 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.4/32 metric 100 scope global dynamic ens4
       valid_lft 3240sec preferred_lft 3240sec
    inet6 fe80::4001:aff:fe00:4/64 scope link
       valid_lft forever preferred_lft forever
```

| Part | Meaning |
|---|---|
| `inet 10.0.0.4/32` | IPv4 address and **prefix length** |
| `scope global` | Usable to talk to other machines |
| `scope link` | Only valid on this local link (IPv6 `fe80::` addresses) |
| `scope host` | Only inside this machine (`127.0.0.1`) |
| `dynamic` + `valid_lft 3240sec` | Given by **DHCP**, lease renews before it runs out (Phase 4) |

## 3.2 Prefix length in one paragraph

`10.0.0.4/24` means: the first **24 bits** (`10.0.0`) are the **network**, the last 8 bits identify the **host**. So `/24` covers `10.0.0.0`–`10.0.0.255`. A **bigger** number means a **smaller** network: `/32` is exactly **one** address; `/16` is 65,536 addresses; `/0` is **everything**.

## 3.3 The GCP surprise: `/32`

On most networks, a server in subnet `10.0.0.0/24` would have `10.0.0.4/24`, and could talk directly to every other address in that range.

On **GCP**, the VM gets `10.0.0.4/32` — just itself. So **every** packet, even to a VM in the same subnet, goes to the **gateway** `10.0.0.1` first, and Google's virtual network delivers it. This is how GCP applies firewall rules and routes centrally. You'll see the effect in the routing table (Phase 6).

## 3.4 The external IP isn't on the VM

`ip addr` never shows the VM's **external** (public) IP. GCP translates it at its edge: traffic to `192.0.2.10` arrives at the VM addressed to `10.0.0.4`. So:

- Your app should **listen on the internal address** (or all addresses, Phase 9) — never try to bind to the external IP; it doesn't exist on the VM.
- To find the external IP from inside the VM, ask the **metadata server** (Note 16, Step 0).

## 3.5 Special address ranges

| Range | Meaning |
|---|---|
| `127.0.0.0/8` (usually `127.0.0.1`) | Loopback — this machine only |
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | **Private** ranges — not routed on the internet |
| `169.254.0.0/16` | **Link-local**. On GCP, `169.254.169.254` is the **metadata server** (Phase 15 security) |
| `0.0.0.0` | "Any address" when listening (Phase 9); "no route / default" in routing |
| `::1` | IPv6 loopback |
| `fe80::/10` | IPv6 link-local |

## 3.6 Adding an address by hand (temporary)

```bash
sudo ip addr add 10.0.0.50/32 dev ens4    # works until reboot or network restart
sudo ip addr del 10.0.0.50/32 dev ens4
```

Anything set with `ip` is **runtime only**. Permanent changes go in the network configuration (Phase 4) — the same "runtime vs saved" idea as `systemctl set-property --runtime` in Note 13.

**Used in:** Note 16, Steps 0, 1, 7.

---

# Phase 4 — How the address gets there: DHCP, netplan, networkd

## 4.1 The chain on Ubuntu Server (and GCP images)

```
cloud-init (first boot)
   │  writes
   ▼
/etc/netplan/50-cloud-init.yaml      ← netplan: what you WANT, in YAML
   │  netplan generate
   ▼
/run/systemd/network/*.network       ← config for the network daemon
   ▼
systemd-networkd                     ← the daemon that APPLIES it
   │  runs a DHCP client on ens4
   ▼
DHCP server (Google's virtual network)
   │  "you are 10.0.0.4/32, gateway 10.0.0.1, MTU 1460, DNS ..."
   ▼
address + routes + MTU set on ens4
```

(Desktop systems usually use NetworkManager instead of networkd. Netplan can drive either.)

## 4.2 Netplan

```yaml
# /etc/netplan/50-cloud-init.yaml (typical GCP)
network:
  version: 2
  ethernets:
    ens4:
      dhcp4: true
      match:
        macaddress: 42:01:0a:00:00:04
      set-name: ens4
```

```bash
sudo netplan get                 # the merged config
sudo netplan try                 # apply, but ROLL BACK in 120 s unless you confirm
sudo netplan apply               # apply for real
```

**Always prefer `netplan try` on a remote server.** If the change cuts off your SSH session, it reverts by itself. `netplan apply` won't.

Don't edit `50-cloud-init.yaml` directly — cloud-init may rewrite it. Add your own file (e.g. `60-ens5.yaml`, Note 12, Step 10).

## 4.3 Checking networkd

```bash
networkctl list                 # all links and their state
networkctl status ens4          # addresses, gateway, DNS, DHCP lease, recent log
journalctl -u systemd-networkd -b
```

| `networkctl` SETUP state | Meaning |
|---|---|
| `configured` | Done — addresses and routes applied |
| `configuring` | Still working (e.g. waiting for DHCP) |
| `unmanaged` | networkd isn't managing it (e.g. `lo`, `docker0`) |
| `failed` | Something went wrong — check the journal |

**Used in:** Note 16, Steps 0, 7.

---

# Phase 5 — Neighbours: from IP address to MAC address

## 5.1 Why it's needed

IP addresses decide **where** a packet should go. But on the local link, a packet is delivered to a **MAC address**. So before sending to the gateway `10.0.0.1`, the kernel must learn the gateway's MAC address. For IPv4 that's **ARP** ("who has 10.0.0.1?"); for IPv6 it's NDP.

## 5.2 The neighbour table

```bash
ip neigh
```
```
10.0.0.1 dev ens4 lladdr 42:01:0a:00:00:01 REACHABLE
```

| State | Meaning |
|---|---|
| `REACHABLE` | Recently confirmed |
| `STALE` | Known but not recently confirmed — will check on next use. Normal |
| `DELAY` / `PROBE` | Checking now |
| `FAILED` | Couldn't get an answer — no path at the link layer |

On a GCP VM you'll normally see **only the gateway**, because of the `/32` address (Phase 3.3): everything goes to it. Google's virtual network answers the ARP. You rarely need to debug ARP on GCP — but you do on Docker bridges and physical networks, where a `FAILED` entry means the other machine isn't answering at the lowest level.

**Used in:** Note 16, Step 0.

---

# Phase 6 — Routing: deciding where a packet goes

## 6.1 The routing table

For **every** packet it sends, the kernel asks: "which route matches this destination?"

```bash
ip route
```
```
default via 10.0.0.1 dev ens4 proto dhcp src 10.0.0.4 metric 100
10.0.0.1 dev ens4 proto dhcp scope link src 10.0.0.4 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
```

Reading one line:

```
default         via 10.0.0.1   dev ens4   proto dhcp   src 10.0.0.4   metric 100
   │               │             │          │             │              │
   │               │             │          │             │              └─ preference when several routes tie (lower wins)
   │               │             │          │             └─ source IP to use for new connections
   │               │             │          └─ who added it (dhcp, kernel, static, boot)
   │               │             └─ which interface to send it out of
   │               └─ next hop: send it to this gateway
   └─ destination: "default" = 0.0.0.0/0 = everything not matched more specifically
```

| Route | Means |
|---|---|
| `default via 10.0.0.1 dev ens4` | Anything without a better match → gateway `10.0.0.1` through `ens4` |
| `10.0.0.1 dev ens4 scope link` | The gateway itself is **directly reachable** on `ens4` (needed because our address is `/32` — otherwise the kernel wouldn't know how to reach the gateway) |
| `172.17.0.0/16 dev docker0 ... linkdown` | Containers' network, on the Docker bridge; currently no containers |

## 6.2 Longest prefix match

If several routes match, the **most specific** (longest prefix) wins:

```
destination 172.17.0.5:
   default (0.0.0.0/0)       matches  — prefix 0
   172.17.0.0/16             matches  — prefix 16   ← wins
```

## 6.3 The most useful routing command: `ip route get`

Don't work out the route in your head. Ask the kernel:

```bash
ip route get 10.0.0.5
```
```
10.0.0.5 via 10.0.0.1 dev ens4 src 10.0.0.4 uid 1000
```

**Meaning:** to reach the database VM in the **same** subnet, the kernel still goes **via the gateway** (GCP `/32`, Phase 3.3), out of `ens4`, using source address `10.0.0.4`.

```bash
ip route get 8.8.8.8                 # to the internet
ip route get 172.17.0.5              # to a container
ip route get 172.16.5.20 from 10.10.0.4   # "if a packet came FROM this address, where would its reply go?"
```

The last form matters for servers with **two** interfaces (Phase 6.5).

## 6.4 Routing works in both directions

A connection needs a working path **there and back**. The other side's routing decides the way back to you — and **your** routing decides the way back to clients. A route that's fine outbound can still be the reason replies never arrive.

## 6.5 Multiple tables and policy routing (brief)

Linux can have **several** routing tables, chosen by **rules**:

```bash
ip rule                     # which table to use, based on source address etc.
ip route show table local   # the kernel's table for this machine's own addresses
ip route show table 100     # a custom table
```

Normally only `main` is used. Servers with **two network interfaces** often need a rule like "packets **from** the second interface's address use a table whose default route goes **out** the second interface" — otherwise replies leave through the wrong interface. Note 16, Step 7 shows this on GCP.

## 6.6 Reverse path filtering

`rp_filter` makes the kernel check: "if I replied to this packet's source, would the reply go out the interface it came in on?"

| Value | Effect |
|---|---|
| `0` | Off |
| `1` (strict) | Drop packets that fail the check — breaks multi-interface setups |
| `2` (loose) | Only drop if there's **no** route back at all. Ubuntu's default |

```bash
sysctl net.ipv4.conf.all.rp_filter net.ipv4.conf.ens4.rp_filter
```

**Used in:** Note 16, Steps 0, 3, 7.

---

# Phase 7 — Forwarding, NAT, and network namespaces

## 7.1 Forwarding: acting as a router

By default a server only handles packets **addressed to itself**. With **IP forwarding** on, it will pass packets **between** interfaces — acting as a router.

```bash
sysctl net.ipv4.ip_forward      # 0 = off, 1 = on
```

Docker turns it **on**, because containers' packets must be forwarded between `docker0` and `ens4`.

## 7.2 NAT: changing addresses on the way through

**NAT (Network Address Translation)** rewrites addresses in packets:

| Kind | Where you see it |
|---|---|
| **Source NAT / masquerade** | Containers (`172.17.0.x`) reach the internet looking like the host (`10.0.0.4`) |
| **Destination NAT / port forwarding** | `docker run -p 8080:80` — traffic to host port 8080 is sent to the container's port 80 |
| **1:1 NAT at the cloud edge** | GCP's external IP → internal IP (Phase 3.4) |

On the host, NAT rules live in nftables/iptables (Note 17).

## 7.3 Network namespaces: separate network stacks

A **network namespace** is a complete, separate copy of the network stack: its **own** interfaces, addresses, routing table, sockets, and firewall rules. Containers use this.

```
host namespace                         container namespace
──────────────                         ───────────────────
lo   127.0.0.1                         lo    127.0.0.1    ← a DIFFERENT loopback
ens4 10.0.0.4                          eth0  172.17.0.2
docker0 172.17.0.1 ──── veth pair ──── (eth0 is the other end of the "cable")
routing: default via 10.0.0.1          routing: default via 172.17.0.1
```

The key consequence: **`127.0.0.1` inside a container is the container itself**, not the host. A container can't reach a host service on `127.0.0.1`. (Note 16, Step 9.)

Looking inside a namespace:

```bash
sudo ip netns list                                   # named namespaces (Docker's aren't named)
PID=$(docker inspect -f '{{.State.Pid}}' worker)
sudo nsenter -t $PID -n ip addr                      # run a host command inside the container's network
sudo nsenter -t $PID -n ss -tnp
```

`nsenter -n` uses the **host's** tools inside the container's network — useful when the container image has no `ip` or `ss`.

**Used in:** Note 16, Steps 8, 9.

---

# Phase 8 — Ports and the two transport protocols

## 8.1 Why ports exist

An IP address gets a packet to the right **machine**. A **port** (a number from 1 to 65535) gets it to the right **program** on that machine.

## 8.2 TCP vs UDP

| | TCP | UDP |
|---|---|---|
| Connection | Set up first (handshake, Phase 10) | None — just send |
| Delivery | Reliable, ordered; lost data is resent | No guarantee, no order |
| Used by | HTTP(S), SSH, PostgreSQL, most app traffic | DNS (mostly), NTP, video/voice, QUIC (HTTP/3) |
| "Is anything there?" | Clear answer: connected, refused, or timed out | Often **no answer either way** — harder to test |

## 8.3 Port ranges

| Range | Name | Notes |
|---|---|---|
| 1–1023 | Well-known / privileged | Binding needs root or `CAP_NET_BIND_SERVICE` (Note 8, Step 1) |
| 1024–49151 | Registered | e.g. 3000, 5432, 6379, 8080 |
| 32768–60999 | **Ephemeral** (Linux default) | Picked automatically for the **client** side of outgoing connections |

```bash
sysctl net.ipv4.ip_local_port_range      # 32768  60999 → 28,232 ports
```

## 8.4 A connection is identified by four things

```
(source IP, source port, destination IP, destination port)   + protocol
```

So one server port (`10.0.0.4:80`) can have **thousands** of connections at once — each differs in the client's IP or port. And one client can open many connections to the same server — each uses a different **ephemeral source port**, which is a limited supply (Note 16, Step 5).

**Used in:** Note 16, Steps 2, 5.

---

# Phase 9 — Sockets: how programs use the network

## 9.1 A socket is a file descriptor

A program doesn't talk to "the network". It asks the kernel for a **socket** — an endpoint that, like a file, gets a **file descriptor** (Note 9, Phase 2.3). Reading and writing that fd sends and receives data.

```bash
sudo ls -l /proc/2890/fd | grep socket | head -3
# 19 -> socket:[48211]
```

Every open connection uses one fd — which is why busy servers hit the open-files limit (Note 14, Step 9).

## 9.2 Server side vs client side

```
SERVER (the API)                         CLIENT (the worker)
socket()                                 socket()
bind(0.0.0.0, 80)   ← pick address+port  │
listen()            ← start accepting    │
accept()  ◄──────── handshake ──────────── connect(10.0.0.4, 80)
   │                                        ↑ kernel picks source port
   │                                          (ephemeral) and source IP
   ▼                                          (from the routing table)
new socket for THIS connection            connected socket
read/write ◄────────── data ─────────────► read/write
close()                                   close()
```

- The **listening socket** stays open and keeps accepting.
- Each `accept()` creates a **new socket** for one connection.

## 9.3 The bind address decides who can connect

This one detail causes a large share of "it works locally but not from outside" problems.

| Bind address | `ss` shows | Who can connect |
|---|---|---|
| `127.0.0.1` | `127.0.0.1:3001` | **Only this machine** |
| `10.0.0.4` | `10.0.0.4:3001` | Only via that one address |
| `0.0.0.0` | `0.0.0.0:3001` | All **IPv4** addresses of the machine |
| `::` (dual-stack) | `*:3001` or `[::]:3001` | All **IPv6 and IPv4** addresses (IPv4 arrives as `::ffff:10.0.0.4`) |
| `::1` | `[::1]:3001` | Only this machine, IPv6 |

Many frameworks default to `127.0.0.1` or `localhost` (safe for development). Node's `server.listen(80)` without a host binds `::` → shows as `*:80`.

Binding wider than needed is a security issue: a database or debug port on `0.0.0.0` is reachable by anything that can reach the VM.

## 9.4 Only one listener per address and port

Two programs can't both listen on `0.0.0.0:80`. The second gets:

```
Error: listen EADDRINUSE: address already in use :::80
```

(Exceptions: `SO_REUSEPORT`, where cooperating processes share a port on purpose; and binding different specific addresses.)

## 9.5 Unix domain sockets

Programs on the **same machine** can also talk through a socket **file** instead of TCP:

```
/run/php/php8.3-fpm.sock
/var/run/docker.sock                     (Note 4, Step 9)
/var/run/postgresql/.s.PGSQL.5432
```

- No ports, no network — faster, and not reachable from outside at all.
- **Access is controlled by file permissions** (Note 5). That's why `docker.sock` being group-writable by `docker` matters so much.

```bash
ss -xlp | head          # listening Unix sockets
```

**Used in:** Note 16, Steps 1, 2, 9, 10.

---

# Phase 10 — The life of a TCP connection

## 10.1 Opening: the three-way handshake

```
client                                 server (LISTEN)
  │ ── SYN ──────────────────────────────►│   "I want to connect"
  │                                       │   state: SYN-RECV
  │ ◄───────────────────────── SYN-ACK ── │   "OK, I'm ready"
  │ ── ACK ──────────────────────────────►│
  │   ESTABLISHED                         │   ESTABLISHED  → waits in the accept queue
  │                                       │   until the app calls accept()
```

## 10.2 Closing

The side that closes **first** sends FIN:

```
side A (closes first)                  side B
FIN-WAIT-1  ── FIN ──────────────────►  CLOSE-WAIT   ← B's app must now call close()
FIN-WAIT-2  ◄─────────────────── ACK
            ◄─────────────────── FIN    LAST-ACK     ← sent when B's app closes
TIME-WAIT   ── ACK ──────────────────►  CLOSED
   (waits 60 s)
CLOSED
```

## 10.3 The states you'll actually see

| State | Meaning | When it's a problem |
|---|---|---|
| `LISTEN` | Waiting for connections | Missing = nothing is listening |
| `SYN-SENT` | We sent SYN, no answer yet | Many/stuck = the other side or a firewall **isn't answering** (timeout coming) |
| `SYN-RECV` | Got SYN, sent SYN-ACK, waiting for ACK | Huge numbers = SYN flood or overloaded server |
| `ESTABLISHED` | Connected, data can flow | Normal |
| `CLOSE-WAIT` | The other side closed; **our app hasn't** | **Growing count = a bug in our app**: it doesn't close sockets. Leaks file descriptors |
| `FIN-WAIT-2` | We closed; waiting for the other side to close | Many = the other side doesn't close |
| `TIME-WAIT` | We closed first; waiting 60 s so late packets can't confuse a new connection | **Normal**. Huge numbers on a **client** can exhaust ephemeral ports (Note 16, Step 5) |

A quick rule: **CLOSE-WAIT is your fault; TIME-WAIT is usually normal.**

## 10.4 Queues on a listening socket

Between "handshake done" and "app called `accept()`", connections wait in the **accept queue**. Its size (the **backlog**) is the smaller of what the app asked for and `net.core.somaxconn`.

If the app is too slow to accept (busy event loop, overloaded), the queue fills, and new connections are **dropped** — clients see slow connects or timeouts while the server looks "up". You can see this directly in `ss` (Phase 12) and with counters:

```bash
nstat -az TcpExtListenOverflows TcpExtListenDrops
```

**Used in:** Note 16, Steps 4, 5, 6.

---

# Phase 11 — What the connection errors really mean

Each error tells you **which layer** failed, and so which of the five questions (Phase 1.3) to ask next.

| Error | What actually happened | Layer / question | First thing to check |
|---|---|---|---|
| **Connection refused** (`ECONNREFUSED`) | The packet **reached** the machine, and it answered with **RST**: nothing listening on that port (or a firewall set to *reject*) | Q3: socket | `ss -tlnp` on the server: listening? on which address? |
| **Connection timed out** (`ETIMEDOUT`) | **No answer at all** — packets dropped somewhere | Q4: firewall (cloud or host), or Q2: route, or host down | From several places; firewall rules; `ip route get` |
| **No route to host** (`EHOSTUNREACH`) | A router (or your own kernel/firewall) said the host can't be reached | Q2 / Q4 | `ip route get`; reject rules |
| **Network is unreachable** (`ENETUNREACH`) | **Your own** kernel has **no route** for that destination — the packet never left | Q1/Q2: local address or route missing | `ip route` — empty or missing default? (Note 12, Step 3 at boot) |
| **Connection reset by peer** (`ECONNRESET`) | Connected, then the other side sent **RST** mid-conversation | App crashed/closed hard; proxy/LB idle timeout | Logs on the other side; timeouts on proxies |
| **Address already in use** (`EADDRINUSE`) | **Your** program tried to listen on a port that's taken | Q3: local socket | `ss -tlnp 'sport = :PORT'` |
| **Cannot assign requested address** (`EADDRNOTAVAIL`) | Binding to an IP not on this machine — or **out of ephemeral ports** | Q3 | `ip addr`; `ss -s` TIME-WAIT count |
| **Name or service not known** / `ENOTFOUND` | DNS lookup failed — no IP at all | Q5 | Note 17 |

The two to memorise: **refused = you reached the machine, nothing's listening. Timeout = something in between dropped it.**

**Used in:** Note 16, Steps 1, 3, and throughout.

---

# Phase 12 — Reading `ss` like a pro

`ss` (socket statistics) replaced the old `netstat`.

## 12.1 The options

| Option | Meaning |
|---|---|
| `-t` / `-u` / `-x` | TCP / UDP / Unix sockets |
| `-l` | Only **listening** sockets |
| `-a` | All (listening + connected) |
| `-n` | Show **numbers**, don't look up names (faster, clearer) |
| `-p` | Show the **process** (needs root to see other users' processes) |
| `-e` | Extra: socket owner UID, inode |
| `-i` | TCP internals: round-trip time, retransmits |
| `-s` | Summary counts |

The two you'll use most:

```bash
sudo ss -tulpn          # everything listening, TCP and UDP, with processes
sudo ss -tnp            # current TCP connections, with processes
```

## 12.2 Reading listening sockets

```bash
sudo ss -tulpn
```
```
Netid State  Recv-Q Send-Q  Local Address:Port  Peer Address:Port Process
udp   UNCONN 0      0       127.0.0.53%lo:53         0.0.0.0:*     users:(("systemd-resolve",pid=512,fd=13))
tcp   LISTEN 0      4096    127.0.0.53%lo:53         0.0.0.0:*     users:(("systemd-resolve",pid=512,fd=14))
tcp   LISTEN 0      4096          0.0.0.0:22         0.0.0.0:*     users:(("sshd",pid=905,fd=3),("systemd",pid=1,fd=79))
tcp   LISTEN 0      511                 *:80               *:*     users:(("node",pid=2890,fd=19))
tcp   LISTEN 0      511           0.0.0.0:443        0.0.0.0:*     users:(("nginx",pid=1201,fd=6),...)
```

| Column | For a LISTEN socket |
|---|---|
| `Recv-Q` | Connections **waiting in the accept queue right now** |
| `Send-Q` | The accept queue's **maximum** (backlog) |
| `Local Address:Port` | **Bind address** (Phase 9.3) and port |
| `Process` | Who owns it. `systemd` with `sshd` = **socket activation** — systemd holds the socket and starts sshd (Note 11, Phase 2.1, `.socket` units) |

`127.0.0.53%lo` = bound to that address on the loopback only (the local DNS resolver, Note 17).

## 12.3 Reading connections

```bash
sudo ss -tnp
```
```
State  Recv-Q Send-Q   Local Address:Port    Peer Address:Port  Process
ESTAB  0      0             10.0.0.4:42118        10.0.0.5:5432  users:(("node",pid=2890,fd=24))
ESTAB  0      0      [::ffff:10.0.0.4]:80   [::ffff:203.0.113.8]:51022 users:(("node",pid=2890,fd=31))
```

| Column | For a connection |
|---|---|
| `Recv-Q` | Bytes received but **not yet read by the app** — growing = app is slow to read |
| `Send-Q` | Bytes sent but **not yet acknowledged** by the other side — growing = network or the other side is slow |

## 12.4 Filters

```bash
ss -tn state established                       # by state
ss -tan state close-wait                       # leaked sockets?
ss -tn state established '( dport = :5432 )'   # connections TO port 5432
ss -tln '( sport = :80 )'                      # listener ON port 80
ss -tn dst 10.0.0.5                            # connections to one host
ss -s                                          # totals by state
```

`lsof -i :80` gives similar answers with a different layout; `netstat` is deprecated and may not be installed.

**Used in:** every step of Note 16.

---

# Phase 13 — Reachability tools: ping, traceroute, MTU

## 13.1 `ping`

`ping` sends **ICMP echo** packets and waits for replies.

```bash
ping -c 3 10.0.0.5
```

Use it carefully:
- A **failed ping doesn't mean TCP will fail.** Many networks block ICMP. On GCP, ICMP is only allowed if a firewall rule allows it.
- A **successful ping doesn't mean your port is open.** It only proves IP-level reachability.

For "can I reach this **port**?", test the port directly: `nc -vz host port` or `curl` (Note 16).

## 13.2 Path tools

| Tool | Shows |
|---|---|
| `traceroute host` / `tracepath host` | Each router on the way, by sending packets with increasing TTL |
| `mtr host` | `traceroute` + `ping` combined, live, with loss per hop |

Inside a cloud VPC, paths are short and hops often don't answer — these tools are more useful for internet paths. Details in Note 19.

## 13.3 MTU: the maximum packet size

The **MTU** is the largest packet an interface sends. Ethernet default is **1500**; **GCP's default is 1460**.

If a path contains a link with a **smaller** MTU than the sender assumes, large packets must be split or must trigger an ICMP "too big" message. If those ICMP messages are blocked, or a virtual interface assumes a bigger MTU than the real one, you get a very specific symptom:

**small requests work; large ones hang** (e.g. TLS handshakes with big certificates, large API responses).

Testing the largest packet that passes without splitting:

```bash
ping -c 2 -M do -s 1432 10.0.0.5     # 1432 data + 28 header = 1460 → OK
ping -c 2 -M do -s 1433 10.0.0.5     # 1461 → "message too long, mtu=1460"
```

`-M do` = "don't fragment". Note 16, Step 8 shows the classic Docker-on-GCP version.

**Used in:** Note 16, Step 8.

---

# Phase 14 — Kernel network settings (sysctl)

Many network behaviours are kernel settings, read and changed with `sysctl`:

| Setting | Default (Ubuntu 24.04) | Meaning |
|---|---|---|
| `net.ipv4.ip_forward` | 0 (1 with Docker) | Act as a router (Phase 7.1) |
| `net.ipv4.ip_local_port_range` | `32768 60999` | Ephemeral ports (Phase 8.3) |
| `net.core.somaxconn` | 4096 | Max accept queue (Phase 10.4) |
| `net.ipv4.tcp_max_syn_backlog` | depends on memory | Max half-open connections |
| `net.ipv4.conf.*.rp_filter` | 2 | Reverse path filter (Phase 6.6) |
| `net.ipv4.tcp_tw_reuse` | 2 (loopback only) | Reuse TIME-WAIT ports for new **outgoing** connections when safe |
| `net.ipv4.tcp_fin_timeout` | 60 | How long in FIN-WAIT-2 (**not** TIME-WAIT, which is fixed at 60 s) |

```bash
sysctl net.ipv4.ip_local_port_range           # read
sudo sysctl -w net.ipv4.tcp_tw_reuse=1        # change now (lost at reboot)
echo 'net.ipv4.tcp_tw_reuse = 1' | sudo tee /etc/sysctl.d/60-myapps.conf
sudo sysctl --system                          # load all files in /etc/sysctl.d
```

Rule: **change a sysctl only to fix a problem you measured.** Copying "performance tuning" lists from the internet often breaks things quietly.

**Used in:** Note 16, Steps 5, 6, 11.

---

# How this goes wrong — overview

| Issue | Why it matters | How to check |
|---|---|---|
| Services bound to `0.0.0.0` that should be local | Databases, Redis, admin panels, debuggers reachable by anything that reaches the VM | `sudo ss -tulpn` — review every non-loopback listener |
| Node debugger (`--inspect=0.0.0.0:9229`) | Anyone who connects can **run code** in the process | `ss -tlnp` for 9229 |
| Unexpected listeners | Backdoors, bind shells | Compare `ss -tulpn` with a baseline |
| Unexpected outbound connections | Miners, data theft, command-and-control (Note 10, Step 9) | `ss -tnp` → remote addresses per process |
| **Metadata server access from the app** | SSRF: an attacker makes the app fetch `169.254.169.254/...token` → **cloud credentials** | Can the app fetch user-supplied URLs? Test with and without the required header |
| IP forwarding on by accident | Server can route traffic between networks | `sysctl net.ipv4.ip_forward` |
| Docker published ports | `-p 8080:80` listens on `0.0.0.0` and Docker's rules can bypass host firewall rules (Note 17) | `ss -tlnp`, `docker ps` |
| Unix sockets with loose permissions | Local users talk to privileged daemons (`docker.sock`) | `ls -l` on socket files |
| Packet capture | `tcpdump` sees all traffic; needs `CAP_NET_RAW` | Who has it (Note 7) |

---

# Summary table

| Concept | Key point |
|---|---|
| Five questions | Interface/address → route → socket → firewall → name |
| Interface | Where packets enter/leave; `UP` + `LOWER_UP` = enabled and connected |
| MTU | Max packet size; GCP = 1460 |
| MAC | Link-layer address; GCP MACs start `42:01` |
| Prefix length | Bigger number = smaller network; `/32` = one address |
| GCP `/32` | Every packet goes via the gateway, even within the subnet |
| External IP | Not on the VM; translated at Google's edge; find it via metadata |
| `scope host/link/global` | Machine only / local link / everywhere |
| `ip addr add` | Runtime only; permanent config in netplan |
| netplan → networkd | YAML → daemon; use `netplan try` remotely |
| Neighbours (ARP) | IP → MAC on the local link; on GCP only the gateway |
| Routing table | Longest prefix wins; `default` = everything else |
| `ip route get` | Ask the kernel which route, interface, and source it will use |
| Both directions | Replies need a route back |
| Policy routing | `ip rule` + extra tables; needed for multi-NIC replies |
| `rp_filter` | Reverse path check; Ubuntu = 2 (loose) |
| Forwarding / NAT | Router behaviour; containers, port publishing, cloud external IPs |
| Network namespace | Separate stack; container `127.0.0.1` ≠ host |
| TCP vs UDP | Connection + reliable vs fire-and-forget |
| Ports | <1024 privileged; ephemeral 32768–60999 |
| 4-tuple | src IP, src port, dst IP, dst port identify a connection |
| Socket | An fd; server: bind → listen → accept; client: connect |
| Bind address | `127.0.0.1` local only; `0.0.0.0` all IPv4; `::`/`*` all |
| EADDRINUSE | Port taken by another listener |
| Unix socket | File-based; protected by file permissions |
| Handshake | SYN → SYN-ACK → ACK |
| CLOSE-WAIT | Peer closed, our app didn't → leak in our code |
| TIME-WAIT | Normal after closing first; can exhaust client ports |
| Accept queue | Full when the app is slow to accept → dropped connections |
| Refused vs timeout | Reached and nothing listening vs dropped on the way |
| `ss` | `-tulpn` listeners; `-tnp` connections; Recv-Q/Send-Q meaning depends on state |
| ping | ICMP only; doesn't prove a port is open or closed |
| MTU symptom | Small works, large hangs |
| sysctl | Change only what you've measured |

---

# Appendix — Command reference

## Interfaces and addresses

| Command | Purpose |
|---|---|
| `ip -br link` / `ip link show DEV` | Interfaces, state, MTU, MAC |
| `ip -s link show DEV` | Counters: errors, drops |
| `ip -br addr` / `ip addr show DEV` | Addresses |
| `sudo ip addr add/del ADDR/PFX dev DEV` | Runtime address change |
| `sudo ip link set DEV up/down` | Enable/disable (careful over SSH) |
| `networkctl list` / `networkctl status DEV` | networkd state |
| `sudo netplan get` / `try` / `apply` | Netplan config |
| `ip neigh` | Neighbour (ARP) table |

## Routing

| Command | Purpose |
|---|---|
| `ip route` | Main routing table |
| `ip route get DEST` | Chosen route, interface, source |
| `ip route get DEST from SRC` | Route for replies from a given address |
| `ip rule` / `ip route show table N` | Policy routing |
| `sysctl net.ipv4.ip_forward` | Forwarding on/off |
| `sysctl net.ipv4.conf.all.rp_filter` | Reverse path filter |

## Sockets

| Command | Purpose |
|---|---|
| `sudo ss -tulpn` | Listening TCP/UDP + processes |
| `sudo ss -tnp` | TCP connections + processes |
| `ss -tan state STATE` | By state (`established`, `close-wait`, `time-wait`, `syn-sent`) |
| `ss -tn '( dport = :PORT )'` / `'( sport = :PORT )'` | Filter by port |
| `ss -tn dst IP` | Filter by remote host |
| `ss -s` | Summary |
| `ss -xlp` | Unix sockets |
| `ss -tni` | TCP internals (rtt, retransmits) |
| `nstat -az TcpExtListenOverflows TcpExtListenDrops` | Accept queue drops |
| `lsof -i :PORT` | Alternative view |

## Namespaces

| Command | Purpose |
|---|---|
| `docker inspect -f '{{.State.Pid}}' NAME` | A container's PID |
| `sudo nsenter -t PID -n COMMAND` | Run a host command inside its network namespace |
| `sudo ip netns list` | Named namespaces |

## Reachability

| Command | Purpose |
|---|---|
| `ping -c 3 HOST` | ICMP reachability |
| `ping -M do -s SIZE HOST` | MTU test |
| `tracepath HOST` / `traceroute HOST` / `mtr HOST` | Path |
| `nc -vz -w 3 HOST PORT` | Is a TCP port reachable? |

## Kernel settings

| Command | Purpose |
|---|---|
| `sysctl NAME` | Read |
| `sudo sysctl -w NAME=VAL` | Change now |
| `/etc/sysctl.d/*.conf` + `sudo sysctl --system` | Persist |