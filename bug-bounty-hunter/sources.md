# Sources & currency — bug-bounty-hunter

Reviewed: **2026-09-24**. This is an authored ops skill (no book source); its
authority is the source hierarchy below plus the verified 2026 state of its 16
companion skills.

## Source hierarchy (binding order)

1. **Current program policy + written authorization** — outranks everything
   in this skill set. Pinned per session in `hunt/<target>/scope.md`
   (URL + revision/date).
2. **Official standards/primary docs** — OWASP API Security Top 10 2023,
   WSTG v4.2 (+`latest` for draft), OWASP ASVS 5.0.0, OWASP MAS, OWASP
   GenAI LLM Top 10 2026, Top 10 for Agentic Applications 2026, selected
   OWASP AI Testing Guide v1 methods, FIRST CVSS v4.0, and provider docs.
3. **Maintained research/labs** — PortSwigger Web Security Academy (rolling);
   lab reproduction ≠ bounty-safe validation, ever.
4. **Tool vendor docs** — version-scoped command syntax only.
5. **Books/community methods** — foundational context; edition limits labeled.
6. **Public precedent** — disclosed reports, writeups, vendor advisories,
   patch analyses. Historical intelligence, not authority: generates
   leads/candidate hypotheses only and never proves a current target is
   vulnerable, in scope, or unfixed — live policy (1) plus fresh
   observation decide. Record format: `chapters/05-hypothesis-engine.md`.

## What changed in the 2026 refresh

- Deny-by-default scope contract added to SKILL.md + session checklist
  (policy pinning, exact allowlist, lead-vs-scope rule, stop conditions).
- Credentials moved to password manager/secret store; notes carry aliases.
- Recon routing split: passive collection → target-traffic (allowlist-derived)
  → intrusive (separately gated). No discovered host/IP auto-in-scope.
- Router entries added for API Top 10 **2023** naming, AI/LLM surfaces
  (OWASP GenAI LLM Top 10 2026 and Agentic Top 10 2026), severity
  precedence (program > platform > CVSS v4).
- Scope pressure scenarios added to session checklist (no scope file,
  unlisted host, CDN/shared IP, found credential).
- Impact hypothesis engine added (`chapters/05-hypothesis-engine.md`) —
  authored methodology: feature/hypothesis cards, hard gates, ordinal
  ranking, append-only outcome ledger, precedent records as leads.

## Companion-skill freshness pointers

- `owasp-api-security-top-10` — refreshed to the 2023 edition (2019 labels
  kept historical). Router cites filenames, not edition years.
- `web-security-academy` — chapter 15 uses OWASP GenAI LLM Top 10 2026,
  the 2026 Agentic risks that map to testable product boundaries, and selected
  AI Testing Guide v1 test IDs; 2025 is historical.
- `recon-pipeline` — commands re-verified against tool releases of
  2026-09-23 (Amass **v5.1.1** is a rewrite — v4 flags do not carry over).

Full per-skill register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
in the BugBounty workspace.

## AI source anchors (checked 2026-09-24)

- OWASP GenAI LLM Top 10 2026: https://genai.owasp.org/resource/owasp-genai-llm-top-10-2026/
- OWASP Top 10 for Agentic Applications 2026: https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/
- OWASP AI Testing Guide v1 (published 2025-11-26): https://owasp.org/projects/ai-testing-guide
- The AI Testing Guide's application, model, infrastructure, and data test
  families are broader than bounty work. Route only security checks that
  touch an in-scope product boundary, and keep policy, controlled-data, and
  least-impact gates in force.

## Methodology crosswalk

- **Basis**: user-shared *BUG BOUNTY METHODOLOGY 2026*, version 3.0,
  reviewed 2026-09-24 —
  https://chatgpt.com/share/6ab41390-e9f8-83ec-b468-53f2a0e39166 .
- Its 12 activity stages are mapped onto the existing seven-phase router in
  `chapters/01-phase-map.md`; they are labels and handoffs, not a replacement
  lifecycle. Functional mapping feeds a feature-led re-entry to Phase 2 for
  deep recon. Human-controlled AI may suggest hypotheses, but cannot establish
  scope, permission, or validation.
- Program policy/written authorization and the official current sources above
  remain authoritative where the synthesized methodology is silent or differs.

## Design references (attribution)

- `chapters/04-engagement-workspace.md` adapts concepts (engagement
  scaffold, finding lifecycle, triage gate, evidence hygiene) from
  Claude-BugHunter (github.com/elementalsouls/Claude-BugHunter) —
  wording is original to this skill; no upstream text copied.
- 2026-09-23: T11 added ch04 (workspace, lifecycle, 7-question gate,
  evidence hygiene, tracker) + wired into session checklist.
- 2026-09-23: hypothesis-engine series added ch05 — original to this
  skill (no upstream source); wired into ch04 workspace records,
  SKILL.md routing, and the source hierarchy above.
- 2026-09-23: browser-assisted program-intake series added ch06
  (`chapters/06-authenticated-program-intake.md`) — program-URL session
  start through the connected browser: provider/sign-in resolution,
  allowlisted researcher-page read, workflow states + resume re-gating.
  Original to this skill (no upstream source): it reads platform UI
  through the available connected browser MCP/extension; no
  Bugcrowd/HackerOne affiliation is implied and no platform policy text
  is copied into the skill (page wording is read live, excerpted short,
  and cited to its source URL). Wired into SKILL.md routing +
  description, ch01 phase map, ch02 session checklist, and ch04
  workspace fields (`engagement.md`, `allowlist.txt`); acceptance cases
  recorded in `designs/browser-assisted-program-intake-cases.md`.
