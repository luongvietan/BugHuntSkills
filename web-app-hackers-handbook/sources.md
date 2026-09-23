# sources.md — web-app-hackers-handbook

## Sources

- **Book**: *The Web Application Hacker's Handbook: Finding and Exploiting
  Security Flaws* — Stuttard & Pinto, Wiley, **2nd Edition (2011)** (PDF in
  the source repo). The 1st edition (2007) was deliberately not converted.
- No living upstream: the book is frozen at 2011; a 3rd edition does not
  exist. Modern successors for each era-dependent area are routed in-file.
- Modern coverage routed to: `web-security-academy` (smuggling/desync,
  cache, prototype pollution, JWT/OAuth, LLM — classes postdating the book),
  `owasp-api-security-top-10` (2023 API taxonomy), `recon-pipeline`
  (tooling currency), `owasp-wstg` (versioned checklist authority).
- Methods-refresh source register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
  (BugBounty workspace).

## Provenance & edition policy

- Value kept: vulnerability *mechanics* — server-side reasoning, assumption
  violations, Hack-steps checklists — which remain accurate because they
  describe how code fails, not how tools look.
- Value marked historical: tool versions, auth mechanisms (pre-MFA/JWT/
  OAuth/WebAuthn), XSS filters (pre-CSP/Trusted Types), browser-era
  techniques (E4X/VBScript/JScript/Flash — already era-marked in-file),
  and any technique requiring a victim client.
- Per-chapter `> **Edition note (2011):**` callouts carry the routing; this
  file is the single point of edition truth.

## Review history

- 2026-09-23: edition-dependence callouts added to ch03/05/07/11/13/19/20,
  SKILL.md + cheatsheet + patterns era notes, sources.md created
  (methods-refresh T5).
