# Glossary — Web Application Hacker's Handbook

Terms as Stuttard & Pinto use them. For program-workflow terms see `bug-bounty-bootcamp/glossary.md`.

## Core concepts

- **Trust boundary** — any hand-off between components/tiers that interpret data differently (app→DB, app→OS, app→browser, front-end→back-end service). WAHH's unit of validation: input must be re-validated at *each* boundary, not once at the edge.
- **Canonicalization** — reducing input to a standard form (decoding, normalization). Defense failures = checking one representation while the interpreter consumes another (decode-then-check vs check-then-decode).
- **Boundary validation** — validating data as it crosses each trust boundary, per the downstream component's syntax rules.
- **Defense mechanisms (the big three)** — authentication, session management, access control; overall security = weakest link. Plus: input handling, function handling, auditing/alerting, management interfaces.
- **Entry point / attack surface** — every location input enters: query/body/cookie/path params, all HTTP headers, uploads, out-of-band channels (mail→app), serialized objects.
- **HACK STEPS** — the book's per-technique checklists: systematic enumeration of parameters, encodings, and stages. Coverage over cleverness.
- **Forced browsing** — accessing application functions/stages out of intended sequence by requesting URLs/actions directly (skips in-browser sequencing controls).
- **Hit detection / oracle** — the signal separating success from noise in automated attacks: status, body, Location, Set-Cookie, timing, side effects. Calibrate against a baseline.

## Attack classes

- **Reflected / stored / DOM-based XSS** — payload rides request→response / persists in data later rendered / lives entirely in client JS source→sink.
- **Second-order injection** — payload stored safely (properly escaped) then concatenated into a different interpreter context later; fires where you can't watch → use timing/OOB.
- **Blind injection** — no output channel; confirm via boolean content deltas, time delays, or out-of-band (DNS/HTTP) interaction.
- **OOB (out-of-band) channel** — exfil/confirmation path outside HTTP responses: DNS lookups, HTTP requests, email to attacker-controlled listener.
- **SQL injection — UNION/blind/tautology** — append `UNION SELECT` matching column count/types; boolean `OR 1=1`/`AND 1=2` oracle; time-based `WAITFOR`/`SLEEP`/`pg_sleep`.
- **XPath/LDAP/NoSQL injection** — same principle against XML queries (`' or '1'='1`, `]|//*|[`), LDAP filters (`*)(uid=*))(|(uid=*`), Mongo-style operators (`$ne`, `$gt`, `$where`).
- **OS command injection** — shell metachars (`; | & && || \n $()` `` ` ``) inject commands; confirm blind via repeated timing.
- **Path traversal** — `../` and encoded/impedance variants (`..\\`, `..;/`, `%2e%2e%2f`, `%252e`, `....//`, `%00` suffix-strip) escape a base directory.
- **LFI/RFI** — path-controlled include of local (→ source disclosure → RCE via log/session/proc injection) or remote (→ direct code exec) files.
- **XXE/XML injection** — entity resolution (`<!ENTITY x SYSTEM …>`) and structural injection into XML/SOAP messages reaching back-end services.
- **HPI / HPP** — HTTP parameter injection: URL-decoded input smuggles `&param=` into a back-end request; parameter pollution: duplicate param names exploited via differing first/last/all handling across components.
- **SMTP/header injection** — `%0a`/`%0d%0a` in mail fields injects `Cc:`/`Bcc:` or full `MAIL FROM`/`RCPT TO`/`DATA` command sequences.
- **CSRF** — cross-site request forgery: SOP permits sending (not reading) cross-origin requests with cookies attached; needs unguessable token to defend.
- **Login CSRF** — force victim into attacker's session so victim-entered data lands in attacker account.
- **UI redress / clickjacking** — iframe overlay tricks user clicks onto a sensitive action; defenses: X-Frame-Options, `frame-ancestors`, frame-busting (all bypassable variants exist).
- **JavaScript hijacking** — cross-site `<script src>` reading JSON/array responses (CSRF for reads).
- **Session fixation** — app issues token pre-login and keeps it post-auth → plant a known token in victim's browser, then ride their session.
- **Open redirect** — user-controlled `Location` target; chain fuel for phishing, token theft, SSRF.
- **DNS rebinding** — DNS TTL/name flips turn same-origin JS into intranet reader.
- **Encryption oracle** — any function that decrypts/encrypts attacker-supplied values; submit foreign ciphertext (reveal) or chosen plaintext (encrypt) to forge tokens.
- **Race condition** — shared/static state accessed by concurrent requests → exploit window between write and read (login race, double-spend, one-time-use reuse).
- **State pollution** — session object written by feature A changes behavior of feature B (registration-overwrite pattern).
- **Enumeration oracle** — any response difference distinguishing valid/invalid usernames, IDs, paths — including timing, hidden fields, cookie sets.
- **Password spraying** — few common passwords across many accounts, evading per-account lockouts.
- **Buffer overflow (stack/heap), off-by-one** — unbounded copies; detect remotely via progressive-length probes at allocation boundaries.
- **Integer overflow / signedness** — arithmetic wrap or signed→unsigned reinterpretation producing tiny allocations or huge copies; probe `0x7fffffff`, `-1`, limit±1.
- **Format string bug** — input reaching `printf`-family as format; `%x` leaks stack, `%n` writes memory.
- **Virtual defacement / Trojan functionality** — XSS payloads injecting fake content or working credential-harvesting UI on the real domain.
- **Base-tag hijacking** — injected `<base href>` repoints relative script includes to attacker's host.

## Techniques & tooling

- **Two-account (A/B) testing** — replay high-priv requests as low-priv; the canonical access-control detector.
- **Burp Suite components** — Proxy, Repeater (request surgery), Intruder (positioned payloads), Sequencer (token randomness), Decoder, Comparer, session-handling rules/macros.
- **Tamper/evasion catalog** — URL/double-URL encoding, HTML entities (dec/hex/no-semicolon), NULL bytes, case mixing, comments-as-space, nested keywords, alternate tags/protocols, multibyte charsets, Unicode lookalikes.
- **Encryption oracle types** — "oracle reveal" (submit ciphertext → see plaintext) and "oracle encrypt" (submit plaintext → get ciphertext); combined = forge any protected value.
- **Match-count oracle** — search/query interfaces revealing hit counts → iterative inference of protected content.
- **Tamper data** — Burp-era terminology for editing requests in-flight; now "request interception."
- **ViewState / opaque state** — serialized client-held state; decode it, tamper, test whether MAC/signature is actually verified.
- **Same-origin policy** — origin = scheme+host+port; governs reads/DOM/cookies, *not* request sending — the gap CSRF lives in.
- **crossdomain.xml / plugin SOPs** — Flash/Java/Silverlight each implement their own cross-domain rules; `crossdomain.xml` `*` = any origin reads your data.
- **Trusted Sites zone** — IE trust level where XSS → ActiveX → host code execution.
- **Virtual hosting / default site** — Host-header routing on shared IPs; wrong/absent Host can surface other tenants' content.
- **Tenant discriminator** — param/cookie/subdomain separating customers in shared apps; swapping it while authenticated = cross-tenant access.
- **WebDAV methods** — `PROPFIND`, `PUT`, `DELETE`, `MOVE`, `SEARCH` — file management over HTTP; enabled on content dirs = upload/modify surface.
- **WAF evasion axes** — encodings it doesn't normalize, NULL-byte truncation (native code), uninspected paths (body, multipart, method), split-payload/HPP, novel syntax.
- **Scanner limits (WAHH)** — no improvisation, no intuition, no context: logic flaws, multistage flows, custom sessions, novel variants are manual territory.
