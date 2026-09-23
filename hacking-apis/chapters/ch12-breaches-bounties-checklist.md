# Ch15 + Appendix A: Breaches, Bounties & API Hacking Checklist

Source: Chapter 15 (Breaches and Bounties), Appendix A (API Hacking Checklist).

## Breach case studies — what the pattern teaches

**Peloton (3M+ users, Jan Masters)** — three unauthenticated data paths at once: `POST /stats/workouts/details` ignoring the "private profile" flag (IDs array in body — brute-forceable or harvested from the app), `GET /api/user/search/:username` leaking pic/location/ID/followers, and unauthenticated GraphQL `user(id:)` queries. Lesson: one weak API ⇒ audit *all* their APIs; "private" flags are client-side decoration until enforced server-side.

**USPS Informed Visibility (60M users)** — any authed user could query *any* account; wildcard support (`?email=*@gmail.com`); excessive data exposure returned all occupants per address. Prior "vulnerability assessment" gave false negatives because tools didn't test the API. Lesson: scanners miss business logic; wildcard search is its own vuln class.

**T-Mobile (2.3M customers)** — Web Services Gateway API: one token + `msisdn=<any phone>` → full customer record (BOLA). Phone numbers are enumerable/public → mass harvest possible. When a finding implies mass exposure, ask the provider for extra test accounts rather than dumping real users.

## Bounty case studies

**Ace Candelario ($2k)** — JS file grep `api|secret|key` → base64 `Authorization: Basic` for BambooHR API in source → decode → valid creds → employee-directory PoC. Exposed keys = broken auth found during discovery; severity = what the key unlocks.

**Omkar Bhagwat ($440)** — dir-enum found `/api/docs`; unauthenticated `/ping`→`pong`; doc map + an exposed Bearer token found elsewhere on the site → admin functions (edit/delete/create accounts) = BFLA. Docs + leaked token + persistence = chain.

**Sam Curry — Starbucks ($4k, ~100M records)** — noticed `/bff/proxy/*` (backend-for-frontend proxying to internal APIs). `/orchestra` blocked input; `/bff/proxy/stream/v1/users/me/streamItems/..\` didn't. `404`→`400` anomaly (Comparer!) led him to brute-force internal paths → hit Microsoft Graph `/search/v1/accounts` → `$count=true` = 99,356,059 customer records. Info disclosure + misconfig + BFLA chained. Lessons: subtle response diffs matter; probe proxies for traversal; one anomaly is a door.

**Mayur Fartade — Instagram ($30k)** — `POST /api/v1/ads/graphql` with `access_token:""` + target `MEDIA_ID` → private posts/stories/reels of anyone (BOLA on GraphQL). Media IDs brute-forceable/leakable. Empty/null token params are worth trying.

## API Hacking Checklist (Appendix A) — the full loop

- **Approach**: black / gray / white box decided? scope + constraints clear?
- **Passive recon**: attack-surface discovery (Amass/dorks/Shodan/API dirs/GitHub/Wayback); exposed secrets checked
- **Active recon**: port/service scan (Nmap ×2); vuln scan (Nikto/ZAP); used app as intended; API dirs searched; endpoints discovered (Gobuster/Kiterunner/DevTools)
- **Endpoint analysis**: docs found & reviewed; API reverse engineered (collection built); responses analyzed for info disclosure/excessive data/logic flaws
- **Authentication**: basic auth testing (brute/spray/OTP); token attacks (Sequencer entropy, position brute-force, JWT none/alg-switch/crack)
- **Authorization**: resource-ID methods mapped; BOLA tested (A-B, matrix, side-channel); BFLA tested (A-B-A, privilege ladder)
- **Mass assignment**: standard params discovered; writable-object endpoints tested with rogue variables
- **Injection**: input-accepting requests listed; fuzzed for injection points; XSS/XAS; DB-specific attacks (SQL/NoSQL); OS command injection
- **Rate limits**: existence confirmed; avoidance tested (throttle); bypass tested (path, headers, tokens, IPs)
- **Evasion**: string terminators; case switching; encoding (incl. double); combined techniques

## Cross-case lessons

Research your target deeply before attacking; always test discovered creds; docs are gold; combine small findings; diff responses obsessively; learn the *provider's* syntax (e.g., MS Graph `$count`) to quantify impact for the report.
