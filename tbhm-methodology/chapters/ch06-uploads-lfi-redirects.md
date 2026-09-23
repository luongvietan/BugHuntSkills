# Ch6: Tactical Fuzzing — File Inclusion, Uploads & Redirects

> **Ceiling note:** LFI = benign file marker; uploads = harmless test file you own; redirects = own-domain destination — no shells, no credential reads, no victim chains (`payloads-all-the-things` ch07/09/12).

Source: `07_File_Upload`. Three related classes around one core idea: **can the app interact with the server filesystem, or take a file/URL as input?**

## Local file inclusion (LFI)

Core question: does the page load or reference files? Tools: Liffy (automated LFI testing) and the SecLists LFI list (`JHADDIX_LFI.txt`). Probe file-ish params (see table below) with traversal + target files (`/etc/passwd`, `php://filter` for source reads, log/session paths for inclusion-to-RCE chains).

## Malicious file upload

Upload functions need many protections at once — any gap is a bug. Attack paths:

- **Unexpected executable formats** — swf, html, php, php3, aspx, etc. → web shell or stored XSS (images too, when rendered/parsed same-origin).
- **Parser attacks** — payloads in metadata/headers that crash or exploit the parser (DoS, or XSS when the parser's output renders).
- **File polyglots** — files valid as two formats at once (GIF+JS, PDF+HTML) to bypass content-type/extension checks and store active content on the target origin (research lineage: dan_crowley's "File in the hole!" talk, Ange Albertini's corkami work).

Bypass techniques to rotate: **content-type spoofing** (send `image/png` header with active payload), **extension trickery** (double extensions `.php.jpg`, case `pHp`, null-byte/alternate data streams era tricks, extension-list confusion), filename-reflection XSS (the stored filename is itself an input field — see ch05 vectors).

Upload only test files you own, use harmless PoCs (a `.html` that runs `alert(document.domain)`, a `.php` that prints `id`/`phpinfo`), and clean up afterwards.

## RFI & open redirects — one param family, two bugs

Any parameter holding a web address can be a redirect **and** a remote-include candidate. Common injection params:

| Bug class | High-value params |
|---|---|
| Redirect | `dest=`, `continue=`, `redirect=`, `url=` (anything containing "url"), `uri=`, `window=`, `next=` |
| RFI/include | `file=`, `document=`, `folder=`, `root=`, `path=`, `pg=`, `style=`, `pdf=`, `template=`, `php_path=`, `doc=` |

### Blacklist bypass ladder (rotations to try when a filter blocks)

- Escape slashes: `/` → `\/`, `//` → `\/\/`
- Single slash instead of double
- Strip the scheme: `continue=//example.com`
- Weird separators: `/\/\`, `|/`, `/%09/` (tab-encoded slash)
- Encoded slashes (URL/double-URL)
- Traversal mutations: `./` → `..//`; `../` → `....//`; `/` → `//`

Escalation note: an open redirect is a chain link — feed it to OAuth/SSO flows (token theft), SSRF allowlists (302-pivot), and phishing-resistant program scopes that still count redirects as impact.
