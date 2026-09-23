# Patterns — Reusable Heuristics from Bug Bounty Bootcamp

Cross-cutting heuristics that apply across vuln classes. Per-class details live in `chapters/`. Era-marked tooling routes to current skills — see `sources.md`.

## The master loop (every vuln chapter)

```
Mechanism → Prevention → Hunting → Bypass → Escalation → Automation
```

Apply it to any new bug class you meet: understand how it works, learn what the defense is *supposed* to do, find where input reaches it, attack the implementation (not the spec), maximize impact, then script the grind.

## Universal heuristics

- **Every input is a hypothesis**: "this param reaches [SQL/shell/template/redirect/object-lookup]". Test the hypothesis, don't just fuzz strings.
- **Baseline → anomaly**: record the normal response (code, length, time, body shape). A fuzzer only finds deltas; sort results by each axis.
- **Two-account discipline**: for anything touching user objects or privileges, always keep a low-priv and a high-priv account; diff their views of the same resource.
- **The validator is the vuln**: defenses fail in predictable ways — substring checks, validation-only-when-present, presence-not-correctness, decode-then-check (or check-then-decode), allowlist prefix/suffix confusion. Name the check, then pick its canonical bypass.
- **Encoding is a bypass library**: URL, double-URL, hex, HTML entities, unicode confusables, case mixing, comments-as-whitespace, alternate IP forms. When a filter blocks, rotate encodings before giving up.
- **Side channels when blind**: no reflection? timing (sleep), OOB DNS/HTTP (Collaborator), side effects (emails, exports, notifications), error messages. A bug without output is still a bug.
- **Old versions keep old bugs**: `/v1/` endpoints, archived pages, legacy subdomains, old mobile app versions — test the past against present-day fixes.
- **Method & content-type confusion**: same action through GET/POST/PUT/DELETE, form vs JSON vs XML vs text/plain — access controls and parsers often differ per path.
- **Hidden parameters**: parameters the server honors but never sends (`admin`, `role`, `debug`, `user_id`) — infer names from JS, docs, responses, common conventions.
- **Chain accounting**: after each finding write "what does this unlock?" Info leak → targeted attack; redirect → SSRF/OAuth; self-XSS → login CSRF; subdomain → cookies/SSO.
- **Rate limits are per-identity**: endpoint, account, IP, token — test at each level and across privilege tiers.
- **Read responses, not pages**: APIs and backends return more than the UI renders. The raw response is ground truth.

## Workflow patterns

- **Recon → notes → hunting → notes**: recon feeds the endpoint/param inventory; hunting feeds back new leads. Structured notes compound across sessions.
- **Manual first, automate second**: walk the app by hand until you could describe its workflows; only then fuzz. Automation without a model finds nothing.
- **Continuous recon**: cron your recon script, diff outputs — new subdomains/endpoints are the least-picked fruit.
- **Tool literacy**: read your tools' docs and (if small/open) source — Sublist3r, Wfuzz. If you can't explain what the tool does, you can't trust or tune it.
- **Minimal viable PoC**: the smallest action proving impact — `version()` not the users table, `id` not `/etc/shadow`, your own account's data not a stranger's. Stops legal/scope problems and still pays.
- **Throttle + permission** for anything volumetric: fuzzing, rate-limit tests, brute force — assume you're the load on a small target.

## Anti-patterns the book warns about

- Reporting the mechanism, not the impact ("XSS found" vs "XSS → session theft → ATO").
- Fuzzing without an anomaly-sorting step — the results file is the product, not the requests sent.
- Trusting UI restrictions — everything client-enforced is server-testable.
- Scanning before scoping — out-of-scope bugs are worth $0 and risk bans.
- Dumping data to "prove" a bug — evidence beyond minimal PoC hurts you at triage and policy review.
