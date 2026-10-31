# SSH & Network Debugging — Theory (Note 19)

Two final topics for the networking area, explained from the ground up:

- **SSH** — how secure remote access actually works: keys, the connection steps, client configuration, jump hosts, and **tunnels** that let you reach private services safely.
- **Network debugging** — a repeatable method and the tools behind it (`curl`, `nc`, `openssl`, `traceroute`/`mtr`, `tcpdump`), so that "it can't connect" becomes a short, ordered investigation instead of guessing.

**Series:**
- Notes 1–14: Shell, Environment, Users, Files, Privilege, Processes, systemd, Logs/Timers/Limits
- Note 15 / 16: Interfaces, Routing & Sockets — Theory / Practice
- Note 17: DNS & Host Firewall — Theory
- **Note 19: SSH & Network Debugging — Theory** ← this note

Each phase has an **"On our server"** example from the team server (`app-server-01`, Notes 4–17), and, where relevant, a **"How this goes wrong"** box for the security track. This note also closes items left open in Note 16: reaching the admin dashboard (Step 1) and the Node debugger (Step 10) through SSH tunnels.

---

## Contents

**Part A — SSH**
1. Phase 1 — What SSH is and what it replaced
2. Phase 2 — Just enough cryptography
3. Phase 3 — What happens when you connect
4. Phase 4 — Host keys: is this really my server?
5. Phase 5 — User keys: proving who you are
6. Phase 6 — `ssh-agent` and agent forwarding
7. Phase 7 — The client config file: `~/.ssh/config`
8. Phase 8 — Jump hosts, bastions, and GCP's alternatives
9. Phase 9 — Tunnels: local, remote, and dynamic forwarding
10. Phase 10 — The server side: `sshd_config`
11. Phase 11 — Copying files over SSH
12. Phase 12 — When SSH fails: reading the errors

**Part B — Network debugging**
13. Phase 13 — The method
14. Phase 14 — The toolbox at a glance
15. Phase 15 — `curl` as a debugging tool
16. Phase 16 — `nc` and `openssl s_client`: testing ports and TLS
17. Phase 17 — Path tools: `traceroute`, `tracepath`, `mtr`
18. Phase 18 — `tcpdump`: watching the packets themselves
19. Phase 19 — Seeing what the firewalls are doing
20. Phase 20 — Putting it together: three decision flows

- How this goes wrong — overview
- Summary table
- Appendix — Command reference

---

# Part A — SSH

# Phase 1 — What SSH is and what it replaced

## 1.1 The problem SSH solves

You need to run commands on a server far away, over networks you don't control (the internet, a café's Wi-Fi). Anyone on the path could **read** your traffic or **change** it.

The old tools — `telnet`, `rsh`, `ftp` — sent everything, **including passwords**, as plain text. Anyone capturing packets (Phase 18 shows how easy that is) could read it.

**SSH (Secure Shell)** gives you:

| Property | Meaning |
|---|---|
| **Encryption** | Nobody on the path can read the traffic |
| **Integrity** | Nobody can change it without being detected |
| **Server authentication** | You know you're talking to the **real** server, not an impostor |
| **User authentication** | The server knows **who you are** — ideally with keys, not passwords |

## 1.2 What you can do with it

| Use | Example |
|---|---|
| Remote shell | `ssh thirana@app-server-01` |
| Run one command | `ssh app-server-01 'systemctl status api'` |
| Copy files | `scp`, `sftp`, `rsync` (Phase 11) |
| Tunnels | Reach a private service through the server (Phase 9) |
| Jump through a bastion | Reach servers with no direct access (Phase 8) |

## 1.3 Client and server

```
your laptop                                   app-server-01
ssh (client) ──── TCP, port 22 ────────────►  sshd (server, Note 16: socket-activated)
```

Everything in Part A is about this one encrypted connection: how it's set up, and what you can carry inside it.

---

# Phase 2 — Just enough cryptography

You don't need the maths. You need four ideas.

## 2.1 Symmetric encryption — one shared key

The **same** key locks and unlocks. Fast — used for **all the actual data** in an SSH session.

Problem: how do two machines that have never met agree on a secret key, over a network someone might be watching? → Phase 3 (key exchange).

## 2.2 Asymmetric keys — a key **pair**

Two keys made together:

| Key | Who has it | Used to |
|---|---|---|
| **Private key** | **Only the owner.** Never shared, never copied to servers | **Sign** things (prove "this came from me") |
| **Public key** | Anyone. Safe to share and paste into servers | **Check** a signature made by the matching private key |

The idea: a signature can only be **made** with the private key, but anyone with the public key can **check** it. So if the server has your public key, you can prove you hold the private key — **without ever sending it**.

## 2.3 Signatures

"Sign" = produce a small piece of data from a message and the private key. Change one bit of the message, or use another key, and the check fails.

## 2.4 Fingerprints

A **fingerprint** is a short hash of a public key — an easy-to-compare ID:

```
SHA256:Wq3n4dIhm1kq4b2LmFQd0qS3vS2+2kO0iY5Hf3W8tHk
```

Same key → same fingerprint. People compare fingerprints instead of long keys.

**Key types today:** `ed25519` — small, fast, modern; **the default choice**. `rsa` (at least 3072 bits) only for very old systems. Avoid `dsa`.

---

# Phase 3 — What happens when you connect

When you run `ssh thirana@app-server-01`, five things happen in order. Knowing them makes every SSH error readable (Phase 12).

```
 client                                              server (sshd)
   │ 1. TCP connection to port 22  ─────────────────►│      (Note 15: socket, handshake)
   │                                                 │
   │ 2. Version exchange  ◄──────────────────────────►│      "SSH-2.0-OpenSSH_9.6"
   │                                                 │
   │ 3. Key exchange  ◄──────────────────────────────►│      both sides compute the SAME session key
   │    (encryption starts here)                     │      without ever sending it
   │                                                 │
   │ 4. SERVER proves its identity  ◄────────────────│      signs with its HOST private key
   │    client checks it against known_hosts         │      (Phase 4)
   │                                                 │
   │ 5. USER proves their identity  ─────────────────►│      signs with the USER private key
   │    server checks it against authorized_keys     │      (Phase 5)
   │                                                 │
   │ 6. Channels: shell, command, file transfer,     │
   │    tunnels — many can share this one connection │
```

| Step | If it fails, you see… |
|---|---|
| 1. TCP | `Connection refused` / `Connection timed out` — a **network** problem (Note 15, Phase 11; Note 17 firewalls) |
| 3. Key exchange | `no matching key exchange method found` — very old client or server |
| 4. Server identity | `Host key verification failed` / `REMOTE HOST IDENTIFICATION HAS CHANGED` |
| 5. User identity | `Permission denied (publickey)` |

