# L08 — ca-shadow · A Hash Is Not a Password

**Act II · Credential Access · T1110.002**
**Status:** Solved
**Flag:** `bl_92dd00cedf7b10028aa41920d240271f`

---

## Challenge brief

> You can read the password database now, but reading a hash is not knowing the password.
> Take one offline and turn it back into the plaintext that opens another account.

MITRE tag T1110.002 is "Brute Force: Password Cracking" — taking a stored password hash
offline and recovering the original plaintext through repeated guessing. The challenge
title "A Hash Is Not a Password" is the core lesson: a hash looks like a credential but
is not usable by itself. You need the plaintext behind it.

You land as root. The job is to find the hash, crack it offline on your local machine,
and use the plaintext to access an encrypted file on the box.

---

## Background: what a password hash is and why cracking works

When you set a password on a Linux system, the system does not store the password itself.
It runs the password through a **hash function** — a one-way mathematical operation that
produces a fixed-length output called a hash. The hash is what gets stored in
`/etc/shadow`.

When you log in, the system hashes whatever you typed and compares the result to the
stored hash. If they match, you're in. The original password is never stored anywhere.

This means:
- An attacker who reads `/etc/shadow` gets hashes, not passwords.
- A hash cannot be mathematically reversed — there is no "decrypt" operation.
- But the attacker can **guess**: take a candidate password, hash it the same way, and
  compare. If the hashes match, the guess was correct. Repeat millions of times.

This is offline cracking. It happens on the attacker's machine, not the target — so
there are no login attempts, no lockouts, and no entries in the target's logs.

---

## Understanding password hash formats

Before you can crack a hash, you need to know what algorithm produced it. The hash
format tells you this. Here are the formats you will encounter on Linux systems, from
most to least common.

### The general structure of a Linux shadow hash

Every hash in `/etc/shadow` follows the same pattern:

```
$id$salt$hash
```

- `$id$` — a short identifier that tells you the algorithm
- `salt` — a random string mixed into the hash to prevent precomputed attacks (explained
  below)
- `hash` — the actual output of the hash function

### Hash format reference

| Prefix | Algorithm | Speed to crack | Notes |
|---|---|---|---|
| `$1$` | MD5crypt | Very fast — millions per second on GPU | Old, considered broken. Still found on legacy systems. |
| `$5$` | SHA-256crypt | Fast | Less common. Found on some embedded and older Linux systems. |
| `$6$` | SHA-512crypt | Slow — tens of thousands per second on GPU | The default on Ubuntu 18.04–22.04, Alpine Linux, CentOS 7. What we cracked here. |
| `$2y$` or `$2b$` | bcrypt | Very slow — hundreds per second on GPU | The default on some newer configs. Designed to be slow. |
| `$y$` | yescrypt | Slowest — tens to hundreds per second on GPU | Default on Ubuntu 24.04. Purpose-built to resist GPU cracking. |

**How to identify which one you have:** just read the prefix at the start of the hash
field. In L08 the hash was:

```
$6$U9/m7KIgPLWc9qzJ$Wo6YPkvE6lNKN8c1y9rmcDCQCJa4yLzeIxxoy2O6s6SnvZxRdze766QTgWbDdECvUg/sggGMBsykTo3ys8IdM/
```

Breaking it down:
- `$6$` — SHA-512crypt
- `U9/m7KIgPLWc9qzJ` — the salt (random, generated when the password was set)
- `Wo6YPkv...s8IdM/` — the actual hash output

### Why does the salt matter?

Without a salt, two users with the same password would have identical hashes. An
attacker could precompute a massive table of `password → hash` pairs (called a rainbow
table) and look up any hash instantly. The salt makes this impossible: even if two users
have the same password, their hashes are different because they have different salts. The
attacker must compute the full hash function for every candidate password against every
individual salt. This is why cracking is slow — and why modern hash algorithms also
deliberately do many rounds of hashing (SHA-512crypt does 5000 rounds by default).

### Non-shadow hashes you will encounter

Beyond `/etc/shadow`, you will encounter hashes stored in other formats:

| Format | Prefix / length | Common context |
|---|---|---|
| MD5 (plain) | 32 hex characters, no `$` prefix | Database passwords, old web apps, checksums |
| SHA-1 | 40 hex characters | Git commit IDs, old web tokens |
| SHA-256 | 64 hex characters | Modern checksums, some API tokens |
| SHA-512 | 128 hex characters | Some database storage |
| NTLM | 32 hex characters | Windows password hashes (different from MD5crypt) |
| bcrypt | `$2y$` or `$2b$` prefix | Web application password storage (PHP, Node, etc.) |

