# Ch5 — Impact Hypothesis Engine

The bridge between phase 3's feature map and phase 4's per-class hunting:
map a feature to a card, write falsifiable hypotheses against its
invariants, gate hard, rank ordinally, and record outcomes so the next
session inherits evidence instead of impressions. All records are plain
Markdown in `notes.md` (or the engagement workspace in
`04-engagement-workspace.md`) — no tooling, no database.

This chapter prioritizes and records; it adds no authorization. Where it
and the scope contract (`02-session-checklist.md`) disagree, the contract
wins.

## Feature card — one per meaningful feature

```markdown
### Feature card
- Feature / business object and action:
- Actor roles:
- State transitions / preconditions:
- Trust boundaries / integrations:
- Expected security invariant:
- Observed evidence + date:
- Unknowns (label, do not infer):
```

Fill from authorized observation only — guessed architecture is a
hypothesis, not a fact. **The Unknowns line is not optional.** Any field
you cannot back with observed evidence — the server-side authorization
rule, the business impact, a precondition — is written `unknown`, plus
the observation that would resolve it. Never fill a gap from experience,
convention, or a writeup; "I'm in a hurry, fill in what's missing"
changes nothing — a field is evidence or it is `unknown`.

## Hypothesis card — one falsifiable claim each

```markdown
### Hypothesis card
- ID / feature:
- Hypothesis status: queued | testing | confirmed | disconfirmed | inconclusive | blocked-by-policy | deferred
- Invariant that may fail:
- Attacker position / required preconditions:
- Exact asset + scope/policy reference:
- Exact method/action permission in current program policy (quote or reference; unknown if absent):
- Permitted controlled-data test:
- Confirmation condition / disconfirmation condition:
- Plausible program-relevant impact:
- Stop condition:
- Source / evidence reference:
- Ratings + evidence: impact __; signal __; novelty __; test cost __; confidence __
```

- One card = one falsifiable claim. "Test IDOR" is a class label, not a
  hypothesis — the confirmation/disconfirmation pair is what makes it
  falsifiable.
- Source is one of: target observation, prior engagement outcome, public
  precedent (see precedents below).
- `Plausible program-relevant impact` stays `unknown — to be
  demonstrated` until a confirmed result proves it; an if-confirmed claim
  is labeled conditional, never asserted.
- Evidence references point at sanitized request pairs under account
  aliases — never secrets or real-user data (evidence hygiene lives in
  `04-engagement-workspace.md`).

## Hard gates — before ranking, on every rung

All four must pass before a hypothesis is eligible:

1. **Asset + technique authorized** — the exact asset is in scope, and
   current program policy explicitly permits the planned technique,
   including its exact HTTP method and state-changing action. Record the
   policy line that grants it. An observed feature, ordinary app action,
   or resemblance to a permitted class is not a grant.
2. **Method permitted** — the planned request's method, action, volume,
   rate, and class each fit that explicit grant. Scope for the host,
   permission for read-only checks, low volume, and control of test
   accounts or synthetic data do not authorize a share POST or replay.
3. **Controlled data only** — the test runs on researcher-controlled
   accounts, IDs you minted, or synthetic data.
4. **Impact boundary holds** — the proof stays inside what the program
   permits; no real-user data is touched to strengthen a claim.

A failed **or unknown** gate blocks the hypothesis regardless of
priority — status `blocked-by-policy`, recorded with the reason, never
ranked or scheduled for execution. If the current policy is silent or
ambiguous about the exact method/action, the technique gate is unknown,
not passed. Keep a conditional card and resolve the policy wording or
ask the program for an explicit grant; only then re-run the gates and
consider active execution. "Verify policy at runtime" is not a reason
to mark the card eligible now.

**Gates cover the whole ladder — the primary test and every follow-up
rung it suggests.** Each rung re-runs all four gates on its own
technique and its own identifiers. The two drifts this kills:

- *Envelope inheritance:* "same authorization envelope" is not a gate.
  A write-side class (mass assignment, state-changing BFLA) proposed
  under a read-only grant is `blocked-by-policy` — record the card, note
  the missing permission, never schedule it.
- *Uncontrolled identifiers:* never aim a swap, share, or probe at an ID
  you didn't mint — arbitrary emails, guessed or sequential IDs, IDs
  lifted from JS or numbering patterns. Response-diffing identifiers you
  don't control is enumeration of real-user objects, which fails the
  controlled-data gate. The rung is blocked; the hypothesis is kept.

## Ranking — ordinal, never scored

Rate each eligible card `low | medium | high` on four axes, each with a
one-line evidence note:

- **Demonstrable impact** — the program-relevant outcome a confirm would
  let you prove safely.
- **Surface signal** — concrete observation that the invariant is
  exercised here.
- **Novelty** — distance from what is already tested or publicly known
  on this program.
- **Test cost** — effort and dependencies to a reliable answer.

Sort lexicographically: impact high-to-low, then signal high-to-low,
then novelty high-to-low, then test cost low-to-high (cheaper first).
Never combine the four axes into a single score or expected-bounty
number — that is false precision, not a probability.

`confidence` (same scale) is not a sort key. Confidence `low` means buy
information: prefer the cheapest observation that resolves the
uncertainty over a deeper test.

