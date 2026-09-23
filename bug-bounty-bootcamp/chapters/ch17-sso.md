# Ch20: Single Sign-On (SAML & OAuth)

Source: Chapter 20. SSO = one identity provider (IdP) authenticates you for many service providers (SPs). The trust bridge — the signed assertion or the token — is the attack surface.

## SAML

Flow: user → SP → redirect to IdP → authenticate → **signed SAML response** POSTed back to SP → logged in. The response XML asserts "this is user@target.com" signed by IdP key.

**Hunting** (capture the SAMLResponse, decode base64 → XML):

1. **Signature missing?** Strip `<ds:Signature>` block → does the SP still accept? 
2. **Signature not validated**: modify `NameID`/attribute (`user@victim.com`), keep invalid signature → accepted?
3. **Signature wrapping (XSW)**: duplicate the assertion — attacker-signed modified copy + original signed copy nested so the validator checks one but the parser uses the other (~8 XSW variants; use SAML Raider Burp ext).
4. **Weak/predictable signature or key leak** (in code repos, metadata endpoints).
5. **XML attacks**: XXE inside SAML (parser processes entities → ch12), comment-splitting `adm<!--x-->in@victim.com`, duplicate-attribute confusion.
6. **No expiry/audience checks**: replay an old captured response; use a token minted for SP-A at SP-B (token confusion across services).

Tools: Burp **SAML Raider** (resign, strip, XSW), base64+XML decoders, SAML-tracer (Firefox) to watch flows.

## OAuth

Flow: `client_id`, `redirect_uri`, `state`, `code` → token. The four classic flaws:

1. **redirect_uri manipulation** — validation on prefix/substring only: `redirect_uri=https://attacker.com`, `…/callback?next=//attacker.com`, `…callback.attacker.com`, `redirect_uri[]=`, open-redirect chaining (Ch7 → the *real* reason open redirects matter: `redirect_uri=…/oauth/callback?redirect=attacker.com` leaks `code` → token exchange → ATO).
2. **Missing/weak `state`** → login CSRF: force victim to link your OAuth account → log into victim's linked session (or vice versa for account linking hijack).
3. **Implicit flow token leak** — `access_token` in URL fragment → Referer leak via 3rd-party resources on the page, browser history, open redirect.
4. **Client secret leak** (mobile apps, JS, public repos) → impersonate the app; or brute-forceable `client_secret`.
5. **Scope escalation**: ask for narrower scope than granted; swap `scope` param, drop it entirely (default = all), or reuse token across APIs.

## First-bug checklist

Map the SSO flow end-to-end → capture response/request → test signature & redirect validation with the bypass families → check `state` → try token replay/confusion → escalate to ATO PoC on your own accounts.
