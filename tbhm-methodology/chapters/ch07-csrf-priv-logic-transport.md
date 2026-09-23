# Ch7: CSRF, Privilege, IDOR, Transport & Business Logic

Source: `08_CSRF` + `09_Privledge_Logic_Transport` + `12_IDOR` (a stub — the real IDOR content lives in §9). These classes share one model: *who can do what, and how is it enforced* — mostly manual work scanners miss.

## CSRF quick-test battery

Run all eight on every state-changing request:

1. **Baseline** — does a normal request with no token protection replay cross-origin? (CSRF normal)
2. **Method swap** — force the action to GET-based; many token checks only exist on POST.
3. **Token = `undefined`** — literal string/null-ish values.
4. **Delete the token value or the whole parameter** — validation "only when present".
5. **Cross-account token** — use *your* valid token in a forged request against the victim.
6. **Same-length garbage** — replace the token with random characters of equal length (length-only check).
7. **Content-Type downgrade** — `application/json` → `text/plain` (or form-encoded) to slip past preflighted/parsed checks.
8. **Vulnerable-subdomain bypass** — mint the token/request from a sibling subdomain whose weaker controls the main app trusts.

## Privilege escalation testing

Logic, privilege, and auth bugs blur together. The TBHM model is deliberately simple: **admin has power, peon has none — can the peon use functions meant for admin?**

- Enumerate functionality restricted to certain user types (see common-functions list).
- Replay those functions as lesser/other roles — including *direct browsing* to sensitive views.
- **Autorize** (Burp plugin) automates the cross-role replay diff.

Common functions/views to cross-test: add-user, delete-user, start-project/campaign, change-account-info (password, CC), customer-analytics, payment-processing, and **any view containing PII**.

## IDOR — Insecure Direct Object References

Commonplace in bounties and hard for scanners to catch. Procedure:

1. **Find any and all UIDs** — numeric IDs, user hashes, emails embedded in requests/files.
2. **Rotate them**: increment, decrement, negative values, other accounts' identifiers.
3. **Attempt sensitive functions with a substituted UID**: change-password, forgot-password, admin-only functions.

High-value IDOR targets: everything in the CSRF battery re-tried cross-account; non-public images; receipts; private files (PDFs, exports); shipping info & purchase orders; sending/deleting messages. Rule of thumb — **any object reference the client supplies is a candidate**.

## Transport — HTTPS *everywhere*

Security-conscious sites enable HTTPS — your job is verifying they did it **everywhere**. They usually miss something:

- Sensitive images fetched over HTTP
- Analytics/beacons carrying session data or PII over HTTP
- Helper script (era): ForceSSL checks which resources still load insecurely; modern equivalent = Burp passive scan + mixed-content/HSTS analysis (preload list, `includeSubDomains`, cookie `Secure` flags).

## Business logic flaws — mostly manual

- **Substituting hashed parameters** — replay/forge "protected" values; weak or reused hashes are forgeable.
- **Step manipulation** — skip, reorder, or replay workflow steps (checkout → confirm without payment).
- **Negatives in quantities/amounts** — negative qty = credit; overflow boundaries too.
- **Authentication bypass** — direct object/state navigation past the login gate.
- **Application-level DoS** — single requests that trigger disproportionate work (regex bombs, expensive queries).
- **Timing attacks** — response-time deltas leaking valid users/tokens.

Two personas make all of this testable: keep a low-priv and an admin account; every sensitive function gets replayed across both (and across two same-priv accounts for IDOR).
