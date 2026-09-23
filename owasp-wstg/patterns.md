# Patterns — Reusable Heuristics from OWASP WSTG v4.2

Cross-cutting tactics the guide applies across all 12 categories. Per-test detail lives in `chapters/`.

## The master loop (every test)

```
Passive map (INFO) → pick test IDs → baseline request → single-variable probe
→ classify delta (status/length/time/error/state) → confirm minimal impact → report with WSTG ID
```

## Universal heuristics

- **Fingerprint first, probe second.** Almost every test opens with "identify the technology" — server (INFO-02), framework (INFO-08), DBMS (INPV-05), template engine (INPV-18), backend protocol (INPV-10). Version-specific attacks outperform generic fuzzing.
- **Baseline → anomaly.** Record the normal response's status, length, timing, and body shape before manipulating; every detection in WSTG is a deviation from that baseline (error strings, deltas, delays, content presence/absence).
- **Vary one variable.** Change exactly one parameter while others stay constant — that's how a vulnerable parameter is isolated from noise (INPV-05 detection discipline applies everywhere).
- **Three-probe identity checks.** Any verifier (login, recovery, existence) gets three probes: valid/valid, valid/invalid, invalid/invalid. Divergent responses = enumeration oracle (IDNT-04 pattern — also timing and redirect variants).
- **Two-session discipline.** Authz work needs same-role pair (horizontal) and cross-role pair (vertical): replay A's exact request under B's token and diff responses byte-for-byte (ATHZ-02/04, SESS-03/09).
- **Client state is a claim.** Hidden fields, disabled inputs, cookies, role flags, dropdown memberships — replay them modified. The server-side check, not the UI control, is what matters (BUSL-02/03, ATHZ-03, IDNT-01).
- **Trust boundaries are where logic breaks.** App→partner, front→backend, HTTP→mail server, page→worker: inject at handoffs — HPP (INPV-04), IMAP/SMTP (INPV-10), SSRF (INPV-19), business handoffs (BUSL-01).
- **Blind channels for blind bugs.** No reflection? Time it (sleep), phone home (OOB DNS/HTTP listener), or read the side effect (email sent, PDF rendered, error-page delta). Used by SQLi OOB, blind SSRF, padding oracle timing, incubated-recall listeners.
- **Encoding rotates past filters.** URL, double-URL, hex, unicode overlong (`%c0%af`), null byte (`%00`), case mixing, comments-as-whitespace — traversal and XSS filters die to encoding more often than to payloads (ATHZ-01, INPV-01, Appendix D).
- **Rollback is a test.** After any state-changing PoC (PUT upload, bucket write, planted cookie, placed order), verify the side effect then remove your artifacts — WSTG repeatedly says clean up your test objects.
- **Defense verification, not defense trust.** For each control, ask "validated where?": CAPTCHA server-checked? CSRF token tied to session? filter on all params or first occurrence? frame-busting on mobile too? Every defense has a canonical bypass listed in the chapters.
- **Misuse generates telemetry.** BUSL-07: while testing everything else, watch whether blatant attack sequences alter app behavior — no response to obvious misuse is itself a finding.
- **Architecture questions.** Firewall? reverse proxy? load balancer? app server? auth backend? Each detected component is a new trust boundary and a new test surface (INFO-10 → CONF-01).
- **Impact statement per ID.** Report shape: `[WSTG-XXX-NN] [what] → [demonstrated impact]` — "IDOR on /orders" is incomplete without "read another test account's invoice."

## Workflow patterns

- **Category order is an engagement plan.** INFO feeds CONF feeds identity/authz/session feeds injection feeds logic — later categories assume earlier maps exist.
- **Spider, then walk.** Automated spidering finds the shallow map; multi-step flows, POST-driven pages, and auth-gated areas need manual traversal (INFO-07).
- **Enumerate, then guess.** Naming-convention discovery (usernames, roles, file suffixes) turns blind fuzzing into dictionary attacks (IDNT-05, CONF-04, ATHN-02).
- **Rollback windows are targets.** Anything the app marks temporary (reservations, holds, pending transactions) — test what happens when it never completes (BUSL-04/06).
- **Record limitations.** What you couldn't test (out-of-scope hosts, no creds, broken features) goes in the report's limitations — an untested category is declared, not silently skipped.

## Anti-patterns the guide warns against

- Flagging version-banner CVEs without checking backported patches or exploitability (CONF-01 false positives both ways).
- Reporting "weak TLS protocol present" without real-world feasibility (most CRYP-01 attacks need nation-state MitM).
- Claiming enumeration/role values exist without demonstrating the switch or the delta.
- Fuzzing everything against every input — focus probes by expected input type (ERRH-01 guidance).
- Testing for `Secure`/HSTS on the main domain while ignoring subdomains that share Domain-scoped cookies (SESS-09 partial-HSTS trap).
