# Glossary — OWASP WSTG v4.2

Terms as the Testing Guide uses them. For hunter-flavored definitions see `bug-bounty-bootcamp/glossary.md`.

## Framework & methodology

- **WSTG-XXX-NN** — stable test identifier (e.g., `WSTG-SESS-05` = CSRF test); use in reports for traceability. Merged stubs in v4.2: INFO-09→INFO-08, INPV-03→CONF-06, ERRH-02→ERRH-01.
- **Passive vs active testing** — passive: observe through a proxy, learn logic, map access points (the INFO category). Active: send crafted input against mapped points (all other categories).
- **Black-box / gray-box** — black-box: no internals, only requests/responses. Gray-box: partial internals (config review, some code access, developer interviews) — WSTG marks which technique applies.
- **Test Objectives** — each WSTG test's stated goals; the checklist items a finding must answer.
- **Forced browsing** — requesting unlinked resources directly (guessing paths, skipping flows) rather than following the UI.
- **Input vector** — any place input enters: query/form params, headers, cookies, upload fields, hidden fields, JSON/XML bodies.
- **Oracle (testing)** — any observable signal that answers a yes/no question: status code, response length, timing, error text, side effect.

## Category terms

- **Dork / Google hacking** — search-operator queries (`site:`, `filetype:`, `inurl:`) mining indexes for exposed data (INFO-01).
- **Banner grabbing / fingerprinting** — identifying software via response headers, error pages, header ordering (INFO-02).
- **Metafiles** — `robots.txt`, `sitemap.xml`, `security.txt`, `humans.txt`, `.well-known/*` (INFO-03).
- **Virtual host enumeration** — discovering apps sharing one IP via Host-header/DNS tricks (INFO-04).
- **Source map** — `.map` file linking minified JS to readable source; production maps = de-facto source disclosure (INFO-05).
- **Subdomain takeover** — dangling DNS record (CNAME/A/NS) pointing at claimable/dead service (CONF-10).
- **RIA cross-domain policy** — `crossdomain.xml`/`clientaccesspolicy.xml` granting Flash/Silverlight/Java cross-domain access (CONF-08).
- **RBAC / role definitions** — permission bundles per user type; tested for switchability and granularity (IDNT-01).
- **Account enumeration oracle** — response divergence between valid and invalid usernames (message, code, timing, redirect) (IDNT-04).
- **Alternative channel** — parallel auth surfaces: mobile site/app, partner sites, staging, IVR/call center (ATHN-10).
- **Forced browsing / direct page request** — skipping the login page by requesting deep URLs (ATHN-04).
- **Horizontal vs vertical escalation** — same-privilege vs higher-privilege resource access (ATHZ-02/03).
- **IDOR / BOLA** — user-controlled object reference lacking per-object authorization (ATHZ-04).
- **Session fixation** — token not rotated at login, letting a planted pre-auth session become authenticated (SESS-03).
- **Session puzzling / variable overloading** — same session variable used in two contexts; seed it innocently, spend it as privilege (SESS-08).
- **Session hijacking (WSTG sense)** — replaying cookies obtainable over HTTP because they lack Secure/HSTS integrity (SESS-09).
- **CSRF** — forged state-changing request riding ambient browser auth (cookie, basic auth); defeated by tokens, SameSite, Origin/Referer checks (SESS-05).
- **HPP** — HTTP parameter pollution: duplicate-name params parsed inconsistently across platforms (ASP.NET concat, PHP last-wins, JSP first-wins) (INPV-04).
- **Incubated vulnerability** — payload stored now, detonated later when recalled (watering-hole pattern) (INPV-14).
- **HTTP splitting / smuggling** — CRLF injection fabricating responses vs. Content-Length/Transfer-Encoding parser desync (INPV-15).
- **Format string injection** — `%x/%s/%n` specifiers reaching `printf`-family calls; `%n` writes memory (INPV-13).
- **SSI injection** — `<!--#exec/include/echo-->` directives evaluated by the server (INPV-08).
- **XSS contexts** — HTML body, attribute, JS string, URL, CSS — encoding requirements differ per context (INPV-01/02).
- **Union / boolean / error-based / OOB / time-delay SQLi** — the five exploitation families of INPV-05.
- **HSTS / `Strict-Transport-Security`** — browser directive forcing HTTPS (`max-age`, `includeSubDomains`, `preload`); "full HSTS" = apex + all subdomains (CONF-07, SESS-03/09).
- **Cookie prefixes** — `__Host-` (secure, no domain, path=/) and `__Secure-` (secure) integrity markers (SESS-02/03).
- **Padding oracle** — decryptor leaking padding validity; enables decrypt/forge without the key (CRYP-02).
- **Source → sink** — DOM XSS model: attacker-readable source (`location.hash`) reaching executable sink (`innerHTML`, `eval`) (CLNT-01..06).
- **XSSI** — sensitive data leaked by including authenticated JS/JSONP responses cross-origin via `<script src>` (CLNT-13).
- **CSWSH** — cross-site WebSocket hijacking: server skipping Origin validation on the WS handshake (CLNT-10).
- **GraphQL introspection** — schema self-description query (`__schema`, `__type`); map for authz/injection testing (APIT-01).
- **Query batching** — multiple GraphQL operations per request (array or aliases); brute-force amplifier (APIT-01).

## Scoring & reporting

- **Risk levels** — WSTG suggests Informational/Low/Medium/High/Critical, defined in an appendix; CVSS optional.
- **Limitations section** — report field for out-of-bounds areas, broken functionality, missing access/time (§5).
- **Maker-checker** — dual-approval control for sensitive admin actions (IDNT-01).