In L07 we found `506ca8261d9d23c860467e1d48753a7a` in `database.yml`. That is 32 hex
characters with no prefix — plain MD5. It was not the hash of a password in that
challenge (it was being used as an encryption key), but recognising the format tells you
immediately: if it were a password hash, it would crack in under a second with hashcat
in mode `-m 0`.

The key habit: **identify the format before picking your tool and mode**. Wrong mode =
wrong algorithm = no match even if you have the right password.

---

## hashcat — how it works and how to use it

### What hashcat is

hashcat is a password recovery tool that runs on your local machine. It takes a hash
file, a wordlist or mask, and a mode number (which tells it the algorithm), then
systematically hashes every candidate and compares the result to your target hash. When
they match, it prints the plaintext.

It uses the GPU rather than the CPU for most hash types, because GPUs can run millions
of parallel computations simultaneously — exactly what hash cracking needs. For
SHA-512crypt on an Apple M4 GPU we got about 17,000–21,000 hashes per second. For plain
MD5 on the same machine it ran at about 29 million per second — the difference reflects
how deliberately slow SHA-512crypt is.

### Installing on Mac with Homebrew

```sh
brew install hashcat
```

### Saving the hash correctly

Always use single quotes when echoing a hash to a file. The hash contains `$` signs,
and without single quotes the shell tries to interpret `$6`, `$U9`, etc. as variable
names, corrupting the hash:

```sh
# correct — single quotes prevent shell interpretation
echo '$6$U9/m7KIgPLWc9qzJ$Wo6...' > ~/hash.txt

# wrong — double quotes allow the shell to expand $6, $U9 etc.
echo "$6$U9/m7KIgPLWc9qzJ$Wo6..." > ~/hash.txt
```

Verify the file looks right before cracking:

```sh
cat ~/hash.txt
```

The `$` signs must all be present and intact.

### Dictionary attack — the standard approach

A dictionary attack hashes every word in a wordlist and compares against your target.
rockyou.txt is the standard wordlist — 14.3 million real passwords leaked from the 2009
RockYou data breach. Because people reuse passwords, it covers a large portion of weak
credentials in use today.

```sh
hashcat -m 1800 -a 0 ~/hash.txt ~/rockyou.txt
```

Flags:
- `-m 1800` — the mode for SHA-512crypt (`$6$`). The mode number maps directly to the
  hash type. Always identify your hash type first, then look up the mode.
- `-a 0` — attack mode 0, which is dictionary (wordlist) attack. This is the default
  and most common attack.
- `~/hash.txt` — your hash file
- `~/rockyou.txt` — the wordlist

### The `-O` flag (optimized kernels)

```sh
hashcat -m 1800 -a 0 ~/hash.txt ~/rockyou.txt -O
```

`-O` switches hashcat from "pure kernels" to "optimized kernels." The difference:
- Pure kernels support passwords up to 256 characters — full coverage, but slower.
- Optimized kernels support passwords up to 15 characters — faster, but a password
  longer than 15 characters will be missed (shown in the status as "Rejected").

For CTF hashes and most real-world weak passwords, optimized is the right choice — the
vast majority of crackable passwords are under 15 characters. Always try `-O` first.
In this challenge the password was `dewdrop` (7 characters) — optimized kernels cracked
it in 3 seconds vs. the ~12 minutes pure kernels would have taken.

### Rules attack — for passwords with transformations

If the plain wordlist fails, the next step is rules. A rules file contains
transformations to apply to each word in the wordlist: capitalize the first letter,
append a number, substitute `a` with `@`, etc. A single word `password` becomes
`Password`, `password1`, `p@ssword`, `PASSWORD`, and many more variants.

```sh
hashcat -m 1800 -a 0 ~/hash.txt ~/rockyou.txt -r ~/best64.rule
```

This multiplies the effective wordlist size by the number of rules (best64 applies 64
transformations per word — 14 million × 64 = ~900 million candidates). It takes much
longer but covers passwords that a plain dictionary misses.

### Checkpoint and restore

If you need to stop hashcat mid-run, press `c` (checkpoint) rather than `q` (quit).
Checkpoint saves progress at the current position and exits cleanly. You can resume from
the exact same point with:

```sh
hashcat --restore
```

Note: `--restore` takes no other arguments — it reads the saved session file on its own.

### Checking the potfile

hashcat caches every cracked hash in a potfile (`~/.hashcat/hashcat.potfile`). If you
run hashcat against a hash that was already cracked in a previous session, it will report
"All hashes found in potfile / Cracked" without running at all. Check the potfile with:

