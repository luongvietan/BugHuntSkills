---
name: tbhm-methodology
description: Jason Haddix's "The Bug Hunter's Methodology" (TBHM) — the classic crowdsourced bug-hunting framework from the "How to Shot Web" lineage. Use when approaching a wide-scope target (`*.acme.com` style), building a discovery/recon stack (OSINT domain enumeration, acquisitions, full-port scans), mapping an application (smart directory brute-force, platform identification, status-code escalation), or doing tactical testing (XSS/SQLi polyglots, LFI/uploads/RFI-redirects, CSRF token games, privilege/IDOR/logic/transport bugs, mobile storage, auxiliary "noise" vulns) — plus the WAHH-derived fast-testing checklist for time-boxed assessments. Verify asset scope and method permission before live use.
---

# The Bug Hunter's Methodology — Jason Haddix (@jhaddix)

Knowledge base distilled from Haddix's TBHM repo/talk series (v1–v4 era). TBHM's signature is a **stage model tuned for crowdsourced bounties** — where you're racing other hunters, paid on impact not volume, and the flagship app is already picked clean:

**Philosophy → Discovery → Mapping → Tactical Testing (XSS · SQLi · FI/Uploads · CSRF) → Privilege/Logic/Transport → Mobile → Auxiliary**

Mental model: on a wide scope, don't hunt where everyone hunts — find the road less traveled (forgotten subdomains, acquisitions, odd ports, mobile sites), map it smarter than the next hunter (stack-matched wordlists, status-code escalation), then run fast tactical batteries per feature class.

Related skills: `recon-pipeline` (runbook for standing up the recon stack), `zseano-methodology` (months-on-one-program depth play), `bug-bounty-bootcamp` (per-vuln mechanism→bypass loop), `bug-bounty-playbook` (aggressive tooling variant), `xss-cheat-sheet` + `payloads-all-the-things` (payload depth), `owasp-wstg` (full checklist authority).

## How to use

**Program & workflow**
- Bounty economics, competition, report templates → `chapters/ch01-philosophy-bounty-model.md`

**Before touching a single app (wide-scope targets)**
- Domain/OSINT discovery, acquisitions, full-port scans → `chapters/ch02-discovery-recon.md`
- Smart directory brute-force, platform ID, OSINT vuln history → `chapters/ch03-mapping-enumeration.md`

**Tactical testing per feature**
- Auth & session quick batteries → `chapters/ch04-auth-session.md`
- XSS & SQLi polyglots, input vectors → `chapters/ch05-tactical-fuzzing-xss-sqli.md`
- LFI, malicious uploads, RFI/open redirects → `chapters/ch06-uploads-lfi-redirects.md`
- CSRF, privilege escalation, IDOR, transport, business logic → `chapters/ch07-csrf-priv-logic-transport.md`

**Wrap-up & speed runs**
- Mobile storage, "noise" vulns, the n-minute data-driven assessment + full task checklist → `chapters/ch08-mobile-aux-checklist.md`

- Terms → `glossary.md`; reusable heuristics → `patterns.md`; quick ref → `cheatsheet.md`

## The TBHM stage model

| Stage | Question it answers | Core output |
|---|---|---|
| 1. Philosophy | What game am I playing? | Crowdsourced mindset: unique bugs > common bugs; impact = payout |
| 2. Discovery | Which assets are least tested? | Domain inventory: subdomains, acquisitions, port-scan surface |
| 3. Mapping | What does the app expose & run on? | Content map + tech stack + known-CVE list |
| 4. Tactical fuzzing | Does this feature take input/display data? | Fast probes per class: polyglot → context → targeted payloads |
| 5. Priv/Logic/Transport | Who can do what, and how is it enforced? | Cross-account diffs, IDOR rotations, HTTPS gaps, logic abuse |
| 6. Mobile | What does the app store/send? | Unencrypted PII at rest, storage/log leakage |
| 7. Auxiliary | What's left that's still reportable? | "Noise" vulns + the n-minute assessment for diminishing returns |

## The Haddix doctrine (mental models)

1. **The road less traveled wins.** In crowdsourced bounties the flagship app is heavily assessed — your edge is obscure surface: `^.acme.com` scopes, acquisitions, forgotten subdomains, weird ports, mobile sites, redesigns.
2. **Port scanning is not just for network pentests.** A `-p-` sweep on web scope finds admin panels, stray services, and unauthenticated consoles nobody else looked at.
3. **80/20 tactical fuzzing.** Time-boxed testing runs on polyglots — one multi-context string first, then targeted payloads only where it lands. Ask per feature: "does it display to users?" (XSS) / "does it call stored data?" (SQLi) / "does it touch the filesystem?" (LFI/upload).
4. **Status codes are a map.** 401/403 dirs from brute-forcing aren't dead ends — recurse into them for misconfigured access control.
5. **Quick batteries for auth & session.** Auth/session bugs "better be quick" — run the short checklist (user enum, lockout, token reuse, cookie invalidation) before deep work.
6. **Privilege bugs need two personas.** Peon vs. admin: replay restricted functions cross-role, rotate every UID (increment/decrement/negative), check non-public files. Autorize automates the diff.
7. **HTTPS everywhere or it doesn't count.** Sensitive images, analytics with session data — hunt what they forgot to encrypt in transit.
8. **Templates for repeat findings.** Custom bugs get custom writeups; your frequent classes get pre-built report templates — and always swap in the right domain/URLs before submitting.

## Scope & ethics

Before live testing, verify the exact asset and technique against current program rules. This skill grants no authorization; if scope or permission is missing or unclear, stop and re-check with the program. TBHM is a competition-shaped methodology — speed matters, but scope is the hard boundary: verify `^.acme.com`-style wildcard scope and acquisition rules in the program policy before touching new surface, throttle port scans and brute-forcing to avoid DoS, use test accounts for cross-account checks, and stop at the minimal proof of impact. Several referenced tools/sources (xssed, Punkspider, Recon-ng-era stacks) are dated — the *workflow* is the durable content; substitute modern equivalents (`recon-pipeline` skill) as needed. Source/version/review metadata: `sources.md`.
