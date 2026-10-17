# Users, Groups & Identity — Practice (Note 4)

Setting up the accounts for a real team server, step by step. Each step shows the command, what the result means, and the decision it leads to.

**Theory for this note:** Note 3. Each step points to the phase that explains it ("Why: Note 3, Phase X").

**Continues from:** Note 2 (the `api` and `worker` services). **Continues into:** Note 6 (folders and files for these accounts) and Note 8 (sudo rights and service limits).

---

## Contents

- The setup
- Step 0 — Look at the starting state
- Step 1 — Plan the groups, accounts, and ID ranges
- Step 2 — Create the groups
- Step 3 — Onboard a developer
- Step 4 — The `usermod -G` mistake
- Step 5 — Fix the service accounts
- Step 6 — The CI `deploy` user and controlling who can SSH
- Step 7 — A contractor with an end date
- Step 8 — The auditor, and the group that doesn't apply yet
- Step 9 — "Can you add me to the docker group?"
- Step 10 — Offboarding the contractor
- Step 11 — Orphaned files and UID reuse
- Step 12 — A second server: making accounts the same everywhere
- State of the server after Note 4
- Scenario recap
- Troubleshooting table
- Appendix — Command reference

---

# The setup

Server: `app-server-01`, Ubuntu 24.04. Same server as Note 2, now used by a small team.

| Who / what | Role | Needs |
|---|---|---|
| `thirana` | Platform lead, existing admin | Full admin (already in `sudo`) |
| `nimal`, `sachini` | Developers | SSH in, work on the project, read app logs |
| `ruwan` | Ops on-call | SSH in, restart services, read all logs |
| `amal` | Contract developer until 2026-12-31 | Same as developers, time-limited |
| `dilini` | Auditor | Read logs only |
| `deploy` | CI/CD pipeline | SSH in with a key, deploy releases |
| `api`, `worker` | Services from Note 2 | Run the apps, share an uploads folder |

This note sets up **identity**: accounts, groups, logins. What each group can **access** (folders, files, sudo) comes in Notes 6 and 8.

---

# Step 0 — Look at the starting state

Never start changing accounts without seeing what's there.

```bash
getent passwd | awk -F: '$3 >= 1000 && $3 < 60000 {print $1, $3, $7}'
```
```
thirana 1000 /bin/bash
```

**Meaning:** one human account. (Range from `/etc/login.defs`, Note 3 Phase 3.3.)

```bash
getent passwd api worker
```
```
api:x:999:999::/home/api:/usr/sbin/nologin
worker:x:998:998::/home/worker:/usr/sbin/nologin
```

**Meaning:** the two service accounts from Note 2. Auto-picked UIDs (999, 998), and a home path `/home/api` that **doesn't exist** (created with `--no-create-home`). Remember this for Step 5.

```bash
getent group sudo docker lxd disk adm
```
```
sudo:x:27:thirana
adm:x:4:syslog,thirana
```

**Meaning:** only `thirana` has sudo. Docker isn't installed (no `docker` group). Good starting point. (Why: Note 3, Phase 4.5.)

Quick safety checks:

```bash
awk -F: '$3 == 0' /etc/passwd             # root:x:0:0:root:/root:/bin/bash   ← only root, good
sudo awk -F: '$2 == ""' /etc/shadow        # (nothing)                          ← no empty passwords
sudo passwd -S root                        # root L ...                         ← root password locked
sudo sshd -T | grep -i permitrootlogin     # permitrootlogin without-password   ← root can log in with a key
```

**Conclusion:** the server is clean, but root can still SSH in with a key. We'll turn that off in Step 6.

Check the defaults new accounts will get:

```bash
grep -E '^(UID_MIN|UID_MAX|SYS_UID_MAX|HOME_MODE|UMASK|USERGROUPS_ENAB)' /etc/login.defs
useradd -D
```
```
UID_MIN 1000 / UID_MAX 60000 / SYS_UID_MAX 999 / UMASK 022 / HOME_MODE 0750 / USERGROUPS_ENAB yes
GROUP=100 ... SHELL=/bin/sh ...
```

**Meaning:** new homes will be `750` (private), each user gets their own group, and **the default shell is `/bin/sh`** — so we must always pass `-s /bin/bash` for humans.

---

# Step 1 — Plan the groups, accounts, and ID ranges

Write the plan down first. It's the source of truth for every server (Step 12).

### ID ranges

| Range | Used for |
|---|---|
| 900–949 | Our service accounts |
| 1000–1099 | Admins |
| 1100–1199 | Staff |
| 1200–1299 | Contractors |
| 1300–1399 | Auditors |
| 1900–1999 | Machine logins (CI/CD) |
| 2000–2099 | Our role groups |

