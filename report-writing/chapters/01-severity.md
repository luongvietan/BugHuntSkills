# Severity — Rubric, Platform Bands, CVSS

Severity = **impact × exploitability**, argued with evidence. Score what you proved, not what the bug could theoretically become. Triagers re-score every report; a defensible medium beats an inflated critical that gets cut — credibility compounds across reports on the same program.

## Step 1 — Grade the impact (what the attacker gains)

| Impact tier | What it looks like | Examples |
|---|---|---|
| **Severe** | Full compromise of confidentiality/integrity at scale, or direct financial loss | RCE on a production host; auth bypass → arbitrary account takeover; SQLi reading the full user table; SSRF reaching cloud metadata / internal admin panels |
| **High** | Compromise of one user's account or sensitive data without their help, or broad-but-shallower exposure | Stored XSS on a main-domain page; IDOR exposing other users' PII records; CSRF changing account email/password; subdomain takeover |
| **Moderate** | Requires victim interaction, limited data, or one-step-removed impact | Reflected XSS needing a click; CSRF on a sensitive-but-recoverable action; IDOR on low-sensitivity objects; session fixation |
| **Low** | Security-relevant but weak standalone impact; hardens posture more than it enables attack | Open redirect (no token capture shown); clickjacking on low-sensitivity page; verbose error disclosure; missing rate limiting on login |

Scale multiplies within a tier: "read any user's invoices" (all users) sits above "read one user's invoices" (cross-account only). **Only claim scale you actually demonstrated** — two test accounts prove the mechanism; the word "any" claims the population, which is fair *if* the mechanism has no scoping.

## Step 2 — Grade exploitability (cost to the attacker)

| Factor | Raises severity | Lowers severity |
|---|---|---|
| Authentication | None needed (pre-auth, internet-facing) | Requires a valid account — especially a paid/privileged one |
| User interaction | None — fires on view/visit | Requires click, then social engineering, then specific browser |
| Reliability | Deterministic, single request | Race condition won once in 50 tries, specific timing |
| Preconditions | Default config, all users affected | Non-default setting, feature-flagged, deprecated endpoint |
| Reach | Works cross-browser/OS, API + web both hit | Narrow client dependency (old IE only, one mobile OS) |

## Step 3 — Combine into a band

| Impact ↓ / Exploitability → | Trivial (no auth, no interaction, reliable) | Moderate (auth OR interaction) | Hard (multiple preconditions) |
|---|---|---|---|
| Severe | Critical | High | High–Medium |
| High | High–Critical | High | Medium |
| Moderate | Medium–High | Medium | Low–Medium |
| Low | Low–Medium | Low | Informational |

Sanity-check the result against the class anchors:

| Vuln class | Typical landing zone | Pushes up | Pushes down |
|---|---|---|---|
| RCE / SQLi w/ data read | Critical–High | Pre-auth, prod host, sensitive schema | Auth-required, read-only user, blind-only |
| SSRF | High–Critical | Cloud metadata hit, internal admin, port-scan proven | Only reaches itself, no internal surface shown |
| Account takeover (any path) | Critical–High | Pre-auth, no interaction | Needs victim click + rare condition |
| Stored XSS (main domain) | High | Auth/admin context, no interaction, wide page reach | Low-traffic page, self-only |
| IDOR | High–Medium | Sensitive object (PII, financial), all-users scale | Non-sensitive object, UUIDs unguessable |
| Reflected/DOM XSS | Medium–Low | Auth bypass via token capture shown | Needs click + modern-browser mitigations apply |
| CSRF | Medium–Low | Account state change (email/password/2FA) | Logout/search CSRF, token partially validated |
| Info disclosure | Medium–Low | Keys/PII in the leak | Stack trace, version banners only |
| Open redirect / clickjacking | Low–Info | Chained into OAuth token theft / cred-harvest flow | Standalone, no chain shown |
| Missing best-practice headers | Informational | Rarely — needs demonstrated exploit | Almost always informative |

## Step 4 — Platform bands

Programs score differently; pitch to the one you're on.

- **HackerOne** — CVSS v3.x calculator drives severity (and often bounty tables). Fill the calculator honestly and paste the vector + score into the report. Common anchors: 9.0–10.0 critical, 7.0–8.9 high, 4.0–6.9 medium, 0.1–3.9 low.
- **Bugcrowd** — VRT (Vulnerability Rating Taxonomy) maps vuln class + context → priority P1–P5. Look up your class in the VRT *first* and cite the category in the report; VRT already bakes in typical severity, so argue only deltas (e.g., VRT lists stored XSS → P1 baseline).
- **Intigriti / YesWeHack / others** — CVSS-based with triager judgment; include vector + a one-line justification per metric you chose.
- **Private/VDP programs** — plain-language bands; map your finding to their published table and quote the row you match.

### CVSS quick guidance

Score in this order: **AV** (Network unless it truly needs local access) → **PR** (None/Low/High = accounts needed before exploiting) → **UI** (None vs. Required — a link click counts) → **S** (Changed only if it escapes a security authority, e.g., stored XSS reaching another user's session) → **C/I/A** (what the attacker actually reads/changes/kills — your demonstrated impact, not theoretical max).

Common mistakes that get scores cut: claiming `C:H` for a bug that reads one field; `S:C` for a same-app bug; `PR:N` when the endpoint requires login; `A:H` because "it could DoS" you never tried (and shouldn't). Keep the vector honest — you can link a public calculator with your vector pre-filled.

## Argue up vs. argue down

**Argue UP when you have new evidence, not adjectives:**
- You can show a chain: self-XSS + CORS leak + CSRF = account takeover. Present the chain as ONE finding at the chain's severity.
- You demonstrated scale: enumerable IDs, no rate limit, count of affected records (from your own data).
- A precondition the triager assumed isn't real: "requires admin" → you show a default-role account reaches it.
- Sensitive data class confirmed: the field isn't "data", it's session tokens / PII / payment details.

**Argue DOWN (yes, deliberately) when:**
- The platform's own rubric lands lower than your ask — accepting their band with a note ("happy to align to program taxonomy") builds triager goodwill that pays on the next report.
- Your interaction claim is shaky: if the attack needs the victim on a specific browser, say so before they find it.
- The bug is a chain-link you couldn't finish: report it at its standalone severity with the chain hypothesis labeled as such — an honest "medium with a path to high" survives triage better than an unproven critical.

**The credibility ledger:** every report on a program teaches the triager your batting average. Hunters who over-claim get their next critical assumed-inflated; hunters who self-correct get the benefit of the doubt on borderline calls.

## Evidence line that always helps

End the severity section with one sentence tying the band to the program's own language: *"Under [program]'s severity table, this matches [band] — [one-line reason]."* You're not asking them to agree with you; you're showing them where the report already fits.
