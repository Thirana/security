# Environment Variables & the `env` Command — Notes (Note 2 of 2)

How processes get their settings, where the values come from, how to keep two apps' settings separate, and how to keep secrets safe.

Builds on **Note 1: Shell & Processes**. Note 1 explains how processes are created (fork, exec, subshells, startup files). This note explains what they carry.

---

## Contents

1. Phase 1 — Core concepts
2. Phase 2 — The `env` command and related commands
3. Phase 3 — Where values come from
4. Phase 4 — Practical scenario: two apps on one server
5. Phase 5 — Troubleshooting: "why did my variable disappear?"
6. Phase 6 — Security
7. Summary table
8. Appendix — Command reference

---

# Phase 1 — Core concepts

## 1.1 Quick recap from Note 1

- Starting a program = **fork** (copy the shell) + **exec** (replace the copy with the program).
- **exec passes only the environment list** to the new program.
- A child **gets a copy**. It can never change its parent.

```
 bash (exported: HOME, PATH, PORT)
   │ fork + exec
   ▼
 node  ← gets its own copy of HOME, PATH, PORT
```

## 1.2 What is an environment variable?

A **`KEY=value` pair** in a process's environment list.

```
HOME=/home/thirana
PATH=/usr/local/bin:/usr/bin:/bin
PORT=3000
DATABASE_URL=postgres://api_rw:S3cret@10.0.0.5:5432/appdb
```

Programs read these to get settings without hardcoding them.

```js
// Node
const port = process.env.PORT || 3000;
```
```python
# Python
import os
port = int(os.environ.get("PORT", 8000))
```

- Names are case-sensitive. Convention: UPPERCASE.
- Values are always **strings**. The app converts `"3000"` to a number.

## 1.3 Each process has its own copy

There is no single global environment. Each process holds its own list. Two apps can both have a `DATABASE_URL` with **different values**, and they never see each other's. Phase 4 uses this a lot.

## 1.4 The environment is read at start

A process gets its environment **when it starts**. If you later edit a `.env` file or a unit file, the running process **does not change**. You must restart it.

## 1.5 Shell variable vs environment variable

| Kind | Who sees it | How to create |
|---|---|---|
| **Shell variable** | Only the current shell (and its subshells) | `NAME=thirana` |
| **Environment variable** | Current shell **and every program it starts** | `export NAME=thirana` |

```bash
FOO=1
bash -c 'echo "program sees: $FOO"'     # program sees:

export FOO=1
bash -c 'echo "program sees: $FOO"'     # program sees: 1
```

Why: `bash -c` is a new program, started with exec, so it only gets the exported list (Note 1, Phase 2.2). A subshell `( echo $FOO )` would see it even without `export`, because a subshell is just a fork copy (Note 1, Phase 4.1).

## 1.6 One-command variables

```bash
PORT=4000 node server.js
```

`PORT=4000` exists only in that `node` process. Your shell doesn't keep it.

## 1.7 Quoting and default values

```bash
export GREETING="hello world"       # quotes needed for spaces
echo "$GREETING"                    # always quote when using

echo "${PORT:-3000}"                # 3000 if PORT is unset or empty
: "${DATABASE_URL:?DATABASE_URL missing}"   # stop the script with an error if unset
```

---

# Phase 2 — The `env` command and related commands

## 2.1 `env` — show the environment

```bash
env                 # all environment variables
env | sort          # easier to read
env | grep -i port  # filter
```

`env` is a separate program (`type env` → `/usr/bin/env`), so it only shows **exported** variables. `set` is a shell built-in and shows everything (Note 1, Phase 3.2).

## 2.2 `env` — run a command with a changed environment

```bash
env PORT=4000 node server.js            # add or override for one command
env -u API_KEY ./tool                   # run without API_KEY
env -i bash --norc                      # run with an EMPTY environment
env -i PATH=/usr/bin:/bin ./script.sh   # empty, plus only what you give
```

`env -i` is useful for testing: "does this still work with nothing set?" That is close to what cron and services get.

## 2.3 `#!/usr/bin/env`

```bash
#!/usr/bin/env python3
```

Means "find `python3` using PATH". So **the PATH at run time decides which python runs** — inside an activated virtualenv, it's the venv's python (Phase 4, step 3).

## 2.4 Related commands

