# Ch 3 — GitHub Dorking + Subdomain Takeover

> **Credentials & takeover rules:** dorking is passive, but a found credential is reported not exercised (masked prefix; validation only under explicit policy authorization — `hacking-the-cloud` ch03). Subdomain takeover PoC = claim the dangling resource and serve *your own marker page* — never real content, never cookies.

Two of the easiest high/critical findings in the book — low skill, high payout.

## GitHub sensitive-info mining

Developers hard-code test accounts, API keys, and secrets, then push to public repos. External hardening is irrelevant if creds are sitting in a public repo.

**Workflow:**
1. Build/borrow a sensitive-word list (filenames, extensions, variable names — e.g. `@obheda12`'s dork list).
2. Search: `Domain.com "<keyword>"` → e.g. `hackerone.com "password"`.
3. Manually sift results — expect ~hours of junk before gold. **The tedium is the moat**: most hunters quit; those who sift get insta-high/critical findings.

Good dorks: `password`, `api_key`, `secret`, `token`, `.env`, `config`, `credentials`, `BEGIN RSA PRIVATE KEY`, internal hostnames.

## Subdomain takeover

**Condition**: subdomain CNAMEs to a third-party service host that no longer exists/is unclaimed → register that resource → you control the subdomain's content. One DNS change can flip safe→vulnerable overnight.

**Workflow:**
1. Find dangling CNAME — error fingerprints (e.g. `There isn't a Github Pages Site here` + 404).
2. Confirm provider is claimable: `github.com/EdOverflow/can-i-take-over-xyz` — per-provider walkthroughs.
3. `dig` the subdomain → get the exact target host (e.g. `user.github.io`).
4. Follow the provider's claim process. For GitHub Pages: create repo matching the CNAME name → add `index.html` → enable Pages → set custom domain to the target subdomain.
5. Visit the subdomain → your page → proven.

Each provider (Heroku, S3, Azure, Shopify…) has its own claim procedure — the can-i-take-over-xyz wiki documents them all.

## Severity note

Full subdomain control → phishing on a trusted domain, cookie theft (if cookies scoped to `*.domain.com`), OAuth/SAML abuse. Easy + high-impact.