**Rule: a removed ID is never given to someone new** (Step 11).

### Groups

| Group | GID | Purpose |
|---|---|---|
| `devs` | 2001 | Developers: project folder, app logs (Note 6) |
| `ops` | 2002 | Ops: service restarts (Note 8), all logs |
| `appdata` | 2003 | Shared data between `api` and `worker` (uploads, Note 6) |
| `ssh-users` | 2004 | Who may log in over SSH at all (Step 6) |

### Accounts

| Account | UID | Groups | Notes |
|---|---|---|---|
| `thirana` | 1000 | sudo, adm, ssh-users | Existing |
| `nimal` | 1101 | devs, ssh-users | |
| `sachini` | 1102 | devs, ssh-users | Joins ops rotation later |
| `ruwan` | 1103 | ops, adm, ssh-users | |
| `amal` | 1201 | devs, ssh-users | Expires 2026-12-31 |
| `dilini` | 1301 | adm, ssh-users | Read-only |
| `deploy` | 1901 | ssh-users | Key-only, restricted key |
| `api` | 901 | appdata | Service, nologin |
| `worker` | 902 | appdata | Service, nologin |

**Why groups and not people:** access is given to `devs`, `ops`, and so on. Joining and leaving then means changing group membership only. (Why: Note 3, Phase 4.4.)

---

# Step 2 — Create the groups

```bash
sudo groupadd -g 2001 devs
sudo groupadd -g 2002 ops
sudo groupadd -g 2003 appdata
sudo groupadd -g 2004 ssh-users
```

Verify:

```bash
getent group devs ops appdata ssh-users
```
```
devs:x:2001:
ops:x:2002:
appdata:x:2003:
ssh-users:x:2004:
```

**Meaning:** the groups exist, with the planned GIDs and no members yet.

Add yourself to `ssh-users` **now**. In Step 6 only this group will be allowed to SSH in, and you don't want to lock yourself out.

```bash
sudo gpasswd -a thirana ssh-users
```

---

# Step 3 — Onboard a developer

```bash
sudo useradd -m -u 1101 -s /bin/bash -c "Nimal Perera" -G devs,ssh-users nimal
```

| Option | Why |
|---|---|
| `-m` | Create `/home/nimal` and copy `/etc/skel` into it |
| `-u 1101` | Fixed UID from the plan |
| `-s /bin/bash` | Otherwise the default `/bin/sh` is used (Step 0) |
| `-c` | Full name, so logs and lists are readable |
| `-G devs,ssh-users` | Supplementary groups |

(Why: Note 3, Phase 5.2.)

### Check the result

```bash
id nimal
```
```
uid=1101(nimal) gid=1101(nimal) groups=1101(nimal),2001(devs),2004(ssh-users)
```

**Meaning:** a private group `nimal` (1101) was created as the primary group, because of `USERGROUPS_ENAB yes`. (Why: Note 3, Phase 4.3.)

```bash
ls -ld /home/nimal
ls -la /home/nimal
```
```
drwxr-x--- 2 nimal nimal 4096 Oct 17 09:10 /home/nimal
.bash_logout  .bashrc  .profile
```

**Meaning:** `750` from `HOME_MODE` — other users can't look inside. The starter files came from `/etc/skel`.

```bash
sudo passwd -S nimal
```
```
nimal L 2026-10-17 0 99999 7 -1
```

**Meaning:** `L` because no password was set (`!` in shadow). That's what we want — nimal will log in with an SSH key only. (Why: Note 3, Phase 6.1.)

### Add the SSH key

Nimal sends his **public** key (`id_ed25519.pub`). Never accept or send private keys.

```bash
sudo install -d -m 700 -o nimal -g nimal /home/nimal/.ssh
sudo install -m 600 -o nimal -g nimal nimal.pub /home/nimal/.ssh/authorized_keys
```

`install` creates the folder or file with the owner and permissions in one step. Why `700`/`600` — and what happens if they're looser — is in Note 6.

### Test

From nimal's laptop:

```bash
ssh nimal@app-server-01
id
```
```
uid=1101(nimal) gid=1101(nimal) groups=1101(nimal),2001(devs),2004(ssh-users)
```

**Conclusion:** the account matches the plan. Repeat for `ruwan` (`-G ops,adm,ssh-users`, UID 1103).

---

# Step 4 — The `usermod -G` mistake

Add the second developer the same way:

```bash
sudo useradd -m -u 1102 -s /bin/bash -c "Sachini Fernando" -G devs,ssh-users sachini
```

