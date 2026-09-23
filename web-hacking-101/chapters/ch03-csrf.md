# Ch 3 — Cross-Site Request Forgery (CSRF)

Victim's browser sends a state-changing request with their cookies — attacker just has to get them to load it. Modern frameworks auto-token; bugs live where tokens are absent, unchecked, or predictable.

## What to check on every state-changing request

1. Is there a CSRF token? Where — header, body param, meta tag?
2. Is it **validated** server-side? (Remove it / empty it / reuse it across accounts.)
3. Is it bound to the session/account? (Token from account A works for B?)
4. SameSite cookies? GET vs POST state changes (GET state changes are auto-vulnerable).
5. Third-party integrations (OAuth connect/disconnect, export/share features) — prime CSRF territory.

## Case patterns

- **Shopify export installed users**: trigger URL GET → victim admin's browser exports the user list to attacker-visible destination. *State change over GET = automatic CSRF.*
- **Shopify Twitter disconnect**: attacker-crafted request disconnects the victim's linked Twitter account — no token check on the disconnect action. *Takeaway: connect/disconnect social integrations are favorite CSRF targets — they change account state and are often added later than the auth system.*
- **Badoo full ATO**: CSRF on the password-change/email-change flow → silent account takeover. *The difference between "annoying CSRF" and "critical" is which state changes are forgeable — aim at email/password/2FA/payout actions.*

## Methodology

1. Inventory every state-changing request while mapping (POST/PUT/DELETE + GET actions).
2. Replay each without the token; then with a token from a second account; then cross-origin via auto-submit form / `<img src>` for GETs.
3. Rank by action sensitivity: email change > payout > settings > trivial.
4. Report impact as "attacker forces victim to X" — not "missing CSRF token".

## Note

Missing tokens aren't the only shape — a param that *acts* like a token (unguessable id per user) can defeat CSRF even without a labeled token. Confirm before reporting "no CSRF protection" (see ch10: confirm the vuln).
