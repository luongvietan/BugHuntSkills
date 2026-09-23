# sources.md — bug-bounty-bootcamp

## Sources

- **Book**: *Bug Bounty Bootcamp* — Vickie Li, No Starch Press (2021),
  PDF in the source repo.
- Current cross-references (2026 refresh): `recon-pipeline` (recon
  runbook + tool versions), `owasp-api-security-top-10` (2023 API
  taxonomy), `hacking-apis`, `owasp-mas` (Android depth beyond ch20),
  `web-security-academy` (race/desync/PP depth), `hacking-the-cloud`
  (SSRF→metadata rules), `report-writing` (severity/reporting current).
- Methods-refresh source register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
  (BugBounty workspace).

## Provenance & currency policy

- Vuln-class concepts are durable — XSS/CSRF/IDOR/SSRF/logic chapters are
  the book's core value and stay as-is.
- Era-specific content marked with `> **…note/gate/boundary:**` callouts:
  platform/payout norms (ch01), recon tooling (ch03 → recon-pipeline),
  IDOR→BOLA/BOPLA naming (ch07), race tooling (ch09), SSRF metadata
  boundary (ch10), Android → owasp-mas (ch20), API → 2023 taxonomy (ch21),
  fuzzing volume gate (ch22).

## Review history

- 2026-09-23: 8 chapter currency/gating callouts, cheatsheet/patterns
  notes, sources.md created (methods-refresh T8).