## 3.1 Key exchange in one paragraph

Both sides generate temporary random values and exchange **parts** of them. Using a mathematical method (Diffie-Hellman), each side combines its secret part with the other's public part and arrives at the **same** session key. Someone watching sees only the public parts and **can't** compute the key. New keys are made for every session, so recording traffic today doesn't help decrypt it later.

---

# Phase 4 — Host keys: is this really my server?

## 4.1 The server's key pair

Every SSH server has its own **host key pair**, created at installation:

```bash
ls /etc/ssh/ssh_host_*
# /etc/ssh/ssh_host_ed25519_key      ← private (root only)
# /etc/ssh/ssh_host_ed25519_key.pub  ← public
```

In step 4, the server **signs** with its host private key. The client checks the signature against the public key it **remembers** for that server.

## 4.2 `known_hosts` and the first connection

The client remembers host keys in `~/.ssh/known_hosts`. The first time, it has nothing to compare with:

```
The authenticity of host 'app-server-01 (10.10.0.4)' can't be established.
ED25519 key fingerprint is SHA256:Wq3n4dIhm1kq4b2LmFQd0qS3vS2+2kO0iY5Hf3W8tHk.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Saying `yes` saves the key — "trust on first use". The safe way is to **compare the fingerprint** with one you got through a trusted channel. On the server itself:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

On GCP, the guest agent also publishes the VM's host keys as guest attributes, which you can read through the API instead of trusting the network:

```bash
gcloud compute instances get-guest-attributes app-server-01 --zone asia-south1-a --query-path=hostkeys/
```

## 4.3 The warning that matters

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
```

The server presented a **different** host key than last time. Two explanations:

| Innocent | Dangerous |
|---|---|
| The VM was **recreated** (new host keys), or the IP now belongs to a different VM — very common in the cloud | Someone is **intercepting** the connection (man-in-the-middle) |

**Don't just delete the line and continue.** Confirm the change is expected (was the VM rebuilt?), check the new fingerprint, **then** remove the old entry:

```bash
ssh-keygen -R app-server-01      # remove the old key for that name
ssh-keygen -R 10.10.0.4          # and for the IP, if stored separately
```

> **How this goes wrong:** teams train themselves to ignore this warning (or set `StrictHostKeyChecking no` everywhere). That removes the protection SSH gives against interception.

---

# Phase 5 — User keys: proving who you are

## 5.1 Creating a key pair

On **your laptop**:

```bash
ssh-keygen -t ed25519 -C "thirana@laptop"
```
```
Enter file in which to save the key (/home/thirana/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
```

| File | What | Permissions (Note 6, Step 8) |
|---|---|---|
| `~/.ssh/id_ed25519` | **Private** key — stays on the laptop | `600` |
| `~/.ssh/id_ed25519.pub` | **Public** key — goes to servers | `644` |

**Passphrase:** encrypts the private key file. If the laptop is stolen, the key alone is useless. Use one; `ssh-agent` (Phase 6) means you type it once per session.

## 5.2 Installing the public key on the server

The public key is added as one line to `~/.ssh/authorized_keys` **on the server**, in the account you'll log in as (Note 4, Step 3):

```bash
ssh-copy-id thirana@app-server-01     # if password login is still possible
# or the admin installs it (Note 4, Step 3):
#   install -m 600 -o thirana -g thirana thirana.pub /home/thirana/.ssh/authorized_keys
```

## 5.3 What actually happens in step 5

```
client: "I want to log in as thirana, with this public key"
server: finds that key in /home/thirana/.ssh/authorized_keys? → yes
client: signs data from THIS session with the private key
server: checks the signature with the public key → valid → logged in
```

The private key **never leaves** the laptop. The signature is tied to this session, so it can't be replayed.

## 5.4 `authorized_keys` options

Each key line can carry **restrictions** in front of it (Note 4, Step 6):

```
restrict,from="203.0.113.10/32",command="/usr/local/bin/deploy.sh" ssh-ed25519 AAAA... ci-deploy
```

| Option | Effect |
|---|---|
| `restrict` | No port forwarding, agent forwarding, X11, or terminal |
| `from="CIDR"` | Key only works from these addresses |
| `command="..."` | Whatever the client asks, **only this command** runs |
| `port-forwarding` / `pty` | Re-enable one thing after `restrict` |

## 5.5 Other ways to authenticate

| Method | Notes |
|---|---|
| `publickey` | Keys (above) — **preferred** |
| `password` | Typed password — exposed to guessing; usually **disabled** on servers (Note 4, Step 6) |
| `keyboard-interactive` | Prompts, used for one-time codes / MFA |
| SSH **certificates** | A key signed by a company CA, with an expiry — no need to copy public keys to every server |
| Cloud identity (GCP OS Login) | Keys and permissions come from Google accounts and IAM (Note 3, Phase 10.2) |

---

# Phase 6 — `ssh-agent` and agent forwarding

## 6.1 The agent

`ssh-agent` is a small program on your laptop that holds your **decrypted** private keys in memory, so you type the passphrase once:

```bash
eval "$(ssh-agent -s)"     # start one (most desktops already run one)
ssh-add ~/.ssh/id_ed25519  # add a key (asks the passphrase once)
ssh-add -l                 # list loaded keys (fingerprints)
ssh-add -D                 # remove all
```

`ssh` asks the agent to **sign** for it. The key itself never leaves the agent.

## 6.2 Agent forwarding — and why to avoid it

`ssh -A server` makes your laptop's agent **reachable from the server**, so from the server you can SSH onwards using your laptop's keys.

The risk: while you're connected, **anyone with root on that server** can use the forwarded agent to log in **anywhere your key works**, as you. The key isn't stolen — but it can be **used**.

**Use ProxyJump instead** (Phase 8). It reaches the next server **through** the first without exposing your agent there.

---

# Phase 7 — The client config file: `~/.ssh/config`

Typing `ssh -i ~/.ssh/work_ed25519 -p 22 -J bastion thirana@10.10.0.4` every time is error-prone. Put it in `~/.ssh/config` on your laptop:

