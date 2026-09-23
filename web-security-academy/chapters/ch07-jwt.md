# Ch07: JWT Attacks

Source: `/web-security/jwt`. JWTs are signed (JWS) or encrypted (JWE) JSON tokens — `base64url(header).base64url(payload).signature`. Used for auth/session/access-control, so flaws = impersonation and priv-esc. The spec is deliberately flexible; bugs live in the *implementation's* verification, not the format.

> **Lab vs live (bounty-safe):** alg-confusion, `kid` injection, `jku`/`jwk`
> tricks are tested with tokens issued to *your own* account — forge yours,
> impersonate your second test account, never a real user. Secret cracking
> (weak HMAC) happens offline on your own token.

## Mechanism recap

Header = metadata (`alg`, `typ`, plus attack-relevant `jwk`, `jku`, `kid`); payload = claims (`sub`, `role`, `exp`, `iat`, custom); signature = keyed hash over header+payload. Anyone can read/forge the contents — only the signature protects integrity. **Every JWT attack is a way to get the server to accept a signature you can produce.**

## Flawed signature verification

- **Arbitrary signatures accepted** — server never actually verifies: modify payload claims (`sub: administrator`, `role: admin`), keep any signature → accepted. Baseline test on every JWT: change a claim, send, watch the result.
- **`alg: none`** — header `{"alg":"none"}` with empty signature. Servers that route verification through the declared `alg` accept unsigned tokens. Try `none`, `None`, `NONE`, `nOnE`.

## Weak secrets

- **Brute-force the HMAC secret**: `hashcat -m 16500 -a 0 token.txt wordlist` (JWT mode) — HS256/384/512 tokens crack offline at GPU speed. Rockyou/secLists get most weak secrets; signing a forged token with the recovered key = full impersonation.

## Header parameter injections (self-signed JWTs the server accepts)

- **`jwk` parameter** — header can carry the public key itself. Servers that verify against the *embedded* `jwk` accept a token signed with your own private key. Embed your JWK in the header, sign, send.
- **`jku` parameter** — URL to a JWKS the server fetches for the key. If the host isn't whitelisted, point `jku` at your own server hosting your JWKS → sign with your key. Whitelist bypasses: `trusted.com@evil.com`, `evil.com?x=trusted.com`, path tricks, SSRF on the fetch.
- **`kid` parameter** — key ID, often used as a filesystem path or DB key. If it's not sanitized: `kid` traversal to a predictable file (`/dev/null` — verify with empty string as HMAC key) or a file whose content you know → sign with that content. `kid` SQLi → return a key of your choosing.
- **Other params**: `cty`, `x5u`, `x5c`, `crit` — same injection surface; anything the verifier *fetches or evaluates* is a vector.

## Algorithm confusion (RS256 → HS256)

- **Mechanism**: RS256 = asymmetric (server signs with private key, verifies with public key). HS256 = symmetric (one secret for both). If the server uses RS256 but *also* accepts HS256 tokens and verifies them with the **public key as the HMAC secret** — the public key is public → you can sign.
- **Steps**: obtain the public key (`/jwks.json`, `.well-known`, TLS cert, or **derive it mathematically from two signed tokens** — the RSA modulus can be computed from signatures); convert to the format the library expects (PEM, exact byte form — no extra newlines); change `alg` to `HS256`; sign header.payload with the public key via HMAC; send.

## Working with JWTs in Burp

JWT Editor extension: edit claims in place, re-sign with generated keys, run `jwk`/`jku`/`kid` attacks via built-in dialogs. Repeater flow: decode → modify → sign → send. Burp Scanner (Pro 2022.5.1+) auto-detects several JWT issues — still verify manually.

## Lab reference

`https://portswigger.net/web-security/all-labs#jwt` — labs: arbitrary signature, `alg:none`, hashcat brute-force, `jwk`, `jku`, `kid` traversal, RS256→HS256 confusion, deriving the public key.
