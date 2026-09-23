# Ch1-2: Picking Programs & Writing Reports

> **Currency note:** platform names, payout norms, and program mechanics are era-specific (book ~2021) — the *selection criteria* and report discipline are durable; current severity/reporting rules live in `report-writing`.

Source: Chapters 1-2. Bug bounty = find vulns in a scoped app, report via a platform (HackerOne, Bugcrowd, Intigriti, Synack, Cobalt…), get paid on severity. Li's framing: treat it as a profession — program selection, report quality, and dispute handling are skills separate from hacking.

## Choosing a program

- **Scope breadth vs. competition.** Big public scopes (Google, Facebook) = more surface but thousands of hunters. Smaller/newer/private programs = less competition, more low-hanging fruit.
- **Industry knowledge beats raw skill.** Pick targets in an industry you understand — you'll spot business-logic flaws pure pentesters miss.
- **Tech stack match.** Target stacks you know (or want to learn). Recon for technologies first; a .NET shop wastes a PHP expert's time.
- **Response metrics matter.** Check time-to-triage, time-to-bounty, average payouts, dispute resolution before investing. A program that triages in 3 months is a different job than one triaging in 3 days.
- **Safe-harbor / policy.** Read the policy page *before* testing: in-scope assets, banned techniques (DoS, social engineering, physical), rate-limit rules, automated-scanning stance.

## Sustainability & failure diagnosis

Li's advice for long-term hunting: set aside regular hours, specialize in 2-3 vuln classes until fluent, automate repetitive recon, take breaks during "bug slumps" (normal — targets get patched, not you getting worse), and treat duplicates as information (you're looking where everyone looks → change recon angle or target).

Diagnose why reports fail: out of scope, duplicate, N/A (no security impact shown), informational (real bug, no demonstrated impact). Most "wasted" reports = failure to *show* impact, not failure to find bugs.

## Writing the report

Structure every report the same way:

1. **Title**: `[Class] in [feature] at [endpoint] leads to [impact]` — e.g. `IDOR on /api/messages allows reading any user's private messages`.
2. **Summary**: 2-3 sentences — vuln class, affected component, business impact.
3. **Severity assessment**: state CVSS if required; otherwise argue impact (data exposure scale, privileges needed, user interaction).
4. **Steps to reproduce**: numbered, copy-pasteable, starting from a clean state. Include the exact request, account setup, and expected vs. actual result. A triager should repro in <5 min without asking questions.
5. **Impact analysis**: what an attacker gains; quantify when possible (number of records, privileges).
6. **Remediation suggestions**: concrete fixes (parametrize queries, enforce object-level authZ, add CSRF token validation).

Golden rules: use **test accounts only** for PoC; never access/modify real user data; stop at the *minimal* demonstration of impact (don't dump a table to prove SQLi — `sleep` or a version string suffices); disclose only via the program channel; don't go public before coordinated disclosure is allowed.

## Disputes & duplicates

- If marked N/A but you believe it's valid: reply once, politely, with *added evidence* — better PoC, escalation chain, real-world scenario. Never argue about money; argue about impact.
- If duplicate: ask (once) whether it was a known issue or a recent report; use it to learn where other hunters look.
- Escalation path: platform mediation exists on HackerOne/Bugcrowd — use it sparingly and with documentation.
