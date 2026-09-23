# Ch1: Philosophy — the Crowdsourced Game

> **Era note:** TBHM is competition-shaped (find it first, speed wins) — durable as mindset, but the ecosystem changed: duplicates policy, AI triage, private invites. Speed never overrides the scope contract (`bug-bounty-hunter` session init).

Source: `01_Philosophy`. TBHM opens by framing *which* game you're playing — bug hunting in a bounty is not a solo pentest, and the economics shape the entire methodology.

## Single-sourced vs crowdsourced testing

| Axis | Single-sourced (traditional pentest) | Crowdsourced (bug bounty) |
|---|---|---|
| Target vulns | Common-ish classes, breadth coverage | Bugs that aren't easy to find |
| Time pressure | Your schedule | Racing against time *and* other hunters |
| Competition | None | Direct — duplicates pay $0 |
| Incentive | Finding count, guaranteed payment | Uniqueness + impact; payment scales with severity |
| Quality bar | Approximation/checklist | Demonstrated attack scenario |

Consequences: don't grind the same low-hanging classes everyone else finds first; invest where scanners and first-pass hunters don't reach (weird surface, logic, chains); and write impact, not mechanism.

## Program types

- **1st-party programs** — run by the vendor itself (Google, PayPal, etc.).
- **2nd-party platforms** — Bugcrowd, HackerOne, Synack, etc. aggregate programs; each program's policy defines scope, banned techniques, and payout rules. Read it before anything else.

## Report templates — the speed edge

Because you're racing, pre-build templates for your most-found vuln classes. Custom bugs always get custom writeups, but a recurring class (XSS, CSRF, IDOR, open redirect) should be a fill-in-the-blank skeleton: summary, severity rationale, numbered repro steps, attack scenario, remediation.

- **Critical protip:** always swap in the right URLs/domains before submitting. A template citing the wrong domain is one of the fastest routes to invalidation — it signals a copy-paste report.
- Write for impact: two classic references Haddix points to are Bugcrowd's "advice for writing a great vulnerability report" and the forum thread "attack scenario and impact are key" — both argue the narrative of what an attacker gains is what moves severity.
