---
name: owasp-wstg
description: Structured test checklist distilled from the OWASP Web Security Testing Guide (WSTG) v4.2. Use when systematically testing an authorized web target end-to-end or when you need the canonical WSTG-XXX test IDs — information gathering and fingerprinting, configuration/deployment review, identity management, authentication, authorization, session management, input validation (XSS, SQLi and every other injection family), error handling, cryptography, business logic, client-side attacks, and API/GraphQL testing. Each test gives objective → procedure → interpretation so you can work a category as a checklist, quote test IDs in reports, or pick the right procedure for a suspected weakness. Assume all testing is authorized and in scope.
---

# OWASP Web Security Testing Guide v4.2 — Test-ID Checklist

> **Edition status (verified 2026-09-23):** v4.2 is the **current stable**
> WSTG release. A v5.0 rewrite is in development upstream (rolling, unstable)
> — its drafts are not quoted here. WSTG-BUSL-10 (Test Payment Functionality)
> was added to live WSTG after the v4.2 PDF froze; it is synthesized in
> ch10 with provenance marked. Version/review metadata: `sources.md`.

Knowledge base distilled from OWASP WSTG v4.2 (the successor to OTGv4). The guide's signature is a **numbered test catalog**: 98 named tests organized into 12 categories (97 in the frozen v4.2 PDF plus WSTG-BUSL-10 added in live WSTG), each identified by a stable ID of the form `WSTG-<CATEGORY>-<NN>` (e.g. `WSTG-INFO-02` = the second Information Gathering test). Quote these IDs in findings to anchor each vulnerability to a recognized methodology.

Mental model: testing splits into **passive** (walk the app through a proxy, understand logic, map every access point — parameters, headers, cookies, APIs) and **active** (work the 12 categories below against each access point found). Every test follows the same skeleton: objective → how to test (black-box first, gray-box where code/config is available) → what a positive result looks like → remediation. Error messages, version banners, and anomalous responses are the raw material; a reported finding must still name its impact.

Related skills: `bug-bounty-bootcamp` + `web-hacking-101` (same vuln classes with hunter framing and case studies), `bug-bounty-playbook` (tooling-heavy playbook), `hacking-apis` + `owasp-api-security-top-10` (deeper API coverage beyond WSTG-APIT-01), `xss-cheat-sheet` (payload reference for WSTG-INPV-01/02 and WSTG-CLNT-01), `zseano-methodology` (recon-first workflow that feeds INFO/CONF).

## How to use

**Start of an engagement (work top-down)**
- Search-engine/metafile recon, server & framework fingerprinting, entry-point and architecture mapping → `chapters/ch01-information-gathering.md`
- Platform config, HTTP methods, admin interfaces, backup files, subdomain takeover, cloud storage → `chapters/ch02-configuration-deployment.md`

**Identity surface**
- Roles, registration, provisioning, account enumeration → `chapters/ch03-identity-management.md`
- Credential transport, default creds, lockout, auth bypass, reset flows → `chapters/ch04-authentication.md`
- Directory traversal, horizontal/vertical authz bypass, privilege escalation, IDOR → `chapters/ch05-authorization.md`
- Session schema, cookie attributes, fixation, CSRF, logout/timeout, puzzling, hijacking → `chapters/ch06-session-management.md`

**Injection & server-side abuse**
- XSS, SQLi (per-DBMS), LDAP/XML/SSI/XPath/IMAP injection, code/command injection, file inclusion, format string, HTTP splitting/smuggling, host header, SSTI, SSRF → `chapters/ch07-input-validation.md`
- Error messages & stack traces → `chapters/ch08-error-handling.md`
- TLS config, padding oracle, plaintext channels, weak algorithms → `chapters/ch09-cryptography.md`

**Logic, client, API**
- Workflow circumvention, forged requests, timing, function limits, file upload abuse → `chapters/ch10-business-logic.md`
- DOM XSS, open redirect, CSS/resource manipulation, CORS, clickjacking, WebSockets, postMessage, browser storage, XSSI → `chapters/ch11-client-side.md`
- GraphQL introspection, authorization, injection, DoS, batching → `chapters/ch12-api-testing.md`

- Terms → `glossary.md`; reusable heuristics → `patterns.md`; quick ref → `cheatsheet.md`

## The WSTG doctrine (mental models)

1. **Test IDs are the checklist.** Work a category end-to-end or pick tests by suspected weakness; record each ID tested so coverage is auditable. Three IDs are merged stubs: WSTG-INFO-09 → INFO-08, WSTG-INPV-03 → CONF-06, WSTG-ERRH-02 → ERRH-01.
2. **Passive before active.** You can't choose the right test until you've mapped entry points, tech stack, and roles. INFO chapter output is the input inventory for everything else.
3. **Fingerprint, then attack the specific.** Nearly every test starts "identify the technology, then apply technology-specific probes." Generic fuzzing is the fallback, not the default.
4. **Every test wants an oracle.** A positive result is a behavioral delta — different status, length, timing, error text, or state change. Record the baseline before manipulating.
5. **Client-side controls are claims, not controls.** Hidden fields, disabled inputs, JS validation, and role flags are all server-testable assumptions; WSTG reuses this idea across AUTHZ, BUSL, and IDNT.
6. **Vary one variable at a time.** WSTG's discipline for isolating injection points — change a single parameter, keep the rest constant, so cause and effect stay linked.
7. **Impact requires a second look.** A reflexion, an error, or a weird response is a candidate, not a finding — chain it (enum → brute force; traversal → config files → creds → admin) or document why impact stops there.

## Scope & ethics

All testing described here assumes written authorization and in-scope targets (pentest engagement or bug bounty program). WSTG itself is methodology-neutral about authorization — the restraint is yours: prefer test accounts, avoid destructive payloads (file writes, account lockout of real users, DoS tests) unless explicitly permitted, throttle brute-force and fuzzing, capture minimal proof of impact, and never access other users' real data to prove a bug.
