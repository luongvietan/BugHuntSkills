# Impact Library — Per-Class Statements, Escalation, Overclaim Traps

Copy a statement, fill the `[blanks]` with what your PoC actually showed, then check the escalate and do-NOT-claim lists. Statements are written in attacker-outcome language — swap placeholders, don't restructure.

**The one rule above all:** every sentence in your report must be either
*demonstrated* (you ran it, evidence attached) or *labeled theoretical*
("a possible escalation path, not tested — would require X"). Never blur
the line: a triager who catches one theoretical claim phrased as fact
discounts the whole report. And "escalate" here means *what to demonstrate
if it exists* — it never means exploit further than the policy permits;
an escalation that needs real-user data, third-party systems, or banned
techniques stays a labeled hypothesis forever.

## IDOR / broken object-level authorization

- **Statement:** "The `[endpoint]` authorizes the caller's session but never verifies ownership of `[object]`. By changing `[parameter]` from my test account's `[ID A]` to `[ID B]`, I retrieved `[data fields]` belonging to a second account I control. The `[sequential/enumerable]` ID space makes this exploitable at scale — `[N]` objects are reachable with one request each."
- **Escalate:** show the ID space is enumerable and unthrottled; name the sensitive fields actually returned (PII, payment data, documents); prove a write-variant exists (modify/delete another user's object) — write IDOR lands a band above read IDOR.
- **Do NOT claim:** "all users' data" if you only read your two test accounts (say "the mechanism has no ownership check; the ID range covers ~[N] objects"); admin access unless the object grants it; financial loss unless the object is money.

## Stored XSS

- **Statement:** "Input stored via `[field/feature]` renders without encoding on `[page]` to every `[audience: user/admin]` who views it. The payload `[minimal payload]` executes in the `[domain]` origin, giving the attacker full script access to the victim's session — demonstrated by `[alert(document.domain) / reading own test data]`."
- **Escalate:** fires without interaction on a high-traffic page; reaches privileged viewers (support/admin dashboards → stored XSS on staff is near-ATO); domain hosts session-bearing cookies (even HttpOnly — XSS doesn't need to read the cookie to ride it).
- **Do NOT claim:** you "stole the session" by popping an alert (say "script execution in victim's origin → actions as that user"); wormability unless you demonstrated self-propagation; RCE — XSS is client-side.

## Reflected / DOM XSS

- **Statement:** "`[parameter/source]` reflects into `[context]` on `[page]` without encoding. A crafted link `[URL]` executes script in the victim's `[domain]` session when clicked — no stored payload needed."
- **Escalate:** demonstrate a concrete victim flow (link + one click → action in victim session, shown on your own two accounts); bypass of the browser's XSS protections; reachable pre-auth/on the login page.
- **Do NOT claim:** account takeover without showing the chain to it (a click-through PoC + realistic delivery scenario is usually Medium); that it's stored — conflating classes tells triage you don't know your own bug.

## CSRF

- **Statement:** "`[state-changing action]` at `[endpoint]` has no anti-CSRF token (or accepts any/blank token). A victim browsing an attacker page silently performs `[action]` with their session — demonstrated by `[email changed / settings modified]` on my own test account."
- **Escalate:** the action is account-recovery-adjacent (email/password/2FA change → persistent takeover); token exists but isn't validated (try missing, blank, other-account's token — each bypass deepens the bug); JSON endpoint exploitable via `fetch` + lax CORS or content-type confusion.
- **Do NOT claim:** impact beyond the action itself ("full account takeover" needs the email-change-to-reset chain shown); GET-based actions as CSRF if nothing state-changing happens; that SameSite doesn't exist — check it before claiming exploitability.

## SQL injection

- **Statement:** "`[parameter]` at `[endpoint]` is concatenated into a SQL query. `[Observable: time delay of 5s / error revealing query fragment / boolean difference]` confirms execution, and `[version() / one row from a non-sensitive table]` demonstrates read capability beyond the intended query."
- **Escalate:** data read shown (version, current user, one table name — minimal proof, not a dump); write/file capability (`INTO OUTFILE`, stacked queries) — state it only if you ran it; the injectable point is pre-auth.
- **Do NOT claim:** "entire database compromised" — you demonstrated capability, not extraction; RCE unless you actually reached `xp_cmdshell`/`udf`; a specific row count you didn't query. Minimal PoC protects you: policies forbid bulk data access.

## SSRF

- **Statement:** "`[parameter]` at `[endpoint]` causes the server to issue requests to an attacker-chosen URL. I confirmed requests reach `[my callback listener]` from the server's network, and demonstrated reach to `[internal host/metadata endpoint / port response difference]`."
- **Escalate:** hit the cloud metadata service or an internal admin panel (only if in-scope — check policy, many programs ban metadata access; when banned, demonstrate internal reachability another way); protocol smuggling (`gopher`, `file://`) only if you executed it; blind SSRF is still reportable — prove the outbound connection and the internal-vs-external response difference.
- **Do NOT claim:** cloud takeover because the URL parser accepted a metadata IP you were told not to fetch; internal network "access" from a single timing difference without a second data point; RCE via SSRF unless a chain actually ran.

## RCE (command injection / deserialization / SSTI / file upload → exec)

- **Statement:** "`[input]` at `[endpoint]` reaches `[shell/template engine/deserializer]` unvalidated. Executed `[id / hostname / harmless marker file]` — output/artifact attached. This is arbitrary code execution as user `[service account]` on `[host role]`."
- **Escalate:** the running user's privileges and what the host reaches (internal network position, data stores) — from commands like `id`, `hostname`, not from pivoting; pre-auth reachability; persistence is **never** your job to prove.
- **Do NOT claim:** you "could have" dumped databases / moved laterally — say "execution confirmed; privilege and network position as shown"; SSTI is RCE *only after* the escape executes — template arithmetic `{{7*7}}` proves injection, name the class correctly; anything you ran beyond the minimal marker — over-execution is a policy violation, not escalation.

## Race condition / TOCTOU

- **Statement:** "Concurrent requests to `[endpoint]` bypass the `[limit/one-time-use/balance check]`. Sending `[N]` parallel requests `[redeemed the coupon N times / withdrew more than the balance / applied the invite twice]` — `[before/after state]` attached."
- **Escalate:** money or quota actually duplicated on your test accounts (a number, not "possible"); reliability shown (works X of Y attempts); the check is on a high-value operation (payment, transfer, signup bonus).
- **Do NOT claim:** financial loss amounts beyond your test transactions; DoS — flooding the endpoint to "prove" it violates most policies; consistency bugs in display-only state as exploitable.

## Open redirect

- **Statement:** "`[parameter]` at `[endpoint]` redirects to an arbitrary external URL without validation. `[URL]` lands the victim on an attacker-controlled page after a trusted-domain link — usable in phishing and OAuth/callback flows."
- **Escalate:** the redirect sits in an auth flow (OAuth `redirect_uri` allowlist bypass → token leakage); `javascript:` or `data:` scheme accepted (→ XSS); chained into a demonstrated account-takeover on your own account.
- **Do NOT claim:** phishing impact as high severity by itself — platforms score standalone redirects Low; that it "steals tokens" without showing the flow; clickjacking-level ratings for it.

## Clickjacking

- **Statement:** "`[page]` lacks `X-Frame-Options`/CSP `frame-ancestors`. I framed it in a test page and the `[sensitive action]` button remains clickable — an attacker page can trick a logged-in victim into `[action]`."
- **Escalate:** the framed action is state-changing and one-click (delete account, authorize app, transfer); combined with a drag-drop/text-injection trick you actually built; frames the *settings* page, not just a public page.
- **Do NOT claim:** account takeover from framing a read-only page; that every missing header is exploitable (the action must be framable AND sensitive); multi-click flows as zero-interaction.

## Information disclosure

- **Statement:** "`[endpoint/artifact]` returns `[what: stack trace / internal IP / user list / hardcoded key]` to `[who: any unauthenticated user / any account]`. This exposes `[why it matters: internal structure for follow-on attacks / valid data directly]`."
- **Escalate:** the leaked item is directly usable (a working token you validated against your own account — never replay someone else's key); PII at scale via an enumerable endpoint; the leak is indexed/cached publicly (exposed via a search engine or public bucket).
- **Do NOT claim:** a version banner or stack trace is "sensitive data exposure" — it's a hardening/informative finding; keys as valid if you didn't safely verify them (verify against a sandbox you own or state unverified); directory listing as breach without reading contents.

## CORS / SOP misconfiguration

- **Statement:** "`[endpoint]` reflects the request `Origin` in `Access-Control-Allow-Origin` while allowing credentialed requests. An attacker page can read `[data]` cross-origin from a logged-in victim's browser — demonstrated with my own test origin and account."
- **Escalate:** `Access-Control-Allow-Credentials: true` present (cookie-authenticated reads → session-riding reads); wildcard-origin reflections including `null`; the readable endpoint returns PII/tokens.
- **Do NOT claim:** CORS findings on endpoints that return nothing sensitive — the misconfig alone is informative; "data theft" if cookies aren't included (`credentials:true` is the difference); that reflected Origin + no credentials still reads data — it doesn't, state the exact precondition.

## Auth / session / SSO (SAML, OAuth, JWT, password reset)

- **Statement:** "`[flow]` fails to `[verify signature / bind state / expire token / invalidate on change]`, allowing `[attacker outcome: forge assertion / fixate session / reuse reset link]`. Demonstrated on my own accounts: `[what happened]`."
- **Escalate:** forgery of another user's identity (two test accounts); a reset/invite token that's reusable, unexpired, or not bound to the requestor; OAuth `redirect_uri` / `state` bypass shown end-to-end.
- **Do NOT claim:** algorithm-confusion or signature bypass you didn't execute ("JWT accepts `none`" only if you logged in with it); weak password policy as auth bypass; session fixation where you can't plant the token in the victim flow.

## File upload

- **Statement:** "`[endpoint]` accepts `[file type]` without validating `[content-type/extension/content]`. Uploaded `[harmless file: .html with alert / .svg / polyglot]` is served at `[location]` under `[context: same origin / separate domain]`, giving `[stored XSS / content spoofing / code exec if executed]`."
- **Escalate:** the file serves from the main origin with an executable/sniffable content type; path traversal in the filename writes outside the uploads dir; the parser executes the file (image library RCE) — only claim what ran.
- **Do NOT claim:** RCE from a file that sits unexecuted in a bucket (it's stored-content risk — name it); that an S3-hosted file hits the app's origin; webshell upload "would work" — upload only if the policy explicitly permits and the file is inert.

## Business logic / integrity

- **Statement:** "`[flow]` trusts `[client-supplied value / step order / client-side check]` for `[price/quantity/role/status]`. By `[tampering / skipping step N]`, I `[purchased at changed price / gained role / bypassed verification]` on my test account — `[before/after]` attached."
- **Escalate:** direct financial impact demonstrated at small scale; the flaw is in a core flow (checkout, payout, KYC) not an edge feature; repeatable without race timing.
- **Do NOT claim:** theoretical fraud scale ("attackers could bankrupt…") — one executed $1 test beats a projected million; missing 2FA as a vuln class unless policy lists it; UX quirks as integrity flaws.

## Subdomain takeover / dangling DNS

- **Statement:** "`[host].example.com` CNAMEs to `[service]` where the resource is unclaimed. I registered the resource and served `[marker page]` at the subdomain — full content control on a `*.example.com` hostname."
- **Escalate:** the domain carries cookies scoped to `*.example.com` (session theft across all subdomains); it's referenced in emails/docs (phishing value); certificate issuance possible via the controlled host.
- **Do NOT claim:** takeover from a dangling CNAME you couldn't claim (reportable as informational at best); cookie theft without showing the cookie scope; severity past the demonstrated control.

## Rate limiting / brute force

- **Statement:** "`[endpoint]` has no effective rate limit on `[action: login / OTP / reset]`. I issued `[N]` requests in `[time]` without lockout or CAPTCHA — `[4-digit OTP space exhaustible in X]` / `[credential stuffing feasible]`."
- **Escalate:** the unthrottled action is OTP/reset (small space → practical bypass); account lockout exists but is bypassable (IP rotation shown at modest scale); pair with a credential-list attack only on your own accounts.
- **Do NOT claim:** brute-force success you didn't run to completion on your own account; "no rate limit" from a handful of requests (test past the threshold — with policy-permitted volumes only); missing CAPTCHA alone as a vuln.
