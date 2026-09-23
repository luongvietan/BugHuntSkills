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

## Rate-Limit Gap Testing
**When to use**: auth, OTP, reset, search, export, upload, paging endpoints
**How**: burst requests; try oversized `size/limit/per_page` values; check per-account vs per-IP keying; verify limit applies on all hosts/versions
**Trade-offs**: demonstrate mechanism with minimal requests — never actually DoS a live target

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
