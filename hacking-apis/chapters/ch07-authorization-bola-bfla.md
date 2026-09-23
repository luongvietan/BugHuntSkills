# Ch10: Exploiting API Authorization — BOLA & BFLA

Source: Chapter 10. AuthN ≠ AuthZ: APIs routinely check *that* you're authenticated but not *what* you're allowed to do.

## Finding BOLA

Not just GET on `/api/v1/user/account/1111`→`1112`. Check every method and every ID location: URL path, request body, headers, composite paths, nested objects, encoded values.

**BOLA test matrix** (all sent with UserA's token):

| Pattern | Valid request | BOLA test |
|---|---|---|
| Predictable ID | `GET /api/v1/account/2222` | `GET /api/v1/account/3333` |
| ID combo | `GET /api/v1/UserA/data/2222` | `GET /api/v1/UserB/data/3333` (token still UserA's) |
| Int ID in body | `POST /account {"Account":2222}` | `{"Account":[3333]}` (array-wrap) |
| Email as ID | `POST /user/account {"email":"UserA@x"}` | `{"email":"UserB@x"}` |
| Group ID | `GET /api/v1/group/CompanyA` | `GET /api/v1/group/CompanyB` |
| Group+user combo | `POST /group/CompanyA {"email":"userA@CompanyA.com"}` | `POST /group/CompanyB {"email":"userB@CompanyB.com"}` |
| Nested object | `POST /checking {"Account":2222}` | `{"Account":{"Account":3333}}` — inner pair may bypass outer validation |
| Multi-object | `POST /checking {"Account":2222}` | `{"Account":2222,"Account":3333,"Account":5555}` |
| Predictable token-as-ID | `POST /account {"data":"DflK1df7jSdfa1acaa"}` | `{"data":"DflK1df7jSdfa2dfaa"}` |

**ID sources:** sequential ints, GUIDs, emails, phone numbers, org IDs/names, encoded payloads. IDs leaked elsewhere (user search, receipt pages, public profiles) feed the fuzz range.

**A-B testing:** create resources as UserA → register UserB → replay UserA's resource requests with UserB's token. Scale is small but proves the class: access one → likely access all at that privilege level. Use 3 accounts to learn the ID-generation pattern.

**Side-channel BOLA:** when direct reads are blocked, existence oracles still leak: 404 (nonexistent) vs 401/405 (exists, unauthorized) → enumerate usernames/IDs/phones. `X-Response-Time` deltas, response-length diffs do the same. Enumerated data feeds brute force and the combo tests above.

## Finding BFLA

Look for *functionality* outside your privilege level: update/delete others' objects, admin actions (user search, account create/delete, token management, logs).

**A-B-A testing:** UserA creates resources → UserB sends GET/PUT/POST/DELETE at them → switch back to UserA and *validate the change* (profile pic gone, field altered). Validation step is what makes it reportable.

**Privilege-ladder BFLA:** if you hold accounts at different levels (user/merchant/admin), replay admin-doc'd actions with the low-priv token. A low-priv `POST /api/admin/find/user` that returns data = privilege escalation. Admin docs tell you exactly which endpoints to try.

**Method discovery:** fuzz the verb (`§GET§`→PUT/DELETE/…) — `400` vs uniform `405` reveals an accepted-but-misformed method worth chasing.

## Safety rails

- **Never fuzz DELETE at scale** — successful deletes are irreversible. BFLA-probe on small ranges or test-account resources only.
- Demonstrate impact without destroying client data; offer to work with the provider for extra test accounts when a vuln implies mass exposure (e.g., brute-forceable phone numbers).
