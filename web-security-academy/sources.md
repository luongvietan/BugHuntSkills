# sources.md — web-security-academy

Reviewed: **2026-09-24**.

## Sources

- **Primary**: PortSwigger Web Security Academy topic guides —
  https://portswigger.net/web-security — free, continuously maintained.
  Each chapter header lists its topic path(s) verbatim.
- **Lab index**: https://portswigger.net/web-security/all-labs — labs are
  deliberately vulnerable sandboxes; their authorization does NOT extend to
  live programs.
- **Current LLM risk model**: OWASP GenAI LLM Top 10 2026 —
  https://genai.owasp.org/resource/owasp-genai-llm-top-10-2026/ — resource
  page dated 2026-08-03; used for current names and boundaries in chapter 15.
- **Agentic risk model**: OWASP Top 10 for Agentic Applications 2026 —
  https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/
  — announced 2025-12-09; route only tool, privilege, goal, and memory
  boundaries that can be assessed with owned test data.
- **Test-method supplement**: OWASP AI Testing Guide v1 —
  https://owasp.org/projects/ai-testing-guide — published 2025-11-26; its
  repository is rolling. Chapter 15 uses only selected application and
  infrastructure test IDs that add actionable procedures to the two Top 10
  taxonomies (APP-01/02/06/08 and INF-03/04).
- **Historical**: OWASP Top 10 for LLM Applications 2025 —
  https://genai.owasp.org/resource/owasp-top-10-for-llm-applications-2025/
  — retain only to interpret older notes; it is superseded for current risk
  names by the 2026 edition.
- Methods-refresh source register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
  (BugBounty workspace).

## Provenance & update policy

- Living source: Academy topics change as PortSwigger publishes research —
  chapters carry per-topic source paths for re-verification.
- ch08 SAML section is explicitly synthesized (no dedicated Academy SAML
  topic); provenance marked in-file.
- 2026-refresh additions marked inline as `> **Lab vs live (bounty-safe):**`
  callouts and the OWASP-2026-mapped ch15 section.

## Review history

- 2026-09-23: LLM section expanded to OWASP GenAI Top 10 structure; lab-vs-
  live callouts added to protocol/race/LLM chapters; sources.md created
  (methods-refresh T5).
- 2026-09-24: chapter 15 and source metadata updated to OWASP LLM/Agentic
  2026 editions and selected AI Testing Guide v1 methods; 2025 is historical.
