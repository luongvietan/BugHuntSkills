# sources.md — report-writing

## Sources

- **Severity precedence**: the *program's* published severity table →
  platform taxonomy/policy → CVSS as supporting evidence. No universal
  platform claims are made; platform specifics are date-stamped 2026-09.
- **CVSS v4.0** (current FIRST standard): spec —
  https://www.first.org/cvss/v4-0/specification-document ; calculator —
  https://www.first.org/cvss/calculator/4-0 . Metric names used in
  `chapters/01-severity.md` follow the v4.0 spec (AT, UI Passive/Active,
  VC/VI/VA + SC/SI/SA). CVSS v3.x retained only for programs that require it.
- **Platform taxonomies**: Bugcrowd VRT (bugcrowd.com/vulnerability-rating-taxonomy),
  HackerOne program severity tables (per-program).
- Methods-refresh source register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
  (BugBounty workspace) — pins CVSS v4.0 as current standard.

## Evidence-handling policy

- PoC evidence: test-account-owned identifiers only; secrets masked
  (`ak12…wxyz`); third-party PII never attached — described, not shown.
- Demonstrated vs theoretical: report claims are either demonstrated
  (evidence attached) or explicitly labeled theoretical escalation.
- Template (`chapters/02-template.md`) requires a scope & evidence
  declaration block: asset-in-scope line + policy version, account
  ownership statement, data-handling statement.

## Review history

- 2026-09-23: severity precedence reframed (program→platform→CVSS), CVSS
  v4.0 metrics added, template evidence declaration added, redaction rules
  expanded, sources.md created (methods-refresh T7).
- Next review trigger: CVSS v4.x revision or major platform policy change.

## Design references (attribution)

- `chapters/02-template.md` pre-draft gate + evidence checklist adapt
  concepts from Claude-BugHunter (github.com/elementalsouls/Claude-BugHunter);
  wording is original to this skill.
- 2026-09-23: T11 added platform-overlay note + evidence checklist +
  pre-draft gate pointer.
