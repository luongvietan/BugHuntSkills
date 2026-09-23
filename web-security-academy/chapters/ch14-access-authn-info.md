# Ch14: Access Control, Authentication & Information Disclosure

Sources: `/web-security/access-control` (+ `/idor`), `/web-security/authentication` (+ `/password-based`, `/multi-factor`, `/other-mechanisms`), `/web-security/information-disclosure`. The "who may do what" cluster — mostly overlap with `bug-bounty-bootcamp` but the Academy's URL-matching/platform-misconfig section is sharper.

## Access control — mechanism

Three axes: **vertical** (user→admin), **horizontal** (user→peer user), **context-dependent** (must complete prior steps). Broken access control = the server never re-checks authority after authentication.

## Broken access control patterns

- **Unprotected functionality**: `/admin`, `/administrator-panel`, robots.txt/`/.git`-leaked admin URLs; admin link hidden in UI but reachable directly; JS files revealing endpoints.
- **Parameter-based control**: `?admin=true`, `role` in cookie/JWT/profile fields honored server-side; hidden params (`admin`, `isAdmin`, `debug`) accepted but never shown — sniff parameter conventions from responses/JS.
- **Platform misconfiguration**: `X-Original-URL`/`X-Rewrite-URL` headers — front-end blocks `/admin` but honors `X-Original-URL: /admin` on a `/` request (framework reads the override header, not the path).
- **URL-matching discrepancies**: case (`/ADMIN` vs `/admin`), trailing slash/suffix (`/admin/`, `/admin.anything`, `/admin%2f`), normalization differences between front-end denylist and back-end router — same seam as cache deception (`ch03`).
- **Method/verb confusion**: `POST /admin/delete` denied but `GET`/`PUT`/other verbs route through; verb-override headers/params.
- **Referer-based access**: page checks `Referer: /admin` — forge it.
- **Multi-step processes**: step 1 validates, step 2/3 trusts — skip straight to the final request.
- **Location-based**: IP-gated admin bypassed via `X-Forwarded-For`/direct back-end URL.
- **IDOR** (horizontal): sequential/GUID object IDs — change `id`, document GUIDs leak via other features, `id` in POST body/cookies; try phantom params (`user_id`) when none appear; user-controlled keys in filenames/import references.
- **Horizontal→vertical escalation**: read another user's object that *contains* privileged data (admin's leaked own profile = pivot to vertical).

## Authentication — mechanism

Three factors (knowledge/possession/inherence) and the classic attacks on each:

- **Username enumeration**: different responses/timing on valid vs invalid user (`Invalid username` vs `Invalid password` strings, response-length deltas, timing when password is validated only for real users → longer response). Fuzz with a fixed wrong password; sort by status/length/time.
- **Brute-force/credential stuffing**: no lockout/rate-limit → Intruder/Turbo Intruder on `auth-lab-usernames`/`auth-lab-passwords` conventions; IP-rotation via `X-Forwarded-For` when IP-blocked; account lockout bypass via intermittent valid-login resets (flawed counting).
- **Password-based flaws**: weak storage (leaked hashes → crack), default credentials, password-spraying (one common password × many users beats lockout).
- **2FA/MFA flaws**: 2FA code verification not bound to the login step — skip straight to post-login pages after step 1; brute-forceable 4-6 digit codes (race before expiry — see `ch15`); code logged/leaked in responses; `verify` page missing rate-limit.
- **Other mechanisms**: "remember me"/"stay logged in" cookies = weak crypto (XOR/base64 of `user:timestamp` — decode, tamper, re-encode); password-reset flows with tokens in URLs that expire poorly, accept any token, or poisonable via Host header (`ch03`); password-change missing current-password check → CSRF-account-takeover; multi-account per session confusion.

## Information disclosure

- **Sources**: error/stack traces (debug pages), comments/source maps, backup files (`index.php~`, `.bak`, `.git`, `.svn`, `/.env`), directory listings, version headers (`Server:`, `X-Powered-By`), `TRACE`/OPTIONS, admin/debug endpoints, robots.txt/sitemap leaks, hardcoded keys in JS, API keys in repos, verbose OAuth/JWT claims.
- **Method**: fuzz for hidden content (backup extensions, `.` dirs), trigger errors deliberately, diff responses across roles, grep client-side code for `key`/`token`/`password`/`secret` patterns, check Wayback for removed-but-cached content.
- **Severity rule**: a leak is only reportable when it *enables* something — `/.git` → source → hardcoded creds → ATO is a report; `Server: nginx/1.18` alone is informational. Always finish the chain or downgrade honestly.

## Lab reference

`https://portswigger.net/web-security/all-labs#access-control-vulnerabilities` · `#authentication` · `#information-disclosure`
