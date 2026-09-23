# sources.md — web-hacking-101

## Sources

- **Book**: *Web Hacking 101: How to Make Money Hacking Ethically* —
  Peter Yaworski, Leanpub (edition ~2017), PDF in the source repo.
  Case-study format: each bug class pairs with real disclosed reports
  and bounty amounts.
- Current cross-references (2026 refresh): `payloads-all-the-things`
  (payload families + per-class ceilings), `web-security-academy`
  (modern desync/cache/PP classes the book predates), `hacking-the-cloud`
  (SSRF→cloud rules), `owasp-api-security-top-10` (API targets),
  `report-writing` (current severity/reporting norms).
- Methods-refresh source register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
  (BugBounty workspace).

## Provenance & restraint policy

- **Historical case studies**: bounty amounts, platforms, and report
  excerpts are era artifacts — do not quote them as current norms.
  The transferable content is the reasoning (what they noticed, what
  they tried next), not the payload text.
- Per-chapter callouts (`> **…rule/ceiling:**`) bound the risky classes:
  SQLi → differential/sleep proof only; open redirect → own-domain PoC;
  subdomain takeover → marker page; RCE/SSTI/SSRF → harmless markers,
  no credential use, no internal sweep.
- The case-study chains (e.g., "redirect → OAuth token theft") are
  escalation *narrative* for reports, not live-program procedure.

## Review history

- 2026-09-23: chapter callouts (ch06/07/09), cheatsheet/patterns era
  notes, SKILL.md era note, sources.md created (methods-refresh T8).
