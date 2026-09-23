# Ch3: API Insecurities — Taxonomy → Test Mapping

Source: Chapter 3 (API Insecurities). Maps each weakness class to what it *looks like* during testing. See companion skill `owasp-api-security-top-10` for the full risk framework; this chapter is the attacker's field guide.

> **Taxonomy note (2026 refresh):** the book predates the OWASP API Top 10 **2023**
> edition. When mapping findings to a current framework, use the 2023 names —
> BOLA (API1), Broken Authentication (API2), **BOPLA** (API3 = excessive data +
> mass assignment merged), Unrestricted Resource Consumption (API4), BFLA
> (API5), and the three new ones: Sensitive Business Flows (API6), SSRF (API7),
> Unsafe API Consumption (API10). The crosswalk lives in
> `owasp-api-security-top-10/SKILL.md`.

## Information disclosure (fuel for everything else)

Sources: API responses themselves, public repos (GitHub), search engines, social media, company websites, public API directories (ProgrammableWeb, RapidAPI, apis.guru).

- **Verbose errors**: stack traces, "User does not exist" vs "Invalid password" → username enumeration. Debug pages (e.g., Django debug mode) leak framework + all endpoints.
- **Headers**: `X-Powered-By`, `Server`, `X-Response-Time` — tech fingerprint + resource-existence side channel.
- **Status-code differentials**: 404 for nonexistent vs 401/405 for existing-but-unauthorized → enumerate usernames, account IDs, phone numbers without ever reading data.

## Excessive data exposure (API3)

Provider sends more than the consumer needs; relies on the *client* to filter. UI looks clean — the raw API response carries extra fields (other users' PII, MFA flags, activation status, admin booleans).

**Test:** use every read endpoint as intended, diff response fields vs what the UI displays. Extra fields usable for attack = vuln. Postman Collection Runner makes this repeatable across a whole collection.

## Lack of resources & rate limiting (API4)

No throttling → DoS, cost amplification, brute-force enabler.

**Test flow:** (1) does a limit exist? — check docs, `x-rate-limit*` headers, marketing pages; (2) trigger it — look for 429 / `Retry-After` / silent ban; (3) bypass it — different params, different clients, different IPs (see `ch10`). A limit that exists but is so lax it doesn't constrain an attack (e.g., 15,000 req/min vs a 150,000-entry wordlist — throttle Wfuzz with `-s` and stay inside it) is still a reportable weakness.

## BOLA vs BFLA — the critical distinction

- **BOLA** (Broken Object Level Authorization): unauthorized access to *resources* — read/modify objects owned by others. Predictable IDs ≠ vulnerable; it's only BOLA when the request actually returns/alters someone else's object.
- **BFLA** (Broken Function Level Authorization): unauthorized use of *functionality* — admin actions (delete users, search all users, manage tokens), lateral or vertical privilege use.

Both = authenticated-but-not-authorized. APIs often check *that* you're logged in, not *what* you're allowed to touch. Details: `ch07`.

## Injection (API8)

Indicators: verbose errors, database errors, unexpected status codes, unexpected behavior after metacharacters. Deliver via keys, tokens, headers, query strings, body params. SQL, NoSQL, OS command, XSS/XAS — see `ch09`.

## Improper asset management (API9)

Retired/dev/legacy/unpatched API versions still live: `legacy-api.example.com`, `/v1/`, `/test/`, `/internal/`. Find via subdomain enum (Amass), version fuzzing on known paths, changelogs, Wayback Machine. Old versions usually have fewer controls.

## Security misconfiguration (API7)

Permissive CORS (`Access-Control-Allow-Origin: *` + credentials), missing security headers, debug mode, unnecessary HTTP methods, default creds, unpatched stack. Detected via Nikto/ZAP scans + manual header review.

## Business logic

Docs + intended-use testing reveal the rules; logic flaws break them: coupon abuse, transfer-amount manipulation, wildcard queries (`/api/v1/find?email=*@gmail.com` — USPS), feature designed without abuse case. These survive scanners — automated tools can't model intent.
