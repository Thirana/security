# Filesystem Structure — Practice (Foundation Note)

A hands-on tour of the tree on your lab server. The goal isn't to change anything — it's to *see* the layout from the theory note on a real machine, so paths stop being abstract. Each step shows the command, the output you should get, and what it tells you.

**Theory for this note:** the Filesystem Structure theory note. **Runs on:** `app-server-01` from Note 0 (the lab). You can do this before Note 1 — it touches almost nothing, so no snapshot is needed.

Log in first:

```bash
ssh app-server-01
```

Outputs below are from an Ubuntu 24.04 Multipass VM (`aarch64`). Yours will differ in sizes, dates, and device names; the shape is what matters.

---

## Contents

- Step 0 — The top of the tree
- Step 1 — Follow one program through the `/usr` merge
- Step 2 — `/etc`: the control room
- Step 3 — `/var`: data that changes
- Step 4 — The fake filesystems: `/proc`, `/sys`, `/dev`
- Step 5 — Disks and mounts
- Step 6 — Moving around: paths in practice
- Step 7 — Trace one application across the tree
- Appendix — command reference

---

# Step 0 — The top of the tree

```bash
ls -l /
```
```
lrwxrwxrwx   1 root root     7 Apr 22  2024 bin -> usr/bin
drwxr-xr-x   5 root root  4096 Oct 17 18:32 boot
drwxr-xr-x  16 root root  3740 Oct 23 14:02 dev
drwxr-xr-x 109 root root  4096 Oct 23 14:04 etc
drwxr-xr-x   4 root root  4096 Oct 23 14:02 home
lrwxrwxrwx   1 root root     7 Apr 22  2024 lib -> usr/lib
drwx------   2 root root 16384 Oct 17 18:31 lost+found
drwxr-xr-x   2 root root  4096 Oct 17 18:27 media
drwxr-xr-x   2 root root  4096 Oct 17 18:27 mnt
drwxr-xr-x   3 root root  4096 Oct 23 14:04 opt
dr-xr-xr-x 125 root root     0 Oct 23 14:02 proc
drwx------   3 root root  4096 Oct 17 18:31 root
drwxr-xr-x  28 root root   860 Oct 23 14:05 run
lrwxrwxrwx   1 root root     8 Apr 22  2024 sbin -> usr/sbin
drwxr-xr-x   3 root root  4096 Oct 23 14:04 srv
dr-xr-xr-x  13 root root     0 Oct 23 14:02 sys
drwxrwxrwt  13 root root  4096 Oct 23 14:04 tmp
drwxr-xr-x  11 root root  4096 Oct 17 18:27 usr
drwxr-xr-x  13 root root  4096 Oct 23 14:02 var
```

**Meaning:** this is the whole top level from Phase 2. Three things to notice right away:
- `bin`, `lib`, `sbin` start with `l` and show `-> usr/...` — they're **symlinks** into `/usr` (the merge, Phase 2).
- `proc` and `sys` have **size 0** and a date of this boot — they aren't on disk; the kernel generates them (Phase 3).
- `tmp` ends in `t` (`drwxrwxrwt`) — the sticky bit, so anyone can write there but only delete their own files (Note 5 covers it).

`lost+found` is where `fsck` puts recovered fragments after a disk check; ignore it. `snap` appears on Ubuntu because of snap packages.

---

# Step 1 — Follow one program through the `/usr` merge

```bash
which bash
```
```
/usr/bin/bash
```

**Meaning:** `which` searches `$PATH` and reports the **real** location — `/usr/bin/bash`, not `/bin/bash`. Yet `/bin/bash` also works. Why? Because `/bin` *is* `/usr/bin`:

```bash
ls -ld /bin /sbin /lib
readlink -f /bin/bash
```
```
lrwxrwxrwx 1 root root 7 Apr 22  2024 /bin -> usr/bin
lrwxrwxrwx 1 root root 8 Apr 22  2024 /sbin -> usr/sbin
lrwxrwxrwx 1 root root 7 Apr 22  2024 /lib -> usr/lib
/usr/bin/bash
```

**Meaning:** `/bin` is a symlink to `usr/bin`, so `/bin/bash` and `/usr/bin/bash` are the same file reached by two names. `readlink -f` follows the links to the one real path. Everything "essential" now lives under `/usr`; the old top-level names still work because they point there. (On an `x86_64` box you'd also see `/lib64`; on this ARM VM there's no `/lib64`.)

---

# Step 2 — `/etc`: the control room

