# Cheatsheet — The Bug Hunter's Methodology (TBHM)

Quick-lookup card. Stage detail in `chapters/`; heuristics in `patterns.md`; terms in `glossary.md`. Authorized, in-scope testing only.

## Session kickoff (stage order)

1. Read the program policy: wildcard scope (`^.acme.com`?), acquisition rules, banned techniques, rate limits.
2. **Discovery**: subdomain/domain enum (Recon-ng-era → `recon-pipeline` modern stack) → Google dorks `site:target.com -www.target.com` → acquisitions via Wikipedia M&A lists → mobile sites/new versions.
3. **Full port scan** all new domains: `nmap -sS -A -PN -p- --script=http-title <host>` — separate webapps, stray services, unauthenticated consoles.
4. **Mapping**: dir brute-force (RAFT, SVN/Git Digger, `v4/all2.txt` + stack-matched lists) → platform ID (Wappalyzer/BuiltWith/retire.js) → CVE check → OSINT vuln history → recurse into 401/403 dirs.
5. **Tactical fuzzing**: polyglot per input; escalate to class payloads where it lands.
6. **Manual stage**: two personas (peon + admin) → priv/IDOR/logic/transport.
7. **Wrap**: mobile storage → "noise" vulns → n-minute battery → report from templates (fix the domains!).

## Per-feature core questions

| Question | Class to test |
|---|---|
| Displays something to users? | XSS (polyglot → context payload) |
| Calls on stored data? | SQLi (blind/time first) |
| Touches filesystem or takes file/URL input? | LFI, upload, RFI, open redirect |
| Changes state? | CSRF battery (8 rotations) |
| References an object by ID? | IDOR (inc/dec/negative/substitute) |
| Restricted to a role? | Privilege replay cross-persona |
| Goes over the wire? | Transport gap check |

## Vuln-class quick tests

| Bug | First probe | Escalate |
|---|---|---|
| XSS | polyglot (`" onclick=alert(1)//<button ' onclick=alert(1)//> */ alert(1)//`) | map context → minimal clean payload; vectors: themes, JSON bodies, filenames, fake params, error pages |
| SQLi | `SLEEP(1) /*' or SLEEP(1) or '" or SLEEP(1) or "*/` | `'+BENCHMARK(40000000,SHA1(1337))+'` timing; sqlmap `-l burp.log` + tamper scripts |
| LFI | file-ish params + `../` / `JHADDIX_LFI.txt` | `/etc/passwd`, `php://filter`, log/session inclusion → RCE |
| Upload | alternate ext (html/php3/aspx), content-type spoof | parser metadata XSS, file polyglots, filename-reflection XSS |
| Redirect/RFI | `dest= continue= redirect= url= uri= window= next=` / `file= path= template= doc=` | blacklist ladder: `\/`, `//`→`/`, scheme-strip, `/%09/`, `./`→`..//`, `../`→`....//` |
| CSRF | replay state-change cross-origin | GET-swap, token=`undefined`, drop token, cross-account token, same-length garbage, `text/plain` downgrade, sibling-subdomain mint |
| Auth battery | user/pass discrepancy on login/reg/reset | lockout, password policy, re-auth for updates, reset-token expiry/reuse |
| Session battery | replay old cookie after logout/rotation | fixation (no new cookie), never-expiring, multi-session, reversible (base64-tell) values |
| IDOR | rotate every UID ±1, negative, cross-account | receipts, non-public images, PDFs, POs, messages, change/forgot-password, admin functions |
| Privilege | replay admin functions as peon (Autorize) | direct-browse sensitive views: analytics, payments, PII |
| Transport | hunt HTTP exceptions | insecure images, analytics w/ session data, HSTS/`Secure` gaps |
| Logic | skip/reorder steps, negative qty, forged hashes | auth bypass, app-level DoS, timing deltas |
| Mobile | check storage for unencrypted PII | system logs, cache.db, plists/dbs, hardcoded-in-binary |

## The n-minute assessment (in order)

1. Polyglots → search, registration, contact, password-reset, comment forms.
2. Scanner pass on those same functions.
3. Cookie lifecycle: logout → check → login → check → replay old.
4. User enumeration on login/registration/reset.
5. Run a reset: plaintext delivery? URL token? predictable? reusable? auto-login?
6. Rotate numeric account IDs in URLs.
7. Sensitive functions: unauth, lower-priv, CSRF, HTTP.
8. Dir brute-force — short SecLists top-list only.
9. Upload functions — alternate executable types.

## Report skeleton

Template per class; always re-check domains/URLs. Title `[class] in [feature] → [attack scenario]` → summary → severity by impact → numbered repro → minimal PoC → remediation. Attack scenario is what gets paid.
