# Ch09: CORS, CSRF & Clickjacking — Cross-Origin Client Attacks

Sources: `/web-security/cors`, `/web-security/csrf`, `/web-security/clickjacking`. Three ways a malicious origin abuses a victim's browser trust in the target. CORS is often rated a "no book depth" gap class; CSRF/clickjacking overlap `bug-bounty-bootcamp` but the Academy's SameSite-bypass catalog is newer.

## CORS — mechanism & misconfigs

SOP lets any site *send* cross-origin requests but not *read* responses. CORS relaxes that via response headers: `Access-Control-Allow-Origin` (who may read), `Access-Control-Allow-Credentials` (cookies allowed). Misconfigurations:

- **Reflected origin** — server copies the request `Origin` into `ACAO` + `Allow-Credentials: true` → any site reads authenticated responses. Probe: `Origin: https://attacker.example` reflected verbatim → exploit with XHR `withCredentials=true` and read the response.
- **Origin whitelist parsing errors** — prefix match (`https://target.com.evil.tld`), suffix match (`https://eviltarget.com`), unescaped regex dots, subdomain-trust abuse via any XSS on a trusted subdomain.
- **`null` origin whitelisted** — sandboxed iframes, `data:`/`file:` contexts, redirects, local files produce `Origin: null`. If `null` is allowed + credentials → deliver via `<iframe sandbox="allow-scripts">` (yields `Origin: null`).
- **XSS via CORS trust relationships** — target trusts a subdomain you can XSS/subdomain-take → your script on the trusted origin gets CORS access to the main site.
- **Breaking TLS with CORS** — HTTPS site trusts an HTTP origin in `ACAO` → MITM/active network attacker serves a page on the HTTP origin, reads HTTPS responses.
- **Intranet CORS without credentials** — internal app allows external origins but auth is IP-based, not cookie-based: victim's browser on the LAN loads your page → you read intranet content through their browser (no credentials needed).
- **Preflight note**: non-simple requests trigger `OPTIONS` preflight; `Access-Control-Allow-Methods/Headers` must also permit the method — test actual exploitability, not just reflected `ACAO`.
- **CORS ≠ CSRF defense**: CORS governs *reading*; a state-changing POST still lands (simple requests aren't preflighted). Never report "CORS misconfig prevents CSRF."

## CSRF — mechanism & bypass catalog

State-changing request sent cross-site, riding ambient cookies. Build: find a sensitive action → check token/SameSite/Referer defenses → deliver auto-submitting PoC (`<form>` + `onload submit`, or `fetch` for simple requests).

- **Token-validation bypasses**: remove the token param entirely; token only validated *when present*; supply your own token for the victim session (token not user-bound); token tied to session via a separate non-HttpOnly cookie — set the cookie AND the param (double-submit weakness); token checked only on POST — switch to GET; token in cookie + param must match but the cookie-setting endpoint lets you plant it.
- **SameSite bypasses**: `Lax` allows top-level GET — find state-changing GETs or force method override (`?_method=POST`, framework overrides); Chrome's 2-minute `Lax`-by-default window on fresh `SameSite=None`-less cookies — pop a window then submit; `SameSite=Strict` bypass via client-side redirect gadget (on-site redirect you control → GET request *within* the site sends Strict cookies); subdomain sibling attacks (SameSite ≠ same-origin — sibling subdomains are same-site).
- **Referer/Origin-based defense bypasses**: validation only when header present → strip it (`<meta name="referrer" content="never">` / `Referrer-Policy: no-referrer`); whitelist checks `target.com` substring → `target.com.evil.tld`, `evil.com/target.com`, `eviltarget.com`; regex anchored badly.
- **Delivery**: auto-submit PoC page; if the action is GET → `<img>`/redirect; combine with XSS for SameSite-strict scenarios (same-origin script doesn't need CSRF at all).

## Clickjacking

Frame a sensitive page in a transparent/positional iframe → victim clicks attacker-aligned UI → real action executes on the target.

- **Basic**: absolute-positioned iframe, `opacity:0`, decoy button aligned over the real control. **Prefilled form**: use GET-params/`?email=` to populate the form so a single click completes the action.
- **Frame-busting scripts** bypasses: `sandbox` attributes (`allow-forms allow-scripts` but NOT `allow-top-navigation`), `X-Frame-Options` ignored by some browsers in certain contexts, IE8's `Security="restricted"` attribute, double-framing to defeat `top!=self` checks.
- **Multistep clickjacking**: chain frames/positions for "Are you sure?" flows.
- **Clickjacking + DOM XSS**: align the frame so a click triggers a DOM-XSS payload entry → turns low-sev framing into script execution.
- **Defenses to check**: `frame-ancestors` CSP directive (modern), `X-Frame-Options: DENY/SAMEORIGIN` (legacy), SameSite cookies making framed actions unauthenticated anyway.
- **Impact rule**: clickjacking is only as good as the action it hijacks — account settings/email change = reportable; like buttons = informational.

## Lab reference

`https://portswigger.net/web-security/all-labs#cross-origin-resource-sharing-cors` · `#cross-site-request-forgery-csrf` · `#clickjacking`
