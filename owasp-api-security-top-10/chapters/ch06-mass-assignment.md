# API6:2019 — Mass Assignment

> **2023 status:** merged → **API3:2023 BOPLA** (write side; `ch03` is the read
> side). **API6:2023 is a different risk** — Unrestricted Access to Sensitive
> Business Flows; see the SKILL.md crosswalk for its routing.

## Core Idea
Frameworks auto-bind client JSON to internal object properties. If the endpoint doesn't whitelist fields, attackers set properties they should never touch — privilege flags, balances, internal params. APIs *helpfully* expose property names via their responses.

**Scores**: Exploitability 2 · Prevalence 2 · Detectability 2 · Technical Impact 2

## Is the API Vulnerable? (Test Conditions)

- Endpoint auto-converts client params into internal object props without filtering sensitivity/exposure.
- Property classes to hunt:
  - **Permission-related**: `user.is_admin`, `user.is_vip`, `role`
  - **Process-dependent**: `user.cash`, `credit_balance` — should only be set post-verification
  - **Internal**: `created_time`, `mp4_conversion_params`, server-side config fields
- Discovery paths: guess property names, read **other** endpoints' responses for property names, check API docs, or just add suspicious fields to the payload and watch them stick.

## Example Attack Scenarios

**Scenario 1 — self-service credits**: `PUT /api/v1/users/me` legit body `{"user_name":"inons","age":24}`; `GET /api/v1/users/me` reveals `credit_balance`. Replay PUT with `{"user_name":"attacker","age":60,"credit_balance":99999}` → free credits. *The GET response advertised exactly which property to abuse.*

**Scenario 2 — mass assignment → command injection**: `GET /api/v1/videos/{id}/meta_data` leaks `"mp4_conversion_params":"-v codec h264"` (shell-backed). `POST /api/v1/videos/new` accepts arbitrary props → set `"mp4_conversion_params":"-v codec h264 && format C:/"` → command executes when the video is converted. *Mass assignment is a delivery mechanism for other bug classes.*

## How To Prevent

- Avoid auto-binding functions where possible.
- **Whitelist** updatable properties; use built-in blacklist features for the rest.
- Explicitly define and enforce input schemas on payloads.

## Anti-patterns

- **Testing only UI-offered fields**: the interesting properties never appear in the form — add them yourself.
- **Ignoring read endpoints**: GET responses are the property-name dictionary for the attack.
- **Thinking it's "just" data tampering**: it chains into privesc, payment bypass, even command injection.

## Key Takeaways

1. Read-then-write: `GET` the resource first, note every property, then PUT/POST/PATCH extra ones back.
2. Prime targets: `is_admin`, `role`, `balance`, `price`, `verified`, `status`, internal/config params.
3. Try both JSON bodies and query params; also nested objects.
4. Look for server-side processing params (conversion, import, export) — they escalate to injection.
5. CWE-915 (Improperly Controlled Modification of Dynamically-Determined Object Attributes).

## Connects To

- **API3 (ch03)**: excessive data exposure reveals the sensitive property names
- **API8 (ch08)**: mass-assigned params can become injection sinks
- **API5 (ch05)**: both are authorization-family flaws — one writes object props, other reaches functions
