# API1:2019 — Broken Object Level Authorization (BOLA / IDOR)

## Core Idea
The #1 most common and impactful API flaw: the server trusts client-supplied object IDs instead of verifying the logged-in user may act on that object. Manipulating any object ID in a request = potential unauthorized access.

**Scores**: Exploitability 3 · Prevalence 3 · Detectability 2 · Technical Impact 3

## Is the API Vulnerable? (Test Conditions)

- Any endpoint that receives an **object ID** and performs an action on it must enforce object-level authorization at the code level.
- Vulnerable pattern: server is stateless, relies on client-sent parameters (IDs in path/query/body/**headers**) to decide which object to access.
- Test every endpoint taking `{id}`, `{name}`, `{guid}`, or custom headers like `X-User-Id` — swap values horizontally (other user) and vertically (privileged object).
- Authorization checks exist but devs forgot to invoke them on some functions — access-control gaps are *not* amenable to automated scanning; manual probing required.

## Example Attack Scenarios (PoC Patterns)

**Scenario 1 — predictable resource path**: e-commerce revenue charts backed by `/shops/{shopName}/revenue_data.json`. Enumerate all shop names via another endpoint → script iterates `{shopName}` → harvest sales data of thousands of stores.

**Scenario 2 — header-based object ID**: wearable PATCH request carries `X-User-Id: 54796`. Decrement to `54795` → successful response → modify other users' account data. *Object IDs travel in headers, not just URL/body.*

## How To Prevent (for report remediation notes)

- Authorization mechanism checking the logged-in user's permission on the *record* in **every** function using client input to reach a DB record.
- Prefer random unpredictable IDs (GUIDs) — defense-in-depth, not a fix alone.
- Write authorization tests; don't deploy changes that break them.

## Anti-patterns

- **GUID = "safe"**: unpredictable IDs reduce enumeration but don't replace authorization checks.
- **Checking only the UI**: hidden buttons aren't authorization; the API call itself is the attack surface.
- **IDs only in URL**: also probe body params, JSON fields, cookies, custom headers.

## Key Takeaways

1. BOLA is the most exploited API risk — test every object-bearing endpoint.
2. IDs appear in path, query, body, headers, cookies — swap them all.
3. Two-account methodology: capture user-A request → replay with user-B's ID → 200 + data = BOLA.
4. Even "proper" authz infrastructure fails when a dev forgets to call it on one function.
5. Report impact: data disclosure, modification, destruction → often full account takeover.

## Connects To

- **API5 (ch05)**: function-level authz is the sibling — BOLA = object access, BFLA = function access
- **API9 (ch09)**: old API versions frequently miss BOLA checks entirely
- CWE-284, CWE-285, CWE-639 (Authorization Bypass Through User-Controlled Key)
