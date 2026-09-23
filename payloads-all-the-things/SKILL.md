---
name: payloads-all-the-things
description: Payload selection library distilled from swisskyrepo/PayloadsAllTheThings. Use when an orchestrator or vuln-class skill asks "which payload for X context" — picking, adapting, and encoding payloads for the core web classes (XSS, SQLi, NoSQL/LDAP/XPath, SSTI, SSRF, XXE, LFI/RFI/traversal, command injection, uploads, deserialization, GraphQL, open redirect/CSRF/clickjacking, auth & tokens/JWT/SAML/OAuth, HTTP smuggling/CORS/CRLF/HPP, IDOR/logic/race/prototype-pollution/misc injection). Each chapter is organized as payload families with context, encoding variants, when-to-use notes, and expected signals. Not a methodology book — pair with a methodology skill for hunting workflow. Assume all testing is authorized and in scope.
---

# PayloadsAllTheThings — Payload Selection Library

Condensed, reorganized reference built from swisskyrepo's **PayloadsAllTheThings** (~40 vuln-class directories). The unit of organization is the **payload family**: a context or prerequisite, a set of example payloads, encoding/parser variants, the expected signal, and a when-to-use note.

Mental model: **context first, payload second.** Every payload in this library only works inside a specific parser context (HTML attribute vs JS string vs URI vs SQL clause vs template expression vs file parser). Identify the context, then pick the family, then mutate for the filter.

Related skills: `bug-bounty-bootcamp` + `web-hacking-101` (vuln-class methodology and hunting workflow), `web-security-academy` (PortSwigger lab structure), `xss-cheat-sheet` (deeper XSS-only reference), `owasp-wstg` (test-ID coverage), `hacking-apis` + `owasp-api-security-top-10` (API-specific testing), `tbhm-methodology` / `zseano-methodology` (engagement workflow), `recon-pipeline` (attack surface), `report-writing` (documenting findings).

## How to use

**Injection — server-side**
- Reflected/stored JS in a page → `chapters/ch01-xss.md`
- Relational DB errors/quotes/arithmetic → `chapters/ch02-sqli.md`
- MongoDB/CouchDB/LDAP/XML-query params → `chapters/ch03-nosql-ldap-xpath.md`
- `{{`, `${`, `#{`, `<%` evaluated in responses → `chapters/ch04-ssti.md`
- URL fetched server-side, metadata, internal hosts → `chapters/ch05-ssrf.md`
- XML/SOAP/docx parsed server-side → `chapters/ch06-xxe.md`
- `include()`/file path params, `../`, wrappers → `chapters/ch07-file-inclusion.md`
- Shell metacharacters reach `system()`/`exec()`/popen → `chapters/ch08-command-injection.md`
- Serialized blobs (`rO0`, `O:`, `gASV`, `AAEAAAD`, `BAgK`) → `chapters/ch10-deserialization.md`
- GraphQL endpoint → `chapters/ch11-graphql.md`
- File upload endpoints → `chapters/ch09-upload.md`

**Client-side / cross-site**
- `?url=`/`?next=`/`?redirect=` params, tabnabbing, CSRF forms, iframe abuse, WebSocket hijack → `chapters/ch12-open-redirect-csrf.md`

**Identity & tokens**
- JWT/SAML/OAuth/session/reset-token/MFA/brute-force → `chapters/ch13-auth-session.md`

**HTTP & platform**
- CL/TE desync, cache deception, CRLF, HPP, CORS, XS-Leak, vhosts, exposed panels → `chapters/ch14-http-infra.md`

**Logic & misc injection**
- IDOR/BOLA, race, mass assignment, hidden params, prototype pollution, type juggling, CSV/LaTeX/XSLT/SSI/ESI, ReDoS, dependency confusion, prompt injection → `chapters/ch15-logic-misc.md`

- Terms → `glossary.md`; selection heuristics → `patterns.md`; context→payload quick map → `cheatsheet.md`

## Selection doctrine

1. **Classify the context before choosing a payload.** "Reflected in HTML attribute inside double quotes" is a different problem than "reflected in `<script>` body." Payloads fail silently when the context guess is wrong.
2. **Start with a detection payload, not an exploit payload.** `'`, `{{7*7}}`, `${{<%[%'"}}%.`, `;sleep 5`, `<!ENTITY example "x">` — prove evaluation first, escalate second.
3. **Baseline vs anomaly.** Compare the payload response against the plain response (status, size, timing, errors). A payload that changes nothing teaches nothing.
4. **Match the parser, not the spec.** Filters are implementation details: test the bypass for *that* WAF/engine/validator — encoding, case, comments, null bytes, unicode normalization, parser differentials.
5. **Blind channel when there's no output.** Time delays (SLEEP/WAITFOR/sleep), DNS lookups, HTTP callbacks to tester-controlled infrastructure (interactsh/Burp Collaborator). Never point OOB payloads at infrastructure you don't control or aren't authorized to use.
6. **Minimal proof.** `id`, `hostname`, a benign `alert(document.domain)`, reading `/etc/hostname` — enough to prove impact. No data dumping, no destructive commands, no reverse shells unless explicitly authorized.
7. **Throttle.** Fuzzing, brute force, race conditions, and batch attacks are rate-sensitive — slow down, respect program limits, ask before volumetric tests.

## Scope & ethics

All payloads assume an authorized, in-scope engagement (bug bounty program, pentest contract, owned lab). Rules of restraint adapted from the source material and the sister skills: read the policy first; use test accounts; prove impact with harmless payloads (`alert(1)`, `id`, `sleep`, a canary file you own); do not extract real user data, do not run destructive or DoS-class payloads (billion laughs, fork bombs, ReDoS at scale) against production; OOB callbacks only to tester-controlled or program-authorized infrastructure; stop at the smallest demonstration that shows impact.
