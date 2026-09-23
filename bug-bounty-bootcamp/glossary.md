# Glossary — Bug Bounty Bootcamp

Terms as Vickie Li uses them. For API-specific terms see `hacking-apis`/`owasp-api-security-top-10` glossaries.

## Hunting & process

- **Attack surface** — all the points where the app accepts input or exposes functionality: endpoints, params, headers, uploads, APIs, mobile components.
- **Attack scenario** — the narrative that turns a technical flaw into business impact (what the attacker does + what they gain). Drives report severity.
- **Passive vs active recon** — passive: no packets to target (cert logs, Wayback, OSINT, Shodan). Active: talking to the target (subdomain brute-force, port scans, directory enum) — needs program permission.
- **Data injection/entry point** — any location input can enter: params, headers, cookies, paths, body fields, filenames.
- **Two-account diff (A/B testing)** — Li's core access-control method: replay account A's requests with account B's session; divergence = authZ bugs. Automated by Autorize/AuthMatrix.
- **Bug chain** — combining low-sev findings into real impact (self-XSS + login CSRF = stored-XSS equivalent; open redirect + OAuth = token theft).
- **Bug slump** — dry spell; expected. Response: switch target/vuln class, learn a new technique, take a break.
- **N/A / Informational / Duplicate** — report verdicts: no security impact shown / real bug but no demonstrated impact / already reported.

## Web fundamentals

- **Origin** — `scheme://host:port` triple; the unit SOP protects.
- **SameSite** — cookie attribute controlling cross-site sending: `Strict` (never), `Lax` (top-level GET only — Chrome default), `None` (always; requires `Secure`).
- **HttpOnly / Secure** — cookie flags: JS can't read / HTTPS only.
- **CORS** — server opt-in to relax SOP via `Access-Control-Allow-*` headers; `Allow-Credentials: true` makes misconfig exploitable with cookies.
- **CSP** — `Content-Security-Policy` header restricting script/style/frame sources; `frame-ancestors` = modern clickjacking defense.
- **X-Frame-Options** — legacy frame control: `DENY`, `SAMEORIGIN`, (obsolete `ALLOW-FROM`).
- **postMessage** — JS API for cross-origin window messaging; must validate `event.origin`.
- **JSONP** — JSON wrapped in a callback function name — a sanctioned SOP bypass that often leaks authenticated data.
- **document.domain** — JS property for relaxing SOP between subdomains; setting it opens parent-domain access.

## Vuln classes

- **Stored / Reflected / DOM / Blind XSS** — persistence & location axes of XSS; blind = fires where you can't see (admin panel) → needs OOB callback.
- **Self-XSS** — XSS that only affects your own account; not reportable alone — must chain.
- **Open redirect** — user-controlled redirect destination; value is in chains (SSRF, OAuth, phishing).
- **Clickjacking** — iframe-based UI redress; needs sensitive action to matter.
- **CSRF** — forced authenticated state change; defenses: tokens, SameSite, Referer/Origin checks.
- **IDOR / BOLA** — missing object-level authorization on a user-controlled object reference.
- **Second-order SQLi** — payload stored safely, executed in a later query elsewhere.
- **Race condition / TOCTOU** — check-then-act window exploited by concurrent requests (single-packet attack = synchronized sends).
- **SSRF / Blind SSRF** — server-side request to attacker-chosen destination; blind = confirmed only via OOB interaction.
- **Deserialization / gadget chain** — crafted object bytes → magic-method execution; chain = sequence of existing class methods reaching RCE.
- **XXE / parameter entities** — XML entity resolution to files/URLs; parameter entities (`%`) enable OOB exfil when output entities are blocked.
- **SSTI / CSTI** — template injection server-side (engine eval → RCE) vs client-side (AngularJS → ≈XSS).
- **LFI / RFI** — local/remote file inclusion via path-controlled include.
- **SAML / IdP / SP / assertion** — identity provider authenticates; service provider consumes the signed assertion XML.
- **OAuth terms** — `client_id`, `redirect_uri`, `state` (CSRF defense), `code`/`token`, scope, implicit vs authorization-code flow.
- **XSW (XML signature wrapping)** — duplicate SAML assertions so validator signs one, parser uses the other.
- **Subdomain takeover** — dangling CNAME → claim the third-party resource → control subdomain content + shared cookies.
- **Certificate pinning** — app trusts only specific certs → must bypass (Frida/Objection) to proxy HTTPS.
- **APK components** — Activities, Services, BroadcastReceivers, ContentProviders; "exported" = reachable by other apps.
- **WSDL** — SOAP service descriptor; a free endpoint map when exposed.
- **GraphQL introspection** — self-describing schema queries (`__schema`, `__type`).
- **Single-packet attack** — Burp's parallel-send over HTTP/2 that lands requests in the same ms window.

## Tools

- **Burp Suite** — proxy/repeater/intruder/sequencer/decoder/collaborator; extensions: Autorize, AuthMatrix, Auto Repeater, SAML Raider, Turbo Intruder.
- **Wfuzz** — CLI web fuzzer, `FUZZ`/`FUZ2Z` position markers, `--hc` hide code, `--follow`, `--filter`.
- **SecLists / FuzzDB / Naughty Strings** — wordlist collections for enum + payload fuzzing.
- **sqlmap** — automated SQLi; `--tamper`, `--level/--risk`, `-r request.txt`.
- **ysoserial / PHPGGC / ysoserial.net** — gadget-chain generators for Java/PHP/.NET deserialization.
- **Collaborator / interactsh** — OOB listener for blind bugs (SSRF, XXE, RCE, blind XSS).
- **Apktool / JADX / Frida / Objection / MobSF** — Android unpack, decompile, instrumentation, auto-analysis.
- **EyeWitness/Snapper** — mass screenshot triage. **Sublist3r/Amass/SubBrute/Altdns/Gobuster/Dirsearch** — subdomain & dir enum. **Masscan/Nmap/Shodan/Censys** — ports/services. **Gitrob/truffleHog/gitleaks** — secret hunting.
