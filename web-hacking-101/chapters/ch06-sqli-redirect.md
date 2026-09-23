# Ch 6 — SQL Injection + Open Redirect

## SQL Injection (the book's angle: framework vulns, not just `' OR 1=1`)

The chapter's case is **Drupal SQL Injection (Drupalgeddon, SA-CORE-2014-005)** — a framework-level flaw, not dev string-concat: Drupal's `expandArguments` built `IN (...)` placeholders from array keys; attacker-controlled array keys became part of the query → unauthenticated SQLi → admin hash extraction → RCE.

**Implications for hunting:**
- SQLi isn't only sloppy devs — the *framework/CMS* can inject the sink. Fingerprint the stack → check for framework CVEs (Drupalgeddon-class) before hand-fuzzing.
- Mass-scan pattern: a framework vuln = every site on that version is a target. This is the 1-day game (cf. playbook skill ch01).
- Manual detection stays classic: `'`, `"`, `)` in params → error/discrepancy → `order by` → `union select` (dialect details in `bug-bounty-playbook` ch06).

## Open Redirect

App takes a param as the redirect target without validating the destination (`redirect_to=`, `domain_name=`, `checkout_url=`, `next=`, `return_to=`). Impact alone = phishing-trust abuse; real money is in chains.

**Cases:**
- **Shopify theme install** ($500): `?domain_name=example.com` → 301 to `example.com/admin`. Trivial param swap.
- **Shopify login** ($500): `?checkout_url=.np` → post-login redirect to `mystore.myshopify.com.np` — *domain-suffix trick*: appends to the host so it still "looks like" shopify.com. Test suffix forms: `.evil.com`, `evil.com`, `//evil.com`, `\@evil.com`.
- **HackerOne/Zendesk interstitial** ($500): `zendesk_session` links were "trusted" — attacker-controlled Zendesk theme injected `<script>document.location.href=…` → trusted-domain link → attacker redirect, no warning. Reported to Zendesk first — rejected; escalated via the HackerOne path. *Takeaways: (1) third-party service features are your redirect surface; (2) "not a bug" isn't final — dig, chain, demonstrate.*
- **Facebook OAuth tokens** (cross-reference ch07): `redirect_uri` on a misconfigured pre-authorized app → access token appended to attacker URL.

## Detection habits

- Every param containing a URL/path/domain → swap in `https://google.com`, then attacker-domain variants (suffix, `@`, `//`, `javascript:`).
- Login/return flows especially — they're where "go back where you came from" params live.
- Trust-boundary jumps: `trusted.com` → partner service → arbitrary site = interstitial bypass.
