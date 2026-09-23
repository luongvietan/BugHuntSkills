# Ch4: Authorization & Session — the Quick Batteries

> **Ownership note:** session-fixation, CSRF-on-auth, and privilege-escalation batteries run on *your own two accounts* — never on discovered user data. Modern auth surface (JWT/OAuth/SAML) routes to `web-security-academy` ch07/08.

Source: `04_Authorization_and_Session`. Haddix's header says it twice: these checks "better be quick." Auth/session weaknesses are fast to test and frequently payout — run the battery early, before deep feature work. (Privilege escalation and transport get their own stage — see ch07.)

## Auth battery (authentication & account flows)

- **User/pass discrepancy flaw** — login error messages distinguish "bad username" vs "bad password" → account enumeration.
- **Registration page harvesting** — sign-up flow reveals whether an email/username is already registered.
- **Login page harvesting** — same enumeration via timing/message deltas on login.
- **Password-reset page harvesting** — "if this account exists…" done wrong; resets that confirm existence.
- **No account lockout** — unlimited password guesses → brute-force resilience test (throttle to program rules).
- **Weak password policy** — short/simple policies enable guessing and credential-stuffing fallout.
- **Password not required for account updates** — changing email/pass/CC without re-authenticating turns any XSS/session bug into ATO.
- **Password-reset tokens** — no expiry? Reusable after reset? Predictable? See ch08's n-minute battery for the full reset flow.

## Session battery (cookies & lifecycle)

- **Failure to invalidate old cookies** — capture a cookie, log out / rotate, replay the old one; if it still works, stolen cookies are forever.
- **No new cookie on login/logout/timeout** — session ID should rotate at every privilege change; fixed IDs → session fixation.
- **Never-ending cookie lifetime** — persistent sessions with no expiry window.
- **Multiple sessions allowed** — concurrent sessions mean an attacker's session survives the victim's logout/password change.
- **Easily reversible cookie** — cookie values that decode trivially (base64 is the classic tell) exposing user IDs, roles, or signed-with-nothing data.

## Workflow note

These batteries pair with the two-persona model: run them with two test accounts so you can immediately pivot a "yes" into a cross-account demonstration (ch07). Anything volumetric (lockout tests, reset spraying) needs program permission and rate throttling.