A month later, sachini joins the ops on-call rotation. Someone runs:

```bash
sudo usermod -G ops sachini
```

```bash
id sachini
```
```
uid=1102(sachini) gid=1102(sachini) groups=1102(sachini),2002(ops)
```

**Meaning:** she's now in `ops`, but **no longer in `devs` or `ssh-users`**. `-G` alone **replaces** the whole list. Her next SSH login will be refused (Step 6). If this had been `thirana` and the list didn't include `sudo`, the server would have lost its admin. (Why: Note 3, Phase 5.3.)

### Fix

```bash
sudo usermod -aG devs,ssh-users,ops sachini
id sachini
```
```
uid=1102(sachini) gid=1102(sachini) groups=1102(sachini),2001(devs),2002(ops),2004(ssh-users)
```

**Rule:** to add, use `usermod -aG` or `gpasswd -a user group`. To remove one group, use `gpasswd -d user group`. Avoid `usermod -G` unless you really mean "set exactly this list".

Sachini was logged in while this happened. Her running session still has the old group list. See Step 8.

---

# Step 5 — Fix the service accounts

Step 0 showed `api` and `worker` with auto UIDs and a home folder that doesn't exist. Two real problems come from that.

### Problem 1: the app can't write its cache

After an update, the API's logs show:

```
Error: EACCES: permission denied, mkdir '/home/api'
```

**Check what HOME the service gets:**

```bash
getent passwd api              # ...:/home/api:/usr/sbin/nologin
ls -ld /home/api               # No such file or directory
ls -ld /home                   # drwxr-xr-x root root /home
```

