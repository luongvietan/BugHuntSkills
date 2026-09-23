# sources.md — web-security-academy

## Sources

- **Primary**: PortSwigger Web Security Academy topic guides —
  https://portswigger.net/web-security — free, continuously maintained.
  Each chapter header lists its topic path(s) verbatim.
- **Lab index**: https://portswigger.net/web-security/all-labs — labs are
  deliberately vulnerable sandboxes; their authorization does NOT extend to
  live programs.
- **LLM risk model**: OWASP Top 10 for LLM Applications (GenAI Security
  Project, 2025 list) — https://genai.owasp.org/llm-top-10/ — used to
  structure the ch15 LLM section beyond the Academy's web-LLM topic.
- Methods-refresh source register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
  (BugBounty workspace).

## Provenance & update policy

- Living source: Academy topics change as PortSwigger publishes research —
  chapters carry per-topic source paths for re-verification.
- ch08 SAML section is explicitly synthesized (no dedicated Academy SAML
  topic); provenance marked in-file.
- 2026-refresh additions marked inline as `> **Lab vs live (bounty-safe):**`
  callouts and the OWASP-LLM-mapped ch15 section.

## Review history

- 2026-09-23: LLM section expanded to OWASP GenAI Top 10 structure; lab-vs-
  live callouts added to protocol/race/LLM chapters; sources.md created
  (methods-refresh T5).
