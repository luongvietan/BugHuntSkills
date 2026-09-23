---
name: report-writing
description: Use when drafting, submitting, or disputing bug bounty / vulnerability disclosure reports — turning a confirmed bug into a persuasive report, choosing a defensible severity, writing reproduction steps a triager can replay in under 5 minutes, phrasing impact for each vuln class, and responding to triage verdicts (N/A, informative, duplicate, spam) or escalating to mediation. Authorized, in-scope testing only.
---

# Report Writing — Bug Bounty Reports That Get Paid

A report is a persuasion document, not a writeup. The reader is a triager who spends ~5 minutes on it and has seen your vuln class a hundred times. They are paid to say "no" to weak claims; your job is to make "yes" the path of least resistance: reproducible in minutes, impact stated in their employer's language, severity argued with evidence rather than adjectives.

**Most reports fail on demonstrated impact, not on the bug.** "SQLi exists" is a finding; "SQLi reads the `users` table — here are two rows from my own test accounts" is a bounty.

## How to use

| Situation | File |
|---|---|
| Picking / defending a severity (CVSS, platform bands, argue up vs. down) | `chapters/01-severity.md` |
| Assembling the report (skeleton, title format, repro-step rules) | `chapters/02-template.md` |
| Writing the impact section for a specific vuln class (phrasing, escalation language, what not to claim) | `chapters/03-impact-library.md` |
| Report came back N/A / informative / duplicate / spam, or you're considering a dispute or mediation | `chapters/04-triage.md` |

Drafting order that works mid-hunt: fill `chapters/02-template.md` skeleton → write repro steps first (they force you to re-verify the bug) → pick the impact statement from `chapters/03-impact-library.md` → set severity with `chapters/01-severity.md` → proofread once against the checklist below → submit → if verdict disappoints, `chapters/04-triage.md`.

## Golden rules

1. **Test accounts only.** Every PoC touches only accounts and data you own. Two accounts you control is enough to prove A→B access control failures; real user data converts a bounty into a ban.
2. **Minimal viable PoC.** Stop at the smallest demonstration that proves impact: a `sleep(5)`, one extra record, `alert(document.domain)` — never a table dump, never pivoting further "to show how bad it could be". Over-exploitation violates most policies and gives triage a reason to close as policy breach.
3. **Repro must run cold.** Steps start from a clean state (fresh browser/incognito, named test accounts) and include the exact request. If the triager can't replay it in <5 minutes without asking you a question, it isn't done.
4. **One bug per report.** Chained bugs go in one report *as a chain*; unrelated bugs never share a report — the weaker one drags the verdict down.
5. **Platform channel only.** Everything about the finding stays inside the program's report thread until the policy's disclosure terms allow otherwise. No tweets, no blog, no "advisory" on your GitHub before coordinated disclosure.
6. **Argue impact, never money.** "This deserves critical" + evidence beats "other programs paid $X for this" every time. Severity is a technical claim; bounty amount is the program's business.
7. **Show the attacker, not the mechanic.** Impact section answers "what does a malicious actor gain" — records read, accounts taken, money moved — not "the app fails to validate the `id` parameter".

## The 30-second checklist (run before every submit)

- Title follows `[class] in [feature] at [endpoint] → [impact]` and names the impact, not just the bug.
- Summary states vuln class + affected component + business impact in ≤3 sentences.
- Repro steps: numbered, from clean state, exact requests, expected vs. actual result, all identifiers test-account-owned.
- Impact quantifies: how many users/records, which data types, what privilege gained, pre-auth or post-auth.
- Severity matched the program's rubric first (platform policy second, CVSS third — v4.0 where the program's calculator shows it; see `chapters/01-severity.md` for metric names).
- Remediation suggests a concrete fix (object-level authZ check, parametrized query, CSRF token binding) — not "sanitize input".
- Scope re-verified: asset is in-scope, technique not on the banned list, no real-user data touched, no DoS.
- Screenshots/logs attached inline at the step they illustrate; video only if the platform accepts it and the bug needs timing/context.

## Failure modes (why reports actually die)

| Symptom | Root cause | Fix |
|---|---|---|
| Closed N/A "no security impact" | Bug described, attacker outcome never demonstrated | Rewrite impact section from `chapters/03-impact-library.md`; add the escalation step you skipped |
| Closed informative | Real bug, theoretical impact ("could maybe leak…") | Demonstrate one concrete victim scenario on your own accounts, or accept informative and move on |
| Closed duplicate | Everyone looks where you looked | Ask once if it was known-vs-recent; take it as recon signal — change angle or target, don't re-argue |
| Severity slashed vs. your estimate | You scored the bug, not the impact; or overclaimed scale | Score from `chapters/01-severity.md` rubric; only claim scale you proved |
| Slow death in "needs more info" | Repro steps assume your session state | Rewrite steps from clean state with exact requests |

## Scope & ethics

All use of this skill assumes authorized testing against in-scope assets under a program policy you've read. When policy and opportunity conflict (out-of-scope asset, banned technique, real user data within reach), policy wins — a great out-of-scope bug pays $0 and can cost your account. Related skills: `bug-bounty-bootcamp` (per-class hunting methodology), `web-hacking-101` (case studies of real reports), `recon-pipeline` (where to point the hunting), `xss-cheat-sheet` (payloads for the PoC itself). Severity-source + evidence-handling metadata: `sources.md`.
