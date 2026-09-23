---
name: bug-bounty-hunter
description: Entry point and router for an authorized bug bounty hunting session. Use when starting or resuming a hunt, deciding which companion skill or chapter applies right now, picking a methodology for a target type (wildcard scope, single web app, API, mobile, cloud), choosing a vuln class to work, or asking "which skill covers X" — it runs the seven-phase loop (program selection, recon, application mapping, vuln hunting, escalation and chaining, reporting, continuous monitoring) and routes each phase to the right one of the 16 companion skills and its chapter files. All testing is assumed authorized and in scope; every active-scanning stage is gated on written authorization plus a fresh scope re-check.
---

# bug-bounty-hunter — session router for the 16-skill arsenal

Not a book and not a technique reference — the dispatcher. A real hunt moves
through seven phases; each phase has a primary skill and named chapter files.
Load this skill to decide *where to look next*; load the routed skill to learn
*how*. Working backward — jumping to payloads before mapping, reporting before
reproducing — is how sessions produce noise instead of bounties.

## The seven-phase loop

```
1 program selection    pick a program you can win on; read the whole policy
        |
        v
2 recon                passive-first asset discovery  (ACTIVE STAGES GATED:
                       written authorization + scope re-check before any
                       packet touches the target)
        |
        v
3 application mapping  walk every feature at every privilege level
        |
        v
4 vuln hunting         work vuln classes one at a time (see vuln-class index)
        |
        v
5 escalation/chaining  turn "bug exists" into "attacker gains X"; chain lows
        |
        v
6 reporting            draft while context is fresh; minimal viable PoC
        |
        v
7 continuous monitoring -> diff next recon run, hunt the delta, loop to 2
```

Phase detail, inputs, exit criteria, and per-phase warnings:
`chapters/01-phase-map.md`. Session start-to-finish checklist:
`chapters/02-session-checklist.md`. Vuln class -> chapter routing:
`chapters/03-vuln-class-index.md`.

## The scope contract — deny by default

Nothing below this line happens until the session-init contract in
`chapters/02-session-checklist.md` is filled. In short:

- **Written policy on file** — URL, revision/date read, safe-harbor clause,
  report channel, stop/contact conditions.
- **Exact allowlist** — in-scope assets enumerated; wildcard semantics written
  down (`*.target.com` includes apex? sub-subdomains? which TLDs?).
- **A discovered asset is a lead, not scope.** A hostname, IP, or bucket found
  during recon is unverified until ownership and allowlist membership are
  confirmed. Shared/CDN infrastructure and third-party services never inherit
  authorization from a related hostname.
- **Missing or ambiguous means stop.** No policy, no allowlist entry, unclear
  technique permission, a service warning, or unexpected real-user data →
  halt that stage, re-check policy or contact the program. Do not proceed on
  a guess; the guess is how accounts get banned.
- **Credentials live in a password manager/secret store.** Notes, logs, and
  evidence carry account aliases and references only — never stored secrets.

## Routing table — phase

| Phase | Primary route | Also load |
|---|---|---|
| 1 Program selection | `tbhm-methodology` ch01, `zseano-methodology` ch01 | `bug-bounty-bootcamp` ch01 (industry, report expectations) |
| 2 Recon | `recon-pipeline` (all 3 files; passive collection first, target-traffic stages run allowlist-derived lists only, intrusive stages separately gated) | `tbhm-methodology` ch02, `bug-bounty-bootcamp` ch03, `zseano-methodology` ch02 |
| 3 App mapping | `web-app-hackers-handbook` ch03, `tbhm-methodology` ch03 | `owasp-wstg` ch01, `zseano-methodology` ch05-ch06 |
| 4 Vuln hunting | `chapters/03-vuln-class-index.md` picks the per-class chapter | `bug-bounty-bootcamp`, `web-security-academy`, `owasp-wstg`, `web-app-hackers-handbook` |
| 5 Escalation/chaining | `bug-bounty-playbook` (exploitation-phase ops) | `bug-bounty-bootcamp` ch14, `web-hacking-101` (chain precedent) |
| 6 Reporting | `report-writing` (all 4 files) | `web-hacking-101` ch10, `bug-bounty-bootcamp` ch01 |
| 7 Monitoring | `recon-pipeline` 03-monitoring | `zseano-methodology` ch07 (review findings, pick next target) |

## Decision rules — target type and situation