```
# --- Team servers -------------------------------------------------
Host bastion
    HostName 203.0.113.50
    User thirana

Host app-server-01
    HostName 10.10.0.4              # management interface (Note 16, Step 7)
    User thirana
    ProxyJump bastion               # Phase 8

Host app-server-02
    HostName 10.10.0.6
    User thirana
    ProxyJump bastion

# --- Defaults for everything (keep LAST) --------------------------
Host *
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes              # offer only this key (avoids "too many authentication failures")
    ServerAliveInterval 30          # keep idle sessions alive through NAT/firewalls
    AddKeysToAgent yes
    ControlMaster auto              # reuse one connection for several sessions
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

Now `ssh app-server-01` does it all.

| Option | Meaning |
|---|---|
| `Host` | Pattern you type; following lines apply to it |
| `HostName` | The real address |
| `User`, `Port`, `IdentityFile` | Defaults for this host |
| `IdentitiesOnly yes` | Use only the listed key, not every key in the agent |
| `ProxyJump` | Go through this host first |
| `LocalForward` | A tunnel opened automatically (Phase 9) |
| `ServerAliveInterval` | Send a keep-alive every N seconds |
| `ControlMaster`/`ControlPath`/`ControlPersist` | **Multiplexing**: later `ssh`/`scp` to the same host reuse the existing connection — much faster |

**How it's read:** for each option, the **first** value found wins. So specific `Host` blocks go **first**, `Host *` **last**.

Multiplexing note (Note 3, Phase 7.3): a reused connection is the **old** session — after being added to a new group, "log in again" may not really start a new login. `ssh -O exit app-server-01` closes the shared connection.

```bash
ssh -G app-server-01 | grep -E '^(hostname|user|proxyjump|identityfile) '   # what ssh will actually use
```

---

# Phase 8 — Jump hosts, bastions, and GCP's alternatives

## 8.1 Why a bastion

Servers with private addresses only (`10.10.0.4`) aren't reachable from the internet. A **bastion** (jump host) is one hardened server that **is** reachable, used only as a gateway:

```
laptop ──► bastion (public IP, SSH only, tightly locked down) ──► private servers
```

Benefits: one place to control and log access; app servers need no public SSH at all.

## 8.2 ProxyJump: how it works

```bash
ssh -J bastion app-server-01      # or ProxyJump in ~/.ssh/config
```

```
laptop ═══ SSH to bastion ═══► bastion
   ║                              │  bastion only opens a plain TCP
   ║                              │  connection onward and passes bytes
   ╚═══════ SSH to app-server-01 (inside the first connection) ═══► app-server-01
```

- The SSH session to `app-server-01` is **end-to-end** from your laptop — the bastion can't read it.
- Your keys and agent stay on the laptop.
- The bastion must allow TCP forwarding (`AllowTcpForwarding`) for this to work (Phase 10).

## 8.3 On GCP: IAP instead of a bastion

GCP's **Identity-Aware Proxy (IAP) TCP forwarding** does the bastion's job as a managed service:

```bash
gcloud compute ssh app-server-01 --zone asia-south1-a --tunnel-through-iap
```

- The VM needs **no public IP** and no bastion.
- Access is controlled by **IAM** (who has the IAP tunnel role) and is logged in Cloud Audit Logs.
- The VPC firewall must allow SSH from Google's IAP range (`35.235.240.0/20`).

Combined with **OS Login** (Note 3, Phase 10.2), who can log in — and who gets sudo — is decided by IAM rather than by keys copied to each VM. (Details belong to the GCP IAM and networking areas.)

---

# Phase 9 — Tunnels: local, remote, and dynamic forwarding

SSH can carry **other** TCP connections inside its encrypted connection. This is called **port forwarding** or **tunnelling**. It's how you reach a service that's deliberately **not** exposed — like the admin dashboard on `127.0.0.1:3001` (Note 16, Step 1).

## 9.1 Local forwarding (`-L`) — "bring a remote service to my laptop"

```bash
ssh -L 3001:127.0.0.1:3001 app-server-01
#      │    │         │
#      │    │         └─ destination port
#      │    └─ destination host — resolved AS SEEN FROM THE SERVER
#      └─ port to open on MY LAPTOP
```

```
laptop                                         app-server-01
browser → localhost:3001                       
   │                                              
   └─► ssh client ════ encrypted SSH ════► sshd ──► 127.0.0.1:3001 (the dashboard)
                                                    "127.0.0.1" here = the SERVER's loopback
```

Open `http://localhost:3001` on the laptop → you're using the dashboard, which never left loopback on the server.

**The one confusing part:** the destination (`127.0.0.1:3001`) is reached **from the server**. So `127.0.0.1` means the **server's** loopback, and an internal IP means "as the server can reach it".

**On our server — closing Note 16's open items:**

```bash
# Admin dashboard (Note 16, Step 1)
ssh -N -L 3001:127.0.0.1:3001 app-server-01

# Node debugger, now bound to loopback (Note 16, Step 10)
ssh -N -L 9229:127.0.0.1:9229 app-server-01      # then attach the debugger to localhost:9229

# The database, which only app servers can reach (Note 16, Step 3)
ssh -N -L 5433:10.0.1.5:5432 app-server-01
psql -h localhost -p 5433 -U readonly appdb
```

`-N` = don't open a shell, just hold the tunnel. Make the common ones permanent in `~/.ssh/config`:

```
Host app-server-01-tunnels
    HostName 10.10.0.4
    ProxyJump bastion
    LocalForward 3001 127.0.0.1:3001
    LocalForward 9229 127.0.0.1:9229
```

By default the laptop end listens on **`127.0.0.1`** only, so other people on your network can't use your tunnel. (Writing `-L 0.0.0.0:3001:...` would share it — usually a bad idea.)

## 9.2 Remote forwarding (`-R`) — "expose my laptop's service on the server"

The reverse direction: a port on the **server** leads back to something reachable from the **laptop**.

```bash
ssh -R 8081:127.0.0.1:3000 app-server-01
```

```
app-server-01: something connects to 127.0.0.1:8081
   └─► sshd ════ encrypted SSH ════► laptop ──► 127.0.0.1:3000 (your local dev server)
```

Use: letting a server-side tool call something running on your laptop for a short test. On the server it listens on loopback only, unless the server allows `GatewayPorts`.

> **How this goes wrong:** remote forwards can expose an internal network (your laptop's, or a server's) through someone else's machine, bypassing firewalls. Servers that don't need it should disable forwarding (Phase 10).

## 9.3 Dynamic forwarding (`-D`) — a SOCKS proxy

```bash
ssh -N -D 1080 app-server-01
curl --socks5-hostname localhost:1080 http://10.0.1.5:8080/health
```

