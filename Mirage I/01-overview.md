# Mirage 1 Track — Concept Prep Notes

Web security track in BreachLab. 41 levels, built as a ladder: starts with what any
visitor can see, climbs through access control and authentication, then injection, the
XSS family, the heavier server-side bugs, and finishes on AI-feature bugs.

These notes map each level name to the concept it points to and what to study. Level
placements for a few ambiguous names (#13, #11, #35) are "most likely" based on where
they sit in the ladder, not certain.

---

# Cluster 1 — What the site gives away (Levels 1–6)

Reconnaissance, and the idea that the browser is not a safe place to keep secrets.

- **Nothing Is Secret** — Information disclosure. Everything shipped to the browser is
  readable: HTML comments, `robots.txt`, `sitemap.xml`, error messages that leak stack
  traces, metadata, verbose headers. Learn what a site exposes to an unauthenticated
  visitor.
- **The Wire** — HTTP itself. Every request and response between browser and server can
  be read and changed with an intercepting proxy (Burp Suite, mitmproxy). Learn HTTP
  methods, headers, status codes, and how to intercept and replay traffic.
- **Peel the Encoding** — Encoding is not encryption. Base64, URL-encoding, hex, and the
  body of a JWT are all reversible with no key. Learn to recognize and decode layered
  encodings, and why "it looks scrambled" means nothing for security.
- **Source Maps Talk** — JavaScript source maps (`.js.map` files). When left on a server,
  they rebuild the original, un-minified source — comments, file structure, sometimes
  secrets. Learn how front-end build output can leak the original code.
- **Left in the Open** — Exposed files and directories. Backup files (`.bak`, `.old`), an
  exposed `.git` folder, `.env` files, directory listing turned on. Learn to look for
  files that were never meant to be reachable.
- **Trust the Client** — Client-side controls. Hidden form fields, disabled buttons, price
  set in JavaScript, "admin=false" in a cookie. Anything the browser enforces, the
  attacker can change. The core lesson of the whole cluster: never trust input from the
  client.

# Cluster 2 — Access control (Levels 7–12)

The server authenticates you correctly but fails to check whether *you* are allowed to
touch *this* thing.

- **Other People's Objects** — IDOR (Insecure Direct Object Reference). Change
  `/invoice/1001` to `/invoice/1002` and see someone else's invoice. The server checks you
  are logged in but not that the object is yours. Learn object-level authorization.
- **Break the Logic** — Business logic flaws. The app does exactly what it was coded to do,
  but the flow is wrong: applying a coupon twice, negative quantities, skipping a payment
  step. Learn to think about intended flow vs. what the app actually allows.
- **Keys in the Bundle** — Hardcoded secrets. API keys, tokens, or credentials baked into
  the JavaScript bundle shipped to every visitor. Learn how secrets end up in front-end
  code and what an attacker does with them.
- **Over-Posting** — Mass assignment. A form sends `name` and `email`, but the server
  blindly binds every field in the request to the database object — so you add `role=admin`
  and it sticks. Learn how frameworks bind request data to models and why allow-lists
  matter.
- **Row-Level Lies** — Row-level / tenant isolation failure. In a multi-tenant app, your
  account should only see its own rows, but a missing filter lets you read or write another
  tenant's data. Learn how data scoping works at the query/database level and how it breaks.
- **Open Rules** — Overly permissive authorization rules. Rules that default to "allow"
  instead of "deny," or wildcard permissions broader than intended (common in cloud storage
  rules and API policies). Learn the default-deny principle and how to read an access policy.

# Cluster 3 — Authentication and session (Levels 13–16)

Attacks on the identity layer itself.

- **Front the Door** — The login mechanism. Credential stuffing, brute force, username
  enumeration, weak lockout, authentication bypass at the entry point. Learn how login
  endpoints are attacked and hardened.
- **Forge the Token** — JWT manipulation. Changing the `alg` to `none`, cracking a weak
  signing secret, or swapping claims because the server does not verify the signature
  properly. Learn JWT structure (header, payload, signature) and the common signing
  mistakes.
- **Reset Without Proof** — Password reset flaws. Predictable or guessable reset tokens,
  tokens that never expire, reset links that leak via the `Referer` header, or a flow that
  lets you reset someone else's password without proving you own the account. Learn how a
  secure reset flow is supposed to work so you can spot the missing step.
- **Unverified Identity** — Trusting an identity claim without checking it.
  Email-verification bypass, trusting a client-supplied user ID, or OAuth/SSO flows where
  the token's audience or issuer is not validated. Learn how an app confirms you are who
  you say you are, and where that check gets skipped.

# Cluster 4 — Injection (Levels 17–23)

Untrusted input reaches a place that treats it as code or commands.

- **Walk the Path** — Path traversal. Using `../../` to escape a directory and read files
  like `/etc/passwd` or application config. Learn how file paths are built from user input.
- **Union of Lies** — UNION-based SQL injection. Appending a `UNION SELECT` to pull data
  from other tables into the page. Learn SQL injection fundamentals and how UNION reads
  extra data.
- **Blind Inference** — Blind SQL injection. No data comes back directly, so you ask
  true/false questions and infer answers from the response or from time delays
  (boolean-based and time-based). Learn how to extract data one bit at a time with no
  visible output.
- **Operators Welcome** — NoSQL injection. Injecting query operators like `$ne`, `$gt`, or
  `$regex` into a MongoDB-style query to bypass a check or dump data. Learn how NoSQL
  queries differ from SQL and why operator injection works.
- **Shell on Demand** — OS command injection. User input flows into a system command, so
  `; cat /etc/passwd` or `$(...)` runs on the server. Direct remote code execution. Learn
  how apps shell out and how to spot unsafe command building.
- **Template Injection** — Server-Side Template Injection (SSTI). Input lands inside a
  server template engine (Jinja2, Twig, Handlebars) and gets evaluated, often leading to
  code execution. Learn the common engines and their test payloads.
- **Other Grammars** — Injection into other parsers. The same "input becomes code" idea
  applied to LDAP, XPath, expression languages, or similar grammars beyond SQL. Learn to
  recognize any place where input is parsed as a query language.

# Cluster 5 — The XSS family (Levels 24–27)

Getting your JavaScript to run in another user's browser.

- **Reflected** — Reflected XSS. Your payload is in the request (a URL parameter) and echoed
  straight back into the page. Needs the victim to click a crafted link. Learn how
  unescaped output in a response causes script execution.
- **Stored** — Stored XSS. The payload is saved (a comment, profile field) and runs for
  everyone who views it. More dangerous because it needs no special link. Learn the
  difference in impact and delivery from reflected.
- **DOM Sink** — DOM-based XSS. No server involved — front-end JavaScript takes
  attacker-controlled data and writes it into a dangerous "sink" like `innerHTML` or `eval`.
  Learn sources and sinks in client-side JavaScript.
- **Past the CSP** — Content Security Policy bypass. CSP is a header meant to stop XSS; this
  level is about the weak policies and gadgets that let script run anyway. Learn how CSP
  works and where real-world policies fall short. Study after the three XSS types above,
  since it builds on them.

# Cluster 6 — Server-side and modern app bugs (Levels 28–35)

The heavier, higher-impact classes.

- **Server-Side Request** — SSRF (Server-Side Request Forgery). You make the server fetch a
  URL you choose, reaching internal services it can access but you cannot. Learn how SSRF
  reaches internal networks (and connects to cloud metadata).
- **Cross-Origin Trust** — CORS misconfiguration. Response headers that trust the wrong
  origins, letting a malicious site read authenticated responses. Learn the same-origin
  policy first, then how CORS relaxes it and how that gets misconfigured.
- **External Entities** — XXE (XML External Entity). An XML parser that processes external
  entities can be made to read local files or make requests. Learn XML parsing and why
  external entities are dangerous.
- **Polyglot Upload** — File upload flaws. Bypassing type/extension checks, uploading a file
  valid as two formats at once (a polyglot) to slip past a filter, or getting an uploaded
  file executed. Learn how upload validation is done and bypassed.
- **Corrupt the Object Graph** — Insecure deserialization. The app rebuilds objects from
  attacker-controlled serialized data, which can trigger code execution or object injection.
  Learn how serialization works in your language and why untrusted input must never be
  deserialized.
- **Schema Is the Map** — GraphQL attacks. Using introspection to reveal the full schema,
  then abusing queries it exposes — over-fetching, missing authorization per field, batching
  abuse. Learn GraphQL basics and introspection.
- **Skip the Middleware** — Middleware/authorization bypass. Reaching a protected route
  without the auth middleware running — for example the Next.js middleware bypass
  (CVE-2025-29927) from early 2025. Learn how middleware-based access control works and how
  a crafted request can route around it.
- **Talk Past the Guard** — Getting a request past a protective layer. The HTTP request
  smuggling / proxy desync and WAF-bypass family: the front proxy and back-end server
  disagree about where one request ends, so a hidden request slips through. Learn how
  front-end and back-end servers parse requests differently.

# Cluster 7 — AI-feature bugs (Levels 36–41)

The newer class the track description mentions. These assume the app has an LLM feature
(chatbot, assistant, RAG search, agent).

- **Second-Order Prompt** — Stored (second-order) prompt injection. Malicious instructions
  are saved somewhere (a document, a profile, a support ticket) and later fed to the model,
  which follows them. The parallel to Stored XSS is deliberate. Learn what prompt injection
  is and how the stored variant is delivered.
- **The Model Is the Vector** — Direct prompt injection and jailbreaking. The model's own
  input is the attack surface — you talk it into ignoring its instructions or revealing its
  system prompt. Learn the direct-injection basics first, since the others build on it.
- **Borrowed Authority** — Excessive agency / confused deputy. The AI feature has real
  permissions (it can call tools, read data, send requests), and you get it to use that
  authority on your behalf — doing something you could not do directly. Learn how
  tool-using agents are given permissions and why that is dangerous.
- **Retrieve by Meaning** — RAG and vector-search attacks. Apps that retrieve documents "by
  meaning" (embeddings/vector databases) can be poisoned by planting content that gets
  retrieved and then trusted by the model. Learn how RAG pipelines work and where untrusted
  content enters them.
- **Past the Gateway** — Bypassing an AI gateway or guardrail. Many apps put a filter in
  front of the model to block bad input/output; this is about getting past it (encoding
  tricks, splitting payloads, indirect phrasing). Parallel to "Past the CSP" and "Talk Past
  the Guard."
- **Full Engagement** — The capstone. A full chained exploit combining several earlier bugs
  end to end. No new concept — it tests whether you can connect the ladder together.

---

# Summary table

| # | Name | Core concept | Study area |
|---|------|--------------|------------|
| 1 | Nothing Is Secret | Information disclosure | Recon, exposed metadata/errors |
| 2 | The Wire | HTTP + intercepting proxies | HTTP fundamentals, Burp |
| 3 | Peel the Encoding | Encoding ≠ encryption | Base64/URL/hex, JWT body |
| 4 | Source Maps Talk | JS source map leakage | Front-end build output |
| 5 | Left in the Open | Exposed files/dirs | `.git`, `.env`, backups, listing |
| 6 | Trust the Client | Client-side control bypass | Hidden fields, client validation |
| 7 | Other People's Objects | IDOR | Object-level authorization |
| 8 | Break the Logic | Business logic flaws | Intended flow vs. allowed flow |
| 9 | Keys in the Bundle | Hardcoded secrets | Secrets in front-end code |
| 10 | Over-Posting | Mass assignment | Model binding, allow-lists |
| 11 | Row-Level Lies | Tenant/row isolation failure | Data scoping at query level |
| 12 | Open Rules | Permissive authz rules | Default-deny, policy review |
| 13 | Front the Door | Login attacks / auth bypass | Credential attacks, lockout |
| 14 | Forge the Token | JWT manipulation | JWT structure, signing flaws |
| 15 | Reset Without Proof | Password reset flaws | Secure reset flow |
| 16 | Unverified Identity | Trusting unverified claims | OAuth/SSO, email verification |
| 17 | Walk the Path | Path traversal | File path handling |
| 18 | Union of Lies | UNION SQL injection | SQLi fundamentals |
| 19 | Blind Inference | Blind SQL injection | Boolean/time-based extraction |
| 20 | Operators Welcome | NoSQL injection | Query operator injection |
| 21 | Shell on Demand | OS command injection | Unsafe command building |
| 22 | Template Injection | SSTI | Template engines |
| 23 | Other Grammars | LDAP/XPath/EL injection | Other query grammars |
| 24 | Reflected | Reflected XSS | Unescaped output |
| 25 | Stored | Stored XSS | Persisted payloads |
| 26 | DOM Sink | DOM XSS | Client sources/sinks |
| 27 | Past the CSP | CSP bypass | How CSP works and fails |
| 28 | Server-Side Request | SSRF | Internal-network reach |
| 29 | Cross-Origin Trust | CORS misconfiguration | Same-origin policy, CORS |
| 30 | External Entities | XXE | XML parsing |
| 31 | Polyglot Upload | File upload bypass | Upload validation |
| 32 | Corrupt the Object Graph | Insecure deserialization | Serialization in your language |
| 33 | Schema Is the Map | GraphQL abuse | Introspection, per-field authz |
| 34 | Skip the Middleware | Middleware/authz bypass | Next.js CVE-2025-29927 pattern |
| 35 | Talk Past the Guard | Request smuggling / WAF bypass | Proxy desync, parser disagreement |
| 36 | Second-Order Prompt | Stored prompt injection | Prompt injection (stored) |
| 37 | The Model Is the Vector | Direct prompt injection | Jailbreaks, system-prompt leak |
| 38 | Borrowed Authority | Excessive agency / confused deputy | Tool-using agents, permissions |
| 39 | Retrieve by Meaning | RAG / vector poisoning | RAG pipelines, embeddings |
| 40 | Past the Gateway | AI guardrail bypass | LLM input/output filtering |
| 41 | Full Engagement | Full chained exploit | Combining earlier bugs |

---

# Learn these first (before Level 1)

Foundations that sit under the whole track:

- **HTTP in depth** — methods, headers, status codes, cookies, and how a request/response
  actually looks on the wire. Ground floor for all 41.
- **An intercepting proxy** — Burp Suite (Community is fine) or mitmproxy. Intercept, edit,
  replay. Used on every level.
- **How a web app is built** — browser (HTML/CSS/JS) vs. server vs. database, and which runs
  where. Needed to tell a client-side bug from a server-side one.
- **Encoding vs. encryption vs. hashing** — so "looks scrambled" is never confused with
  "protected."
- **Authentication vs. authorization** — authn is "who are you," authz is "are you allowed."
  Clusters 2 and 3 live on this line.
- **OWASP Top 10** — read once as a map; most of the track is a guided tour of it, with the
  AI cluster matching the OWASP Top 10 for LLM Applications.

Best companion resource: **PortSwigger Web Security Academy** — free, hands-on, covers
roughly levels 1–35 almost one-to-one. For the AI cluster (36–41), use the OWASP LLM Top 10.