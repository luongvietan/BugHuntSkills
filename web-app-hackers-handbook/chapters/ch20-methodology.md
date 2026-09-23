# Ch21: A Web Application Hacker's Methodology

Source: Chapter 21. The end-to-end assessment loop: exhaustive mapping → systematic per-area testing → reporting. Coverage is the methodology — each area gets *all* its checks, because the one you skip is where the bug is.

## Phase 1 — Map & prepare

- Enumerate all content/functionality (spider + manual walkthrough at each privilege level); brute-force hidden content; mine public sources (Wayback, search, `robots.txt`).
- Catalog **every entry point**: params (URL, body, cookie, path), headers (Referer, UA, XFF, Host, custom), uploads, out-of-band channels.
- Fingerprint technology (banner, errors, extensions, cookies, URL structure) → derive default-content paths + likely bug classes.
- Map defenses: session mechanism, access-control points, input filters, WAF, rate limits.
- Build a checklist-driven workspace: each functional area → its tests → status.

## Phase 2 — Per-area testing (the master checklist)

- **Client-side controls**: tamper every client-transmitted value; bypass each client-side validation; decode opaque blobs; test extensions.
- **Authentication**: enumeration oracles; rate limits/lockouts; brute force (spray); multistage defects; recovery; remember-me; credential handling.
- **Session management**: token entropy/structure; predictability; fixation; cookie flags/scope; termination; disclosure channels.
- **Access controls**: two-account replay; forced browsing; ID cycling; multistage entries; method swaps; privilege params; Referer/IP checks.
- **Input-based injection**: per parameter — SQL, OS command, traversal, file inclusion, XPath/LDAP/NoSQL, XML/SOAP, HPI/HPP, SMTP/header injection, back-end request abuse.
- **Logic**: assumption list → param removal, stage skipping, cross-role params, numeric edges, state pollution, races.
- **XSS/user attacks**: reflection hunt by context; stored sweep (incl. admin surfaces, uploads, OOB); DOM source→sink audit; CSRF/clickjacking/redirects; local privacy.
- **Information disclosure**: provoke errors at every layer; debug artifacts; public sources; inference channels.
- **Native components**: length boundaries, integer edges, format strings on reachable parsers.
- **Architecture & server**: tier trust, shared hosting/vhosts, tenant discrimination, default content/creds, WebDAV, proxy, WAF bypass.

## Phase 3 — Escalate & chain

- For each finding: what does it unlock? (info leak → targeted attack; redirect → token theft; self-XSS → stored via chain; low-priv read → IDOR at scale).
- Map partial findings against each other — most critical compromises are 2–3 medium bugs composed.

## Reporting discipline

- Document per finding: location, mechanism, reproduction steps, evidence (minimal PoC), business impact, remediation. Separate verified findings from suspicions (note untested hypotheses — they still inform the retest).
- Preserve the test data matrix: what was tested, what couldn't be (time limits, access, scope) — coverage gaps are part of the deliverable.
- Recommendations should name the *mechanism* to fix (parameterize, server-side check, boundary validation), not the payload.

## Checklist

- [ ] Map complete before deep testing (but keep mapping — new content appears).
- [ ] Every checklist item per area marked done/skipped-with-reason.
- [ ] Two-account testing completed for access control and logic.
- [ ] Findings chained; each has minimal-PoC evidence.
- [ ] Coverage gaps + untested hypotheses documented in the report.
