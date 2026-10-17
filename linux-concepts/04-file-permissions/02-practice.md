# Files, Directories & Permissions — Practice (Note 6)

Setting up the folders and files for the team server, step by step. Each step shows the command, what the result means, and the decision it leads to.

**Theory for this note:** Note 5. Each step points to the phase that explains it ("Why: Note 5, Phase X").

**Continues from:** Note 4 (accounts and groups). **Continues into:** Note 8 (sudo rights, capabilities, and systemd limits for these services).

---

## Contents

- The setup
- Step 0 — Look at the starting state
- Step 1 — Plan the layout
- Step 2 — The team project folder
- Step 3 — Code releases the app can't modify
- Step 4 — Config and secrets
- Step 5 — Uploads shared between two services
- Step 6 — Logs for three audiences, and an append-only audit log
- Step 7 — Backups
- Step 8 — "My SSH key stopped working"
- Step 9 — Full "Permission denied" walkthrough
- Step 10 — Docker bind mount with the wrong UID
- Step 11 — The `chmod -R 777` incident, and `noexec` for uploads
- Step 12 — Make it repeatable: layout script, ACL backup, audit
- State of the server after Note 6
- Scenario recap
- Troubleshooting table
- Appendix — Command reference

---

# The setup

The server from Note 4:

| Account | UID | Groups |
|---|---|---|
| `thirana` | 1000 | sudo, adm, ssh-users |
| `nimal`, `sachini` | 1101, 1102 | devs (sachini also ops), ssh-users |
| `ruwan` | 1103 | ops, adm, ssh-users |
| `kavindu` | 1202 | devs, ssh-users (contractor) |
| `dilini` | 1301 | adm, ssh-users (auditor) |
| `deploy` | 1901 | ssh-users |
| `api` | 901 | appdata |
| `worker` | 902 | appdata |

Groups: `devs` 2001, `ops` 2002, `appdata` 2003, `ssh-users` 2004, plus system `adm` 4.

---

# Step 0 — Look at the starting state

### umask of people and services

As nimal:

```bash
umask
```
```
0002
```

**Meaning:** on Ubuntu, normal users get `002` because of user private groups (`USERGROUPS_ENAB yes`). New files will be `664`, new folders `775`. (Why: Note 5, Phase 7.3.)

The running API service:

```bash
PID=$(systemctl show api -p MainPID --value)
grep Umask /proc/$PID/status
```
```
Umask:  0022
```

**Meaning:** systemd's default. The API's new files will be `644` — readable by group **and other**. Keep this in mind for Steps 5 and 7.

### Key system files

```bash
stat -c '%a %A %U:%G %n' /etc/passwd /etc/shadow /home /home/nimal /tmp
```
```
644 -rw-r--r-- root:root /etc/passwd
640 -rw-r----- root:shadow /etc/shadow
755 drwxr-xr-x root:root /home
750 drwxr-x--- nimal:nimal /home/nimal
1777 drwxrwxrwt root:root /tmp
```

**Meaning:** all as expected (Note 5, Phase 13.1).

### Link protections

```bash
sysctl fs.protected_symlinks fs.protected_hardlinks
```
```
fs.protected_symlinks = 1
fs.protected_hardlinks = 1
```

**Meaning:** link attacks in `/tmp` and other shared folders are blocked (Note 5, Phase 1.3).

### Current app files from Note 2

```bash
stat -c '%a %U:%G %n' /etc/myapps/* 
ls -ld /opt/api
```
```
644 root:root /etc/myapps/common.env
600 api:api /etc/myapps/api.env
600 worker:worker /etc/myapps/worker.env
drwxr-xr-x 5 api api 4096 ... /opt/api
```

**Meaning:** two things to improve. The services **own** their config files, and `api` owns its code folder. Steps 3 and 4 fix both.

---

# Step 1 — Plan the layout

| Path | Owner:group | Mode | Who needs what | Step |
|---|---|---|---|---|
| `/srv/projects` | `root:devs` | `2770` + default ACL | devs read/write everything | 2 |
| `/opt/api` | `deploy:deploy` | `755` | deploy writes releases; api reads only | 3 |
| `/var/cache/api` | `api:api` | `750` | api cache | 3 |
| `/etc/myapps` | `root:root` | `755` | Config folder | 4 |
| `/etc/myapps/common.env` | `root:root` | `644` | Non-secret, everyone may read | 4 |
| `/etc/myapps/api.env` | `root:root` | `600` | Only systemd (root) reads it | 4 |
| `/etc/myapps/worker.env` | `root:worker` | `640` | systemd + worker's cron script | 4 |
| `/srv/uploads` | `root:appdata` | `2770` + default ACL | api + worker read/write; nobody else | 5 |
| `/var/log/myapps/api` | `api:adm` | `2750` + ACL for devs | api writes; adm + devs read | 6 |
| `/var/backups/appdb` | `worker:ops` | `2750` | worker writes; ops read | 7 |

Principles behind it (Note 5, Phase 13.2):
- **Apps can't change their own code or config.**
- **Each folder has one clear owner** (who writes it) and a group (who uses it).
- **"Other" gets nothing** wherever data is private.
- **Use ACLs only where one group isn't enough.**

