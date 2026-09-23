# Ch 1 — Approach: From Zero to Tested Target

The book's process chapters (Background + Getting Started + Tools) distilled: recon in parallel, then manual exploration, then targeted testing, then automation for scale.

## Background you must have

- URL → DNS → IP → TCP:80/443 → HTTP request/response. `dig A domain`, `nc ip 80` for manual checks.
- HTTP methods: GET (read — shouldn't alter data), HEAD, POST (invoke functions), PUT/PATCH (update existing), DELETE, TRACE, CONNECT, OPTIONS.
- Rendering: browser parses HTML/CSS/JS — so JS source is *your* readable code (see ch04, hacktivity case).

## The 10-step process (book's own summary)

1. **Enumerate subdomains** (if `*.site.com` in scope): `knockpy domain.com -w subdomains-top1mil-110000.txt` (SecLists DNS list) + **enumall** (Recon-ng script scraping Google/Bing/Baidu) + manually browse ipv4info.com.
2. **ZAP Forced Browse** on the main site for hidden dirs/files (Burp content discovery equivalent). Run in background.
3. **Map the tech stack** with Wappalyzer while proxying everything through Burp (new project per program).
4. **Explore functionality** — note features matching vuln classes (the mapping habit).
5. **Test each mapped area** — manual first.
6. **Automate the enum output**: subdomains → **EyeWitness** (screenshot ports 80/443/8080/8443 — spot takeovers, admin panels, CI servers); IPs → `nmap -sSV -oA out -T4 -iL ips.csv` (service versions; this is how Andy Gill found PornHub's memcache :60893).
7. **Mobile apps** — proxy phone traffic through Burp; SSL pinning needs workarounds (MobSF, JD-GUI for static analysis).
8. **API layer** — read the developer docs, test OAuth scopes, look for free-account-bypass functionality, submit payloads via API when the web UI strips them.
9. **GitRob** — crawl target's public GitHub repos + contributors for configs/keys.
10. **Pay for functionality** — paid features get less attention; if price is reasonable, subscribe and test.

## Mapping rules while exploring

- Rails sites → `/CONTENT_TYPE/RECORD_ID` URL convention → try record IDs you shouldn't access + try `.json` suffix on record URLs (`/reports/12345.json` — inadvertent API exposure).
- AngularJS → inject `{{4*4}}[[5*5]]` (both syntaxes — don't know which they'll render).
- SPA/JS-rendered pages → search JS source for `POST`/`DELETE` paths — hidden endpoints (the hacktivity voting case).
- CSRF tokens in Rails → embedded in `<meta>` headers; test token handling across two accounts.
- Note where files are served from: S3? third-party JS? Each third-party service = new attack vector.

## Universal probes while mapping

- `<img src="x" onerror=alert(1)>` in every field — `x` fails to load → `onerror` fires.
- Watch what the response does: encoded chars? stripped attributes? Then adapt.

## While testing, keep eyes on

- State-changing requests missing/ignoring CSRF tokens.
- ID parameters (two accounts = known-good + victim set).
- XML upload fields (bulk importers) → XXE.
- URLs with record IDs (IDOR, HPP) or redirect params.
- Params echoed in responses (CRLF, XSS, redirect).
- Server version banners (PHP/Apache/Nginx) → unpatched CVEs.

## Anti-patterns the book warns about

- Don't pick Uber/Shopify/Twitter as your first target — veterans sweep them daily.
- Don't jump straight to payloads — understand the app first ("try not to start hacking right away").
- Don't stop at scanners — the author works mostly manually; scanners supplement, not replace.
