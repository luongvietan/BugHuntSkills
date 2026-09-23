# Ch4–5: The API Hacking System & Vulnerable Targets

Source: Chapter 4 (Setting Up an API Hacking System), Chapter 5 (Setting Up Vulnerable API Targets).

## Core toolchain and what each tool is *for*

| Tool | Role in the workflow |
|---|---|
| **Burp Suite** | Intercept/validate APIs (Proxy), replay & tweak (Repeater), fuzz (Intruder), entropy analysis (Sequencer), compare responses (Comparer), extensions (IP Rotate, InQL, Autorize-class helpers). CE Intruder is throttled → use Wfuzz/Gobuster for bulk brute force. |
| **Postman** | API client: build/maintain request collections, import OpenAPI/Swagger specs, collection & environment **variables** (`{{token}}`, `{{baseUrl}}`), Collection Runner for wide fuzzing, proxy capture mode for reverse engineering, Tests panel (`pm.test`) for anomaly assertions. |
| **Kiterunner (kr)** | Best-in-class API endpoint discovery: uses all HTTP verbs and realistic API path shapes, `.kite` route files, `-H` auth header support, `kb replay` to dissect interesting hits (can proxy replay through Burp). |
| **Wfuzz** | Fast CLI fuzzer/brute-forcer: `FUZZ`/`FUZ2Z`/`FUZ3Z` positions, `-z file/list/range`, `--hc/--hl/--hw/--hh` response hiding, `-t` threads / `-s` delay throttle, `-e encoders` payload processing (base64, md5, urlencode, random_upper; `,`-separate and `@`-chain). |
| **OWASP Amass** | External attack-surface mapping: `intel` (certs, reverse whois, ASN), `enum -passive/-active -brute -w wordlist`, `viz` graph export. |
| **Nmap** | Detection scanning: `-sC -sV -oA` general detection, `-p-` all-ports. APIs live on non-standard ports — don't stop at 80/443. Also `--script http-waf-detect`. |
| **Gobuster** | URI/subdomain brute force: `dir -u URL -w wordlist -x 200,301 -b 302`. Faster than Burp Intruder CE. |
| **OWASP ZAP** | Crawler + passive/active scanner; HUD for manual explore; Search tab for "API/GraphQL/JSON/RPC/XML" in results. |
| **Chrome DevTools** | Network (JS files → Sources, XHR/Ajax), Memory heap snapshots (search "api/v1/v2/swagger"), Performance timeline (catch API calls behind UI actions), Storage (cookie tampering). |
| **Arjun** | HTTP parameter discovery — heuristic + wordlist → valid params (mass-assignment hunting). `--stable` to slow down. |
| **jwt_tool** | JWT analysis + Playbook Scan (`-M pb`), forge alg:none variants (`-X a`). |
| **SQLmap** | `-r saved_request -p param`, `--dump-all`, `--dump -T tbl -C col -D db`, `--os-shell`, `--os-pwn`. |
| **FoxyProxy** | One-click browser → Burp/Postman proxy switching. |
| **SecLists + Assetnote wordlists** | Fuzzing payloads (metacharacters, User-Agents, graphql.txt), API-specific route lists, `big-list-of-naughty-strings`. |

## Practice targets (Ch5)

- **crAPI** (completely ridiculous API) — the book's main lab: auth, BOLA, mass assignment, NoSQLi, JWT.
- **Pixi** (OWASP DevSlop) — documented API w/ Swagger; auth bypass, NoSQLi, mass assignment.
- **DVGA** (Damn Vulnerable GraphQL App) — GraphQL introspection, BOLA, command injection.
- Others worth adding: Tiredful API, vAPI, OWASP Juice Shop, WebGoat.

## Setup habits that pay off

- Save every successful auth request in Postman — tokens expire/get revoked mid-test; regenerate quickly.
- Store tokens as collection/environment variables (`{{hapi_token}}`); one swap tests all requests as a different user.
- Configure Postman to proxy through Burp — every crafted request is also interceptable/fuzzable.
- Keep a `~/api/wordlists` tree: API paths, common dirs, fuzzing payloads, User-Agents, Kiterunner `.kite` routes.
- Create **burner accounts** before attacking: several disposable accounts/tokens with non-correlating registration data (different emails/names; consider VPN at registration). Needed to probe security-control thresholds without losing your main access.