---

# Step 2 — The team project folder

### Attempt 1: a group folder

```bash
sudo mkdir /srv/projects
sudo chgrp devs /srv/projects
sudo chmod 770 /srv/projects
```

Nimal creates a file:

```bash
echo "plan" > /srv/projects/plan.md
ls -l /srv/projects/plan.md
```
```
-rw-rw-r-- 1 nimal nimal 5 Oct 17 10:10 plan.md
```

Sachini tries to edit it:

```bash
echo "more" >> /srv/projects/plan.md
# bash: /srv/projects/plan.md: Permission denied
```

**Meaning:** the file's group is `nimal` (his **primary** group), not `devs`. For sachini, that's the "other" class: `r--`. (Why: Note 5, Phase 5.2 and Phase 4.)

### Attempt 2: SGID on the folder

```bash
sudo chmod 2770 /srv/projects
sudo chgrp devs /srv/projects/plan.md     # fix the existing file once
```

New files now inherit `devs`:

```bash
touch /srv/projects/todo.md      # as nimal
ls -l /srv/projects/todo.md
```
```
-rw-rw-r-- 1 nimal devs 0 Oct 17 10:15 todo.md
```

**Meaning:** group `devs`, and `rw` for the group because nimal's umask is `002`. Sachini can edit it. (Why: Note 5, Phase 8.2.)

### Problem: one person's umask breaks it

Kavindu's `~/.bashrc` (copied from his old company) has `umask 022`. His files:

```bash
touch /srv/projects/api-notes.md     # as kavindu
ls -l /srv/projects/api-notes.md
```
```
-rw-r--r-- 1 kavindu devs 0 ... api-notes.md
```

**Meaning:** SGID fixed the **group** but not the group's **permission bits**. With umask `022`, the group gets only `r`. (Why: Note 5, Phase 8.2 — "SGID fixes the group, not the bits".)

We can't rely on everyone's umask (or every editor and tool they use).

### Fix: default ACL

```bash
sudo setfacl -R -m g:devs:rwX /srv/projects       # existing files and folders
sudo setfacl -R -m d:g:devs:rwX,d:o::--- /srv/projects   # default for new ones
```

Kavindu creates another file:

```bash
touch /srv/projects/api-notes-2.md
ls -l /srv/projects/api-notes-2.md
getfacl /srv/projects/api-notes-2.md
```
```
-rw-rw----+ 1 kavindu devs 0 ... api-notes-2.md

# owner: kavindu
# group: devs
user::rw-
group::rwx      #effective:rw-
group:devs:rwX  #effective:rw-
mask::rw-
other::---
```

**Meaning:** the default ACL **replaced** his umask: group `devs` gets `rw` no matter what his umask says. (Why: Note 5, Phase 9.5.) The `+` in `ls -l` shows an ACL is present.

### Problem: files moved in keep their old settings

Nimal drafts a design doc in his home, then moves it:

```bash
mv ~/design.md /srv/projects/
ls -l /srv/projects/design.md
```
```
-rw------- 1 nimal nimal 2048 ... design.md
```

**Meaning:** `mv` on the same filesystem is only a rename — it keeps the owner, group, mode, and (no) ACL. Neither SGID nor the default ACL applies, because no new file was created. (Why: Note 5, Phase 5.2.)

**Fix:**

```bash
cp ~/design.md /srv/projects/ && rm ~/design.md   # copy = new file → gets devs + ACL
# or, after a mv:
chgrp devs /srv/projects/design.md && chmod g+rw /srv/projects/design.md
```

Tell the team: **copy into shared folders, don't move.**

### Decision: can teammates delete each other's files?

With `rwx` on the folder, any dev can delete any file in it (Note 5, Phase 3.5). For a team project folder that's fine — use git for history. If you don't want that, add the sticky bit (`chmod +t`) so people can only delete their own files.

---

# Step 3 — Code releases the app can't modify

### The current problem

At some point someone ran `chown -R api:api /opt/api` "because the app needs access".

```bash
sudo -u api test -w /opt/api/current/server.js && echo WRITABLE
```
```
WRITABLE
```

**Meaning:** if the API is ever broken into, the attacker can rewrite `server.js` — a backdoor that survives restarts. The app only needs to **read** its code. (Why: Note 5, Phase 13.2.)

### New layout

```
/opt/api                       deploy:deploy 755
├── releases/                  deploy:deploy 755
│   ├── 2026-10-10-1/          deploy:deploy dirs 755, files 644
│   └── 2026-10-17-1/
└── current -> releases/2026-10-17-1      (symlink)
```

```bash
sudo chown -R deploy:deploy /opt/api
sudo chmod -R u=rwX,go=rX /opt/api
```

`u=rwX,go=rX` gives directories `755` and files `644`, and keeps `x` on files that already had it (like a CLI script). (Why: Note 5, Phase 6.3.)

Check:

```bash
sudo -u api test -r /opt/api/current/server.js && echo READ_OK
sudo -u api test -w /opt/api/current/server.js && echo WRITABLE || echo "not writable"
```
```
READ_OK
not writable
```

### Deploying a release (as `deploy`)