```sh
hashcat -m 1800 ~/hash.txt --show
```

This is useful when you think a hash was cracked but the result wasn't printed clearly.

### hashcat mode reference (most common for Linux work)

| Hash type | Mode | When you see it |
|---|---|---|
| Plain MD5 | `-m 0` | Config files, old databases |
| SHA-1 | `-m 100` | Old web app storage |
| SHA-256 (plain) | `-m 1400` | Checksums, some APIs |
| MD5crypt `$1$` | `-m 500` | Legacy Linux shadow |
| SHA-256crypt `$5$` | `-m 7400` | Some Linux shadow |
| SHA-512crypt `$6$` | `-m 1800` | Linux shadow (Ubuntu 18–22, Alpine) |
| bcrypt `$2y$` | `-m 3200` | Web app password storage |
| NTLM | `-m 1000` | Windows environments |

---

## Initial hypothesis

Root shell, credential access challenge. The challenge title says "password database" —
that points directly to `/etc/shadow`. One user will have a real hash. Crack it offline,
then the plaintext unlocks something on the box.

---

## Enumeration log

### Step 1 — Read /etc/shadow

```sh
cat /etc/shadow
```

Result: one user with a real hash — `svc-deploy`:

```
svc-deploy:$6$U9/m7KIgPLWc9qzJ$Wo6YPkvE6lNKN8c1y9rmcDCQCJa4yLzeIxxoy2O6s6SnvZxRdze766QTgWbDdECvUg/sggGMBsykTo3ys8IdM/:20750:0:99999:7:::
```

All other accounts have `*` (locked) or `!` (no password). `svc-deploy` is the target.

Hash format: `$6$` — SHA-512crypt. Hashcat mode: `-m 1800`.

---

### Step 2 — Look around the box before cracking

```sh
cat /etc/passwd | grep svc-deploy
find /opt -type f 2>/dev/null
ls /home/svc-deploy
```

Found:
- `svc-deploy` has shell `/sbin/nologin` (cannot log in interactively via SSH, but `su`
  works)
- `/opt/deploy/backup.enc` — another OpenSSL-encrypted file, same structure as L07
- `/home/svc-deploy/` — empty

**Conclusion:** The pattern is the same as L07. Crack the hash to get the passphrase,
use the passphrase to decrypt the vault.

---

### Step 3 — Save the hash on the local Mac

```sh
echo '$6$U9/m7KIgPLWc9qzJ$Wo6YPkvE6lNKN8c1y9rmcDCQCJa4yLzeIxxoy2O6s6SnvZxRdze766QTgWbDdECvUg/sggGMBsykTo3ys8IdM/' > ~/hash.txt
cat ~/hash.txt   # verify the $ signs are intact
```

---

### Step 4 — Download rockyou.txt

```sh
curl -L https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt -o ~/rockyou.txt
```

133MB file, 14.3 million passwords. Verify it downloaded correctly:

```sh
file ~/rockyou.txt    # should say: Unicode text, UTF-8 text
head -5 ~/rockyou.txt # should show: 123456, 12345, 123456789, password, iloveyou
```

---

### Step 5 — Run hashcat with optimized kernels

```sh
hashcat -m 1800 -a 0 ~/hash.txt ~/rockyou.txt -O
```

Result after 3 seconds:

```
$6$U9/m7KIgPLWc9qzJ$Wo6...:dewdrop

Status: Cracked
Progress: 53272/14344384 (0.37%)
```

Password is `dewdrop`. It was only 0.37% into rockyou.txt — very early in the list.
The `-O` flag made the difference: optimized kernels ran at ~21,000 H/s vs. ~14,000 H/s
for pure kernels, and more importantly the kernel startup is much faster.

---

### Step 6 — Write proof and verify (first attempt)

```sh
printf 'dewdrop' > /tmp/.ca-files-proof && verify
```

Result:
```
Not solved yet. no proof on the box yet
Write the value you recover to /tmp/.ca-shadow-proof, then run verify again.
```

Wrong proof filename. The grader uses `/tmp/.ca-shadow-proof` for this level (shadow
credential, not files). The proof file name pattern follows the challenge tag: `ca` for
Credential Access, `shadow` for the specific source.

```sh
printf 'dewdrop' > /tmp/.ca-shadow-proof && verify
```

Result:
```
Not solved yet. the proof does not match this box's secret
```

The grader doesn't want the plaintext password — it wants the decrypted vault contents,
same as L07.

---

### Step 7 — Use the password to access the account and the vault

