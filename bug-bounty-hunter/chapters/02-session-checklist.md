# 02-session-checklist.md — session init to wrap

Copy-paste checklist for one hunting session. Boxes are ordered; a gate you
cannot check means go back a phase, not forward on a hunch. Replace
`<target>` with the program's short name everywhere.

Suggested layout (created once per target):

```
hunt/<target>/
  scope.md          # policy URL + revision/date, inclusions, exclusions,
                    # banned techniques, safe-harbor notes, stop conditions
  allowlist.txt     # machine-readable: one in-scope host/pattern per line —
                    # the ONLY source active stages may draw targets from
  notes.md          # running log: feature + hypothesis cards, request
                    # pairs, anomaly list, triage events
                    # (formats: 05-hypothesis-engine.md)
  evidence/         # screenshots, saved requests, PoC files per finding
recon/<target>/<YYYYMMDD>/   # per-run pipeline output (see recon-pipeline)
```

URL-intake sessions keep the same contract files under the engagement
workspace `engagements/<program-or-alias>/<YYYY-MM-DD>/` instead —
layout and `engagement.md` fields in `04-engagement-workspace.md`.

## Program-URL intake — when invoked with a Bugcrowd/HackerOne URL

Session start runs through `06-authenticated-program-intake.md` instead of
manual policy collection. The contract boxes below still apply — intake
fills them, it does not replace them.

- [ ] **Program URL parsed** — platform + program slug identified; active
      phase stated; workspace `engagements/<program-or-alias>/<YYYY-MM-DD>/`
      created under the active project root (root rule:
      `04-engagement-workspace.md`)
- [ ] **Browser surface resolved** — connected `mcp__cua_repl.js`
      preferred, else any connected read-only browser surface; browser
      access state (`available` | `signed-in` | `blocked`) recorded in
      `engagement.md`. `blocked` stops only the browser-dependent step —
      hand the user the smallest action (connect a surface or open the
      signed-in page); never a credential-bearing HTTP/API fallback
- [ ] **Authenticated page review complete** — only the program's
      allowlisted researcher-facing pages (brief, scope/rewards, rules,
      visible known issues, changelog, report instructions); each material
      page recorded with source URL, retrieval timestamp (UTC), and the
      displayed last-updated/revision indicator
- [ ] **Scope contract filled from the exact page wording** — `scope.md` +
      `allowlist.txt` written from what the pages show; an unreadable or
      ambiguous section is a recorded gap with the affected scope field
      marked `UNKNOWN` (uppercase — the scope-contract gap marker;
      card-field gaps below stay lowercase `unknown`), never inferred from
      adjacent wording
- [ ] **Resume policy freshness** — on resume, re-open the live program
      pages before any active work and compare the displayed
      update/revision indicator with the saved snapshot; a material change
      re-gates the affected asset/technique/method checks
      (`blocked-by-policy` until re-verified). The ~7-day re-read below
      applies either way

## Session init (phase 1 gate) — the scope contract

All boxes required before any target traffic. An unchecked box means the
phase does not start.

- [ ] **Policy pinned**: URL, revision or date read recorded in `scope.md`
      (policies change silently — re-read on any resume after ~7 days, and
      treat a changed policy as a new contract)
- [ ] **In-scope assets enumerated exactly** in `scope.md` + `allowlist.txt` —
      including wildcard interpretation (`*.target.com` covers apex?
      sub-subdomains? which TLDs? on-prem vs SaaS?)
- [ ] **Exclusions written** — CDN/shared ranges, third-party SaaS, acquired
      domains, "anything not listed is out" clauses
- [ ] **Banned/permissioned techniques written** — DoS/volume tests, automated
      scanner limits, brute force, social engineering, physical, spam —
      plus which need explicit written permission first
- [ ] **Safe-harbor clause located**; written authorization confirmed for any
      active stages planned this session
- [ ] **Rate/volume limits noted** from policy or platform defaults
- [ ] **Report channel noted** (platform form vs. email vs. VDP contact) and
      disclosure terms read (can you ever write it up?)
- [ ] **Stop/contact conditions written** — who to contact (program triage,
      platform support) and when (scope ambiguity, service impact, unexpected
      real-user data, found credential of unclear ownership)
- [ ] **Severity/payout table reviewed** so hunting effort aims at what pays
- [ ] **Two test accounts registered** (different privilege levels); test data
      seeded; credentials in a **password manager/secret store** — notes and
      evidence carry account *aliases* only, never secrets, never real-user
      anything

## Scope pressure scenarios — pass before hunting

Mental-firewall checks; each must produce the halt/redact behavior, not a
workaround:

- [ ] **No scope file** (`scope.md`/`allowlist.txt` missing or empty) →
      active target list is EMPTY; recon proceeds passive-only or not at all.
      Never "everything discovered is in scope."
- [ ] **Discovered-but-unlisted host** (new subdomain in passive output) →
      it stays a lead in `new-since-last-run.txt` until ownership + allowlist
      membership is confirmed; it is not fed to active stages automatically.
- [ ] **Shared/CDN address** (asset resolves to Cloudflare/Fastly/Akamai or a
      third-party SaaS IP) → no port scan/dir-brute against that IP; hostname-
      scoped HTTP probing only if the hostname itself is allowlisted.
- [ ] **Found credential** (key/token/secret in public code or a response) →
      do NOT validate it against any service; record it redacted (type +
      location + first few chars masked) and report via policy channel.
      Validation requires explicit written permission AND confirmed program
      ownership of the credential.

