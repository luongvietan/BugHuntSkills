# Ch 14 — XXE, CSP Bypass, RPO

> **Marker-only rule:** XXE = OOB DNS hit or benign file read (`/etc/hostname`); CSP bypass = a harmless `alert` demonstrating script execution under the policy; billion-laughs/exfil chains are lab material (`payloads-all-the-things` ch06 ceiling).

## XXE — XML External Entity

**Rule: see `<?xml version="1.0"?>` in a request → test XXE immediately.**

XML basics: prolog → root → children. **DTD** (`<!DOCTYPE>`) defines structure; **entities** are variables — `<!ENTITY user "Ghostlulz">` → `&user;`. **External entities** pull from URL/file: `<!ENTITY ext SYSTEM "file:///path">`.

**Classic file-read PoC** — replace a reflected node value with your entity:

```xml
<?xml version="1.0"?>
<!DOCTYPE r [ <!ENTITY xxe SYSTEM "file:///etc/hostname"> ]>  <!-- book classic uses /etc/passwd — on a live program the marker file is /etc/hostname per the callout -->
<stockCheck><productId>&xxe;</productId></stockCheck>
```

Requires the entity to be **returned in the response** (otherwise → blind/OOB channels). File read → config/creds → full compromise.

## CSP bypass

CSP = response header of `directive: sources;` pairs. Read every policy like an ACL.

**Directives**: `default-src` (catch-all) · `script-src` (JS) · `style-src` · `img-src` · `connect-src` (AJAX/WS) · `font-src` · `object-src` · `media-src` · `frame-ancestors` (clickjacking).

**Source values**: `*` any · `'none'` block · `'self'` same-origin · `data:` data-URLs · host list · `https:` scheme · `'unsafe-inline'` inline JS → kills XSS protection · `'unsafe-eval'` eval() · `'sha256-'`/`'nonce-'` hash/nonce gate.

**Bypass patterns:**
1. **Misconfig**: `default-src 'self' *` (wildcard = no policy); `script-src 'unsafe-inline' 'unsafe-eval' … data:` — `<iframe/src="data:text/html,<svg onload=alert(1)>">` sails through.
2. **JSONP**: whitelisted host with a `callback=` endpoint returns attacker JS. `script-src … accounts.google.com` → load `accounts.google.com/o/oauth2/revoke?callback=alert(1337)` as the script src → CSP-sanctioned payload delivery.
3. **CSP injection**: user input reflected inside the CSP header itself → inject a source you control (`?vuln=evil.com` → `script-src evil.com`).

## RPO — Relative Path Overwrite

Path confusion where the **browser resolves relative resource URLs differently** than intended → attacker controls which CSS/JS gets loaded.

- Page loads `<link href="style.css">` (relative) → from `/a/b/` it fetches `/a/b/style.css`; trick routing so the same page renders at `/a/b/c/` → browser fetches `/a/b/c/style.css` → if that path serves attacker-influenced content → injected CSS/JS on a trusted origin.
- Mostly → defacement/CSS injection (low severity); sometimes → XSS or data extraction when combined with reflected input.

## Closing note

These close the book's "More OWASP" set: XXE when XML flows, CSP review on every XSS that "should" work but doesn't, RPO as a niche path-confusion trick. Stack rank your time: XSS/SQLi/IDOR pay daily; these win when everyone else gave up.