Opens a **SOCKS proxy** on your laptop. Any program configured to use it (a browser, `curl`) has its connections made **from the server**. Useful for browsing several internal web UIs at once without a separate `-L` for each.

## 9.4 Summary of the three

| Flag | Opens a port on | Traffic exits from | Typical use |
|---|---|---|---|
| `-L` | Your **laptop** | The **server** | Reach private services (dashboards, DBs, debuggers) |
| `-R` | The **server** | Your **laptop** | Expose a local service to the server, briefly |
| `-D` | Your **laptop** (SOCKS) | The **server** | Browse many internal services |

---

# Phase 10 — The server side: `sshd_config`

## 10.1 Where and how

| Item | Detail |
|---|---|
| Main file | `/etc/ssh/sshd_config` |
| Drop-ins | `/etc/ssh/sshd_config.d/*.conf` — read **first**; for each option the **first** value wins (Note 4, Step 6) |
| Check syntax | `sudo sshd -t` |
| See effective settings | `sudo sshd -T` |
| Apply | `sudo systemctl reload ssh` |

**Ubuntu 24.04 detail:** SSH is **socket-activated** (`ssh.socket`, seen in Note 16, Step 0). Most settings apply with a reload, but changing `Port` or `ListenAddress` needs `sudo systemctl daemon-reload` and `sudo systemctl restart ssh.socket`, because systemd owns the listening socket.

## 10.2 Settings that matter

| Setting | Recommended | Why |
|---|---|---|
| `PermitRootLogin` | `no` | Log in as yourself, then sudo (Note 3, Phase 8.3) |
| `PasswordAuthentication` | `no` | Keys only — no password guessing |
| `KbdInteractiveAuthentication` | `no` (unless used for MFA) | Another password-style path |
| `PubkeyAuthentication` | `yes` | Keys |
| `AllowGroups` | `ssh-users` | Only this group may log in (Note 4, Step 6) |
| `MaxAuthTries` | `3`–`6` | Fewer guesses per connection |
| `LoginGraceTime` | `30` | Drop half-finished logins quickly |
| `AllowAgentForwarding` | `no` (unless needed) | Phase 6.2 |
| `AllowTcpForwarding` | `no` on app servers; `yes` on bastions | Tunnels (Phase 9) only where intended |
| `X11Forwarding` | `no` | Not needed on servers |
| `ClientAliveInterval` / `ClientAliveCountMax` | `300` / `2` | Clean up dead sessions |

## 10.3 `Match` blocks — different rules for different users

```
# /etc/ssh/sshd_config.d/20-deploy.conf
Match User deploy
    AllowTcpForwarding no
    AllowAgentForwarding no
    PermitTTY no
    ForceCommand /usr/local/bin/deploy.sh
```

Settings under `Match` apply only to matching connections — here, the CI `deploy` user (Note 4, Step 6; Note 8, Step 2) can only ever run the deploy script.

Test what a specific connection would get:

```bash
sudo sshd -T -C user=deploy,host=ci,addr=203.0.113.10 | grep -E 'allowtcpforwarding|forcecommand'
```

## 10.4 Reading the server's side

```bash
sudo journalctl -u ssh --since "1 hour ago"
```
```
sshd[6120]: Accepted publickey for thirana from 10.10.0.10 port 51022 ssh2: ED25519 SHA256:Wq3n...
sshd[6188]: Connection closed by authenticating user nimal 10.10.0.10 port 51110 [preauth]
sshd[6201]: User amal from 10.10.0.10 not allowed because none of user's groups are listed in AllowGroups
sshd[6230]: Authentication refused: bad ownership or modes for directory /home/nimal
```

The server log often says **exactly** why a login failed — the client usually only sees "Permission denied".

---

# Phase 11 — Copying files over SSH

| Tool | Best for | Example |
|---|---|---|
| `scp` | One-off copies | `scp report.csv app-server-01:/tmp/` |
| `sftp` | Interactive browsing and transfer | `sftp app-server-01` |
| `rsync` (over SSH) | Large or repeated copies — sends only **changes**, can resume | `rsync -avz --progress dir/ app-server-01:/srv/projects/dir/` |
| SSH + `tar` | Streaming many files, no temp archive | `ssh app-server-01 'tar -czf - /var/log/myapps' > logs.tgz` |

Notes:
- Modern OpenSSH's `scp` uses the SFTP protocol underneath.
- All of them use your `~/.ssh/config` (aliases, ProxyJump, keys).
- `rsync -a` preserves owners **by number** and permissions (Note 3, Phase 10; Note 5, Phase 9.6 — add `-A` for ACLs).
- Trailing slash matters in rsync: `dir/` copies the **contents**, `dir` copies the folder itself.

---

# Phase 12 — When SSH fails: reading the errors

## 12.1 First tool: verbose mode

```bash
ssh -v app-server-01        # -vv / -vvv for more
```

Read it in the order of Phase 3. Key lines:

```
debug1: Connecting to 10.10.0.4 [10.10.0.4] port 22.            ← step 1 (TCP)
debug1: Connection established.
debug1: Remote protocol version 2.0, remote software version OpenSSH_9.6
debug1: Server host key: ssh-ed25519 SHA256:Wq3n...             ← step 4
debug1: Host '10.10.0.4' is known and matches the ED25519 host key.
debug1: Offering public key: /home/thirana/.ssh/id_ed25519 ED25519 SHA256:Kx1... ← step 5
debug1: Server accepts key: ...
Authenticated to 10.10.0.4 ([10.10.0.4]:22) using "publickey".
```

The **last successful line** shows which step failed.

## 12.2 Common errors

| Error (client side) | Step | Likely cause | Check |
|---|---|---|---|
| `Connection timed out` | 1 | Firewall drop (cloud or host), wrong IP, wrong route back (Note 16, Step 7) | Phase 20 flow; firewall logs |
| `Connection refused` | 1 | sshd not listening on that address/port, or a firewall reject | `ss -tlnp` on server; `systemctl status ssh.socket` |
| `Could not resolve hostname` | before 1 | DNS / typo (Note 17) | `getent hosts NAME` |
| `Host key verification failed` / `IDENTIFICATION HAS CHANGED` | 4 | VM rebuilt — or interception | Phase 4.3 |
| `Permission denied (publickey)` | 5 | Key not in `authorized_keys`; wrong user; wrong key offered; bad permissions; not in `AllowGroups`; account expired (Note 4, Step 10) | **Server log** (Phase 10.4) |
| `Too many authentication failures` | 5 | Agent offered many keys before the right one (`MaxAuthTries`) | `IdentitiesOnly yes` + `IdentityFile` |
| Login hangs for seconds before the prompt | after 5 | Server doing reverse DNS on your IP, or GSSAPI attempts | `UseDNS no` (default now), `GSSAPIAuthentication no` |
| `channel N: open failed: administratively prohibited` | 6 | Tunnel requested but forwarding disabled on the server | `AllowTcpForwarding` (Phase 10) |
| `bind [127.0.0.1]:3001: Address already in use` | 6 | Your laptop already uses that local port | Pick another local port (`-L 3002:...`) |

