# Ch0–2: Testing Approach & API Fundamentals

Source: Chapter 0 (Preparing for API Security Testing), Chapter 1 (How Web Applications Work), Chapter 2 (The Anatomy of Web APIs).

## Choosing the engagement approach (Ch0)

| Approach | What you know | Threat model |
|---|---|---|
| **Black box** | Nothing beyond what's publicly exposed | Opportunistic external attacker. Client discloses no docs, creds, or internals. Bug bounties are "mostly black box" — you get scope, not internals. |
| **Gray box** | Some docs, a low-priv account, maybe architecture notes | Better-informed attacker (insider knowledge leak). Reallocates effort from recon to exploitation. |
| **White box** | Source code, SDK, all docs, all roles' accounts | Malicious insider / full audit. Most thorough — but test the API, not just the supporting controls. |

**Scoping questions (gray/white box):** authentication & authorization model, which privilege levels exist, whether a WAF is in play, whether API docs can be audited, which environments are in scope, and constraints on destructive testing.

Rule of thumb: in white/gray box, *request direct API access* so you're testing the API itself rather than the perimeter controls. Evasive techniques (ch10) are mostly for black box work.

## How web apps work (Ch1) — only what matters for APIs

- HTTP is stateless; REST APIs keep no session state → every request must carry its own auth context (token/key). Anything used for *attribution* (IP, token, headers, request metadata) is both a security-control anchor and an evasion target.
- Status codes are signal, not noise: 2xx success, 3xx redirects, 4xx client errors (401 unauth, 403 forbidden, 404 not found, 405 method not allowed, 429 rate-limited), 5xx server errors. The *difference* between a 404 and a 405/401 on resource paths is a side channel for resource existence.
- Databases: relational (SQL) vs non-relational (NoSQL — MongoDB, CouchDB, etc.). APIs favor NoSQL for scale → NoSQL injection is *more* likely than SQLi in modern API targets, and less understood by defenders.
- Supporting infra worth fingerprinting: web server (Nginx/Apache/Werkzeug), framework (Django/Express/Rails), DB type, CDN/WAF (X-CDN, Server headers).

## Anatomy of web APIs (Ch2)

- **REST**: resource-per-endpoint, HTTP verbs map to CRUD (GET/POST/PUT/PATCH/DELETE), JSON/XML bodies, stateless. Most common target.
- **SOAP**: XML envelopes, stricter contracts (WSDL), legacy enterprise/financial.
- **GraphQL**: single POST endpoint, consumer-declared queries/mutations — behaves more like a database query than REST. See `ch11-graphql.md`.
- **Auth models**: basic auth (base64 user:pass per request — usually registration only), API keys, bearer tokens, JWTs, OAuth. JWT = `header.payload.signature`, all base64 — see `ch05` for attacks.
- **Spec formats**: OpenAPI/Swagger (`"swagger":"2.0"`), RAML, API Blueprint, Postman collections. Finding a spec ≈ finding the full attack map — import straight into Postman.

**Key habit:** docs/specs describe *intended* use. The attack surface is the gap between intended use and enforced behavior — undocumented endpoints, unchecked methods, unfiltered params.
