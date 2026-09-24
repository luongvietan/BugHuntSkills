# sources.md — owasp-api-security-top-10

Reviewed: **2026-09-24**.

## Sources

- **Book**: *OWASP API Security Top 10 — 2019* (PDF in the source repo; the
  project's first stable edition, RC/final 2019).
- **Current authority**: OWASP API Security Top 10 **2023** (stable release,
  June 2023) — https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- Project home: https://owasp.org/www-project-api-security/
- Methods-refresh source register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
  (BugBounty workspace) — pins edition status and related references.

## Provenance & edition policy

- Chapters `ch01–ch10` are synthesized from the **2019** list (the book's
  structure — chapter numbers match 2019 risk IDs).
- `SKILL.md` carries the maintained **2019 → 2023 crosswalk** and coverage
  routing for the three 2023-new risks (API6 business flows, API7 SSRF,
  API10 unsafe API consumption). Quote 2023 IDs in new reports.
- 2019 risks dropped from the 2023 list (Injection API8, Logging API10) are
  kept — they remain real test classes; the crosswalk marks their status.

## Review history

- 2026-09-23: crosswalk added vs official 2023 edition page; new-risk routing
  verified; sources.md created (methods-refresh T4).
- 2026-09-24: Core Testing Model changed to current API1:2023–API10:2023
  names and bounded first checks; 2019 chapter paths remain explicitly
  historical and routed through the crosswalk.
- Next review trigger: a new OWASP API Top 10 edition or release candidate.