---

# Part B — Network debugging

# Phase 13 — The method

## 13.1 Before running any command: describe the problem precisely

"It doesn't work" can't be debugged. Write down:

| Item | Example |
|---|---|
| **From** where (which host, which container, which user) | `worker@2` on `app-server-01`, as user `worker` |
| **To** what (name, IP, port, protocol) | `db.internal.team.example`, TCP 5432 |
| **What exactly** happens (exact error, how fast) | `ETIMEDOUT` after ~20 s |
| **Since when**, and what changed | Since this morning's VPC change |
| Does it work from **anywhere** else? | Works from `app-server-02` |

## 13.2 Follow the five questions in order

From Note 15, Phase 1.3 — each layer depends on the ones before it:

```
Q5 Name      → does the name resolve to the right IP?             (getent, dig)          Note 17
Q1 Interface → do I have an address and working link?             (ip addr, ip link)     Note 15
Q2 Route     → is there a route there AND back?                   (ip route get)         Note 15
Q4 Firewall  → is anything dropping/rejecting it?                 (nft, cloud rules)     Note 17
Q3 Socket    → is something listening on that address and port?  (ss on the far side)   Note 15
then: the application itself (TLS, auth, HTTP status, app logs)
```

(Name comes first in practice, because you can't test the rest without the right IP.)

## 13.3 Rules that save hours

| Rule | Why |
|---|---|
| **Let the error tell you the layer** | Refused vs timeout vs `ENOTFOUND` already rules out most layers (Note 15, Phase 11; Note 17, Phase 6.2) |
| **Test from more than one place** | The VM itself, a neighbour, outside (Note 16, Step 1) — the differences point to the layer |
| **Test the lowest layer you can** | Before blaming the app, check the port with `nc`; before blaming the network, check DNS with `getent` |
| **Change one thing at a time** | Otherwise you won't know what fixed it |
| **Watch for the error changing** | Timeout → refused → success is progress (Note 16, Step 3) |
| **Compare with a baseline** | "What's different from normal?" (Note 16, Step 11) |
| **Never "fix" by opening everything** | `0.0.0.0/0` rules, disabled firewalls, `chmod 777` (Note 6) hide the cause and create a new problem |

---

# Phase 14 — The toolbox at a glance

| Question | Tool | Shows |
|---|---|---|
| What IP does this name give my app? | `getent hosts NAME` | NSS result (Note 17, Phase 10) |
| What does DNS itself say? | `dig NAME`, `dig @SERVER NAME` | Raw DNS answers |
| Do I have addresses and a link? | `ip -br addr`, `ip -br link` | Note 15 |
| Which way will packets go? | `ip route get IP [from SRC]` | Note 15 |
| Is the TCP port reachable? | `nc -vz -w 3 HOST PORT` | succeeded / refused / timed out (Phase 16) |
| Does the HTTP service answer, and which part is slow? | `curl -v`, `curl -w` | Every phase of a request (Phase 15) |
| Is TLS working? | `openssl s_client` | Certificates, names, expiry (Phase 16) |
| Where on the path does it break? | `tracepath`, `traceroute -T`, `mtr` | Hops (Phase 17) |
| Is anything listening? | `ss -tlnp` on the far side | Note 15, Phase 12 |
| Is my firewall dropping it? | `nft list ruleset` counters, `journalctl -k` | Note 17, Phase 23 |
| Do packets actually arrive / leave? | `tcpdump` | The ground truth (Phase 18) |
| Cloud side? | VPC firewall logs, Connectivity Tests, flow logs | Phase 19 |

---

# Phase 15 — `curl` as a debugging tool

## 15.1 `curl -v`: every step of a request

```bash
curl -v https://api.team.example/health
```
```
* Host api.team.example:443 was resolved.              ← Q5: DNS worked
* IPv4: 192.0.2.10
*   Trying 192.0.2.10:443...                           ← Q2/Q4: attempting TCP
* Connected to api.team.example (192.0.2.10) port 443  ← TCP handshake done
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):      ← TLS begins
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* Server certificate:
*  subject: CN=api.team.example
*  expire date: Jan 20 10:12:01 2027 GMT
*  SSL certificate verify ok.                          ← certificate valid
> GET /health HTTP/2                                   ← ">" = what we sent
> Host: api.team.example
>
< HTTP/2 200                                           ← "<" = what came back
< content-type: application/json
```

Wherever the output **stops**, that's the layer that failed:

| Stops after… | Failing layer |
|---|---|
| Nothing / "Could not resolve host" | DNS |
| `Trying ...` (then timeout) | Path / firewall |
| `Trying ...` → "Connection refused" | Nothing listening |
| `Connected` → TLS errors | Certificates / TLS config |
| Response with 4xx / 5xx | The application |

## 15.2 `curl -w`: which phase is slow?

```bash
curl -s -o /dev/null -w '
dns:      %{time_namelookup}s
connect:  %{time_connect}s
tls:      %{time_appconnect}s
first byte: %{time_starttransfer}s
total:    %{time_total}s
' https://api.team.example/health
```
```
dns:      0.004s
connect:  0.031s
tls:      0.089s
first byte: 2.412s
total:    2.415s
```

Each value is the time **since the start**, so subtract to get each phase:

| Gap | Phase | Here |
|---|---|---|
| `dns` | Name lookup | 4 ms — fine |
| `connect − dns` | TCP handshake (network round trip) | 27 ms — fine |
| `tls − connect` | TLS handshake | 58 ms — fine |
| `first byte − tls` | **Server processing** | **2.3 s — the problem is the app**, not the network |

This one command often ends the "is it the network?" argument.

## 15.3 Bypassing parts on purpose

| Option | Effect | Use |
|---|---|---|
| `--resolve api.team.example:443:10.0.0.4` | Use this IP for that name — **skip DNS** without editing `/etc/hosts` | Test one backend directly, with the correct name for TLS |
| `-H 'Host: api.team.example'` | Send a different Host header | Test name-based virtual hosts by IP |
| `-4` / `-6` | Force IPv4 / IPv6 | IPv6-first problems (Note 17, Phase 11) |
| `--connect-timeout 5` / `-m 10` | Limit connect / total time | Don't wait minutes for a timeout |
| `-k` | Skip certificate checks | **Only** to confirm a certificate is the problem — never in scripts |
| `-I` | Headers only | Quick status checks |

---

# Phase 16 — `nc` and `openssl s_client`: testing ports and TLS

## 16.1 `nc`: is the TCP port reachable?

```bash
nc -vz -w 3 10.0.1.5 5432
```

| Output | Meaning |
|---|---|
| `Connection to 10.0.1.5 5432 port [tcp/postgresql] succeeded!` | Network path and listener are fine |
| `... failed: Connection refused` | Reached the host, nothing listening (or a reject) |
| `... timed out: Operation now in progress` | Dropped somewhere |

`nc` tests **only** the TCP connection — no app protocol. If `nc` succeeds but the app fails, look at the **application** layer (TLS, auth, protocol).

## 16.2 `nc` as a test **listener**: separate network from app

When you're unsure whether the network or the app is the problem, take the app **out** of the picture:

```bash
# on app-server-01 (pick a free port the firewall allows for the test):
nc -lk 9000

# on app-server-02:
echo hello | nc -w 3 10.0.0.4 9000
```

If `hello` appears on `app-server-01`, the **path and firewalls** are fine for that port — any remaining problem is in the real service. If not, it's the network. Stop the test listener afterwards (Part B, "How this goes wrong").

UDP (`nc -u`) is harder: with no reply there's no "connected" or "refused" — silence can mean success or failure. Test UDP services with their own tools (`dig` for DNS).

## 16.3 `openssl s_client`: look at TLS directly

TLS is the encryption layer under HTTPS. Its common failures — expired certificate, wrong name, missing intermediate certificate — are clearer here than in a browser:

```bash
openssl s_client -connect api.team.example:443 -servername api.team.example -brief </dev/null
```
```
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Peer certificate: CN = api.team.example
Verification: OK
```

```bash
# Expiry dates and names on the certificate
openssl s_client -connect api.team.example:443 -servername api.team.example </dev/null 2>/dev/null \
  | openssl x509 -noout -dates -subject -ext subjectAltName
```

| Problem | Shows as |
|---|---|
| Expired | `Verification error: certificate has expired` |
| Wrong name | Certificate's names (SAN) don't include the host you used |
| Missing intermediate | `unable to get local issuer certificate` — works in some browsers, fails in `curl`/apps |
| Wrong certificate entirely | Forgot `-servername` (SNI): the server returned its default certificate |

---

# Phase 17 — Path tools: `traceroute`, `tracepath`, `mtr`

## 17.1 How traceroute works

Every IP packet has a **TTL** (time to live) — a counter each router **lowers by 1**. When it hits 0, that router drops the packet and sends back an ICMP "time exceeded" message, revealing its address.

traceroute sends packets with TTL 1, then 2, then 3…, so each router on the path reveals itself in turn:

```
TTL=1 → router 1 replies "time exceeded"
TTL=2 → router 2 replies
...
TTL=n → destination answers
```

## 17.2 The tools

```bash
tracepath -n api.mailprovider.example           # no root needed; also reports path MTU
sudo traceroute -n -T -p 443 api.mailprovider.example   # TCP probes to port 443
mtr -n -rwc 50 api.mailprovider.example         # 50 rounds, report mode
```

`-T -p 443` matters: many networks drop traceroute's default probes but allow TCP to 443, so you test the **same kind** of traffic your app sends.

## 17.3 Reading `mtr`

```
HOST                         Loss%  Snt   Last   Avg  Best  Wrst
 1. 10.0.0.1                  0.0%   50    0.4   0.5   0.3   1.1
 2. ???                      100.0   50    0.0   0.0   0.0   0.0
 3. 198.51.100.1              0.0%   50    2.1   2.3   1.9   4.0
 4. 203.0.113.9              40.0%   50   31.0  31.4  30.2  33.0
 5. 203.0.113.25              0.0%   50   31.2  31.6  30.8  34.1
```

| What you see | Meaning |
|---|---|
| `???` / 100% loss at a middle hop, but later hops fine | That router simply **doesn't answer** probes. **Not a problem** |
| Loss at hop 4, but **0% at the destination** | Hop 4 limits how fast it answers probes. **Not real loss** |
| Loss that starts at a hop and **continues to the destination** | **Real** loss from that point on |
| Big latency jump that continues to the end | A slow link at that point (often distance — e.g. undersea cable) |

Inside a cloud VPC, traceroute usually shows few or no intermediate hops (the virtual network isn't made of normal routers you can see), so these tools are most useful for **internet** paths.

---

# Phase 18 — `tcpdump`: watching the packets themselves

## 18.1 What it is

`tcpdump` shows the actual packets passing an interface. Every other tool **infers**; tcpdump **shows**. When two teams disagree about whether traffic arrives, a capture settles it.

It needs root (or `CAP_NET_RAW`, Note 7) because it can see **all** traffic on the machine.

## 18.2 The basic command

```bash
sudo tcpdump -ni ens4 'tcp port 5432 and host 10.0.1.5'
```

| Part | Meaning |
|---|---|
| `-n` | Don't turn IPs into names (faster, avoids extra DNS traffic) |
| `-i ens4` | Which interface (`-i any` = all, and shows interface + direction) |
| `'...'` | A **filter**: only packets that match |
| `-c 20` | Stop after 20 packets |
| `-w file.pcap` | Save to a file (open later in **Wireshark**) |
| `-r file.pcap` | Read a saved file |

## 18.3 Filters

| Filter | Matches |
|---|---|
| `host 10.0.1.5` | To or from that IP |
| `src host 10.0.0.6` / `dst host ...` | One direction |
| `port 443` / `dst port 22` | A port |
| `tcp` / `udp` / `icmp` | A protocol |
| `udp port 53` | DNS traffic |
| `net 172.16.5.0/24` | A range |
| `and` / `or` / `not` / `( )` | Combine |

Example: `'tcp port 22 and not net 10.10.0.0/24'` — SSH from anywhere except the management subnet.

## 18.4 Reading a line

```
10:12:01.381 ens4 Out IP 10.0.0.4.42118 > 10.0.1.5.5432: Flags [S], seq 3811, win 64240, length 0
│            │    │      │              │                │
│            │    │      │              │                └─ TCP flags (below)
│            │    │      │              └─ destination IP.port
│            │    │      └─ source IP.port
│            │    └─ direction (with -i any)
│            └─ interface (with -i any)
└─ time
```

| Flags | Meaning |
|---|---|
| `[S]` | SYN — "I want to connect" |
| `[S.]` | SYN-ACK — "OK" |
| `[.]` | ACK |
| `[P.]` | Data (push) |
| `[F.]` | FIN — closing |
| `[R]` / `[R.]` | RST — reset / refused |

## 18.5 Recognising the patterns

| Capture shows | Meaning | Note 15 error |
|---|---|---|
| `[S]` out … `[S]` out (retries) … nothing back | Dropped somewhere **after** us, or the reply is lost | Timeout |
| `[S]` out, `[R.]` back | Reached the host; nothing listening (or reject) | Refused |
| `[S]`, `[S.]`, `[.]`, then data | Network fine — look at the application | — |
| `[S]` **in** on one interface, `[S.]` **out** on another | Asymmetric routing (Note 16, Step 7) | Timeout at the client |
| Same data sent again and again | Retransmissions — packet loss, or MTU (Note 16, Step 8) | Slow / hang |
| DNS query (`udp port 53`) with no reply | Resolver unreachable | `EAI_AGAIN` |

## 18.6 Capture at both ends

The most powerful technique: run `tcpdump` on **both** machines at the same time.

| Sender shows | Receiver shows | Problem is… |
|---|---|---|
| SYN leaving | **Nothing** | Between them — cloud firewall, routing |
| SYN leaving | SYN arriving, **no** SYN-ACK leaving | Receiver: host firewall, or nothing listening, or reply routed elsewhere |
| SYN leaving | SYN arriving, SYN-ACK leaving | Sender never gets it — the **return** path |

On the receiving server, a SYN that tcpdump sees but the app never gets usually means the **host firewall** dropped it (tcpdump sees packets **before** the `input` hook, Note 17, Phase 15).

> **How this goes wrong:** captures contain **real data** — passwords in plain-text protocols, tokens, personal data. Capture only what you need (filters, `-c`), store files privately (`umask 077`), and **delete them** when done.

---

# Phase 19 — Seeing what the firewalls are doing

## 19.1 Host firewall (Note 17)

```bash
sudo nft list ruleset | grep -B2 counter        # which rules are matching (packet counts)
sudo journalctl -k -g 'myfw-in-drop' --since "10 min ago"   # logged drops
sudo conntrack -L -d 10.0.1.5 -p tcp --dport 5432           # is the connection tracked?
```

A drop counter rising **while** you retry = your host firewall is dropping it.

## 19.2 Cloud firewall (GCP)

The VM can't see what Google's network drops. GCP's tools for that:

| Tool | Answers |
|---|---|
| **Firewall Rules Logging** | Which VPC rule allowed or denied a connection (enable per rule) |
| **Connectivity Tests** | Simulates a packet from A to B:port and reports which rule or route would stop it — without sending traffic |
| **VPC Flow Logs** | Samples of the connections actually seen on a subnet |

(Details belong to the GCP networking area.)

---

# Phase 20 — Putting it together: three decision flows

## 20.1 "A can't connect to B:port"

```
1. What exactly is the error?                    (Phase 13.1)
   ├─ ENOTFOUND / "not known"      → name doesn't exist      → Note 17 (spelling, record, search domain)
   ├─ EAI_AGAIN / "temporary"      → resolver problem         → resolvectl status, dig @resolver
   ├─ ENETUNREACH                  → no route on A            → ip route on A
   ├─ refused                      → reached B, nothing there → step 5
   └─ timeout / hangs              → something dropping       → steps 2–4

2. Name OK?            getent hosts NAME on A  → right IP?
3. Route there?        ip route get IP on A
4. Port reachable?     nc -vz -w 3 IP PORT from A
                       ├─ from another host too: same result?
                       ├─ host firewall on B: counters / logs        (Phase 19.1)
                       ├─ cloud firewall: rules, Connectivity Test   (Phase 19.2)
                       └─ still unclear → tcpdump on A and B        (Phase 18.6)
5. Listening?          ss -tlnp on B — right port? right ADDRESS (127.0.0.1 vs all)?
6. Route back?         ip route get A_IP from B_IP on B (multi-NIC, Note 16 Step 7)
7. Network all OK, app still fails?
                       curl -v / openssl s_client / app logs on B → TLS, auth, protocol
```

## 20.2 "It's slow"

```
1. curl -w timings                                        (Phase 15.2)
   ├─ dns slow            → resolver / search domains (Note 17, Phases 7–8)
   ├─ connect slow        → network distance, packet loss (mtr), full accept queue (Note 16 Step 6)
   ├─ tls slow            → TLS config, large certificate chains, MTU (Note 16 Step 8)
   └─ first byte slow     → the application (logs, CPU, database) — not the network
2. Only sometimes slow?   → see 20.3
```

## 20.3 "Only sometimes"

Intermittent problems usually come from something that **varies per connection or over time**:

| Pattern | Possible cause | Check |
|---|---|---|
| Fails for some clients, not others | One bad backend behind DNS round-robin or a load balancer | `getent ahosts`, test each IP with `curl --resolve` |
| Large requests fail, small work | MTU (Note 16, Step 8) | `ping -M do -s`, container MTU |
| Fails under load only | Ephemeral ports (Note 16, Step 5), accept queue (Step 6), conntrack table full (Note 17, Phase 23), fd limit (Note 14, Step 9) | `ss -s`, `nstat`, `dmesg`, `/proc/PID/limits` |
| Fails after a while idle | A firewall or NAT forgot an idle connection | Keep-alives (`ServerAliveInterval`, app keep-alive settings) |
| Fails right after a change, then fixes itself | DNS caching / TTL (Note 17, Phase 5) | `dig` against resolver vs authoritative |
| Only on IPv6-capable clients | Broken IPv6 path | `curl -4` vs `curl -6` |

---

# How this goes wrong — overview

| Issue | Why it matters | How to check |
|---|---|---|
| Password SSH enabled | Endless guessing from the internet | `sudo sshd -T \| grep passwordauthentication` |
| Root SSH login | No accountability, full power on compromise | `sudo sshd -T \| grep permitrootlogin` |
| Private keys without passphrase on laptops | Stolen laptop = server access | Policy; `ssh-keygen -y -f key` asks for one if set |
| Forgotten keys in `authorized_keys` | Ex-staff and old CI keys still work (Note 4, Step 10) | Review fingerprints regularly |
| Agent forwarding to shared servers | Root there can use your key while you're connected | Use ProxyJump; `AllowAgentForwarding no` |
| Ignoring host key warnings | Interception goes unnoticed | Team habit; `StrictHostKeyChecking` not disabled |
| TCP forwarding allowed everywhere | Users tunnel around firewalls; remote forwards expose internal services | `AllowTcpForwarding no` except on bastions |
| Tunnels bound to `0.0.0.0` on laptops | Others on the café Wi-Fi use your tunnel | Default `127.0.0.1`; no `GatewayPorts` |
| Test listeners left running (`nc -lk`) | An open port nobody owns | `ss -tlnp` baseline (Note 16, Step 11) |
| Packet captures left on disk | Contain secrets and personal data | Delete after use; restrict who has `CAP_NET_RAW` |
| `curl -k` in scripts | Certificate checks off → interception possible | `grep -r 'curl -k\|--insecure'` |
| Firewalls disabled "to test" | Forgotten, server left open | `ufw status`, `nft list ruleset` in the audit |

---

# Summary table

| Concept | Key point |
|---|---|
| SSH | Encrypted, authenticated remote access; replaced plain-text telnet/rsh |
| Private / public key | Private signs, stays home; public checks, goes on servers |
| Fingerprint | Short ID of a key, for comparing |
| ed25519 | Default key type |
| Connection steps | TCP → version → key exchange → **server** proves identity → **user** proves identity → channels |
| Key exchange | Both sides compute the same session key without sending it |
| Host key / `known_hosts` | How the client knows the server is genuine; trust on first use |
| "Identification has changed" | VM rebuilt — or interception; verify, then `ssh-keygen -R` |
| `authorized_keys` | Public keys allowed per account; options `restrict`, `from=`, `command=` |
| Passphrase + agent | Protect the key file; type once per session |
| Agent forwarding | Avoid — root on the remote can use it; use ProxyJump |
| `~/.ssh/config` | Aliases, keys, ProxyJump, tunnels; first value wins, `Host *` last |
| Multiplexing | Reuse one connection (`ControlMaster`) |
| Bastion / ProxyJump | One gateway; end-to-end session through it |
| GCP IAP | Managed bastion; IAM-controlled; range `35.235.240.0/20` |
| `-L` | Laptop port → service as seen **from the server** |
| `-R` | Server port → service reachable from the laptop |
| `-D` | SOCKS proxy through the server |
| `sshd_config` | Keys only, no root, `AllowGroups`, forwarding only where needed; `sshd -t` / `-T` |
| Ubuntu 24.04 | `ssh.socket` — port changes need `daemon-reload` + restart the socket |
| Server log | Says why a login failed |
| `ssh -v` | Last successful line = failing step |
| Debugging method | Precise description → error type → five questions in order → test from several places |
| `curl -v` / `-w` | Where it stops / which phase is slow |
| `curl --resolve` | Test one IP with the right name, no DNS change |
| `nc -vz` / `nc -l` | Port test / prove the path without the app |
| `openssl s_client` | Certificate, name, expiry, chain |
| traceroute / mtr | TTL-based path; loss only counts if it continues to the end |
| tcpdump | Ground truth; filters; flags `[S]`, `[S.]`, `[R]`; capture both ends |
| Firewall visibility | nft counters/logs on host; rules logging / Connectivity Tests in GCP |

---

# Appendix — Command reference

## SSH keys and identity

| Command | Purpose |
|---|---|
| `ssh-keygen -t ed25519 -C "comment"` | New key pair |
| `ssh-keygen -lf FILE.pub` | Fingerprint of a key |
| `ssh-keygen -R HOST` | Remove a host from `known_hosts` |
| `ssh-keyscan HOST` | Fetch a server's public host keys |
| `ssh-copy-id USER@HOST` | Install your public key |
| `ssh-add KEY` / `ssh-add -l` / `ssh-add -D` | Agent: add / list / clear |

## SSH connecting and config

| Command | Purpose |
|---|---|
| `ssh -v HOST` | Verbose (debug) |
| `ssh -J BASTION HOST` | Through a jump host |
| `ssh -G HOST` | Show the final config ssh will use |
| `ssh -O exit HOST` | Close a multiplexed connection |
| `gcloud compute ssh VM --zone Z --tunnel-through-iap` | GCP IAP |

## Tunnels

| Command | Purpose |
|---|---|
| `ssh -N -L LPORT:DEST:DPORT HOST` | Local forward |
| `ssh -N -R RPORT:DEST:DPORT HOST` | Remote forward |
| `ssh -N -D 1080 HOST` | SOCKS proxy |
| `LocalForward LPORT DEST:DPORT` | Same, in `~/.ssh/config` |

## SSH server

| Command | Purpose |
|---|---|
| `sudo sshd -t` | Check config syntax |
| `sudo sshd -T [-C user=U,host=H,addr=A]` | Effective settings (for a given connection) |
| `sudo systemctl reload ssh` | Apply most changes |
| `sudo systemctl daemon-reload && sudo systemctl restart ssh.socket` | Apply port/address changes (Ubuntu 24.04) |
| `sudo journalctl -u ssh` | Server log |

## File copy

| Command | Purpose |
|---|---|
| `scp FILE HOST:PATH` | Copy |
| `rsync -avz --progress SRC/ HOST:DST/` | Sync changes |
| `ssh HOST 'tar -czf - DIR' > out.tgz` | Stream an archive |

## Debugging

| Command | Purpose |
|---|---|
| `getent hosts NAME` / `dig NAME` | Name resolution (Note 17) |
| `ip route get IP [from SRC]` | Route (Note 15) |
| `nc -vz -w 3 HOST PORT` | TCP port test |
| `nc -lk PORT` | Temporary test listener |
| `curl -v URL` | Every step of a request |
| `curl -s -o /dev/null -w '...%{time_namelookup}...' URL` | Phase timings |
| `curl --resolve NAME:PORT:IP URL` | Skip DNS for one test |
| `openssl s_client -connect HOST:443 -servername HOST -brief` | TLS check |
| `... \| openssl x509 -noout -dates -subject -ext subjectAltName` | Certificate details |
| `tracepath -n HOST` | Path + MTU, no root |
| `sudo traceroute -n -T -p 443 HOST` | TCP traceroute |
| `mtr -n -rwc 50 HOST` | Path with loss per hop |
| `sudo tcpdump -ni IFACE 'FILTER'` | Watch packets |
| `sudo tcpdump -ni any -c 50 -w cap.pcap 'FILTER'` | Save a capture (delete after) |
| `sudo nft list ruleset` / `journalctl -k -g PREFIX` | Host firewall counters / logs |