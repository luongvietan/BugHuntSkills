# Ch7-8: Open Redirects & Clickjacking

Source: Chapters 7-8. Two low-solo-impact, high-chain-value bug classes — both usually *starter* vulnerabilities.

## Open redirects

Mechanism: the app redirects to a user-controlled destination. Value alone is limited (phishing assist), but it chains: **SSRF** (server fetches attacker URL), **OAuth token theft** (leak `code`/`token` to attacker domain), **referer/secret leakage**, **login-page phishing**.

**Hunting**:
- Grep requests/params for redirect semantics: `redirect`, `redir`, `next`, `return`, `returnTo`, `dest`, `destination`, `url`, `uri`, `u`, `n`, `forward`, `RelayState`, `continue`, `checkout_url`, `to`, `go`, `rurl`.
- Watch every 3xx — including referer-based redirects after login/logout/registration.
- Test `?next=https://attacker.com` and `/redirect?url=//attacker.com`.

**Bypassing allowlists** (this is where open redirects actually live):
- `https://example.com.attacker.com` (suffix check fail)
- `https://attacker.com/?next=example.com` / `https://attacker.com#example.com` (contains-substring check fail — the book's code-review example)
- `https://attacker.com\@example.com` and `https://example.com@attacker.com` (userinfo abuse)
- Scheme/case/encoding: `//attacker.com`, `/\/attacker.com`, `%2f%2f`, `attacker%E3%80%82com` (unicode dots), `javascript:` scheme when it's rendered client-side
- Backslash confusion `https://example.com\attacker.com` (browser parses `\` as `/`)

**First-redirect checklist**: enumerate redirect params → confirm control with `attacker.com` → characterize the validator (exact match? contains? suffix? scheme?) → apply the matching bypass family → escalate (OAuth leak / SSRF / phishing chain).

## Clickjacking

Mechanism: iframe the victim's sensitive page inside an attacker page; invisible/opacity-0 layers trick the user into clicking buttons they can't see.

**Hunting**: for each *state-changing* page (account settings, delete, transfer, OAuth consent), try framing it in a test HTML file. Check defenses:
- `X-Frame-Options: DENY|SAMEORIGIN` — missing? Only on some pages? `ALLOW-FROM` is obsolete/unsupported in Chrome+Firefox.
- CSP `frame-ancestors` — missing entirely? Only `*`?
- Frame-busting JS — bypassed via `sandbox` attribute on your iframe (blocks the busting script), double iframes, or `X-Frame-Options` checked only on top-level nav.
- Mobile/responsive layouts, drag-and-drop variant (drag data across frames).

**Impact & reporting reality**: most programs mark clickjacking N/A unless you show a *sensitive action* (delete account, change email, transfer money, OAuth authorize). PoC = HTML file, victim clicks once, account state changes. Always pair with the action's impact statement.