## Per-phase gates

### Recon (phase 2)

- [ ] Initial pass: dated run dir `recon/<target>/<YYYYMMDD>/` created;
      scope files snapshotted; passive stages complete before Phase 3
- [ ] Every tool wrote a file; artifacts normalized (`sort -u`, lowercase);
      `new-since-last-run.txt` diffed vs prior run (or recorded as baseline)
- [ ] ACTIVE GATE before any Phase 2 target traffic: written authorization
      confirmed AND each active-stage target re-checked against `scope.md`
      exclusions — including assets discovered this run. The initial pass
      stays passive; use the program-listed application URL for Phase 3's
      separately authorized functional walk-through without a broad active
      pre-mapping scan
- [ ] Feature-led re-entry only: a Phase 3 feature card or trust-boundary
      question names the recon need; candidate targets/methods are rechecked
      against the allowlist and policy before target traffic; passive stages
      complete before any active stage starts
- [ ] Re-entry findings remain leads until ownership and allowlist checks
      confirm them; add only confirmed in-scope hosts to Burp scope

### Application mapping (phase 3)

- [ ] Every in-scope host walked once per privilege level
- [ ] Program-listed live hosts staged in Burp scope; tech fingerprints
      observed during the authorized walk-through annotated with candidate
      vuln classes. New discoveries remain leads for a feature-led Phase 2
      re-entry and scope re-check.
- [ ] Endpoint + parameter + role inventory in `notes.md`
- [ ] Both accounts' request logs captured for later A-to-B replay
- [ ] Candidate vuln classes listed per function (from the map, not memory)
- [ ] Feature card per meaningful feature in `notes.md`
      (`05-hypothesis-engine.md`) — evidence dated, every gap labeled
      `unknown`; no card, no hypotheses

### Vuln hunting (phase 4)

- [ ] Hypothesis cards written off the feature cards
      (`05-hypothesis-engine.md`) — one falsifiable claim each; "test
      IDOR" is a class label, not a hypothesis
- [ ] HARD GATE: all four ch05 gates pass **before** ordering — a failed
      OR unknown gate blocks execution regardless of priority
      (`blocked-by-policy`, recorded with the reason)
- [ ] Ordering ordinal (impact -> signal -> novelty -> cost, never a
      combined score); the selected hypothesis runs via its vuln-class
      chapter — ch05 prioritizes, the class skills execute
- [ ] Public precedents filed as leads only — never authorization, never
      proof of a live bug; scope/policy re-checked before any derived
      test
- [ ] No safe controlled-data proof → stop and clarify with the program;
      real-user data is never the fallback
- [ ] Classes worked one at a time with the routed chapter open
      (`03-vuln-class-index.md` for the per-class chapter)
- [ ] Banned-technique list re-checked before rate/volume/fuzz-heavy tests
- [ ] Every candidate bug has a confirming request/response pair in
      `evidence/`; card status updated (`confirmed` / `disconfirmed` /
      `inconclusive`)
- [ ] Anomaly list written — unexplained weirdness is phase-5 fuel
- [ ] Test accounts only; zero real-user data touched

### Escalation/chaining (phase 5)

- [ ] Per bug: "attacker gains X" answered with demonstrated steps, not
      adjectives
- [ ] Chain hypotheses tested on owned accounts only
- [ ] PoC stopped at minimal demonstration — no pivoting beyond the claim
- [ ] Severity hypothesis per bug, scored against the program's table
- [ ] Demonstrated impact + verdict written back to the hypothesis card;
      each escalation rung re-ran the four gates on its own technique and
      identifiers

### Reporting (phase 6)

- [ ] Candidate passed the 7-question validation gate
      (`04-engagement-workspace.md`) — outcome `continue`; anything else
      stops or marks that candidate, not the engagement
- [ ] One report per bug (a tested chain = one report describing the chain)
- [ ] Repro steps verified cold — fresh session, named test accounts, <5 min
- [ ] Title names the impact; evidence attached inline at the proving step
- [ ] Scope re-verified for the exact asset + technique in the report
- [ ] Submitted via the program's channel; thread link logged in `notes.md`
- [ ] Each program verdict appends a dated triage event
      (`05-hypothesis-engine.md`) — append-only: `duplicate` is novelty
      evidence, not disconfirmation; `accepted` then `paid` = two events

### Continuous monitoring (phase 7)

- [ ] Pipeline scheduled (cron/Task Scheduler) per `recon-pipeline`
      `chapters/03-monitoring.md`
- [ ] Alert fires on diff output, not on run success
- [ ] Program scope-update feed / changelog subscribed
- [ ] Scope-drift rule on file: every new-since-last-run asset gets the
      phase-2 ownership + exclusion re-check before active follow-up
- [ ] Outcome review folded into the next queue for this program +
      comparable feature context only — sparse verdicts never build a
      cross-program model

## End of session

- [ ] All confirmed bugs drafted into reports **today** — context decays
      overnight; repro steps written cold never get easier
- [ ] `notes.md` closed out: open hypotheses with card statuses current,
      anomalies not yet explained, defenses observed (they seed next
      session's bypass work)
- [ ] Triage ledger current — every verdict received appended as a dated
      event; lessons scoped to this program + comparable features
- [ ] `new-since-last-run.txt` + asset changes logged for the next recon diff
- [ ] Findings reviewed for lessons: which asset type / vuln class paid —
      feeds phase 1 of the next session
- [ ] Evidence folder matches report attachments; nothing sensitive beyond
      your own test-account material retained
