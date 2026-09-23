# Ch13: Attacking Users — Other Techniques

Source: Chapter 13. Beyond XSS: attacks that ride the browser's cross-domain plumbing, exploit the trust a user's browser places in the app, or steal data the browser already holds.

## Cross-site request forgery (CSRF)

- **Mechanism**: SOP doesn't stop a page on origin A *issuing* requests to origin B — and the browser attaches B's cookies. If the app's state-changing requests have no unguessable element, an attacker's page replays them as the victim.
- **Requirements to be real**: a privileged action + cookie-only session handling + request params fully predictable (nothing unguessable). POST-only doesn't save you (auto-submit forms; `text/plain`/`form` content types send without preflight).
- **Bypass attempts on anti-CSRF defenses**: token only validated *when present* → drop the param; token checked against a per-session store loosely → supply *your own* token in victim's request (attacker-token substitution); token tied to cookie but cookie user-settable (cookie tossing); Referer/Origin checks that pass when header *absent*; method/content-type swaps (`GET`↔`POST`, JSON→`text/plain`) that reach the same handler through a path without token checks.
- **Authentication CSRF/login-CSRF**: force-login victim into *attacker's* account → victim enters data attacker reads; same flaw patterns apply.
- **One-click / multi-step**: chain a stored-XSS + CSRF where token is needed — XSS reads the token (token ≠ XSS defense).

## UI redress (clickjacking)

- Sensitive action in an iframe under an attacker page; overlay tricks (`opacity`, cursor spoofing) get the victim's real clicks. Defense to verify: `X-Frame-Options`/`frame-ancestors`, frame-busting JS (bypasses: `sandbox`, content-type quirks, onbeforeunload loops, referrer-check gating). Needs a *sensitive action* to matter — map which actions are single-click.

## Cross-domain data capture

- **HTML/CSS injection** (no script needed): inject form/anchor/img to exfil page data to attacker URL; CSS `background:url()` attribute selectors extract tokens/values char-by-char.
- **JavaScript hijacking**: JSON/array responses served to any requester — a cross-site `<script src>` or array-redefinition can read "JSON" data (CSRF for *reads*). Defense: non-executable prefixes, tokens, `Content-Type` enforcement.
- **SOP + extensions**: Flash `crossdomain.xml` (`*` = any site reads app responses), Java/Silverlight plugin policies — each is a separate SOP implementation with its own bypass history. **Proxy-service endpoints** (`/proxy?url=`) intentionally cross domains.
- **DNS rebinding**: attacker DNS answers flip TTL/name→internal IP → same-origin JS reads the intranet/host — defeats hostname-based trust.

## Injection into the client's request plumbing

- **HTTP header injection**: `%0d%0a` in reflected headers (Location, Set-Cookie) → response splitting → arbitrary page/cookie injection → effectively XSS delivered via redirect.
- **Cookie injection**: CRLF into Set-Cookie, subdomain cookie tossing, or XSS-planted cookies → set preferences, fixation tokens, or XSS-triggering cookie values.
- **Open redirects**: `redirect=`, `return=` targets — value in chains (phishing realism, token-theft via redirect fragment, SSRF pivots). Bypass filters: `//evil`, `evil.com@`, `target.evil.com`, absolute-prefix addition (`/\\evil.com`), encoding variants, `javascript:`/`data:` schemes.
- **Client-side SQL injection / HPP**: HTML5 `openDatabase` injection; client-side duplicate-param smuggling into JS routing.

## Local privacy & browser attacks

- Browser-side data the app enabled: autocomplete on sensitive fields, cached sensitive pages (check `Cache-Control`), URL/history leaking tokens, Flash LSO/Silverlight/IE `userData` stores surviving "logout", logging keystrokes/port-scanning LAN via JS delivered through an XSS.
- Native plugins (ActiveX especially): verify origin checks can't be satisfied by an XSS, methods can't be reached with attacker args; old plugin versions carry known bugs.
- MITM context: HTTP sites hand everything to the network — cookies without `secure`, redirect over HTTP to HTTPS (`secure` on the *login* page only still leaks), HSTS gaps.

## Checklist

- [ ] Every state-changing request checked for token+its validation behavior (drop/substitute/method-swap).
- [ ] Sensitive actions framed: XFO/frame-ancestors absent + real consequence → clickjacking.
- [ ] JSON/data endpoints fetched cross-origin (`<script src>`, CORS, crossdomain.xml).
- [ ] Redirect params → open-redirect filter bypass set.
- [ ] CRLF probes in every input that lands in a header.
- [ ] Cookie scope/subdomain paths reviewed for injection + fixation (pre-login token kept post-login).
- [ ] Autocomplete/caching/local stores audited on sensitive data.
