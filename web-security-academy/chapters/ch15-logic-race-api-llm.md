# Ch15: Business Logic, Race Conditions, API Testing, GraphQL & LLM Attacks

Sources: `/web-security/logic-flaws` (+`/examples`), `/web-security/race-conditions`, `/web-security/api-testing` (+`/server-side-parameter-pollution`, `/top-10-api-vulnerabilities`), `/web-security/graphql`, `/web-security/llm-attacks`. The "app does exactly what it was told — the spec was wrong" cluster plus the newest API surfaces.

## Business logic vulnerabilities

Flaws in workflow design/rules, not injection — scanners can't find these; you must understand intended behavior then violate assumptions:

- **Trust in client-side controls**: price/quantity/currency tampering in cart/checkout, hidden form fields, role flags in requests.
- **Flawed workflow enforcement**: skip a step (go straight to order-confirm), replay steps, do steps out of order, call the "admin only" function as low-priv user.
- **Integrity check failures**: tamper with values *after* the check — change price in the order-review POST, not the add-to-cart step.
- **Domain-specific rules**: negative quantities, currency-mixing to confuse rounding, oversized values causing integer overflow, loyalty/discount stacking beyond limits, transaction rollback that refunds without reclaiming goods.
- **Anomalous input handling**: deliberately weird values (negative, zero, `NaN`, huge, type-confused) anywhere business rules make assumptions.
- **Method**: map the workflow → identify every rule the app assumes ("user won't send negative", "step 2 only after step 1") → break each assumption once → chain small violations into money/inventory/permission impact.

## Race conditions (TOCTOU)

Check-then-act window: the app reads state (balance, coupon-used, invite-count), verifies, then writes — concurrent requests all pass the check.

