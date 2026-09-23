# Patterns — reusable moves from Hacking APIs

## Recon patterns

- **Dork stack**: Google GHDB (`inurl:"/wp-json/wp/v2/users"`, `intitle:"index of" api_key`) → Shodan (`hostname:"t.com" "content-type: application/json"`) → Amass (`enum -passive -d t.com | grep api`) → ProgrammableWeb/RapidAPI → GitHub (org + `api-key`/`token`, check *History/Issues/PRs*) → Wayback for retracted docs.
- **Version drift**: found `/api/v2` → try `/v1`, `/v3`, `/test`, `/internal`, `/mobile`, `/legacy`, `legacy-api.*` subdomains. Older = fewer controls (improper asset management).
- **DevTools API hunt**: Network→JS Sources (grep `api|secret|key`) → Memory heap snapshot (search `api|v1|swagger`) → Performance record (click login → `POST /identity/api/auth/login`). Compare heap snapshots across auth states for hidden paths.
- **Robots.txt first** — disallowed paths are the target's own secret map.

## Endpoint analysis patterns

- **Spec-first**: always hunt `swagger:"2.0"`/OpenAPI/RAML/collection before manual mapping; Import→Link in Postman.
- **baseURL variable**: `{{baseURL}}` per collection → re-version every request by editing one variable; also swap whole environments (dev/stage/prod).
- **Reverse engineer by proxy**: Postman capture + FoxyProxy + use *every* feature (register/login/reset/all links/profile/shop/forum) → prune → folderize.
- **Baseline ritual**: intended-use requests until all 200s; record normal status/size/timing/shape; every later anomaly is measured against this.
- **Admin-docs-as-target-list**: privileged actions from public docs → replay unauth → low-priv → admin; gated admins = tomorrow's goal after token forgery.

## Auth attack patterns

- **Fail-signature hide**: discover the canonical failure (`--hc 405`, fixed length) → hide it → anomalies pop.
- **Spray math**: lockout=10 → ≤9 passwords; list = Season+Year+!, Password1!, org+@year; cluster bomb (users × pwds).
- **OTP probe**: reset your own account → learn OTP shape → brute-forcer payload charset+length to match.
- **Sequencer funnel**: live-capture N tokens → char-position analysis → brute-force only low-entropy positions (static prefix + tiny charset = few hundred requests).
- **JWT ladder**: replay → strip sig → `alg:none` (`jwt_tool -X a`) → RS256→HS256 with public key as HMAC secret → crack HS256 secret → forge claims (`is_admin`, `sub`, `userID`).

## Authorization patterns

- **BOLA matrix**: ID in path/body/headers; int/email/group/composite; array-wrap `{"id":[3333]}`; nest `{"id":{"id":3333}}`; duplicate keys; token-shaped IDs incremented char-wise.
- **Existence oracle**: 404-vs-401/405 or length/time diffs → enumerate valid users/IDs without reading data → feed combo tests.
- **BFLA A-B-A**: B touches A's resource → A verifies change → reportable proof.
- **Verb fuzz**: `§GET§` → PUT/POST/PATCH/DELETE/OPTIONS; a lone 400 among 405s = real method, wrong payload → Arjun the params.
- **Mass-assignment combo**: BFLA write + injected `email`+`mfa:false` → password reset → takeover.

## Injection patterns

- **Metachar probe set**: `' '' ;%00 -- -- - "" ; ' OR '1 ' OR 1-- - " OR ""=" OR 1=1 ' OR ''='` → watch for DB-flavored errors.
- **NoSQL trio**: `{"$gt":""}` / `{"$ne":""}` / `{"$nin":[1]}` in credential fields → auth bypass; `{"$where":"sleep(1000)"}` → timing confirm.
- **Command-sep × cmd cluster bomb**: `| || & && ' " ; '"` × `whoami/uname -a/cat /etc/passwd` (*nix) or `ver/dir/ipconfig` (Windows).
- **JSON-quote position fix**: nested-object payload breaks syntax → move `§` outside the value quotes: `{"k":§"v"§}`.
- **Uncheck Intruder URL-encoding** when payloads must arrive raw.
- **XAS route**: API field → rendered page → `<script>` in `city`, product name, comment; try `Content-Type: text/html`.

## Evasion patterns

- **Null-byte sandwich**: `%00`+payload+`%00` via Intruder processing rules (encode first, wrap second).
- **Path jitter for rate limits**: `%00`/`%20`/case/hyphen mutations on the path + junk `?test=§N§` (pitchfork).
- **Origin-header spray**: X-Forwarded-For/X-Host/X-Client-IP/etc with 127.0.0.1/private/peer-IPs; User-Agent cycling.
- **IP Rotate**: AWS API Gateway egress = a fresh IP per request.
- **Throttle-to-stay-legal**: Wfuzz `-s`, Intruder Resource Pool ms — a limit you can live inside still demonstrates weakness.

## GraphQL patterns

- **Endpoint hunt**: `/graphql /graphiql /altair /playground /console /query` + version/legacy variants; cookie flip `env=graphiql:disable→enable` to unlock IDEs.
- **Introspection first** (`__schema`) → InQL scan → body-read reverse engineering, in that order.
- **200-blind rule**: ignore status codes; diff bodies/lengths; GraphQL errors live in `"errors":[]`.
- **Arg-level BOLA**: fuzz sequential IDs inside query args (`pId`, `userId`, `media_id`); mutations' `variables` are the injection surface.
