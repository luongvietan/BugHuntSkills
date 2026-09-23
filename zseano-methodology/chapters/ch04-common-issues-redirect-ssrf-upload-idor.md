# Common Issues I Start With — Redirects, SSRF, Uploads, IDOR, CORS, SQLi, Logic

> **Ceiling + taxonomy note:** redirects → own-domain destination; SSRF → own listener / harmless metadata read only (`hacking-the-cloud` gates); uploads → harmless file you own; IDOR → own two accounts, now named BOLA/BOPLA (`owasp-api-security-top-10`); SQLi → differential proof. Escalation chains are report narrative.

## Core Idea
Same playbook as XSS: every vuln class is a filter-hunting exercise. Find the feature that *should* be protected, map its filter, bypass it, then chain the "harmless" result into impact (redirect→token leak, SSRF→AWS keys, upload→RCE).

## Open URL Redirects

- **Why hunt them**: near-100% chain value when the site has OAuth — `redirect_url` whitelists `*.target.com`, so an open redirect *on* target.com smuggles the auth token to you.
- **Param names to dork** (upper/lower case): `return, return_url, rUrl, cancelUrl, url, redirect, follow, goto, returnTo, returnUrl, r_url, history, goback, redirectTo, redirectUrl, redirUrl`.
- **Encoding the chain**: parameters get dropped across multiple redirects — URL-encode `? & # / \` (sometimes double-encode) so the *browser* decodes them at the right hop:
  `Location: /redirect%3Fgoto=https://evil.com/%253Fexample=hax`
- **Filter-bypass payload set**: `\/evil.com`, `//evil.com`, `\\evil.com`, `//theirdomain@evil.com`, `https://evil.com%3F.theirdomain.com/`, `https://theirdomain.computer/`, `/%0D/evil.com` (+`%09 %00 %0a %07`), `//%2F/evil.com`, `////evil.com`, `/%5Cevil.com`, `//google%E3%80%82com` (ideographic full stop).
- **JS redirects**: `Location:` header kills XSS, but `window.location` redirects accept `javascript:` — bypass filters with `java%0d%0ascript%0d%0a:alert(0)`, `jjavascriptavascript…` char-stuffing, `java%07script:`, `java%09scrip%07t:`.

## SSRF

- **Where**: any feature taking a URL — webhooks, API consoles, import-from-URL, `url=` params (found one on Yahoo that way).
- **Redirect handling is the filter gap**: host `<?php header("Location: ".$_GET['url']); ?>` on XAMPP+ngrok → does the server follow? Filter checks the *input* but not the *redirect destination* → internal read. Add `sleep(1000)` before redirect → timeout bugs. Chain discovered open-redirects when external URLs are blocked.
- Also: third-party software (Jira etc.) with known CVEs.

## File Uploads (stored XSS → RCE)

- **Expect a filter; probe its axis**: try `.txt .svg .xml` first (forgotten types), then `.png/.gif/.jpg` to see normalization (all saved as `.jpg`? trusting extension?).
- **Filename tricks**: `zseano.php/.jpg` (validator sees `.jpg`, server writes `.php`), `zseano.html%0d%0a.jpg` (CRLF cuts the name), no filename/extension at all, `.html` with `Content-Type: image/png`, XSS in the filename itself (`58832.jpg<svg onload=confirm()>`).
- **Content tricks**: keep `‰PNG` magic bytes + `<script>` body — validators that only sniff headers pass it; mismatched `Content-Type: text/html` vs extension tells you what's trusted.

## IDOR

- **ELI5**: change the integer (`/user/1`→`/user/2`); also works on GUIDs *when leaked* — find GUIDs in public pages (`/images/users/{guid}/photo.png`), Google index, source code.
- **Try integers against encrypted-looking IDs** — "security through obscurity"; servers often process `1` the same.
- **Inject the param**: JSON bodies — add `{"id":"1"}` especially on `PUT`; HPP on reset flows.
- **Mobile APIs first** — highest IDOR hit-rate (mobile app calls API with just your user ID).
- **Escalate laterally**: no ownership check on objects → check role boundaries (guest→admin actions, free→paid features). "If they aren't checking I own the ID, what else did they forget?"

## CORS

- `Access-Control-Allow-Origin` on response (+`Allow-Credentials:true` when cookies needed) → send `Origin: anythingtheirdomain.com` — substring checks pass attacker-owned domains.
- Habit: add `Origin: theirdomain.com` to **every** request, grep responses for `Access-Control-Allow-Origin`. Harmless-looking endpoint bypasses get reused later.

## SQL Injection

- Legacy code first. Skip `'`-error probing (errors are usually off) — go straight to **time-based**: `' or sleep(15) and 1=1#`, `' or sleep(15)#`, `' union select sleep(15),null#` (15-30s window).

## Business/Application Logic

- **No payload — just misuse**: loan max £1,000 → set £10,000; "Coming soon" premium feature → is it actually unreachable?
- **New-feature × old-feature seams**: new upgrade flow requiring only payment data bypassed phone-verified ownership — sandbox/test CC numbers weren't blacklisted (test cards from worldpay/paypal docs).
- **Email-domain privileges**: sign up with `@target.com` — sometimes whitelists verification or grants staff features (no verification done).
- **Source**: API docs, "privacy" toggles (is that post *really* private?), role levels (guest→moderator→admin API calls).

## Anti-patterns

- **Dropping redirect params unencoded** — they vanish at the first hop.
- **Testing upload extension only** — filename, content-type, magic bytes, and path-parsing bugs are all separate axes.
- **GUIDs = unattackable** — find where they leak before giving up.
- **Ignoring "harmless" finds** — open redirect + OAuth flow = token leak = account takeover.

## Key Takeaways

1. OAuth login + open redirect on whitelisted domain = token exfiltration = ATO.
2. SSRF filters validate input, not redirect targets — always test redirect-following.
3. Upload tests = matrix of {extension, filename parsing, content-type, magic bytes}.
4. On IDOR: `GET`→`POST` switches can error-leak the "hash" they added as a fix.
5. Business logic pays best with least noise — understand the intended flow, then violate one assumption.

## Connects To

- **ch03**: same filter-hunting process applied to XSS/CSRF
- **ch05**: features where these get aimed (login redirect params, developer consoles, payment)
- **ch07**: case studies — 30+ redirects→token leak, SSRF redirect→AWS keys, sandbox-CC bypass
