# Lab Setup with Multipass — Practice (Note 0)

Building `app-server-01` on your Mac, so you can run every practice note on a real Ubuntu server and break it safely. Each step shows the command, what the result means, and why it's done this way.

**Theory for this note:** Note 0 (Introduction). **Continues into:** Note 1 — when you finish this note, the server is in the state Note 1 starts from.

---

## Contents

- The setup
- Step 0 — Check Multipass on your Mac
- Step 1 — A dedicated SSH key for the lab
- Step 2 — The cloud-init file
- Step 3 — Launch `app-server-01`
- Step 4 — Give `thirana` a password
- Step 5 — SSH in from your Mac
- Step 6 — Compare the starting state with the notes
- Step 7 — Base tools
- Step 8 — Node with nvm (Note 1's setup)
- Step 9 — The API stand-in
- Step 10 — The worker stand-in
- Step 11 — Snapshot the clean lab
- Step 12 — Working through the notes
- Step 13 — Extras, installed when a note needs them
- Step 14 — Logging in as the other people
- Where the lab differs from the notes
- Lab safety
- Troubleshooting table
- Appendix — Command reference

---

# The setup

```
┌────────────────────── your Mac ──────────────────────┐
│                                                      │
│   Terminal ── ssh thirana@app-server-01 ──┐          │
│                                           ▼          │
│   multipass ──────────────▶ ┌──────────────────────┐ │
│   (launch, stop, snapshot)  │ VM: app-server-01    │ │
│                             │ Ubuntu 24.04 LTS     │ │
│                             │ 2 CPUs, 4 GB, 20 GB  │ │
│                             │ users: thirana (you) │ │
│                             │        ubuntu (mp)   │ │
│                             └──────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

**Why Multipass:**

| Need | Multipass gives |
|---|---|
| A real Ubuntu server, not a container | A full VM with its own kernel and systemd as PID 1 — Notes 11–14 need this |
| The same setup as a cloud VM | It's configured with **cloud-init**, as on GCP or AWS |
| Mistakes you can undo | **Snapshots**: go back to before any note |
| Fast rebuilds | A fresh server in about a minute |

**Why not Docker for the lab:** a container has no systemd, no real boot, and shares your Mac's Docker VM kernel. Half the series wouldn't work.

**What you'll end up with:** the server Note 1 assumes — Ubuntu 24.04, you logged in as `thirana` over SSH, Node installed with nvm, the API at `/opt/api`, and the worker at `/opt/worker`.

---

# Step 0 — Check Multipass on your Mac

```bash
multipass version
multipass list
```
```
multipass   1.16.4+mac
multipassd  1.16.4+mac
Name                    State             IPv4             Image
phantom-lab             Stopped           --               Ubuntu 24.04 LTS
```

**Meaning:** Multipass is installed and its background service (`multipassd`) is running. Any existing VMs are listed. They're separate machines and won't be touched.

```bash
multipass find
```
```
Image                       Aliases           Version          Description
22.04                       jammy             20261020         Ubuntu 22.04 LTS
24.04                       noble             20261017         Ubuntu 24.04 LTS
26.04                       resolute,lts      20261016         Ubuntu 26.04 LTS
```

**Meaning:** the `lts` alias (and the default when you don't name an image) is now **26.04**. The series uses **24.04**, so always ask for `24.04` by name. Otherwise package versions, paths, and outputs won't match the notes. (Why: Note 0, Phase 1.4.)

---

# Step 1 — A dedicated SSH key for the lab

```bash
ls ~/.ssh/*.pub
```
```
/Users/thiranaembuldeniya/.ssh/google_compute_engine.pub
```

**Meaning:** the only key there is for GCP. Make a separate key just for the lab rather than reusing it:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/lab_ed25519 -C "thirana@lab"
```

A passphrase is optional for a lab key. If you set one, macOS can remember it in the keychain (`ssh-add --apple-use-keychain ~/.ssh/lab_ed25519`).

**Why a separate key:** one key per purpose. You can throw the lab key away, rebuild, or share the lab setup without ever involving the key that opens your real cloud servers. It's the same idea as role groups in Note 4: access you can remove without side effects.

---

# Step 2 — The cloud-init file

**cloud-init** runs on the VM's first boot and sets it up from a YAML file — users, keys, packages. Cloud providers use the same tool.

```bash
mkdir -p ~/labs/app-server-01
cat > ~/labs/app-server-01/cloud-init.yaml <<EOF
#cloud-config
users:
  - name: thirana
    uid: 1000
    gecos: Thirana
    shell: /bin/bash
    groups: [sudo, adm]
    ssh_authorized_keys:
      - $(cat ~/.ssh/lab_ed25519.pub)
  - default
EOF
cat ~/labs/app-server-01/cloud-init.yaml
```

The `$(cat ...)` puts your public key into the file, so check that the `ssh-ed25519 AAAA... thirana@lab` line is there.

| Line | Why |
|---|---|
| `name: thirana`, `uid: 1000` | The notes expect `thirana` to be UID 1000, the first human account (Note 4, Step 0) |
| `shell: /bin/bash` | The default for new users is `/bin/sh` (Note 4, Step 0) |
| `groups: [sudo, adm]` | Admin rights and log reading, as in Note 4's plan |
| `ssh_authorized_keys` | Your lab key, so you can SSH in. SSH stays key-only |
| No `sudo:` line | cloud-init would otherwise write a `NOPASSWD:ALL` rule for `thirana`. Leaving it out means `sudo` asks for your password, as in every note |
| `- default`, **after** `thirana` | Also creates Multipass's own user, `ubuntu`. Listed second, it gets UID **1001**, leaving 1000 for you |

### Why `ubuntu` must stay

Multipass runs `multipass shell`, `exec`, and `transfer` by SSH-ing into the VM as `ubuntu`. It's tempting to rename the default user to `thirana` (`system_info: default_user: name: thirana`). **Don't.** Multipass then can't log in, and `launch` fails after several minutes with:

```
launch failed: The following errors occurred:
app-server-01: timed out waiting for response
```

---

# Step 3 — Launch `app-server-01`

```bash
multipass launch 24.04 \
  --name app-server-01 \
  --cpus 2 --memory 4G --disk 20G \
  --cloud-init ~/labs/app-server-01/cloud-init.yaml
```
```
Launched: app-server-01
```

| Option | Why |
|---|---|
| `24.04` | The release the series uses (Step 0) |
| `--name app-server-01` | Also becomes the VM's hostname, matching the notes |
| `--cpus 2` | So CPU limits and `nice` (Notes 9, 14) have something to share |
| `--memory 4G` | Note 14 sets memory limits up to 1800 MB for workers; 4 GB leaves room for the rest |
| `--disk 20G` | Room for the journal (Note 14 caps it at 1 GB and keeps 5 GB free). The disk grows as used; it doesn't take 20 GB at once |

The first launch downloads the image (a few hundred MB); after that, a launch takes about a minute.

```bash
multipass info app-server-01
```
```
Name:           app-server-01
State:          Running
Snapshots:      0
IPv4:           192.168.252.5
Release:        Ubuntu 24.04.5 LTS
CPU(s):         2
Disk usage:     2.0GiB out of 19.3GiB
Memory usage:   351.2MiB out of 3.8GiB
```

**Meaning:** the VM is up, on a private network between your Mac and the VM. **Note the IPv4 address** — you need it in Step 5. Yours may differ.

---

# Step 4 — Give `thirana` a password

`thirana` has an SSH key but no password yet, so `sudo` can't work. Log in the Multipass way, as `ubuntu`, and set one:

```bash
multipass shell app-server-01
```
```bash
whoami                     # ubuntu
sudo passwd thirana        # ubuntu's sudo doesn't ask for a password; you type thirana's new one twice
sudo passwd -S thirana     # thirana P ...   ← P = has a usable password (it was L, locked)
exit
```

Pick a lab-only password, never one you use anywhere else.

**Meaning:** this is the **only** time you need the `ubuntu` user. From now on, work as `thirana` over SSH. If you work inside `multipass shell`, you're `ubuntu` (UID 1001, passwordless sudo), and outputs won't match the notes.

---

# Step 5 — SSH in from your Mac

Add a host entry to `~/.ssh/config` on your **Mac**, using the IP from Step 3:

```
Host app-server-01
    HostName 192.168.252.5
    User thirana
    IdentityFile ~/.ssh/lab_ed25519
    IdentitiesOnly yes
```

| Setting | Why |
|---|---|
| `Host app-server-01` | Lets you type `ssh app-server-01`, as the notes do |
| `User thirana` | The default login name; `ssh nimal@app-server-01` still overrides it |
| `IdentitiesOnly yes` | Offer only the lab key, not every key you have (e.g. the GCP one) |

```bash
ssh app-server-01
```
```
The authenticity of host '192.168.252.5' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
thirana@app-server-01:~$
```

**Meaning:** first contact — SSH records the VM's host key in `~/.ssh/known_hosts`. If you **rebuild** the VM later, it gets a new host key (and maybe a new IP), and SSH will warn loudly. That's expected for a rebuilt lab; see the troubleshooting table.

Check that sudo asks for a password:

```bash
sudo -k; sudo -n true
```
```
sudo: a password is required
```

**Meaning:** `thirana`'s sudo behaves like the notes': full admin, through the `sudo` group, with a password. (Why: Note 7, Phase 4.)

---

# Step 6 — Compare the starting state with the notes

Run the checks from Note 4, Step 0 now, so you know the lab's differences before they surprise you.

```bash
getent passwd | awk -F: '$3 >= 1000 && $3 < 60000 {print $1, $3, $7}'
```
```
thirana 1000 /bin/bash
ubuntu 1001 /bin/bash
```

**Meaning:** you're UID 1000, as in the notes. `ubuntu` is the one extra account (Step 2).

```bash
getent group sudo adm lxd
```
```
sudo:x:27:thirana,ubuntu
adm:x:4:syslog,thirana,ubuntu
lxd:x:104:ubuntu
```

**Meaning:** `ubuntu` is also in `sudo`, `adm`, and `lxd` (`lxd` is root-equivalent, like `docker` — Note 3, Phase 4.5). Notes 4 and 8 show only `thirana` here.

```bash
sudo cat /etc/sudoers.d/90-cloud-init-users
```
```
# User rules for ubuntu
ubuntu ALL=(ALL) NOPASSWD:ALL
```

**Meaning:** `ubuntu` can become root **without a password**. That's how Multipass manages the VM. In a real audit (Note 8, Step 6) this line is a serious finding. In the lab, it's the known back door: leave it and don't copy it anywhere real.

```bash
sudo passwd -S root
sudo sshd -T | grep -iE 'permitrootlogin|passwordauthentication'
```
```
root L ...
permitrootlogin without-password
passwordauthentication no
```

**Meaning:** same as the notes: root's password is locked, root can still log in with a key (Note 4 fixes this in Step 6), and passwords are off for SSH.

```bash
ls -ld /var/log/journal
```
```
drwxr-sr-x+ 3 root systemd-journal 4096 ... /var/log/journal
```

**Meaning:** this image already keeps the journal on disk. Note 14, Step 2 starts from a server where it **doesn't** — see "Where the lab differs from the notes".

**Conclusion:** the lab matches the notes except for the `ubuntu` account and its rights. Keep the full list (later in this note) in mind when an output has an extra line.

---

# Step 7 — Base tools

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y acl python3-venv
```

| Package | Needed for |
|---|---|
| `acl` | `getfacl` / `setfacl` (Note 5, Phase 9; Note 6) — **not** installed by default |
| `python3-venv` | The worker's virtualenv (Step 10) |

Already on the image: `man` pages, `strace`, `tmux`, `htop`, `jq`, `git`, `curl`, `python3`, `cron`, `rsyslog`.

**Not yet:** Node from `apt`, Docker, nginx, PostgreSQL. Notes 2 and 4 have steps that depend on them being **missing** at first. Step 13 says when to add each one.

If `apt upgrade` installed a new kernel, reboot so you're running it (Note 0, Phase 2.4):

```bash
ls /boot/vmlinuz-*; uname -r       # newest installed vs running
sudo reboot                        # only if they differ; then ssh back in
```

---

# Step 8 — Node with nvm (Note 1's setup)

Note 1 assumes Node was installed with **nvm**, for `thirana` only. Download the installer, **read it**, then run it:

```bash
curl -fsSLo /tmp/nvm-install.sh https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh
less /tmp/nvm-install.sh
bash /tmp/nvm-install.sh
```

**Why not `curl ... | bash`:** piping a download straight into a shell runs code you've never seen, as you. It's a common way to get compromised. Reading the script first is a good habit, even in a lab.

```bash
grep -n NVM ~/.bashrc
```
```
119:export NVM_DIR="$HOME/.nvm"
120:[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
121:[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```

**Meaning:** the installer added its loading lines to `~/.bashrc`, exactly what Note 1, Step 2 investigates. They only take effect in **new** shells, so load them now and install Node 20:

```bash
source ~/.bashrc
nvm install 20
which node; node -v
```
```
/home/thirana/.nvm/versions/node/v20.20.2/bin/node
v20.20.2
```

**Meaning:** Node lives in your home folder, so only you can use it. Note 1 shows `v20.11.0`; any 20.x is fine. That's the setup Note 1 Step 4 (`nvm` in scripts) and Note 2 Step 4 (`sudo node`) are about.

---

# Step 9 — The API stand-in

The notes describe the team's API but never show its code. This small stand-in behaves the way the notes need:

| Behaviour | Used in |
|---|---|
| Listens on `PORT` (default 3000) | Notes 1, 2, 8 |
| `/health` answers `ok` | Health checks (Notes 2, 12) |
| `/` shows its PID, the user it runs as, and **whether** `DATABASE_URL` / `API_KEY` are set (never their values) | Notes 2, 3, 7 |
| `/slow` takes 10 s to answer | Graceful shutdown (Notes 9, 10, 12) |
| `SIGTERM` → stop accepting, finish open requests, exit 0 | Note 9, Phase 8.2 |
| Logs requests to stdout | Logs (Notes 13, 14) |

Note 1's `deploy.sh` runs `git pull` and `npm ci` in `/opt/api`, so the folder is also a git repository with a `package-lock.json`.

```bash
sudo mkdir /opt/api
sudo chown thirana:thirana /opt/api
cd /opt/api
```

`/opt/api/server.js`:

```bash
cat > server.js <<'EOF'
// Stand-in for the team API. Reads PORT, answers on /health, stops cleanly on SIGTERM.
const http = require('http');
const os = require('os');

const port = Number(process.env.PORT || 3000);

const server = http.createServer((req, res) => {
  console.log(`${req.method} ${req.url}`);
  if (req.url === '/health') return res.end('ok\n');
  if (req.url === '/slow') return setTimeout(() => res.end('done\n'), 10000);   // a request still open at shutdown
  res.end(JSON.stringify({
    app: 'api',
    pid: process.pid,
    user: os.userInfo().username,
    env: { DATABASE_URL: 'DATABASE_URL' in process.env, API_KEY: 'API_KEY' in process.env },   // set or not, never the value
  }) + '\n');
});

server.listen(port, () => console.log(`api listening on ${port}`));

process.on('SIGTERM', () => {
  console.log('SIGTERM: stopping');
  server.close(() => process.exit(0));                     // stop new connections, wait for open ones
  setTimeout(() => process.exit(1), 25000).unref();        // safety net
});
EOF
```

`/opt/api/package.json`:

```bash
cat > package.json <<'EOF'
{
  "name": "api",
  "version": "1.0.0",
  "private": true,
  "main": "server.js",
  "scripts": { "start": "node server.js" }
}
EOF
npm install          # no dependencies; this only writes package-lock.json, which npm ci needs
```

Make it a git repository, with a local "remote" so `git pull` works:

```bash
git config --global user.name "Thirana"
git config --global user.email "thirana@app-server-01"
git config --global init.defaultBranch main

printf 'node_modules/\n' > .gitignore
git init -q
git add . && git commit -qm "Stand-in API"
git init -q --bare ~/git/api.git
git remote add origin ~/git/api.git
git push -q -u origin main
```

Try it:

```bash
PORT=3000 API_KEY=test node server.js &
curl -s localhost:3000/
curl -s localhost:3000/health
kill %1
```
```
api listening on 3000
GET /
{"app":"api","pid":2714,"user":"thirana","env":{"DATABASE_URL":false,"API_KEY":true}}
GET /health
ok
SIGTERM: stopping
```

**Meaning:** it runs as `thirana` (you started it), sees `API_KEY` but not `DATABASE_URL`, and on `kill` (which sends SIGTERM) it shuts down cleanly. When systemd runs it as `api` in Note 2, `"user"` changes — that's Note 3's whole topic in one field.

---

# Step 10 — The worker stand-in

| Behaviour | Used in |
|---|---|
| `python -m worker`: a job loop, one job every `JOB_SECONDS` (default 5) | Notes 2, 9, 10, 12 |
| Health endpoint on `HEALTH_PORT` (default 8001) | Note 2, Step 1 |
| `SIGTERM` or `Ctrl+C` → finish the current job, then exit 0 | Note 9, Phase 8.3; Note 10, Step 4 |
| `python -m worker.backup`: `pg_dump` if PostgreSQL settings are in the environment, otherwise a fake 30-second dump | Notes 2, 6, 10, 14 |
| `python -m worker.reindex`: a long job (50 min); doesn't handle SIGHUP, so it dies when the terminal closes | Note 10, Step 2 |

```bash
sudo mkdir -p /opt/worker/worker
sudo chown -R thirana:thirana /opt/worker
cd /opt/worker
: > worker/__init__.py
```

`/opt/worker/worker/__main__.py`:

```bash
cat > worker/__main__.py <<'EOF'
"""Stand-in for the team worker: a job loop plus a health endpoint on HEALTH_PORT."""
import http.server
import os
import signal
import threading
import time

stopping = False


def on_term(signum, frame):
    global stopping
    stopping = True          # just set a flag; the main loop checks it


class Health(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"ok\n")

    def log_message(self, *args):
        pass


signal.signal(signal.SIGTERM, on_term)
signal.signal(signal.SIGINT, on_term)

port = int(os.environ.get("HEALTH_PORT", "8001"))
health = http.server.HTTPServer(("127.0.0.1", port), Health)
threading.Thread(target=health.serve_forever, daemon=True).start()
print(f"worker started, pid {os.getpid()}, health on {port}", flush=True)

job = 0
while not stopping:
    job += 1
    print(f"job {job}: start", flush=True)
    time.sleep(int(os.environ.get("JOB_SECONDS", "5")))   # the current job always finishes
    print(f"job {job}: done", flush=True)

print("stop requested: finished current job, exiting", flush=True)
EOF
```

`/opt/worker/worker/backup.py`:

```bash
cat > worker/backup.py <<'EOF'
"""Stand-in nightly backup: real pg_dump if PostgreSQL settings are in the environment, else a fake dump."""
import datetime
import os
import subprocess
import sys
import time

out_dir = os.environ.get("BACKUP_DIR", "/var/backups/appdb")
if not os.path.isdir(out_dir):
    sys.exit(f"backup: {out_dir} does not exist")
out = os.path.join(out_dir, f"db-{datetime.date.today()}.sql")

if "PGDATABASE" in os.environ:
    # Connection settings come from PGHOST/PGUSER/PGPASSWORD..., not arguments (Note 10, Step 10)
    with open(out, "w") as f:
        subprocess.run(["pg_dump"], stdout=f, check=True)
else:
    time.sleep(int(os.environ.get("BACKUP_SECONDS", "30")))
    with open(out, "w") as f:
        f.write("-- fake dump\n")

print(f"backup written to {out}", flush=True)
EOF
```

`/opt/worker/worker/reindex.py`:

```bash
cat > worker/reindex.py <<'EOF'
"""Stand-in long job: runs for REINDEX_MINUTES (default 50). No SIGHUP handling, so it dies with its terminal."""
import os
import time

total = int(os.environ.get("REINDEX_MINUTES", "50")) * 60
for done in range(0, total, 10):
    print(f"reindex: {done // 60} of {total // 60} min", flush=True)
    time.sleep(10)
print("reindex: finished", flush=True)
EOF
```

Create the virtualenv and try it:

```bash
python3 -m venv venv
JOB_SECONDS=2 venv/bin/python -m worker
```
```
worker started, pid 3120, health on 8001
job 1: start
job 1: done
job 2: start
^Cjob 2: done
stop requested: finished current job, exiting
```

**Meaning:** `Ctrl+C` (SIGINT) didn't cut job 2 off; the worker finished it, then exited. That's the graceful pattern from Note 9, Phase 8.3.

```bash
BACKUP_DIR=/tmp BACKUP_SECONDS=1 venv/bin/python -m worker.backup
```
```
backup written to /tmp/db-2026-10-22.sql
```

**Meaning:** the backup works when its folder exists. Without `BACKUP_DIR`, it uses `/var/backups/appdb`, which doesn't exist until Note 6. That's a real failure to read in the logs when cron runs it in Note 2.

---

# Step 11 — Snapshot the clean lab

Log out, then from your **Mac**:

```bash
multipass stop app-server-01
multipass snapshot app-server-01 --name note-00-ready --comment "clean lab: users, nvm, stand-in apps"
multipass start app-server-01
multipass list --snapshots
```
```
Snapshot taken: app-server-01.note-00-ready
Instance        Snapshot        Parent   Comment
app-server-01   note-00-ready   --       clean lab: users, nvm, stand-in apps
```

**Meaning:** you can always come back to this exact state. Snapshots need the VM **stopped**; on a running one you get `Multipass can only take snapshots of stopped instances.`

**The server is now in Note 1's starting state.**

---

# Step 12 — Working through the notes

### The routine

```
read the theory phase → do the practice steps on app-server-01 → snapshot → next note
```

After each practice note:

```bash
multipass stop app-server-01
multipass snapshot app-server-01 --name after-note-04 --comment "accounts and groups done"
multipass start app-server-01
```

To go back (for example, after an experiment goes wrong):

```bash
multipass stop app-server-01
multipass restore app-server-01.after-note-04 --destructive
multipass start app-server-01
```

`--destructive` throws away the current state. Without it, Multipass asks whether to snapshot the current state first. That's useful if you might want to look at what broke.

### Three places to be careful

**Note 2, Step 9 (Docker Compose).** Note 4, Step 0 expects Docker to be **missing**. Either read Step 9 without running it, or snapshot first, install Docker (Step 13), try it, and restore afterwards.

**Note 4, Step 6 (`AllowGroups ssh-users`).** Once SSH only lets `ssh-users` in, Multipass's own `ubuntu` user is locked out:

```bash
multipass exec app-server-01 -- whoami        # from the Mac
```
```
exec failed: ssh failed to authenticate: 'Access denied for 'publickey'. Authentication that can continue: publickey'
```

Your own SSH still works, so fix it from inside:

```bash
sudo usermod -aG ssh-users ubuntu
```

This is the lab's one deliberate exception to Note 4's plan.

**Note 6, Step 11 (`chmod -R 777`) and anything you're unsure about.** Take a snapshot **first**. The VM is there to be broken, but restoring takes seconds and rebuilding takes minutes.

---

# Step 13 — Extras, installed when a note needs them

Install these at the point shown, not before. Several earlier steps depend on them being missing.

| Install | When | Command |
|---|---|---|
| Node system-wide (`/usr/bin/node`) | Note 2, Step 5 (services; `api` has no nvm) | `sudo apt install -y nodejs` (Node 18 — fine for the stand-in) |
| PostgreSQL | Note 2, Steps 7–8, if you want a real `pg_dump` | below |
| Docker | Note 2, Step 9 (optional, see Step 12); Note 10, Step 8 | `sudo apt install -y docker.io docker-compose-v2` |
| nginx | Note 12, Step 7 | `sudo apt install -y nginx` |

### PostgreSQL for the lab

The notes' database is on another server (`10.0.0.5`). In the lab, run it on the same VM and use `127.0.0.1`:

```bash
sudo apt install -y postgresql
sudo -u postgres psql <<'EOF'
CREATE ROLE api_rw LOGIN PASSWORD 'ApiPass';
CREATE ROLE backup_ro LOGIN PASSWORD 'BkPass';
CREATE DATABASE appdb OWNER api_rw;
GRANT pg_read_all_data TO backup_ro;
EOF
```

| Notes say | In the lab |
|---|---|
| `postgres://api_rw:...@10.0.0.5:5432/appdb` | `postgres://api_rw:ApiPass@127.0.0.1:5432/appdb` |
| `PGHOST=10.0.0.5` | `PGHOST=127.0.0.1` |

Check the backup user:

```bash
PGHOST=127.0.0.1 PGDATABASE=appdb PGUSER=backup_ro PGPASSWORD=BkPass psql -c 'select current_user'
```
```
 current_user
--------------
 backup_ro
```

### A second server (Note 4, Step 12)

Same cloud-init, new name:

```bash
multipass launch 24.04 --name app-server-02 --cpus 1 --memory 1G --disk 10G \
  --cloud-init ~/labs/app-server-01/cloud-init.yaml
```

Then repeat Steps 4–5 for it (password, `~/.ssh/config` entry). Delete it when you're done with that step (see the appendix).

---

# Step 14 — Logging in as the other people

From Note 4 on, the notes SSH in as `nimal`, `deploy`, and others. Give each one a lab key on your Mac:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/lab_nimal -C "nimal@lab" -N ''
cat ~/.ssh/lab_nimal.pub        # paste this into nimal's ~/.ssh/authorized_keys when Note 4 creates the account
```

```bash
ssh -i ~/.ssh/lab_nimal nimal@app-server-01
```

Or test as another user **without** SSH, from your `thirana` session:

```bash
sudo -iu nimal          # a login shell as nimal
sudo -u nimal id        # one command as nimal
```

`sudo -u` skips SSH, so it **won't** show SSH problems such as `AllowGroups` or key permissions (Note 4, Step 6; Note 6, Step 8). For those, use real SSH.

---

# Where the lab differs from the notes

| Where | The notes show | The lab shows | Why / what to do |
|---|---|---|---|
| Note 4, Step 0 — human accounts | Only `thirana 1000` | Also `ubuntu 1001` | Multipass's account (Step 2). Leave it |
| Note 4, Step 0 — `getent group sudo adm` | `thirana` only | `thirana,ubuntu`; also `lxd:x:104:ubuntu` | Same account |
| Note 4, Step 6 — `AllowGroups ssh-users` | Everyone outside the group is locked out | So is Multipass | `sudo usermod -aG ssh-users ubuntu` (Step 12) |
| Note 8, Step 6 — sudo audit | Only the team's rules | Also `ubuntu ALL=(ALL) NOPASSWD:ALL` in `90-cloud-init-users` | Known lab exception; a real finding anywhere else |
| Note 1 — `node -v` | `v20.11.0` | A later 20.x | Any 20.x behaves the same here |
| Note 2 — database | `10.0.0.5` | `127.0.0.1` | Step 13 |
| Note 10, Step 6 — NFS server `10.0.0.9` | A hung NFS mount | No NFS server | Read along |
| Note 14, Step 2 — logs gone after reboot | `/var/log/journal` missing | It exists | To reproduce: `sudo rm -rf /var/log/journal && sudo systemctl restart systemd-journald` |
| Note 14, Step 11 — log shipping | GCP Ops Agent → Cloud Logging | No GCP | Read along |
| GCP OS Login (Note 3) | Accounts from Google identities | Local accounts | Read along |
| Architecture | Usually `x86_64` | `aarch64` (Apple Silicon) | Commands and concepts are the same |
| IP addresses | `10.x`, `198.51.100.x` | `192.168.252.x` | Use your VM's IP |

---

# Lab safety

| Rule | Why |
|---|---|
| The VM is for practice only | It has a passwordless-root account (`ubuntu`) on purpose |
| By default only your Mac can reach it | Multipass's network is private to your Mac. Don't port-forward or bridge it to your LAN |
| Use made-up secrets | The notes use fake passwords (`BkPass`); keep it that way |
| Lab keys stay lab keys | Never add `lab_ed25519` to a real server |
| Never copy the lab's cloud-init to a real server | It's built around Multipass's needs, not least privilege |
| Snapshot before anything destructive | Restoring is faster than debugging a broken lab |

---

# Troubleshooting table

| Symptom | Cause | Fix |
|---|---|---|
| `launch failed: ... timed out waiting for response` | Default user renamed, or a broken cloud-init file | Keep `- default` in `users:` (Step 2); check YAML indentation; `multipass delete --purge app-server-01` and launch again |
| Launch gave you 26.04 | No image named | Always `multipass launch 24.04` |
| `Permission denied (publickey)` as `thirana` | Key not in cloud-init, or SSH using another key | `grep ssh-ed25519 ~/labs/app-server-01/cloud-init.yaml`; `IdentitiesOnly yes` in `~/.ssh/config` |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | You rebuilt the VM; new host key | `ssh-keygen -R app-server-01; ssh-keygen -R <IP>` — **only** because you know you rebuilt it |
| `ssh: connect to host ... Operation timed out` | VM stopped, or its IP changed | `multipass list`; update `HostName` in `~/.ssh/config` |
| `sudo: a password is required` / `Sorry, try again` | Expected / wrong password | Use `thirana`'s password from Step 4; reset with `multipass shell` + `sudo passwd thirana` |
| `multipass exec` / `shell`: `Access denied for 'publickey'` | `AllowGroups` without `ubuntu` (Note 4, Step 6) | `sudo usermod -aG ssh-users ubuntu` from your own SSH session |
| Output has an extra `ubuntu` line | Multipass's account | Expected — see the differences table |
| `snapshot failed: Multipass can only take snapshots of stopped instances.` | VM running | `multipass stop` first |
| `nvm: command not found` right after installing | Current shell didn't read the new `~/.bashrc` lines | `source ~/.bashrc` or log in again (Note 1, Phase 5) |
| `Error: listen EADDRINUSE ... :3000` | An earlier API is still running | `ss -tlnp \| grep 3000`, stop that process (Note 10) |
| `pkill -f <name>` also killed your SSH session | Your own command line contained the pattern | Use `pkill -f '[n]ame'` or `pgrep -a` first (Note 9, Phase 3.2) |
| Disk full | Journal or test files | `df -h`; `journalctl --disk-usage`; Note 14, Steps 3 and 10 |

---

# Appendix — Command reference

## Multipass (on your Mac)

```bash
multipass find                                          # available images
multipass launch 24.04 --name N --cpus 2 --memory 4G --disk 20G --cloud-init FILE
multipass list                                          # VMs, state, IPs
multipass info N                                        # details: IP, CPU, memory, disk
multipass start N / stop N / restart N
multipass shell N                                       # log in as ubuntu (setup only)
multipass exec N -- CMD                                 # one command as ubuntu
multipass transfer FILE N:/home/ubuntu/                 # copy files in (or N:/path FILE to copy out)
multipass delete --purge N                              # remove a VM for good
```

## Snapshots

```bash
multipass stop N
multipass snapshot N --name NAME --comment "TEXT"       # VM must be stopped
multipass list --snapshots
multipass restore N.NAME --destructive                  # VM must be stopped
multipass delete --purge N.NAME                         # delete one snapshot
```

## SSH (on your Mac)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/lab_ed25519 -C "thirana@lab"
ssh app-server-01                                       # uses ~/.ssh/config
ssh -i ~/.ssh/lab_nimal nimal@app-server-01             # as another user
ssh-keygen -R app-server-01                             # forget an old host key after a rebuild
```

## Inside the VM

```bash
sudo passwd -S thirana                                  # P = password set, L = locked
sudo -k; sudo -n true                                   # does sudo need a password?
cd /opt/api && PORT=3000 node server.js                 # run the API stand-in
cd /opt/worker && venv/bin/python -m worker             # run the worker stand-in
venv/bin/python -m worker.backup                        # backup (BACKUP_DIR, PG* variables)
venv/bin/python -m worker.reindex                       # long job (REINDEX_MINUTES)
```

## Rebuild from scratch

```bash
multipass delete --purge app-server-01
multipass launch 24.04 --name app-server-01 --cpus 2 --memory 4G --disk 20G \
  --cloud-init ~/labs/app-server-01/cloud-init.yaml
# then Steps 4–11 again
```
