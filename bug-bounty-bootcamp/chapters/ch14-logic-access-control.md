# Ch17: Application Logic Errors & Broken Access Control

Source: Chapter 17. The highest-bounty, least-scanned category: flaws in *what the app is supposed to do*, invisible to scanners because they're unique to the business.

## Mental model

Every feature encodes **business rules** ("a user can reset only their own password", "coupons apply once", "transfers can't exceed balance"). Logic bug = rule enforced in UI/business layer but not in the code path you can reach, or rules that contradict each other. You find them by *understanding the workflow, then violating its assumptions* — not by fuzzing payloads.

## Hunting method

1. **Map workflows end-to-end**: signup → verify → login → reset → purchase → refund → invite → roles. For each step ask: what is the app assuming?
2. **Attack the assumptions**:
   - *Step order*: skip verification steps; replay step 3 before step 2; submit final action without intermediate confirmations (price-before-confirm, skip CAPTCHA).
   - *Multi-use*: reuse reset links/invite codes/coupons; race them (Ch12).
   - *Tamperable state*: price, quantity, role, `user_id`, `is_admin` in request bodies (mass assignment) — flip values server never intended you to set.
   - *Numeric abuse*: negative quantities (refund more than paid), huge amounts (integer overflow wraps to negative/small), float rounding (`0.001` units), currency-mixing.
   - *Trust boundaries*: fields marked "server-set" (created_by, verified) that actually come from the request; client-side validation only.
   - *Password reset flow*: token in URL leaks via Referer to 3rd-party assets; token predictable/short; token not invalidated after use; reset lets you set `user_id`; host-header poisoning (`X-Forwarded-Host: evil.com` → reset link points at attacker domain → token capture = classic ATO).
   - *Signup/invite*: invite-yourself paths, role escalation via `role=admin` param, workspace/tenant confusion.
   - *OAuth/SSO flows*: state param CSRF, redirect_uri manipulation (chains with Ch7 open redirects), account-linking takeover.
   - *2FA/OTP*: brute-force no rate limit, response manipulation (`"success":false`→`true`), skip via direct endpoint hit, OTP in response body.
3. **Broken access control sweep** (distinct from IDOR — *function-level* not object-level):
   - Low-priv user hits admin endpoints directly (`/admin/users`, `/api/v1/admin/…`).
   - Method swap on restricted action (GET→POST→PUT→DELETE).
   - Forced browsing to hidden/internal paths from recon + JS files.
   - Role/permission params in registration or profile update (`admin=1`, `role=administrator`).
   - Auth required on page but not on its API (`/api/…` behind `/admin` page checks nothing).

## Escalation & first bug

Impact = business damage: free purchases, negative balance, mass account takeover via reset poisoning, privilege escalation to admin. Chains: logic flaw (invite yourself) → IDOR (read others) → data breach. **Checklist**: map each workflow → list its assumptions → violate one at a time → measure business impact → build minimal PoC walkthrough.
