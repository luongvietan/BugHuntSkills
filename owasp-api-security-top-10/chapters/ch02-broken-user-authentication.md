# API2:2019 — Broken User Authentication

## Core Idea
Authentication endpoints are exposed to everyone and often misimplemented: missing anti-brute-force controls, weak token handling, wrong-mechanism-for-use-case. Two sub-issues: **lack of protection** on auth endpoints and **misimplementation** of the mechanism itself.

**Scores**: Exploitability 3 · Prevalence 2 · Detectability 2 · Technical Impact 3

## Is the API Vulnerable? (Test Conditions)

Flag the API if it:

- Permits **credential stuffing** (accepts breached user/pass lists as a "password oracle")
- Permits **brute force** on one account — no captcha/lockout
- Allows **weak passwords**
- Sends **auth tokens/passwords in the URL**
- **Doesn't validate token authenticity** — accepts unsigned or weakly signed JWT (`"alg":"none"`), ignores expiration
- Uses **plaintext / weakly-hashed passwords** or weak encryption keys

Treat *forgot/reset password* flows with the same scrutiny as login.

## Example Attack Scenarios

**Scenario 1 — credential stuffing oracle**: no automated-threat protection → app becomes a credential validator for leaked lists.

**Scenario 2 — reset-token brute force**: `POST /api/system/verification-codes` sends a **6-digit SMS token**; no rate limit on `GET /api/system/verification-codes/{smsToken}` → multithreaded script walks all 10⁶ combos in minutes → account takeover. *Short numeric token + no rate limit = certain brute force.*

## How To Prevent

- Map **all** auth flows (mobile, web, deep links, one-click) — forgotten flows are unprotected flows.
- Don't reinvent auth/token/password storage — use standards; **OAuth and API keys are not authentication**.
- Reset/recovery endpoints get the *same* brute-force, rate-limit, lockout treatment as login — actually **stricter** rate limiting than regular endpoints.
- MFA where possible; lockout/captcha; weak-password checks.
- API keys = client app/project identity, never user authentication.

## Anti-patterns

- **Rate limiting only on login**: reset/verify endpoints usually get forgotten — check them first.
- **JWT accepted without signature/expiry validation** — probe `alg:none`, weak secrets, expired tokens.
- **API keys as user auth** — they're project credentials, spoofable.

## Key Takeaways

1. Brute-force surfaces: login, register, reset, OTP/verification-code, token refresh — test each for lockout/captcha/rate limit.
2. JWT checks: `alg:none`, weak HMAC secret, missing `exp` validation, signature not verified.
3. Sensitive data in URL (tokens, passwords) — leaks via logs/proxies.
4. Credential stuffing is a detection problem too — no throttling = valid finding (ties to API4, API10).
5. SMS/OTP short codes need hard rate limits; 6 digits = ~15 min of requests at modest speed.

## Connects To

- **API4 (ch04)**: missing rate limits are the mechanical enabler for these attacks
- **API10 (ch10)**: credential stuffing succeeded in the wild because no alerts fired
- CWE-798; OWASP Authentication Cheat Sheet, Key Management Cheat Sheet
