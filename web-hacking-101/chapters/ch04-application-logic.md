# Ch 4 — Application Logic Vulnerabilities

The book's flagship chapter — no payload needed, just abusing *what the code allows*. Harder to find (no scanner), less crowded, high payout.

## The family

| Pattern | Mechanic | Canonical case |
|---------|----------|----------------|
| Mass assignment | Framework binds all submitted params to the model | Egor Homakov on Rails/GitHub — submitted `created_at` + SSH keys → repo access (2012) |
| Missing per-request authz | Replay API call after privilege removal | Shopify `POST /admin/mobile_devices.json` — remove perms, replay → still receives order notifications ($500) |
| Race condition | Two mutually-exclusive ops run near-simultaneously | Starbucks card transfer ×2 cURL → balance doubled ($0 but canonical) |
| Unguessable-looking but exposed id | Account id in iframe/URL treated as auth | Binary.com `PIN=CRxxxxxx` in iframe → any account, any action incl. PAYOUT ($300) |
| Unprotected new feature | Fresh code, zero scrutiny | HackerOne Signal — self-close reports → rep boost ($500) |
| Third-party service misconfig | S3/Zendesk/memcache treated as "not ours" | Shopify S3 ($1000), HackerOne S3 ($2500 — see below), PornHub memcache :60893 ($2500) |
| 2FA logic flaw | Second factor not bound to the account/job | GitLab — add `user[login]` to OTP POST → switch which account the OTP unlocks |
| Info disclosure | Debug files left in prod | Yahoo `phpinfo.php` found by scanning their /14 WHOIS range with a bash loop |
| Hidden endpoints | JS source exposes unreleased API paths | HackerOne hacktivity voting — grep minified JS for `POST` → vote endpoint live before launch (swag) |

## Techniques that produced these

- **Two accounts always**: known-good + victim. Replay A's requests as B; diff what's allowed.
- **Proxy everything**: Shopify privesc was literally "remove params, replay the same POST". Interception beats page-source guessing.
- **Read the JS**: devtools `{}` pretty-print on minified bundles; search `POST`, `DELETE`, `fetch(`, `/api/` → endpoints you were never shown. Blackbox → semi-whitebox.
- **S3 bucket hunting** (author's own $2500): bucket_finder-style script + wordlist of `<company>` permutations (`.com`, `-backup`, `-media`, `.marketing`, `.files`…). `Access Denied` on `ls` ≠ safe → try `aws s3 mv test.txt s3://bucket` — **readable-blocked but writable** is common. Verify ownership is the org's before reporting (global namespace).
- **Staging/dev bias**: `stage.*`, `dev.*`, CI servers — weaker configs than prod. Andy Gill: nmap `-sSV -p- -T4` on stage.pornhub.com → memcache open → `nc` connect, no auth → stats/dump (DoS, poisoned cache→XSS, data leak).
- **IP-range automation**: WHOIS the target's netblock (Yahoo owned a /14 = 260k IPs) → bash loop `wget` for known files (`/phpinfo.php`). Automation is the only way at that scale.
- **2FA checklist**: token lifetime, max attempts (rate limit), reuse of expired tokens, guessing feasibility, **binding** — does the OTP/submission carry or accept an account identifier?
- **New-feature radar**: subscribe to target blogs/changelogs — new code = unreviewed attack surface (Signal, Twitter intents, Facebook XSS cases all were fresh features).

## Meta-takeaways (author's own)

- "Don't underestimate your ingenuity and the potential for errors from developers" — even security companies misconfigure buckets.
- Don't quit at first failure: `ls` denied → try `mv`.
- Knowledge compounding: knowing S3 misconfigs existed → knew what to test.
- Race conditions: repeat the near-simultaneous request several times — first attempt often loses the race; respect traffic load.
- Unencrypted-looking values are invitations: `PIN=CRxxxxxx` plaintext vs the hashed password next to it — play with the plaintext one.