The `svc-deploy` user can't read `backup.enc` directly (it's owned by root, group root,
`r-----`). But we can switch to the account to explore it:

```sh
su -s /bin/sh svc-deploy
# enter: dewdrop
```

```sh
ls -la /home/svc-deploy/
ls -la /opt/deploy/backup.enc
```

`backup.enc` shows `rw-r-----` — group readable only for root group. `svc-deploy` is
not in the root group, so even as `svc-deploy` we can't read it.

Exit back to root:

```sh
exit
```

---

### Step 8 — Decrypt the vault as root

As root, decrypt `backup.enc` using `dewdrop` as the passphrase:

```sh
openssl enc -d -aes-256-cbc -pbkdf2 -in /opt/deploy/backup.enc -pass pass:dewdrop 2>/dev/null
```

Result: `4a0c24dd71c3e1139c5b580cca0c78bc`

Write to proof file and verify:

```sh
printf '4a0c24dd71c3e1139c5b580cca0c78bc' > /tmp/.ca-shadow-proof && verify
```

**Level solved.**

---

## What the vulnerability was

The `svc-deploy` user's password was weak — `dewdrop` is a common English word that
appears early in the rockyou.txt wordlist (0.37% in). SHA-512crypt is a strong hashing
algorithm when paired with a strong password. When the password is weak, the algorithm's
strength doesn't matter — every word in the dictionary gets tried automatically.

The chain of mistakes: weak password → hash stored in `/etc/shadow` (root-readable after
escalation) → cracked offline → used as the vault key → vault decrypted. Each step
followed from the previous one.

---

## Dead ends and wrong turns

### Cracking the wrong hash

The most significant mistake in this session: the first hash we cracked came from the
L07 box, not the L08 box. When the L07 SSH session timed out and was reconnected, a new
L08 box was provisioned — with a different hash. We spent about 30 minutes cracking
`$6$v9xUCKRUKaHdYE9H$RDH...` (from the old box) rather than
`$6$U9/m7KIgPLWc9qzJ$Wo6...` (the correct L08 hash).

**The lesson:** always read the hash directly from the box you are currently connected
to, immediately before cracking. Each BreachLab box is unique per session. If you
reconnect, the hash changes. Verify the hash in `~/hash.txt` matches what's currently
in `/etc/shadow` before starting a crack run.

### Writing the password as the proof

The grader for this level wanted the *decrypted vault contents*, not the cracked
password itself. The pattern is consistent with L07: the cracked or found credential is
a key that unlocks something, and the thing it unlocks is the actual answer.

### Writing to the wrong proof file

The grader told us the correct filename on the first failed `verify` call:
`/tmp/.ca-shadow-proof`. Always run `verify` with whatever guess you have first — the
error message tells you the expected filename if you got it wrong.

### The rules attack (abandoned)

We started a rules attack (`-r ~/best64.rule`) that estimated 12 hours to complete.
This was the wrong next step — for a CTF, if rockyou.txt plain dictionary doesn't work,
the first fix is to check whether you have the right hash, not to escalate to a longer
attack. We were cracking the wrong hash the whole time.

---

## Carryover findings

None. This level's chain was self-contained: shadow hash → crack → vault decrypt →
flag.

---

## Command reference

| Command | What it does |
|---|---|
| `cat /etc/shadow` | Read all password hashes — look for accounts with a `$` hash instead of `*` or `!` |
| `echo '$6$salt$hash' > ~/hash.txt` | Save hash to file — single quotes required to prevent shell `$` expansion |
| `cat ~/hash.txt` | Verify the hash saved correctly before cracking |
| `brew install hashcat` | Install hashcat on Mac via Homebrew |
| `curl -L <url> -o ~/rockyou.txt` | Download rockyou.txt wordlist |
| `hashcat -m 1800 -a 0 ~/hash.txt ~/rockyou.txt -O` | Crack a `$6$` SHA-512crypt hash with dictionary attack, optimized kernels |
| `hashcat -m 0 -a 0 ~/hash.txt ~/rockyou.txt` | Crack a plain MD5 hash |
| `hashcat -m 1800 ~/hash.txt --show` | Show previously cracked results from the potfile |
| `hashcat --restore` | Resume a run from a saved checkpoint (no other arguments) |
| `su -s /bin/sh <user>` | Switch to a user that has `/sbin/nologin` as their shell |
| `openssl enc -d -aes-256-cbc -pbkdf2 -in <file> -pass pass:<password> 2>/dev/null` | Decrypt an OpenSSL-encrypted file |
| `printf '<value>' > /tmp/.ca-shadow-proof && verify` | Write proof and verify — printf avoids trailing newline |