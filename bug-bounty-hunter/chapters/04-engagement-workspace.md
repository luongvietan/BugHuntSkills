# Ch4 — Engagement Workspace, Finding Lifecycle & Validation Gate

Optional but recommended structure for anything longer than a single
session. Concepts adapted from the Claude-BugHunter engagement scaffold
(see `../sources.md` for attribution); wording is original to this skill.

## Engagement workspace

```text
engagements/<program-or-alias>/<YYYY-MM-DD>/
  engagement.md          # program link, scope snapshot date, allowed methods, stop conditions
                         # + intake/session metadata (fields below)
  scope.md               # the scope contract — see chapters/02-session-checklist.md
  allowlist.txt          # machine-readable in-scope hosts/patterns — the ONLY
                         # source active stages may draw targets from
  recon/                 # recon-pipeline dated runs + verified/leads split
    verified-assets.md
    unverified-leads.md  # discovered != authorized
  findings/
    F-001.md
  evidence/              # restricted local storage; share sanitized copies only
  submissions.md         # platform IDs, dates, state, next action — no creds/PII
  notes.md               # feature map, hypothesis queue, precedents, outcomes, dead ends
```

The layout is a default, not a requirement — `recon-pipeline`'s dated
`recon/<target>/<YYYYMMDD>/` output can live inside `recon/` as-is. The
non-negotiable properties are the separations: scope contract, verified
assets, unverified leads, finding records, evidence, submission state.

Create the workspace under the active user project root — the directory the
session's work belongs to, not the skills repository. If the current
directory is the skills repository or is not a clear writable project, use
the task's active project root instead. If no project root is available,
STOP before creating any file and ask the user for a destination.

### `engagement.md` — intake and session metadata

The durable record of where the engagement stands; the intake chapter
(`06-authenticated-program-intake.md`) writes here as it runs, and resume
reads it first. Fields:

- **Platform URL** and **program URL** — the intake input (e.g. the
  Bugcrowd/HackerOne program URL from the invocation).
- **Last read time (UTC)** and the program page's displayed
  last-updated/revision indicator verbatim — the freshness anchor the
  resume policy check compares against.
- **Relevant source links** — brief, scope/rewards, rules, report
  instructions; every material claim links to its source.
- **Browser access state** — `available`, `signed-in`, or `blocked`.
- **Current phase** — the seven-phase position the session is parked at.
- **Completion summary** — what finished this session.
- **Blocker list** — per blocker: what is known, what remains unknown, why
  the workflow stopped, and the smallest user action that unblocks it.

Metadata only — `engagement.md` never stores credentials, cookies, tokens,
or unrelated account content; the browser access state label is the only
session fact it records.

### `scope.md` minimum fields

```markdown
# Program scope snapshot
- Program/platform URL:
- Snapshot captured (UTC):
- Rules/policy URL + revision/date shown:
- In-scope assets + qualifying conditions:
- Explicitly out-of-scope assets/features:
- Allowed testing methods:
- Prohibited methods + rate limits:
- Accepted impact categories / exclusions:
- Safe-harbor / disclosure requirements:
- Test-account labels (never credentials):
- Stop conditions / emergency contact:

## Asset verification
| Asset | Evidence it is in scope | Verified at (UTC) | Status |
```

Ambiguous wording or a policy change pauses testing of the affected
asset/method until resolved. A dated snapshot is evidence of what was
read — never a standing authorization; re-check on schedule.

### Scratchpad sections (`notes.md`)

Optional sections that pay off on multi-session targets:

- **feature map** — one card per meaningful feature; evidence dated,
  gaps labeled `unknown` (format: `05-hypothesis-engine.md`).
- **hypothesis queue** — hypothesis records in priority order with
  current status; record template below, method in
  `05-hypothesis-engine.md`.
- **public precedents** — writeup/advisory/disclosure references that
  generated leads: URL, publisher, disclosure date, applicability
  limits. Leads, never proof.
- **outcome events** — append-only triage ledger: dated verdicts
  (`accepted`, `paid`, `duplicate`, `N/A`, `informative`, …) with the
  evidence-based reason.
