# L07 — ca-files · What the App Left Behind
 
**Act II · Credential Access · T1552.001**
**Status:** Solved
**Flag:** `bl_03a9e330aabdd2489a67b929423806ca`
 
---
 
## Challenge brief
 
> Now that you own the box, read it like an attacker. Configs, connection strings,
> dotfiles, history. Someone stored a key where it was easy, not where it was safe.
 
MITRE tag T1552.001 is "Credentials in Files" — the technique of finding secrets that
were written to disk in plaintext or weakly protected form. The challenge title "What
the App Left Behind" signals that an application was deployed on this box and left
credentials in its config files.
 
You land as root. There is no escalation to do. The job is pure enumeration: find what
was carelessly stored.
 
---
 
## What MITRE ATT&CK tags mean
 
Each challenge in the Phantom track is tagged with a MITRE ATT&CK ID. This is not a
CTF convention — it is a real-world framework.
 
MITRE is a US non-profit research organization. ATT&CK (Adversarial Tactics, Techniques,
and Common Knowledge) is a publicly maintained knowledge base that documents how real
attackers actually operate, based on observed incidents, malware analysis, and threat
intelligence reports. Security teams, red teamers, and detection engineers all use it
as a shared language.
 
The framework organizes attacker behavior into:
- **Tactics** — the goal (e.g. "Credential Access" — the attacker wants credentials)
- **Techniques** — how they achieve it (e.g. T1552 — "Unsecured Credentials")
- **Sub-techniques** — a more specific variation (e.g. T1552.001 — credentials
  specifically found in files)
So T1552.001 means: the attacker found credentials that were stored insecurely in a
file on disk. The full catalog is at attack.mitre.org — each entry lists real-world
threat groups observed using the technique, detection guidance, and mitigations.
 
The Phantom track tags each level to connect what you are doing in the CTF to real
attacker behavior documented in the wild.
 
---
 
## How the challenge was designed
 
This level has three layers of misdirection before the real answer:
 
**Layer 1 — named decoys.** Two of the credentials on the box are labelled as fakes by
the people who built the challenge. `cache.url` contains a Redis password literally
named `notthisone`. A dotfile called `.profile.d-decoy` exports a token called
`LEGACY_TOKEN` with the value `deadbeefdeadbeef...` — `deadbeef` is a well-known
placeholder value in programming, and the filename contains the word `decoy`. These are
designed to be found quickly and dismissed.
 
**Layer 2 — the plausible-looking credential.** `database.yml` contains a real-looking
PostgreSQL password: `506ca8261d9d23c860467e1d48753a7a`. It looks like an MD5 hash.
The natural assumption is that this is the answer. It isn't — and the challenge is
designed so that you try it, fail, and have to think harder.
 
**Layer 3 — the vault.** `vault.enc` is an OpenSSL-encrypted binary blob. It contains
the actual secret the grader wants, but it is unreadable until decrypted. The decryption
key is the database password from layer 2. So the database password is not the answer
— it is the key to the answer. The real credential is locked inside the vault, and the
key to the vault was stored carelessly in a plaintext config file next to it.
 
This design teaches two things at once:
- Credentials in config files are a real attack surface even when the config itself is
  not the final target.
- "Protecting" a secret with encryption means nothing if you store the key in the same
  place as the encrypted file.
---
 
## Why the database password was used as the vault key
 
This is the realistic part of the challenge — it reflects how this mistake actually
happens in production.
 
The app needs two things at runtime: the database password (to connect to PostgreSQL)
and the vault passphrase (to decrypt `vault.enc` and get whatever sensitive value is
inside — maybe an API key or a signing secret).
 
A developer thinks: "I already have a config file for credentials. I'll store the vault
passphrase there too. And since the app already has the database password in memory at
startup, I'll just reuse that as the vault passphrase — one less thing to manage."
 
This is called **credential reuse** — using the same secret for multiple purposes. The
problem is that it collapses two separate security boundaries into one. If an attacker
reads `database.yml`, they get both the database access and the vault key simultaneously.
 
The encryption on the vault itself was technically sound — AES-256 is strong. The
mistake was not in the algorithm but in key management: the key was stored right next to
the thing it was supposed to protect. It is the equivalent of locking a safe and leaving
the combination written on a sticky note on the door.
 
---
 
## Initial hypothesis
 
Root shell, credential access challenge, app files present. The most common places an
app leaves credentials:
 
1. Config files (`database.yml`, `.env`, `config.yml`)
2. Shell history (commands that included passwords inline)
3. Dotfiles in home directories
4. Environment variables (runtime secrets passed via env)
5. SSH keys
Start with history and work outward.
 
