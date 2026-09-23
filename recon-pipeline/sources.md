# sources.md — recon-pipeline

## Sources

- **Authored ops skill** (no book source). Authority = pinned tool
  versions + traffic-class model + companion skills.
- Tool versions pinned 2026-09-23 (see source register):
  Amass **v5.1.1** (v4/v5 CLI split documented), Nmap 7.991, httpx
  v1.12.0, dnsx v1.3.1, TruffleHog v3.97.6, Gitleaks v8.30.1,
  Gowitness 3.2.0 (`scan file -f --write-db`), Masscan 1.3.2
  (exceptional path only).
- Methods-refresh source register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
  (BugBounty workspace).

## Contract summary

- Traffic classes: **P** passive third-party / **T** target traffic /
  **I** intrusive — T+I gated on written authorization, allowlist-derived
  target files, exclusions, conservative rates.
- Discovered assets are *leads* until confirmed on the allowlist;
  `*.example.com` never covers the apex.
- Dated outputs `recon/<target>/<YYYYMMDD>/`, canonical + raw artifacts,
  `new-since-last-run.txt`, passive/active scheduled separately with a
  fresh scope re-check before active runs.
- Secrets found during recon are never written to notes/logs — counts,
  locations, and masked prefixes only.

## Review history

- 2026-09-23: runbook hardened (allowlist heredoc fix, wildcard semantics,
  Amass v5 + Gowitness v3 syntax, exclusion path fix, masscan→exceptional,
  secret-handling rules) — methods-refresh T3; sources.md created T9.