```bash
REL=/opt/api/releases/2026-10-17-2
mkdir "$REL"
tar -xzf /tmp/api-build.tgz -C "$REL"
(cd "$REL" && npm ci --omit=dev)

ln -s "$REL" /opt/api/current.new
mv -T /opt/api/current.new /opt/api/current     # swap the symlink in one step
```

Why `mv -T` instead of `ln -sfn`: `ln -sfn` deletes and then recreates the link, so there's a tiny moment with no `current`. `mv -T` replaces it in one rename, so the service never sees a missing path.

`deploy` can do this because it owns `/opt/api` (creating and renaming entries needs `w` on the **directory**, Note 5, Phase 3.2). Restarting the service needs sudo — that's Note 8.

### Problem: the app wanted to write inside its code folder

After the change, the API logs:

```
Error: EACCES: permission denied, mkdir '/opt/api/releases/2026-10-17-2/.cache'
```

**Meaning:** a library was writing its cache next to the code. That only worked before because `api` owned the code. The right fix is a proper cache folder, not giving write access back.

```bash
sudo install -d -o api -g api -m 750 /var/cache/api
```

Then point the library at it through an env var in `/etc/myapps/api.env` (Note 2), e.g. `CACHE_DIR=/var/cache/api`. (Note 8 shows systemd's `CacheDirectory=`, which creates this automatically.)

---

# Step 4 — Config and secrets

### The current problem

From Note 2, each service **owns** its env file:

```
600 api:api /etc/myapps/api.env
```

**Meaning:** the API can **edit its own secrets and config**. Same problem as the code in Step 3.

### Who actually reads each file?

This decides the narrowest permission that works:

| File | Read by | Why |
|---|---|---|
| `common.env` | systemd, cron scripts, people | Non-secret shared settings |
| `api.env` | **systemd only** | `EnvironmentFile=` is read by systemd **as root**, before it switches to the `api` user |
| `worker.env` | systemd **and** the worker's cron backup script | The cron script runs **as `worker`** and sources the file (Note 2, Step 8) |

### Apply

```bash
sudo chown root:root /etc/myapps && sudo chmod 755 /etc/myapps

sudo chown root:root   /etc/myapps/common.env && sudo chmod 644 /etc/myapps/common.env
sudo chown root:root   /etc/myapps/api.env    && sudo chmod 600 /etc/myapps/api.env
sudo chown root:worker /etc/myapps/worker.env && sudo chmod 640 /etc/myapps/worker.env
```

### Test

```bash
sudo -u api cat /etc/myapps/api.env
# cat: /etc/myapps/api.env: Permission denied     ← app can't read the file directly...

sudo systemctl restart api
PID=$(systemctl show api -p MainPID --value)
sudo tr '\0' '\n' < /proc/$PID/environ | grep -c DATABASE_URL
# 1                                               ← ...but systemd still gave it the values
```

```bash
sudo -u worker test -r /etc/myapps/worker.env && echo "worker can read"
sudo -u worker test -w /etc/myapps/worker.env || echo "worker can't write"
sudo -u worker test -r /etc/myapps/api.env    || echo "worker can't read api secrets"
```
```
worker can read
worker can't write
worker can't read api secrets
```

**Meaning:** each identity has exactly the access it needs. A broken-into API can't change its config, and the worker can't read the API's secrets.

### Writing config files safely

When updating a config file, set owner and mode in the same command so it never exists with the wrong ones:

```bash
sudo install -m 640 -o root -g worker worker.env.new /etc/myapps/worker.env
```

---

# Step 5 — Uploads shared between two services

`api` saves user uploads. `worker` processes them, moves results to `processed/`, and deletes the originals. Nobody else should see user files.

### Setup

```bash
sudo install -d -o root -g appdata -m 2770 /srv/uploads
```

### Problem: worker can't delete files

The API creates one folder per day. A day later, the worker logs:

```
PermissionError: [Errno 13] Permission denied: '/srv/uploads/2026-10-18/img-001.png'
```

**Step 1 — who is the worker process?**

```bash
WPID=$(systemctl show worker -p MainPID --value)
grep -E '^(Uid|Groups)' /proc/$WPID/status
```
```
Uid:    902 902 902 902
Groups: 902 2003
```

**Meaning:** it has `appdata` (2003). Groups are fine.

**Step 2 — the path:**

```bash
namei -l /srv/uploads/2026-10-18/img-001.png
```
```
drwxr-xr-x root root    /
drwxr-xr-x root root    srv
drwxrws--- root appdata uploads
drwxr-sr-x api  appdata 2026-10-18
-rw-r--r-- api  appdata img-001.png
```

**Meaning:** the day folder is `drwxr-sr-x` — group `appdata` has `r-x`, **no `w`**. Deleting a file needs `w` on its **directory** (Note 5, Phase 3.5). The API created that folder with its umask `022` (Step 0).

Also notice `img-001.png` is `-rw-r--r--`: readable by "other". The parent folder blocks others today, but if the file is ever copied elsewhere it stays world-readable.

### Fix: default ACL (works whatever each app's umask is)

```bash
sudo setfacl -R -m g:appdata:rwX,o::--- /srv/uploads          # existing content
sudo setfacl -R -m d:g:appdata:rwX,d:o::--- /srv/uploads      # new content
```

Next day's folder:

```bash
namei -l /srv/uploads/2026-10-19/img-001.png
```
```
drwxrws---+ root appdata uploads
drwxrws---+ api  appdata 2026-10-19
-rw-rw----+ api  appdata img-001.png
```

**Meaning:** new folders and files are group-writable and closed to others, no matter the API's umask. The worker can now move and delete. (Why: Note 5, Phase 9.5.)

The alternative is setting `UMask=0007` in both services' units. That works too, but only for those two processes — the ACL protects the folder no matter who writes into it. Note 8 sets a service umask anyway, as an extra layer.

### Check that devs can't see user uploads

```bash
sudo -u nimal ls /srv/uploads
# ls: cannot open directory '/srv/uploads': Permission denied
```

---

# Step 6 — Logs for three audiences, and an append-only audit log

`api` writes log files. Three audiences:

| Who | Group | Access |
|---|---|---|
| The API | owner `api` | Write |
| Auditor, ops (`dilini`, `ruwan`, `thirana`) | `adm` | Read |
| Developers | `devs` | Read |

Two groups need read → one group isn't enough → ACL. (Why: Note 5, Phase 9.1.)

### Setup

```bash
sudo install -d -o root -g root -m 755 /var/log/myapps
sudo install -d -o api  -g adm  -m 2750 /var/log/myapps/api
sudo setfacl -m g:devs:rX,d:g:devs:rX,d:g::rX,d:o::--- /var/log/myapps/api
```

- `2750 api:adm` — api writes; `adm` reads; SGID so new log files get group `adm`.
- ACL — `devs` read, now and for new files.

After the API writes `app.log`:

```bash
getfacl /var/log/myapps/api/app.log
```
```
# owner: api
# group: adm
user::rw-
group::r--
group:devs:r--
mask::r--
other::---
```

```bash
sudo -u dilini  tail -1 /var/log/myapps/api/app.log    # works
sudo -u sachini tail -1 /var/log/myapps/api/app.log    # works
sudo -u kavindu tail -1 /var/log/myapps/api/app.log    # works (devs)
sudo -u worker  tail -1 /var/log/myapps/api/app.log    # Permission denied
```

### Log rotation

`/etc/logrotate.d/myapps`:

```
/var/log/myapps/api/*.log {
    daily
    rotate 14
    compress
    missingok
    su api adm
    create 0640 api adm
}
```

`create 0640 api adm` sets owner and mode on the new log file. `su api adm` runs rotation as that user/group (logrotate requires it when the folder isn't root-owned). New files are created inside the folder, so they also get the default ACL.

### Append-only audit log

The API also writes `audit.log` (who changed what). Nobody — not even root by mistake — should be able to edit or truncate it.

```bash
sudo chattr +a /var/log/myapps/api/audit.log
lsattr /var/log/myapps/api/audit.log
```
```
-----a--------e------- /var/log/myapps/api/audit.log
```

Test as root:

```bash
sudo bash -c ': > /var/log/myapps/api/audit.log'
# bash: /var/log/myapps/api/audit.log: Operation not permitted

sudo rm /var/log/myapps/api/audit.log
# rm: cannot remove '...': Operation not permitted

sudo bash -c 'echo test >> /var/log/myapps/api/audit.log'   # works: appending is allowed
```

**Meaning:** "Operation not permitted" (EPERM), not "Permission denied" — that's the clue it's an **attribute**, not permission bits. (Why: Note 5, Phase 10 and 12.)

The API opens the file in append mode, so it keeps working. Rotation would fail, so exclude `audit.log` from the logrotate pattern above (it matches `*.log`) — rename it to `audit.trail`, or give it its own rule:

```
/var/log/myapps/api/audit.trail {
    monthly
    rotate 12
    prerotate
        chattr -a /var/log/myapps/api/audit.trail
    endscript
    postrotate
        chattr +a /var/log/myapps/api/audit.trail
    endscript
    create 0640 api adm
}
```

---

# Step 7 — Backups

The worker's nightly cron job (Note 2, Step 8) dumps the database. Ops need to read dumps to restore. Nobody else.

### Setup

```bash
sudo install -d -o worker -g ops -m 2750 /var/backups/appdb
```

- Owner `worker` → can create and delete dumps
- Group `ops` (SGID) → can read
- Other → nothing

### Problem: dumps are `644`

```bash
ls -l /var/backups/appdb
```
```
-rw-r--r-- 1 worker ops 52M Oct 18 02:00 db-2026-10-18.sql
```

**Meaning:** the cron job runs with umask `022`. Right now the folder (`750`) stops others from reaching the file (Note 5, Phase 3.4):

```bash
sudo -u nimal cat /var/backups/appdb/db-2026-10-18.sql
# Permission denied
namei -l /var/backups/appdb/db-2026-10-18.sql
# drwxr-s--- worker ops appdb    ← nimal is "other" here: no x, can't pass
```

But if someone copies the dump to `/tmp` for a quick restore test, `cp` keeps `644` → **readable by every user**.

**Fix:** set umask in the backup script so dumps are never readable by others:

```bash
#!/bin/bash
set -euo pipefail
umask 027                 # new files 640, new dirs 750
...
```

Next dump: `-rw-r----- worker ops`.

### Protect the monthly backup from deletion

The script also cleans old dumps (`find ... -mtime +30 -delete`). The first-of-month dump must survive cleanup, mistakes, and even a careless `rm` as root:

```bash
sudo chattr +i /var/backups/appdb/db-2026-11-01.sql
```

```bash
sudo -u worker find /var/backups/appdb -mtime +30 -delete
# find: cannot delete '/var/backups/appdb/db-2026-11-01.sql': Operation not permitted
```

**Meaning:** immutable files can't be deleted, renamed, or changed by anyone until the attribute is removed. When the retention period really ends: `sudo chattr -i file && sudo rm file`.

---

# Step 8 — "My SSH key stopped working"

Nimal wanted to share a file with sachini, so he ran:

```bash
chmod 775 ~
```

Next morning:

```
nimal@laptop:~$ ssh nimal@app-server-01
nimal@app-server-01: Permission denied (publickey).
```

### Diagnose (thirana, on the server)

```bash
sudo grep nimal /var/log/auth.log | tail -2
```
```
sshd[6120]: Authentication refused: bad ownership or modes for directory /home/nimal
```

```bash
namei -l /home/nimal/.ssh/authorized_keys
```
```
drwxr-xr-x root  root  /
drwxr-xr-x root  root  home
drwxrwxr-x nimal nimal nimal         ← group-writable
drwx------ nimal nimal .ssh
-rw------- nimal nimal authorized_keys
```

**Meaning:** sshd's `StrictModes` (on by default) refuses keys if the home folder, `~/.ssh`, or `authorized_keys` can be written by anyone other than the user. A group-writable home means someone in that group could replace `.ssh` and add their own key (Note 5, Phase 3.5).

### Fix

```bash
sudo chmod 750 /home/nimal
```

And the right way to share: put the file in `/srv/projects` (Step 2).

### The client side

On nimal's laptop, after copying his key from a USB stick:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
Permissions 0644 for '/home/nimal/.ssh/id_ed25519' are too open.
```

```bash
chmod 600 ~/.ssh/id_ed25519
```

---

# Step 9 — Full "Permission denied" walkthrough

Sachini (devs) reports:

```bash
tail /var/log/myapps/api/app.log
# tail: cannot open '/var/log/myapps/api/app.log' for reading: Permission denied
```

It worked in Step 6. Follow the method from Note 5, Phase 14.

**1. Who is she, really?**

```bash
id      # in her session
```
```
uid=1102(sachini) gid=1102(sachini) groups=1102(sachini),2001(devs),2002(ops),2004(ssh-users)
```

**Meaning:** she has `devs` in her live session. Not a stale-group problem.

**2. Can she get through the path?**

```bash
namei -l /var/log/myapps/api/app.log
```
```
drwxr-xr-x root   root   /
drwxr-xr-x root   root   var
drwxrwxr-x root   syslog log
drwxr-x--- root   adm    myapps     ← here
drwxr-s---+ api   adm    api
-rw-r-----+ api   adm    app.log
```

**Meaning:** `/var/log/myapps` was changed to `750 root:adm` (someone "tightened" it). Sachini isn't in `adm` → she's "other" → no `x` → she can't pass through, whatever the inner ACL says. (Why: Note 5, Phase 3.4.)

The folder holds only per-app subfolders, which have their own protection. Restore it:

```bash
sudo chmod 755 /var/log/myapps
sudo chgrp root /var/log/myapps
```

Try again:

```bash
tail /var/log/myapps/api/app.log
# tail: cannot open ... Permission denied
```

Still denied. Continue.

**3. Bits and owner on the file:**

```bash
stat -c '%a %A %U:%G' /var/log/myapps/api/app.log
```
```
600 -rw-------+ api:adm
```

**Meaning:** mode is now `600`. Not what logrotate creates (`640`).

**4. The ACL:**

```bash
getfacl /var/log/myapps/api/app.log
```
```
user::rw-
group::r--        #effective:---
group:devs:r--    #effective:---
mask::---
other::---
```

**Meaning:** someone ran `chmod 600 app.log` ("logs are sensitive"). On a file with an ACL, `chmod` on the group bits **sets the mask**. Mask `---` silently removes access for `devs` **and** `adm`, without deleting the entries. (Why: Note 5, Phase 9.4.)

**Fix:**

```bash
sudo chmod 640 /var/log/myapps/api/app.log      # mask back to r--
```

**5–8.** Parent directory, attributes, mounts, AppArmor — no need to check; the cause is found. But confirm:

**9. Test as her:**

```bash
sudo -u sachini test -r /var/log/myapps/api/app.log && echo yes
# yes
sudo -u dilini  test -r /var/log/myapps/api/app.log && echo yes
# yes
```

**Conclusion:** two separate causes, found in order. Widening permissions at the first error (e.g. `chmod 755` on the file) would have hidden the real problems.

---

# Step 10 — Docker bind mount with the wrong UID

The team tries running the worker in a container, using the host's uploads folder.

`Dockerfile` (simplified):

```dockerfile
FROM python:3.12-slim
RUN useradd -u 10001 appuser
USER 10001
...
```

```bash
docker run --rm -v /srv/uploads:/data myorg/worker
```
```
PermissionError: [Errno 13] Permission denied: '/data'
```

### Diagnose

```bash
docker run --rm myorg/worker id
```
```
uid=10001(appuser) gid=10001(appuser) groups=10001(appuser)
```

```bash
ls -ldn /srv/uploads
```
```
drwxrws---+ 4 0 2003 4096 ... /srv/uploads
```

**Meaning:** the kernel only sees **numbers**. The container process is UID 10001 with group 10001. On the host, `/srv/uploads` allows group 2003 and nothing for others. 10001 is neither → "other" → no access. (Why: Note 3, Phase 10.3 and Note 5, Phase 4.)

### Fix

Run the container with the **host's** worker identity:

```bash
docker run --rm --user 902:902 --group-add 2003 -v /srv/uploads:/data myorg/worker
```

Compose:

```yaml
services:
  worker:
    image: myorg/worker
    user: "902:902"
    group_add: ["2003"]
    volumes:
      - /srv/uploads:/data
```

Numeric IDs work even though the container's `/etc/passwd` has no `worker` user. The ACL from Step 5 applies to the container exactly as it does to the host service.

**Wrong fixes:** `chmod 777 /srv/uploads`, or `chown -R 10001 /srv/uploads` (breaks the host services). Match the numbers instead.

---

# Step 11 — The `chmod -R 777` incident, and `noexec` for uploads

During an outage at 2 AM, someone "fixed" an upload error with:

```bash
sudo chmod -R 777 /srv/uploads
```

### Find the damage

```bash
sudo find /srv -xdev -perm -0002 \( -type f -o -type d \) | head
```
```
/srv/uploads
/srv/uploads/2026-10-19
/srv/uploads/2026-10-19/img-001.png
...
```

**What this means in practice** (Note 5, "How this goes wrong"):
- Any local user, and any compromised service (e.g. the web server's `www-data`), can read every user's uploads.
- They can replace or plant files that the worker will then process.
- Every uploaded file is now **executable**.
- With an ACL present, `chmod 777` also set the mask to `rwx`, widening every ACL entry.

### Restore the design

```bash
sudo chmod -R u=rwX,g=rwX,o= /srv/uploads          # dirs 770, files 660
sudo find /srv/uploads -type f -exec chmod a-x {} +  # no executable uploads
sudo chown -R :appdata /srv/uploads
sudo chmod 2770 /srv/uploads
sudo setfacl -R -m g:appdata:rwX,o::--- /srv/uploads
sudo setfacl -R -m d:g:appdata:rwX,d:o::--- /srv/uploads
```

(A numeric `chmod 777` on a directory keeps an existing SGID bit, but it's simplest to set it again explicitly.)

Then find the original error the right way (Step 9's method). In this incident it was a stale group list in a service that hadn't been restarted after Note 4, Step 5 (Note 3, Phase 7.3).

### Extra layer: `noexec` on uploads

Uploaded files should never be run. Make the folder its own mount with `noexec,nosuid,nodev` using a bind mount of itself.

`/etc/fstab`:

```
/srv/uploads  /srv/uploads  none  bind,noexec,nosuid,nodev  0  0
```

```bash
sudo mount /srv/uploads
findmnt -o TARGET,OPTIONS /srv/uploads
```
```
TARGET       OPTIONS
/srv/uploads rw,nosuid,nodev,noexec,relatime
```

(If the options don't show, apply them with `sudo mount -o remount,bind,noexec,nosuid,nodev /srv/uploads`.)

Test as the API:

```bash
sudo -u api bash -c 'printf "#!/bin/sh\necho hi\n" > /srv/uploads/x.sh && chmod +x /srv/uploads/x.sh'
sudo -u api /srv/uploads/x.sh
# sudo: unable to execute /srv/uploads/x.sh: Permission denied
sudo -u api sh /srv/uploads/x.sh
# hi
sudo rm /srv/uploads/x.sh
```

**Meaning:** direct execution is blocked, but an interpreter can still read and run the file. `noexec` is an extra layer, not a full defence. (Why: Note 5, Phase 11.)

---

# Step 12 — Make it repeatable: layout script, ACL backup, audit

### Layout script

Like `accounts.sh` in Note 4, one script creates the layout the same way on every server. Safe to run many times.

`/root/provision/layout.sh`:

```bash
#!/bin/bash
# Create the folder layout from Note 6, Step 1. Run after accounts.sh.
set -euo pipefail

dir() {   # dir OWNER:GROUP MODE PATH
  mkdir -p "$3"
  chown "$1" "$3"
  chmod "$2" "$3"
}

file_perm() {   # file_perm OWNER:GROUP MODE PATH   (only if the file exists)
  if [ -e "$3" ]; then chown "$1" "$3"; chmod "$2" "$3"; fi
}

# Team project folder
dir root:devs 2770 /srv/projects
setfacl -m g:devs:rwX,d:g:devs:rwX,d:o::--- /srv/projects

# Code releases
dir deploy:deploy 0755 /opt/api
dir deploy:deploy 0755 /opt/api/releases
dir api:api       0750 /var/cache/api

# Config and secrets
dir root:root 0755 /etc/myapps
file_perm root:root   0644 /etc/myapps/common.env
file_perm root:root   0600 /etc/myapps/api.env
file_perm root:worker 0640 /etc/myapps/worker.env

# Uploads
dir root:appdata 2770 /srv/uploads
setfacl -m g:appdata:rwX,o::---,d:g:appdata:rwX,d:o::--- /srv/uploads

# Logs
dir root:root 0755 /var/log/myapps
dir api:adm   2750 /var/log/myapps/api
setfacl -m g:devs:rX,d:g:devs:rX,d:g::rX,d:o::--- /var/log/myapps/api

# Backups
dir worker:ops 2750 /var/backups/appdb

echo "layout OK"
```

It sets the **top-level** folders. Fixing existing content inside them (`-R`) stays a deliberate manual step, so a re-run never mass-changes data.

### Back up ACLs

ACLs are easy to lose in restores (Note 5, Phase 9.6). Save them:

```bash
sudo getfacl -R /srv/projects /srv/uploads /var/log/myapps > /root/provision/acls-$(date +%F).txt
# later, if needed:
sudo setfacl --restore=/root/provision/acls-2026-10-17.txt
```

### Regular audit

Each line should print **nothing** (or only what you expect).

```bash
# World-writable files and dirs
sudo find / -xdev -type f -perm -0002 2>/dev/null
sudo find / -xdev -type d -perm -0002 ! -perm -1000 2>/dev/null

# Orphaned files
sudo find / -xdev \( -nouser -o -nogroup \) 2>/dev/null

# Group/other-writable files in /etc, systemd units, cron
sudo find /etc /etc/systemd /etc/cron* -xdev -type f -perm /022 2>/dev/null

# Apps must not write their own code or config
for u in api worker; do
  sudo -u $u find /opt/api /etc/myapps -writable 2>/dev/null | sed "s/^/$u can write: /"
done

# Secrets: expected 600/640, never readable by other
stat -c '%a %U:%G %n' /etc/myapps/*

# Attributes in use (expected: audit log, monthly backups)
sudo lsattr -R /var/log/myapps /var/backups/appdb 2>/dev/null | grep -E '^[^ ]*[ia]'

# Mount options on uploads
findmnt -o TARGET,OPTIONS /srv/uploads
```

---

# State of the server after Note 6

This is the starting point for Note 8.

| Path | Owner:group | Mode | Extra |
|---|---|---|---|
| `/srv/projects` | root:devs | 2770 | default ACL devs rwX |
| `/opt/api` (+ `releases/`) | deploy:deploy | 755 | `current` symlink |
| `/var/cache/api` | api:api | 750 | |
| `/etc/myapps` | root:root | 755 | |
| `common.env` / `api.env` / `worker.env` | root:root / root:root / root:worker | 644 / 600 / 640 | |
| `/srv/uploads` | root:appdata | 2770 | default ACL appdata rwX; `noexec,nosuid,nodev` bind mount |
| `/var/log/myapps/api` | api:adm | 2750 | ACL devs rX; `audit.trail` append-only |
| `/var/backups/appdb` | worker:ops | 2750 | monthly dumps immutable |

**Still open for Note 8:**
- `deploy` can install releases but can't restart the service yet (needs a narrow sudo rule).
- The API wants port 80 without root.
- Services still run with umask `022` and full access to the rest of the filesystem — systemd can restrict that.

---

# Scenario recap

| Step | What happened | Concept (Note 5) |
|---|---|---|
| 0 | Checked umask of users and services, system files | umask sources, standard modes (7.3, 13.1) |
| 1 | Planned owner, group, mode for every folder | App layout principles (13.2) |
| 2 | Team folder: wrong group, umask, `mv` | SGID, default ACL, `mv` keeps metadata (5.2, 8.2, 9.5) |
| 3 | App could rewrite its code | Code owned by `deploy`, read-only for app; capital `X` (6.3, 13.2) |
| 4 | Apps owned their secrets | Narrowest owner/mode for who really reads (13.2) |
| 5 | Worker couldn't delete uploads | Delete needs directory `w`; default ACL beats umask (3.5, 9.5) |
| 6 | Logs for two groups; tamper-proof audit log | ACLs, `chattr +a`, EPERM vs EACCES (9, 10, 12) |
| 7 | Backups `644`; protect monthly dumps | Directory protects contents; umask in scripts; `chattr +i` (3.4, 7, 10) |
| 8 | SSH key refused after `chmod 775 ~` | StrictModes, directory write risk (3.5, 13.1) |
| 9 | Log unreadable: path, then ACL mask | Debugging method; mask set by `chmod` (3.4, 9.4, 14) |
| 10 | Container couldn't use uploads | Kernel checks numbers; match UID/GID (4) |
| 11 | `chmod -R 777` cleanup; `noexec` uploads | Restoring design; mount options (6.3, 11) |
| 12 | Script, ACL backup, audit | Repeatability, `getfacl -R`, audits |

---

# Troubleshooting table

| Symptom | Likely cause | Check | Fix |
|---|---|---|---|
| Teammate can't edit a file in a shared folder | Wrong group (no SGID) or group lacks `w` (umask) | `ls -l`, `getfacl` | SGID on folder + default ACL; `chgrp`/`chmod` existing |
| File in shared folder has the creator's private group | Created before SGID, or `mv`'d in | `ls -l` | `chgrp`; copy instead of move |
| Can read files but can't delete them | No `w` on the **directory** | `namei -l`, `getfacl` on parent | Default ACL / group write on directory |
| "Permission denied" on a `644` file | A directory on the path lacks `x` for you | `namei -l` | Fix the directory, not the file |
| ACL entry exists but access denied | Mask limits it (`#effective:---`) | `getfacl` | `chmod g+...` or `setfacl -m m::rX` |
| New files ignore your umask | Folder has a default ACL (expected) | `getfacl -d dir` | Adjust the default ACL |
| "Operation not permitted" even as root | Immutable / append-only attribute | `lsattr` | `chattr -i` / `-a` (on purpose) |
| `./script`: Permission denied, file is `755` | `noexec` mount | `findmnt -T script` | Run from another location, or that's intended |
| SSH key refused, key is correct | Home/`.ssh`/`authorized_keys` too open | `auth.log`, `namei -l` | `chmod 750 ~`, `700 ~/.ssh`, `600 authorized_keys` |
| ssh client: "UNPROTECTED PRIVATE KEY FILE" | Private key readable by others | `ls -l ~/.ssh/id_*` | `chmod 600` |
| Container gets Permission denied on a bind mount | UID/GID inside ≠ allowed on host | `docker run ... id`, `ls -ln` | `--user` + `--group-add` with host IDs |
| App writes cache/temp into code folder | Code was writable before | App error path | Proper `/var/cache/<app>`; keep code read-only |
| Files created by a service readable by all | Service umask `022` | `grep Umask /proc/PID/status` | Default ACL with `o::---`, `UMask=` (Note 8), umask in scripts |
| ACLs gone after restore | Backup/restore without ACL flags | `getfacl` | `rsync -A`, `tar --acls`, `setfacl --restore` |
| Service can't access something after group change | Old group list in running process | `/proc/PID/status` Groups | Restart the service |

---

# Appendix — Command reference

## Inspect

| Command | Purpose |
|---|---|
| `stat -c '%a %A %U:%G %n' paths` | Mode and owner, compact |
| `namei -l /full/path` | Every directory on the path |
| `getfacl path` / `getfacl -d dir` | ACL / default ACL only |
| `lsattr path` / `lsattr -d dir` | Attributes |
| `findmnt -T path` / `findmnt -o TARGET,OPTIONS path` | Mount options |
| `grep -E '^(Uid\|Groups\|Umask)' /proc/PID/status` | A service's identity and umask |
| `sudo -u user test -r/-w/-x path && echo yes` | Test access as someone |
| `sudo -u user find dir -writable` | Everything a user can write under `dir` |

## Set up folders and files

| Command | Purpose |
|---|---|
| `install -d -o OWNER -g GROUP -m MODE dir` | Create a folder with owner + mode |
| `install -m 640 -o root -g grp src dst` | Place a file with owner + mode |
| `chmod 2770 dir` | SGID team/shared folder |
| `chmod -R u=rwX,go=rX dir` | Code: dirs 755, files 644 |
| `chmod -R u=rwX,g=rwX,o= dir` | Shared data: dirs 770, files 660 |
| `find dir -type f -exec chmod a-x {} +` | Remove execute from all files |
| `ln -s target link.new && mv -T link.new link` | Swap a symlink in one step |

## ACLs

| Command | Purpose |
|---|---|
| `setfacl -R -m g:grp:rwX dir` | Existing content |
| `setfacl -R -m d:g:grp:rwX,d:o::--- dir` | New content |
| `setfacl -m g:devs:rX,d:g:devs:rX dir` | Read-only access for a second group |
| `chmod 640 file` (with ACL) | Sets the mask to `r--` |
| `getfacl -R dirs > acls.txt` / `setfacl --restore=acls.txt` | Back up / restore ACLs |

## Attributes and mounts

| Command | Purpose |
|---|---|
| `chattr +a file` / `-a` | Append-only on / off |
| `chattr +i file` / `-i` | Immutable on / off |
| `/srv/uploads /srv/uploads none bind,noexec,nosuid,nodev 0 0` | fstab line: bind mount with options |
| `mount -o remount,bind,noexec,nosuid,nodev /path` | Apply options now |

## Containers

| Command | Purpose |
|---|---|
| `docker run --user UID:GID --group-add GID ...` | Run with host-matching IDs |
| Compose `user:` / `group_add:` | Same in compose |
| `docker run --rm IMAGE id` | See the image's default identity |

## Audit

| Command | Purpose |
|---|---|
| `find / -xdev -type f -perm -0002` | World-writable files |
| `find / -xdev -type d -perm -0002 ! -perm -1000` | World-writable dirs without sticky |
| `find / -xdev \( -nouser -o -nogroup \)` | Orphaned files |
| `find /etc /etc/systemd /etc/cron* -type f -perm /022` | Writable config/units/cron by group/other |
| `lsattr -R dir \| grep -E '^[^ ]*[ia]'` | Files with `i` or `a` |