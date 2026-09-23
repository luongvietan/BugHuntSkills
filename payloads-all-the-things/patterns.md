# Patterns — payload selection heuristics

Reusable rules that decide *which* payload family to try and *how* to mutate it. Ordered roughly by frequency of payoff.

## The master loop

1. **Classify the sink** — where does input land? (HTML/attr/JS/URI; SQL clause; template tag; file path; header; serialized blob; URL fetched server-side.) If you can't name the context, don't fire a payload yet.
2. **Fire a detection payload** — inert, context-matched: `'`, `{{7*7}}`, `${{<%[%'"}}%.`, `;sleep 5`, `<!ENTITY x "y">`, `$(id)`, `O:`, `{{"foo"}}`, `../`. 
3. **Read the anomaly** — diff status/size/time/errors vs baseline. No anomaly → the context guess was wrong or the input is properly handled; move to the next parameter or mutate encoding.
4. **Fingerprint the engine/parser** — error text, version banners, behavior on paired payloads (see per-chapter fingerprint tables).
5. **Pick the family** — union/error/blind/time/OOB for SQLi; reflected contexts for XSS; scheme list for SSRF; wrapper list for LFI.
6. **Mutate for the filter** — encoding, case, comments, whitespace substitutes, keyword splitting, alternate syntax, parser differentials.
7. **Escalate minimally** — `id`/`hostname`/`alert(domain)`/`version()`/a canary row. Stop where impact is proven.

## Universal heuristics

- **Baseline first.** Capture the normal response; every later claim of "different" needs that reference.
- **Pair payloads.** `1 AND 1=1` vs `1 AND 1=2`, `{{3*4/2}}` vs `{{3*)2(/4}}` — true/false pairs rule out interference.
- **Polyglots for unknown contexts.** When the sink could be several places (`jaVasCript:...polyglot`, `SLEEP(1) /*' or SLEEP(1) or '" ...`), a polyglot tests all of them in one shot.
- **Hex/base64 the inner payload** when the transport layer mangles it (routed SQLi `0x27...`, php filter `convert.base64-encode`, `data:...;base64`).
- **Two decoders = mutation surface.** Anywhere input passes through ≥2 parse stages, try double-encoding, overlong UTF-8, unicode normalization, and format-wrapper tricks.
- **Filter = blacklist → find the miss.** Lists are enumerated: try `/**/` for space, `'`→hex, case variants, keyword-insertion (`UNIunionON`), comment-interleave, alternate verbs (LIKE/REGEXP/BETWEEN for `=`).
- **Validator ≠ consumer.** Parser differentials (URL `@`/`\`/`#` splits, HPP first-vs-last, `Transfer-Encoding` obfuscation, `0://` schemes) make the checker and the doer disagree.
- **Output missing ≠ dead.** Switch oracle: timing (SLEEP/sleep/BENCHMARK), DNS (LOAD_FILE UNC/xp_dirtree/nslookup), HTTP callback, error-content oracles, content-length deltas.
- **Gadgets beat payloads.** For deserialization/SSTI, don't craft syntax — identify the framework, run the chain generator (phpggc/ysoserial/ysoserial.net/SSTImap), then adapt transport encoding.
- **Escape the sandbox inward.** Blocked `system`? enumerate loaded classes/modules (`__subclasses__`, `getClass().forName`, `Object.prototype` pollution) — the dangerous function usually exists inside the app already.
- **State-changing endpoints are CSRF candidates; navigational endpoints are open-redirect candidates.** Same param name (`next`/`return`/`redirect`) → test both.
- **Auth bypasses compose.** JWT alg confusion + claim tamper; OAuth redirect_uri + open redirect; SAML unsigned assertion + comment truncation. Never test one mechanism in isolation.
- **Side-effects are the bug.** In logic testing, the impact is the *side effect* (email sent, file written, cache stored, token minted) — assert on that, not on the response body.
- **Callbacks only to owned infra.** Every OOB/interactsh/Collaborator payload must point at infrastructure you control or the program explicitly authorizes. Log the exact beacon received as evidence.
- **Volume = permission.** Fuzzing, brute force, batching, race floods, and ReDoS are rate-sensitive — confirm the program allows it, throttle, and watch for WAF bans that pollute your signal.
- **Don't escalate past proof.** A read of `/etc/hostname` proves XXE; a `sleep 5` proves command injection; `alert(document.domain)` proves XSS. Anything more is scope creep.

## Context→first-probe map

| Observed sink | First probe | Family file |
|---|---|---|
| Reflected in HTML/attr/JS | `patt'"><>/${{7*7}}` | ch01 |
| `?id=`, search, filter | `'`, `AND 1=1`/`AND 1=2`, `SLEEP(5)` | ch02 |
| JSON/form login, Mongo-ish API | `{"$ne":null}`, `user[$ne]=x` | ch03 |
| Input inside `{{`,`${`,`#{`,`<%` | `{{7*7}}`, `${{<%[%'"}}%.` | ch04 |
| `url=`, `webhook`, `image-url` | `http://127.0.0.1`, metadata, callback | ch05 |
| XML/SOAP/docx/svg body | `<!ENTITY x "y">`, `file:///etc/hostname` | ch06 |
| `page=`, `file=`, `include`, path | `../`, `php://filter/...base64...` | ch07 |
| `ip=`, `host=`, ping/dns/convert tools | `;sleep 5`, `$(id)`, backticks | ch08 |
| File upload | exec-extension + MIME/magic-byte variants | ch09 |
| `rO0`,`O:`,`gASV`,`BAgK`,`AAEAAAD` blobs | format-specific blind probe (URLDNS/nslookup) | ch10 |
| `/graphql` | `{__schema{types{name}}}` | ch11 |
| `next`/`redirect`/`url` in links | `//YOUR-CALLBACK`, bypass list | ch12 |
| state-changing POST, no token | autosubmit form, text/plain JSON trick | ch12 |
| JWT cookie/header | decode → `alg:none`, `kid`, jku, crack | ch13 |
| reset/forgot/mfa endpoints | host-poison, param pollution, `otp:[]` | ch13 |
| proxy/CDN in front | CL.TE probe, cache-deception suffix, CRLF | ch14 |
| `Origin:` reflected | ACAO/ACAC check → PoC XHR | ch14 |
| `id=`/`uid=`/`doc=` in API | increment, wildcard, array, HPP | ch15 |
| JSON merge/URL parse in JS app | `{"__proto__":{"status":510}}` | ch15 |

## Anti-patterns

- Firing exploit payloads before a detection payload proves evaluation.
- Assuming the WAF rule list — mutate until the filter reveals itself; never claim "blocked" from one failed string.
- Copying a gadget payload without checking the target's library versions (phpggc chain must match installed packages).
- Pointing OOB payloads at shared/public infrastructure you don't control.
- Reporting "potential" from a payload that never produced an observable signal — the anomaly IS the finding.
