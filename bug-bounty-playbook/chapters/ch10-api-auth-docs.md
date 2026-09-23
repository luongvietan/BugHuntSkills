# Ch 10 — API Authentication + Documentation

Break auth → ATO → privesc. Three schemes + how to find the docs that reveal design flaws.

## HTTP Basic

`Authorization: Basic base64(user:pass)` — cleartext creds on every request (eavesdrop if TLS weak/absent). Identify by the browser's native popup.

## JWT attacks

Structure: `base64(header).base64(payload).signature` — decode at jwt.io or Burp. Header picks the alg; payload holds the claims the server **trusts**; signature is the only tamper check. Forging a signature = impersonate anyone.

1. **Deleted signature**: strip signature → still accepted? → modify claims freely (`"name":"admin"`).
2. **`alg:none`**: set `{"alg":"none"}` + empty signature → accepted? Originally a debug feature. Burp plugin **JSON Web Token Attacker** automates it.
3. **Brute-force HMAC secret**: HS256 = symmetric → weak secret → crack → sign anything. Tools: `jwtcat`, `jwt-cracker`, `jwt-pwn`, `c-jwt-cracker` (search "jwt cracker").
4. **RS256→HS256 confusion**: server verifies with `verify(alg, public_key, token)`; switch header to HS256 → server verifies an HMAC **using the public key as the secret** — and the public key is public → forge tokens signed with it.

## SAML attacks

SSO: login once at the IdP → SAML assertion (XML) carries identity to the SP. Assertion = `Subject` (NameID = username/email) + `Signature` whose `Reference URI` points at the element it protects. Break the signature↔data binding → become anyone.

1. **Signature removal**: blank out `<ds:SignatureValue></ds:SignatureValue>` or delete the signature block entirely (SAML Raider "Remove Signatures") → some SPs accept unsigned → edit `NameID` to victim.
2. **XML comment injection**: parsers strip comments → username `admin<!--x-->@gmail.com` becomes `admin@gmail.com` post-parse.
3. **XML Signature Wrapping (XSW)**: validation happens on one element, processing on another — smuggle a malicious assertion with the same ID. Apply via SAML Raider → `Apply XSW` → resend; no error = vulnerable. Eight variants:
   - **XSW1/2** — attack the *response* signature: original response embedded inside signature (enveloping) or detached signature alongside.
   - **XSW3** — malicious assertion placed *above* original (parser takes first).
   - **XSW4** — original assertion embedded *inside* the malicious one.
   - **XSW5** — copy original signature into evil assertion; signature still references original.
   - **XSW6** — original assertion embedded in original signature, all inside evil assertion.
   - **XSW7** — evil assertion (same ID) inside `<Extensions>` (loose schema element).
   - **XSW8** — evil assertion + original signature; original assertion hidden inside `<Object>` in the signature.

## API documentation — design flaws at scale

*"The vast majority of vulnerabilities I find in APIs are the result of a design flaw"* — docs list every endpoint + params → spot flaws like "password reset takes userId + newPassword" (instant IDOR).

**Swagger** (REST, JSON docs) — hunt paths: `/api`, `/swagger/index.html`, `/swagger/v1/swagger.json`, `/swagger-ui.html`, `/swagger-resources`. Look for design flaws, auth gaps, hidden admin/password-reset endpoints.
- Swagger-UI itself has XSS history: `?url=<script>…` (issue #1262) and persistent XSS via malicious JSON spec `?url=https://attacker/x.json` (#3847).

**Postman** — import Swagger/WADL JSON → test every endpoint systematically.

**WSDL** (SOAP docs) — `example.com/?wsdl`, `*.wsdl` → import into **SoapUI** → templated requests.

**WADL** (REST docs, XML) — `*.wadl` → import into Postman.

## Order of operations

API found → find its docs (swagger/wsdl/wadl paths) → enumerate endpoints → test auth scheme (JWT/SAML attacks) → design flaws (IDOR/mass assignment/hidden functions) → standard OWASP per param.
