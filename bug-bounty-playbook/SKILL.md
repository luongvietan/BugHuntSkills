---
name: bug-bounty-playbook
description: Exploitation-phase playbook from "Bug Bounty Playbook V2" by Alex Thomas (ghostlulz). Use when exploiting a scoped bug-bounty target — known-CVE workflow, CMS scanners, GitHub dorking, subdomain takeover, exposed databases, brute forcing, Burp workflow, SQLi/XSS/upload/IDOR techniques, API testing (REST/SOAP/GraphQL, JWT/SAML attacks), web cache poisoning/deception, SSTI, XXE, CSP bypass, RPO. Assume all testing is authorized and in scope.
---

# Bug Bounty Playbook V2 — Exploitation Phase

> Book 2 of the author's methodology: Book 1 covered recon + fingerprinting; this volume is the **exploitation phase** — "where all the true hacking occurs". Manual-first: learn each technique by hand before automating.

## The meta-loop

1. **Fingerprint → known vulns**: identify the tech stack (Wappalyzer + footer/banner fallbacks), search Google/ExploitDB/NVD for vulns, find a PoC (GitHub/ExploitDB — beware fake PoCs), test on a local vuln box, then hit the target.
2. **1-day racing**: monitor exploit feeds (ExploitDB, Twitter). When a new CVE drops, mass-scan your known targets before defenders patch. Speed is the exploit.
3. **OWASP core**: ~80% of bounties come from a handful of classes — XSS (most-paid), SQLi, IDOR. Know them cold, by hand, across DB engines and contexts.
4. **Everything is a lead in Burp**: live in HTTP history; POST → stored XSS/CSRF, URL with id/email/username → IDOR, JSON MIME → API, `url=`/`callback=` params → SSRF/SOP bypass.

## Chapter map

| # | Chapter | When to load |
|---|---------|--------------|
| 01 | [Known-vulnerability cycle](chapters/ch01-known-vulnerabilities.md) | Any new target; fingerprint → CVE → PoC → exploit; 1-day scanning |
| 02 | [CMS hacking](chapters/ch02-cms.md) | WordPress/Drupal/Joomla/AEM/other CMS detected |
| 03 | [GitHub dorking + subdomain takeover](chapters/ch03-github-subdomain-takeover.md) | Easy high-impact leads; dangling CNAME fingerprints |
| 04 | [Exposed databases](chapters/ch04-exposed-databases.md) | firebaseio.com URL, port 9200/27017/5984/9042 open |
| 05 | [Brute forcing + Burp Suite](chapters/ch05-brute-force-burp.md) | Any login screen; proxy/intruder/repeater workflow |
| 06 | [SQL injection](chapters/ch06-sql-injection.md) | Quotes → errors; MySQL/Postgres/Oracle union + error-based |
| 07 | [XSS](chapters/ch07-xss.md) | Reflected/stored/DOM; breakouts, polyglot, cookie-stealer PoC |
| 08 | [Upload, traversal, redirect, IDOR](chapters/ch08-upload-traversal-redirect-idor.md) | Upload forms, `?page=` params, redirect params, object IDs |
| 09 | [API types](chapters/ch09-api-types.md) | Identify REST/RPC/SOAP/GraphQL; GraphQL introspection |
| 10 | [API auth + docs](chapters/ch10-api-auth-docs.md) | Basic/JWT/SAML attacks; Swagger/Postman/WSDL/WADL discovery |
| 11 | [Web cache poisoning + deception](chapters/ch11-cache-attacks.md) | CDNs/caching; unkeyed inputs, path confusion |
| 12 | [SSTI](chapters/ch12-ssti.md) | `{{7*7}}` detection; Jinja2/Tornado/ERB/Slim/Freemarker RCE |
| 13 | [OSRF, prototype pollution, CSTI](chapters/ch13-osrf-prototype-csti.md) | Lesser-known classes; Angular expressions |
| 14 | [XXE, CSP bypass, RPO](chapters/ch14-xxe-csp-rpo.md) | XML in requests; CSP header analysis; path-confusion defacement |

## Quick reference

- `cheatsheet.md` — payload/command checklist per vulnerability class.
- `patterns.md` — recurring decision rules ("if you see X → test Y").
- `glossary.md` — terms (unkeyed input, XSW, MRO, path confusion…).

## Field instincts (author's calibration)

- *"If there is a login screen it should be brute forced"* — try default creds first (SecLists), it's cheap.
- *"If you see `<?xml` in Burp → test XXE immediately."*
- *"If you see `*.firebaseio.com` → append `/.json`."*
- Adobe AEM ≈ instant win — riddled with public vulns (`aemhacker`).
- Self-XSS isn't dead — chain with **web cache poisoning** to make it stored.
- Open redirect alone is low — chain into OAuth token theft / SSRF.
- Un-guessable IDs may just be `md5(int)` — hash small integers and check.
- Always demo impact past `alert()`: cookie theft → account takeover.
- When you find an unknown CMS/service: ExploitDB CVEs → GitHub scanner → else move on (unless hunting 0-days).