---
 
## Enumeration log
 
### Step 1 — Shell history
 
```sh
cat /root/.ash_history
```
 
Alpine uses `ash`, so the history file is `.ash_history`, not `.bash_history`. Result:
only commands from the current session. Nothing useful.
 
**Conclusion:** root's history was clean. Move to user enumeration.
 
---
 
### Step 2 — User enumeration
 
```sh
cat /etc/passwd
```
 
Result: one real user besides root — `operator` (uid 1000, home `/home/operator`,
shell `/bin/sh`). All other entries have `/sbin/nologin`, meaning they are system
accounts that cannot log in interactively.
 
**Conclusion:** `operator` is the app user. Check their home directory.
 
---
 
### Step 3 — Operator's home directory
 
```sh
ls -la /home/operator/
```
 
The `-la` flag is essential here. Without `-a`, the shell glob skips dotfiles entirely
on Alpine's ash. Result: completely empty — no files, no dotfiles.
 
```sh
cat /home/operator/.ash_history 2>/dev/null
cat /home/operator/.sh_history 2>/dev/null
```
 
No history files for operator either.
 
**Conclusion:** The credential was not left in the user's home. The challenge title says
"what the app left behind" — look for the application itself.
 
---
 
### Step 4 — Application directory
 
```sh
ls /opt
```
 
Result: a `webapp` directory. This is the deployed application.
 
```sh
ls -la /opt/webapp/
ls -la /opt/webapp/config/
ls -la /opt/webapp/shared/
```
 
Found three files:
- `/opt/webapp/config/cache.url` — 42 bytes
- `/opt/webapp/config/database.yml` — 118 bytes
- `/opt/webapp/shared/vault.enc` — 64 bytes, permissions `rw-------` (root only)
```sh
cat /opt/webapp/config/cache.url
```
```
redis://:notthisone@cache.internal:6379/0
```
 
The password in this Redis URL is literally `notthisone`. Set aside.
 
```sh
cat /opt/webapp/config/database.yml
```
```yaml
production:
  adapter: postgresql
  host: db.internal
  username: webapp
  password: 506ca8261d9d23c860467e1d48753a7a
```
 
A real-looking PostgreSQL password. This looks like the answer.
 
```sh
cat /opt/webapp/shared/vault.enc
```
 
Binary output — garbage characters. This is an encrypted file.
 
**Conclusion:** Three candidates found. Tried the database password with the grader —
no match. The Redis password is named to tell you it's wrong. The vault is unreadable
without decryption. Need to keep looking.
 
---
 
### Step 5 — Hidden files across the filesystem
 
The challenge description specifically mentioned "dotfiles." The `ls -la` from earlier
only covered known directories. Hidden files can exist anywhere — search the whole
filesystem:
 
```sh
find / -not -path '/proc/*' -not -path '/sys/*' -name '.*' -type f 2>/dev/null
```
 
**What this command does:**
- `find /` — start from the root, search everything
- `-not -path '/proc/*'` and `-not -path '/sys/*'` — skip `/proc` and `/sys`. These
  are virtual filesystems that expose kernel internals. `/proc` alone contains thousands
  of entries for every running process. Searching them is slow, produces noise, and on
  some boxes `/proc` entries can cause `find` to hang. Always exclude them.
- `-name '.*'` — match only files whose name starts with `.` (dotfiles)
- `-type f` — files only, not directories
- `2>/dev/null` — suppress permission-denied errors, which appear constantly when
  searching system directories as a non-root user (here we are root, but it's a good
  habit)
Result included `/root/.profile.d-decoy`. Read it:
 
```sh
cat /root/.profile.d-decoy
```
```
export LEGACY_TOKEN=deadbeefdeadbeefdeadbeefdeadbeef
```
 
Named `decoy`, value is `deadbeef` — a well-known placeholder. This is another
deliberate red herring.
 
Also found `/.dockerenv` — a standard empty file Docker creates in every container to
mark that the environment is inside Docker. Not useful for credentials.
 
**Conclusion:** Two more decoys found. No new real credentials from dotfiles alone.
 
---
 
### Step 6 — Finding files by modification time
 
A useful technique when you want to isolate what was planted for the challenge vs. what
came with the base OS: compare modification timestamps. The original setup files have
one date; anything added later stands out.
 
```sh
find / -not -path '/proc/*' -not -path '/sys/*' -newer /opt/webapp/config/cache.url -type f 2>/dev/null
```
 
