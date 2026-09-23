# Step One — Getting a Feel for Things (First-Look Playbook)

## Core Idea
Before scanners: (0) read prior disclosed bugs for leads, then (1) manually walk the app's key features asking a fixed battery of questions. Assumption: *it should be secure* — your job is to challenge that on every input. This pass builds the mental map + notes + first findings.

## Frameworks Introduced

- **Lead-first recon**: before touching the app, search `domain.com vulnerability` on Google, HackerOne hacktivity, OpenBugBounty — disclosed bugs hand you a starting point, and old fixes can sometimes be re-bypassed.
- **Question-driven exploration**: each feature gets a standard question set — the questions ARE the methodology. Note answers + weird behaviors; wordlist building starts day one.

## The Feature Question Lists

### Registration Process
- What's required (name/bio/photo) and **where does it get reflected** after signup? (may only appear post-completion — retest later)
- Photo upload: `.txt/.xml/.svg` extension probes; check where it lands.
- Social/OAuth signup: which providers, what profile data is trusted (imported album named `<script>alert(0)</script>` → stored XSS), token-leakable flows?
- Allowed chars: `<> " '`, unicode, `%00`, `myemail%00@email.com` — same on mobile signup? (mobile = different codebase, often less filtering — "LOTS of stored XSS" this way)
- `@target.com` email — blacklisted? why? bypassable? (staff privileges)
- Revisiting register page when authenticated → redirect parameter?
- Params in source/JS; per-language/device variants; `login.js`-style per-feature JS files (more URLs inside)
- Google dorks: `site:example.com inurl:register inurl:&` (+`signup`, `join`)

### Login & Reset Password
- Redirect params on login: `returnUrl, goto, return_url, returnUri, cancelUrl, back, returnTo` — always try even if unseen.
- `myemail%00@email.com` login → treated as `myemail@email.com`? If yes → `%00` signup → ATO path; same for username claims.
- Social logins (geo-variant — WeChat for CN users); login-only social flow that links post-auth = separate test surface.
- Mobile login flow differences (UX-driven = simpler = weaker checks).
- Reset: params → inject `id` (IDOR/HPP); **Host header trust** — `Host: evil.com` → reset link emailed pointing at evil.com → token leak.
- Rate-limit on login/register/reset: often informative/out-of-scope — check program policy before burning time.

### Updating Account Information
- CSRF presence + validation behavior: blank token, same-length token, error leaks (framework info).
- Email/password change without re-confirmation → chain with XSS → ATO.
- `<> " '` unicode `%09 %07 %0d%0a` in every field — where reflected?
- Profile URL field → `javascript:alert(0)` filtering.
- Mobile vs desktop update flows (API + IDOR + different filtering).
- Media upload filtering; storage domain (CDN in CSP?).
- Entity-escaped sinks: `runjs('userinput&lt;&quot;')` → `');alert('x');` context escape.

### Developer Tools (API consoles, webhooks, OAuth apps, GraphQL explorers)
- Where hosted — AWS? (then target = AWS keys); can you see responses (easier impact)?
- Webhook test features = SSRF candidates.
- OAuth app flows: does "Cancel/Deny" still issue a token with permissions? `returnTo`/`cancelUrl` param control; whitelisted redirect domains (`theirdomain.com`, `amazonaws.com`, `localhost`).
- Token lifecycle: disconnect → invalidated? Docs/wiki reveal token usage (turns leaks into P1).
- Separate dev-site account vs shared session — token exchange via redirect = open-redirect chaining spot.
- Different codebase than main app → fresh upload/filter tests.

### Main Feature (the business's crown jewel)
- Map top-down: the feature the company is built on (Dropbox→uploads, AOL→mail) — should be most secure; verify.
- Mobile-only features; per-country TLD variants (different codebases/payment options).
- Shared data sources across features — same API request or different endpoints per page?
- Paid vs free: can free account access paid features? (test both accounts)
- Oldest features (failed launches → orphaned code) + upcoming features (Twitter/newsletters → `.js` references before release; `false`→`true` flag flips).
- Privacy settings — is "private" actually private?
- Role levels (admin/mod/user/guest): cross-level API calls — "eyes should light up."

### Payment Features
- Free→paid feature access without paying (some programs reject "loss of revenue only" — but it unlocks test surface).
- Payment data in DOM → XSS chain for impact.
- Per-country payment options: US "Checking Account" accepted **sandbox** details → bypassed verification → page ownership. Test CC numbers (worldpay/paypal test docs).

## Anti-patterns

- **Scanner-first** — manual feel comes before automation; tools lack judgment about "should this work this way."
- **Single-pass testing** — first look is never complete; the whole methodology assumes re-visits.
- **Desktop-only testing** — mobile apps/APIs carry separate bugs.
- **Skipping one-time requests** — first-run mobile requests (device registration, GDPR consent) happen once; proxy must be watching.

## Key Takeaways

1. Fixed question battery per feature → consistent coverage + notes for later passes.
2. `false`→`true`, `canEdit:false`, sandbox-CC — "testing if features work as intended" is most of the job.
3. Auth surfaces (register/login/reset/update) are the densest bug territory — `%00`, host-header, redirect params, OAuth.
4. Developer consoles concentrate SSRF + token + permission bugs; check AWS hosting early.
5. Main-feature focus: crown-jewel code is oldest/most-patched but its *seams* (new↔old, mobile↔desktop, free↔paid) break.

## Connects To

- **ch03/ch04**: the vuln techniques these questions invoke
- **ch06**: Step Two expands beyond the main app
- **ch07**: findings that came from this exact question list
