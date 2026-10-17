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

Linux starts a new program in two steps.

**fork** — the shell makes a **copy of itself**. The copy is the child process. It has a new PID but the same memory: same variables, same current folder, same open files.

**exec** — the child **replaces itself** with the new program (`node`). The PID stays the same. The old shell code is gone. Only a few things survive the replace:

- current folder
- open files (stdin, stdout, stderr)
- user and group
- **the environment variables** (only the exported ones — Note 2)

```
 bash (PID 100)
   │
   │ fork
   ▼
 bash copy (PID 101)       ← same memory as parent, including ALL shell variables
   │
   │ exec node server.js
   ▼
 node (PID 101)            ← bash code replaced; only exported variables passed in
   │
   │ exits with code 0
   ▼
 bash (PID 100) gets $? = 0
```

This one picture explains most of both notes:

- **A child can't change its parent** — it is a separate copy.
- **Only exported variables reach programs** — exec passes only the environment list.
- **`cd` in a script doesn't change your folder** — the script is a child.

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

Some "commands" are shell functions loaded from startup files. They exist **only in shells that loaded them**.

```bash
type nvm         # nvm is a function
which nvm        # (nothing) — which only searches PATH for files
```

This is why `nvm use 20` works in your terminal but fails in scripts, cron, and services: those shells never loaded the function. (Scenario step 4.)

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

## 5.1 Three types

| Type | When it happens | Check |
|---|---|---|
| **Login shell** | SSH login, console login, `su - user`, `sudo -i`, `bash -l` | `echo $0` shows `-bash` |
| **Interactive non-login** | New terminal window on a desktop, typing `bash`, VS Code terminal | prompt shown, no `-` |
| **Non-interactive** | `bash script.sh`, `./script.sh`, cron jobs, systemd services | no prompt |

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

**Gotcha:** if you create a `~/.bash_profile`, bash stops reading `~/.profile`, so `~/.bashrc` is no longer loaded on login either. (Scenario step 2.)

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

| How | Runs in | Uses shebang? | Needs `chmod +x`? | Can change your shell? |
|---|---|---|---|---|
| `./deploy.sh` | New process | **Yes** | **Yes** | No |
| `bash deploy.sh` | New bash process | **No** (ignored) | No | No |
| `sh deploy.sh` | New sh process | **No** (ignored) | No | No |
| `source deploy.sh` or `. deploy.sh` | **Current shell** | No | No | **Yes** |

## 6.1 The shebang

The first line of a script, starting with `#!`:

```bash
#!/bin/bash
#!/bin/sh
#!/usr/bin/env python3
```

When you run `./deploy.sh`, the **kernel** reads this line and runs that interpreter with your script. It only matters for `./script` — `bash script` and `sh script` ignore it.

## 6.2 When to use which

- **`./script`** — normal way to run a script. The shebang decides the interpreter.
- **`bash script`** — quick test, or no execute permission. Careful: the result may differ from `./script` if the shebang says `sh`.
- **`source script`** — when the script's job is to **change your current shell**: load variables, activate a virtualenv, reload `~/.bashrc`.

---

# Phase 7 — `sh` vs `bash` vs `zsh`

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