| Command | What it does |
|---|---|
| `printenv` | Same as `env` |
| `printenv HOME` | One variable |
| `echo "$HOME"` | One variable (shell expands it) |
| `set` | All shell variables + env variables + functions |
| `export VAR=value` | Create and export |
| `export -p` | List exported variables |
| `export -n VAR` | Un-export (keep as shell variable) |
| `unset VAR` | Delete |
| `readonly VAR` | Prevent changes in this shell |

## 2.5 Important variables

| Variable | Meaning |
|---|---|
| `PATH` | Folders searched for commands, left to right, first match wins |
| `HOME` | Home folder |
| `USER` / `LOGNAME` | Username |
| `SHELL` | Login shell |
| `PWD` | Current folder |
| `LANG` / `LC_*` | Language and locale |
| `TZ` | Time zone |
| `EDITOR` | Default editor |
| `LD_LIBRARY_PATH` | Extra folders for shared libraries |

### PATH

```bash
echo "$PATH" | tr ':' '\n'      # one folder per line
which node                      # which file will run
```

Add a folder without breaking PATH:

```bash
export PATH="$HOME/bin:$PATH"   # correct
export PATH="$HOME/bin"         # WRONG: ls, cat, etc. stop working
```

---

# Phase 3 — Where values come from

A process's environment is whatever its parent passed in, plus whatever was added along the way. Who the parent is decides what you get.

| Context | Gets its environment from |
|---|---|
| Your SSH shell | PAM (`HOME`, `USER`, `SHELL` from `/etc/passwd`) + `/etc/environment` + startup files (`/etc/profile`, `~/.profile`, `~/.bashrc` — which ones: Note 1, Phase 5) + SSH `AcceptEnv` (e.g. `LANG`) |
| A program you run | A copy of your shell's exported variables |
| `sudo cmd` | A **reset**, small environment + `secure_path` as PATH |
| systemd service | A small clean environment + `Environment=` / `EnvironmentFile=` in the unit. **No shell startup files** |
| cron job | A tiny environment (`HOME`, `LOGNAME`, `SHELL=/bin/sh`, `PATH=/usr/bin:/bin`) + variables at the top of the crontab |
| Docker container | Image `ENV` + `-e` / `--env-file` / compose `environment:` / `env_file:` |

Where to put things:

| Goal | Put it in |
|---|---|
| Your own interactive settings | `~/.bashrc` |
| System-wide for all logins | `/etc/environment` or `/etc/profile.d/*.sh` |
| An app's config and secrets | Its own env file, loaded by systemd/Docker |
| A cron job | Inside the script (or top of the crontab for non-secrets) |

Rule of thumb: **app settings don't belong in your shell's startup files.** Step 1 shows why.

---

# Phase 4 — Practical scenario: two apps on one server

## The setup

One Ubuntu server, two apps that work together:

| | App A: API | App B: Worker |
|---|---|---|
| Language | Node.js (installed with **nvm**) | Python (in a **virtualenv**) |
| Folder | `/opt/api` | `/opt/worker` |
| Job | Serves HTTP on port 3000 | Sends emails from a queue, nightly DB backup, small health endpoint |
| Needs | `PORT`, `DATABASE_URL`, `API_KEY` | `HEALTH_PORT`, `DATABASE_URL`, `SMTP_PASSWORD`, `API_BASE_URL` |
| DB user | `api_rw` (read/write) | `backup_ro` (read-only) |
| Runs as | user `api` | user `worker` |

Both use a variable called `DATABASE_URL`, with **different values**. The worker calls the API at `API_BASE_URL`.

Each step: the problem, the command to check, what the result means, and the fix.

---

## Step 1 — Running both by hand: port clash

Earlier, while building the API, you added this to `~/.bashrc`:

```bash
export PORT=3000
```

Terminal 1:

```bash
cd /opt/api && node server.js        # listening on 3000
```

Terminal 2:

```bash
cd /opt/worker && venv/bin/python -m worker
```

Result: `OSError: [Errno 98] Address already in use`

**Check who is using the port:**

```bash
ss -ltnp | grep 3000
```

Result: `node` is listening on 3000.

**Check what the worker received:**

```bash
env | grep PORT        # PORT=3000
```

**Meaning:** the worker's health server reads `PORT` (default 8000). Your shell exported `PORT=3000` for **every** program it starts, so both apps took 3000.

**Fix:**

1. Remove `export PORT=3000` from `~/.bashrc`. App settings don't belong in your shell.
2. Give each app its own clearly named variable: the worker uses `HEALTH_PORT`.
3. Keep each app's settings in its own env file (next step).

