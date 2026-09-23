# Cheatsheet — OWASP API Security Top 10 (2019 → 2023 crosswalk): Tester Decision Table

Risk column shows `2019 → 2023` mapping. Quote 2023 IDs in reports.

## Where to look → what to try

| Surface | Try first | Risk (19→23) |
|---|---|---|
| Any `{id}`/`{guid}`/`{name}` in path, query, body, header | Swap ID for another user's → 200+data = BOLA | API1 → API1 |
| Custom headers (`X-User-Id`, `X-Account`) | Modify value → horizontal access | API1 → API1 |
| Login / register / reset / OTP verify / token refresh | Lockout missing, weak password, low-volume spray on own accounts only | API2 → API2 |
| JWT | `alg:none`, weak HMAC secret, no `exp` check, signature ignored | API2 → API2 |
| Any JSON response | Diff fields vs UI → tokens/PII/internal props = exposure | API3 → API3 BOPLA (read) |
| Object mutation endpoints | Add `is_admin`, `role`, `balance`, `verified`, internal params | API6 → API3 BOPLA (write) |
| `size`, `limit`, `per_page`, `count`, `page` | Large values → slowdown/errors/overflow — few requests, measure, stop | API4 → API4 |
| Upload + server-side processing (thumbs, convert) | One oversized file → memory/CPU signal; no repeated load | API4 → API4 |
| Every endpoint | GET→PUT/DELETE/PATCH; `users`→`admins`; `/export_all`, `/new`, `/internal` | API5 → API5 |
| Purchase/book/vote/coupon flows | Low-volume abuse simulation on own accounts — auto-purchase, re-use coupon, flood-by-design | — → API6 (new) |
| URL params, webhook/callback registration, file-import-by-URL | `http://127.0.0.1/`, `http://169.254.169.254/`, collaborator URL → SSRF | — → API7 (new) |
| Endpoints using shell-backed features | `$(cmd)`, `;cmd`, `|cmd`, backticks in params | API8 → dropped 2023* |
| JSON/query params (Mongo-ish) | `[$ne]`, `[$gt]`, `[$regex]`, object/array juggling | API8 → dropped 2023* |
| Subdomains & paths | `beta/staging/dev/test/mbasic/legacy` hosts, `/v1↔v2↔v3` rotation; documented-vs-running drift | API9 → API9 |
| Web root | `.git`, `.env`, `.bash_history`, swagger/openapi files | API7 → API8 |
| CORS | `Origin: evil.com` reflection + `Access-Control-Allow-Credentials` | API7 → API8 |
| Errors | Force 4xx/5xx → stack traces, versions, paths | API7 → API8 |
| Third-party API the target consumes | Trust boundary: unvalidated upstream data stored/reflected, weak TLS to provider, over-scoped provider token | — → API10 (new) |
| Logged fields (UA, names, params) | `%0d%0a`, format strings → log injection | API10 → dropped 2023* |

\* dropped from the 2023 top 10 ≠ not a bug — still test; report under the
class name, not a 2023 ID.

## Severity & report hints

- **BOLA**: object access on other users → usually highest severity; prove with 2 accounts.
- **Reset/OTP without rate limit**: account-takeover potential — top-tier finding.
- **Mass assignment**: severity = property sensitivity (`is_admin`/`balance` >> cosmetic fields).
- **Rate limiting/DoS class**: show mechanism with a handful of requests; never degrade the service.
- **Excessive data**: severity scales with data sensitivity, not volume.
- **Shadow API**: same bug on `v1` that `v2` fixed is still a valid, often worse, finding.
- **Missing monitoring**: secondary finding — pair with the exploit it failed to catch.

## Risk scores (Exploitability/Prevalence/Detectability/Technical)

| Risk | E | P | D | T |
|---|---|---|---|---|
| API1 BOLA | 3 | 3 | 2 | 3 |
| API2 Broken Auth | 3 | 2 | 2 | 3 |
| API3 Excessive Data | 3 | 2 | 2 | 2 |
| API4 Rate Limiting | 2 | 3 | 3 | 2 |
| API5 BFLA | 3 | 2 | 1 | 2 |
| API6 Mass Assignment | 2 | 2 | 2 | 2 |
| API7 Misconfiguration | 3 | 3 | 3 | 2 |
| API8 Injection | 3 | 2 | 3 | 3 |
| API9 Assets Mgmt | 3 | 3 | 2 | 2 |
| API10 Logging/Monitoring | 2 | 3 | 1 | 2 |

## Golden rules

- Client filters nothing — the API response is the truth.
- Two test accounts minimum: horizontal (BOLA) + vertical (BFLA) need proofs.
- Every protection added later is a version-diff to test on old/shadow hosts.
- IDs live in headers and cookies too, not just URLs.
- Auth endpoints need *stricter* limits than normal endpoints — check they got them.
