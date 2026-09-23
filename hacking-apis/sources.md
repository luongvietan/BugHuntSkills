# sources.md — hacking-apis

## Sources

- **Book**: *Hacking APIs* — Corey Ball, No Starch Press, **Early Access**
  edition (PDF in the source repo). Early-access text predates the OWASP API
  Security Top 10 **2023** edition.
- **Current taxonomy**: OWASP API Security Top 10 **2023** (stable, June 2023)
  — https://owasp.org/API-Security/editions/2023/en/0x11-t10/
  The companion skill `owasp-api-security-top-10/SKILL.md` carries the
  maintained 2019→2023 crosswalk this skill references.
- Tooling currency: `recon-pipeline` covers amass v4↔v5 differences; tool
  versions pinned in the methods-refresh source register
  (`docs/superpowers/2026-09-23-bug-bounty-source-register.md`, BugBounty
  workspace).

## Provenance

- Chapters are synthesized from the book; the book's own chapter numbers are
  preserved in file titles (`Ch6`, `Ch7`…) — file numbers ≠ book numbers.
- 2026-refresh additions are marked inline with `> **…(2026 refresh):**`
  callouts: 2023-taxonomy note (ch01), authorization-boundary matrix (ch04),
  volume rules (ch05), evasion gating (ch10), GraphQL bounded probing (ch11),
  amass v5 note (ch03).

## Review history

- 2026-09-23: API-2023 crosswalk routing, volume/gating callouts, sources.md
  created (methods-refresh T4).
