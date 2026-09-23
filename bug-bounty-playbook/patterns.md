# Patterns — Bug Bounty Playbook V2

Decision rules and recurring signals, by chapter.

## Signals → immediate tests

- `Powered by <CMS>` / Wappalyzer hit → run that CMS's dedicated scanner (ch02).
- `*.firebaseio.com` → append `/.json` (ch04).
- Port `9200` → `/_cat/indices?v`; port `27017` → `mongo <ip>`; ports `5984/9042` → CouchDB/Cassandra (ch04).
- `There isn't a Github Pages Site here` (or provider 404 fingerprint) → subdomain takeover via can-i-take-over-xyz (ch03).
- Any login screen → default creds (SecLists) → hydra (ch05).
- POST request → think stored XSS/CSRF; URL param = id/email/username → IDOR; `url=`/`callback=` → SSRF/redirect/SOP; JSON MIME → API (ch05).
- `'`, `"` → SQL error → read the fingerprint (`ORA-`, `psycopg2`, mysql) → dialect playbook (ch06).
- Input reflected → check the context (text node / attribute / encoded) → breakout or event handler (ch07).
- `<?xml version=` in request → XXE entity into a reflected node (ch14).
- `/graphql` style endpoint → introspection `{__schema{types{name,fields{name}}}}` → no-auth data pull (ch09).
- JWT in `Authorization: Bearer` → strip sig → `alg:none` → crack HMAC → RS256→HS256 (ch10).
- SAML response in SSO flow → blank SignatureValue → remove Signature → XML comment → XSW via SAML Raider (ch10).
- Cache headers (`X-Cache`, `Age`, `Vary`, CDN host) → Param Miner unkeyed input → force-cache a param bump → self-XSS → stored (ch11).
- Sensitive per-user page → path-confusion suffixes (`%0A`,`%3B`,`%23`,`.css`) → check if cached → deception (ch11).
- `{{`/`${`/`<%=` style reflection or error → SSTI detect `{{7*7}}` → fingerprint `{{7*'7'}}` → engine RCE chain (ch12).
- Input lands inside a URL the app builds (`img src`, fetch) → `../` → OSRF (ch13).
- JSON merge endpoint → `{"__proto__":{…}}` → prototype pollution (ch13).
- AngularJS app + encoded HTML output → `{{1+1}}` → `{{constructor.constructor('alert(1)')()}}` (ch13).
- `script-src` has `'unsafe-inline'`/`data:`/whitelisted JSONP host → CSP bypass (ch14).

## Meta-patterns (author's heuristics)

- New tech found → fingerprint → Google/ExploitDB/NVD → PoC → local test → fire. Never blind-fire.
- Unknown CMS/service → ExploitDB CVEs → GitHub scanner → else move on.
- "Unguessable" ids → hash small ints (md5/sha1) → probably sequential underneath.
- Low-severity bugs are chain material: self-XSS + cache poisoning = stored; open redirect + OAuth = token theft.
- Hours of manual sifting (GitHub dorks, result triage) is the moat — that's where the crits are.
- Manual first, automate after — understand the exploit before pointing a scanner at it.