**What this command does:**
- `-newer /opt/webapp/config/cache.url` — find files modified more recently than
  `cache.url`. `cache.url` was created on Sep 7 (the box setup date). Anything newer
  was added after the base image was built — which means it was planted for the
  challenge.
Result: `/etc/profile`, `/etc/hosts`, `/etc/hostname`, `/etc/phantom-banner`,
`/etc/resolv.conf`, the tmp files we wrote, `/opt/webapp/config/database.yml`,
`/opt/webapp/shared/vault.enc`, `/root/.ash_history`, `/usr/local/bin/verify`,
`/.dockerenv`.
 
`/etc/profile` and `/etc/phantom-banner` were modified after setup. Read both — profile
was stock Alpine, and phantom-banner was just the level banner displayed on login.
 
**Conclusion:** No new credential files found this way, but useful for confirming the
full scope of what was planted.
 
---
 
### Step 7 — Keyword search in config directories
 
When file names don't give it away, search file contents:
 
```sh
grep -r 'password\|secret\|key\|token\|pass' /opt/ 2>/dev/null
```
 
**What this command does:**
- `grep -r` — recursive grep, searches all files under the given directory
- `'password\|secret\|key\|token\|pass'` — the `\|` is grep's OR operator in basic
  regex. This matches any line containing any of these words.
- `2>/dev/null` — suppress errors for unreadable files
Result: only the database password we already had.
 
```sh
grep -r 'password\|secret\|key\|token\|pass' /etc/ 2>/dev/null | grep -v Binary
```
 
Same search in `/etc/`. The extra `| grep -v Binary` filters out lines where grep
detected a binary file and printed "Binary file X matches" — those are not readable
credentials. Result: only standard system config files, nothing planted.
 
**Conclusion:** No new credentials found by content search.
 
---
 
### Step 8 — Searching for key files by extension
 
```sh
find / -not -path '/proc/*' -not -path '/sys/*' \( -name '*.key' -o -name '*.pem' -o -name '*.env' \) 2>/dev/null
```
 
**What this command does:**
- `\( ... \)` — groups the `-name` conditions together (the backslashes escape the
  parentheses so ash treats them as find syntax, not shell grouping)
- `-name '*.key' -o -name '*.pem' -o -name '*.env'` — the `-o` is find's OR operator.
  Match files ending in `.key`, `.pem`, or `.env`. These are the most common extensions
  for key files and environment variable files.
Result: only `/etc/ssl/cert.pem` and `/etc/ssl1.1/cert.pem` — standard CA certificate
bundles that come with Alpine. Nothing planted.
 
**Conclusion:** No key files on disk. The decryption key is not stored as a separate
file.
 
---
 
### Step 9 — Environment variables and cron
 
```sh
env
```
 
No credentials in the current environment — only standard shell variables (PATH, HOME,
TERM, etc.).
 
```sh
cat /etc/environment
cat /etc/crontabs/root
```
 
`/etc/environment` is the PAM environment file — empty except for comments. Crontab
contained only standard Alpine periodic maintenance jobs, nothing planted.
 
**Conclusion:** No runtime credentials found.
 
---
 
### Step 10 — Decrypting the vault
 
After exhausting all other paths, the vault remained as the only unread thing on the
box. The `Salted__` prefix visible in the raw binary output is the OpenSSL salted
encryption header — a signature of files created with `openssl enc`.
 
The structure of the challenge clicked here: the database password in `database.yml`
was stored carelessly. The vault is next to it. The vault is encrypted. The most
"convenient" place to store the vault's passphrase — which is what the challenge is
about, convenience over safety — would be right there in the same config directory.
 
Try the database password as the vault's decryption passphrase:
 
```sh
openssl enc -d -aes-256-cbc -pbkdf2 -in /opt/webapp/shared/vault.enc -pass pass:506ca8261d9d23c860467e1d48753a7a 2>/dev/null
```
 
**What this command does:**
- `openssl enc` — the OpenSSL symmetric encryption/decryption tool
- `-d` — decrypt (without this flag, openssl enc encrypts)
- `-aes-256-cbc` — the cipher. AES-256 in CBC mode is the most common choice for
  `openssl enc`. If this had failed, the next tries would be `-aes-128-cbc` and
  `-aes-256-gcm`.
- `-pbkdf2` — the key derivation function. OpenSSL uses PBKDF2 by default in newer
  versions when this flag is present. The `Salted__` header in the file confirms a salt
  was used, which is consistent with PBKDF2. Without this flag, older versions of
  OpenSSL use a weaker derivation and the decryption may produce garbage even with the
  correct passphrase.