**Extra gotcha:** many `.env` loader libraries (Node `dotenv`, Python `python-dotenv`) **do not override** a variable that already exists. So even if the worker's `.env` said `PORT=8001`, the exported `PORT=3000` from your shell would win. Another reason not to export app settings globally.

---

## Step 2 — One env file per app, plus a shared one

Layout:

```
/etc/myapps/
├── common.env    # shared, no secrets      (644, root)
├── api.env       # API only, has secrets   (600, owner api)
└── worker.env    # worker only, has secrets (600, owner worker)
```

`common.env`:

```
APP_ENV=production
LOG_LEVEL=info
TZ=Asia/Colombo
```

`api.env`:

```
PORT=3000
DATABASE_URL=postgres://api_rw:ApiPass@10.0.0.5:5432/appdb
API_KEY=abc123
```

`worker.env`:

```
HEALTH_PORT=8001
LOG_LEVEL=debug
DATABASE_URL=postgres://backup_ro:BkPass@10.0.0.5:5432/appdb
SMTP_PASSWORD=mailpass
API_BASE_URL=http://127.0.0.1:3000
```

Keep the format simple: `KEY=value`, no `export`, no spaces around `=`. Then both bash and systemd can read the same file.

Permissions:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin api
sudo useradd --system --no-create-home --shell /usr/sbin/nologin worker

sudo chown api:api /etc/myapps/api.env       && sudo chmod 600 /etc/myapps/api.env
sudo chown worker:worker /etc/myapps/worker.env && sudo chmod 600 /etc/myapps/worker.env
sudo chmod 644 /etc/myapps/common.env
```

### Running an app by hand without polluting your shell

```bash
( set -a; . /etc/myapps/common.env; . /etc/myapps/api.env; set +a
  exec node /opt/api/server.js )
```

What each part does:

| Part | Purpose |
|---|---|
| `( ... )` | A subshell (Note 1, Phase 4.1). Variables loaded here **vanish** when it ends, so your shell stays clean |
| `set -a` | Auto-export every variable defined after this. The files have no `export`, so without this node wouldn't see them |
| `. file` | Load the file into the subshell (same as `source`) |
| `exec node` | Replace the subshell with node (Note 1, Phase 4.2). No extra shell process |

Check your shell afterwards:

```bash
echo "$API_KEY"       # (empty) — good, the secret didn't leak into your shell
```

### Which value wins?

`LOG_LEVEL` is in both `common.env` (`info`) and `worker.env` (`debug`). When files are loaded in order, **the later one wins**. Loading `common.env` first then `worker.env` gives the worker `debug`. That's the pattern: shared defaults first, app-specific overrides after.

---

## Step 3 — The Python virtualenv is just an env change

```bash
cd /opt/worker
which python             # /usr/bin/python
echo "$VIRTUAL_ENV"      # (empty)

source venv/bin/activate

which python             # /opt/worker/venv/bin/python
echo "$VIRTUAL_ENV"      # /opt/worker/venv
echo "$PATH"             # /opt/worker/venv/bin:/usr/local/bin:/usr/bin:...
```

**Meaning:** `activate` does nothing magical. It:
- puts `venv/bin` at the **front of PATH**, so `python` and `pip` resolve there first,
- sets `VIRTUAL_ENV`,
- changes your prompt and adds a `deactivate` function.

That's why it must be **sourced** (Note 1, Phase 6): it has to change your current shell. Running `bash venv/bin/activate` would change a child and do nothing for you.

nvm works the same way: `nvm use 20` changes PATH in your current shell.

**Conclusion for services and cron:** you don't need `activate` at all. Call the venv's python by full path:

```bash
/opt/worker/venv/bin/python -m worker
```

Python finds the venv's packages from the binary's location.

---

## Step 4 — Running the API with `sudo` (port 80)

```bash
sudo node /opt/api/server.js
```

### Problem 1: `sudo: node: command not found`

```bash
which node             # /home/thirana/.nvm/versions/node/v20.11.0/bin/node
sudo env | grep PATH   # PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

**Meaning:** sudo ignores your PATH and uses `secure_path` from `/etc/sudoers`. Your nvm folder isn't in it.

### Problem 2: the app can't see `DATABASE_URL`

```bash
sudo env | grep DATABASE_URL     # (nothing)
```

**Meaning:** sudo **resets the environment** (`Defaults env_reset`). Only a few safe variables survive.

