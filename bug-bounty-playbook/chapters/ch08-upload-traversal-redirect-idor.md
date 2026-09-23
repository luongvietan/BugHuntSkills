# Ch 8 — File Upload, Directory Traversal, Open Redirect, IDOR

## File upload → RCE

Any upload feature = test. Goal: upload executable code inside the web root → browse to it → RCE.

**Minimal webshells** (PHP): `<?php if(isset($_REQUEST['cmd'])){echo "<pre>";system($_REQUEST['cmd']);echo "</pre>";die;}?>` — ASPX equivalent exists (cmd.exe via ProcessStartInfo). Then locate the upload path (dir listing, predictable `/uploads/`, image URL after legit upload) → `?cmd=whoami`.

**Bypasses:**
- **Content-Type check**: server trusts the request's `Content-Type` → change `application/x-php` → `image/jpeg` in Burp. Passes.
- **Filename/extension blacklist**: regex misses alternates → `.phpt`, `.phtml`, `.php5`, `.pht`, double extensions `shell.php.jpg`, case `sHeLL.PhP`. One forgotten extension = bypass.
- Also try: null bytes (legacy), `.htaccess`/`.user.ini` upload on Apache, SVG/XML → stored XSS, content sniffing.

## Directory traversal

App uses user input to fetch files (`?page=index.html`) → `../` walks the tree.

- `?page=../../../../etc/passwd` — classic PoC on Linux (`%2e%2e%2f`, `..;/`, `....//` for filters).
- Targets beyond `/etc/passwd`: **config files, source code, credentials**; on upload features, traversal can mean **arbitrary file overwrite** (`../../var/www/x.php`).

## Open redirect

User-controlled data lands in a redirect target (`?url=`, `?next=`, `?redir=`). Test by pointing at `https://google.com` — if it follows, confirmed.

Alone it's low-impact; **value is in chains**: OAuth `redirect_uri` token theft, SSRF pivot, CSP/JSONP tricks, phishing realism. Report it cheap; escalate it smart.

## IDOR — the author's favorite

Object reference (user id/username/email/uuid) in request → server fetches without authz check. Change the id → read/write other users' data or **send commands** as them (change email → account takeover; add admin).

**Workflow:**
1. Watch Burp for requests carrying *your* identifier (`userId`, email, guid) — reads *and* writes.
2. Swap for another user's id (create a second test account → easy victim ids).
3. **"Unguessable" ids**: hash-then-check — `8f14e45fceea167a5a36dedd4bea2543` = `md5("7")`. Hash small integers; if sequential, script the enumeration → mass PII leak.

**Impact framing**: read = PII disclosure; write/command = account takeover / privesc. Easy to find, almost always high severity.
