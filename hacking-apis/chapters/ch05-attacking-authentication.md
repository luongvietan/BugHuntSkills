# Ch8: Attacking API Authentication

Source: Chapter 8. Goal: go from no access → unauthorized access → other users' resources → privileged access. Classic flaws (bad passwords, default creds, verbose errors, weak reset flows) plus API-specific ones (no rate limit on auth, one token for everything, low-entropy tokens, JWT misconfig).

## Classic attacks

**Brute force** — capture the real auth request, replicate body format in Wfuzz:

```
wfuzz -d '{"email":"a@email.com","password":"FUZZ"}' --hc 405 \
  -H 'Content-Type: application/json' -z file,rockyou.txt \
  http://target:8888/api/v2/auth
```

`--hc` hides the *known failure* code (405 here); 2xx/3xx or anomalous length = success. Missing `Content-Type: application/json` → expect 415s. Generate targeted lists from excessive-data-exposure leaks + CUPP/Mentalist.

**Password-reset / OTP / MFA brute force** — capture the reset request (`{"email":…,"otp":"1234","password":"New…"}`), mark OTP as payload position, Burp Intruder **brute forcer** payload: charset 0–9, min=max=4 (or whatever the OTP format is — probe your own account's reset to learn it). No rate limit = 4-digit OTP falls fast.

**Password spraying** — defeat lockout policies: many users × few passwords (keep attempts < lockout threshold). List = obvious patterns (`Password1!`, `Season+Year+Symbol` — `Winter2021!`, `QWER!@#$`) + org-targeted guesses (`Twitter@2022`, founder/product names). Intruder **cluster bomb**: position on username (or user part before `@domain`) + position on password; payload set 1 = collected usernames, set 2 = short password list. Sort results by status/length for the anomaly.

**Base64-encoded creds** — encoding ≠ security. Intruder payload processing → `Base64-encode` rule auto-encodes each candidate.

## Token forgery — Sequencer entropy analysis

Collect tokens → analyze randomness → brute-force only the weak positions.

- **Manual load**: Sequencer → Manual Load → paste ≥100 tokens → Analyze Now. Report shows entropy quality; **Character-Level Analysis → Character Set** reveals which positions vary. Example from the book: token `Ab4dt0k3nXX#` — first 9 chars static, last 3 = letter[a-d] + letter[a-d] + digit → only ~160 combos to brute.
- **Live capture**: intercept token-issuing request → Send to Sequencer → Live Capture → Custom location (highlight the token in response) → start. Collects up to 20,000 tokens — also a ready-made pool of *valid identities* for evading attribution if old tokens aren't invalidated.
- **Brute-force weak positions**: Intruder cluster bomb, 3 positions, brute-forcer payloads with the observed charsets; or Wfuzz `-z list,a-b-c-d -z list,a-b-c-d -z range,0-9` against `FUZZFUZ2ZFUZ3Z` in the token header. Then map each valid token's privileges by running the Postman collection with the captured token swapped into `{{hapi_token}}`.

## JWT abuse

Recognize: 3 base64 parts (`eyJ…`.`eyJ…`.sig). Decode with Burp Decoder or `jwt_tool <token>` (also `-t URL -rc "Header: token" -M pb` for a playbook scan).

1. **Replay**: pass a captured/leaked JWT as your own — often still valid.
2. **Unsigned signature strip**: delete signature, keep trailing period (`header.payload.`) — some APIs accept it.
3. **alg:none**: decode header → `"alg":"none"` → re-encode, remove signature. `jwt_tool <JWT> -X a` generates several none-variant tokens. If accepted → edit payload claims freely (`"username":"root"`, `"superadmin":true`).
4. **Algorithm switch (RS256→HS256)**: if provider accepts multiple algorithms, switch asymmetric RS256 → symmetric HS256, then sign with the provider's *public key* (findable: TLS cert, JWKS endpoint, GitHub, docs) as if it were the HMAC secret. If the verifier doesn't pin the alg, the forged token validates.
5. **Weak secret crack**: HS256 secret = password; crack with jwt_tool/hashcat + wordlists, then forge anything.

Always mine the decoded payload: `sub`, `userID`, `name`, `is_admin`, `iat/exp/nbf` — tells you what to forge.
