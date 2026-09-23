# Ch9: Cross-Site Request Forgery

Source: Chapter 9. CSRF = trick a victim's browser into sending an authenticated state-changing request. Preconditions: cookie-based auth + predictable request + browser sends cookies cross-site (SameSite posture).

## Hunting

1. **Inventory state-changing requests**: POST/PUT/DELETE endpoints that change email, password, payout info, roles, settings. Logout/login CSRF are lower impact but still reportable chains.
2. **For each, identify the defense**:
   - No CSRF token at all → trivially exploitable.
   - Token present but validation only runs *when the param exists* → drop `csrf_token=` entirely (the book's code-review example).
   - Token not tied to the session → swap in *your* account's token.
   - Token in a cookie that's also echoed in the body (broken double-submit) → set the cookie yourself.
   - Referer/Origin check → remove the header, or `https://target.com.attacker.com`, or check runs only when header *present*.
   - GET-based state change (`GET /change_password?new=…`) → no token needed at all; sensitive values leak into URLs/logs too.
3. **SameSite triage**: `SameSite=Strict` → CSRF dead on modern browsers; `Lax` → only top-level GET navigations carry cookies (GET-based changes still work); `None` → full CSRF surface. No SameSite attr → browser default (Chrome=Lax).
4. Build the PoC: auto-submitting HTML form (or `<img>`/fetch for GET) hosted on your test domain, victim logged in → state changes.

## Bypass tricks

- JSON POST blocked by CSRF token but the endpoint also accepts `application/x-www-form-urlencoded` → re-encode body as form.
- Content-Type check only → submit via `text/plain` fetch or a form with crafted `enctype`.
- Token in URL (`?token=…`) → leak it via Referer to attacker domain, then CSRF.
- Header-required (`X-CSRF`) → check for `X-Requested-With` CORS permissiveness or JSONP endpoint as alternate path.

## Escalation & automation

Value = which action you force: password/email change (ATO) > financial > settings > logout/login CSRF. Login CSRF + stored self-XSS = classic chain to real XSS. Escalate by chaining to OAuth flows (`/oauth/authorize` without confirmation). Burp Pro's "Generate CSRF PoC" and OWASP ZAP automate the form; the book's point — CSRF is usually reported fast, so hunt it on *every* state-changing endpoint, including API endpoints that accept cookie auth.

**First-CSRF checklist**: list state-changing requests → check token defense → check SameSite → attempt the 5 bypass families → build auto-submit PoC → pick the highest-impact action.