Ways to pass variables (if sudoers allows):

```bash
sudo -E cmd                          # keep your whole environment
sudo --preserve-env=DATABASE_URL cmd # keep only listed variables
sudo DATABASE_URL="..." cmd          # set one
```

**Better fix:** don't run apps as root at all. Keep the API on port 3000 and put nginx in front on port 80, or allow node to bind low ports:

```bash
sudo setcap 'cap_net_bind_service=+ep' /usr/bin/node
```

The proper way to run both apps is systemd (next step).

---

## Step 5 — Both apps as systemd services

For services, install Node system-wide (e.g. `/usr/bin/node`), because the `api` user has no nvm.

`/etc/systemd/system/api.service`:

```ini
[Unit]
Description=API
After=network.target

[Service]
User=api
WorkingDirectory=/opt/api
EnvironmentFile=/etc/myapps/common.env
EnvironmentFile=/etc/myapps/api.env
ExecStart=/usr/bin/node /opt/api/server.js
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Problem 1: first try at the worker unit fails

You copy what you do in the terminal:

```ini
ExecStart=source /opt/worker/venv/bin/activate && python -m worker
```

```bash
sudo systemctl start worker
systemctl status worker
```

Result: the service fails to start, saying the executable is not valid / not found.

**Meaning:** systemd runs `ExecStart` **directly with exec, not through a shell**. `source` is a bash built-in (Note 1, Phase 3), not a program, and `&&` is shell syntax. There is no shell to understand them.

**Fix:** use the venv's python directly (Step 3):

`/etc/systemd/system/worker.service`:

```ini
[Unit]
Description=Worker
After=network.target api.service

[Service]
User=worker
WorkingDirectory=/opt/worker
EnvironmentFile=/etc/myapps/common.env
EnvironmentFile=/etc/myapps/worker.env
ExecStart=/opt/worker/venv/bin/python -m worker
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now api worker
```

### Problem 2: "Is the right env loaded?"

```bash
systemctl show worker -p Environment
```

Result: `Environment=` (empty)

**Meaning:** `systemctl show -p Environment` only lists `Environment=` lines from the unit. It does **not** show values from `EnvironmentFile=`. Empty here doesn't mean the files failed.

**Check the real running process instead:**

```bash
PID=$(systemctl show worker -p MainPID --value)
sudo cat /proc/$PID/environ | tr '\0' '\n' | sort
```

**What this is:** `/proc/<PID>/environ` holds the environment the process got at start. Entries are separated by null bytes (`\0`), so `tr` turns them into new lines.

Result includes:

```
API_BASE_URL=http://127.0.0.1:3000
DATABASE_URL=postgres://backup_ro:...
HEALTH_PORT=8001
LOG_LEVEL=debug
TZ=Asia/Colombo
...
```

**Conclusion:** the worker got `common.env` + `worker.env`, and `LOG_LEVEL=debug` won because `worker.env` is listed later.

systemd order rules:
- Multiple `EnvironmentFile=` → read in order, **later wins**.
- `EnvironmentFile=` values **override** `Environment=` values.
- Values are taken mostly literally — don't rely on `$OTHER_VAR` references inside these files.

Notice: the worker has `DATABASE_URL` with the `backup_ro` user and the API has it with `api_rw`. Same name, different values, separate processes (Phase 1.3).

---

## Step 6 — Change one app's env, restart only that app

You want the worker less noisy. Edit `worker.env`: `LOG_LEVEL=warn`.

```bash
sudo cat /proc/$(systemctl show worker -p MainPID --value)/environ | tr '\0' '\n' | grep LOG_LEVEL
```

Result: `LOG_LEVEL=debug` — still the old value.

**Meaning:** the environment is read **at start** (Phase 1.4). Editing the file changes nothing for the running process.

```bash
sudo systemctl restart worker
# check again → LOG_LEVEL=warn
```

The API wasn't touched and kept running. Separate env files + separate services = you can change one app without restarting the other.

(If you edit a **unit file** instead of an env file, run `sudo systemctl daemon-reload` before restarting.)

---

## Step 7 — Isolation: can one app see the other's secrets?

### Reading another process's environment

As the `worker` user, try to read the API's environment:

```bash
API_PID=$(systemctl show api -p MainPID --value)
sudo -u worker cat /proc/$API_PID/environ
```

Result: `Permission denied`

```bash
sudo cat /proc/$API_PID/environ | tr '\0' '\n' | grep API_KEY   # root can
```

**Meaning:** a process's environment is readable only by **the same user** and **root**. Because the apps run as different users (`api`, `worker`), a bug or break-in in the worker can't simply read `API_KEY`.

If both ran as the same user (say `thirana`), each could read the other's secrets.

### Child processes inherit everything

The API calls an image tool:

```js
const { spawn } = require('child_process');
spawn('convert', ['in.png', 'out.jpg']);
```

By default, `convert` gets a copy of the API's **entire** environment, including `API_KEY` and `DATABASE_URL`. It doesn't need them.

**Fix:** pass only what the child needs:

```js
spawn('convert', ['in.png', 'out.jpg'], { env: { PATH: process.env.PATH } });
```

```python
# Python worker calling pg_dump: pass only what it needs
subprocess.run(["pg_dump", url], env={"PATH": os.environ["PATH"]})
```

From a shell script:

```bash
env -i PATH=/usr/bin:/bin convert in.png out.jpg
```

---

## Step 8 — The worker's nightly backup in cron

Cron job for the `worker` user:

```bash
sudo crontab -u worker -e
```

```
0 2 * * * /opt/worker/backup.sh
```

First version of `backup.sh`:

```bash
#!/bin/bash
python -m worker.backup
```

Next day: failed.

**Check cron's environment** — temporary job:

```
* * * * * env > /tmp/cron-env.txt
```

```
HOME=/nonexistent
LOGNAME=worker
PATH=/usr/bin:/bin
SHELL=/bin/sh
```

**Meaning:**
- No `DATABASE_URL`, no `LOG_LEVEL` — cron doesn't read env files or startup files.
- `python` is `/usr/bin/python` (or missing), not the venv's.
- (Shell side of cron — `sh` vs `bash` — is Note 1, step 6.)

**Reproduce cron's world by hand:**

```bash
sudo -u worker env -i PATH=/usr/bin:/bin SHELL=/bin/sh /opt/worker/backup.sh
```

**Fix:** make the script load exactly what it needs:

```bash
#!/bin/bash
set -euo pipefail

