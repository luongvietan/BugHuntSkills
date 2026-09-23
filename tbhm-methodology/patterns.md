# Patterns — Reusable Heuristics from TBHM

Cross-cutting heuristics distilled from Haddix's methodology. Per-stage details live in `chapters/`; execution runbook lives in `recon-pipeline`.

## The master stage loop

```
Philosophy → Discovery → Mapping → Tactical fuzzing → Priv/Logic/Transport → Mobile → Auxiliary
```

Apply per target: understand the bounty economics, find untested surface, map it smarter, fuzz fast per feature, then spend manual time where scanners can't go (cross-account, logic, transport). Loop back to Discovery whenever scope or features change.

## Universal heuristics

- **Competition shapes target selection**: the flagship app is picked clean. Rank surface by (untouched × sensitive), not (big × famous).
- **Port scans are web recon**: a `-p-` sweep finds the admin panel on :8443 that no directory list would reach.
- **Newness predicts weakness**: acquisitions, redesigns, new features, new app versions — code written since the last audit.
- **Core questions route the fuzz**: "displays to users?" → XSS · "calls stored data?" → SQLi · "touches filesystem/URL?" → LFI/RFI/upload/redirect · "changes state?" → CSRF · "references an object?" → IDOR.
- **Polyglot first, precise second**: one multi-context string per input smoke-tests every context; invest in crafted payloads only where reflection/behavior confirms a landing.
- **Status codes map hidden depth**: `401`/`403` isn't a dead end — recurse wordlists into denied roots; parent-protected ≠ child-protected.
- **History predicts flaws**: prior disclosures (archives, hacktivity, writeups) fingerprint the team's filter style — adjacent params share the bug; patched spots invite bypass.
- **Two personas minimum**: low-priv + admin (plus a second same-priv account for IDOR). Any "yes" in the auth/session battery becomes a cross-account demo.
- **Rotate every identifier**: UIDs, hashes, emails — increment, decrement, negate, substitute. If the client supplies the reference, test it.
- **Token checks fail predictably**: absent, `undefined`, same-length garbage, cross-account, method-swapped, content-type-downgraded — the eight CSRF rotations generalize to any validator.
- **HTTPS is all-or-nothing**: hunt the exceptions — images, analytics beacons, API calls on plain HTTP.
- **Templates for repeats, customs for novelty**: pre-write report skeletons for your top classes; spend writing time on unique bugs where the attack scenario earns severity.

## Workflow patterns

- **Discovery → notes → mapping → fuzz → notes**: every lead class (redirect, file, url, token) greps out of the crawl and into the fuzz queue.
- **Automate the boring edge**: domain enum, screenshots, port scans, dir brute-force are scriptable; logic and cross-account diffs stay manual.
- **Crawl-grep lead mining**: dump crawler output to JSON/text, grep param names by bug class — the cheapest lead generator per host.
- **Diminishing-returns ordering**: the n-minute battery front-loads highest-yield checks (polyglots → cookie lifecycle → reset flow → UID rotation) so any cutoff still leaves a tested target.
- **Dated tools, durable workflow**: Recon-ng/xssed-era tooling is replaced; the *sequence* (discover→map→tactical→manual) and the question battery are what persist.

## Anti-patterns TBHM implies

- Grinding the flagship login page on a `*.corp.com` wildcard scope.
- Fuzzing without a per-feature hypothesis — volume isn't a methodology.
- Reporting mechanism without attack scenario ("XSS exists" vs. "XSS → session theft on the admin panel").
- Discarding non-200 brute-force results instead of recursing into denied paths.
- Deep-diving auth/session for hours — those batteries are explicitly the *quick* wins; run them early and move on.
- Template reports with stale domains — instant invalidation signal.
