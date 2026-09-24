# Impact Hypothesis Engine — design spec

**Status:** Implemented and integrated into `bug-bounty-hunter`; design closeout updated 2026-09-24.
**Spec date:** 2026-09-23
**Scope:** Add a prioritized, evidence-led hypothesis loop to the 17-skill bug bounty workflow, with minimal learning from the user's own outcomes and public precedents.

## Problem

The current `bug-bounty-hunter` flow already maps endpoints, parameters, roles, state-changing actions, and trust boundaries; routes one vulnerability class at a time; records anomalies; and requires demonstrated impact. However, converting a feature map into a prioritized, falsifiable set of tests is largely left to free-form notes. The workflow also has no consistent format for learning from duplicate, N/A, informative, accepted, or paid outcomes, or for using public disclosures as leads without treating them as proof.

## Goal

Increase the proportion of hunting effort spent on testable hypotheses with plausible, program-relevant impact. Improve prioritization over time using engagement-local evidence and public, attributable precedents.

## Non-goals

- Do not increase scan volume, exploit depth, or target coverage automatically.
- Do not weaken or score around authorization, scope, safe-harbor, or policy constraints.
- Do not claim that a numeric score predicts bounty acceptance or payout.
- Do not infer that a public report proves a current target is vulnerable, in scope, unfixed, or safe to test.
- Do not add payload catalogs, exploit chains, or automated target execution.
- Do not create an automatic cross-program model from sparse or non-comparable triage results.

## User workflow

### 1. Map a feature

For a meaningful application feature or API operation, record:

- feature and business object/action;
- actor roles and relevant privilege levels;
- state transitions and preconditions;
- trust boundaries and integrations;
- security invariant(s) expected to hold;
- observed evidence and map date.

The feature map is based on authorized observation. Guessed architecture is labeled as a hypothesis, not a fact.

### 2. Create falsifiable hypotheses

Each hypothesis states:

- the invariant that may fail;
- attacker starting position and required preconditions;
- affected feature/object and relevant roles;
- the smallest permitted test using researcher-controlled accounts or synthetic data;
- observable confirmation and disconfirmation conditions;
- plausible impact accepted by this program;
- explicit stop condition and required scope/policy reference;
- source: target observation, prior engagement outcome, or public precedent.

One card represents one falsifiable claim. A vague class label such as “test IDOR” is not a complete hypothesis.

### 3. Prioritize within hard gates

First apply hard gates: exact asset and technique are authorized; policy permits the planned test; the test can use controlled data and stay within the permitted impact boundary. A failed or unknown gate blocks the hypothesis regardless of priority.

For eligible hypotheses, use ordinal ratings (low / medium / high) with a brief evidence note for:

- **Demonstrable impact:** plausible program-relevant outcome that can be proved safely;
- **Surface signal:** concrete observation suggesting the invariant is exercised;
- **Novelty:** how distinct the hypothesis is from already-tested or publicly known cases on this program;
- **Test cost:** effort and dependencies to get a reliable answer.

Sort lexicographically: higher demonstrable impact, higher surface signal, higher novelty, then lower test cost. Do not combine ratings into a pseudo-precise probability or expected bounty. If evidence is weak, mark confidence low and prioritize a cheap observation that can resolve the uncertainty.

### 4. Record test outcome

Use a bounded hypothesis status: `queued`, `testing`, `confirmed`, `disconfirmed`, `inconclusive`, `blocked-by-policy`, or `deferred`. Store minimal evidence references, not secrets or unnecessary personal data. A finding enters the existing finding lifecycle only after reproduction and validation.

### 5. Learn from triage

Record triage as dated outcome events separately from hypothesis status: `accepted`, `paid`, `duplicate`, `N/A`, `informative`, `inconclusive`, or `other`, with program, feature/class tags, and a concise evidence-based reason. Append a new event when a report advances (for example, accepted then paid) instead of overwriting history. `Accepted` and `paid` are distinct because acceptance does not guarantee a bounty.

