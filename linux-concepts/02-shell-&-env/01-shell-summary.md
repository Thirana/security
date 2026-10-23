# How node gets its environment from bash

## The scenario

You are in a bash shell on the VM, in the `~/lab` folder. You set two variables and start a node app:

```bash
ubuntu@shell-lab:~/lab$ export PORT=3000
ubuntu@shell-lab:~/lab$ COLOR=blue
ubuntu@shell-lab:~/lab$ node server.js
```

In one line: bash makes a copy of itself (fork), the copy turns into node (exec), and node starts with **its own copy** of bash's exported variables.

```
 bash (exported: HOME, PATH, PORT)
   │ fork + exec
   ▼
 node  ← gets its own copy of HOME, PATH, PORT
```

---

## Step 0 — What the current shell holds before you press Enter

### Exported variables (the environment)

These are the variables that will be copied into node. (Example values.)

```
HOME=/home/ubuntu
PATH=/home/ubuntu/.nvm/versions/node/v22.11.0/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PORT=3000
```

Where each one came from:

- `HOME` — set at login from the user's account.
- `PATH` — system default, then `~/.profile`, then nvm (loaded by `~/.bashrc`) added its node folder at the front.
- `PORT` — you ran `export PORT=3000` (or `set -a; source app.env; set +a`).

You can see the full list of exported variables with `env` (or `export -p`).

### Not exported (only inside this shell)

```
COLOR=blue                 # shell variable, typed without export
alias ll='ls -alF'         # from ~/.bashrc
nvm                        # shell function, from ~/.bashrc sourcing nvm.sh
```

### Process details

- PID: `1401`
- Current folder: `/home/ubuntu/lab`
- User: `ubuntu`
- Open files: stdin, stdout, stderr = your terminal

### What will survive into node

- Exported variables (`HOME`, `PATH`, `PORT`) — **yes**
- Current folder, user, open files — **yes**
- Plain shell variables (`COLOR`) — no
- Aliases and functions (`ll`, `nvm`) — no
- PID — no, node gets a new one

---

## Step 1 — Find the program

`node` has no `/` in it, so bash searches the folders in `PATH` from left to right. The first folder is nvm's, so it finds:

```
/home/ubuntu/.nvm/versions/node/v22.11.0/bin/node
```

This is the step that fails in cron or `ssh host cmd`: their `PATH` doesn't include nvm's folder, so you get `node: command not found` before node even starts.

---

## Step 2 — fork

bash copies itself. The copy (the **child**) gets a new PID, say `1510`, and its parent is bash (PPID `1401`).

At this moment the child still has **everything**: `COLOR=blue`, `ll`, `nvm`, all of it. bash now waits for the child to finish.

---

## Step 3 — exec

The child asks the kernel: "replace me with the program in this file."

- The kernel checks the file has the `x` bit. `node` is a compiled program, so no shebang is involved.
- The child's memory is wiped and node loads in. `COLOR`, `ll`, `nvm` are gone.
- Kept across exec: the **environment** (`HOME`, `PATH`, `PORT`), the PID, the current folder, the user, and the open files.

That's why node prints to your terminal (it kept stdout) and `process.cwd()` is `/home/ubuntu/lab`.

---

## Step 4 — node runs with its own copy

Inside node:

| Code | Result |
|---|---|
| `process.env.HOME` | `'/home/ubuntu'` |
| `process.env.PORT` | `'3000'` |
| `process.env.COLOR` | `undefined` |
| `process.cwd()` | `'/home/ubuntu/lab'` |
| `process.pid` / `process.ppid` | `1510` / `1401` |

The copy is separate in both directions:

- If node does `process.env.PORT = '9000'`, bash's `PORT` stays `3000`. A child can't change its parent.
- If bash's `PORT` changes after node started (for example, node running in the background), node still sees `3000`. The copy was taken once, at start.

---

## Step 5 — node exits

node's memory, including its copy of the environment, is thrown away. The only thing bash gets back is the **exit code** (`$?`) — plus whatever node printed to the terminal.

---

## The full picture

```
 bash  PID 1401   cwd /home/ubuntu/lab
 ├─ exported:     HOME=/home/ubuntu
 │                PATH=/home/ubuntu/.nvm/versions/node/v22.11.0/bin:/usr/local/sbin:...:/bin
 │                PORT=3000
 ├─ shell only:   COLOR=blue, alias ll, function nvm
 │
 │ 1. look up "node" in PATH → /home/ubuntu/.nvm/versions/node/v22.11.0/bin/node
 │ 2. fork  → child PID 1510 (full copy of bash, COLOR/ll/nvm included)
 │ 3. exec  → memory wiped, node loaded
 │            kept: environment, current folder, user, open files
 ▼
 node  PID 1510, parent 1401   cwd /home/ubuntu/lab
 ├─ own copy:     HOME=/home/ubuntu
 │                PATH=/home/ubuntu/.nvm/versions/node/v22.11.0/bin:...
 │                PORT=3000
 ├─ missing:      COLOR, ll, nvm
 └─ on exit → only the exit code goes back to bash
```

---

## Same scenario, different ways of starting node

| How node is started | What node's copy looks like | Why |
|---|---|---|
| `node server.js` in your terminal | `HOME`, `PATH`, `PORT=3000` | Normal fork + exec from bash |
| `PORT=5000 node server.js` | `PORT=5000` for node only; bash keeps `3000` | Value is put into the child's copy only |
| `./server.js` with `#!/usr/bin/env node` | Same as the first row | Kernel reads the shebang and runs `/usr/bin/env`; `env` searches `PATH` (from its copy) for node and execs it. The environment passes through both steps |
| `bash -c 'node server.js'` | Same as the first row | A chain of copies: bash → bash → node |
| From cron | Tiny environment: `PATH=/usr/bin:/bin`, no `PORT` | Parent is cron, not your terminal. Nothing to inherit. Usually fails at Step 1 |
| `ssh host 'node server.js'` / `multipass exec` | Non-login, non-interactive: `~/.bashrc` stops at the guard, so no nvm `PATH` | Fails at Step 1 |
| `sudo node server.js` | Cleaned environment: `PORT` dropped, `PATH` replaced with sudo's safe default | sudo resets the environment by default for security |
| systemd service | Only what the unit file gives (`Environment=`, `EnvironmentFile=`) | Parent is systemd; reads no startup files |
| `source server.js` | Doesn't work | `source` makes **bash** read the file as shell commands. node is a different program, so it always runs as a child. To give it config, export it before starting it |

**One rule covers all of these:** node gets a copy of its **parent's** environment at the moment it starts. So the question is always "who is the parent, and what did the parent have exported?"

---

## Summary table

| Question | Answer |
|---|---|
| What does node inherit? | Exported variables, current folder, user, open files |
| What doesn't it inherit? | Plain shell variables, aliases, functions (wiped at exec) |
| Why? | fork copies everything; exec keeps only the environment and process details |
| Can node change bash's variables? | No — it has its own copy |
| Does node see later changes in bash? | No — the copy is taken once, at start |
| What comes back to bash? | Exit code (`$?`) and printed output |
| Why does it work in my terminal but not in cron / ssh / sudo / systemd? | Different parent, so a different (smaller) environment to copy |
| How do I give node config reliably? | Set it where node is started: `export` / `set -a; source file` in the script, `Environment=` in systemd |