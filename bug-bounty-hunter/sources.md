# Sources & currency — bug-bounty-hunter

Reviewed: **2026-09-23**. This is an authored ops skill (no book source); its
authority is the source hierarchy below plus the verified 2026 state of its 16
companion skills.

## Source hierarchy (binding order)

1. **Current program policy + written authorization** — outranks everything
   in this skill set. Pinned per session in `hunt/<target>/scope.md`
   (URL + revision/date).
2. **Official standards/primary docs** — OWASP API Security Top 10 2023,
   WSTG v4.2 (+`latest` for draft), OWASP MAS, OWASP GenAI Security Project
   (LLM Top 10 2025), FIRST CVSS v4.0, provider docs for cloud behavior.
3. **Maintained research/labs** — PortSwigger Web Security Academy (rolling);
   lab reproduction ≠ bounty-safe validation, ever.
4. **Tool vendor docs** — version-scoped command syntax only.
5. **Books/community methods** — foundational context; edition limits labeled.

## What changed in the 2026 refresh

- Deny-by-default scope contract added to SKILL.md + session checklist
  (policy pinning, exact allowlist, lead-vs-scope rule, stop conditions).
- Credentials moved to password manager/secret store; notes carry aliases.
- Recon routing split: passive collection → target-traffic (allowlist-derived)
  → intrusive (separately gated). No discovered host/IP auto-in-scope.
- Router entries added for API Top 10 **2023** naming, AI/LLM surfaces
  (OWASP GenAI 2025), severity precedence (program > platform > CVSS v4).
- Scope pressure scenarios added to session checklist (no scope file,
  unlisted host, CDN/shared IP, found credential).

## Companion-skill freshness pointers

- `owasp-api-security-top-10` — refreshed to the 2023 edition (2019 labels
  kept historical). Router cites filenames, not edition years.
- `web-security-academy` — LLM chapter expanded to OWASP GenAI 2025 risks.
- `recon-pipeline` — commands re-verified against tool releases of
  2026-09-23 (Amass **v5.1.1** is a rewrite — v4 flags do not carry over).

Full per-skill register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
in the BugBounty workspace.
