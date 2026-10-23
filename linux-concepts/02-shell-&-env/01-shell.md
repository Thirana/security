# Shell & Processes — Notes (Note 1 of 2)

How the shell works, how it runs commands, and how processes are created. This is the base for **Note 2: Environment Variables**, which explains what processes pass to each other.

---

## Contents

1. Phase 1 — What a shell is
2. Phase 2 — How the shell runs a command
3. Phase 3 — Built-in commands vs programs
4. Phase 4 — Parent, child, subshells, and `exec`
5. Phase 5 — Shell types and startup files
6. Phase 6 — Running scripts: `./script`, `bash script`, `source`
7. Phase 7 — `sh` vs `bash` vs `zsh`
8. Phase 8 — Practical scenario: deploying a Node API
9. Summary table
10. Appendix — Command reference

---

# Phase 1 — What a shell is

## 1.1 Terminal, shell, kernel

Three different things that people often mix up:

| Part | What it is | Examples |
|---|---|---|
| **Terminal** | The window that shows text and takes your keystrokes | GNOME Terminal, Windows Terminal, iTerm, an SSH client |
| **Shell** | A **program** running inside the terminal. It reads your command, understands it, and starts other programs | `bash`, `sh` (`dash`), `zsh` |
| **Kernel** | The core of Linux. Actually creates processes, opens files, manages memory | Linux kernel |

```
You type → Terminal → Shell (bash) → asks Kernel → Kernel starts the program
```

The shell is just a normal program. You can start another one inside it by typing `bash`, and leave it with `exit`.

## 1.2 Which shell am I using?

```bash
echo $0            # current shell, e.g. bash or -bash
ps -p $$           # $$ = PID of the current shell
cat /etc/shells    # shells installed on this system
echo $SHELL        # your LOGIN shell (from /etc/passwd), not always the current one
```

Example: `echo $SHELL` says `/bin/bash`, but if you typed `zsh`, you are now in zsh. `$SHELL` doesn't change.

A `-` in front (`-bash`) means it is a **login shell** (Phase 5).

---

# Phase 2 — How the shell runs a command

## 2.1 The steps

When you type `node server.js --port 3000`:

1. **Read and split** the line into words.
2. **Expand**: replace `$VAR`, `~`, `*.js`, `$(...)`.
3. **Find the command**, in this order:
   1. Alias (`alias ll='ls -l'`)
   2. Shell function
   3. Built-in command (`cd`, `export`)
   4. A file found by searching **PATH**, left to right
4. If it's a program file: **fork**, then **exec** (next section).
5. **Wait** for it to finish and save the exit code in `$?`.

```bash
type -a node      # shows every match: alias, function, builtin, or file path
echo $?           # exit code of the last command (0 = success)
```

The shell remembers where it found programs (`hash`). If you install a new version in a different folder and the old one still runs:

```bash
hash -r           # clear the shell's remembered paths
```

## 2.2 fork and exec

This is the heart of the whole note. Take it slowly.

### First: what a process is

A **program** is a file on disk (`/usr/bin/node`). A **process** is that program **running** — loaded into memory, with its own private memory, its own current folder, its own open files, and a number that identifies it while it runs, the **PID** (process ID). Running the same program twice gives two processes with two PIDs. (Note 9 goes deep on processes; this much is enough here.)

Your shell is itself a process. So the question "how does the shell run `node`?" is really "how does one process start **another** process?".

### The surprise: it takes two steps, not one

You'd expect one step: "shell, please start `node`." Linux doesn't work that way. A process can't conjure another process out of nothing — the **only** way to make a new process is for an existing one to **copy itself**. So starting `node` is two steps:

1. **fork** — the shell clones itself. Now there are two near-identical shells.
2. **exec** — the clone throws away the shell program it's running and **becomes** `node` instead.

That "copy yourself, then immediately replace the copy" feels wasteful the first time you see it. Stay with it — the gap between the two steps is exactly what makes the shell powerful (below).

### fork — clone the process

**fork** makes a **copy of the calling process**. The copy is the **child**; the original is the **parent**. Right after the fork they're almost identical — same memory, same variables (exported *and* not), same current folder, same open files. The one thing that differs is the PID: the child gets a new one.

```bash
echo "shell  $$"          # the shell's PID, e.g. 35776
bash -c 'echo "child $$"' # a child's PID,  e.g. 35777  ← a different process
echo "shell  $$"          # still 35776                 ← the parent is unchanged
```

The child is a **separate** process. Nothing it does to its own memory, variables, or current folder can reach back and change the parent. Hold onto that — it's the reason `cd` in a script doesn't move *your* shell (Step 3), and the reason a child can't change a parent's variables (Note 2).

