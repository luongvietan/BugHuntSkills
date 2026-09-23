# Ch6: Attacking Authentication

Source: Chapter 6. Authentication is a chain: login quality + password recovery + remember-me + impersonation + credential handling — and the chain's strength is its weakest link. Attacks split into **design defects** and **implementation defects**; even "secure" designs fail in code.

## Mechanism → attack surface

- **Enumeration oracles**: distinct messages for bad username vs bad password (login, registration "already taken", password recovery "no such user"). Baseline responses for known-good vs known-bad usernames — diff message text, status, length, timing, hidden fields, cookies.
- **Brute force**: rate-limit analysis first — lockout policy, per-IP vs per-account throttling, CAPTCHA after N failures. Note what's *counted*: rotating IPs, rotating usernames with a fixed common password (spraying), or parallel sessions may defeat counters. Failed attempts sometimes still return different behavior at the lockout boundary — an oracle.
- **Fail-open logic**: "verbose failure" modes — some apps treat exceptions/edge inputs (empty password, very long string, SQLi chars, nonexistent user) via a code path that skips credential checks or crashes into an authenticated state. Submit boundary inputs and watch for any *different* error or unexpected success.
- **Multistage login defects**: stage 2+ assumes stage 1 succeeded and re-checks nothing → access later stages directly (forced browsing); stage N stores partial state in the session that can be poisoned (see ch10 logic ex.5); credentials split across stages allow attacking the weakest stage (e.g., memorable-word guessing has low entropy).
- **Password recovery**: predictable/e-mailable tokens, security questions weaker than passwords, recovery flows that skip to password-set, tokens not bound to account/session, Referer leakage of token-bearing URLs.
- **Remember-me / impersonation**: persistent cookies are de facto credentials — check construction (predictable? `username+timestamp`? reversible?), try token from another user, check whether theft via XSS gives full reauth. Impersonation functions often guard only the UI link.
- **Credential handling**: creds in URL (logs, Referer leak), plaintext storage signals (password emailed on recovery = reversible storage), weak change-password flows (no current-password check → pairs with CSRF/XSS to full ATO).

## Attack method

1. Instrument a **baseline**: two test accounts → record full responses to every credential-related request.
2. Username enum across login/register/recover → wordlist run.
3. Password attack: spray common passwords across *many* usernames (safer than hammering one account); respect lockouts — pause before threshold.
4. Multistage: skip stages, reorder, resubmit stage-1 with stage-3 params, test each stage's re-validation.
5. Exercise recovery + remember-me end-to-end; decode every token; test token↔account binding.
6. Post-auth checks: change-password, change-email — try cross-account parameter swaps (your session + victim's username parameter = **username parameter privilege escalation**).

## Defense/evasion notes

- CAPTCHA: often solvable by re-requesting the login without the CAPTCHA field, via a different client/API path, or after session reset — verify it's enforced server-side, not just rendered.
- Lockouts: check enforcement is per-account *and* global; IP rotation may defeat per-IP counters.
- Strong passwords are irrelevant if recovery is weak — always test the whole chain.

## Checklist

- [ ] Enumeration oracle found or ruled out across login/register/recover.
- [ ] Rate limit/lockout characterized; spray feasibility noted.
- [ ] Fail-open/boundary inputs tried (empty, null bytes, SQLi chars, overlong).
- [ ] Every multistage sequence attacked out-of-order.
- [ ] Recovery token entropy + binding tested; remember-me token decoded/replayed.
- [ ] Authenticated-area credential functions tested cross-account.
