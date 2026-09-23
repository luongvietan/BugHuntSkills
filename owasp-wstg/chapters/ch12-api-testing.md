# 4.12 API Testing (WSTG-APIT)

Source: WSTG v4.2 §4.12 — currently a single test covering GraphQL. For broader API methodology (REST authz, BOLA, rate limits) pair with the `hacking-apis` and `owasp-api-security-top-10` skills.

## Test list

- **WSTG-APIT-01 — Testing GraphQL.** Objective: probe GraphQL-specific and generic API weaknesses. Procedure:
  - **Introspection** — send `query IntrospectionQuery { __schema { queryType { name } mutationType { name } types { name kind fields { name } } } }` (or use GraphiQL/Playground/Voyager/clairvoyance if introspection is disabled — try alias/fragment tricks); map queries, mutations, types, and interesting fields (auth/token-ish names).
  - **Authorization** — GraphQL enforces nothing by default: call sensitive queries/mutations discovered via introspection as anonymous and low-priv users (e.g., an `auth` query returning tokens; mutations changing data without rights). Missing object-level checks = BOLA.
  - **Injection** — arguments concatenate into backend queries (SQLi via `namePrefix`-style args extracting `CONFIG`/secret tables), XSS via error messages echoing input, and other standard INPV payloads through query arguments.
  - **DoS queries** — build deep nested/recursive queries (object↔object loops, e.g., `dog { veterinary { dogs { … } } }` at depth) plus huge limits (`limit: 1000000`) to exhaust resources.
  - **Batching attacks** — send arrays of queries or same-field aliases (`second: Veterinary(id:"2")`) in one request to brute-force IDs/tokens under rate limits and WAF per-request counting.
  - Interpretation: exposed introspection is informational alone but powers everything else; unauthorized data/mutations, successful injection, resource-exhausting depth, and rate-limit-evading batches are the reportable findings. Remediation reference: timeouts, max query depth, max complexity, complexity-based throttling, restricted introspection.
  - Tools: GraphiQL/Playground, Voyager, clairvoyance, graphql-path-enum, BatchQL, graphql-cop, InQL (Burp), ZAP.

## Common findings

Introspection enabled in production (common); token/secret-returning queries callable unauthenticated; mutation authorization gaps; SQLi through arguments; alias-batching brute force; unbounded depth causing resource exhaustion.

## Escalation notes

Introspection → full schema → targeted BOLA hunting (map to ATHZ-02/04 techniques); injection through arguments → same escalation as INPV-05/01; batching → password-reset/OTP brute force at scale; DoS findings should be reported with rate/depth-limit recommendations rather than demonstrated at volume on production.