- **leads**, **dead ends** (so you don't retest them), **tool/version
  notes** (what ran, when, against what).

These sections carry references and masked identifiers only — no
credentials, secrets, or personal data in any field; raw captures stay
in `evidence/`.

### Hypothesis record — copy into `notes.md`, one per falsifiable claim

Tracking record, not the analytic card — the card format, hard gates,
and ranking rubric live in `05-hypothesis-engine.md`. `confirmed` here
is a technical verdict only: it enters the finding lifecycle + gate
below before drafting, like any other candidate.

```markdown
# H-### | [Feature] | [invariant that may fail]

- Hypothesis status: queued | testing | confirmed | disconfirmed |
  inconclusive | blocked-by-policy | deferred
- Triage events — append-only, one dated line per program verdict,
  never overwrite:
  - [YYYY-MM-DD UTC] accepted | paid | duplicate | N/A | informative |
    inconclusive | other — evidence-based reason; next action
- Hypothesis card ref / linked feature card:
- Exact asset + scope/policy reference:
- Test/proof references: sanitized request-pair or evidence-file IDs —
  raw material stays in `evidence/`; no credentials/secrets/PII here
- Source provenance: target observation | prior engagement outcome |
  public precedent (URL + disclosure date under precedents)
- Applicability limits: what transfers to this target and what doesn't
- Stop condition:
```

## Finding lifecycle

Every candidate carries a status that moves one direction:

```text
lead → validated → drafted → submitted → triaged → paid | closed
```

Track per finding: status, next action, platform report ID, timestamps.
Status tracking lives in `findings/F-###.md` + `submissions.md` —
separate from raw PoC material. `lead` means "interesting signal, not yet
confirmed"; never report from `lead`.

### Finding record (`findings/F-###.md`)

```markdown
# [Asset] | [Bug class] | [Demonstrated impact]

- Status: lead | validated | drafted | submitted | triaged | paid | closed
- First observed (UTC):
- Last reproduced (UTC):
- Program/platform:
- Asset + in-scope evidence:
- Policy/taxonomy reference:
- Severity rationale: current platform/program method; vector+version if used
- Preconditions / attacker role:
- Test accounts: researcher-controlled labels only — no credentials
- Duplicate/known-issue check:

## Summary
Root cause + demonstrated impact, no speculative chains.

## Reproduction
1. Preconditions and account roles.
2. Exact minimal request/action — placeholders for secrets.
3. Expected result.
4. Actual result.

## Impact demonstrated
Only what was safely observed/reproduced; name synthetic or
researcher-owned data used.

## Evidence index
| File | What it proves | Sanitized/checked | Third-party data? |

## Suggested remediation

## Submission / triage history
- Report ID: / Submitted (UTC): / Status: / Next action + due:
```

## Candidate validation gate — before drafting any report

Answer each with evidence; outcomes: `continue`, `stop candidate`,
`needs scope clarification`, `inconclusive`. Stopping a candidate never
stops unrelated authorized work.

1. **Reproduction** — exact request/response, account role, preconditions
   replayed end-to-end?
2. **Program fit** — current policy accepts this class + demonstrated
   impact?
3. **Authorization** — asset explicitly in scope; technique permitted;
   any third-party service/tenant involved?
4. **Preconditions** — access, interaction, configuration, role an
   attacker needs?
5. **Alternative explanation** — intended behavior, documented feature,
   known issue, or test artifact? For auth/access-control candidates,
   retry with a minimally valid request so malformed input isn't
   mistaken for a bypass.
6. **Impact proof** — demonstrated with researcher-controlled accounts or
   synthetic data at the least intrusive proof the program permits; no
   real-user data accessed just to strengthen the report.
7. **Submission readiness** — evidence sufficient, sanitized,
   non-duplicative per available program info, platform-compatible.

A `validated` finding that fails the gate drops to `inconclusive` or
`stop candidate` — record the reason in the finding; it does not block
other work.

## Evidence hygiene — before storing or attaching

- Prefer the minimal request/response or screenshot that proves the claim.
- Redact session cookies, `Authorization`, CSRF/session tokens, API keys,
  secrets in URLs/bodies — then *inspect the sanitized artifact*, don't
  trust the redaction command.
- Incidental third-party data: stop further access, minimize exposure,
  redact from shared evidence, follow program handling rules.
- Check screenshots at full resolution — URL bars, side panels, terminal
  history, hidden headers.
- Raw captures stay in restricted local storage, out of git/backups where
  practical; share sanitized copies through approved channels only.
- Retain/delete per program instructions; rotate any test credential that
  was exposed.

## Submission tracker (`submissions.md`)

```markdown
| Finding | Platform report ID | Submitted (UTC) | Status | Last update | Next action |
```

Administrative metadata only — never report payloads, credentials, or
personal data in the tracker.
