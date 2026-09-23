# API5:2019 — Broken Function Level Authorization (BFLA)

## Core Idea
Role/group-scoped *functions* (not objects) exposed to the wrong caller — admin endpoints callable by regular users, or HTTP-method swaps turning a read into a write. APIs are predictable: `GET`→`PUT`, `/users`→`/admins` guessing works.

**Scores**: Exploitability 3 · Prevalence 2 · Detectability 1 · Technical Impact 2

## Is the API Vulnerable? (Test Questions)

Deep-analyze the authz mechanism while mapping the app's roles/groups/hierarchy:

- Can a regular user call **administrative endpoints**?
- Can a user perform sensitive actions by **changing the HTTP method** (GET→DELETE, GET→PUT)?
- Can group-X users reach group-Y functions by **guessing URL + params** (e.g., `/api/v1/users/export_all`)?
- **Don't infer role from path**: admin functions hide under `api/users` as often as under `api/admins`.

## Example Attack Scenarios

**Scenario 1 — method swap to admin function**: invite-only app calls `GET /api/invites/{guid}` (response leaks role + email). Attacker rewrites to `POST /api/invites/new` with `{"email":"hugo@malicious.com","role":"admin"}` — an admin-console endpoint with no function-level check → self-invites as admin.

**Scenario 2 — predictable admin path**: `GET /api/admin/v1/users/all` returns all user details; attacker who learned the API structure guesses it — no authz check.

## How To Prevent

- Consistent authz module invoked by **all** business functions — **deny by default**, explicit grants per role per function.
- Review endpoints against function-level flaws *with business logic and group hierarchy in mind*.
- Admin controllers inherit from an abstract admin controller enforcing checks; admin functions inside regular controllers get the same checks.

## Anti-patterns

- **Assuming path = privilege**: `/api/users/...` can host admin functions; enumerate methods on every endpoint.
- **Testing only documented endpoints**: function-level flaws live on undocumented/hidden endpoints.
- **One-role testing**: need ≥2 accounts at different privilege levels to prove BFLA.

## Key Takeaways

1. Method-swapping is the cheapest BFLA probe: replay request with PUT/POST/DELETE/PATCH.
2. Mutate path segments: `users`→`admins`, `user`→`admin`, append `/all`, `/export`, `/new`, `/internal`.
3. Compare privilege levels: what low-priv account can do vs. what it *should* do.
4. Detectability is hard (score 1) — high-value manual finding, scanners miss it.
5. Report: unauthorized admin functionality → account creation, data export, privileged ops.

## Connects To

- **API1 (ch01)**: BOLA = object-level twin; test both on every endpoint
- **API9 (ch09)**: undocumented/old API versions are where BFLA hides
- CWE-285; OWASP Forced Browsing, Top10-2013-A7
