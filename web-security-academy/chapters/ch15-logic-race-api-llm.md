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

Mapped against the **OWASP Top 10 for LLM Applications** (GenAI Security
Project, 2025 list — `genai.owasp.org/llm-top-10/`). Academy labs cover the
web-visible subset; the fuller risk model:

- **Surface mapping**: LLM features call APIs/plugins/functions — map which
  (ask it, or infer from responses); treat every LLM-callable API as
  publicly accessible. Document the model's reachable action set before
  attacking it.
- **LLM01 Prompt injection — direct**: make the model ignore instructions →
  invoke functions it shouldn't (delete, refund, send email). Escalate
  gradually: get it to reveal its instructions, then to violate one.
- **LLM01 Prompt injection — indirect**: poison a data source the LLM reads
  (web page, email, product description, support ticket) → when the
  *victim's* LLM reads it, the payload executes in their context —
  CSRF-equivalent for AI. Bounty-safe version: poison a document *you* own
  that only your own session/agent will read.
- **LLM02 Sensitive information disclosure**: prompt the model to regurgitate
  sensitive context, PII in its context window, secrets in its system
  prompt, or other users' conversation data. Own-context proof only —
  demonstrate with data you planted or your own account's data.
- **LLM05 Improper output handling**: LLM output rendered as HTML/JS → XSS
  via crafted model responses; output passed to shell/SQL → injection
  through the model. Verify with a harmless marker payload.
- **LLM06 Excessive agency / unsafe tool calls**: the model calls real
  APIs with the user's permissions — chain injection→function call→data
  theft/action. Inventory the tool list first; a model that can *only read*
  is low-impact, one that can *send/delete/pay* is the finding.
- **LLM07 System prompt leakage**: coax out the hidden instructions — often
  contains role definitions, secrets, or business logic worth a report on
  its own; also maps the model's intended limits for the next probe.
- **LLM08 RAG / vector-store isolation**: knowledge-grounded bots retrieve
  per-tenant documents — test whether your queries surface another tenant's
  chunks (embedding-space cross-tenant leak) or whether poisoned documents
  rank-manipulate answers. Own-tenant data only.
- **LLM10 Unbounded consumption / resource risks**: expensive queries,
  recursive tool loops, context-window stuffing — a per-request cost
  amplifier. Demonstrate *one* expensive request; never a load test —
  volumetric rules apply exactly as for API4:2023.
- **AI-scanner SSRF** (newest): AI-powered scanners/proxies fetch URLs —
  prompt-inject the scanner into fetching internal resources
  (routing-based SSRF) or carrying injected payloads onward.
- **Lab vs live**: PortSwigger labs authorize full exploitation chains. On a
  live program, LLM testing stays inside: your own documents/accounts, no
  real-user data, no destructive tool calls, no spam/resource abuse — a
  model doing a real action (email sent, record deleted) needs a test-
  owned endpoint to receive it, and stops there.

## Lab reference

`https://portswigger.net/web-security/all-labs#business-logic-vulnerabilities` · `#race-conditions` · `#api-testing` · `#graphql-api-vulnerabilities` · `#web-llm-attacks`