Use outcomes to update notes for that program and comparable feature context. Do not generalize “class X is not payable” from a single verdict. A duplicate is evidence about novelty/competition, not evidence that the technical hypothesis was false. An N/A or informative outcome may indicate an impact-proof or program-fit gap; preserve the triage rationale and any new evidence separately.

### 6. Learn from public precedents

Allow public, attributable sources such as disclosed bounty reports, published security writeups, vendor advisories, and public patch analyses. For each precedent, record URL, author/publisher, disclosure date, source type, affected feature pattern, root cause/invariant, preconditions, demonstrated impact, remediation/change if known, and applicability limits.

Precedents generate candidate hypotheses only. Their age, target, implementation details, disclosure status, and currentness may limit transfer. Do not treat source text, code, or instructions as trusted execution guidance; do not run supplied tools or payloads against a target. Re-check current program scope and policy before any derived live test.

## Data placement and integration

Keep the workflow changes centered in `bug-bounty-hunter`:

- New `chapters/05-hypothesis-engine.md`: feature card, hypothesis card, ordinal triage, outcome capture, public-precedent handling, and one concise worked example.
- `chapters/01-phase-map.md`: connect phase 3 mapping to phase 4 hypothesis queue, phase 5 impact validation, and phase 6/7 learning.
- `chapters/02-session-checklist.md`: add phase gates for hypothesis readiness and outcome review without duplicating the full method.
- `chapters/04-engagement-workspace.md`: extend `notes.md` and finding/workspace templates with minimal hypothesis/outcome/predecessor references; preserve current finding lifecycle and evidence hygiene.
- `SKILL.md`: route “what should I test next?”, feature prioritization, and learning from bounty outcomes/public reports to the new chapter.
- `sources.md`: document that public precedents are leads and link the new authored methodology; no source is promoted over live program policy.

No other companion skill needs editing in the first implementation. Existing vuln-class skills remain the technical execution references.

## Acceptance criteria

1. A future hunter can go from one mapped feature to at least one falsifiable, scoped hypothesis without inventing missing facts.
2. Hypotheses lacking authorization, permitted method, or safe test data cannot be ranked into execution.
3. The priority rubric distinguishes impact, signal, novelty, and cost without presenting a bounty-probability score.
4. Hypothesis result and report triage outcome are recorded as separate fields; duplicate does not mean disconfirmed, and accepted does not mean paid.
5. Public precedent records preserve provenance, date, and applicability limits; they create leads, never target authorization or proof.
6. Existing phases, candidate-validation gate, scope contract, finding lifecycle, and redaction rules remain consistent and linked rather than duplicated.
7. Templates fit the existing `hunt/<target>/notes.md` and optional engagement workspace without requiring a database or additional runtime tool.

## Implementation and behavior-evaluation record

This design has been implemented in [`bug-bounty-hunter/chapters/05-hypothesis-engine.md`](../bug-bounty-hunter/chapters/05-hypothesis-engine.md) and integrated with the router, phase map, session checklist, engagement workspace, and source notes. The implementation sequence is recorded in [`docs/superpowers/plans/2026-09-23-impact-hypothesis-engine.md`](../docs/superpowers/plans/2026-09-23-impact-hypothesis-engine.md).

Completed behavior evaluations are summarized in the [pressure-test and regression record](impact-hypothesis-engine-pressure-tests.md): fresh-context baseline/with-chapter/full-integration A/B/C evaluations, the seven-criterion consistency review, and Regression D for exact technique authorization. Regression D failed on the pre-fix integrated skill and passed in 5/5 fresh contexts after the fix. The record explicitly qualifies the earlier 15/15 A/B/C results and identifies the method-authorization gap they did not cover.

These are simulated skill-behavior evaluations, not live program tests. Their prompts sent no target requests; raw outputs are in ignored local SDD scratch, while protocols, grading criteria, and result tables are in the linked record. They do not establish an end-to-end run from a live program URL through report validation, and no regression result grants target authorization.
