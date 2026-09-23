---
name: owasp-api-security-top-10
description: "Knowledge base from \"OWASP API Security Top 10 - 2019\". Use when testing APIs for BOLA/IDOR, broken authentication, excessive data exposure, rate-limit gaps, BFLA, mass assignment, misconfiguration, injection, shadow/legacy API versions, or writing API-security findings and remediation."
---

<!-- argument-hint: [API risk number, vuln class, or endpoint type] -->

# OWASP API Security Top 10 — 2019
**Author**: OWASP API Security Project | **Pages**: ~31 | **Risks**: 10 | **Generated**: 2026-09-23

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

2019 edition (the newer 2023 list renames some risks — e.g. API3→Broken Object Property Level Authorization, API6→Unrestricted Access to Sensitive Business Flows — but the test methodology carries over). Offensive techniques are for authorized testing only; respect program scope, never degrade production.
