# Ch08: OAuth 2.0 & OpenID Connect Vulnerabilities

Source: `/web-security/oauth`. OAuth = delegated authorization framework (client app ↔ resource owner ↔ OAuth provider). Used for SSO login it's both ubiquitous and easy to implement wrong — the Academy calls it "inherently prone to implementation mistakes." Companion to `ch07-jwt.md` (OIDC tokens are JWTs) and `bug-bounty-bootcamp`'s SSO chapter.

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

## Attack order

1. Map the flow: which grant type, which params sent, where `state`/`nonce` appear, how `redirect_uri` is validated.
2. Cheap wins first: implicit-grant identity tampering → `redirect_uri` manipulation → missing `state` CSRF.
3. Then provider-side: dynamic registration, `request_uri`, scope escalation.
4. Chain: OAuth findings feed `ch03-cache-host.md` (leak via cached redirect) and open-redirect findings from `ch05-dom-attacks.md`/`ch09` feed `redirect_uri` bypasses.

## Lab reference

`https://portswigger.net/web-security/all-labs#oauth-authentication`
