# Ch10: Attacking Back-End Components

Source: Chapter 10. The app is a broker between user input and back-end components: OS, filesystem, XML/SOAP services, internal HTTP services, mail servers. Each hand-off is an injection boundary — data safe in the app becomes metacharacters in the component.

## OS command injection

- **Mechanism**: input concatenated into a shell command → separators/pipes inject new commands: `; | & && || \n $( ) `` `` `.
- **Confirm blind**: time delay — ping loops, `timeout`, or a slow command; **repeat the delay measurement** to rule out noise (a 5s delay ×3 confirms; once proves nothing).
- **Retrieve output**: redirect to webroot (`; ls > /var/www/x.txt`), TCP/OOB to your listener, DNS exfil.
- **Evasion**: escaped-metachar defense beaten by escaping the escape (`foo\;ls` → `\\` escapes the backslash, `;` executes — see ch10 logic ex.8); `%0a` newlines when spaces are filtered; `${IFS}` for spaces; quoting variants.
- **Where**: ping/IP tools, email/contact functions (often direct `mail`/`sendmail` shell calls — see SMTP below), file ops, PDF/image processing.

## Path traversal

- **Mechanism**: filename param → filesystem path. Canonicalization gap: `../` variants — `..\\`, `..;/` (IIS), `%2e%2e%2f`, double-encode `%252e`, `....//` (survives single-pass `../` strip), absolute paths, `%00` to null-terminate a forced suffix (`file=../../../etc/passwd%00.jpg`).
- **Targets**: config files, source code, logs (→ creds, tokens), `/etc/passwd`, Windows `win.ini`/`boot.ini`, app data dirs.
- **Write traversal** = file upload/download with controllable path → overwrite/place webshell, cron, authorized_keys.

## File inclusion

- **LFI**: `?page=` or template param includes local file → traversal → source disclosure → RCE via log/session/PHP-session-data/proc injection or uploaded file.
- **RFI**: URL param includes remote file → direct code exec; try `http://`, `ftp://`, `php://input`, `data:` wrappers.

## XML & SOAP injection

- XML parser boundary: XXE-lite (`<!ENTITY x SYSTEM "file:///etc/passwd">`), XML well-formedness errors → info leak, attribute/element injection.
- SOAP: back-end web service trusts front-end validation → inject XML elements/parameters the UI never sends; WSDL (if exposed) is a free endpoint+parameter map.

## Server-side HTTP & parameter pollution

- **Server-side redirect/proxy**: app fetches a URL you supply (page-rendering, feed fetch, `url=` params) → reach internal network, loopback admin interfaces, cloud metadata (`169.254.169.254`), bypassing IP controls — proto-SSRF.
- **HTTP parameter injection (HPI)**: your param value is URL-decoded into a back-end request → inject `&` to add/override params the internal service trusts.
- **HTTP parameter pollution (HPP)**: duplicate param names — different components take first/last/all values → bypass front-end checks that inspect one while back-end uses another; encoding-order differences between front-end and back-end parsers.

## SMTP / mail-header injection

- Feedback/contact/notify functions pass your fields into SMTP headers or the SMTP conversation → `%0a`/`%0d%0a` injects `Cc:`/`Bcc:` (spam relay) or full `MAIL FROM`/`RCPT TO`/`DATA` command sequences (arbitrary email as the app). Test every field incl. hidden `To` fields; try both `\n` and `\r\n`. Check these functions for OS command injection too — mail features are commonly implemented as direct command calls and get less security review.

## Checklist

- [ ] Shell metachars in every param reaching OS-ish functions; blind timing confirms repeated.
- [ ] Traversal variants incl. encoding, null-byte suffix, both slash directions.
- [ ] `page=`/`template=`/`include` params: local then remote targets.
- [ ] Every XML/SOAP input: structure injection + WSDL recon.
- [ ] `url=`/feed/proxy params pointed at loopback/metadata/OOB listener.
- [ ] Duplicate params + `&`-injection tested wherever front-end relays to a back-end service.
- [ ] Mail functions: header injection, full SMTP command injection, and OS command probes.