export PATH=/usr/bin:/bin

set -a
. /etc/myapps/common.env
. /etc/myapps/worker.env
set +a

: "${DATABASE_URL:?DATABASE_URL not set}"

exec /opt/worker/venv/bin/python -m worker.backup
```

Remove the temporary `env` cron job after checking.

(Alternative: run the backup as a **systemd timer** with the same `EnvironmentFile=` lines. Then cron and the service can't drift apart.)

---

## Step 9 — Both apps in Docker Compose

`compose.yaml`:

```yaml
services:
  api:
    image: myorg/api:${API_VERSION}
    env_file:
      - common.env
      - api.env
    environment:
      PORT: "3000"

  worker:
    image: myorg/worker:${WORKER_VERSION}
    env_file:
      - common.env
      - worker.env
    environment:
      API_BASE_URL: "http://api:3000"
```

### Problem 1: worker can't reach the API

`worker.env` says `API_BASE_URL=http://127.0.0.1:3000`. Inside the worker container, `127.0.0.1` is the **worker container itself**, not the API.

**Fix:** use the Compose service name: `http://api:3000`. The value is set in `environment:`, which **overrides** `env_file:` in Compose.

Same variable, different correct value depending on where the app runs. That's exactly why it's an env variable and not hardcoded.

### Problem 2: two different `.env` ideas in Compose

| Thing | Used for | Reaches the container? |
|---|---|---|
| `.env` file next to `compose.yaml` | Filling in `${API_VERSION}` in the compose file itself | **No** (only if you reference it) |
| `env_file:` | Variables **inside** the container | **Yes** |
| `environment:` | Variables inside the container, overrides `env_file:` | **Yes** |

People often put app secrets in the project `.env` and wonder why the app doesn't see them.

**Check what Compose will really use:**

```bash
docker compose config              # compose file with ${...} filled in
docker compose exec worker env     # real env inside the running container
```

### Secrets in images

```dockerfile
ENV API_KEY=abc123        # DON'T — stays in the image layers forever
```

Anyone with the image can see it:

```bash
docker history myorg/api
docker image inspect myorg/api
```

Pass secrets at run time (`env_file:`), or better, as mounted secret files or from a secret manager. On GCP Cloud Run, `--set-secrets DATABASE_URL=db-url:latest` injects from Secret Manager instead of storing the value in the service config.

