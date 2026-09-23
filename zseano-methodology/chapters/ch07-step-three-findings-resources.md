# Step Three — Rinse & Repeat, Automation, Case Studies & Resources

> **Currency note:** automation and monitoring cadence is durable; referenced tools and case-study payouts are era-marked. Continuous recon with diff triage lives in `recon-pipeline` ch03; severity/reporting norms in `report-writing`.

## Core Idea
Months in, you hold a complete mental map + notes. Now: automate the repeatable recon, monitor for change, rotate across 5-6 wide-scope programs, and let patches/new-features generate your next leads. Real findings below show the loop paying out.

## Automate These Two Things

1. **Subdomain/file/directory/leak scanning** — full pipeline on autopilot: amass→httprobe→ffuf→GitHub-dorks; CertSpotter for new certs; LazyRecon-style orchestration. Manual recon wastes hacking time.
2. **Change detection** — watch pages + `.js` files for new links/code; new functionality = new bugs. Spot `code present, feature off` → try enabling (`true`/`false`).

Also: follow @disclosedh1 + program "Updates" — new scope = new bugs.

## The Lifestyle Loop
5-6 wide-scope programs in rotation → always something fresh; big companies ship changes constantly → mistakes constantly. "Make bug bounties work for you."

## Case Studies (findings → the lesson)

**30+ open redirects → token leak → ATO.** Dorked redirect params en-masse; OAuth login whitelisted `*.target.com`; chained redirect → auth token to attacker; wiki docs explained token usage → P1. Fixed one redirect, flow stayed broken → re-exploited multiple times. *Lesson: fix the chain, not the link — and keep testing their patches.*

**Mobile-app stored XSS on a heavily-tested program.** First-ever app request (GDPR consent) carried `returnurl` → `"><script>alert(0)</script>` stored. One-time requests are invisible to hunters who proxy late. *Install the app, watch request #1.*

**IDOR → patch analysis → site-wide pattern.** `api/user/1` IDOR patched with a "hash" requirement; switching GET→POST caused an error leaking the hash anyway. Devs code the intended method only → same flaw class across the app; some programs fix only reported endpoints — re-test everywhere.

**Site-wide CSRF via frameable re-submit.** Blank CSRF token → error page reflecting *your new values* + re-submit form + no `X-Frame-Options` → iframe the error, user clicks = change committed. Every feature shared the flaw.

**Sandbox-CC business logic.** Phone-verification for page ownership; new "upgrade" feature needed only payment details; **sandbox credit cards always return true** and weren't blacklisted → claim any page. Dev tried blacklist patch → bypassed with different test numbers. *New features built on old assumptions.*

**Wayback → similar endpoint → ATO.** Old `robots.txt`/`SingleSignIn?userid=` leaked emails; discovered `DoubleSignIn` via Wayback — same param pattern logged him *into* the account. *Endpoint names are a fingerprint; reuse proven params on similar endpoints.*

**SSRF redirect → AWS keys.** API console filtered `localhost/127.0.0.1` input only; `mysite.com/redirect.php` → 302 → internal read + AWS keys. *Filters check input, not resolution.*

**WebSocket origin trust.** No origin verification on WS server → cross-site WS connection read user data. *WS needs the same origin/session checks as HTTP.*

**`@company.com` signup → admin.** Email `zseano@company.com` whitelisted verification → admin on any page. Also see the support-desk ticket trick (securinti) for helpdesk-based email ownership.

**`canEdit:"false"` — flip it.** Admin-only user-edit protected by client-side `canEdit:false`; tried editing admin's details → worked. *"false" is the new "true" — test, don't trust.*

## Resources Named in the Book

Payloads/research: PayloadsAllTheThings, zseano.com payloads, d3adend ghettoBypass, masatokinugawa filterbypass wiki, Awesome-WAF · Recon: certspotter API, yougetsignal, publicwww, apkscan.nviso.be, tarnish (Chrome ext analyzer) · Dorks: exposingtheinvisible dorking guide, `inurl: vulnerability/responsible disclosure` · Community: medium bugbountywriteup, @disclosedh1, zseano/following list · Learning: PayloadsAllTheThings for payload *reasoning* (why it exists, what it bypasses).

## Anti-patterns

- **One-and-done programs** — value compounds monthly; rotate instead of churning.
- **Assuming patch = fixed** — patches reveal dev thinking and are often bypassable or endpoint-local.
- **Automation of judgment** — automate collection; never automate the "should this work" question.

## Key Takeaways

1. Patch-bypass is a first-class technique: read the fix, find the gap, apply site-wide.
2. One-time requests (first app launch), one-time flows (connect social) are unobserved bug habitats.
3. Client-side flags (`canEdit`, `isPremium`, feature toggles) are server-side tests waiting to happen.
4. Similar endpoint names share params/bugs — Wayback + naming patterns = exploitation shortcuts.
5. "1,000+ vulnerabilities... simply used their site as intended" — methodology > tricks.

## Connects To

- **ch05/ch06**: the passes that generate the leads these cases exploit
- **ch01**: the mindset sustaining the loop (patience, notes, dev-thinking)
