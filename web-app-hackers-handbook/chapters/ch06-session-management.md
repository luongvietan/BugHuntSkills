# Ch7: Attacking Session Management

Source: Chapter 7. HTTP is stateless, so the app builds sessions — usually via a token in a cookie. If an attacker can **predict**, **capture**, or **fix** another user's token, authentication is bypassed completely. Two defect families: **token generation** and **token handling**.

## Token generation attacks

- **Meaningful structure**: tokens encoding username/userID/email/role/timestamp — decode base64/hex/encodings, diff tokens across accounts and logins. XOR/predictable transformation of a user identifier = forge other users' tokens.
- **Predictable sequences**: sequential IDs, time-based tokens, weak PRNG seeds, tokens dependent on client IP/User-Agent, per-login derived components. Collect many tokens in series → analyze for: sequential segments, time-dependency (token at T vs T+1), static vs variable segments, patterns at bit level.
- **Weak entropy**: short tokens, low character variety, small effective keyspace — measure how much of the token actually varies. Brute-forcing session tokens can be more feasible than brute-forcing passwords (no lockout semantics, valid/invalid per request).
- **Testing method**: scripted login loops to harvest hundreds/thousands of tokens; Burp Sequencer for statistical randomness; manual diff is often enough — align tokens, mark changing nibbles/bytes, check whether variable portion is sequential, time-based, or random.
- **Session fixation seed**: token issued pre-login and *kept* post-login → attacker fixes victim's token (see ch12 session fixation delivery).

## Token handling attacks

- **Disclosure**: tokens over HTTP (mixed-content sites — check whether `secure` flag is set and actually enforced), in URLs (Referer leaks to third-party assets, server logs), log/error messages, per-user session pages.
- **Cookie scope**: `domain` too broad (shared across subdomains — a less-trusted subdomain can steal it), `path` too broad, missing `secure`/`HttpOnly`. Domain-scope mistakes make XSS on *any* subdomain into session theft for the main app.
- **Termination**: logout that doesn't invalidate server-side (replay old token after logout), timeout policy absent/excessive, concurrent-session handling (does a second login kill the first — for fixation defense, it should).
- **Token→session binding**: token not actually tied to the session state (app accepts token for user A in user B's context), session data stored client-side trusting integrity (see opaque-data tampering, ch04).
- **Hijack post-auth**: check whether token alone suffices or app re-verifies IP/UA — re-verification is a defense to verify, not assume.

## Attack method

1. Harvest token samples (scripted logins) → align and diff → identify structure, variable segments, sequence/timing patterns.
2. Statistical check (Sequencer) where patterns aren't obvious → measure effective entropy.
3. If partially predictable: guess the variable part of a *live* victim token from a few samples.
4. Handling tests: HTTP vs HTTPS exposure, URL-borne tokens, cookie scope, logout invalidation, concurrent sessions, IP/UA binding.
5. Fixation test: does login rotate the token? If not → fixation exploitable.

## Checklist

- [ ] Token format fully decoded; meaning of every segment known.
- [ ] Entropy measured; predictability ruled out or exploited.
- [ ] Tokens never appear in URLs; `secure`/`HttpOnly`/domain/path flags verified.
- [ ] Logout + timeout actually invalidate; concurrent-session behavior noted.
- [ ] Login rotates token (no fixation); session not bindable cross-user.
