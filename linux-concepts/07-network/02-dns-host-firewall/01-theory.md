# DNS & Host Firewall — Theory (Note 17)

Two of the five networking questions from Note 15, explained from the ground up:

- **Q5 — Name:** how a name like `db-02` or `api.mailprovider.example` becomes an IP address, on the internet and inside a Linux server.
- **Q4 — Firewall (on the VM):** how the Linux kernel decides, for every packet, whether to let it through — and how that stacks with the cloud firewall.

**Series:**
- Notes 1–14: Shell, Environment, Users, Files, Privilege, Processes, systemd, Logs/Timers/Limits
- Note 15 / 16: Interfaces, Routing & Sockets — Theory / Practice
- **Note 17: DNS & Host Firewall — Theory** ← this note
- Note 19: SSH & Network Debugging — Theory

Both topics are explained assuming no prior knowledge. Each phase has a short **"On our server"** example from the team server (`app-server-01`, Notes 4–16), and, where relevant, a **"How this goes wrong"** box for the security track.

---

## Contents

**Part A — DNS**
1. Phase 1 — Why names exist
2. Phase 2 — How names are organised
3. Phase 3 — DNS records: what a name can point to
4. Phase 4 — The journey of one lookup
5. Phase 5 — Caching and TTL
6. Phase 6 — Answers, failures, and the errors apps show
7. Phase 7 — How Linux resolves a name: the local chain
8. Phase 8 — Search domains and short names
9. Phase 9 — DNS inside GCP
10. Phase 10 — Tools: which one follows which path
11. Phase 11 — DNS gotchas in real applications

**Part B — Host firewall**
12. Phase 12 — What a firewall does
13. Phase 13 — Stateful firewalls and connection tracking
14. Phase 14 — Where filtering happens: cloud vs host vs app
15. Phase 15 — Netfilter: the packet's path through the kernel
16. Phase 16 — The tools: nftables, iptables, ufw — and who's in charge
17. Phase 17 — nftables from the ground up
18. Phase 18 — A complete ruleset for our server
19. Phase 19 — ufw: the simple front-end
20. Phase 20 — Drop vs reject: what the client sees
21. Phase 21 — Docker and the host firewall
22. Phase 22 — Changing firewall rules without locking yourself out
23. Phase 23 — Logging, counters, and the connection table

- How this goes wrong — overview
- Summary table
- Appendix — Command reference

---

# Part A — DNS

# Phase 1 — Why names exist

## 1.1 The problem

Computers connect using **IP addresses** (Note 15). People don't want to remember `10.0.1.5` or `142.250.193.14`. And even for programs, hard-coding IPs is fragile:

| Problem with raw IPs | Example |
|---|---|
| IPs change | The database moves from `10.0.0.5` to `10.0.1.5` (Note 16, Step 3). Every config with the old IP breaks |
| One service, many IPs | A big API runs on dozens of servers; a name can return several IPs to spread the load |
| Many services, one IP | One server hosts `api.example.com` and `admin.example.com`; the name tells it which site you want |

**DNS (Domain Name System)** is the system that turns a **name** into an **IP address** (and other information). Change the IP in DNS once, and every client that looks up the name gets the new one.

## 1.2 What it looks like from a program

A program doesn't "know" DNS. It just asks the operating system: "give me the address for `db-02.internal`". The OS does the lookup and hands back an IP. Then the program connects to that IP (Note 15, Phase 9.2).

```
app:   connect("db-02.internal", 5432)
         │
         ├─ 1. name → IP     (this part: DNS)          → 10.0.1.5
         └─ 2. connect to 10.0.1.5:5432 (Note 15)
```

So a "connection" problem can actually be a **name** problem — the app never got the right IP, or any IP at all. That's question **Q5**.

---

# Phase 2 — How names are organised

## 2.1 Labels, read right to left

A name like `api.mailprovider.example` is made of **labels** separated by dots. It's a **tree**, read **right to left**, from general to specific:

```
                      .                  ← the root (usually invisible)
         ┌────────────┼────────────┐
        com          lk         example   ← top-level domains (TLDs)
         │            │            │
      google        mo        mailprovider    ← domains
         │            │            │
        www          www          api          ← subdomains / hosts
```

| Part | Example | Meaning |
|---|---|---|
| Root | `.` | The top of the tree |
| TLD | `com`, `lk`, `org` | Top-level domain |
| Domain | `mailprovider.example` | Registered by an organisation |
| Subdomain / host | `api.mailprovider.example` | Named by the domain's owner |

A **fully qualified domain name (FQDN)** is the complete name to the root. Strictly, it ends with a dot: `api.mailprovider.example.` — the final dot is the root. Usually it's left out; it matters for search domains (Phase 8).

## 2.2 Zones and who is responsible

The tree is split into **zones**. Each zone is managed by an organisation that runs (or rents) **authoritative name servers** — the servers that hold the **official** answers for names in that zone.

```
root zone          → run by root server operators; knows who handles each TLD
"example" TLD zone → knows which servers handle "mailprovider.example"
mailprovider.example zone → the company's own DNS servers (or Cloudflare, Cloud DNS, Route 53…)
                            hold the records for api.mailprovider.example
```

Each level only knows **who to ask next** for the level below. This is called **delegation**, and it's what lets the system scale to the whole internet.

**On our server:** the team's public domain is hosted on a DNS provider (e.g. Cloudflare or Cloud DNS). Its authoritative servers hold the record for `api.team.example` → `192.0.2.10` (the VM's external IP from Note 16).

---

# Phase 3 — DNS records: what a name can point to

## 3.1 Records

Each zone is a list of **records**. A record has a **name**, a **TTL** (Phase 5), a **type**, and a **value**:

```
api.team.example.    300    IN    A    192.0.2.10
│                    │      │     │    │
│                    │      │     │    └─ value
│                    │      │     └─ type
│                    │      └─ class (always IN = internet)
│                    └─ TTL in seconds: how long others may cache this answer
└─ name
```

## 3.2 The record types you'll meet

| Type | Holds | Example value | Used for |
|---|---|---|---|
| **A** | An IPv4 address | `192.0.2.10` | Name → IPv4. The most common |
| **AAAA** | An IPv6 address | `2001:db8::10` | Name → IPv6 |
| **CNAME** | **Another name** (an alias) | `app.hostingprovider.example.` | "This name is the same as that name — look that one up" |
| **MX** | Mail server for the domain | `10 mx1.mailhost.example.` | Where to deliver email |
| **TXT** | Free text | `"v=spf1 include:_spf.google.com ~all"` | Email security (SPF, DKIM, DMARC), domain ownership proofs |
| **NS** | Name servers for a zone | `ns1.dnsprovider.example.` | Delegation (Phase 2.2) |
| **SOA** | Zone details | serial, timers | Zone management, negative-cache time |
| **PTR** | A **name** for an IP (reverse) | `api.team.example.` | IP → name (Phase 3.4) |
| **SRV** | Host **and port** for a service | `10 5 5060 sip.example.` | Some protocols find servers this way |
| **CAA** | Which certificate authorities may issue certificates | `0 issue "letsencrypt.org"` | Security |

