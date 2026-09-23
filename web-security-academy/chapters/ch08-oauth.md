# Ch08: SSO Attacks — OAuth 2.0, OpenID Connect & SAML

Sources: `/web-security/oauth` + SAML synthesis (no dedicated Academy SAML topic exists — `/web-security/saml` is not on the site; SAML coverage here aligns with `bug-bounty-bootcamp` ch17 and standard SAML attack research). OAuth = delegated authorization framework (client app ↔ resource owner ↔ OAuth provider); SAML = XML-based SSO assertion flow (IdP → SP). Both implement "log in with another identity" — the trust bridge (token or signed assertion) is the attack surface. Companion to `ch07-jwt.md` (OIDC tokens are JWTs).

> **Lab vs live (bounty-safe):** OAuth/OIDC/SAML attack chains (redirect_uri
> abuse, code interception, account linking, signature bypass) are victim-
> delivery class. Demonstrate with your own client app and your own two
> accounts; a working token/authorization-code capture against a real user
> is out of bounds without explicit permission.

## Mechanism & recon

- **Grant types**: **authorization code** (server-side code→token exchange, `redirect_uri` + `state` + `client_secret` protected) vs **implicit** (token delivered through the browser in the URL fragment — inherently weaker, mostly legacy but still found).
- **Identify OAuth**: login-with-X buttons, `/oauth/authorize?client_id=...&redirect_uri=...&response_type=...&scope=...&state=...` flows, `code`/`access_token`/`id_token` in redirects.
- **Recon the provider**: fetch `/.well-known/oauth-authorization-server` and `/.well-known/openid-configuration` — endpoints, supported grant types, scopes, JWKS URI, registration endpoint. Map the full flow in Burp before attacking.

## Vuln patterns (authorization-code & implicit)

- **Implicit grant trust** — browser-delivered token/user data; if the client app *trusts the POSTed identity fields without verifying the token server-side*, change `email`/`username` in the final login POST → logged in as anyone. Test by replaying the last step with another user's identifiers.
- **Flawed CSRF protection (`state` missing/abused)** — `state` binds the callback to the user's session. Missing/predictable `state` = login CSRF: complete an OAuth flow with *your* social account, capture the callback URL (`/oauth-callback?code=...`), send it to the victim → they log into *your* linked account context (or their account gets linked to your social profile → ATO via social login).
- **Leaking codes/tokens via `redirect_uri`** — the authorization server validates `redirect_uri` before redirecting the code. Loose validation lets `redirect_uri=https://attacker.tld/callback` (or open-redirect/subdomain-takeover/path-traversal tricks on a whitelisted host: `redirect_uri=https://client.com/../redirect`, suffix/prefix confusion, `client.com.evil.tld`, `evil.com@client.com`) → victim's `code` lands on your server → exchange it for their session.
- **Flawed scope validation** — ask for more scope than the user consented to, reuse tokens across scopes, upgrade `scope` on the token exchange, or use an all-scope default when the parameter is dropped.
- **Unverified user registration** — provider/client trusts self-asserted identity data (email claimed during registration without verification) → register as the victim's email → log in as them.
- **`state`/`nonce` secondary issues**: predictable values, reflected without binding, `state` used as open-redirect vehicle.

## OpenID Connect layer

OIDC adds standardized identity on top: ID token (a JWT), `UserInfo` endpoint, standardized claims (`sub`, `email`), discovery doc, `code`/`token`/`id_token` response types.

- **Unprotected dynamic client registration** — `/register`/`/reg` endpoint accepts arbitrary client metadata: register a client with your `redirect_uri`/`logo_uri`/`client_uri` pointing at attacker infra → leak codes or serve malicious consent pages. SSRF if the provider fetches the URIs.
- **Authorization requests by reference (`request_uri`)** — OIDC lets the client pass a URL pointing to the JWT request object instead of inline params. If the provider fetches arbitrary `request_uri` values → SSRF, or host your own request object with modified claims.

## SAML attacks

Flow: user → SP → redirect to IdP → authenticate → **signed SAML response** (XML assertion) POSTed back to SP → logged in. Test by capturing the `SAMLResponse`, base64-decoding to XML, and attacking the signature/validation layer:

- **Signature not verified / missing** — strip the `<ds:Signature>` block entirely → SP still accepts? Then modify `NameID`/attributes (`user@victim.com`) keeping an invalid signature → accepted = critical.
- **Signature wrapping (XSW)** — duplicate the assertion: keep the original signed copy for the validator while placing your modified copy where the *parser* reads identity (~8 canonical XSW layouts — wrapping inside `Response`, inside `Signature`'s `Object`, sibling assertion swap). Use **SAML Raider** (Burp extension) for XSW generation, resigning, and stripping.
- **Canonicalization / comment injection** — XML canonicalization differences between signature-validator and parser: `adm<!--x-->in@victim.com` verifies as one string, reads as another; duplicate-attribute confusion; namespace tricks.
- **IdP/SP confusion** — assertion minted for SP-A replayed at SP-B (audience not enforced); a token for one service consumed by a sibling; `Recipient`/`AudienceRestriction` ignored.
- **Response replay** — old captured assertions accepted (no `NotOnOrAfter`/`InResponseTo` enforcement); replay a victim's captured response to log in as them.
- **XML-layer attacks** — XXE inside the SAML XML (parser resolves entities → `ch12-ssrf-xxe.md`); XInclude; DTD-based DoS.
- **Tooling**: SAML Raider, SAML-tracer (browser) to watch flows, base64+XML decode in Burp.

## Attack order

1. Map the flow: which grant type, which params sent, where `state`/`nonce` appear, how `redirect_uri` is validated.
2. Cheap wins first: implicit-grant identity tampering → `redirect_uri` manipulation → missing `state` CSRF.
3. Then provider-side: dynamic registration, `request_uri`, scope escalation.
4. Chain: OAuth findings feed `ch03-cache-host.md` (leak via cached redirect) and open-redirect findings from `ch05-dom-attacks.md`/`ch09-cors-csrf-clickjacking.md` feed `redirect_uri` bypasses.

## Lab reference

`https://portswigger.net/web-security/all-labs#oauth-authentication` — OAuth/OIDC labs only; **no SAML labs exist on the Academy** (not a site topic).
