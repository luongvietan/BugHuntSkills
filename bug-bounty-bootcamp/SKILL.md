---
name: bug-bounty-bootcamp
description: Bug bounty methodology from "Bug Bounty Bootcamp" by Vickie Li (No Starch). Use when working a web target end-to-end — picking programs, scoping recon (subdomains, certs, GitHub, S3, directory brute-force), hunting the core web vuln classes (XSS, redirects, clickjacking, CSRF, IDOR, SQLi, race conditions, SSRF, deserialization, XXE, SSTI, logic errors, RCE, SOP/CORS, SAML/OAuth, info disclosure), escalating and chaining impact, source code review, Android app hacking, API testing, and fuzzing with Wfuzz/Burp Intruder. Each vuln chapter follows the author's loop — mechanism → prevention → hunting → bypassing protections → escalation → automation. Verify asset scope and method permission before live use.
---

# Bug Bounty Bootcamp — Vickie Li's End-to-End Methodology

Knowledge base distilled from Vickie Li's *Bug Bounty Bootcamp* (25 chapters, 4 parts). The book's signature is a repeatable per-vuln loop:

**Mechanism → Prevention → Hunting → Bypassing Protections → Escalation → Automation → "Finding your first …" checklist.**

Mental model: every input is a potential injection point; every protection is a claim you should verify. Start manually to learn the app's logic, automate the boring parts, and always ask "what does this bug let me do next?" — a low-severity finding is usually a chain link, not an endpoint.

Related skills: `web-hacking-101` (case-study driven version of the same vuln classes), `bug-bounty-playbook` (more aggressive tooling playbook), `hacking-apis` + `owasp-api-security-top-10` (deeper API coverage), `xss-cheat-sheet` (XSS payload reference).

## How to use

**Before hacking**
- Choosing programs, writing reports, staying sustainable → `chapters/ch01-industry-reports.md`
- Web fundamentals + Burp setup → `chapters/ch02-foundations-setup.md`
- Recon pipeline + bash automation → `chapters/ch03-recon.md`

**Hunting a specific vuln class**
- XSS → `chapters/ch04-xss.md` | Redirects & clickjacking → `chapters/ch05-open-redirects-clickjacking.md`
- CSRF → `chapters/ch06-csrf.md` | IDOR → `chapters/ch07-idor.md`
- SQLi → `chapters/ch08-sqli.md` | Race conditions → `chapters/ch09-race-conditions.md`
- SSRF → `chapters/ch10-ssrf.md` | Deserialization → `chapters/ch11-deserialization.md`
- XXE → `chapters/ch12-xxe.md` | SSTI → `chapters/ch13-ssti.md`
- Logic errors & broken access control → `chapters/ch14-logic-access-control.md`
- RCE → `chapters/ch15-rce.md` | SOP/CORS/JSONP/postMessage → `chapters/ch16-sop.md`
- SAML & OAuth → `chapters/ch17-sso.md` | Info disclosure → `chapters/ch18-info-disclosure.md`

**Advanced techniques**
- Source code review → `chapters/ch19-code-review.md` | Android → `chapters/ch20-android.md`
- APIs → `chapters/ch21-api-hacking.md` | Fuzzing → `chapters/ch22-fuzzing.md`

- Terms → `glossary.md`; reusable heuristics → `patterns.md`; quick ref → `cheatsheet.md`

## The Li doctrine (mental models)

1. **Scope first, always.** Read the policy, stay in scope, prefer test accounts. A great out-of-scope bug is worth $0 and can get you banned.
2. **Recon is ROI.** Attack surface → dedupe. Recon tells you which 5 endpoints deserve 5 hours, not which 500 exist.
3. **Manually walk every feature at every privilege level before automating.** You can't fuzz for a business-logic flaw.
4. **Two-account diff.** Most access-control bugs fall out of replaying requests between a low-priv and a high-priv account — including adding params that never appear in the original request.
5. **Protections are hypotheses.** Missing token? Validate when present? Referer check? Each defense has a canonical bypass — test the implementation, not the spec.
6. **Impact is engineered, not found.** Self-XSS + login CSRF + CORS leak = account takeover. Write down how each finding could chain.
7. **Automation is a metal detector.** Fuzzers flag anomalies; manual analysis confirms impact. Understand every tool you run.

## Scope & ethics

Before live testing, verify the exact asset and technique against current program rules. This skill grants no authorization; if scope or permission is missing or unclear, stop and re-check with the program. The book is explicit about restraint: use test accounts, upload only harmless test files you own (S3), avoid reading sensitive data or running destructive payloads, throttle fuzzing to avoid DoS, get written permission before rate-limit tests, and stop at the minimal PoC that proves impact. Edition/tool currency + routing metadata: `sources.md`.
