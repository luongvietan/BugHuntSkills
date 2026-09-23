# Ch6: Cross-Site Scripting

Source: Chapter 6. XSS = injecting attacker JS into pages rendered for other users. Three types: **reflected** (payload in request → echoed in response), **stored** (saved server-side → served to victims later; most dangerous), **DOM** (client-side source → sink, server never sees it). Blind XSS = stored payload fires in a panel you can't see (admin dashboards, logs) — use a callback payload (XSS Hunter-style).

## Hunting

1. Find every input that gets rendered: forms, search, profile fields, comments, file names, error messages, URL params, HTTP headers (`User-Agent`, `Referer`).
2. Inject a benign marker (`XSS_TEST_123`) and find *where and how* it lands in the response — that determines the context.
3. Bypass the UI: modify the request in Burp even when the form restricts length/type; test stored fields via API, not just the page.
4. Classify context before picking payload:

| Context | Payload shape |
|---|---|
| HTML body | `<script>alert()</script>`, `<img src=x onerror=alert()>` |
| Inside attribute | `" onmouseover=alert() x="` or `"`><script>…` |
| Inside `<script>` string | `</script><script>alert()</script>` or `';alert();//` |
| URL/`href` | `javascript:alert()` |
| DOM sink | trace source (`location.hash`, `postMessage`) → sink (`innerHTML`, `eval`) |

## Bypassing protections

- **Filter is a blocklist of tags/attrs**: try `<img>`, `<svg>`, `<details>`, event handlers (`onerror`, `onload`, `onfocus`), `javascript:` URIs.
- **Case/whitespace sensitivity**: `<ScRiPt>`, `<img/src=x`, `<svg onload=…>`.
- **Encoding**: URL-encode, double-encode, HTML entities, `String.fromCharCode`, base64 in `data:` URIs.
- **Filter logic flaws**: removing `<script>` once → `<scr<script>ipt>` reconstructs it; keyword check → `java\tscript:` or newline injection; allowlist checks prefix → `javascript&colon;` entities.
- **CSP present**: check for `unsafe-inline`, JSONP endpoints on allowed origins, missing `object-src`/`base-uri`.
- If it only fires on your own account (self-XSS): don't report alone — chain with login CSRF or a stored vector, or demonstrate impact via leaked data.

## Escalation & impact

An `alert()` proves execution; the bounty is in what execution *buys*: read `document.cookie` only when HttpOnly is absent — otherwise perform actions as the victim (CSRF-equivalent, but silent), read page content the victim sees (PII, CSRF tokens → chain to full ATO), keylog forms, or redirect to a look-alike. Combine: XSS on one subdomain + wildcard cookies = compromise of the parent app; stored XSS in admin-viewed field = privileged XSS.

## Automation & first bug

Burp scanner/XSSHunter for stored+blind; dalfox/arf for reflected params; Wfuzz with payload lists over GET/POST params. **First-XSS checklist**: enumerate inputs → marker → identify context → smallest context payload → try bypasses → escalate to session/PII/ATO → document the chain.
