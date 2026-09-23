# API7:2019 — Security Misconfiguration

## Core Idea
The broadest surface: unhardened stack, missing patches, extra HTTP verbs, absent TLS/security headers, permissive CORS, verbose errors, open cloud storage. Automated tooling exists, making it the most-detectable risk.

**Scores**: Exploitability 3 · Prevalence 3 · Detectability 3 · Technical Impact 2

## Is the API Vulnerable? (Checklist)

- Missing **security hardening** anywhere in the stack; bad cloud permissions (S3 buckets)
- **Missing security patches** / out-of-date systems
- **Unnecessary features enabled** (e.g., extra HTTP verbs)
- **TLS missing** — on any traffic, including "harmless" static assets
- **Security headers absent** (CSP, HSTS, X-Frame-Options, X-Content-Type-Options…)
- **CORS missing or misconfigured** (overly permissive origins)
- **Error messages leak stack traces** or sensitive detail

## Example Attack Scenarios

**Scenario 1 — leaked ops file**: `.bash_history` exposed at web root contains `curl -H 'authorization: Basic Zm9vOmJhcg=='` → valid creds + discovery of undocumented DevOps endpoints.

**Scenario 2 — default DB config**: search-engine recon finds an internet-facing database admin system on default port, auth disabled by default → millions of PII/auth records.

**Scenario 3 — partial TLS**: API on HTTPS but profile images over HTTP → response-size patterns let attacker track user content preferences (metadata leak even without content access).

## How To Prevent

- Repeatable **hardening process** across the stack; review orchestration files, API components, cloud permissions (S3).
- TLS for **all** interactions including static assets.
- Automated continuous config assessment.
- Response schemas for error payloads — no stack traces.
- Allow only required HTTP verbs; disable the rest (e.g., HEAD).
- Proper CORS policy for browser-accessed APIs.

## Anti-patterns

- **TLS only on "sensitive" paths**: HTTP assets still leak metadata/cookies.
- **CORS `*` or reflecting any Origin**: probe `Origin: https://evil.com` + credentials flag.
- **Verbose errors**: trigger 4xx/5xx deliberately and mine stack traces, versions, paths.

## Key Takeaways

1. Recon finds config: web-root dotfiles (`.bash_history`, `.git`, `.env`), default ports/services, old admin panels.
2. Method inventory: `OPTIONS` + try PUT/DELETE/PATCH/TRACE/HEAD on every endpoint.
3. Header audit: security headers missing → report with concrete abuse path.
4. CORS: test origin reflection, `null` origin, subdomain wildcards — credentials support raises impact.
5. Error responses are intel: stack traces reveal framework, paths, query internals.

## Connects To

- **API9 (ch09)**: unpatched old versions and forgotten hosts are misconfiguration at fleet scale
- **API5 (ch05)**: disabled-verb check pairs with method-swap BFLA probing
- OWASP Secure Headers Project, Testing Guide config chapters; CWE-2/16/388
