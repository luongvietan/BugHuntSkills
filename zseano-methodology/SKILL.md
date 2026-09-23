---
name: zseano-methodology
description: "Knowledge base from \"zseano's methodology\" by Sean Roesner (zseano). Use when approaching a bug bounty target — program choice, first-look feature testing, filter/WAF bypass thinking, open redirect/OAuth chains, SSRF, uploads, IDOR, business logic, recon expansion, and long-term target methodology."
---

<!-- argument-hint: [vuln type, feature, or methodology step] -->

# zseano's methodology
**Author**: Sean Roesner (@zseano) | **Pages**: ~70 | **Generated**: 2026-09-23

## How to Use This Skill

- **Without arguments** — the core methodology below
- **With a vuln type** — `SSRF`, `IDOR`, `open redirect`, `CSRF`; I load the technique chapter
- **With a feature** — `registration`, `login`, `payment`, `developer console`; I give the question battery
- **With a phase** — `recon`, `second pass`, `automation`

When you ask about a topic not covered below, I read the relevant chapter file before answering.

---

## The Methodology (3 steps + meta-rules)

**Meta-rules:**
- **Question everything** — code takes params and executes; probe referenced *and* unreferenced ones (beta features = weak protection).
- **"Where there's a filter, there's a bypass"** — first pass hunts filters, not bugs. A filter fingerprints the dev's security model; same gaps repeat (XSS filter style predicts SSRF/upload/CORS filters).
- **"The trend is your friend"** — devs repeat mistakes; one bug class = site-wide hunt. Patches reveal how they think — read the fix, bypass it, sweep.
- **Months on one program** — wide scope, big names, more teams = more mistakes. Notes from day one → custom wordlists → treasure map.
- **Expect it to work as intended — verify anyway.** "false" is the new "true".

**Step One — manual first look:** mine disclosed bugs (Google/hacktivity/OBB) → walk key features with the question battery: registration, login/reset, account update, developer tools, main feature, payments. Test the bug classes you know best (XSS, CSRF, redirects, SSRF, uploads, IDOR, CORS, SQLi, logic) — the goal is mapping *how the site works*, not exhaustive coverage.

**Step Two — expand the surface:** dork (functionality keywords, file extensions, GitHub secrets, mobile UA, omitted results) → subdomain triage via `robots.txt` + functionality smell → Wayback for forgotten files → ffuf with stack-matched wordlists → param replay across endpoints (GET+POST) → **second pass reading every per-endpoint `.js` file** (comments, hidden endpoints, unreleased features).

**Step Three — rinse & repeat:** automate recon + diff-monitoring (.js, pages, subdomains, certs) → test features pre-release (`feature:false`→`true`) → rotate 5-6 wide-scope programs → patches and releases keep generating leads.

**Highest-yield chains:** OAuth whitelist + on-domain open redirect = token theft → ATO · SSRF filter + your 302 = internal read / AWS keys · mobile app first-launch request = stored XSS nobody watched · sandbox CC + new payment feature = verification bypass.

---

## Chapter Index

| # | Title | Key Content |
|---|-------|----------------|
| [ch01](chapters/ch01-mindset-program-notes.md) | Mindset, Program Choice & Notes | question-everything model, lead system, program selection, note→wordlist pipeline |
| [ch02](chapters/ch02-basic-toolkit.md) | Basic Toolkit | Burp, amass→httprobe→dnsgen→aquatone, ffuf, wordlists, Wayback/ParamScanner/AnyChanges |
| [ch03](chapters/ch03-common-issues-xss-csrf.md) | Common Issues — XSS & CSRF | 2-step filter mapping, WAF bypasses (param-name payload), blind XSS, referer tricks |
| [ch04](chapters/ch04-common-issues-redirect-ssrf-upload-idor.md) | Common Issues — Redirects/SSRF/Uploads/IDOR/CORS/SQLi/Logic | redirect filter payloads + OAuth chains, SSRF redirect bypass, upload matrix, GUID leaks, business logic |
| [ch05](chapters/ch05-step-one-first-look.md) | Step One — First-Look Playbook | disclosed-bug mining + question batteries for registration/login/reset/account/dev-tools/main-feature/payments |
| [ch06](chapters/ch06-step-two-attack-surface.md) | Step Two — Expanding Attack Surface | dorking, subdomain triage, robots+wayback, param replay, `.js` second pass |
| [ch07](chapters/ch07-step-three-findings-resources.md) | Step Three — Automation, Findings, Resources | what to automate, 10 case studies (redirect→ATO, SSRF→AWS keys, patch bypasses), resource list |

## Topic Index

- **Account takeover** → ch04, ch07
- **Business logic** → ch04, ch05, ch07
- **CSRF** → ch03, ch07
- **IDOR / BOLA** → ch04, ch05, ch07
- **OAuth / token leaks** → ch04, ch05, ch07
- **Open redirect** → ch04, ch07
- **Program strategy** → ch01, ch07
- **Recon / dorking / subdomains** → ch02, ch06
- **SSRF** → ch04, ch07
- **Uploads** → ch04, ch05
- **WAF / filter bypass** → ch03, ch04
- **XSS (reflected/stored/blind)** → ch03, ch05, ch07

## Supporting Files

- [glossary.md](glossary.md) — every tool, technique, and term
- [patterns.md](patterns.md) — reusable techniques: filter-first, encoding ladder, chains, replay sweeps, patch intel
- [cheatsheet.md](cheatsheet.md) — the loop, filter→bypass map, first-look battery, recon order

---

## Scope & Limits

Methodology and mindset, not a payload encyclopedia — pair with `xss-cheat-sheet` / `owasp-api-security-top-10` for payload depth. Some referenced tools age (amass/httprobe era) — the *workflow* is the durable content. For authorized testing only; follow program scope/rules.
