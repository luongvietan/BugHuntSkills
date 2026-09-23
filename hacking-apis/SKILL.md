---
name: hacking-apis
description: API hacking methodology from "Hacking APIs" by Corey Ball (No Starch, Early Access). Use when testing APIs end-to-end — choosing black/gray/white box approach, API recon (dorks, Shodan, Amass, GitHub, DevTools, Gobuster, Kiterunner), endpoint analysis with Postman, authentication attacks (brute force, password spraying, token forgery, JWT abuse), wide/deep fuzzing, BOLA/BFLA A-B(-A) testing, mass assignment, SQL/NoSQL/OS/XAS injection, WAF evasion and rate-limit bypass, and GraphQL attacks. Verify asset scope and method permission before live use.
---

# Hacking APIs — Methodology & Attack Playbook

Knowledge base distilled from Corey Ball's *Hacking APIs* (Early Access). The book's core loop:

**Recon → Endpoint analysis → Intended-use baseline → Attack (auth, fuzz, BOLA/BFLA, mass assignment, injection) → Evasion/rate-limit testing → Report.**

Mental model: APIs are self-service machines. Documentation tells you how to use them; your job is to use them *unintended*. Most early wins come not from bypassing firewalls but from simply using an endpoint as designed — with someone else's resource ID, a forged token, or an extra JSON variable.

Companion skill `owasp-api-security-top-10` covers the risk taxonomy — including the maintained **2019 → 2023 crosswalk** (this book predates the 2023 edition; use 2023 IDs in reports) — this skill covers *how to test for it* with tools.

## How to use

- Planning an API test → `chapters/ch00-approach-and-fundamentals.md`
- Mapping vuln classes to what you observe → `chapters/ch01-insecurity-taxonomy.md`
- Building/refreshing a hacking rig → `chapters/ch02-lab-and-tooling.md`
- Finding the API & docs → `chapters/ch03-discovering-apis.md`
- Building request collections & first wins → `chapters/ch04-endpoint-analysis.md`
- Attacking auth, tokens, JWTs → `chapters/ch05-attacking-authentication.md`
- Designing fuzz campaigns → `chapters/ch06-fuzzing.md`
- BOLA/BFLA methodology → `chapters/ch07-authorization-bola-bfla.md`
- Mass assignment → `chapters/ch08-mass-assignment.md`
- Injection (SQL/NoSQL/OS/XSS/XAS) → `chapters/ch09-injection.md`
- WAF/rate-limit bypass → `chapters/ch10-evasion-rate-limits.md`
- GraphQL → `chapters/ch11-graphql.md`
- Case studies + final checklist → `chapters/ch12-breaches-bounties-checklist.md`
- Terms → `glossary.md`; reusable patterns → `patterns.md`; quick ref → `cheatsheet.md`

## The Ball doctrine (mental models)

1. **Distrust and verify.** Docs are a starting point, never complete. Always test methods/endpoints/params not in the documentation.
2. **Baseline first.** Use the API as intended before attacking. Anomalies are only visible against a baseline of normal responses (status code, size, timing, body shape).
3. **Three perspectives.** Analyze every app as guest, authenticated user, and administrator — each reveals different functionality and docs.
4. **Where + what.** Fuzzing success = knowing *where* to fuzz (inputs that reach a database/filesystem/interpreter) and *what* to fuzz with (payloads matched to the backend tech).
5. **Attribution is the exploit surface for evasion.** Stateless APIs must attribute requests via IP, tokens, headers, metadata — rotate or spoof whichever component the control relies on.
6. **Small findings chain.** Info disclosure → targeted fuzzing → verbose error → injection → BOLA → mass assignment → account takeover. A "minor" finding is often the tip of the iceberg.
7. **Test as intended, then as an adversary.** Ask of every endpoint: what can I do, can I touch other users' resources, how are resources identified, can I upload/inject?

## Scope & ethics

Before live testing, verify the exact asset and technique against current program rules. This skill grants no authorization; if scope or permission is missing or unclear, stop and re-check with the program. Destructive-capable attacks (DELETE fuzzing, mass resource changes, `--os-shell`, `--dump-all`) belong on test accounts and non-production data — the book is explicit: validate BFLA deletes on your own resources, and prefer demonstrating impact over actually deleting client data. Volume-gated chapters (auth brute-force concepts ch05, evasion/rate-limits ch10, GraphQL cost tests ch11) carry per-chapter rules: low-volume proof on your own accounts by default; scale needs explicit policy permission. Edition/tool-currency metadata: `sources.md`.