- **API target** -> `hacking-apis` (ch03 discover, ch05 auth, ch07 BOLA/BFLA,
  ch08 mass assignment = BOPLA-write, ch09 injection, ch11 GraphQL) + `owasp-api-security-top-10`
  as the vuln-class checklist — aligned to the **2023 edition** (API1 BOLA,
  API3 BOPLA, API6 sensitive business flows, API7 SSRF, API10 unsafe API
  consumption; 2019 names are historical labels inside that skill).
- **AI/LLM product surface** (chatbots, agents, RAG features, LLM-backed
  endpoints) -> `web-security-academy` ch15 for Web LLM attack classes +
  OWASP GenAI Top 10 2025 risk names (prompt injection, sensitive info
  disclosure, excessive agency, vector/embedding, unbounded consumption) —
  test owned prompts/agents only; never aim a lab technique at shared model
  capacity.
- **General web testing** -> `bug-bounty-bootcamp` per-class chapters, or
  `web-app-hackers-handbook` for deeper mechanism, `owasp-wstg` for checklist
  coverage, `web-security-academy` for the modern class.
- **Modern classes** (request smuggling, HTTP/2 desync, prototype pollution,
  CSWSH, JWT/OAuth) -> `web-security-academy` ch01-ch09 — other books predate them.
- **Cloud touch** (SSRF reaches metadata, IAM keys found, buckets/snapshots) ->
  `hacking-the-cloud` ch02-ch05.
- **Mobile** -> `owasp-mas` (methodology in ch01, test areas ch02-ch08) +
  `bug-bounty-bootcamp` ch20 for the Android bounty angle.
- **"Which payload for this context"** -> `payloads-all-the-things` (reference
  skill — answers payload questions, not methodology); `xss-cheat-sheet` for
  XSS filter-bypass specifically.
- **Wide-scope wildcard** (`*.target.com`) -> `tbhm-methodology` ch02-ch03 +
  `recon-pipeline` for the runnable pipeline.
- **Deep technique / evasion** (WAF bypasses, weird parsers, protocol edge
  cases) -> `web-app-hackers-handbook`; `web-hacking-101` for a case-study
  precedent showing the class paid out before.
- **Methodology / mindset / "how do good hunters work"** -> `zseano-methodology`,
  `tbhm-methodology`.
- **Exploitation-phase ops** (brute-forcing with Burp, known-CVE checks, CMS,
  cache attacks, OSRF) -> `bug-bounty-playbook`.
- **Report drafting, severity, triage disputes** -> `report-writing`. Severity
  is program-rubric first, platform policy second; CVSS v4.0 is the current
  FIRST standard but only where a program accepts or requests it — never a
  universal score.
- **Scope or permission question mid-hunt** -> stop; re-read
  `hunt/<target>/scope.md` and the live policy. If the answer is still
  ambiguous, contact the program — the router has no "probably fine" route.

## Golden rules

1. **Scope before skill.** No routing decision outranks the program policy.
   Re-read scope at every phase gate; a newly found asset is a lead, not
   permission.
2. **Active stages need written authorization.** Passive recon is default-safe.
   DNS brute-force, port scans, dir brute-force, fuzzing, and exploit attempts
   require the policy's explicit safe-harbor — `recon-pipeline` gates them
   per-stage; honor the same gates when any skill tells you to send traffic.
3. **One vuln class at a time.** Breadth-first mapping, then depth-first per
   class. The index in `chapters/03-vuln-class-index.md` exists so "which skill
   for X" is a lookup, not a rathole.
4. **Test accounts only, minimal PoC.** Two accounts you control prove A-to-B
   access bugs; a `sleep(5)` proves injection. Over-exploitation converts a
   bounty into a policy breach.
5. **Low bugs are chain links.** Note how each finding could combine (self-XSS
   + login CSRF + CORS leak = ATO) before writing it off — phase 5 is a phase,
   not an afterthought.
6. **Report while fresh; recon while you wait.** Draft the report the day the
   bug lands. While triage chews, run the next recon diff — phases 6 and 7 feed
   phase 2, not the couch.

## Files

- `chapters/01-phase-map.md` — seven phases: purpose, inputs, actions,
  skill+chapter routing, exit criteria, do-not-skip warnings.
- `chapters/02-session-checklist.md` — copy-paste session init, per-phase
  gates, end-of-session wrap.
- `chapters/03-vuln-class-index.md` — vuln class -> primary skill+chapter ->
  alternates -> payload chapter.

## Scope & ethics

Every routed skill assumes authorized testing against in-scope assets under a
program policy actually read. This router adds nothing to that authority — it
spends it more carefully. When policy and opportunity conflict (out-of-scope
asset, banned technique, real user data within reach), policy wins; a great
out-of-scope bug pays $0 and can cost the account. Source hierarchy + review metadata: `sources.md`.
