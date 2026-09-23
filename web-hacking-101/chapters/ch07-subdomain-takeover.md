# Ch 7 — Subdomain Takeover + Stale-Asset Abuse

## The mechanic

DNS entry points a subdomain at a third-party service resource that nobody claims anymore:

```
sub.example.com → CNAME → unicorn457.heroku.com (unclaimed)
→ attacker registers unicorn457.heroku.com → serves content AS sub.example.com
```

Phishing on a trusted domain, cookie scope abuse (`*.example.com`), OAuth/CORS trust inheritance.

## Finding candidates

- **KnockPy** + SecLists subdomain list for enumeration.
- **Google dorks**: `site:*.target.com`.
- **Recon-ng / enumall** for broader enum.
- For each resolved subdomain → does the CNAME target exist? Service fingerprints: `NoSuchBucket` (S3), "There isn't a GitHub Pages site here", Heroku "No such app", Zendesk "this help center no longer exists", Shopify "Sorry, this shop is unavailable"… (see can-i-take-over-xyz in the playbook skill).
- Prioritize services that let anyone claim a named resource: S3, GitHub Pages, Heroku, Zendesk, Shopify, Azure, Tumblr, Fastly.

## Cases

- **Ubiquiti** ($500): `assets.goubiquiti.com` → S3 CNAME, bucket never created → claim `uwn-images.s3-website-us-west-1.amazonaws.com` → host fake Ubiquiti.
- **Scan.me → Zendesk** ($1,000): `support.scan.me` CNAME → `scan.zendesk.com` unclaimed → registered it, done. "PAY ATTENTION" — this took minutes.
- **Facebook access tokens** (undisclosed, the crown jewel): not classic subdomain takeover but the *same stale-asset idea applied to OAuth apps*.

## The Facebook OAuth lesson (stale assets beyond DNS)

Flow: user → app → FB grants code → app exchanges for token → token back via `redirect_uri`.

Philippe Harewood found Facebook-owned apps **pre-authorized for every user** (e.g. "Content Tab of a Page on www", list at `/search/me/apps-used`) whose `redirect_uri` pointed to a **claimable host**. Exploit URL:

```
facebook.com/v2.5/dialog/oauth?response_type=token&display=popup&client_id=APP_ID&redirect_uri=REDIRECT_URI
```

No consent popup (already authorized) → victim clicks → `REDIRECT_URI/#access_token=…` → log tokens → full account + Instagram (FB GraphQL exchanged FB token for sibling-app tokens).

## Generalized pattern

Stale/abandoned-but-trusted resources:
- DNS records → dead services (classic takeover).
- Pre-authorized OAuth apps → claimable redirect_uri (token theft).
- Old subdomains pointing at deleted SaaS tenants (Zendesk, Shopify).
- Decommissioned-but-referenced assets: JS files on dead CDNs, docs sites, status pages.

**Rule: when a company changes infrastructure, what did they leave pointing at something they no longer own?**