```bash
ls /etc | head
ls -l /etc/ssh/sshd_config /etc/passwd /etc/fstab
```
```
ModemManager
PackageKit
X11
adduser.conf
alternatives
apparmor
...
-rw-r--r-- 1 root root 3517 ... /etc/ssh/sshd_config
-rw-r--r-- 1 root root 1708 ... /etc/passwd
-rw-r--r-- 1 root root  146 ... /etc/fstab
```

**Meaning:** everything in `/etc` is a **text file** — no programs. These three alone describe how you log in (`sshd_config`), who the accounts are (`passwd`), and what gets mounted at boot (`fstab`). Confirm there are no binaries:

```bash
file /etc/passwd /etc/ssh/sshd_config
```
```
/etc/passwd:          ASCII text
/etc/ssh/sshd_config: ASCII text
```

**Conclusion:** to understand a server, you read `/etc`. (`/etc` has over a hundred entries — the `head` just shows the first few.) Later notes spend most of their time here.

---

# Step 3 — `/var`: data that changes

```bash
ls /var
ls /var/log | head
```
```
backups  cache  crash  lib  local  lock  log  mail  opt  run  snap  spool  tmp
README  alternatives.log  apport.log  apt  auth.log  btmp  dmesg  journal  ...
```

**Meaning:** `/var` is the opposite of `/usr` — everything here grows and changes while the system runs. `/var/log` has the logs (Note 13), `/var/lib` holds service state and databases, `/var/spool` holds queues like cron.

```bash
sudo du -sh /var/log /usr 2>/dev/null
```
```
25M   /var/log
1.6G  /usr
```

**Meaning:** `/usr` is large but static (the installed software); `/var/log` is small now but only grows. This is exactly why they're kept on separate branches — and sometimes separate filesystems (Step 5).

---

# Step 4 — The fake filesystems: `/proc`, `/sys`, `/dev`

These look like directories but aren't on any disk (Phase 3). Read them like files:

```bash
cat /proc/version
```
```
Linux version 6.8.0-147-generic ... (Ubuntu 24.04) ...
```

Each running process has a folder under `/proc`. Your shell's own is `/proc/self`:

```bash
ls -F /proc/self/ | head -8
```
```
attr/
autogroup
auxv
cgroup
clear_refs
cmdline
comm
coredump_filter
```

```bash
cat -v /proc/self/cmdline; echo
```
```
cat^@-v^@/proc/self/cmdline^@
```

**Meaning:** `/proc/self/cmdline` is the command line of the process reading it — here, your `cat`. The `^@` are the NUL bytes separating the arguments. Nothing is stored; the kernel builds this the instant you read it. Note 9 lives in this directory.

Now the device files:

```bash
ls -l /dev/null /dev/sda /dev/urandom
```
```
crw-rw-rw- 1 root root 1, 3 ... /dev/null
brw-rw---- 1 root disk 8, 0 ... /dev/sda
crw-rw-rw- 1 root root 1, 9 ... /dev/urandom
```

**Meaning:** `c` = character device (a byte stream: `/dev/null`, `/dev/urandom`), `b` = block device (a disk: `/dev/sda`). Instead of a size they show a **major, minor** number (the driver and which device). Note that `/dev/sda` is readable by group `disk` — reading it directly is reading the raw disk, under every file permission (Note 0, Phase 3.3).

---

# Step 5 — Disks and mounts

The one tree you've been walking is actually assembled from several filesystems (Phase 4).

```bash
lsblk
```
```
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda       8:0    0   12G  0 disk
├─sda1    8:1    0   11G  0 part /
├─sda15   8:15   0   99M  0 part /boot/efi
└─sda16 259:0    0  923M  0 part /boot
```

**Meaning:** one disk (`sda`, 12G) split into three partitions. `sda1` is mounted at `/` (the root filesystem), and `/boot` and `/boot/efi` are **separate filesystems** grafted onto the tree at those points.

```bash
df -h
```
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        11G  2.0G  8.7G  19% /
/dev/sda16      891M   61M  768M   8% /boot
...
```

```bash
findmnt
```
```
TARGET        SOURCE     FSTYPE OPTIONS
/             /dev/sda1  ext4   rw,relatime,discard,errors=remount-ro,commit=30
└─/boot       /dev/sda16 ext4   rw,relatime
  └─/boot/efi /dev/sda15 vfat   rw,relatime,fmask=0077,...
