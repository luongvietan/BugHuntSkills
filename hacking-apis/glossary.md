# Glossary — Hacking APIs

| Term | Meaning |
|---|---|
| **A-B testing** | Create resources as UserA, attempt access as UserB → BOLA detection. |
| **A-B-A testing** | Same + verify modifications back as UserA → BFLA detection. |
| **Attack surface** | Total set of network-exposed systems from which data can be extracted or entry gained. |
| **Attribution** | How stateless APIs identify who sent requests — IP, tokens, origin headers, metadata (rate/request patterns). The thing you rotate/spoof to evade controls. |
| **BFLA** | Broken Function Level Authorization — unauthorized use of *functionality* (admin actions, other roles' features). |
| **BOLA** | Broken Object Level Authorization — unauthorized access to *resources/objects* owned by others. |
| **Burner account** | Disposable test account/token for probing security-control thresholds without risking main access. |
| **CRUD** | Create-Read-Update-Delete — the operations HTTP verbs map to in REST. |
| **crAPI** | "completely ridiculous API" — the book's intentionally vulnerable lab app. |
| **DVGA** | Damn Vulnerable GraphQL Application — GraphQL practice target. |
| **Excessive data exposure** | API returns more fields than the consumer needs; client-side filtering trusted instead of server-side. |
| **Fuzzing (wide/deep)** | Wide: one payload across all requests (asset mgmt, methods, disclosures). Deep: many payloads across one request's every surface (BOLA, injection, mass assignment). |
| **GHDB** | Google Hacking Database — public repo of dorks that expose sensitive info/systems. |
| **GraphQL** | Single-endpoint query language: `query`=read, `mutation`=write, `subscription`=realtime; errors arrive in 200-response bodies. |
| **Improper asset management** | Retired/dev/legacy/unpatched API versions still reachable (`legacy-api.`, `/v1/`). |
| **Introspection** | GraphQL feature exposing the entire schema (`__schema` query) — full request map if enabled. |
| **JWT** | JSON Web Token — `base64(header).base64(payload).signature`; header declares `alg` (HS256 symmetric / RS256 asymmetric / none). |
| **Mass assignment** | API binds client-supplied JSON to server-side variables without filtering → add `"admin":true`, `"mfa":false`, foreign IDs/orgs. |
| **OSINT** | Open-source intelligence — passive recon from public sources. |
| **Pixi** | OWASP DevSlop vulnerable API w/ Swagger docs — book lab target. |
| **POS passwords** | "Path of small resistance" — guessable-but-policy-compliant passwords for spraying (`Winter2021!`, `Password1!`). |
| **Sequencer** | Burp module measuring token entropy (bit/char-level, FIPS tests) — finds which token positions are predictable. |
| **Side-channel BOLA** | Existence oracle via status codes (404 vs 401/405), response length, or timing (`X-Response-Time`). |
| **Spec (OpenAPI/Swagger, RAML)** | Machine-readable API definition — import into Postman for instant request map. |
| **String terminator** | Metachar that ends server-side string processing (`%00`, `0x00`, `//`, `;`, `%09`…) → bypasses input filters. |
| **XAS** | Cross-API Scripting — XSS delivered through a consumed third-party API (or own API → own webapp). |
| **X-CDN headers** | `X-CDN`, `X-Kong-Proxy-Latency`, `Server: Zenedge` etc. — reveal CDN/WAF presence. |
