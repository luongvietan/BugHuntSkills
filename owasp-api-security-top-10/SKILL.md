---
name: owasp-api-security-top-10
description: "Knowledge base from the OWASP API Security Top 10 — source book is the 2019 edition, with a maintained crosswalk to the current API1:2023–API10:2023 list. Use when testing APIs for BOLA/IDOR, broken authentication, BOPLA/excessive data exposure & mass assignment, resource-consumption and rate-limit gaps, BFLA, sensitive-business-flow abuse, SSRF, misconfiguration, inventory/shadow-version drift, unsafe third-party API consumption, or writing API-security findings and remediation."
---

<!-- argument-hint: [API risk number, vuln class, or endpoint type] -->

# OWASP API Security Top 10 — 2019 edition + 2023 crosswalk
**Author**: OWASP API Security Project | **Pages**: ~31 | **Risks**: 10 | **Generated**: 2026-09-23 | **Refresh**: crosswalk vs official 2023 list, 2026-09-23

> **Edition note.** The source book documents the **2019** list. The current
> authoritative list is **API Security Top 10 2023** (stable, June 2023 —
> owasp.org/API-Security). Quote 2023 IDs in new reports; use the crosswalk
> below to translate this book's chapter numbers.

## How to Use This Skill

- **Without arguments** — the testing model + risk map below
- **With a risk** — ask about `BOLA`, `mass assignment`, `rate limiting`; I load that chapter
- **With an endpoint** — describe the endpoint (params, methods, auth) and I map which risks to test
- **Browse** — ask "list the top 10"

When you ask about a topic not covered below, I read the relevant chapter file before answering.

---

## Core Testing Model

**The 10 risks, ranked by bounty value:**

| Rank | Risk | One-line test |
|---|---|---|
| API1 | **Broken Object Level Authorization** | Swap object ID (path/query/body/header) → other user's data |
| API2 | **Broken User Authentication** | Brute force/OTP without lockout; JWT `alg:none`/weak validation; creds in URL |
| API3 | **Excessive Data Exposure** | Response fields > UI fields → leaked PII/tokens/internal props |
| API4 | **Lack of Resources & Rate Limiting** | `size=200000`, upload bombs, missing per-client caps |
| API5 | **Broken Function Level Authorization** | HTTP method swap; `users`→`admins`; guess admin endpoints |
| API6 | **Mass Assignment** | Add `is_admin`/`role`/`balance`/internal props to mutation payloads |
| API7 | **Security Misconfiguration** | `.git`/dotfiles, extra HTTP verbs, CORS reflection, stack traces, TLS gaps |
| API8 | **Injection** | SQL/NoSQL (`[$ne]`)/command (`$(x)`) via any input reaching an interpreter |
| API9 | **Improper Assets Management** | `v1↔v2` rotation; beta/staging/legacy hosts missing newer protections |
| API10 | **Insufficient Logging & Monitoring** | Attack traffic raises no alert; log injection |

## 2019 → 2023 crosswalk (current list)

| 2019 chapter | 2023 risk | What changed |
|---|---|---|
| API1 BOLA | **API1:2023 BOLA** | unchanged — still the highest-yield class |
| API2 Broken User Auth | **API2:2023 Broken Authentication** | rename; now explicitly includes 3rd-party auth-provider flaws |
| API3 Excessive Data Exposure | **API3:2023 Broken Object Property Level Authorization (BOPLA)** | merged with 2019 mass assignment — one category: object *properties* readable or writable without authorization (GET = 2019 API3, PUT/POST = 2019 API6) |
| API6 Mass Assignment | → merged into **API3:2023 BOPLA** | write-side of the same property boundary |
| API4 Lack of Resources & Rate Limiting | **API4:2023 Unrestricted Resource Consumption** | broadened: response size, CPU/memory, per-user quotas, per-request cost — not just rate counts |
| API5 BFLA | **API5:2023 BFLA** | unchanged |
| API7 Security Misconfiguration | **API8:2023 Security Misconfiguration** | renumbered only |
| API8 Injection | **dropped from 2023 top 10** | still test it — injection now lives under generic classes (route: `payloads-all-the-things`, `web-security-academy`) |
| API9 Improper Assets Mgmt | **API9:2023 Improper Inventory Management** | renamed; adds documented-vs-running version drift, data flow inventory |
| API10 Insufficient Logging | **dropped from 2023 top 10** | keep as secondary/defense-in-depth note in reports |
| — (new) | **API6:2023 Unrestricted Access to Sensitive Business Flows** | NEW: abuse of a flow working as designed — scalping, spam, fake-account/voucher farming, comment flooding. Test = low-volume simulation of the abuse pattern on your own accounts, never at scale |
| — (new) | **API7:2023 SSRF** | NEW: URL-reachable parameters, webhook/callback registration, file-import-by-URL, redirect chains → `payloads-all-the-things` SSRF chapter + `hacking-the-cloud` for metadata paths |
| — (new) | **API10:2023 Unsafe Consumption of APIs** | NEW: the API *you* consume is the risk — trusting third-party responses, weak TLS to providers, unvalidated upstream data (stored-XSS via partner API), loose provider scope |

