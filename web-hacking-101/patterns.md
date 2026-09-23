# Patterns — Web Hacking 101

Distilled "if you see X → test Y" rules from the case takeaways.

## Recon signals

- `*.site.com` scope → subdomain enum first (KnockPy + enumall + ipv4info) → EyeWitness triage → nmap interesting hosts (esp. `stage.*`, `dev.*`, CI).
- WHOIS netblock → sweep IP ranges for known files (`phpinfo.php`, `.git`, admin panels) — automate it.
- New feature/changelog/blog launch → test immediately; fresh code is the least reviewed.
- Third-party services in use (S3, Zendesk, Shopify, Heroku) → each is an attack surface: dangling CNAMEs, bucket perms, interstitial redirects.
- Public GitHub org → GitRob/dork for keys, configs, internal hostnames.

## Input-flow signals

- Input renders in page → HTMLi/XSS probes; malformed HTML too (two src attrs, stray `=`).
- Input forwarded to another service/request (share links, unsubscribe, webhooks) → HPP: duplicate the param.
- Input lands in `Set-Cookie`/headers → CRLF `%0d%0a`; filtered → UTF-8/double-encode (`%E5%98%8A`, `%250d`).
- Param contains URL/path → open redirect + SSRF probes (`?url=`).
- URL param triggers server fetch → SSRF; bypass extension checks with `?`, `%00`, double slashes → aim at `169.254.169.254`.
- XML anywhere (body, `.docx/.xlsx/.gpx/.svg` upload) → XXE entity; blind → OOB DTD chain.
- `url=`/`image=`/`callback=` params → SSRF/redirect watchlist.
- File upload reaching converters (resize, docs) → library CVE checks (ImageTragick pattern).

## Logic signals

- Numeric/identifier params (`uid`, `PIN`, record ids in `/content/id` URLs) → swap, duplicate (HPP), `.json` suffix, other-account replay.
- Any action that "should be exclusive" (transfer, redeem, vote, claim) → fire near-simultaneous requests (race).
- 2FA/OTP flows → add `user[login]`-style account params; test lifetime, attempts, reuse, binding.
- Feature gated by UI but API reachable (mobile app calls `admin/*.json`) → replay via proxy after removing perms.
- Value looks unencrypted next to encrypted siblings → play with it.
- Paywalled/premium features → fewer eyes; subscribe and test if in scope.

## Filter behaviors

- Client-side validation exists → server probably trusts input → bypass with proxy; flag the field.
- Char blocked → encode (UTF-8 overlong, double-encode, null byte, case).
- Sanitizer "fixes" malformed HTML → feed it malformed HTML deliberately; the fix is the bug.
- First attempt fails → duplicate the param / second account / different event path (keyboard vs mouse) / different render surface (email vs page).

## Reporting patterns

- `alert()` finds the bug; impact narrative sells it (session theft, ATO, what it does to THEIR users).
- Confirm first: fresh browser/version, second run, actual security boundary crossed — "don't shout hello".
- Rejected ≠ wrong: re-verify, video PoC, explain the exploit path, stay polite — or chain it further.
