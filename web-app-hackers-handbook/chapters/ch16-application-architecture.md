# Ch17: Attacking Application Architecture

Source: Chapter 17. The application isn't one box — tiers trust each other, and hosts share platforms. Architectural attacks exploit the **trust between components**, not the code inside one.

## Tiered architectures

- **Trust between tiers**: web tier → app tier → data/services tier. Each tier often assumes the *previous* tier validated input and authenticated the caller — internal services exposed without the perimeter's checks. Finding a way to reach a back tier directly (SSRF-style server-side requests, misrouted vhosts, direct service ports, web-service endpoints) lets you skip every front-line control.
- **Defense-in-depth failures**: a DB query is filtered at the web tier but not at an internal service tier; an internal API has "no auth needed — only the web tier can reach us." Probe for reachable internal endpoints: different ports, IP addresses, service names, web-service/SOAP interfaces on the same host.
- **Segregation gaps**: components separated logically but not physically/credential-wise — one compromised tier reaches everything (flat network, shared service accounts, shared DB creds).

## Shared hosting & ASPs

- **Virtual hosting**: many sites, one IP. Default/vhost-selection bugs: request with wrong/missing `Host`, IP literal, or alternate names → reach the *default* site or other tenants' admin panels. Path overlaps between vhosts can leak files.
- **Shared application services (ASP model)**: one codebase serves many customers, separated by a "customer" discriminator (param, cookie, subdomain, header). Test: **cross-tenant** access by switching that discriminator while holding your own session — the classic ASP flaw (your session + `customer=B` = B's data). Registering a new account/tenant gives a clean test identity.
- **Access mechanisms on shared infra**: FTP/SSH/control panels on shared hosts — default creds, weak isolation, writable shared dirs.
- **Attacks between co-hosted apps**: one tenant's vuln reaches others via shared filesystem (upload → neighbor's webroot), shared DB (cross-schema access), shared memory/session stores.
- **Cloud variants**: same discriminator-abuse logic applies to tenant IDs, org IDs, project IDs in modern multi-tenant apps; metadata endpoints and management APIs are the shared-control-plane equivalent.

## Attack method

1. Diagram the architecture: tiers, trust links, shared components (DB, filesystem, session store, service accounts).
2. Probe each trust boundary: can the back tier be reached without the front? Does it re-authenticate/validate?
3. Shared-environment tests: alternate `Host` values, IP literals, tenant-discriminator swapping, cross-vhost paths, default-site content.
4. Look for lateral reach: writable shared dirs, shared creds in config files, cross-tenant session stores.

## Checklist

- [ ] Each tier's reachability tested directly (ports, services, internal endpoints).
- [ ] Trust-assumption violations: internal requests missing auth/validation.
- [ ] Vhost/default-site/host-header manipulation tried.
- [ ] Tenant discriminator identified and swapped cross-tenant (your two test tenants).
- [ ] Shared components checked for cross-tenant paths (FS, DB, sessions).
