# sources.md — bug-bounty-playbook

## Sources

- **Book**: *Bug Bounty Playbook V2* — Alex Thomas (ghostlulz), PDF in the
  source repo. Exploitation-phase playbook; era ≈ 2021–2022 tooling.
- Current cross-references: `owasp-api-security-top-10` (API 2023
  taxonomy), `hacking-the-cloud` (credential rules), `web-security-academy`
  (cache/desync/PP ceilings), `payloads-all-the-things` (payload families).
- Methods-refresh source register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
  (BugBounty workspace).

## Provenance & restraint policy

- Chapters are technique synthesis, era ≈ book publication; tool versions
  and platform specifics are point-in-time.
- Per-chapter callouts (`> **…rule/gate/boundary/ceiling:**`) bound every
  risky class: exploit modules → own lab first; scanners → volume gate;
  found creds → report not exercise; exposed DBs → listing-only PoC;
  brute force → low-volume own-account proof; SQLi/XSS/SSTI/XXE/CSP →
  markers; uploads/webshells → inert owned files; cache poisoning → own
  cache-buster; GraphQL introspection → schema, not credential queries.
  Where a body example exceeds its callout (book's escalation chains),
  the callout wins — inline notes mark those spots.
- The playbook mindset is offense-shaped by design — the callouts are the
  contract that keeps it inside program policy.

## Review history

- 2026-09-23: 9 chapter callouts (risky-action gating), cheatsheet/patterns
  ceiling notes, sources.md created (methods-refresh T8).