Every 2023 risk is covered: 7 map to existing chapters, API6/API7/API10 route
to the methods above. No coverage gaps.

**Universal method:** (1) inventory hosts/versions/endpoints; (2) probe with 2 accounts — horizontal for BOLA, vertical for BFLA; (3) diff responses vs UI for exposure; (4) fuzz IDs, methods, params, operators; (5) check the version-gap on shadow hosts.

**Highest-yield first pass:** BOLA ID-swap on every object endpoint → method swap + admin-path guesses → response/UI diff → OTP/reset rate-limit check → `v1` rotation.

**Report framing:** BOLA and OTP-brute-force = account-takeover class; mass-assignment severity = property sensitivity; never actually DoS for rate-limit findings — show the mechanism with a few requests.

---

## Chapter Index

| # | Title | Key Content |
|---|-------|----------------|
| [ch01](chapters/ch01-broken-object-level-authorization.md) | API1 Broken Object Level Authorization | ID-swap methodology, header IDs, GUID caveat |
| [ch02](chapters/ch02-broken-user-authentication.md) | API2 Broken User Authentication | credential stuffing, OTP brute force, JWT flaws, auth-flow coverage |
| [ch03](chapters/ch03-excessive-data-exposure.md) | API3 Excessive Data Exposure | response/UI diff, generic serializers, token leaks |
| [ch04](chapters/ch04-lack-of-resources-rate-limiting.md) | API4 Lack of Resources & Rate Limiting | size-param amplification, upload bombs, limiter keying |
| [ch05](chapters/ch05-broken-function-level-authorization.md) | API5 Broken Function Level Authorization | method swap, admin-path guessing, role mapping |
| [ch06](chapters/ch06-mass-assignment.md) | API6 Mass Assignment | property hunting, GET→PUT write-back, chain to RCE |
| [ch07](chapters/ch07-security-misconfiguration.md) | API7 Security Misconfiguration | hardening gaps, CORS, headers, verbs, error leaks |
| [ch08](chapters/ch08-injection.md) | API8 Injection | NoSQL operators, command injection, interpreter map |
| [ch09](chapters/ch09-improper-assets-management.md) | API9 Improper Assets Management | version rotation, shadow hosts, protection gaps |
| [ch10](chapters/ch10-insufficient-logging-monitoring.md) | API10 Insufficient Logging & Monitoring | detection absence, log injection, alert gaps |

## Topic Index

- **Account takeover** → ch02, ch01
- **Admin endpoints / privilege escalation** → ch05, ch06
- **Brute force / OTP / credential stuffing** → ch02, ch04
- **CORS / headers / TLS** → ch07
- **DoS / resource exhaustion** → ch04
- **IDOR / object access** → ch01
- **JWT attacks** → ch02
- **NoSQL / SQL / command injection** → ch08, ch06
- **PII / data leaks** → ch03, ch09
- **Rate limiting** → ch04, ch02
- **Staging / beta / legacy hosts** → ch09
- **Logging / detection** → ch10

## Supporting Files

- [glossary.md](glossary.md) — every term (BOLA, BFLA, mass assignment…) with definitions
- [patterns.md](patterns.md) — probe techniques: ID swap, method swap, response diff, rotation, fuzzing
- [cheatsheet.md](cheatsheet.md) — surface→test table, severity hints, risk score matrix

---

## Scope & Limits

Source = 2019 edition; the crosswalk above maps every chapter to the current
2023 list — quote 2023 IDs in reports. Offensive techniques are for authorized
testing only; respect program scope, never degrade production — resource-
consumption and business-flow tests run low-volume on your own accounts.
See `sources.md` for edition + review metadata.
