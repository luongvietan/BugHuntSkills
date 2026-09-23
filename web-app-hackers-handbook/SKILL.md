---
name: web-app-hackers-handbook
description: Deep vulnerability-mechanics knowledge base from "The Web Application Hacker's Handbook" (2nd ed., Stuttard & Pinto, Wiley 2011) — the "why does this attack work" reference. Use when hunting or explaining the core web vuln classes end-to-end: mapping attack surface, bypassing client-side controls, authentication, session management, access controls, SQL/XPath/LDAP/NoSQL injection, OS command injection, path traversal, file inclusion, XML/SOAP and back-end protocol injection, HTTP parameter injection/pollution, business-logic flaws, reflected/stored/DOM XSS + filter bypass, CSRF/clickjacking/cross-domain capture, open redirects, session fixation, attack automation/fuzzing, information disclosure, memory-safety bugs in native components, shared-hosting attacks, app-server/default-content flaws, source code review, tooling, and assessment methodology. Each chapter follows the book's signature: mechanism → attack → defense/evasion → checklists. All testing assumed authorized and in scope.
---

# The Web Application Hacker's Handbook — Stuttard & Pinto's Vulnerability Mechanics

Knowledge base distilled from *The Web Application Hacker's Handbook: Finding and Exploiting Security Flaws*, 2nd Edition (21 chapters). Where `bug-bounty-bootcamp` gives the hunter's loop, this skill gives the *mechanics*: what the server actually does with your input, the assumption a developer made, and the precise point where that assumption breaks. Its signature per vulnerability class:

**Mechanism → Attack → Defense/evasion → Hack-steps checklist.**

Mental model: the client is hostile territory — every request is fully user-controlled (parameters, headers, sequence, timing). Every defense is a *mechanism* implemented in *code*, so it fails the way code fails: case-sensitivity, ordering, canonicalization gaps, partial coverage, a check applied in the wrong place, state kept in the wrong scope. Find the assumption, violate it precisely.

Related skills: `bug-bounty-bootcamp` (program workflow + per-vuln hunting loop), `web-hacking-101` (case studies), `xss-cheat-sheet` (payload reference), `owasp-top-10`/`owasp-api-security-top-10` (taxonomies), `hacking-apis` (API-specific coverage).

## How to use

**Orientation**
- Core security problem + defense mechanisms → `chapters/ch01-defense-mechanisms.md`
- HTTP, cookies, encodings, same-origin, remoting → `chapters/ch02-web-technologies.md`
- Mapping, entry points, hidden content, fingerprinting → `chapters/ch03-mapping-application.md`

**Server-side attack classes**
- Bypassing client-side controls → `chapters/ch04-client-side-controls.md`
- Authentication → `chapters/ch05-authentication.md` | Session management → `chapters/ch06-session-management.md`
- Access controls → `chapters/ch07-access-controls.md`
- SQLi + NoSQL/XPath/LDAP injection → `chapters/ch08-attacking-data-stores.md`
- OS command injection, traversal, file inclusion, SOAP/XML, HPI/HPP, SMTP → `chapters/ch09-backend-components.md`
- Business-logic flaws → `chapters/ch10-application-logic.md`

**Client-side attack classes**
- XSS (reflected/stored/DOM + full filter-bypass catalog) → `chapters/ch11-xss.md`
- CSRF, UI redress, cross-domain capture, redirects, session fixation, local privacy → `chapters/ch12-attacking-users.md`

**Technique, tooling & depth**
- Custom automation, enumeration, fuzzing → `chapters/ch13-automating-attacks.md`
- Information disclosure → `chapters/ch14-information-disclosure.md`
- Native memory bugs (overflow, integer, format string) → `chapters/ch15-native-compiled.md`
- Tiered architecture & shared hosting → `chapters/ch16-application-architecture.md`
- App server, default content, WebDAV, WAFs → `chapters/ch17-application-server.md`
- Source code review signatures → `chapters/ch18-source-code-review.md`
- Intercepting proxies, scanners, custom tools → `chapters/ch19-hackers-toolkit.md`
- End-to-end methodology checklist → `chapters/ch20-methodology.md`

- Terms → `glossary.md`; reusable heuristics → `patterns.md`; quick ref → `cheatsheet.md`

Numbering note: `chNN` file names are generated-skill order; each chapter's title cites the WAHH source chapter, which runs one higher for files `ch02`+ (`ch01` covers WAHH ch1–2, `ch10-application-logic.md` covers WAHH Chapter 11).

## The WAHH doctrine (mental models)

1. **The client is hostile.** Every assumption that "the browser will only send what the app asked for" is a bug waiting to be found. Parameters can be added, removed, reordered, replayed, and re-encoded; stages can be skipped.
2. **Security mechanisms are code.** They have implementations, and implementations have edges: case, order of operations, partial matching, wrong storage scope, validation applied to the wrong representation of the data.
3. **Canonicalization is where filters die.** Decode order (URL → HTML → Unicode → path → query) determines what the filter saw versus what the interpreter gets. Test every decoding boundary, not the payload.
4. **Weakest link arithmetic.** Auth, sessions, and access control interlock; the overall security equals the weakest of the three. A bulletproof login behind a predictable token is still a compromised account.
5. **Data crosses trust boundaries; validate at each one.** Input validation isn't a perimeter filter applied once — it's a per-boundary contract. Data safe entering component A can be lethal entering component B (second-order injection, stored XSS).
6. **State and sequence are attack surface.** Session objects, static/shared storage, workflow order, and timing all encode assumptions. Out-of-order requests, cross-feature state pollution, and race windows are where logic flaws live.
7. **Baseline → anomaly → mechanism.** A tester fuzzes strings; a hacker instruments deltas (status, length, time, cookies, error text) and asks *what processing must be happening* to produce each delta. Inference is a weapon.
8. **Checklists over luck.** The book's `HACK STEPS` are systematic enumeration: every parameter, every stage, every encoding, both accounts. Coverage beats cleverness.

## Scope & ethics

All testing is assumed authorized and in scope (bug bounty / commissioned pentest). Apply the book's techniques with restraint: prefer test accounts you control; demonstrate impact with the smallest possible proof (a version string, your own row, a delay — never a data dump); throttle brute-force, fuzzing, and race-condition attempts so you aren't the load test; obtain permission before volumetric testing; upload only harmless test content; and stop once the vulnerability is proven. Never weaponize findings beyond the agreed scope.