---

## Scenario recap

| Step | What happened | Concept |
|---|---|---|
| 1 | Both apps grabbed port 3000 | Global `export` in `~/.bashrc` reaches every program; dotenv doesn't override |
| 2 | Per-app env files | Separate files, `set -a`, subshell + `exec`, later file wins |
| 3 | `activate` | Just a PATH + variable change; must be sourced; not needed for services |
| 4 | `sudo` broke PATH and variables | `env_reset`, `secure_path` |
| 5 | `source` in `ExecStart` failed; `show` looked empty | systemd uses no shell; check `/proc/<PID>/environ` |
| 6 | New value not used until restart | Env is read at start; restart one service only |
| 7 | Worker can't read API's env; child tools inherit secrets | Separate users; pass a minimal env to children |
| 8 | Backup failed in cron | Tiny cron env; script loads its own |
| 9 | Worker couldn't reach API in Compose | Same name, different value per context; Compose `.env` vs `env_file:` |

---

# Phase 5 — Troubleshooting: "why did my variable disappear?"

| Symptom | Likely cause | How to check | Fix |
|---|---|---|---|
| Gone in a new terminal/SSH session | Only set with `export` in the old shell | `grep VAR ~/.bashrc ~/.profile` | Put it in a startup file (or the app's env file) |
| `echo $VAR` works, the program doesn't see it | Not exported | `env \| grep VAR` | `export VAR` or `set -a` before loading |
| Set inside a script, gone after | Script ran as a child | `type` of how you ran it | `source script` |
| Set inside a loop/pipe, gone after | Pipe subshell (Note 1, step 5) | `echo $BASHPID` inside | `while ...; done < file` |
| Works in terminal, not in systemd | Services don't read shell files | `cat /proc/<PID>/environ` | `EnvironmentFile=` |
| Works in terminal, not in cron | Tiny cron env | `env > /tmp/cron-env.txt` in cron | Load env inside the script |
| Missing with `sudo` | `env_reset` | `sudo env` | `--preserve-env=VAR`, or avoid sudo for apps |
| Edited the file, app still uses old value | Env read at start | `/proc/<PID>/environ` | Restart (and `daemon-reload` for unit changes) |
| Wrong value, defined in two places | Load order | Check order of files | Later `EnvironmentFile=` wins; Compose `environment:` beats `env_file:` |
| `.env` value ignored | Already exported in the shell; dotenv won't override | `env \| grep VAR` before starting | Remove the global export |
| Container has a different value than expected | Compose `.env` vs `env_file:` confusion | `docker compose config`, `docker compose exec svc env` | Put container vars in `env_file:` / `environment:` |
| `command not found` with sudo/cron/service | Different PATH | `sudo env`, cron env dump | Full paths, or set PATH in the script |

---

# Phase 6 — Security

## 6.1 Who can read a process's environment?

| Method | Who can read |
|---|---|
| `/proc/<PID>/environ` | Same user and root |
| `ps eww <PID>` | Your own processes |
| `systemctl show <svc> -p Environment` | **Any user** (only `Environment=` lines) |
| `docker inspect` | Anyone with Docker access (effectively root) |
| Logs / CI output with `env` dumps | Anyone who can read the logs |

Env variables are convenient and acceptable for secrets on a well-run server, but they are **not a vault**.

Command-line arguments are **worse**: any user can see them with `ps aux`.

```bash
mysql -p'S3cret' ...      # visible to every user
```

Order from worst to best: arguments → env variables → protected files → secret manager.

## 6.2 How secrets leak through env

1. **Logs** — `console.log(process.env)`, `print(os.environ)`, `env` in CI steps.
2. **Error pages** — frameworks in debug mode print the whole environment.
3. **Child processes** — inherit everything (Step 7).
4. **Crash dumps** — contain process memory.
5. **Shell history** — `export API_KEY=abc123` is saved in `~/.bash_history`.
   ```bash
   export HISTCONTROL=ignorespace   # commands starting with a space aren't saved
    export API_KEY=abc123           # note the leading space
   read -s API_KEY && export API_KEY # type without showing or saving
   ```
6. **Git** — committed `.env`. Add it to `.gitignore`; commit `.env.example` with fake values.
7. **Docker images** — `ENV` with secrets stays in the layers.
8. **Error-tracking tools** that send environment data.
9. **Global exports** — a secret in `~/.bashrc` reaches every program you run.

## 6.3 Protecting secrets with multiple apps

- **One user per app** (`api`, `worker`). Then `/proc/<PID>/environ` protects each app from the other.
- **One env file per app**, `chmod 600`, owned by that app's user. Shared non-secret settings in a separate file.
- **Least privilege values**: different DB users per app (`api_rw`, `backup_ro`). Same variable name, smallest possible rights.
- **Minimal env for child processes** (`env: {PATH: ...}`, `env -i`).
- In systemd, secrets in `EnvironmentFile=`, not `Environment=` (visible via `systemctl show`).
- Better: **systemd credentials** (`LoadCredential=`) — the secret is given as a file, not an env variable.
- Best: a **secret manager** (GCP Secret Manager, Vault), fetched at start or injected by the platform.
- Rotate any secret that shows up in a log, image, or repo.

## 6.4 PATH hijacking

If someone can write to a folder that comes **early** in PATH, they can place a fake `ls`, `node`, or `python` there.

Bad:

```bash
export PATH=".:$PATH"            # current folder first
export PATH="/tmp/tools:$PATH"   # world-writable folder
```

Rules:
- No `.`, empty entries (`::`), or world-writable folders in PATH.
- Scripts run by root, cron, or services: set PATH at the top, or use full paths.
- `#!/usr/bin/env python3` trusts PATH. For root-owned scripts, use an absolute path.
- A virtualenv or nvm folder at the front of PATH is fine **only if** it's owned by a trusted user and not writable by others.

Check for writable PATH folders:

```bash
echo "$PATH" | tr ':' '\n' | xargs -I{} sh -c 'test -w "{}" && echo "writable: {}"'
```

## 6.5 Dangerous variables (why sudo resets the environment)

Some variables change **what code a program loads**. Passed into a root process, they give root.

| Variable | Risk |
|---|---|
| `LD_PRELOAD` | Loads an attacker's library first |
| `LD_LIBRARY_PATH` | Loads fake libraries |
| `PATH` | Runs fake commands |
| `BASH_ENV` | Bash runs this file before any non-interactive script |
| `PYTHONPATH`, `NODE_OPTIONS`, `PERL5LIB` | Load attacker's code into interpreters |
| `IFS` | Changes how shells split words |

That's why sudo has `env_reset` and `secure_path` on by default, and `LD_*` variables are removed for setuid programs.

Sudoers settings:

```
Defaults env_reset
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults env_keep += "LANG"        # keep only safe, needed variables
```

Never add `LD_PRELOAD`, `PATH`, `PYTHONPATH`, etc. to `env_keep`. Be careful giving `SETENV:` or `sudo -E` to users with limited sudo rights — they can pass these variables to the allowed command.

## 6.6 Security checklist

| Check | Why |
|---|---|
| No app settings or secrets in `~/.bashrc`, `/etc/environment`, Dockerfile `ENV` | Reaches everything / permanent |
| Each app runs as its own non-root user | Isolates `/proc/<PID>/environ` |
| Each app's env file is `600`, owned by its user, in `.gitignore` | Prevent reads and commits |
| Different credentials per app, least privilege | Limits damage if one leaks |
| Children get a minimal env | Tools don't inherit secrets |
| Secrets via `EnvironmentFile=`, credentials, or a secret manager | Not visible in `systemctl show` |
| No env dumps in logs, CI, or error pages | Prevent leaks |
| No `.` or writable folders in PATH; scripts set PATH | Prevent hijacking |
| `env_reset` on; no risky `env_keep` / `SETENV` | Prevent privilege escalation |
| Secrets never passed as command arguments | Visible to all users in `ps` |

---

# Summary table

| Concept | Key point |
|---|---|
| Environment variable | `KEY=value` string in a process's environment list |
| Per process | Each process has its own copy; same name can differ per app |
| Inheritance | Children get a copy of exported variables; can't change the parent |
| Read at start | Changing a file needs a restart |
| Shell vs env variable | Only `export`ed variables reach programs |
| `VAR=x cmd` | Variable for one command only |
| `env` / `env -i` | Show env / run with an empty env |
| `set -a; . file; set +a` | Load and export a `KEY=value` file |
| `( ...; exec app )` | Load env and start an app without polluting your shell |
| Load order | Later file wins; systemd `EnvironmentFile=` beats `Environment=`; Compose `environment:` beats `env_file:` |
| virtualenv / nvm | Just PATH changes; must be sourced; use full paths in services |
| sudo | Resets env, uses `secure_path` |
| systemd | Clean env; `EnvironmentFile=`; no shell in `ExecStart` |
| cron | Tiny env; script loads its own |
| Docker Compose | `env_file:` / `environment:` for containers; project `.env` only fills `${...}` |
| `/proc/<PID>/environ` | Real env of a running process; same user + root only |
| Isolation | Separate users, separate files, least privilege, minimal child env |
| Security | Env leaks easily; protect files, avoid dumps, guard PATH and `LD_*` |

---

# Appendix — Command reference

## Viewing

| Command | Purpose |
|---|---|
| `env` / `printenv` | List environment variables |
| `printenv VAR` / `echo "$VAR"` | One variable |
| `env \| sort` / `env \| grep VAR` | Sorted / filtered |
| `set` | All shell + env variables + functions |
| `export -p` | Exported variables |
| `echo "$PATH" \| tr ':' '\n'` | PATH one folder per line |
| `which <cmd>` / `type -a <cmd>` | Which file or function runs |
| `sudo env` | Env sudo gives |
| `systemctl show <svc> -p Environment` | Only `Environment=` lines of a unit |
| `systemctl show <svc> -p MainPID --value` | Main PID of a service |
| `sudo cat /proc/<PID>/environ \| tr '\0' '\n'` | Real env of a running process |
| `docker compose config` | Compose file with `${...}` filled in |
| `docker compose exec <svc> env` | Env inside a container |
| `ss -ltnp` | Which process is on which port |

## Setting and removing

| Command | Purpose |
|---|---|
| `VAR=value` | Shell variable only |
| `export VAR=value` | Environment variable |
| `VAR=value cmd` | One command only |
| `export -n VAR` / `unset VAR` | Un-export / delete |
| `readonly VAR` | Lock in this shell |
| `export PATH="$HOME/bin:$PATH"` | Add to PATH safely |
| `set -a; . file; set +a` | Load and export a file |
| `source venv/bin/activate` / `deactivate` | Enter / leave a virtualenv |

## Running with a controlled environment

| Command | Purpose |
|---|---|
| `env VAR=x cmd` | Add/override for a command |
| `env -u VAR cmd` | Remove one variable |
| `env -i PATH=/usr/bin:/bin cmd` | Clean env with only PATH |
| `( set -a; . a.env; . b.env; set +a; exec app )` | Start an app with files loaded, shell stays clean |
| `sudo -u worker env -i PATH=/usr/bin:/bin SHELL=/bin/sh ./script.sh` | Simulate cron as a user |
| `sudo --preserve-env=VAR cmd` / `sudo -E cmd` | Keep variables with sudo (if allowed) |

## systemd

| Line / command | Purpose |
|---|---|
| `Environment=KEY=value` | Set a variable in the unit (visible via `systemctl show`) |
| `EnvironmentFile=/path/file` | Load a `KEY=value` file; multiple allowed, later wins |
| `LoadCredential=name:/path` | Give a secret as a file instead of env |
| `sudo systemctl daemon-reload` | After editing a unit file |
| `sudo systemctl restart <svc>` | Apply env changes |

## Docker

| Item | Purpose |
|---|---|
| `docker run -e VAR=x` / `--env-file file` | Pass env to a container |
| Compose `env_file:` | Env files for a service |
| Compose `environment:` | Values for a service, overrides `env_file:` |
| Project `.env` | Fills `${...}` in `compose.yaml` only |
| `docker history` / `docker image inspect` | Check images for baked-in `ENV` |

## Shell expansion helpers

| Syntax | Meaning |
|---|---|
| `"$VAR"` | Value (always quote) |
| `${VAR:-default}` | Default if unset or empty |
| `${VAR:?message}` | Error if unset or empty |
| `: "${VAR:?missing}"` | Required-variable check at the top of a script |

## Security helpers

| Command | Purpose |
|---|---|
| `chmod 600 file` / `chown user:user file` | Lock an env file to one app user |
| `export HISTCONTROL=ignorespace` | Leading-space commands aren't saved |
| `read -s VAR && export VAR` | Enter a secret without showing or saving it |
| `sudo visudo` | Check `env_reset`, `secure_path`, `env_keep` |
| `echo "$PATH" \| tr ':' '\n' \| xargs -I{} sh -c 'test -w "{}" && echo "writable: {}"'` | Find writable PATH folders |