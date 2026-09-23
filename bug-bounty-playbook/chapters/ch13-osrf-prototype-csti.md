# Ch 13 — OSRF, Prototype Pollution, CSTI

Three underrated classes most hunters never test.

## OSRF — On-site Request Forgery

Like CSRF but the forced request originates **from the target site itself** (carries cookies, Origin, IP trust). Root cause: user input controls part of a URL the app/browser requests (e.g. `<img src>` built from a param).

**Mechanic**: inject `../` into the URL fragment the app embeds → request walks up directories → hit internal endpoints.

```
<img src="/images{INPUT}.jpg">
INPUT = ../../admin/add?username=ghost%26password=lulz%26dummy=
→ browser requests /admin/add?username=ghost&password=lulz&dummy=.jpg
```

- `%26` = `&` URL-encoded so extra params bind to the *inner* request, not the outer URL.
- Trailing junk (`.jpg` appended by the app) → soak it with a **dummy param** (`&dummy=`).
- Confirm: `../` works → path traversal in URL construction = OSRF. Then find interesting endpoints (`/admin/add`, state-changing GETs).

## Prototype pollution (JavaScript)

JS objects inherit via prototypes: modify `Object.prototype` once → every object gains the property.

**Trigger**: merge/extend functions that copy user-controlled JSON into objects. If the merge walks `__proto__`, your property lands on the prototype → inherited app-wide.

```json
{"__proto__":{"admin":true}}
```

→ later code checks `user.admin` → inherited `true` → privesc. Also leads to XSS, DoS, and RCE (gadget-dependent). Treat it as **object injection**: overwrite any property anywhere.

## CSTI — Client-Side Template Injection (AngularJS)

Server encodes `<>`/`htmlspecialchars` → classic XSS blocked… but **Angular expressions need no special chars**:

- `{{1+1}}` → `2` = CSTI confirmed.
- `alert(1)` fails — expressions eval against `$scope`, not window.
- Sandbox escape: `{{constructor.constructor('alert(1)')()}}` — `Function` constructor → arbitrary JS.

Version note: AngularJS 1.2–1.5 has a sandbox → look up the version-specific bypass; 1.6+ removed it (no sandbox at all). Any client template framework accepting user input is in-scope for this class.

## Why these matter

Old/obscure ≠ rare — few hunters test them → low competition. OSRF adds an internal-request primitive; prototype pollution escalates to privesc/RCE; CSTI defeats the "encoding is enough" defense.
