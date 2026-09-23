# Ch2: Discovery — Find the Road Less Traveled

Source: `02_Discovery`. On wide-scoped programs the flagship application is heavily assessed; your edge is finding the applications (or parts of applications) that are *less tested*.

## Surface-expansion checklist

- **Wildcard scope is your friend** — `^.acme.com` (all subdomains) means the inventory itself is the vulnerability mine.
- **Domain discovery via search engines** — Google et al.; automatable with recon frameworks (Haddix's era: Recon-ng + his `enumall` script; modern stack lives in `recon-pipeline`).
- **Full port scans on all discovered domains** — find obscure web servers and extraneous services (see below).
- **Acquisitions** — enumerate acquired companies and check the program's acquisition policy (example rule of thumb: some programs only cover acquisitions after a grace period). Lists: Wikipedia M&A pages per parent company.
- **Functionality changes & redesigns** — newly shipped features are the least-regression-tested surface.
- **Mobile sites & new app versions** — separate apps, separate bugs, fewer hunters.
- **Trademark/privacy-policy search on the parent company** — surfaces branded properties not obviously under the flagship domain.

## The recon stack (tooling table)

| Task | TBHM-era tools | Notes |
|---|---|---|
| Domain/subdomain enum | Recon-ng (+ `enumall` automation script from jhaddix/domain) | Chainable recon framework; automate the whole sweep |
| Search-engine discovery | Google dorks — `site:paypal.com -www.paypal.com -www.sandbox` | Negations peel away known hosts to reveal the long tail |
| M&A enumeration | Wikipedia "list of mergers and acquisitions by X" | Cross-check each entity against program scope/acquisition rules |
| Port/service discovery | `nmap -sS -A -PN -p- --script=http-title <target>` | SYN scan, service+OS fingerprint, skip host discovery, **all 65535 ports**, grab HTTP titles |
| OSINT aggregation | Intrigue (see ch03) | Framework wrapping DNS brute-force, spidering, scans |

## Port scanning pays off on web scope

Haddix's refrain: "port scanning is not just for netpen." A full-port sweep across newly found targets routinely yields:

- separate webapps living on non-80/443 ports
- extraneous services that shouldn't be public
- canonical wins: an unauthenticated Jenkins script console on one big target; RDP exposed on another, vulnerable to a known RCE bulletin

Throttle scans to the program's rate rules; on cloud-hosted targets confirm scanning is permitted.
