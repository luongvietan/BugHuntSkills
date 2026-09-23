# API9:2019 — Improper Assets Management

## Core Idea
Old versions, beta/staging hosts, and undocumented endpoints run the same data with weaker defenses. Version-rotate (`v2`→`v1`) and hunt alternate hosts — yesterday's API misses today's fixes (rate limits, authz, WAF rules).

**Scores**: Exploitability 3 · Prevalence 3 · Detectability 2 · Technical Impact 2

## Is the API Vulnerable? (Inventory Questions)

For every host, the org should answer — if it can't, it's probably exposed:

- Environment (production/staging/test/dev)? Required network access (public/internal/partners)?
- Which API version? What data (PII?) and data flow?
- Documentation current? Retirement plan per version?
- Host + integrated-service inventories complete?
- **Old versions running unpatched?**

## Example Attack Scenarios

**Scenario 1 — version rollback**: redesigned service leaves `api.someservice.com/v1` running, unprotected, same DB. Attacker finds `v2` in the app, swaps `v2`→`v1` → old API exposes PII of **100M+ users**.

**Scenario 2 — protection gap on beta host**: `www.socialnetwork.com` rate-limits reset-password brute force via a *separate gateway component*. Researcher finds `www.mbasic.beta.socialnetwork.com` running the same API **without the gateway** → brute-forces 6-digit reset tokens → reset any user's password. *External protection layers don't follow the API to shadow hosts.*

## How To Prevent

- Inventory all API hosts: environment, network exposure, version.
- Inventory integrated services: role, data exchanged, sensitivity.
- Document auth, errors, redirects, rate limiting, CORS, endpoints+params+requests+responses — auto-generate in CI/CD (OpenAPI).
- Extend external protections (API firewall) to **all** exposed versions, not just production.
- No production data on non-production deployments — or give them production-grade protection.
- On security improvements in new versions: risk-analyze backport vs. forced retirement.

## Anti-patterns

- **Version rotation untested**: `v1`, `v2`, `v3`, `beta`, `staging`, `test`, `dev`, `mbasic`, `legacy` — try them all on the same host path and as subdomains.
- **Trusting gateway controls**: the API behind them may be wide open on other hosts.
- **Old docs ignored**: outdated OpenAPI/Swagger docs still map the attack surface.

## Key Takeaways

1. Subdomain recon for `api.*`, `beta`, `staging`, `dev`, `test`, `mbasic`, `legacy` hosts.
2. Path rotation: `/v1/` ↔ `/v2/` ↔ `/v3/` ↔ unversioned — old versions keep old bugs.
3. Every control added "later" (rate limit, authz, WAF) is a diff between versions — test the gap.
4. Exposed debug/admin endpoints and `.git`/swagger files on shadow hosts.
5. CWE-1059 (Incomplete Documentation).

## Connects To

- **API4 (ch02 rate-limit scenario, ch04)**: protection layers missing on shadow versions
- **API7 (ch07)**: misconfig per host multiplies with unmanaged inventory
- **API5 (ch05)**: undocumented endpoints = undocumented function-level flaws