## 3.3 CNAME — the alias, and its rules

```
www.team.example.    300  IN  CNAME  api.team.example.
api.team.example.    300  IN  A      192.0.2.10
```

Looking up `www.team.example` returns the CNAME, then the resolver follows it to `api.team.example` and gets the A record. Two rules:

- A name with a CNAME can't have **any other** records.
- So the **bare domain** (`team.example`, called the "apex") can't be a CNAME, because it must have NS and SOA records. Providers work around this with "CNAME flattening" or "ALIAS" records — they resolve the target themselves and return A records.

## 3.4 Reverse DNS (PTR)

Normal DNS: name → IP. **Reverse DNS**: IP → name. It uses a special zone where the IP is written backwards:

```
10.0.0.4  →  4.0.0.10.in-addr.arpa.  PTR  app-server-01.internal.
```

It matters for: readable logs (names instead of IPs), email servers (many reject mail from IPs without a matching PTR), and some security tools. The **owner of the IP range** controls PTR records — for cloud IPs, that's set through the cloud provider.

---

# Phase 4 — The journey of one lookup

## 4.1 The players

| Player | What it does | Example |
|---|---|---|
| **Stub resolver** | Tiny client inside the operating system. Asks one question, waits for one final answer | glibc in your app + systemd-resolved (Phase 7) |
| **Recursive resolver** | Does the **work**: asks root, TLD, and authoritative servers in turn; caches results | Your ISP's DNS, `8.8.8.8`, `1.1.1.1`, **GCP's metadata server `169.254.169.254`** |
| **Authoritative servers** | Hold the official records for their zones | Root servers, TLD servers, the domain's DNS provider |

## 4.2 Step by step

The worker on `app-server-01` looks up `api.mailprovider.example` for the first time:

```
 worker
   │ 1. "address of api.mailprovider.example?"
   ▼
 stub resolver on the VM (systemd-resolved, 127.0.0.53)
   │ 2. not in cache → ask the recursive resolver
   ▼
 recursive resolver (GCP, 169.254.169.254)
   │ 3. not in cache → start at the root
   ├──► root server:            "who handles .example?"       → "ask these TLD servers"
   ├──► .example TLD server:    "who handles mailprovider.example?" → "ask ns1.dnsprovider.example"
   ├──► ns1.dnsprovider.example: "A record for api.mailprovider.example?" → "203.0.113.25, TTL 300"
   │
   │ 4. cache it for 300 s, return it
   ▼
 systemd-resolved: cache it, return it
   ▼
 worker: connect to 203.0.113.25:443
```

| Term | Meaning |
|---|---|
| **Recursive** query | "Give me the **final answer**" (what the stub asks the recursive resolver) |
| **Iterative** queries | "Tell me who to ask next" (what the recursive resolver does with root/TLD/authoritative servers) |

The second time any program on the VM asks — within 5 minutes — the answer comes straight from a cache in step 2 or 3. That's why DNS is fast most of the time.

## 4.3 DNS on the wire

| Transport | Port | When |
|---|---|---|
| **UDP** | 53 | Normal queries (small, fast) |
| **TCP** | 53 | Large answers (the UDP reply says "truncated, retry on TCP"), zone transfers |
| DNS over TLS (DoT) | 853 | Encrypted DNS |
| DNS over HTTPS (DoH) | 443 | Encrypted DNS inside HTTPS |

**Firewall consequence:** if you restrict outgoing traffic, allow **both UDP and TCP** port 53 to your resolver. Allowing only UDP works until an answer is large — then lookups fail mysteriously.

---

# Phase 5 — Caching and TTL

## 5.1 Everyone caches

Answers are cached by: the recursive resolver, the local stub (systemd-resolved), sometimes the **application runtime** (Java), browsers, and connection pools that keep using an IP they already have.

The **TTL** on a record tells caches how long they may keep it. `TTL 300` = 5 minutes.

## 5.2 What "DNS propagation" really means

When you change a record, nothing is "sent out". Caches around the world simply keep the **old** answer until **their copy's TTL runs out**, then ask again. "Waiting for DNS to propagate" = waiting for old cached copies to expire.

| TTL | Changes take effect | Cost |
|---|---|---|
| 60 s | Within ~1 minute | More queries to your DNS servers |
| 3600 s | Up to an hour | Fewer queries, slow to change |

**Migration trick:** a day before moving a service to a new IP, **lower the TTL** (e.g. 3600 → 60). After the move, raise it again.

## 5.3 Negative caching

"This name **doesn't exist**" (NXDOMAIN, Phase 6) is cached too — for a time set in the zone's SOA record. So if a service looks up a name **before** you create it, it may keep getting "doesn't exist" for a while **after** you create it.

**On our server:**

```bash
resolvectl statistics          # cache hits/misses in systemd-resolved
sudo resolvectl flush-caches   # clear the VM's local cache
```

Flushing only clears **this VM's** cache — not the recursive resolver's, and not caches inside running apps.

---

# Phase 6 — Answers, failures, and the errors apps show

## 6.1 Kinds of DNS answer

| Answer | Meaning | Typical cause |
|---|---|---|
| **NOERROR** + records | Success | — |
| **NOERROR**, no records ("NODATA") | The **name exists**, but has no record of the **type** asked | Asked for AAAA (IPv6), only A exists — normal; or a typo in record type |
| **NXDOMAIN** | The **name doesn't exist** at all | Typo, record not created, wrong search domain (Phase 8) |
| **SERVFAIL** | The resolver **couldn't get** an answer | Authoritative servers down/misconfigured, DNSSEC validation failed |
| **REFUSED** | The server won't answer you | Asking a server that doesn't serve you |
| **Timeout** | No reply | Resolver unreachable, firewall blocking port 53 |

## 6.2 How apps report them

The same failures, in the words apps use:

