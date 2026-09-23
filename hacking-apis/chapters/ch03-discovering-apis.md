# Ch6: Discovering APIs

Source: Chapter 6. Goal: locate APIs, validate they're live, and harvest creds/version/docs/business-purpose intel — *before* attacking. Any exposed key/PII found during recon is a reportable finding on its own.

## Passive recon (OSINT, no target interaction)

Three phases: **cast a wide net → adapt & focus → document the attack surface.**

- **Google dorks** — `site:`/`inurl:`/`intitle:`/`filetype:`. Start broad, narrow with target domain. GHDB favorites:
  - `inurl:"/wp-json/wp/v2/users"` — WP API user dirs
  - `intitle:"index of" api_key OR "api key" OR apiKey` — exposed keys
  - `ext:php inurl:"api.php?action="`, `intitle:"index.of" intext:"api.txt"`
- **API directories** — ProgrammableWeb, RapidAPI, apis.guru: endpoints, versions, auth model, docs links, SDKs (download & review source), changelog (past vulns, old versions).
- **Shodan** — `hostname:"target.com" "content-type: application/json"`, `"wp-json"`, `"200 OK"`. Finds APIs w/o naming conventions via response fingerprinting.
- **OWASP Amass** — `amass enum -passive -d target.com | grep api` → subdomains incl. `legacy-api`, `*-backup`, `dev` (improper-asset-management candidates). `amass intel -addr IPs`, `-d domain -whois`, `enum -active -brute -w API_superlist`, `viz` for graphs.
- **GitHub** — search `org-name` + `api-key`/`password`/`token`. Mine all tabs: Code (search "api","key","secret"; check **History** for removed secrets), Issues (open issue = live bug), Pull requests (unmerged secret removal still shows the key in Files Changed). Even without secrets, harvest languages, endpoints, docs.
- **Wayback Machine** — retracted API docs, old endpoints, deprecated versions.
- **Pastehunter** — pastebin-style leaks.
- **Robots.txt** — disallowed paths literally list what they want hidden (`Disallow: /api/`).

## Active recon — 4-phase loop

**Phase 0 — Opportunistic exploitation**: any vuln found at any phase → exploit now, return to process after.

**Phase 1 — Detection scanning**: `nmap -sC -sV target -oA name` + `nmap -p- target`. Flag every HTTP/HTTPS service, non-standard ports (APIs hide on 8000s, 5000, 8888), DBs (27017 MongoDB), `Content-Type: application/json` + error bodies in script output.

**Phase 2 — Hands-on analysis**: browse the app as **guest / authenticated user / admin**. Intercept with Burp. Look for API calls behind search bars, auth flows, uploads.
- DevTools **Network**: open JS files in Sources, search "api","apikey","secret","password"; XHR filter for Ajax calls.
- DevTools **Memory**: heap snapshot → search `api`, `v1`, `v2`, `swagger`, `rest`, `dev`. Compare snapshots across auth states/features to reveal hidden API paths.
- DevTools **Performance**: record a UI action → see API requests it fires (e.g., login click → `POST /identity/api/auth/login`).

**Phase 3 — Targeted scanning**: refine by API type/version/webapp; e.g., WordPress → `/wp-json/wp/v2`. Kick off brute-forcers, keep analyzing manually while they run.

## Validating discovered APIs

- **Burp Repeater**: send real request vs gibberish path. 401 (exists, needs auth) vs 404 (doesn't) — and check `WWW-Authenticate` headers for verbose path leaks like `/api/auth`.
- **ZAP**: Quick Start automated scan → Spider/Sites tabs; Search tab for `API`, `GraphQL`, `JSON`, `RPC`, `XML`. Manual Explore + HUD for guided browsing.
- **Gobuster**: `gobuster dir -u http://target:8000 -w common_apis_160 -x 200,202,301 -b 302` — API-focused wordlists beat giant generic ones.
- **Kiterunner**: `kr scan http://target -w routes-large.kite` — hits with all verbs + realistic shapes (POST `/api/v1/user/create` not just GET). `kr brute target -w list.txt` for text lists; multi-target = line-separated file. **`kr kb replay "<result line>" -w routes.kite`** replays a hit to see the full response; `--proxy=http://127.0.0.1:8080` routes the replay into Burp.

## Document everything

Screenshot findings, keep a task list of recon artifacts (endpoints, creds, versions, docs, business purpose). Revisit the list when exploitation stalls.
