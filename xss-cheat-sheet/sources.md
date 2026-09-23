# sources.md — xss-cheat-sheet

## Sources

- **Book**: *XSS Cheat Sheet* — Rodolfo Assis (Brute Logic), PDF in the
  source repo. ~2018 era; browser targets Firefox 58 / Chrome 63.
- Current cross-references (2026 refresh): `payloads-all-the-things`
  ch01 (XSS payload families + ceiling), `web-security-academy` (DOM
  clobbering, CSP/taint-flow modern depth), `report-writing` (impact
  narrative for the report).
- Methods-refresh source register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
  (BugBounty workspace).

## Provenance & restraint policy

- Context model (HTML/JS/attr/URL contexts) is durable and remains the
  core value; filter/WAF bypass specifics and browser quirks are
  era-marked — mutate against the observed filter.
- **Live-program ceiling**: `alert(document.domain)` is the whole PoC.
  Stored/blind vectors run only in own-account surfaces; victim-visible
  delivery, session/token capture, and exfil receivers are report
  narrative unless explicitly authorized — and real-user cookies/tokens
  are never collected even then.
- Per-chapter callouts (`> **…marker/boundary/note:**`) bound each
  section: basics → proof marker; advanced → delivery boundary;
  filter-bypass → era + pacing gate; exploitation → narrative not
  procedure; misc → lab-verify exotics.

## Review history

- 2026-09-23: 5 chapter callouts, SKILL.md scope section extended,
  cheatsheet/patterns ceiling notes, sources.md created (methods-refresh T8).