- `-in /opt/webapp/shared/vault.enc` — the input file to decrypt
- `-pass pass:506ca8261d9d23c860467e1d48753a7a` — the passphrase, in `pass:VALUE`
  format. OpenSSL supports several passphrase input methods (`pass:`, `file:`, `env:`,
  `stdin`). Using `pass:` provides it directly on the command line.
- `2>/dev/null` — if the passphrase is wrong, OpenSSL prints a "bad decrypt" error to
  stderr. Suppressing stderr means a wrong passphrase produces no output, and a correct
  one produces only the plaintext.
Result:
```
b1a707a245309673888bb4d6923367af
```
 
The vault contained a credential. Write it to the proof file:
 
```sh
printf 'b1a707a245309673888bb4d6923367af' > /tmp/.ca-files-proof && verify
```
 
`printf` rather than `echo` to avoid a trailing newline, which can cause exact-match
graders to fail.
 
**Level solved.**
 
---
 
## What the vulnerability was
 
Two separate mistakes compounded each other:
 
1. The database password (`506ca8261d9d23c860467e1d48753a7a`) was stored in plaintext
   in a config file readable by any process running on the box.
2. The vault (`vault.enc`) was encrypted — which looks like good security — but the
   decryption key was stored in the same directory. Encrypting a file and storing the
   key next to it gives you the appearance of security with none of the actual
   protection. Anyone who can read the directory can decrypt the vault.
The challenge title captures this precisely: "someone stored a key where it was easy,
not where it was safe."
 
---
 
## Dead ends
 
| What we tried | Why we tried it | Why it failed |
|---|---|---|
| `notthisone` (Redis password) | It was a credential on the box | Named to tell you it's wrong; grader rejected it |
| `506ca8261d9d23c860467e1d48753a7a` (database password) | Most plausible-looking credential | It was the vault key, not the answer itself |
| `webapp:506ca8261d9d23c860467e1d48753a7a` (user:password format) | Grader might want a compound value | Grader wants just the value, not the username |
| `deadbeefdeadbeefdeadbeefdeadbeef` (LEGACY_TOKEN) | Found in a dotfile | Filename said `.profile.d-decoy`; value is a placeholder |
| SSH keys, `.env` files, key files by extension | Standard credential locations | None existed on this box |
| `/proc` environment of running processes | Challenge references `proc(5)` | No app process was running; only `sleep infinity` as PID 1 |
 
---
 
## Carryover findings
 
The database password `506ca8261d9d23c860467e1d48753a7a` for user `webapp` is very
likely the input for L08. The L08 description says: *"You can read the password
database now, but reading a hash is not knowing the password. Take one offline and turn
it back into the plaintext that opens another account."* That is a hash-cracking
challenge, and this MD5-looking hash is the one we have.
 
---
 
## Command reference
 
| Command | What it does |
|---|---|
| `cat /root/.ash_history` | Read ash shell history (Alpine uses ash, not bash) |
| `cat /etc/passwd` | List all users and their shells — look for real shells like `/bin/sh` |
| `ls -la /home/operator/` | List home directory including dotfiles — `-a` is required |
| `ls /opt` | Check for deployed applications |
| `find / -not -path '/proc/*' -not -path '/sys/*' -name '.*' -type f 2>/dev/null` | Find all dotfiles on the filesystem, excluding noisy virtual filesystems |
| `find / -not -path '/proc/*' -not -path '/sys/*' -newer <reference-file> -type f 2>/dev/null` | Find files modified after a reference date — isolates what was planted |
| `find / -not -path '/proc/*' -not -path '/sys/*' \( -name '*.key' -o -name '*.pem' -o -name '*.env' \) 2>/dev/null` | Search for key and env files by extension |
| `grep -r 'password\|secret\|key\|token\|pass' /opt/ 2>/dev/null` | Search file contents for credential keywords |
| `grep -r '...' /etc/ 2>/dev/null \| grep -v Binary` | Same search in /etc/, filtering out binary file match notices |
| `env` | Show current process environment variables |
| `cat /etc/environment` | Show global PAM environment file |
| `cat /etc/crontabs/root` | Show root's crontab — sometimes contains inline credentials |
| `cat /etc/shadow` | Show password hashes — `*` is locked, `!` is no password |
| `openssl enc -d -aes-256-cbc -pbkdf2 -in vault.enc -pass pass:PASSPHRASE 2>/dev/null` | Decrypt an OpenSSL-encrypted file; suppress bad-decrypt errors with 2>/dev/null |
| `printf 'VALUE' > /tmp/.ca-files-proof` | Write proof without trailing newline — safer than echo for exact-match graders |