- **Limit overrun (single-endpoint)**: parallel-send the same request N× — coupon redeemed multiple times, balance overdrawn, 6 seats on a 5-seat plan. Tools: **Repeater "Send group in parallel"** (HTTP/2 single-packet attack — removes network jitter), Turbo Intruder `race.py`, Intruder null-payload spam (cruder).
- **Multi-endpoint races**: two endpoints on the same object raced (buy+refund, login+password-reset); align the race windows — some need the requests staged so both land inside one open window.
- **Session-based locking**: frameworks lock per-session — race from *two sessions* or two connection pools to dodge it.
- **Partial construction**: object usable mid-creation — race an operation against the record's own creation (register + immediately use).
- **Hidden multi-step sequences**: server-side processing has invisible stages; race into the gap between them.
- **Time-sensitive attacks**: password-reset tokens, expiring codes — collision/single-packet at the moment of validity.
- **Connection warming**: send an innocuous request first to warm the back-end connection, then fire the race group so all requests land simultaneously.
- **Methodology**: predict potential collisions → probe for clues (response deltas on parallel sends) → prove with minimal count needed for impact, then stop (volumetric rules apply).
- **Bounty-safe boundary**: races against *your own* objects (own coupon, own balance, own invites) only. A successful race on a shared resource (event seats, stock, another user's quota) can cause real financial/inventory damage — prove with the smallest parallel count on owned resources and stop; never keep hammering "to be sure."

## API testing

- **Recon**: browse the app with Burp — note every API call (`/api/`, `/v1/`, JSON/XML bodies); check `robots.txt`, `sitemap`, JS bundles for endpoints.
- **Documentation discovery**: `/api`, `/swagger`, `/swagger-ui`, `/openapi.json`, `/api-docs`, `/graphql`, `/redoc` — machine-readable docs = free endpoint/param map; also `/api/index`, `/api/swagger.json` variants.
- **HTTP methods**: `GET`→`POST`/`PUT`/`PATCH`/`DELETE`/`OPTIONS`/`HEAD`/`TRACE`/`CONNECT`/`DEBUG`-class verbs; `OPTIONS` response reveals allowed methods; verb change often bypasses method-scoped authZ.
- **Content types**: `application/json`↔`application/x-www-form-urlencoded`↔`text/xml`↔`multipart` — parser differences produce different vulns (JSON→`$ne` NoSQLi, XML→XXE); hidden parameters honored but never documented (`?debug`, `?admin`, version params).
- **Mass assignment**: POST a property the object has but the form never sends (`isAdmin`, `role`, `verified`, `price`) — find candidate property names by reading GET responses on the same object.
- **Server-side parameter pollution (SSPP)**: server-side component parses query/params differently than the front-end — duplicate params (`a=1&a=2`), truncated query strings (`%23`/`&`-injection into an internal URL the server builds), REST-path injection (`/users/../admin`), structured-data injection (extra JSON fields merged internally). Detect by injecting valid/invalid params and watching for internal-URL errors.
- **Hidden endpoints**: Intruder wordlist on paths; versioned APIs (`/v1/`) with old bugs.

## GraphQL

- **Find endpoints**: `/graphql`, `/graphql/v1`, `/api/graphql`, `/graphiql`, `/playground`, `/v1/graphql`; also `POST` JSON with `{"query":`/`__schema` markers.
- **Introspection**: `__schema{types{name}}`/`__type(name:)` queries map the whole API. If introspection is off: `__suggestions`-style autocomplete (field-name hints on error), "did you mean" errors, or full-universal queries (`query{__typename}` probing field names wordlist-style).
- **Bypass introspection defenses**: GET requests, URL-encoded queries, or `__schema` blocked → try query aliases/wrapping.
- **Aliases for rate-limit bypass**: `mutation{a1:login(...){token} a2:login(...){token}}` — dozens of attempts in ONE request → brute-force at wire speed, no per-request limit.
- **Exploiting unsanitized arguments**: injection inside argument values (SQLi/NoSQLi/OS-cmd reach the same sinks as REST); IDOR on `id:` arguments — GraphQL doesn't fix broken object-level authZ.
- **GraphQL CSRF**: `POST` with `Content-Type: application/json` can't be sent cross-site simply — but `GET /graphql?query=...` or form-encoded posts sometimes work → CSRF-able mutations.

## LLM attacks (web LLM)

Use the **OWASP GenAI LLM Top 10 2026** as the current risk vocabulary;
2025 is historical. The 2026 categories are LLM01 Prompt Injection, LLM02
Sensitive Information Disclosure, LLM03 Excessive Agency, LLM04 Supply Chain,
LLM05 Data and Model Poisoning, LLM06 Unbounded Consumption, LLM07
Misinformation, LLM08 Hidden Context Exposure, LLM09 Vector and Embedding
Weaknesses, and LLM10 Improper Output Handling. Add the **OWASP Top 10 for
Agentic Applications 2026** when the product can plan or act, especially ASI01
Agent Goal Hijack, ASI02 Tool Misuse and Exploitation, ASI03 Identity and
Privilege Abuse, and ASI06 Memory and Context Poisoning. The Academy's Web LLM
labs demonstrate mechanics; these taxonomies help map product boundaries.

### Map the boundary before choosing a test

- Map direct input, retrieved content, tool output, conversation context,
  persistent memory, connected data, and downstream output/action sinks.
- Record the account/tenant principal, the tool's effective identity, each
  allowed action, and which server-side control should authorize it. Do not
  assume model-generated text itself is evidence of a security impact.
- OWASP AI Testing Guide v1 adds selected procedure IDs: APP-01/02 (direct and
  indirect prompt injection), APP-06 (agent behavior limits), APP-08 (embedding
  manipulation), and INF-03/04 (plugin boundary and capability misuse). Use
  these as test methods, not as another risk taxonomy.

### Bounded tests for authorized product surfaces

- **LLM01 / ASI01 prompt or goal hijack**: test one direct prompt or one
  researcher-owned document in a researcher-owned session. Do not plant
  instructions in shared, public, or victim-facing content.
- **LLM02 sensitive disclosure / LLM08 hidden context exposure**: use a
  synthetic marker that you put in your own test context. Never probe for
  another user's messages, private documents, credentials, or live system
  secrets; stop if any appear unexpectedly.
- **LLM03 excessive agency / ASI02 tool misuse / ASI03 identity and privilege**:
  inventory documented or observable tools, then verify authorization with a
  harmless read-only check or a no-op routed to a sink you control. Do not
  send real email, issue refunds/payments, delete records, change privileges,
  or exercise another principal's credentials. Exact action permission is a
  separate gate from permission to use the chat feature.
- **ASI06 memory and context / LLM09 vector and embedding weaknesses**: use
  synthetic documents in two researcher-controlled test tenants, if the
  program provides them. Check only those test identities and IDs; stop on
  any foreign content. Never poison a shared corpus or persistent memory.
- **LLM04 supply chain / LLM05 data and model poisoning**: use in-scope
  inventories and configuration evidence first. Do not upload packages, alter
  training data, or persist a poisoned artifact as a live proof.
- **LLM07 misinformation**: report only a security-relevant invariant failure
  with demonstrated product impact; model inaccuracy alone is not proof of a
  security boundary break.
- **LLM10 improper output handling**: use an inert marker to check a documented
  renderer or parser boundary. Do not use a payload that can execute a real
  command, script, or state change.
- **Resource consumption (LLM06 / API4:2023)**: obey the program's rate and
  cost limits; use at most the smallest permitted single request, never loops,
  load tests, or context stuffing.
- **AI fetch / scanner SSRF**: only test a URL-fetch feature with a
  researcher-controlled canary endpoint. Do not direct it at internal IPs,
  metadata services, or third-party hosts unless the program grants that
  exact destination and method.

### Lab vs live

PortSwigger labs authorize the lab's full exercise. They grant no permission
on a bounty target. Before each live check, re-run the asset, technique,
account/data, rate, and impact gates. Keep synthetic evidence, use the
least-impact observable result, and stop immediately if the test could affect
another user, shared state, or service availability.

## Lab reference

`https://portswigger.net/web-security/all-labs#business-logic-vulnerabilities` · `#race-conditions` · `#api-testing` · `#graphql-api-vulnerabilities` · `#web-llm-attacks`
