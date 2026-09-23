# Patterns & Techniques — OWASP API Security Top 10

## ID Swap (BOLA Probe)
**When to use**: any endpoint taking an object identifier (path, query, body, header, cookie)
**How**: capture request as user A → replay with user B's ID/value → success + foreign data = BOLA. Decrement/increment numeric IDs; try GUIDs of other resources leaked elsewhere.
**Trade-offs**: needs 2 accounts for clean proof; GUIDs slow enumeration but don't prevent it

## Method Swapping (BFLA Probe)
**When to use**: every discovered endpoint, especially under shared paths like `/api/users`
**How**: replay with GET→POST/PUT/PATCH/DELETE; mutate path segments (`users`→`admins`, append `/all`, `/export`, `/new`, `/internal`)
**Trade-offs**: cheap probe, high value; requires mapping app's role model to report correctly

## Response-vs-UI Diff (Excessive Data)
**When to use**: every JSON response, always
**How**: list response fields → compare to rendered UI → unexplained fields (tokens, emails, internal flags, `live_*` secrets) = exposure
**Trade-offs**: manual judgment needed — only report genuinely sensitive extras

## Mass Assignment Write-Back
**When to use**: PUT/POST/PATCH endpoints on object resources
**How**: `GET` the object → catalog properties → resend mutation adding juicy fields (`is_admin`, `role`, `balance`, `verified`, internal params); test nested objects + query params too
**Trade-offs**: may need knowledge of business logic to pick impactful props; watch for chained sinks (conversion params → RCE)

## Version/Host Rotation (Shadow API)
**When to use**: on every API target, before deep testing
**How**: rotate `/v1/↔/v2/↔/v3/`, try unversioned paths; enumerate subdomains `api.`, `beta`, `staging`, `dev`, `test`, `mbasic`, `legacy`; diff behaviors vs production (missing rate limit/authz/WAF)
**Trade-offs**: shadow hosts sometimes out of scope — check program rules

## Rate-Limit Gap Testing → API4:2023 Unrestricted Resource Consumption
**When to use**: auth, OTP, reset, search, export, upload, paging endpoints
**How**: small bursts only; try oversized `size/limit/per_page` values; check per-account vs per-IP keying; verify limit applies on all hosts/versions. 2023 widened the class: response-size amplification, per-request cost, CPU/memory-heavy ops, quota absence all count — not just request rate.
**Trade-offs**: demonstrate mechanism with minimal requests — never actually DoS a live target

## Sensitive Business Flow Abuse — API6:2023 (new)
**When to use**: flows where the *legitimate* function is the weapon — purchase, booking, voting, coupon/invite redemption, comment/review posting, account creation
**How**: identify the flow's business constraint ("one per user", "first come first served", "rate-limited by design") → simulate the abuse pattern **at low volume on your own accounts**: re-use a coupon across your two test accounts, script a handful of automated buys, double-submit a form. The finding is the *absence of a control*, proven small
**Trade-offs**: NEVER run at scale — a real scalping/spam run is a program violation; 5-10 requests proves the missing control

## SSRF via API — API7:2023 (new)
**When to use**: any parameter holding a URL, webhook/callback registration, import-by-URL, PDF/report generators, link-preview unfurlers, file fetchers
**How**: `http://127.0.0.1`, `http://169.254.169.254` (cloud metadata — check `hacking-the-cloud` for the per-provider paths), your own collaborator URL, redirect chains to bypass naive filters. Payload depth: `payloads-all-the-things` SSRF chapter
**Trade-offs**: metadata hits are read-only probes only — never mint creds or pivot without explicit authorization

## Unsafe Consumption of APIs — API10:2023 (new)
**When to use**: features built on third-party APIs the target consumes — payment/status webhooks, partner data feeds, OAuth-provider integrations, "powered by X" widgets
**How**: ask what the target trusts from its providers: unvalidated upstream data stored then reflected (stored XSS via partner API), missing TLS verification to the provider, provider tokens with wider scope than needed, trusting upstream `Content-Type`. Probe by observing how upstream responses are handled, not by attacking the provider
**Trade-offs**: the third-party API itself is out of scope — you test the *target's handling* of it

## NoSQL Operator Injection
**When to use**: JSON APIs backed by Mongo-like stores
**How**: `param[$ne]=x`, `param[$gt]=`, `{"$regex":"^a"}`, type-juggle strings→arrays/objects
**Trade-offs**: syntax varies (query-string arrays vs JSON body); `$where`/`$regex` higher impact but noisier

## Command-Injection Probing
**When to use**: params reaching OS-backed features (restore, convert, import, ping, filename)
**How**: `$(id)`, `` `id` ``, `;id`, `|id`, `&&id` in fields; check firmware/IoT endpoints especially
**Trade-offs**: blind sinks need timing/OOB; destructive commands out of bounds

## Config Recon Pass
**When to use**: start of every API assessment
**How**: probe `.git`, `.env`, `.bash_history`, `/swagger`, `/openapi.json`, `OPTIONS` method inventory, security-header audit, CORS origin reflection, force errors for stack traces
**Trade-offs**: noisy but cheap; findings chain into everything else

## Auth-Flow Coverage Map
**When to use**: before declaring auth tested
**How**: list ALL flows — login, register, refresh, reset, OTP verify, deep-link/one-click, mobile-only paths; apply brute-force/lockout/weak-token tests to each
**Trade-offs**: forgotten flows are where findings live; reset ≠ login protection levels