**Meaning:** systemd sets `HOME` from `/etc/passwd`. A library tried to create its cache in `$HOME`. The folder doesn't exist, and `api` can't create it because `/home` is writable only by root. (Env side: Note 2. Why it's denied: Note 5.)

**Fix:** give each service a real home **outside `/home`**, the normal place for service data:

```bash
sudo usermod -d /var/lib/api api
sudo install -d -o api -g api -m 750 /var/lib/api

sudo usermod -d /var/lib/worker worker
sudo install -d -o worker -g worker -m 750 /var/lib/worker
```

(systemd can also create this for you with `StateDirectory=` — Note 8.)

### Problem 2: IDs don't match the plan

The plan says `api` = 901, `worker` = 902. Fix it now, while few files exist.

```bash
sudo usermod -u 901 api
```
```
usermod: user api is currently used by process 2314
```

**Meaning:** you can't change the UID of an account with running processes. Stop the service first.

```bash
sudo systemctl stop api worker

sudo usermod -u 901 api && sudo groupmod -g 901 api
sudo usermod -u 902 worker && sudo groupmod -g 902 worker
```

`usermod -u` updates ownership **inside the home folder only**. Find everything else still owned by the old numbers:

```bash
sudo find / -xdev \( -uid 999 -o -gid 999 -o -uid 998 -o -gid 998 \) -ls
```
```
... 999 999 ... /etc/myapps/api.env
... 998 998 ... /etc/myapps/worker.env
```

**Meaning:** the env files from Note 2 now show bare numbers. Fix them:

```bash
sudo find / -xdev -uid 999 -exec chown -h 901 {} +
sudo find / -xdev -gid 999 -exec chgrp -h 901 {} +
sudo find / -xdev -uid 998 -exec chown -h 902 {} +
sudo find / -xdev -gid 998 -exec chgrp -h 902 {} +
```

(`-h` changes a symlink itself, not the file it points to. `-xdev` stays on one filesystem; repeat for other mounts if needed.)

### Add both services to the shared group

```bash
sudo gpasswd -a api appdata
sudo gpasswd -a worker appdata
sudo systemctl start api worker
```

### Check what the running processes actually have

```bash
PID=$(systemctl show api -p MainPID --value)
grep -E '^(Uid|Gid|Groups)' /proc/$PID/status
```
```
Uid:  901 901 901 901
Gid:  901 901 901 901
Groups: 901 2003
```

**Meaning:** the new UID and the `appdata` group (2003) are active. This only shows because we **restarted** — a running process keeps the group list it started with. (Why: Note 3, Phase 7.3.)

### Test as the service

```bash
sudo su - api
# This account is currently not available.     ← nologin works

sudo -u api id
# uid=901(api) gid=901(api) groups=901(api),2003(appdata)

sudo -u api bash -c 'echo $HOME; touch $HOME/test && echo ok'
# /var/lib/api
# ok
```

**Conclusion:** service accounts can't log in, but you can still test as them with `sudo -u`. (Why: Note 3, Phase 8.2.)

---

# Step 6 — The CI `deploy` user and controlling who can SSH

### Create the account

`deploy` is not a person, but it must log in over SSH, so it needs a real shell.

```bash
sudo useradd -m -u 1901 -s /bin/bash -c "CI/CD deploy" -G ssh-users deploy
```

### A restricted key

The CI system has its own key pair. On the server, the key line in `authorized_keys` gets **options** that limit what it can do:

```
restrict,from="203.0.113.10/32" ssh-ed25519 AAAAC3Nz... ci-deploy
```

| Option | Effect |
|---|---|
| `restrict` | Turns off port forwarding, agent forwarding, X11, and the interactive terminal |
| `from="IP"` | Key works only from the CI runner's fixed IP (use if your CI has one) |

```bash
sudo install -d -m 700 -o deploy -g deploy /home/deploy/.ssh
sudo install -m 600 -o deploy -g deploy ci-deploy.authorized_keys /home/deploy/.ssh/authorized_keys
```

What `deploy` may **do** (restart services, write releases) comes in Note 6 (folders) and Note 8 (one narrow sudo rule).

### Only `ssh-users` may SSH in

Right now, any account with a key and a shell could SSH in. If someone placed a key in `/var/lib/api/.ssh/`, `api` could still be reached through tools that don't need a shell.

Create a drop-in config. On Ubuntu, files in `/etc/ssh/sshd_config.d/` are read before the main file, and **the first value found wins**, so a low number like `10-` beats the cloud image's `50-cloud-init.conf`.

`/etc/ssh/sshd_config.d/10-access.conf`:

```
AllowGroups ssh-users
PermitRootLogin no
PasswordAuthentication no
```

**Before reloading, keep your current SSH session open** (a safety net), then:

```bash
sudo sshd -t                     # syntax check: no output = OK
sudo sshd -T | grep -Ei '^(allowgroups|permitrootlogin|passwordauthentication)'
```
```
permitrootlogin no
passwordauthentication no
allowgroups ssh-users
```

**Meaning:** `sshd -T` prints the **effective** settings after all files are merged. Our values won.

```bash
sudo systemctl reload ssh
```

Test from a **second** terminal that you can still log in. Then test someone outside the group — e.g. sachini right after the Step 4 mistake:

```bash
sudo grep 'AllowGroups' /var/log/auth.log | tail -1
```
```
sshd[4021]: User sachini from 198.51.100.7 not allowed because none of user's groups are listed in AllowGroups
```

**Conclusion:** `ssh-users` membership is now the on/off switch for SSH access — useful again in Step 10.

---

# Step 7 — A contractor with an end date

```bash
sudo useradd -m -u 1201 -s /bin/bash -c "Amal Silva (contractor)" \
    -G devs,ssh-users -e 2026-12-31 amal
```

`-e` sets the **account expiry**. After that date, no login works — password or key. (Why: Note 3, Phase 6.3.)

Amal will log in with a key (password SSH is off). He gets a password **for sudo** later (Note 8), so set one he must change on first use, plus an aging policy:

```bash
sudo passwd amal                  # temporary password, sent to him privately
sudo chage -d 0 amal              # must change it at first use
sudo chage -M 90 -W 7 -I 14 amal  # max 90 days, warn 7 days before, lock 14 days after expiry
```

Check:

```bash
sudo chage -l amal
```
```
Last password change                    : password must be changed
Password expires                        : password must be changed
Password inactive                       : password must be changed
Account expires                         : Dec 31, 2026
Minimum number of days between password change  : 0
Maximum number of days between password change  : 90
Number of days of warning before password expires: 7
```

**Meaning:** two separate clocks — a **password** policy (change every 90 days) and an **account** end date (31 Dec). The account date is the one that matters for contractors.

Add his SSH key the same way as Step 3.

---

# Step 8 — The auditor, and the group that doesn't apply yet

Dilini starts with just SSH access:

```bash
sudo useradd -m -u 1301 -s /bin/bash -c "Dilini Jayasinghe (audit)" -G ssh-users dilini
```

The audit scope is agreed later: she needs to read system and app logs. On Ubuntu, `adm` can read `/var/log/*` files and the systemd journal. (If she only needed the journal, `systemd-journal` would be the narrower choice.) (Why: Note 3, Phase 4.5.)

```bash
sudo gpasswd -a dilini adm
```

Dilini is already logged in. She tries:

```bash
cat /var/log/auth.log
```
```
cat: /var/log/auth.log: Permission denied
```

### Diagnose

```bash
ls -l /var/log/auth.log
```
```
-rw-r----- 1 syslog adm 52311 Oct 17 10:02 /var/log/auth.log
```

**Meaning:** group `adm` can read. So: is she in `adm`?

```bash
id            # in her session
```
```
uid=1301(dilini) gid=1301(dilini) groups=1301(dilini),2004(ssh-users)
```
```bash
id dilini     # the database
```
```
uid=1301(dilini) gid=1301(dilini) groups=1301(dilini),4(adm),2004(ssh-users)
```

**Meaning:** the database has `adm`, but her **running session** doesn't. The group list was fixed when she logged in. (Why: Note 3, Phase 7.3.)

### Fix

Log out and back in. Or, for a quick check without logging out:

```bash
sg adm -c 'tail -3 /var/log/auth.log'      # one command with the adm group
```

After a new login:

```bash
id                          # ... 4(adm) ...
journalctl -u api --since today | tail -3   # works
sudo -l                     # Sorry, user dilini may not run sudo   ← good: read-only role
cat /etc/shadow             # Permission denied                      ← good
```

**Conclusion:** the auditor can read logs and nothing more. Keep in mind that logs can contain sensitive data, so `adm` is a real privilege — give it on purpose, not by default.

---

# Step 9 — "Can you add me to the docker group?"

Nimal asks to be added to `docker` so he can run containers on the server without sudo.

### Check what that means

```bash
ls -l /var/run/docker.sock
```
```
srw-rw---- 1 root docker 0 Oct 17 08:00 /var/run/docker.sock
```

**Meaning:** anyone in group `docker` can talk to the Docker daemon, which runs as **root**. A member can start a container that mounts the host's `/` and runs as root inside it — which is root on the host, because container UIDs are host UIDs. (Why: Note 3, Phase 4.5 and 10.3.)

So "add to docker" = "give root, without sudo's logging and rules".

### Decision

| Option | When |
|---|---|
| Treat it as admin access: approve formally, same as `sudo` | If he really needs root-level control |
| **Rootless Docker** under his own user | For running his own test containers |
| Don't run ad-hoc containers on a production server | Usually the right answer; use CI or a dev machine |

For `app-server-01` (production): **no**. He uses CI for builds.

### Make it checkable

Add this to the regular audit:

```bash
for g in sudo docker lxd disk adm shadow; do getent group "$g"; done
```

---

# Step 10 — Offboarding the contractor

Amal's contract ends early, on 20 Nov. Someone does the "obvious" thing:

```bash
sudo passwd -l amal
```

The next day, auth.log shows:

```bash
sudo grep 'Accepted' /var/log/auth.log | grep amal | tail -1
```
```
sshd[5122]: Accepted publickey for amal from 198.51.100.23 port 51544 ssh2
```

**Meaning:** `passwd -l` only locked the **password**. He logs in with a **key**, which still works. (Why: Note 3, Phase 6.4.)

### Do it properly — in this order

**1. Disable all logins**

```bash
sudo usermod -L -e 1 amal          # lock password AND expire the account
sudo gpasswd -d amal ssh-users     # SSH refuses him even if expiry is undone by mistake
sudo gpasswd -d amal devs          # remove access groups
```

Check:

```bash
sudo chage -l amal | grep 'Account expires'    # Account expires : Jan 02, 1970
id amal                                        # groups=1201(amal) only
```

**2. End what's running**

```bash
loginctl list-sessions | grep amal
sudo loginctl terminate-user amal     # end his sessions
sudo pkill -KILL -u amal              # kill any leftover processes
sudo crontab -l -u amal               # anything scheduled?
sudo crontab -r -u amal               # remove it
```

**3. Find everything he owns outside his home**

```bash
sudo find / -xdev -user amal -not -path '/home/amal/*' -ls 2>/dev/null
```
```
... amal amal ... /var/tmp/amal-load-test/results.csv
... amal devs ... /srv/projects/feature-x/notes.md
```

**4. Hand over** what the team still needs:

```bash
sudo chown -R nimal /srv/projects/feature-x/      # or a team owner (Note 6)
sudo rm -rf /var/tmp/amal-load-test
```

**5. Check his key isn't used anywhere else**

People sometimes paste their key into shared accounts like `deploy`. Compare fingerprints, not comments (comments can say anything):

```bash
ssh-keygen -lf /home/amal/.ssh/authorized_keys
sudo ssh-keygen -lf /home/deploy/.ssh/authorized_keys
```

If a fingerprint matches, remove that line from `deploy`.

**6. Archive the home folder** (if your policy needs it)

```bash
sudo install -d -m 700 /root/offboarded
sudo tar -czpf /root/offboarded/amal-2026-11-20.tgz -C /home amal
```

**7. Delete later** (e.g. after 30 days)

```bash
sudo userdel -r amal
```

**Conclusion:** offboarding is **disable → stop → find → hand over → archive → delete later**. Expiry plus removal from `ssh-users` closes every login path immediately. (Why: Note 3, Phase 9.4.)

---

# Step 11 — Orphaned files and UID reuse

After `userdel`, someone forgot a folder in step 3 of offboarding.

```bash
sudo find / -xdev \( -nouser -o -nogroup \) -ls 2>/dev/null
```
```
... 1201 devs ... /srv/projects/old-spike/data.json
```

**Meaning:** the file's owner is the bare number 1201. No account has that UID any more. (Why: Note 3, Phase 9.2.)

In January, a colleague onboards a new contractor. He looks at the contractor range, sees "1201 is free", and runs:

```bash
sudo useradd -m -u 1201 -s /bin/bash -G devs,ssh-users kavindu
ls -l /srv/projects/old-spike/data.json
```
```
-rw-rw-r-- 1 kavindu devs ... data.json
```

**Meaning:** the new contractor now **owns Amal's old file** — and anything else left behind with UID 1201. (Why: Note 3, Phase 9.3.)

### Fix and prevent

- Before deleting an account, **reassign or remove** everything from step 3 of offboarding. After deleting, run the `-nouser` check.
- Keep a **retired IDs** list in the plan (Step 1): `1201 — amal — retired 2026-11-20`. Never reuse them.
- Give kavindu the next unused number (1202) instead.

---

# Step 12 — A second server: making accounts the same everywhere

A second server, `app-server-02`, is added. Someone created `api` there by hand earlier. You copy uploaded files over with `rsync -a`:

```bash
ls -l /srv/uploads/ | head -3     # on app-server-02
```
```
-rw-r----- 1 901 2003 ... img-001.png
```

```bash
getent passwd api                 # api:x:997:997:...   (on server-02)
getent group appdata              # (nothing)            (on server-02)
```

**Meaning:** `rsync -a` copied the **numbers**. On server-01, 901 is `api` and 2003 is `appdata`. On server-02, `api` is 997 and there's no `appdata` group, so nobody there owns these files and the API can't read them. (Why: Note 3, Phase 1.2 and 10.1.)

### Fix: one provisioning script for every server

`/root/provision/accounts.sh`:

```bash
#!/bin/bash
# Create the same groups and accounts on every server, from the Step 1 plan.
# Safe to run many times. Stops if an existing account has the wrong ID.
set -euo pipefail

check_uid() {   # check_uid NAME EXPECTED_UID
  local actual
  actual=$(id -u "$1")
  if [ "$actual" != "$2" ]; then
    echo "ERROR: $1 has UID $actual, expected $2. Renumber it first (Note 4, Step 5)." >&2
    exit 1
  fi
}

group() {   # group NAME GID
  getent group "$1" >/dev/null || groupadd -g "$2" "$1"
}

service_user() {   # service_user NAME ID
  getent group "$1"  >/dev/null || groupadd --system -g "$2" "$1"
  getent passwd "$1" >/dev/null || useradd --system -u "$2" -g "$1" \
      -d "/var/lib/$1" -M -s /usr/sbin/nologin -c "$1 service" "$1"
  check_uid "$1" "$2"
  install -d -o "$1" -g "$1" -m 750 "/var/lib/$1"
}

human_user() {   # human_user NAME ID "Full Name" GROUPS [EXPIRY]
  getent group "$1"  >/dev/null || groupadd -g "$2" "$1"
  getent passwd "$1" >/dev/null || useradd -m -u "$2" -g "$1" -s /bin/bash -c "$3" "$1"
  check_uid "$1" "$2"
  usermod -aG "$4" "$1"
  if [ -n "${5:-}" ]; then usermod -e "$5" "$1"; fi
  if [ -f "/root/provision/keys/$1.pub" ]; then
    install -d -m 700 -o "$1" -g "$1" "/home/$1/.ssh"
    install -m 600 -o "$1" -g "$1" "/root/provision/keys/$1.pub" "/home/$1/.ssh/authorized_keys"
  fi
}

# --- Groups ---
group devs      2001
group ops       2002
group appdata   2003
group ssh-users 2004

# --- Services ---
service_user api    901
service_user worker 902
usermod -aG appdata api
usermod -aG appdata worker

# --- People and machine logins ---
human_user nimal   1101 "Nimal Perera"              devs,ssh-users
human_user sachini 1102 "Sachini Fernando"          devs,ops,ssh-users
human_user ruwan   1103 "Ruwan Bandara"             ops,adm,ssh-users
human_user dilini  1301 "Dilini Jayasinghe (audit)" adm,ssh-users
human_user deploy  1901 "CI/CD deploy"              ssh-users
human_user kavindu 1202 "Kavindu (contractor)"      devs,ssh-users 2027-06-30

# Retired IDs (never reuse): 1201 amal 2026-11-20
```

Run it on server-02:

```bash
sudo bash /root/provision/accounts.sh
```
```
ERROR: api has UID 997, expected 901. Renumber it first (Note 4, Step 5).
```

**Meaning:** the script found the mismatch instead of silently working around it. Renumber `api` on server-02 using the Step 5 procedure (stop service → `usermod -u` / `groupmod -g` → `find ... -exec chown`), then run the script again. It completes.

Now check the copied files again:

```bash
ls -l /srv/uploads | head -2
```
```
-rw-r----- 1 api appdata ... img-001.png
```

**Meaning:** no file changes were needed. Once server-02 uses the same numbers as server-01, the copied files are owned by the right names automatically. That's the payoff of fixed IDs.

**Conclusion:** IDs come from one plan and one script, so every server agrees. In bigger setups the same idea is done with Ansible's `user`/`group` modules, or by central identity — on GCP, **OS Login** gives human accounts (and their sudo, via IAM roles) from Google identities, so you don't create people on each VM by hand. (Why: Note 3, Phase 10.2.)

---

# State of the server after Note 4

This is the starting point for Note 6.

| Account | UID | Groups | Login |
|---|---|---|---|
| `thirana` | 1000 | sudo, adm, ssh-users | SSH key |
| `nimal` | 1101 | devs, ssh-users | SSH key |
| `sachini` | 1102 | devs, ops, ssh-users | SSH key |
| `ruwan` | 1103 | ops, adm, ssh-users | SSH key |
| `kavindu` | 1202 | devs, ssh-users | SSH key, expires 2027-06-30 |
| `dilini` | 1301 | adm, ssh-users | SSH key, read-only |
| `deploy` | 1901 | ssh-users | Restricted SSH key |
| `api` | 901 | appdata | nologin, home `/var/lib/api` |
| `worker` | 902 | appdata | nologin, home `/var/lib/worker` |

SSH: only `ssh-users`, no root login, no passwords.

---

# Scenario recap

| Step | What happened | Concept (Note 3) |
|---|---|---|
| 0 | Checked accounts, groups, defaults | `getent`, `login.defs`, UID 0, empty passwords (Phases 1–3) |
| 1 | Wrote ID ranges, groups, accounts | Role groups, fixed IDs (4.4, 10) |
| 2 | Created groups | `groupadd -g` (5.4) |
| 3 | Onboarded a developer | `useradd` options, UPG, `HOME_MODE`, key-only (4.3, 5.2, 6.1) |
| 4 | `usermod -G` removed groups | `-G` replaces, use `-aG` (5.3) |
| 5 | Service had no home; IDs renumbered | Service homes in `/var/lib`, `usermod -u`, restart for groups (3.2, 7.3) |
| 6 | Restricted CI key; only `ssh-users` can SSH | Machine logins, `AllowGroups` (3.1, 8.3) |
| 7 | Contractor with an end date | Account vs password expiry (6.2, 6.3) |
| 8 | Auditor couldn't read logs after being added | Session group list is fixed at login (7.3) |
| 9 | Docker group request refused | Root-equivalent groups (4.5) |
| 10 | `passwd -l` didn't stop a key login | Expire + remove from `ssh-users` (6.4, 9.4) |
| 11 | New user inherited old files | Orphaned files, UID reuse (9.2, 9.3) |
| 12 | Second server had different IDs | Fixed IDs, provisioning script, OS Login (10) |

---

# Troubleshooting table

| Symptom | Likely cause | Check | Fix |
|---|---|---|---|
| New user has no home folder | `useradd` without `-m` | `ls -ld /home/user` | `mkhomedir_helper user` or create with `install -d` + copy `/etc/skel` |
| Plain `$` prompt, no history/tab tricks | Shell is `/bin/sh` (no `-s`) | `getent passwd user` | `usermod -s /bin/bash user` |
| User lost groups (or sudo!) | `usermod -G` without `-a` | `id user` | `usermod -aG ...`; if sudo lost, another admin or the cloud console fixes it |
| "Permission denied" right after being added to a group | Session has old group list | `id` vs `id user` | Log in again, `newgrp`, or `sg group -c cmd` |
| Service still can't use the new group | Process started before the change | `/proc/PID/status` Groups line | Restart the service |
| `su - api`: account not available | `nologin` shell (expected) | `getent passwd api` | `sudo -u api cmd` to test |
| App: `EACCES ... mkdir '/home/svc'` | Service home doesn't exist | `getent passwd svc`, `ls -ld` | `usermod -d /var/lib/svc` + create it |
| `usermod: user X is currently used by process` | Changing UID of a running account | `ps -u X` | Stop the service first |
| Locked user still logs in | `passwd -l` doesn't block keys | `grep Accepted /var/log/auth.log` | `usermod -e 1`, remove from `ssh-users` |
| SSH: "not allowed because none of user's groups are listed in AllowGroups" | Not in `ssh-users` | `id user` | `gpasswd -a user ssh-users` |
| sshd setting seems ignored | Another file set it first | `sshd -T \| grep setting` | Put it in a lower-numbered drop-in |
| Owner shows as a number | Deleted user or ID mismatch | `ls -n`, `getent passwd NUM` | Reassign with `chown`; align IDs |
| New user owns old files | Reused UID | `find / -uid NUM` | Never reuse IDs; clean up before `userdel` |
| `groupdel: cannot remove the primary group of user` | Group is someone's primary | `getent passwd \| awk -F: '$4==GID'` | Change their primary group first |

---

# Appendix — Command reference

## Inspect

| Command | Purpose |
|---|---|
| `getent passwd \| awk -F: '$3>=1000 && $3<60000'` | Human accounts |
| `getent passwd user` / `getent group name` | One account / group |
| `id` / `id user` | Session vs database groups |
| `sudo passwd -S user` / `sudo chage -l user` | Password and expiry status |
| `grep -E '^(Uid\|Gid\|Groups)' /proc/PID/status` | Identity of a running process |
| `useradd -D` / `grep ... /etc/login.defs` | Defaults |
| `sudo sshd -T` | Effective SSH settings |
| `for g in sudo docker lxd disk adm shadow; do getent group $g; done` | Powerful groups |

## Create

| Command | Purpose |
|---|---|
| `groupadd -g GID name` | Role group with fixed GID |
| `useradd -m -u UID -s /bin/bash -c "Name" -G g1,g2 user` | Human account |
| `useradd -m -u UID -s /bin/bash -c "Name" -G g -e DATE user` | Contractor |
| `useradd --system -u ID -g grp -d /var/lib/svc -M -s /usr/sbin/nologin svc` | Service account |
| `install -d -m 700 -o u -g u /home/u/.ssh` | SSH folder |
| `install -m 600 -o u -g u key.pub /home/u/.ssh/authorized_keys` | SSH key |

## Change

| Command | Purpose |
|---|---|
| `usermod -aG g user` / `gpasswd -a user g` | Add to a group |
| `gpasswd -d user g` | Remove from a group |
| `usermod -d /var/lib/svc svc` | Change home |
| `usermod -u NEW user` + `groupmod -g NEW group` | Renumber (service stopped) |
| `find / -xdev -uid OLD -exec chown -h NEW {} +` | Fix files after renumbering |
| `chage -d 0 user` / `chage -M 90 -W 7 -I 14 user` | Force change / aging |
| `usermod -e DATE user` | Account expiry |

## Offboard

| Command | Purpose |
|---|---|
| `usermod -L -e 1 user` | Lock password + expire account |
| `gpasswd -d user ssh-users` | Block SSH |
| `loginctl terminate-user user` / `pkill -KILL -u user` | End sessions / processes |
| `crontab -l -u user` / `crontab -r -u user` | Show / remove crontab |
| `find / -xdev -user user -not -path '/home/user/*'` | Files outside home |
| `ssh-keygen -lf authorized_keys` | Compare key fingerprints |
| `tar -czpf archive.tgz -C /home user` | Archive home |
| `userdel -r user` | Delete (later) |
| `find / -xdev \( -nouser -o -nogroup \)` | Check for orphans |

## SSH access control

| Item | Purpose |
|---|---|
| `/etc/ssh/sshd_config.d/10-access.conf` | Drop-in that wins over later files |
| `AllowGroups ssh-users` | Only this group can SSH |
| `PermitRootLogin no` / `PasswordAuthentication no` | No root login / keys only |
| `restrict,from="IP"` in `authorized_keys` | Limit a key |
| `sshd -t` / `sshd -T` / `systemctl reload ssh` | Check / show effective / apply |