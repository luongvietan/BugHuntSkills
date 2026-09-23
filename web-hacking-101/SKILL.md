---
name: web-hacking-101
description: Case-study knowledge base from "Web Hacking 101" by Peter Yaworski. Use when hunting web vulnerabilities — each chapter pairs a bug class (HTML injection, HPP, CRLF, CSRF, application logic, XSS, SQLi, open redirect, subdomain takeover, XXE, RCE, template injection, SSRF, memory) with real disclosed reports, bounties, and the takeaways that made them work. Also covers target approach, reporting etiquette, and tooling. Assume all testing is authorized.
---

# Web Hacking 101 — Learning from Real Reports

> Peter Yaworski's book teaches hacking through *disclosed bug-bounty reports* — each vulnerability class is a chapter of real cases (Shopify, Twitter, HackerOne, Google, Facebook, Yahoo…) with difficulty, bounty, and takeaways. The value is pattern recognition: seeing *how* a hunter noticed the bug, not just the payload.

## Core mental models

- **Vuln class ↔ functionality mapping**: don't spray payloads — map each feature to the classes it can host (`url=` → SSRF/redirect; XML upload → XXE; `id=` → IDOR/HPP; 2FA flow → logic).
- **Takeaways > payloads**: each case ends with the insight that mattered — persistence after a failed first attempt, services-as-attack-surface, new functionality as fresh meat.
- **Attack surface ≠ website**: S3 buckets, Zendesk, OAuth apps, staging servers, JS files, mobile APIs, GitHub repos, CIDR ranges — all in scope.
- **Persistence is the exploit**: many cases succeeded on the 2nd–6th attempt (double `uid`, encoded CRLF, second account, second request).
- **Confirm before you submit** — "don't shout hello before crossing the pond."

## Chapter map

| # | Chapter | Load when |
|---|---------|-----------|
| 01 | [Approach: info gathering → testing → automation](chapters/ch01-methodology.md) | Starting a new program; subdomain enum, tech map, 10-step process |
| 02 | [HTML injection, HPP, CRLF](chapters/ch02-injection-primitives.md) | Reflected HTML, duplicate params, `%0d%0a` in headers/cookies |
| 03 | [CSRF](chapters/ch03-csrf.md) | State-changing requests without token validation |
| 04 | [Application logic](chapters/ch04-application-logic.md) | Mass assignment, race conditions, privesc, S3, 2FA, hidden endpoints — the flagship chapter |
| 05 | [XSS](chapters/ch05-xss.md) | Reflected/stored/self-XSS + real bypass patterns (malformed HTML, second-order fields) |
| 06 | [SQLi + open redirect](chapters/ch06-sqli-redirect.md) | SQLi via framework flaws; `redirect=`, `domain_name=`, `checkout_url=` params |
| 07 | [Subdomain takeover + stale assets](chapters/ch07-subdomain-takeover.md) | Dangling CNAMEs, unclaimed SaaS, OAuth `redirect_uri` abuse |
| 08 | [XXE](chapters/ch08-xxe.md) | XML/docx/gpx uploads; blind XXE → OOB DTD exfil chain |
| 09 | [RCE + template injection + SSRF](chapters/ch09-rce-template-ssrf.md) | ImageMagick-style command injection, `{{7*7}}` SSTI/CSTI, `url=` → AWS metadata |
| 10 | [Memory vulns + report writing](chapters/ch10-memory-reporting.md) | Buffer overflow/null byte basics; disclosure guidelines, report content, triage empathy |

## Quick reference

- `cheatsheet.md` — per-class probe list (param → payload → expected signal).
- `patterns.md` — the takeaway patterns distilled ("if X then test Y").
- `glossary.md` — terms (HPP, CRLF splitting, OOB XXE, CSTI, mass assignment…).

## Author's calibration (Yaworski)

- Start on **broad-scope, no-bounty programs** — less competition, same bugs; skill first, money after.
- `<img src="x" onerror=alert(1)>` everywhere + `{{4*4}}[[5*5]]` on Angular — the two universal probes he plants during mapping.
- **Don't alert() in reports** — explain what the bug does to *their* site ("steal session of any user who views this page"), not what XSS is.
- Read the disclosure guidelines *before* hunting — his first Shopify report was a known-out-of-scope bug: -5 rep and a lesson.
- Bug bounty is relationships: triagers fight noise, prioritization, confirmation, resourcing — write reports that make their job easy.

Era note: case studies are historical (book ~2017) — payouts, tools, and even some techniques are period artifacts. The durable value is how researchers *found and reasoned about* the bugs. Current-method routing + proof ceilings: `sources.md` and per-chapter callouts.