### exec — become a different program

The clone doesn't stay a shell. **exec** replaces the program running inside the process with a new one (`node`): the old shell code and memory are wiped and `node` is loaded in their place. Crucially, this happens **inside the same process** — the **PID does not change**. The process that was a bash clone is now `node`, same PID, same desk.

### Why some things survive exec and others don't

This is the part worth slowing down on. A running process has **two separate kinds of state** — remember these two as **kind 1** and **kind 2**; later phases refer back to them by those names:

1. **Kind 1 — the program's own memory.** Its code, and the variables and data it built up while running. For the shell clone, that's all of bash: its shell variables, functions, aliases, everything.
2. **Kind 2 — the kernel's record about the process.** A small card the kernel keeps for every process: its PID, which user and group it runs as, its current folder, a table of its open files, and its environment list. This lives in the **kernel**, not inside the program.

**exec wipes kind 1 and keeps kind 2.** The program's memory is thrown out and replaced by `node` — so bash's shell variables, functions, and aliases are **gone**. But the kernel's card is untouched, so everything on it carries straight into `node`:

| Survives (on the kernel's card) | So the new program… |
|---|---|
| **current folder** | starts in the same directory you were in |
| **open files** (stdin, stdout, stderr) | reads and writes the same places — this is how a redirect or pipe set up in the gap reaches `node` |
| **user and group** | runs as the same identity (unless the gap changed it, e.g. `sudo -u`) |
| **the environment variables** | receives them — but only the *exported* ones, because only those were copied onto the card as the environment (Note 2) |

That last row is the whole reason the exported/not-exported distinction matters: an ordinary shell variable lives in the program's memory (kind 1), so exec throws it away; an **exported** variable was placed in the environment list on the kernel's card (kind 2), so it survives into the new program. Note 2 is entirely about that line.

### The whole sequence

```
 bash (PID 100)
   │
   │ fork
   ▼
 bash copy (PID 101)       ← same memory as parent, including ALL shell variables
   │
   │ exec node server.js
   ▼
 node (PID 101)            ← bash code replaced; same PID; only exported variables passed in
   │
   │ exits with code 0
   ▼
 bash (PID 100) gets $? = 0
```

The parent (PID 100) was never touched; it just waits for the child (PID 101) to finish and collects its exit code in `$?` (step 5 of 2.1).

### Why two steps? The gap is the point

Here's the payoff that makes the design click. Between the fork and the exec, the child is **still running shell code**, and it's a throwaway copy — so the shell can **adjust the child's world** before it becomes the new program, without affecting itself. That gap is how the shell implements its most everyday features:

| You type | In the gap, the child... | then execs |
|---|---|---|
| `node > out.log` | points its stdout at `out.log` | `node` — which writes to the file without knowing |
| `sudo -u api node` | switches user to `api` | `node` — now running as `api` |
| `a \| b` | wires its output to `b`'s input | the two programs |

The program being run needs to know nothing about any of this. It just starts with the folder, files, user, and environment the child arranged. One clone-then-replace mechanism gives you redirection, pipes, user switching, and more.

### What this one picture explains

- **A child can't change its parent** — it's a separate copy (fork).
- **Only exported variables reach programs** — exec passes only the environment list (Note 2).
- **`cd` in a script doesn't change your folder** — the script runs in a child (Step 3).
- **Redirection and pipes work at all** — arranged in the gap between fork and exec.

## 2.3 Seeing it happen

```bash
pstree -p $$                          # your shell and its children
ps -ef --forest                       # all processes as a tree

strace -f -e trace=execve bash -c 'ls /tmp'
```

The `strace` output shows the `execve(...)` call — the actual exec — with the program path, arguments, and environment.

---

# Phase 3 — Built-in commands vs programs

## 3.1 What's the difference?

| | Built-in | Program |
|---|---|---|
| Where it lives | Inside the shell's own code | A separate file (`/usr/bin/ls`) |
| How it runs | Runs **inside the current shell** | fork + exec, runs as a **child** |
| Can change the current shell? | **Yes** | **No** |

```bash
type cd          # cd is a shell builtin
type export      # export is a shell builtin
type ls          # ls is /usr/bin/ls
type env         # env is /usr/bin/env
```

## 3.2 Why some commands must be built-ins

`cd`, `export`, `source`, `set`, `unset`, `exit`, `read`, `alias` all **change the shell itself**: its folder, variables, or state. If they were separate programs, they'd run as children, change their own copy, and exit. Your shell would not change at all.

That's also why:

- `env` (a program) can only show **exported** variables — it's a child, and it only receives what exec passed to it.
- `set` (a built-in) can show **all** shell variables — it looks inside the shell directly.

## 3.3 Functions and aliases

Not every "command" is a program on disk. Two of them — aliases and functions — are things the shell **keeps in its own memory**, which changes where they're available.

**An alias** is a simple text shortcut. You define one, and from then on the shell swaps the short word for the longer text *before* running the command:

```bash
alias ll='ls -l'      # define it
ll                    # the shell rewrites this to: ls -l
```

**A function** is a named block of shell commands, run inside the current shell (like a built-in you wrote yourself):

```bash
greet() { echo "hi from PID $$"; }    # define it
greet                                 # runs the block: hi from PID 36067
```

Neither is a file. There's no `ll` or `greet` anywhere in `/usr/bin`, so a PATH search would never find them — which is why they sit **above** PATH in the lookup order from 2.1 (alias → function → builtin → PATH).

### Why they exist "only in shells that loaded them"

Think back to the two kinds of state from Phase 2.2. An alias or a function lives in **the shell's own memory** (kind 1) — not as a file on disk, and not on the kernel's card. So:

- It exists only inside the one shell process where you defined it (or where a startup file defined it).
- A **child** shell starts as a fresh program and doesn't have it — nothing in your shell's memory crosses into a new process:

```bash
greet() { echo "hi"; }
greet                    # hi
bash -c greet            # bash: greet: command not found   ← the child never defined it
```

This is exactly why a new shell needs to **re-read** its startup files to get its aliases and functions back (Phase 5). Each shell rebuilds them from scratch; they don't travel with you.

### Why this bites: nvm

`nvm` isn't a program — it's a shell **function**, loaded by `~/.bashrc`:

```bash
type nvm         # nvm is a function
which nvm        # (nothing) — which only searches PATH for files
```

So `nvm use 20` works in your interactive terminal (which read `~/.bashrc`), but fails in scripts, cron jobs, and services — those run in non-interactive shells that read **no** startup files (Phase 5.2), so the function was never defined there. The fix is to load nvm's file inside the script, or skip the function and call the `node` binary by its full path (Scenario step 4).

---

# Phase 4 — Parent, child, subshells, and `exec`

## 4.1 Subshells

A **subshell** is a forked copy of the shell that doesn't exec anything — it keeps running shell code. Because it's a fork, it sees **all** variables (even unexported ones), but **its changes are lost** when it ends.

Things that create a subshell:

| Syntax | Example |
|---|---|
| `( ... )` | `( cd /tmp; ls )` |
| Pipes `\|` | `cat file \| while read line; do ...; done` — each side runs in a subshell in bash |
| Command substitution `$( ... )` | `now=$(date)` |
| Background `&` | `long_task &` |

```bash
X=1
( X=2; echo "inside: $X" )    # inside: 2
echo "outside: $X"            # outside: 1

( cd /tmp ); pwd              # still your old folder
```

**Subshell vs new program — the important difference:**

```bash
Y=hello                        # NOT exported
( echo "$Y" )                  # hello      ← subshell is a fork copy, sees it
bash -c 'echo "$Y"'            # (empty)    ← new program via exec, only gets exported vars
```

Grouping **without** a subshell — runs in the current shell:

```bash
{ cd /tmp; ls; }               # note the spaces and the final ;
```

`$$` always shows the main shell's PID. `$BASHPID` shows the real PID, so it changes inside a subshell:

```bash
echo $$ $BASHPID; ( echo $$ $BASHPID )
```

## 4.2 `exec` — replace the shell

`exec <program>` skips the fork. The shell **replaces itself** with the program. Same PID, and the shell is gone.

```bash
exec node server.js      # the shell becomes node; when node exits, the session ends
```

Why use it: in a **startup script** that sets things up and then starts the app. Without `exec`, the shell stays as an extra parent process in between.

```bash
#!/bin/sh
# entrypoint.sh
set -e
echo "starting..."
exec node /app/server.js     # node takes over the shell's PID
```

This matters for **signals** (stop requests). See scenario step 7.

`exec` also has a second, unrelated use — redirecting the current shell's output:

```bash
exec > /var/log/deploy.log 2>&1   # everything after this goes to the log
```

---

# Phase 5 — Shell types and startup files

### The problem startup files solve

Every new shell is a fresh process (fork + exec, Phase 2). From 3.3, you know that means it starts **empty** of aliases, functions, and the prompt — none of those cross over from another shell. So how does a shell you just opened already know your `PATH`, your aliases, your `nvm`, your prompt?

It runs **startup files** — scripts of shell commands (`~/.bashrc` and friends) that the shell `source`s into itself (Phase 6) the moment it starts, rebuilding your setup. "Startup files" are just normal shell scripts that bash reads automatically at the right moment.

The catch: bash reads **different** startup files depending on *how* it was started, and getting surprised by which file ran (or didn't) is behind most "it works in my terminal but not in the script" problems. So first, the three ways a shell starts.

## 5.1 Three types

| Type | When it happens | Why it's treated specially |
|---|---|---|
| **Login shell** | SSH login, console login, `su - user`, `sudo -i`, `bash -l` | You just **arrived** — set up one-time things (PATH, environment) |
| **Interactive non-login** | New terminal window on a desktop, typing `bash`, VS Code terminal | You're **already in**, opening another shell for typing — just needs the per-shell comforts (aliases, prompt) |
| **Non-interactive** | `bash script.sh`, `./script.sh`, cron jobs, systemd services | A **program**, not a person — no prompt, wants to start fast and predictably |

```bash
shopt -q login_shell && echo login || echo non-login
[[ $- == *i* ]] && echo interactive || echo non-interactive
```

## 5.2 Which files bash reads

| Shell type | Files read, in order |
|---|---|
| **Login** | `/etc/profile` (which loads `/etc/profile.d/*.sh`) → **the first one that exists** of `~/.bash_profile`, `~/.bash_login`, `~/.profile` |
| **Interactive non-login** | `/etc/bash.bashrc` (Debian/Ubuntu) → `~/.bashrc` |
| **Non-interactive** | **None** (only the file named in `$BASH_ENV`, if set) |
| On logout (login shell) | `~/.bash_logout` |

Ubuntu's default `~/.profile` loads `~/.bashrc`, so SSH logins get both.

### Why the split makes sense

The login vs non-login division looks arbitrary until you connect it to the two kinds of state from Phase 2.2:

- **One-time setup** — `PATH`, environment variables — goes in the **login** files (`~/.profile`). You set it once when you arrive, because environment variables are **kind 2 state: they inherit automatically** from a shell to every child it starts (Note 2). So a terminal you open later already has your `PATH` without re-reading anything.
- **Per-shell setup** — aliases, functions, the prompt — goes in `~/.bashrc`, read by **every interactive** shell. These are **kind 1 state: they do *not* inherit** (3.3), so each new interactive shell has to rebuild them for itself.

That's the whole logic: inherit it once → login file; rebuild it every time → `~/.bashrc`. And since a non-interactive shell (a script) is neither — it inherits the environment from its parent and needs no aliases or prompt — it reads **nothing**, which is exactly why `nvm` is missing there (3.3).

**Gotcha:** if you create a `~/.bash_profile`, bash reads it and **stops** looking for `~/.profile` (it reads only the first of the three that exists). Ubuntu's `~/.profile` was the file loading `~/.bashrc`, so now `~/.bashrc` isn't loaded on login either, and your aliases and `nvm` vanish. (Scenario step 2.)

## 5.3 What to put where

| Put in | What |
|---|---|
| `~/.profile` (or `~/.bash_profile`) | Things needed once per login: PATH additions, environment variables |
| `~/.bashrc` | Things for interactive use: aliases, functions, prompt, nvm loading |
| Nothing — put it in the script itself | Anything a script, cron job, or service needs |

Simple rule used on Ubuntu: put most things in `~/.bashrc`, and keep `~/.profile` loading it.

## 5.4 `su` and `sudo` shell forms

| Command | Gives you |
|---|---|
| `su user` | Shell as `user`, keeps much of your current environment |
| `su - user` | **Login shell** as `user`, clean environment, runs their startup files |
| `sudo -s` | Root shell, non-login |
| `sudo -i` | Root **login** shell (runs root's startup files) |

---

# Phase 6 — Running scripts: `./script`, `bash script`, `source`

There are a few ways to run a script, and they differ on two questions that you already have the tools to answer:

1. **Does it run in a new process, or in your current shell?** The first three below make a **child** (fork + exec, Phase 2), so anything the script changes — its folder, its variables — is lost when it ends (Step 3). `source` is the exception: it runs the lines **in your current shell**, so its changes stick. That's the single most important column.
2. **Who launches the file — the kernel, or an interpreter you named?** With `./deploy.sh` *you* ask the kernel to execute the file directly, so the kernel must be told it's executable (`chmod +x`) and must read the first line to learn which interpreter to use (the **shebang**). With `bash deploy.sh` you launch the `bash` program and hand it the script as a plain text file to read — so no execute bit is needed and the shebang is just a comment (bash is already the interpreter).

| How | Runs in | Uses shebang? | Needs `chmod +x`? | Can change your shell? |
|---|---|---|---|---|
| `./deploy.sh` | New process | **Yes** | **Yes** | No |
| `bash deploy.sh` | New bash process | **No** (ignored) | No | No |
| `sh deploy.sh` | New sh process | **No** (ignored) | No | No |
| `source deploy.sh` or `. deploy.sh` | **Current shell** | No | No | **Yes** |

So the first three answers to "can it change my shell?" are all **No** for the same reason — they're children — and `source` is **Yes** for the same reason — it isn't a child at all.

## 6.1 The shebang

A script is just a text file — the system has no built-in idea that `deploy.sh` is "a bash script" or that `backup.py` is "a Python script". The **shebang** is how the file says which interpreter should run it. It's the first line, starting with the two characters `#!` followed by the path to a program:

```bash
#!/bin/bash
#!/bin/sh
#!/usr/bin/env python3
```

### What the kernel actually does with it

Recall from Phase 6's intro that `./deploy.sh` asks the **kernel** to execute the file directly. The kernel reads the first two bytes; if they're `#!`, it doesn't try to run the file as a program itself — instead it runs the interpreter named on that line and hands it the script. So:

```bash
./deploy.sh
```
effectively becomes:
```bash
/bin/bash ./deploy.sh     # because the first line was #!/bin/bash
```

The shebang is literally **a path to a program the kernel will launch**, which is why it must be an **absolute path** (`/bin/bash`, not `bash`) — the kernel doesn't do a PATH search.

### Why `#!` is a `#`

Notice the line also starts with `#`, which is a **comment** in bash, sh, Python, and most scripting languages. That's deliberate: the kernel treats `#!…` as the interpreter line, but once bash (or python) is running the script, that same first line is just a comment it harmlessly ignores. One line, read two different ways.

### `#!/usr/bin/env python3` — and why it's common

A plain `#!/usr/bin/python3` hardcodes one exact path. If Python lives somewhere else — a virtualenv, a user install, a different distro — the script breaks. `#!/usr/bin/env python3` instead runs the `env` program, whose job here is to **search `$PATH`** for `python3` and run the first one it finds:

```bash
#!/usr/bin/env python3    # runs whichever python3 is first on PATH (e.g. a virtualenv's)
```

The trade-off: it's portable, but you get *whichever* `python3` is on `$PATH`, which isn't always the one you meant. (`env` is itself at a fixed path, `/usr/bin/env`, so that part is safe.)

### No shebang, or the wrong one

If a file has **no** shebang and you run `./script`, the kernel can't tell what interpreter to use and refuses to execute it (the raw `execve` fails with `ENOEXEC`). Shells then paper over this inconsistently — bash, for instance, falls back to running it with a plain `sh` — so the script may run under a different shell than you expect, or not at all. Don't rely on the fallback: **give every script a shebang.**

### When it matters

The shebang only has an effect when the **kernel** launches the file — that is, `./script` (and the same when cron or systemd runs a script by its path). When you run `bash deploy.sh` or `sh deploy.sh`, *you've already named the interpreter*, so the shebang line is ignored and runs only as a comment. This is the mechanism behind Scenario step 6: a script tested with `bash backup.sh` passes, but the same file run as `./backup.sh` (or by cron) obeys its `#!/bin/sh` shebang and runs under dash, where bash-only syntax fails.

## 6.2 When to use which

- **`./script`** — normal way to run a script. The shebang decides the interpreter.
- **`bash script`** — quick test, or no execute permission. Careful: the result may differ from `./script` if the shebang says `sh`.
- **`source script`** — when the script's job is to **change your current shell**: load variables, activate a virtualenv, reload `~/.bashrc`.

---

# Phase 7 — `sh` vs `bash` vs `zsh`

"The shell" isn't one program — it's a family of them, and they don't all understand the same syntax. `bash` added many conveniences over the years; the minimal POSIX shell (`dash` on Ubuntu) doesn't have them. A script written with bash conveniences will **fail** if something runs it with a plainer shell — and on Ubuntu that happens more easily than you'd expect, because of one trap:

> **`/bin/sh` is not bash.** On Ubuntu it's `dash`, a small, fast, strict POSIX shell. So a script with `#!/bin/sh` runs under dash, and any bash-only syntax in it breaks — even though typing `bash script` by hand would have worked. This is the core of Scenario step 6.

This phase is about knowing which shell you're actually running under, and which syntax is safe in each.

## 7.1 The shells

| Shell | Notes |
|---|---|
| `sh` | The POSIX standard shell. On Ubuntu/Debian `/bin/sh` is **dash** — small and fast, fewer features |
| `bash` | Default interactive shell on most Linux. Has many extras over `sh` |
| `zsh` | Default on macOS. Similar to bash, different startup files |

```bash
ls -l /bin/sh       # /bin/sh -> dash   (Ubuntu)
```

## 7.2 Bash-only features ("bashisms") that break in `sh`/dash

| Bash | Works in sh/dash? | sh-safe version |
|---|---|---|
| `[[ $x == y ]]` | No | `[ "$x" = y ]` |
| `source file` | No | `. file` |
| `arr=(a b c)` | No | No arrays — use other ways |
| `{1..5}` | No | `seq 1 5` |
| `function name { }` | No | `name() { }` |
| `$RANDOM`, `<( )`, `==` inside `[ ]` | No | — |

Tools to catch these:

```bash
shellcheck deploy.sh       # general script linter, very useful
checkbashisms deploy.sh    # finds bash-only syntax in sh scripts
```

Rule: if a script uses bash features, its shebang must be `#!/bin/bash`.

## 7.3 zsh startup files (for macOS or zsh users)

| File | Read by |
|---|---|
| `~/.zshenv` | **Every** zsh, including scripts |
| `~/.zprofile` | Login shells |
| `~/.zshrc` | Interactive shells |
| `~/.zlogin` | Login shells, after `.zshrc` |

zsh does not read `~/.bashrc` or `~/.profile` (unless set up to).

---

# Phase 8 — Practical scenario: deploying a Node API

## The setup

- Ubuntu server, you log in as `thirana` over SSH.
- Node.js API at `/opt/api`, Node installed with **nvm**.
- A `deploy.sh` script, a nightly backup script, and later a Docker container.

Each step shows the problem, how to check it, what the result means, and the fix. The environment-variable side of the same deployment is in **Note 2**.

---

## Step 1 — SSH in: what kind of shell is this?

```bash
ssh thirana@server
echo $0
```

Result: `-bash`

**Meaning:** the `-` means a **login shell**. So bash read `/etc/profile`, then `~/.profile`, and Ubuntu's `~/.profile` loaded `~/.bashrc`.

Check where nvm gets loaded:

```bash
grep -n nvm ~/.bashrc ~/.profile
```

Result: nvm's setup lines are in `~/.bashrc` (the nvm installer put them there).

**Conclusion:** `node` works in SSH sessions because `~/.bashrc` runs on login through `~/.profile`.

---

## Step 2 — You create `~/.bash_profile` and `node` disappears

You add an alias to a new file:

```bash
echo "alias api-logs='journalctl -u api -f'" > ~/.bash_profile
```

Next SSH login:

```bash
node -v
```

Result: `node: command not found`

**Check** which files bash reads for a login shell (Phase 5.2): the **first one that exists** of `~/.bash_profile`, `~/.bash_login`, `~/.profile`.

```bash
ls -la ~/.bash_profile ~/.profile
```

**Meaning:** `~/.bash_profile` now exists, so bash reads it and **skips `~/.profile`**. `~/.profile` was the file loading `~/.bashrc`, so nvm never loaded.

**Fix:** delete `~/.bash_profile` and put the alias in `~/.bashrc`, or make `~/.bash_profile` load the others:

```bash
# ~/.bash_profile
[ -f ~/.profile ] && . ~/.profile
```

---

## Step 3 — `deploy.sh` and the folder that doesn't change

```bash
# deploy.sh
cd /opt/api
git pull
npm ci
```

```bash
./deploy.sh
```

Result: `bash: ./deploy.sh: Permission denied`

**Meaning:** `./script` needs execute permission (Phase 6).

```bash
chmod +x deploy.sh
./deploy.sh
pwd
```

Result: the deploy works, but `pwd` still shows your **home folder**, not `/opt/api`.

**Meaning:** `./deploy.sh` runs as a **child process** (fork + exec). Its `cd` changed the child's folder. The child exited, and your shell never changed.

**Conclusion:** this is expected. If you want to stay in `/opt/api`, `cd` there yourself — or `source deploy.sh`, but sourcing a deploy script is risky because any `exit` in it closes your terminal.

Also, `deploy.sh` has **no shebang**. Add `#!/bin/bash` as the first line so the interpreter is always clear.

---

## Step 4 — `nvm use` fails inside `deploy.sh`

You add a line to make sure the right Node version is used:

```bash
#!/bin/bash
nvm use 20
cd /opt/api
...
```

Result: `nvm: command not found`

**Check:**

```bash
type nvm       # nvm is a function
which nvm      # (nothing)
```

**Meaning:** nvm is a **shell function** (Phase 3.3), loaded by `~/.bashrc`. `./deploy.sh` runs in a **non-interactive** shell, which reads **no startup files** (Phase 5.2), so the function doesn't exist there.

**Fix (pick one):**

```bash
# A: load nvm inside the script
export NVM_DIR="$HOME/.nvm"
. "$NVM_DIR/nvm.sh"
nvm use 20

# B: skip nvm and call the node binary by full path
/home/thirana/.nvm/versions/node/v20.11.0/bin/node -v
```

For services and cron, **B** (full path, or a system-wide Node install) is more reliable.

---

## Step 5 — A health-check counter that stays at 0

A script checks API endpoints listed in a file:

```bash
#!/bin/bash
ok=0
cat endpoints.txt | while read url; do
  if curl -fs "$url" > /dev/null; then
    ok=$((ok + 1))
  fi
done
echo "healthy: $ok"
```

Result: `healthy: 0`, even though the endpoints are up.

**Check:** add a print inside the loop:

```bash
    echo "inside loop ok=$ok (PID $BASHPID)"
```

Result: `ok` goes up inside the loop. `$BASHPID` there is different from the main script's PID.

**Meaning:** in bash, each side of a pipe runs in a **subshell** (Phase 4.1). The loop increased `ok` in its own copy. That copy ended with the pipe.

**Fix:** feed the file with a redirect, so the loop runs in the current shell:

```bash
while read url; do
  ...
done < endpoints.txt
```

For command output instead of a file:

```bash
done < <(curl -s http://localhost:3000/routes)
```

---

## Step 6 — The backup script works by hand, fails in cron

```bash
#!/bin/sh
# backup.sh (copied from an old template)
source /opt/api/.env
if [[ -z "$DATABASE_URL" ]]; then echo "no DB url"; exit 1; fi
pg_dump "$DATABASE_URL" > /backups/db.sql
```

You test it:

```bash
bash backup.sh       # works
```

Cron entry: `0 2 * * * /opt/api/backup.sh`

Next morning, the cron mail/log says:

```
/opt/api/backup.sh: 3: source: not found
/opt/api/backup.sh: 4: [[: not found
```

**Meaning:**
- `bash backup.sh` **ignores the shebang** (Phase 6) — your test ran in bash.
- Cron runs `/opt/api/backup.sh` directly, so the kernel uses the shebang `#!/bin/sh` → **dash** on Ubuntu.
- `source` and `[[ ]]` are bash-only (Phase 7.2).

**Check it the way cron runs it:**

```bash
./backup.sh          # same error as cron
ls -l /bin/sh        # /bin/sh -> dash
```

**Fix (pick one):**
- Change the shebang to `#!/bin/bash`, or
- Make it sh-safe: `. /opt/api/.env` and `[ -z "$DATABASE_URL" ]`.

**Lesson:** test scripts the same way they will be run (`./script`), not with `bash script`. Run `shellcheck` on them.

(Cron also gives scripts a tiny environment — Note 2 covers that part.)

---

## Step 7 — Docker container takes 10 seconds to stop

`Dockerfile`:

```dockerfile
COPY entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

`entrypoint.sh`:

```sh
#!/bin/sh
echo "running migrations..."
node migrate.js
node server.js
```

```bash
time docker stop api
```

Result: about **10 seconds**, and the app logs show no clean shutdown.

**Check the process tree inside the container:**

```bash
docker exec api ps -o pid,ppid,cmd
```

```
PID PPID CMD
  1    0 /bin/sh /entrypoint.sh
  7    1 node server.js
```

**Meaning:**
- `docker stop` sends **SIGTERM** (a "please stop" signal) to **PID 1** only.
- PID 1 is `sh`, not node. `sh` does not pass the signal to its child.
- Docker waits 10 seconds, then sends **SIGKILL** and kills everything without a clean shutdown.

**Fix:** use `exec` for the last command (Phase 4.2):

```sh
#!/bin/sh
set -e
node migrate.js
exec node server.js
```

Now:

```
PID PPID CMD
  1    0 node server.js
```

node is PID 1 and receives SIGTERM directly. Also make sure the app handles it:

```js
process.on('SIGTERM', () => server.close(() => process.exit(0)));
```

(PID 1 ignores signals it has no handler for. Without this handler, or without `docker run --init`, node as PID 1 can still ignore SIGTERM.)

Under **systemd** this is less of a problem, because systemd sends the stop signal to all processes of the service by default. Using `exec` is still good practice so the main PID is your app.

---

## Scenario recap

| Step | Problem | Shell concept |
|---|---|---|
| 1 | Why does node work on SSH? | Login shell → `~/.profile` → `~/.bashrc` |
| 2 | node disappears after creating `~/.bash_profile` | Login shell reads only the first of three files |
| 3 | `cd` in script doesn't change your folder | Script = child process |
| 4 | `nvm` not found in script | Functions only exist where loaded; scripts read no startup files |
| 5 | Counter stays 0 after a pipe loop | Pipes run in subshells |
| 6 | Works with `bash script`, fails in cron | Shebang `sh` = dash; bashisms; test the real way |
| 7 | Container slow to stop | Shell as PID 1 doesn't pass signals; use `exec` |

---

# Summary table

| Concept | Key point |
|---|---|
| Terminal vs shell | Terminal = window; shell = program that runs commands |
| Command lookup | Alias → function → builtin → PATH (left to right) |
| fork | Shell makes a copy of itself (child) |
| exec | Child replaces itself with the program; only exported env passes through |
| Child vs parent | A child can never change its parent |
| Built-ins | Run inside the shell; needed for `cd`, `export`, `source` |
| Functions (nvm) | Exist only in shells that loaded them |
| Subshell | `( )`, pipes, `$( )`, `&`; sees all variables, changes are lost |
| `{ ...; }` | Grouping in the current shell, no subshell |
| `exec cmd` | Replace the shell with `cmd`; same PID |
| Login shell | Reads `/etc/profile` + first of `~/.bash_profile`/`~/.bash_login`/`~/.profile` |
| Interactive non-login | Reads `~/.bashrc` |
| Non-interactive (scripts, cron, services) | Reads no startup files |
| `./script` | Uses shebang, needs `chmod +x`, child process |
| `bash script` | Ignores shebang |
| `source script` | Runs in current shell, can change it |
| `/bin/sh` on Ubuntu | dash: no `[[ ]]`, `source`, arrays |
| PID 1 in containers | Should be your app (`exec`), and it should handle SIGTERM |

---

# Appendix — Command reference

## Identify the shell

| Command | Purpose |
|---|---|
| `echo $0` | Current shell (`-bash` = login shell) |
| `ps -p $$` | Current shell process |
| `echo $SHELL` | Your login shell from `/etc/passwd` |
| `cat /etc/shells` | Installed shells |
| `ls -l /bin/sh` | What `sh` really is (dash on Ubuntu) |
| `shopt -q login_shell && echo login` | Login shell check |
| `[[ $- == *i* ]] && echo interactive` | Interactive check |

## Commands and lookup

| Command | Purpose |
|---|---|
| `type -a <cmd>` | All matches: alias, function, builtin, file |
| `which <cmd>` | File found in PATH only |
| `command -v <cmd>` | Portable way to find a command (works in sh) |
| `hash -r` | Clear remembered command paths |
| `echo $?` | Exit code of last command |
| `help` / `help cd` | List / explain bash built-ins |

## Processes

| Command | Purpose |
|---|---|
| `echo $$` / `echo $BASHPID` | Main shell PID / real current PID |
| `pstree -p $$` | Your shell's process tree |
| `ps -ef --forest` | All processes as a tree |
| `strace -f -e trace=execve <cmd>` | See exec calls |
| `exec <cmd>` | Replace the shell with a program |
| `exec > file 2>&1` | Redirect all later output of this shell |

## Subshells and grouping

| Syntax | Runs in |
|---|---|
| `( cmds )` | Subshell |
| `{ cmds; }` | Current shell |
| `a \| b` | Each side in a subshell (bash) |
| `$( cmd )` | Subshell |
| `cmd &` | Background subshell |
| `while ...; done < file` | Current shell (avoids pipe subshell) |
| `done < <(cmd)` | Current shell, input from a command |

## Scripts

| Command | Purpose |
|---|---|
| `chmod +x script.sh` | Allow `./script.sh` |
| `./script.sh` | Run with shebang interpreter |
| `bash script.sh` | Run with bash, ignore shebang |
| `source script.sh` / `. script.sh` | Run in current shell |
| `bash -n script.sh` | Syntax check without running |
| `bash -x script.sh` | Run and print each command (debugging) |
| `shellcheck script.sh` | Lint for common mistakes |
| `checkbashisms script.sh` | Find bash-only syntax in sh scripts |

## Shells and users

| Command | Purpose |
|---|---|
| `bash` / `exit` | Start / leave a nested shell |
| `bash -l` | Start a login shell |
| `su - user` | Login shell as another user |
| `sudo -i` / `sudo -s` | Root login shell / root non-login shell |