# API3:2019 — Excessive Data Exposure

## Core Idea
APIs return more data than the UI shows, trusting the client to filter. Sniff the raw response — the "extra" fields (PII, tokens, internal props) are the vulnerability. Scanners can't detect this; human judgment about data sensitivity is required.

**Scores**: Exploitability 3 · Prevalence 2 · Detectability 2 · Technical Impact 2

## Is the API Vulnerable? (Test Conditions)

- API returns sensitive data **by design**, relying on client-side filtering before display.
- Signature smell: generic serializers (`toJSON()`, `to_string()`) dumping whole models — response fields never rendered in UI.
- Method: intercept traffic, compare **response fields vs. what the UI actually displays**. Anything extra = candidate.

## Example Attack Scenarios

**Scenario 1 — generic serializer**: `/api/articles/{id}/comments/{id}` returns comment metadata + author's full User object (PII) because the endpoint uses generic `toJSON()` on the User model. UI shows only the comment.

**Scenario 2 — full-list response**: IoT camera endpoint `/api/sites/111/cameras` returns *all* site cameras (`{"id","live_access_token","building_id"}`) — GUI displays only the guard's authorized subset. **`live_access_token` leakage = direct camera access.**

## How To Prevent

- **Never rely on client-side filtering** — the response is the trust boundary.
- Review every response: is each field legitimate for this consumer? Ask "who is the consumer of the data?"
- Avoid generic serializers; cherry-pick returned properties.
- Classify PII/sensitive data; audit every endpoint returning it.
- Schema-based response validation (including error responses) as enforcement layer.

## Anti-patterns

- **Judging by the UI**: mobile/web apps hide fields — always read raw JSON.
- **Generic `to_json()`/`to_string()` on models** — the root cause in most cases.
- **Filtering after transport**: data already left the trust boundary.

## Key Takeaways

1. Baseline test: proxy the app, diff response JSON against rendered UI fields — delta = exposure.
2. Hunt for tokens, IDs, emails, phones, internal flags, `live_access_token`-style secrets in responses.
3. Severity scales with data sensitivity, not response size.
4. Errors also leak — validate error payloads for internal detail.
5. Related CWE-213 (Intentional Information Exposure).

## Connects To

- **API6 (ch06)**: mass assignment is the write-side twin — excessive exposure leaks the property names worth overwriting
- **API1 (ch01)**: BOLA often *manifests* as excessive data on other users' objects
- **ch08 (API8)**: injection → mass disclosure when no record limits