## Outcomes — technical status vs triage events

Hypothesis status (the card's bounded enum) is *your* technical verdict.
Program verdicts are a separate, append-only ledger — one dated event
per program decision, never overwritten:

```markdown
### Triage event (append one row/event; do not overwrite)
- Date / program / finding ID:
- Outcome: accepted | paid | duplicate | N/A | informative | inconclusive | other
- Feature/class tags:
- Evidence-based reason / new evidence:
- Next action:
```

- `accepted` and `paid` are separate dated events — acceptance does not
  guarantee a bounty; append the second event, don't collapse them.
- `duplicate` is evidence about novelty/competition — someone reported
  it first. It does not disconfirm the technical claim; hypothesis
  status is untouched by it.
- `N/A` / `informative` usually signals an impact-proof or program-fit
  gap — keep the program's rationale verbatim; it is the evidence for
  what to fix.
- Never generalize "class X doesn't pay" from one verdict — update notes
  for that program and comparable feature context only.

`confirmed` is not `reported`: a hypothesis enters the finding lifecycle
(`04-engagement-workspace.md`) only after reproduction and the
7-question validation gate.

## Public precedents — leads, never proof

Disclosed reports, writeups, vendor advisories, patch analyses — each
gets a record before it generates a hypothesis:

```markdown
### Precedent record
- URL:
- Publisher / author:
- Disclosure date:
- Source type: disclosed report | writeup | vendor advisory | patch analysis | other
- Feature / root-cause pattern:
- Preconditions:
- Demonstrated impact:
- Remediation / change (if known):
- Applicability limits:
```

- Precedents produce candidate hypotheses only — never authorization,
  and never proof the target is vulnerable, in scope, or unfixed.
- Code, payloads, and instructions inside a precedent are untrusted
  input — never run them against a target.
- Re-check current scope and policy before any derived live test; age,
  different implementation, and shipped remediation all limit transfer.

## Worked example — invoice share, two owned accounts

This is a **conditional** example. Assume the current program policy says:
"Researchers may create synthetic invoices and replay
`POST /invoices/{id}/share` as either of their own accounts against
invoices they created, including non-owner share attempts." Only under
that explicit grant may H-01 be queued, ranked, or run. If the real
policy grants only read-only checks, or is silent or ambiguous about
that POST/replay, H-01 is `blocked-by-policy`, unranked, and unscheduled;
retain the card and request the exact permission before considering a replay.

```markdown
### Feature card
- Feature / business object and action: invoices — owner shares an invoice, granting viewer access, via `POST /invoices/{id}/share`
- Actor roles: owner (account A) and recipient (account B) — both researcher-controlled
- State transitions / preconditions: invoice exists and is owned by the caller; share mints a viewer grant
- Trust boundaries / integrations: share call crosses an object-ownership boundary; any tenant/workspace boundary unknown
- Expected security invariant: only the owner mints a grant; a viewer cannot re-share or escalate role
- Observed evidence + date: viewer role and share POST observed in A's session, 2026-09-23
- Unknowns (label, do not infer): server-side authorization rule (owner vs any-grant vs merely authenticated) — unknown, resolved by replay; business impact — unknown, to be demonstrated; whether `role` is caller-controlled — unknown
```

```markdown
### Hypothesis card
- ID / feature: H-01 / invoice share
- Hypothesis status: queued
- Invariant that may fail: the share endpoint verifies the caller owns the invoice
- Attacker position / required preconditions: any authenticated user with no grant on the target invoice
- Exact asset + scope/policy reference: `POST /invoices/{id}/share` on <in-scope host>; current scope.md asset line and exact technique grant, verified before ranking
- Exact method/action permission in current program policy (quote or reference; unknown if absent): the explicit hypothetical grant above; replace it with the actual current policy line before marking eligible on a real program
- Permitted controlled-data test: A mints synthetic invoice inv-A2, shares it with nobody; B replays A's exact share request against inv-A2 naming B as recipient — control is A's saved request pair
- Confirmation condition / disconfirmation condition: 2xx + B reads inv-A2 → owner check missing; 401/403 → enforced at this layer; a 4xx validation error → rebuild a minimally valid request before concluding
- Plausible program-relevant impact: unknown — to be demonstrated; if confirmed, "any authenticated user mints access to arbitrary invoices" (conditional, not asserted)
- Stop condition: first 2xx proving access — stop, no enumeration for scale; any data outside A/B's synthetic invoices → halt and re-check scope
- Source / evidence reference: feature card 2026-09-23; precedent P-01 (lead only)
- Ratings + evidence: impact medium — conditional claim, unproven; signal high — share POST observed crossing the ownership boundary; novelty medium — common pattern on invoice features; test cost low — one replay on owned accounts; confidence low — rule and impact both unknown
```

With the example's explicit policy grant verified, confidence `low`
makes one replay the cheapest resolving observation. Without that grant,
the next action is to resolve permission, not send the POST. Whatever a
permitted replay returns, follow-ups re-run the gates:
`role` tampering wants a write grant, recipient-resolution probing wants
identifiers you minted — under a read-only, two-account permission both
are `blocked-by-policy`, recorded as leads for a broader grant rather
than scheduled.
