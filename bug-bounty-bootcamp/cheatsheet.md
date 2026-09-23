# Cheatsheet — Bug Bounty Bootcamp

Quick-lookup card. Details in `chapters/`; mindset in `SKILL.md`; terms in `glossary.md`. Edition-era notes per chapter; modern routing + sources in `sources.md`.

## Session kickoff

1. Read program policy: scope, banned techniques, rate rules.
2. Walk the app manually at every privilege level (Burp logging on).
3. Recon: subdomains (crt.sh, Amass, gobuster dns) → live hosts → dir enum → screenshots → GitHub/pastes → tech fingerprint.
4. Build input inventory: params, headers, endpoints, uploads, APIs, hidden params.
5. Two test accounts: low-priv + high-priv. Burp scope set. Notes file open.

## Vuln-class quick tests

| Bug | First probe | Confirm |
|---|---|---|
| XSS | `XSS_TEST_123` marker → context → `<svg onload=alert()>` | context payload fires |
| Open redirect | `?next=https://evil.com` | `evil.com` loaded; try `//`, `@`, suffix, encoding |
| Clickjacking | frame sensitive page | no XFO/frame-ancestors + action works |
| CSRF | state-change req, no token | auto-submit PoC; try token-drop, referer-drop, method-swap |
| IDOR | replay A's object req as B | B reads/writes A's object; try phantom params |
| SQLi | `'`, `"`, `\` → error/delta | `OR 1=1` vs `OR 1=2`, `SLEEP(5)`, UNION count |
| Race | parallel-send limited action 10-20× | doubled effect / negative balance |
| SSRF | `?url=http://COLLAB` | inbound hit; then `127.0.0.1`, `169.254.169.254` |
| Deserialization | `O:`, `rO0`, `gASV`, `AAEAAAD` blobs | tamper field; PHPGGC/ysoserial + OOB |
| XXE | `<!ENTITY xxe SYSTEM "file:///etc/passwd">` | file in response; else param-entity OOB |
| SSTI | `{{7*7}}` → `49` | engine fingerprint → idiom RCE/OOB |
| Logic | skip/replay workflow steps | business rule violated (free, negative, escalate) |
| RCE | `;id`, `$(id)`, `` `id` `` | output/timing/OOB; minimal proof only |
| CORS | `Origin: https://evil.com` | reflected ACAO + credentials → readable |
| SAML | decode response, strip sig | accepted → modify NameID → login as other |
| OAuth | `redirect_uri=` variations | code/token lands on attacker domain |
| Info leak | traversal, `.git`, `.env`, `.map` | secret validated; chain to impact |

## Commands

```bash
gobuster dns -d target.com -w subs.txt          # subdomain brute
sort -u w1.txt w2.txt                            # merge wordlists
wfuzz -w paths.txt --hc 404 --follow http://t/FUZZ   # dir enum
wfuzz -w ids.txt "http://t/view?user_id=FUZZ"    # IDOR scan
wfuzz -w users.txt -w pass.txt --basic FUZZ:FUZ2Z http://t/admin
sqlmap -u "http://t/?p=1" --batch --level=3      # SQLi confirm
aws s3 cp TEST s3://BUCKET/ && aws s3 rm s3://BUCKET/TEST   # S3 write test
apktool d app.apk                                # unpack APK
adb devices -l && adb install app.apk            # Android
objection -g pkg explore → android sslpinning disable   # pinning bypass
```

## Bypass quick ref

- Filters: case mix, encoding (URL/hex/entities), comments `/**/`, `<scr<script>ipt>`, alternate tags/attrs, `data:` URIs.
- Redirect validators: `target.evil.com`, `evil.com?x=target.com`, `evil.com@target.com`, `//`, `\`, unicode dots.
- CSRF tokens: drop param, your-token-for-victim, cookie-echo double-submit, referer-absence.
- SSRF allowlists: `127.1`, decimal/hex/octal IP, `nip.io`, redirect-302, `file://`/`gopher://`, DNS rebind.
- JWT: replay, strip sig, `alg:none`, RS256→HS256 with public key, crack weak secret.

## Report skeleton

Title `[class] in [feature] at [endpoint] → [impact]` → summary → severity → numbered repro steps → impact analysis → remediation. Test accounts only. Minimal PoC. Platform channel only.

## When stuck

- Switch vuln class or subdomain — fresh surface.
- Old API versions, Wayback endpoints, archived JS.
- Read the JS/source harder — the endpoint you need is undocumented.
- Re-check your notes for unchained findings — two lows often make a high.