```

**Meaning:** `findmnt` is the clearest view — the tree, which device is behind each mount point, the filesystem type, and the options. Note `/boot/efi` is `vfat` (the EFI firmware needs a FAT filesystem); everything else is `ext4`.

Where does this come from at boot? `/etc/fstab`:

```bash
grep -v '^#' /etc/fstab | sed '/^$/d'
```
```
LABEL=cloudimg-rootfs   /         ext4  discard,commit=30,errors=remount-ro  0 1
LABEL=BOOT              /boot     ext4  defaults                             0 2
LABEL=UEFI              /boot/efi vfat  umask=0077                           0 1
```

**Meaning:** each line is device → mount point → type → options. Devices are named by **LABEL** (not `/dev/sda1`) so a changed device name doesn't break boot (Phase 4.4). Compare with the live kernel view:

```bash
cat /proc/mounts | grep ext4
```
```
/dev/sda1 / ext4 rw,relatime,discard,errors=remount-ro,commit=30 0 0
/dev/sda16 /boot ext4 rw,relatime 0 0
```

**Conclusion:** `fstab` is the plan; `/proc/mounts` (and `findmnt`) is what actually happened. Note 6 adds a mount of its own with `noexec,nosuid` options, and Note 14 chases a filesystem that fills up.

---

# Step 6 — Moving around: paths in practice

```bash
cd /var/log
pwd
cd ..
pwd
cd -
pwd
```
```
/var/log
/var
/var/log
```

**Meaning:** `..` went up to the parent; `cd -` jumped back to where you were. These are the `.`/`..` entries from Phase 5. See that they're real directory entries:

```bash
ls -a /etc/ssh | head -4
```
```
.
..
moduli
ssh_config
```

**Meaning:** every directory contains `.` (itself) and `..` (its parent). Now absolute vs relative:

```bash
cd /etc
head -1 os-release        # relative: /etc + os-release
head -1 /etc/os-release   # absolute: same file, from anywhere
realpath ssh/../fstab     # a detour up through ssh/ and back
```
```
PRETTY_NAME="Ubuntu 24.04.5 LTS"
PRETTY_NAME="Ubuntu 24.04.5 LTS"
/etc/fstab
```

**Meaning:** the relative path worked only because you were in `/etc`; the absolute path works from anywhere. `realpath` collapses the detour (into `ssh/`, back out with `..`) into the one true path. This is the difference Note 1 relies on for `./script` vs a command on `$PATH`.

---

# Step 7 — Trace one application across the tree

This is Phase 6 made concrete. The lab's API (set up in Note 0) scatters across the tree by purpose. Find its pieces:

```bash
sudo find /opt /etc/myapps /var/log/myapps /srv -name '*api*' -o -path '*myapps*' 2>/dev/null | sort
```
```
/etc/myapps
/etc/myapps/api.env
/etc/myapps/common.env
/opt/api
/var/log/myapps
/var/log/myapps/api
/var/log/myapps/api/app.log
```

**Meaning:** one application, three locations, each chosen by *kind* of file:

| Found | Kind | Branch |
|---|---|---|
| `/opt/api/server.js` | the program | hand-installed software → `/opt` |
| `/etc/myapps/*.env` | its configuration | config → `/etc` |
| `/var/log/myapps/api/app.log` | its log | variable data → `/var` |

```bash
cat /etc/myapps/common.env
tail -1 /var/log/myapps/api/app.log
```
```
PORT=3000
2026-10-23 api started
```

**Conclusion:** finding the program in `/opt` told you to look in `/etc` for its config and `/var/log` for its activity — and they were there. That reflex ("one find points at the next") is the whole payoff of the layout. The security version of it — *where secrets and backdoors hide in this same tree* — is the post-exploitation filesystem note.

---

# Appendix — command reference

## Walking the tree

```bash
ls -l /                      # the top level
ls -la ~                     # a home, dotfiles included
ls -a <dir>                  # see . and ..
file <path>                  # what kind of thing is it
stat <path>                  # size, owner, inode, timestamps
du -sh <dir>                 # how big is this subtree
readlink -f <path>           # follow symlinks to the real target
realpath <path>              # resolve a path (and any ..) to absolute
```

## Disks and mounts

```bash
lsblk                        # disks and partitions
lsblk -f                     # ... with filesystem type, LABEL, UUID
df -h                        # mounted filesystems, free space
findmnt                      # the mount tree (clearest)
mount | column -t            # every mount with options
cat /proc/mounts             # live mounts from the kernel
cat /etc/fstab               # mounts set up at boot
```

## The fake filesystems

```bash
cat /proc/version            # kernel version
cat /proc/self/cmdline       # the reading process's command line
ls /proc/<pid>/              # one process's live directory
ls /sys/class/net            # network interfaces
ls -l /dev/null /dev/sda     # device files and their types
```

## Paths

```bash
pwd                          # where am I
cd <abs>  /  cd <rel>        # absolute vs relative
cd ..                        # up one
cd -                         # previous directory
cd                           # home
```
