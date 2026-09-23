# Ch8: Attacking Access Controls

> **Edition note (2011):** fully durable — the trust-boundary model maps
> 1:1 onto modern IDOR/BOLA/BFLA. For API object/property authorization use
> the 2023 taxonomy in `owasp-api-security-top-10` (BOLA + BOPLA + BFLA).

Source: Chapter 8. Access control = logic deciding whether a request may proceed given the authenticated identity. Broken when users access resources/actions outside their privilege — **vertical** (user→admin) or **horizontal** (user→user). The core flaw: the app trusts something user-controllable (URL, parameter, stage reached, Referer, IP) to indicate authority.

## Mechanism → canonical failure points

- **Unprotected URLs / forced browsing**: admin functions reachable by URL knowledge alone — `/admin`, `/manager`, function names in JS. Low-priv user simply requests them.
- **Identifier-based access**: object references (`docid=`, `account=`, `uid=`) used directly without ownership check — the classic IDOR. Cycle/guess IDs; try other users' identifiers in *every* object-loading function (view, edit, delete, download).
- **Multistage workflow**: strict check at stage 1, none at stages 2+ ("anyone who got here must be authorized"). Skip to later stages directly; also reverse — replay stage-1 checks with stage-3 actions.
- **Static files**: ACL enforced on dynamic pages but not on the static content they serve (`/reports/q3.pdf` requested directly).
- **HTTP method confusion**: check on GET only; same action via POST/PUT/DELETE/HEAD unprotected (and vice versa). Platform configs (`<http-method>` restrictions) routinely get this wrong.
- **Parameter-based privilege flags**: `admin=1`, `role=`, `isadmin`, `access=` — parameters the server honors but UI never sends. Try adding them everywhere (incl. registration/profile update — mass-assignment style).
- **Referer-based controls**: "admin pages only from admin menu" checked via Referer — forgeable per request.
- **IP/geolocation controls**: `X-Forwarded-For`/`X-Real-IP` spoofing where IP checks read proxy headers; geo checks bypassed via proxy in allowed region.
- **Location-based**: controls on the *menu/page* not on the action's endpoint or its API.

## Two-account methodology (the core technique)

1. Use **two accounts at different privilege levels** (ideally + an unauthenticated session).
2. Walk all functionality as high-priv; log every request.
3. Replay each request as low-priv (and as anonymous): same params, then minus cookie, then with low-priv cookie.
4. Compare: identical effect = broken control. Partial/different response may still allow the sensitive part — check state change server-side, not just the response.
5. Horizontal: as user B, request user A's resources via every discovered identifier — including IDs harvested from A's session that never appear in B's UI.
6. For multistage flows: complete as high-priv, then as low-priv jump directly to each later stage.
7. Try adding privilege-flavored params (`admin=true`, `role=x`) at each tier; try method swaps on every restricted action.

## Where access checks hide

- Central filter/dispatcher vs per-function checks vs front-controller rewrites — locate it; per-function checks get forgotten (new endpoints, API twins of checked pages, alternate content-types reaching same handler).

## Checklist

- [ ] Every high-priv request replayed as low-priv and anonymous; deltas catalogued.
- [ ] Every object identifier cycled cross-user.
- [ ] Every multistage flow entered at each stage directly.
- [ ] Static resources fetched directly, bypassing page-level checks.
- [ ] Method swaps tried on restricted endpoints.
- [ ] Privilege params injected; Referer/IP-based checks forged.
- [ ] API endpoints checked for parity with their page-level controls.