| What happened | glibc / Python | Node.js |
|---|---|---|
| NXDOMAIN (name doesn't exist) | `[Errno -2] Name or service not known` | `ENOTFOUND` |
| Name exists, no address | `[Errno -5] No address associated with hostname` | `ENOTFOUND` / `ENODATA` |
| SERVFAIL or timeout (temporary) | `[Errno -3] Temporary failure in name resolution` | `EAI_AGAIN` |

The key split:
- **"Not known" / `ENOTFOUND`** = a definite "no". Check the **spelling, the record, the search domain**.
- **"Temporary failure" / `EAI_AGAIN`** = the lookup **couldn't complete**. Check whether the **resolver is reachable** and working.

Neither is a connection error: the app never got as far as connecting (Note 15, Phase 11).

---

# Phase 7 — How Linux resolves a name: the local chain

This is the most Linux-specific part — and where most "DNS works but my app fails" puzzles come from.

## 7.1 The full chain on Ubuntu

```
app (node, python, curl, psql…)
  │  getaddrinfo("db-02")       ← standard C library function (glibc)
  ▼
NSS — /etc/nsswitch.conf        "hosts: files dns"  → try these sources, in order
  │
  ├─ files → /etc/hosts         ← a local, hand-written list. Checked FIRST
  │
  └─ dns → /etc/resolv.conf     ← "nameserver 127.0.0.53"
              │
              ▼
         systemd-resolved (127.0.0.53)   ← local caching stub
              │  upstream DNS from DHCP (networkd, Note 15 Phase 4)
              ▼
         GCP resolver 169.254.169.254    ← recursive resolver
              │  internal names (*.internal, private zones) answered here
              │  everything else → the internet's DNS (Phase 4)
              ▼
         answer
```

## 7.2 `/etc/nsswitch.conf` — the order of sources

```bash
grep '^hosts' /etc/nsswitch.conf
# hosts:          files dns
```

`files dns` means: look in `/etc/hosts` first; if not found, use DNS.

## 7.3 `/etc/hosts` — the local override

```bash
cat /etc/hosts
```
```
127.0.0.1 localhost
10.0.0.4 app-server-01.asia-south1-a.c.team-project.internal app-server-01   # Added by Google
169.254.169.254 metadata.google.internal   # Added by Google
```

Because `files` comes first, **anything in `/etc/hosts` wins over DNS**. Useful for testing ("send this name to that IP, on this machine only"). Dangerous if forgotten — or if an attacker edits it (Part A "How this goes wrong").

## 7.4 `/etc/resolv.conf` — which DNS server to ask

```bash
ls -l /etc/resolv.conf
cat /etc/resolv.conf
```
```
lrwxrwxrwx 1 root root 39 ... /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf

nameserver 127.0.0.53
options edns0 trust-ad
search asia-south1-a.c.team-project.internal c.team-project.internal google.internal
```

| Line | Meaning |
|---|---|
| `nameserver 127.0.0.53` | Ask the local **systemd-resolved** stub (the listener you saw in Note 16, Step 0) |
| `search ...` | Domains to try for short names (Phase 8) |
| `options edns0 trust-ad` | Technical DNS options |

It's a **symlink** managed by systemd-resolved. **Don't edit it by hand** — it's regenerated. Change DNS settings through netplan/networkd or resolved's config instead.

## 7.5 systemd-resolved — the local stub

```bash
resolvectl status ens4
```
```
Link 2 (ens4)
    Current Scopes: DNS
         Protocols: +DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 169.254.169.254
       DNS Servers: 169.254.169.254
        DNS Domain: asia-south1-a.c.team-project.internal c.team-project.internal google.internal
```

**Meaning:** on this interface, resolved forwards questions to `169.254.169.254` (learned from DHCP), and applies those search domains.

```bash
resolvectl query db-02         # resolve through resolved, showing where the answer came from
```

**On our server:** this whole chain is why `app-server-01` can reach `db-02` by its short name — the next phases explain the "short name" part.

---

# Phase 8 — Search domains and short names

## 8.1 How a short name gets completed

When a name has **fewer dots** than a threshold (`ndots`, default **1**), the resolver tries adding each **search domain** in turn:

```
app asks for: db-02            (0 dots < ndots 1 → use search list first)
  try: db-02.asia-south1-a.c.team-project.internal   → NXDOMAIN
  try: db-02.c.team-project.internal                 → ✔ 10.0.1.5   (stop)
```

A name with enough dots is tried **as-is first**, then with search domains if that fails.

A **trailing dot** means "this is complete — don't add anything":

```bash
getent hosts db-02.c.team-project.internal.
```

## 8.2 Why it matters

| Situation | Effect |
|---|---|
| Short names in config (`db-02`) | Work only on machines with the right search domains. Move the app to another project/network/container → `NXDOMAIN` |
| Many search domains + many lookups | Each lookup of an **external** name may first try several wrong completions → extra DNS traffic, slower lookups |
| Kubernetes (`ndots: 5`) | Almost every external name tries 4–5 search domains first — a well-known source of slow DNS |

**Rule:** in application configuration, prefer **full names** (`db-02.c.team-project.internal` or a private-zone name like `db.internal.team.example`). Short names are for humans at a prompt.

---

# Phase 9 — DNS inside GCP

GCP gives every VM DNS without any setup:

| Feature | What it gives you |
|---|---|
| **Metadata server** `169.254.169.254` | The resolver every VM uses (from DHCP). Answers internal names and forwards everything else to public DNS |
| **Internal (zonal) DNS** | Every VM automatically gets `VM_NAME.ZONE.c.PROJECT_ID.internal` → its internal IP. Works from VMs in the same VPC network |
| **Cloud DNS private zones** | Your own names visible **only** inside chosen VPCs, e.g. `db.internal.team.example` → `10.0.1.5`. Clean names that don't depend on VM names or zones |
| Cloud DNS public zones | The authoritative servers for your public domain (Phase 2.2) |

**Good pattern for the team:** give services stable names in a **private zone** (`db.internal.team.example`, `images.internal.team.example`). When a server moves (Note 16, Step 3), change **one DNS record**, not every env file.

(Cloud-side details — zones, forwarding, peering — belong to the GCP networking area.)

---

# Phase 10 — Tools: which one follows which path

This table explains most "but `dig` says it works!" confusion:

| Tool | Uses `/etc/hosts`? | Uses search domains? | Talks to | Same result as your app? |
|---|---|---|---|---|
| `getent hosts NAME` / `getent ahosts NAME` | **Yes** | **Yes** | The full NSS chain (Phase 7.1) | **Yes** — the best "what does my app see?" test |
| `resolvectl query NAME` | No | Yes | systemd-resolved | Close (skips `/etc/hosts`) |
| `dig NAME` | **No** | **No** (by default) | The DNS server in `/etc/resolv.conf` directly | **Not always** |
| `dig @SERVER NAME` | No | No | A server you choose | No — for testing **that server** |
| `host NAME` / `nslookup NAME` | No | Yes | DNS directly | Not always |

## 10.1 `getent` — what the app sees

```bash
getent hosts db-02
# 10.0.1.5        db-02.c.team-project.internal
getent ahosts api.mailprovider.example     # all addresses, IPv4 and IPv6
```

## 10.2 `dig` — the DNS expert's tool

```bash
dig api.mailprovider.example
```
```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41022
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;api.mailprovider.example.      IN      A

;; ANSWER SECTION:
api.mailprovider.example. 247   IN      A       203.0.113.25

;; Query time: 1 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
```

| Part | Meaning |
|---|---|
| `status: NOERROR` | The answer type (Phase 6.1) |
| `flags: qr rd ra` | It's a response; recursion was requested and is available. `aa` would mean the answer came **directly** from an authoritative server |
| `ANSWER` + `247` | The record, with the **remaining** TTL in this cache (counts down from 300) |
| `Query time: 1 msec` | Fast = from a cache |
| `SERVER` | Who answered |

Useful forms:

```bash
dig +short api.mailprovider.example           # just the IPs
dig api.team.example AAAA                     # a specific record type
dig MX team.example                           # mail servers
dig @1.1.1.1 api.team.example                 # ask a public resolver (compare caches)
dig @ns1.dnsprovider.example api.team.example # ask the authoritative server directly — the "truth"
dig +trace api.team.example                   # do the whole root → TLD → authoritative walk yourself
dig -x 10.0.0.4                               # reverse lookup (PTR)
```

**Comparison technique:** if `dig @authoritative` shows the new IP but `dig` (via your resolver) shows the old one, it's **caching** (wait for TTL). If even the authoritative server shows the old one, the **record itself** wasn't changed.

---

# Phase 11 — DNS gotchas in real applications

| Gotcha | What happens | What to do |
|---|---|---|
| **Existing connections ignore DNS changes** | Connection pools (DB, HTTP keep-alive) keep using the old IP until connections close | Restart, or set a max connection lifetime |
| **Runtimes that cache** | The JVM can cache lookups (setting `networkaddress.cache.ttl`); some clients cache forever | Know your runtime's behaviour |
| **Node's lookups use a small thread pool** | `dns.lookup()` (used by `http`, `net`) runs `getaddrinfo` on libuv's 4-thread pool. Slow DNS can stall unrelated file operations | Keep DNS fast; reuse connections (Note 16, Step 5) |
| **IPv6 first** | A name with an AAAA record may be tried over IPv6 first; if IPv6 doesn't work on the path, connections wait for a timeout | Check `getent ahosts`; test with `curl -4` / `-6` |
| **`/etc/hosts` left from a test** | One machine silently goes to the wrong IP | `getent hosts NAME` vs `dig +short NAME` — different answers = check `/etc/hosts` |
| **Containers have their own resolver config** | Docker writes its own `/etc/resolv.conf` inside containers; search domains may differ | `docker exec NAME cat /etc/resolv.conf` |
| **DNS outage looks like "everything is down"** | Every new connection by name fails with `EAI_AGAIN` | Check resolver reachability first (`resolvectl status`, `dig`) |

> **How DNS goes wrong (security):**
> - **`/etc/hosts` tampering** — an attacker with root sends `api.mailprovider.example` to their own server. Watch changes to `/etc/hosts` and `/etc/resolv.conf`.
> - **DNS spoofing / cache poisoning** — fake answers inserted on the way. DNSSEC lets resolvers verify answers are signed by the zone owner.
> - **DNS rebinding** — a name first resolves to a public IP (passes your SSRF check, Note 16, Step 10), then to `169.254.169.254` or `127.0.0.1` when the app actually connects. **Resolve once, check the IP, then connect to that exact IP.**
> - **DNS as a data-theft channel** — malware sends data hidden inside lookups (`c2VjcmV0.attacker.example`). Very long or random-looking names in DNS logs are a warning sign.
> - **Dangling records / subdomain takeover** — a CNAME still points to a deleted cloud resource; someone else creates that resource and now serves content on **your** subdomain. Remove DNS records when you remove services.

---

# Part B — Host firewall

# Phase 12 — What a firewall does

## 12.1 The idea

A **firewall** looks at each packet and decides what happens to it, based on **rules**. A rule has two parts:

```
IF the packet matches these conditions   →   THEN do this
   (protocol, addresses, ports, interface,       (accept, drop, reject, log…)
    connection state, local user…)
```

The rules are checked **in order**. The **first rule whose action is final** (accept, drop, reject) decides. If **no** rule matches, the chain's **default policy** decides.

## 12.2 The three main actions

| Action | What happens to the packet | What the sender experiences |
|---|---|---|
| **accept** | Let it through | Normal |
| **drop** | Silently thrown away | **Nothing** — waits, then **times out** |
| **reject** | Thrown away **and** an error is sent back | **Immediate** failure: "refused" or "no route to host" |

Phase 20 connects these to the errors in Note 15, Phase 11.

## 12.3 Two basic styles

| Style | Default | Rules list | Used for |
|---|---|---|---|
| **Default deny** (allowlist) | Drop everything | What's **allowed** | **Incoming** traffic on servers — the safe choice |
| **Default allow** (blocklist) | Accept everything | What's **blocked** | Outgoing traffic, in most setups |

A default-deny firewall fails **safe**: a service you forgot to mention isn't exposed.

---

# Phase 13 — Stateful firewalls and connection tracking

## 13.1 The problem with simple rules

Our server connects **out** to the database: `10.0.0.4:42118 → 10.0.1.5:5432`. The reply comes **back** to `10.0.0.4` on port **42118** — an ephemeral port picked at random (Note 15, Phase 8.3).

A firewall that only looks at single packets ("stateless") would need to allow incoming traffic on **all** ephemeral ports to let replies in. That's thousands of open ports.

## 13.2 Connection tracking

Linux's firewall is **stateful**: the kernel keeps a **connection tracking table** (conntrack) of every connection it has seen. Each packet is labelled with a **state**:

| State | Meaning |
|---|---|
| `new` | The first packet of a connection (e.g. a TCP SYN) |
| `established` | Part of a connection that has seen traffic **both ways** |
| `related` | A new connection linked to an existing one (e.g. an ICMP error about it) |
| `invalid` | Doesn't fit any known connection — broken or suspicious |

This allows one simple rule near the top of every ruleset:

```
accept packets that are "established" or "related"
```

Replies to connections **we** started, and all later packets of connections we **allowed**, pass automatically. Every other rule only has to decide about **new** connections.

```bash
sudo conntrack -L -p tcp --dport 5432 | head -2    # (package: conntrack)
# tcp  6 431998 ESTABLISHED src=10.0.0.4 dst=10.0.1.5 sport=42118 dport=5432 src=10.0.1.5 dst=10.0.0.4 sport=5432 dport=42118 [ASSURED] ...
```

Each entry stores both directions — that's how the reply is recognised.

---

# Phase 14 — Where filtering happens: cloud vs host vs app

On our GCP server, a packet passes **three** filters:

```
internet ──► [1] VPC firewall rules ──► VM ──► [2] host firewall ──► [3] socket bind address ──► app
              (Google's network)              (Linux kernel)          (Note 15, Phase 9.3)
```

| Layer | Sees | Can match on | Can't |
|---|---|---|---|
| **1. Cloud firewall** (VPC rules) | Traffic **before** it reaches the VM | IPs, ports, protocols, network tags, service accounts | See **which process or user** on the VM; protect traffic **between** processes on the same VM |
| **2. Host firewall** (nftables/ufw) | Every packet in and out of **this VM's kernel** | All of the above **plus** interface (`docker0`, `lo`), connection state, and for **outgoing** traffic, the **local user** that sent it | Protect anything if a root attacker turns it off |
| **3. Bind address** | Whether a socket exists for that address | `127.0.0.1` vs all addresses | Distinguish between remote clients |

## 14.1 Why have a host firewall if the cloud already filters?

**Defence in depth** — each layer covers the others' gaps:

| Gap | Covered by the host firewall |
|---|---|
| The VPC allows **all traffic inside the subnet** (Note 16 setup). One compromised VM can reach every port on every other VM — like the open debugger in Note 16, Step 10 | Allow only the ports this VM really serves, even to internal neighbours |
| A VPC rule is changed by mistake (someone opens `0.0.0.0/0` on all ports for a test) | The host still blocks |
| You want to block **outgoing** traffic per user (the `api` user → metadata server, Note 16, Step 10) | Only the host can see local users |
| Containers and their published ports (Phase 21) | Rules on `docker0` and Docker's chains |

The cloud firewall stays the **first** line; the host firewall is the **second**.

---

# Phase 15 — Netfilter: the packet's path through the kernel

## 15.1 Hooks

Inside the kernel, the firewall system is called **Netfilter**. It has **hooks** — fixed points on a packet's path where rules can run:

```
                         incoming packet
                               │
                         ┌─────▼──────┐
                         │ PREROUTING │   ← destination NAT happens here (port forwarding)
                         └─────┬──────┘
                       routing decision:
                     is it for THIS machine?
                   ┌───────────┴────────────┐
                 yes                        no (and forwarding on)
                   │                        │
             ┌─────▼─────┐            ┌─────▼─────┐
             │   INPUT   │            │  FORWARD  │   ← containers' traffic passes here
             └─────┬─────┘            └─────┬─────┘
                   │                        │
             local process                  │
             (node, nginx…)                 │
                   │                        │
             ┌─────▼─────┐                  │
             │  OUTPUT   │  ← packets the   │
             └─────┬─────┘    server sends  │
                   │                        │
                   └──────────┬─────────────┘
                        ┌─────▼──────┐
                        │ POSTROUTING│   ← source NAT / masquerade happens here
                        └─────┬──────┘
                              ▼
                        outgoing packet
```

| Hook | Packets that pass | Typical use |
|---|---|---|
| `prerouting` | Every incoming packet | Destination NAT (port forwarding) |
| `input` | Packets **for this machine** | **Protecting the server's services** — the main one |
| `forward` | Packets **passing through** (routed) | Containers, VPN gateways, routers |
| `output` | Packets **created by this machine** | Restricting outgoing traffic (per user) |
| `postrouting` | Every outgoing packet | Source NAT (containers reaching the internet) |

## 15.2 Which hook for which question

| Question | Hook |
|---|---|
| "Who may connect **to** my API?" | `input` |
| "Where may my API connect **out to**?" | `output` |
| "What may containers reach / who may reach containers?" | `forward` (+ NAT) |

**On our server:** an internet request to the API (port 80) passes `prerouting → input → node`. The API's reply passes `output → postrouting`. A worker container calling the internet passes `prerouting → forward → postrouting` (masqueraded, Note 15, Phase 7.2).

---

# Phase 16 — The tools: nftables, iptables, ufw — and who's in charge

Several tools write rules into Netfilter. It helps to know how they relate:

| Tool | What it is | Status |
|---|---|---|
| **nftables** (`nft`) | The modern rule system and command | **Current standard** |
| iptables (legacy) | The older rule system | Being replaced |
| **iptables-nft** | The `iptables` **command**, but writing **nftables** rules underneath | Ubuntu's default `iptables` |
| **ufw** | "Uncomplicated Firewall" — simple commands that generate rules | Ubuntu's friendly front-end |
| firewalld | Front-end on RHEL/Fedora | Not default on Ubuntu |
| **Docker** | Writes its own rules (via iptables) when it starts | Always present if Docker runs |

## 16.1 One manager, and one way to see everything

Rules from different tools all end up in the kernel together. Problems start when two tools **both** think they own the firewall.

- Pick **one** tool to manage your rules: **ufw** (simple servers) **or** **nftables directly** (more control).
- Let Docker manage **its** rules, and add yours **alongside** (Phase 21).
- Whatever wrote them, this shows the **real** rules in the kernel:

```bash
sudo nft list ruleset
```

---

# Phase 17 — nftables from the ground up

## 17.1 The building blocks

```
table   ← a container, for one address family
 └─ chain   ← a list of rules; a "base chain" is attached to a hook
     └─ rule   ← match conditions + action
```

| Piece | Meaning |
|---|---|
| **Table** | Holds chains. Has a **family**: `ip` (IPv4), `ip6`, **`inet` (both — use this)**, `bridge`, `netdev` |
| **Base chain** | Attached to a **hook** (Phase 15) with a **type** (`filter` or `nat`), a **priority** (order among chains on the same hook), and a **policy** (`accept` or `drop`) for packets no rule decided |
| **Regular chain** | Not attached to a hook; reached with `jump` — for organising rules |
| **Rule** | Conditions + action. Checked top to bottom |
| **Set** | A named list (IPs, ports) a rule can match against in one step |

## 17.2 A minimal example, line by line

```
table inet myfw {
    chain input {
        type filter hook input priority filter; policy drop;

        ct state established,related accept
        iifname "lo" accept
        tcp dport 22 accept
    }
}
```

| Line | Meaning |
|---|---|
| `table inet myfw` | A table named `myfw` for IPv4 **and** IPv6 |
| `chain input { type filter hook input ...` | A base chain on the **input** hook — packets for this machine |
| `priority filter` | Standard priority for filtering (0) |
| `policy drop` | Anything not accepted below is **dropped** (default deny) |
| `ct state established,related accept` | Replies and ongoing connections pass (Phase 13) |
| `iifname "lo" accept` | Traffic on loopback (the machine talking to itself) passes |
| `tcp dport 22 accept` | New connections to port 22 pass |

## 17.3 Common match conditions

| Condition | Example |
|---|---|
| Protocol + port | `tcp dport 443`, `udp dport 53`, `tcp dport { 80, 443 }` |
| Source / destination address | `ip saddr 172.16.5.0/24`, `ip daddr 169.254.169.254`, `ip6 saddr ::1` |
| Interface in / out | `iifname "ens4"`, `oifname "docker0"` |
| Connection state | `ct state new`, `ct state invalid` |
| Local user (**output** only) | `meta skuid "api"`, `meta skuid 901` |
| ICMP | `ip protocol icmp`, `icmp type echo-request`, `meta l4proto ipv6-icmp` |
| Set | `ip saddr @office_vpn` |

## 17.4 Actions

| Action | Effect |
|---|---|
| `accept` | Allow (final for this chain) |
| `drop` | Silently discard (final) |
| `reject` | Discard and send an error (final; Phase 20) |
| `counter` | Count packets/bytes (continues) |
| `log prefix "..."` | Write to the kernel log (continues) |
| `jump CHAIN` / `return` | Go to another chain / come back |

Several can be combined in one rule, and they run **left to right**: `counter log prefix "drop: " drop` counts, logs, then drops. A `limit rate 5/minute` in a rule stops the rule for packets over the rate, so put it **just before** the statement you want limited (usually `log`).

## 17.5 Working with nft

```bash
sudo nft list ruleset                     # everything
sudo nft list table inet myfw             # one table
sudo nft -a list chain inet myfw input    # with rule handles (for deleting one rule)
sudo nft add rule inet myfw input tcp dport 443 accept      # add at the end (runtime only)
sudo nft delete rule inet myfw input handle 12             # delete by handle
sudo nft -c -f /etc/nftables.conf         # CHECK a file without applying it
sudo nft -f /etc/nftables.conf            # apply a file — atomically (all or nothing)
```

Rules added with `nft add` are **runtime only**, like `ip addr add` (Note 15). The saved configuration is a file, loaded at boot by `nftables.service`:

```bash
sudo systemctl enable --now nftables
```

---

# Phase 18 — A complete ruleset for our server

Applying Phases 12–17 to `app-server-01`. This also closes two items Note 16 left open: containers reaching the host API (Step 9) and blocking the `api` user from the metadata server (Step 10).

`/etc/nftables.d/myfw.nft` (included from `/etc/nftables.conf`):

```
# Our own table only. We do NOT "flush ruleset" — that would also delete Docker's rules (Phase 21).
table inet myfw
delete table inet myfw

table inet myfw {

    set office_vpn {
        type ipv4_addr; flags interval
        elements = { 172.16.5.0/24 }
    }

    chain input {
        type filter hook input priority filter; policy drop;

        # 1. Connections already allowed, and replies to ours
        ct state established,related accept
        ct state invalid drop

        # 2. The machine talking to itself
        iifname "lo" accept

        # 3. Ping and the ICMP messages TCP needs (e.g. "packet too big", Note 15 Phase 13.3)
        meta l4proto { icmp, ipv6-icmp } accept

        # 4. SSH: office VPN only (via either interface)
        tcp dport 22 ip saddr @office_vpn accept

        # 5. Public web traffic
        tcp dport { 80, 443 } accept

        # 6. Containers calling the host API (Note 16, Step 9)
        iifname "docker0" tcp dport 80 accept

        # 7. Everything else: count it, log a few (rate-limited), then the policy drops it
        counter
        limit rate 5/minute log prefix "myfw-in-drop: "
    }

    chain output {
        type filter hook output priority filter; policy accept;

        # Note 16, Step 10: the API user may not reach the metadata server
        meta skuid "api" ip daddr 169.254.169.254 counter log prefix "myfw-meta-block: " reject
    }
}
```

Line by line, the important decisions:

| Rule | Why |
|---|---|
| `policy drop` on input | Default deny: only listed services are reachable — even from the internal subnet |
| `ct state established,related accept` first | Replies and existing connections never need special rules (Phase 13) |
| `ct state invalid drop` | Junk packets that fit no connection |
| ICMP accepted | Breaking ICMP breaks MTU discovery and makes debugging harder |
| SSH only from `@office_vpn` | Matches the VPC rule — two layers saying the same thing |
| `docker0` → port 80 | Containers reach the API via `host.docker.internal` |
| `counter`, then rate-limited `log`, then the `drop` policy | Count every dropped packet, but log only a few per minute — visibility without filling the journal (Note 14, Step 3). The order matters: `limit` before `log`, so only the logging is limited |
| Output `policy accept` | Outgoing is allowed by default… |
| …except `api` → `169.254.169.254` | Only the host firewall can match a **local user**. `reject` so the app fails fast with a clear error |

Load safely (Phase 22), then check:

```bash
sudo nft -c -f /etc/nftables.d/myfw.nft && sudo nft -f /etc/nftables.d/myfw.nft
sudo -u api curl -s -m 3 -H "Metadata-Flavor: Google" http://169.254.169.254/ ; echo "exit $?"
# exit 7    ← connection refused: blocked by our reject
```

Note on the metadata block: the VM's **own** tools (the guest agent, `gcloud`, the Ops Agent from Note 14) run as root and still need the metadata server. That's why the rule matches **only** the `api` user.

---

# Phase 19 — ufw: the simple front-end

**ufw** is the easiest way to run a default-deny host firewall on Ubuntu. It writes Netfilter rules for you.

## 19.1 Basic use

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 172.16.5.0/24 to any port 22 proto tcp     # SSH from office VPN
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow in on docker0 to any port 80 proto tcp
sudo ufw enable
sudo ufw status verbose
sudo ufw status numbered            # with numbers, for deleting
sudo ufw delete 3                   # delete rule 3
sudo ufw logging low                # log blocked packets
```

## 19.2 ufw or nftables?

| | ufw | nftables directly |
|---|---|---|
| Learning curve | Very low | Moderate |
| Typical inbound rules | Easy | Easy |
| Per-user outgoing rules (Phase 18's metadata block) | Only by editing its raw rule files (`/etc/ufw/before.rules`) | Built in (`meta skuid`) |
| Sets, counters, complex logic | Limited | Full |
| Seeing what it really did | `sudo nft list ruleset` | Same |

A sensible rule of thumb: **ufw** for simple servers; **nftables** when you need per-user, per-interface, or container-aware rules — like this server. **Don't run both** as managers.

---

# Phase 20 — Drop vs reject: what the client sees

The firewall's choice directly decides which error the client gets — the errors from Note 15, Phase 11:

| Firewall action | Client sees (TCP) | Time to fail |
|---|---|---|
| `drop` | **Connection timed out** | Seconds to minutes (retries) |
| `reject with tcp reset` | **Connection refused** | Instant |
| `reject` (default ICMP port-unreachable) | **Connection refused** | Instant |
| `reject with icmpx admin-prohibited` | **No route to host** | Instant |

So **"No route to host"** often isn't a routing problem at all — it's a firewall **rejecting** (RHEL's firewalld does this by default).

| Use | When |
|---|---|
| **drop** | Traffic from the **internet** — gives scanners nothing, costs nothing |
| **reject** | Traffic from **your own** systems and users — fast, clear failures save debugging time (Phase 18's metadata block) |

---

# Phase 21 — Docker and the host firewall

## 21.1 What Docker adds

When Docker starts, it creates its own rules:

| Rules | Purpose |
|---|---|
| `nat` table, `DOCKER` chain (prerouting) | **Published ports**: `-p 8080:80` → traffic to host port 8080 is redirected to the container |
| `filter` table, `FORWARD` → `DOCKER-USER`, `DOCKER` chains | Allow traffic to/from containers |
| `nat` `POSTROUTING` masquerade | Containers reach the internet as the host (Note 15, Phase 7.2) |

## 21.2 The classic surprise: published ports bypass your input rules

```bash
docker run -d -p 8080:80 nginx
sudo ufw status              # 8080 is NOT allowed
curl http://10.0.0.4:8080    # from app-server-02 → works anyway!
```

**Why:** traffic to a published port is redirected to the container in **prerouting**. After that, it's no longer "for this machine" — it goes through **forward**, not **input** (Phase 15). Your input rules (and ufw's) never see it.

## 21.3 Fixes

| Fix | How |
|---|---|
| **Publish only on loopback** when the port is for local use | `-p 127.0.0.1:8080:80` — reachable only from the host (and via SSH tunnel, Note 19) |
| Don't publish at all | Containers on a shared Docker network reach each other by name, without published ports |
| Filter in **`DOCKER-USER`** | Docker's documented place for your rules about container traffic; it's checked **before** Docker's own |
| Keep the cloud firewall strict | The VPC rule is still the first line |

Example `DOCKER-USER` rule (iptables syntax, since Docker uses it): allow published ports only from the office VPN:

```bash
sudo iptables -I DOCKER-USER -i ens4 ! -s 172.16.5.0/24 -m conntrack --ctstate NEW -j DROP
```

## 21.4 Don't wipe Docker's rules

A common `nftables.conf` starts with `flush ruleset`. Reloading it **while Docker runs** deletes Docker's rules too — containers lose networking until Docker restarts. That's why Phase 18 manages **only its own table** (`delete table inet myfw`, then recreate it).

---

# Phase 22 — Changing firewall rules without locking yourself out

A firewall mistake on a remote server can cut off **your own SSH session**. Habits that prevent it:

| Habit | Why |
|---|---|
| **Keep your current SSH session open** while testing | Established connections usually survive (rule `ct state established accept`) — you keep a way in |
| **Allow SSH first**, before setting `policy drop` or `ufw enable` | Order matters |
| **Check before applying**: `nft -c -f FILE` | Catches syntax errors |
| **Apply files, not single commands**: `nft -f FILE` | Atomic — all rules change at once, never half-applied |
| **Schedule an automatic undo** | If you get locked out, the server reverts itself |
| Test from a **second** session before closing the first | Proves new connections work |
| Know your **escape hatch** | On GCP: the **serial console** works even when the network is blocked |

An automatic undo with a transient timer (Note 13, Phase 7.4):

```bash
sudo systemd-run --on-active=5min --unit=fw-undo /usr/sbin/nft delete table inet myfw
sudo nft -f /etc/nftables.d/myfw.nft
# test from a NEW session... if everything works:
sudo systemctl stop fw-undo.timer
```

If the new rules lock you out, `myfw` is deleted after 5 minutes and access returns. (With ufw: `sudo systemd-run --on-active=5min ufw disable`.)

---

# Phase 23 — Logging, counters, and the connection table

## 23.1 Counters: which rule is being hit?

```bash
sudo nft list chain inet myfw input
```
```
        tcp dport { 80, 443 } accept
        ...
        counter packets 18211 bytes 1092660
        limit rate 5/minute log prefix "myfw-in-drop: "
```

The `counter` just before the end of the chain counts everything the policy is about to drop. If it rises while a connection fails, **your firewall** is the one dropping it. (Add `counter` to any rule — `tcp dport 22 ip saddr @office_vpn counter accept` — to see whether it's being matched.)

## 23.2 Logs

```bash
sudo journalctl -k -g 'myfw-in-drop' --since "10 min ago"
```
```
kernel: myfw-in-drop: IN=ens4 OUT= MAC=... SRC=10.0.0.6 DST=10.0.0.4 LEN=60 ... PROTO=TCP SPT=51220 DPT=3001 SYN
```

**Meaning:** `app-server-02` tried a new connection (`SYN`) to port 3001, and the host firewall dropped it. `IN=ens4` shows which interface it came in on.

Always **rate-limit** log rules (`limit rate 5/minute`) — an attack or a misbehaving client can otherwise fill the journal.

## 23.3 The connection tracking table

```bash
sudo conntrack -C                                  # number of tracked connections
sysctl net.netfilter.nf_conntrack_max              # the table's maximum
sudo conntrack -L -p tcp --dport 80 | head         # tracked connections to port 80
```

If the table fills, the kernel **drops new connections** and logs:

```
nf_conntrack: nf_conntrack: table full, dropping packet
```

Symptoms look like random connection timeouts under heavy load. Fix by raising `nf_conntrack_max` (with enough memory) — after measuring (Note 15, Phase 14).

---

# How this goes wrong — overview

| Issue | Why it matters | How to check |
|---|---|---|
| No host firewall ("the VPC handles it") | One compromised VM reaches every port on its neighbours | `sudo nft list ruleset` — is there an input `policy drop`? |
| Input `policy accept` with a few blocks | Blocklists miss new services | Review the policy |
| Docker published ports on all addresses | Bypass ufw/input rules (Phase 21) | `docker ps` PORTS column, `ss -tlnp` |
| Firewall disabled "for testing" and forgotten | Back to no host protection | `ufw status`, `systemctl is-active nftables` |
| Two managers fighting | Rules silently replaced on reload | Who loads rules at boot? |
| `flush ruleset` with Docker running | Containers lose networking | Check `/etc/nftables.conf` |
| Unlimited log rules | Journal flood (Note 14, Step 3) | `limit rate` on every log rule |
| ICMP fully blocked | MTU problems, harder debugging | Allow essential ICMP |
| Outgoing unrestricted for app users | Stolen-token / data-theft paths (metadata, arbitrary hosts) | Output rules per user for sensitive destinations |
| `/etc/hosts`, `/etc/resolv.conf` changed | Traffic silently redirected | File integrity checks; `getent` vs `dig` |
| DNS rebinding in URL-fetch features | SSRF check bypassed | Resolve once, check, connect to that IP |
| Dangling DNS records | Subdomain takeover | Review records when deleting services |

---

# Summary table

| Concept | Key point |
|---|---|
| DNS | Turns names into IPs (and more); lets IPs change behind stable names |
| Name tree | Read right to left: root → TLD → domain → host |
| Zone / authoritative server | Who holds the official records for part of the tree |
| A / AAAA / CNAME | IPv4 / IPv6 / alias to another name |
| MX / TXT / NS / SOA / PTR | Mail / text (SPF etc.) / delegation / zone info / reverse |
| Stub vs recursive resolver | Asks for the final answer vs does the root→TLD→authoritative work |
| DNS ports | UDP **and** TCP 53 |
| TTL | How long answers are cached; "propagation" = caches expiring |
| Negative caching | "Doesn't exist" is cached too |
| NXDOMAIN vs SERVFAIL | Name doesn't exist vs resolver couldn't answer |
| App errors | "Not known"/`ENOTFOUND` = no such name; "Temporary failure"/`EAI_AGAIN` = resolver problem |
| Linux chain | `getaddrinfo` → nsswitch → `/etc/hosts` → `/etc/resolv.conf` → resolved (127.0.0.53) → upstream |
| `/etc/hosts` | Checked first; overrides DNS on this machine |
| `/etc/resolv.conf` | Symlink managed by resolved — don't edit |
| Search domains / ndots | Short names get completed; use full names in configs |
| GCP DNS | Metadata server resolver; `VM.ZONE.c.PROJECT.internal`; private zones |
| `getent hosts` | What your app sees |
| `dig` | Raw DNS; skips `/etc/hosts`; `@server`, `+trace`, `+short` |
| Firewall rule | Match → action; first final action wins; else policy |
| accept / drop / reject | Allow / silent (timeout) / error back (refused or no route) |
| Default deny | Safest for incoming |
| Stateful / conntrack | `established,related accept` handles replies |
| Cloud vs host vs bind | Three filters; host adds interfaces, state, local user |
| Netfilter hooks | prerouting, input, forward, output, postrouting |
| Tools | nftables (modern), iptables-nft, ufw (simple), Docker (own rules) |
| nftables pieces | table (inet) → base chain (hook, priority, policy) → rules, sets |
| `nft -c -f` / `nft -f` | Check / atomic apply |
| Docker published ports | Go through forward, not input — bypass input/ufw rules |
| Lockout safety | Keep session, allow SSH first, timed undo, serial console |
| Counters / logs | Prove which rule acts; rate-limit logging |
| conntrack full | New connections dropped under load |

---

# Appendix — Command reference

## DNS

| Command | Purpose |
|---|---|
| `getent hosts NAME` / `getent ahosts NAME` | Resolve like an app (incl. `/etc/hosts`) |
| `resolvectl status [DEV]` | Resolver per interface, upstream servers, search domains |
| `resolvectl query NAME` | Resolve through systemd-resolved |
| `resolvectl statistics` / `sudo resolvectl flush-caches` | Cache stats / clear local cache |
| `dig NAME [TYPE]` / `dig +short NAME` | DNS query / just the answer |
| `dig @SERVER NAME` | Ask a specific server |
| `dig +trace NAME` | Follow root → TLD → authoritative |
| `dig -x IP` | Reverse lookup |
| `cat /etc/resolv.conf` / `cat /etc/hosts` / `grep hosts /etc/nsswitch.conf` | Local configuration |
| `docker exec NAME cat /etc/resolv.conf` | Container's resolver config |

## nftables

| Command | Purpose |
|---|---|
| `sudo nft list ruleset` | Everything in the kernel (any tool) |
| `sudo nft list table inet T` / `list chain inet T C` | One table / chain |
| `sudo nft -a list chain inet T C` | With rule handles |
| `sudo nft add rule inet T C ...` | Add (runtime) |
| `sudo nft delete rule inet T C handle N` | Delete one rule |
| `sudo nft -c -f FILE` / `sudo nft -f FILE` | Check / apply atomically |
| `sudo systemctl enable --now nftables` | Load `/etc/nftables.conf` at boot |

## ufw

| Command | Purpose |
|---|---|
| `sudo ufw default deny incoming` / `allow outgoing` | Defaults |
| `sudo ufw allow PORT/tcp` / `allow from CIDR to any port N proto tcp` | Rules |
| `sudo ufw allow in on IFACE to any port N proto tcp` | Per interface |
| `sudo ufw enable` / `disable` / `status verbose` / `status numbered` | Control / view |
| `sudo ufw delete N` / `sudo ufw logging low` | Delete rule / log blocks |

## Docker and connection tracking

| Command | Purpose |
|---|---|
| `docker run -p 127.0.0.1:HOST:CONT ...` | Publish on loopback only |
| `sudo iptables -L DOCKER-USER -n -v` | Your rules for container traffic |
| `sudo conntrack -L [-p tcp --dport N]` / `conntrack -C` | Tracked connections / count |
| `sysctl net.netfilter.nf_conntrack_max` | Table size |

## Safety and logs

| Command | Purpose |
|---|---|
| `sudo systemd-run --on-active=5min --unit=fw-undo CMD` | Automatic undo |
| `sudo systemctl stop fw-undo.timer` | Cancel the undo |
| `sudo journalctl -k -g PREFIX` | Firewall log lines |

## Key nftables syntax

| Syntax | Meaning |
|---|---|
| `table inet NAME { ... }` | IPv4 + IPv6 table |
| `type filter hook input priority filter; policy drop;` | Base chain on input, default deny |
| `ct state established,related accept` | Allow replies |
| `iifname "lo" accept` | Loopback |
| `tcp dport { 80, 443 } accept` | Ports |
| `ip saddr @SET accept` | Source in a set |
| `meta skuid "USER"` | Local user (output) |
| `counter` + `limit rate 5/minute log prefix "X: "` (before a `drop` policy) | Count all, log a few, drop |
| `reject` / `reject with tcp reset` / `reject with icmpx admin-prohibited` | Refused / refused / no route to host |