# 03-vuln-class-index.md — vuln class -> chapter routing

Lookup table for phase 4. Columns: **primary** (open this first), **alternates**
(second opinion / deeper mechanism / another author's loop), **payloads**
(which payload for this context — reference lookups, not methodology).

Paths are relative to the skills root: `skill-name/chapters/file.md`. Every
path below was verified against the on-disk `chapters/` listing — if one ever
misses, run `ls <skill>/chapters/` and pick the closest real file; file names
are generated-skill order and do not always equal source-book chapter numbers.

Note on `web-app-hackers-handbook`: `chNN` file numbers are generated order —
each chapter file's title cites the WAHH source chapter, which runs one higher
for files `ch02`+ (`ch10-application-logic.md` covers WAHH Chapter 11).

| Vuln class | Primary skill + chapter | Alternates | Payload chapter |
|---|---|---|---|
| XSS | `web-security-academy/chapters/ch10-xss.md` | `bug-bounty-bootcamp/chapters/ch04-xss.md`; `web-app-hackers-handbook/chapters/ch11-xss.md`; `bug-bounty-playbook/chapters/ch07-xss.md`; `tbhm-methodology/chapters/ch05-tactical-fuzzing-xss-sqli.md`; `zseano-methodology/chapters/ch03-common-issues-xss-csrf.md` | `payloads-all-the-things/chapters/ch01-xss.md`; `xss-cheat-sheet/chapters/ch03-filter-bypass.md` |
| SQLi | `web-security-academy/chapters/ch11-sqli-nosql.md` | `bug-bounty-bootcamp/chapters/ch08-sqli.md`; `web-app-hackers-handbook/chapters/ch08-attacking-data-stores.md`; `bug-bounty-playbook/chapters/ch06-sql-injection.md`; `web-hacking-101/chapters/ch06-sqli-redirect.md` | `payloads-all-the-things/chapters/ch02-sqli.md` |
| NoSQLi (NoSQL/LDAP/XPath) | `web-security-academy/chapters/ch11-sqli-nosql.md` | `web-app-hackers-handbook/chapters/ch08-attacking-data-stores.md` | `payloads-all-the-things/chapters/ch03-nosql-ldap-xpath.md` |
| SSTI | `bug-bounty-bootcamp/chapters/ch13-ssti.md` | `bug-bounty-playbook/chapters/ch12-ssti.md`; `web-hacking-101/chapters/ch09-rce-template-ssrf.md` | `payloads-all-the-things/chapters/ch04-ssti.md` |
| SSRF | `bug-bounty-bootcamp/chapters/ch10-ssrf.md` | `web-security-academy/chapters/ch12-ssrf-xxe.md`; `hacking-the-cloud/chapters/ch02-metadata-services-ssrf.md` (cloud pivot); `zseano-methodology/chapters/ch04-common-issues-redirect-ssrf-upload-idor.md`; `web-hacking-101/chapters/ch09-rce-template-ssrf.md` | `payloads-all-the-things/chapters/ch05-ssrf.md` |
| XXE | `bug-bounty-bootcamp/chapters/ch12-xxe.md` | `web-security-academy/chapters/ch12-ssrf-xxe.md`; `web-hacking-101/chapters/ch08-xxe.md`; `bug-bounty-playbook/chapters/ch14-xxe-csp-rpo.md` | `payloads-all-the-things/chapters/ch06-xxe.md` |
| LFI / path traversal | `web-app-hackers-handbook/chapters/ch09-backend-components.md` | `tbhm-methodology/chapters/ch06-uploads-lfi-redirects.md`; `bug-bounty-playbook/chapters/ch08-upload-traversal-redirect-idor.md` | `payloads-all-the-things/chapters/ch07-file-inclusion.md` |
| Command injection / RCE | `web-app-hackers-handbook/chapters/ch09-backend-components.md` | `bug-bounty-bootcamp/chapters/ch15-rce.md`; `web-hacking-101/chapters/ch02-injection-primitives.md`; `web-security-academy/chapters/ch13-injection-files.md` | `payloads-all-the-things/chapters/ch08-command-injection.md` |
| File upload | `bug-bounty-playbook/chapters/ch08-upload-traversal-redirect-idor.md` | `tbhm-methodology/chapters/ch06-uploads-lfi-redirects.md`; `web-security-academy/chapters/ch13-injection-files.md`; `zseano-methodology/chapters/ch04-common-issues-redirect-ssrf-upload-idor.md` | `payloads-all-the-things/chapters/ch09-upload.md` |
| Deserialization | `bug-bounty-bootcamp/chapters/ch11-deserialization.md` | `web-app-hackers-handbook/chapters/ch09-backend-components.md` | `payloads-all-the-things/chapters/ch10-deserialization.md` |
| IDOR / BOLA | `owasp-api-security-top-10/chapters/ch01-broken-object-level-authorization.md` | `bug-bounty-bootcamp/chapters/ch07-idor.md`; `hacking-apis/chapters/ch07-authorization-bola-bfla.md`; `zseano-methodology/chapters/ch04-common-issues-redirect-ssrf-upload-idor.md`; `owasp-wstg/chapters/ch05-authorization.md` | `payloads-all-the-things/chapters/ch15-logic-misc.md` |
| BFLA (function-level authZ) | `owasp-api-security-top-10/chapters/ch05-broken-function-level-authorization.md` | `hacking-apis/chapters/ch07-authorization-bola-bfla.md`; `owasp-wstg/chapters/ch05-authorization.md` | `payloads-all-the-things/chapters/ch15-logic-misc.md` |
| Mass assignment | `owasp-api-security-top-10/chapters/ch06-mass-assignment.md` | `hacking-apis/chapters/ch08-mass-assignment.md` | `payloads-all-the-things/chapters/ch15-logic-misc.md` |
| Authentication / session | `web-app-hackers-handbook/chapters/ch05-authentication.md` + `ch06-session-management.md` | `owasp-wstg/chapters/ch04-authentication.md` + `ch06-session-management.md`; `tbhm-methodology/chapters/ch04-auth-session.md`; `owasp-api-security-top-10/chapters/ch02-broken-user-authentication.md`; `hacking-apis/chapters/ch05-attacking-authentication.md`; `web-security-academy/chapters/ch14-access-authn-info.md` | `payloads-all-the-things/chapters/ch13-auth-session.md` |
| JWT / OAuth / SAML / SSO | `web-security-academy/chapters/ch07-jwt.md` + `ch08-oauth.md` | `bug-bounty-bootcamp/chapters/ch17-sso.md`; `hacking-apis/chapters/ch05-attacking-authentication.md` | `payloads-all-the-things/chapters/ch13-auth-session.md` |
| CSRF | `web-security-academy/chapters/ch09-cors-csrf-clickjacking.md` | `bug-bounty-bootcamp/chapters/ch06-csrf.md`; `web-hacking-101/chapters/ch03-csrf.md`; `tbhm-methodology/chapters/ch07-csrf-priv-logic-transport.md`; `zseano-methodology/chapters/ch03-common-issues-xss-csrf.md` | `payloads-all-the-things/chapters/ch12-open-redirect-csrf.md` |
| CORS / SOP / postMessage | `web-security-academy/chapters/ch09-cors-csrf-clickjacking.md` | `bug-bounty-bootcamp/chapters/ch16-sop.md` | `payloads-all-the-things/chapters/ch12-open-redirect-csrf.md` |
| Clickjacking | `web-security-academy/chapters/ch09-cors-csrf-clickjacking.md` | `bug-bounty-bootcamp/chapters/ch05-open-redirects-clickjacking.md` | `payloads-all-the-things/chapters/ch12-open-redirect-csrf.md` |
| Open redirect | `bug-bounty-bootcamp/chapters/ch05-open-redirects-clickjacking.md` | `zseano-methodology/chapters/ch04-common-issues-redirect-ssrf-upload-idor.md`; `tbhm-methodology/chapters/ch06-uploads-lfi-redirects.md`; `web-hacking-101/chapters/ch06-sqli-redirect.md` | `payloads-all-the-things/chapters/ch12-open-redirect-csrf.md` |
| Request smuggling / HTTP desync | `web-security-academy/chapters/ch01-request-smuggling.md` + `ch02-http2-desync.md` | `web-app-hackers-handbook/chapters/ch02-web-technologies.md` (protocol grounding) | `payloads-all-the-things/chapters/ch14-http-infra.md` |
| Cache poisoning / host attacks | `web-security-academy/chapters/ch03-cache-host.md` | `bug-bounty-playbook/chapters/ch11-cache-attacks.md` | `payloads-all-the-things/chapters/ch14-http-infra.md` |
| Prototype pollution | `web-security-academy/chapters/ch04-prototype-pollution.md` | `bug-bounty-playbook/chapters/ch13-osrf-prototype-csti.md` | `payloads-all-the-things/chapters/ch15-logic-misc.md` |
| DOM clobbering / DOM attacks | `web-security-academy/chapters/ch05-dom-attacks.md` | `xss-cheat-sheet/chapters/ch02-advanced.md` | `payloads-all-the-things/chapters/ch01-xss.md` |
| WebSockets / CSWSH | `web-security-academy/chapters/ch06-websockets.md` | `owasp-wstg/chapters/ch11-client-side.md` | `payloads-all-the-things/chapters/ch14-http-infra.md` |
| GraphQL | `hacking-apis/chapters/ch11-graphql.md` | `web-security-academy/chapters/ch15-logic-race-api-llm.md` | `payloads-all-the-things/chapters/ch11-graphql.md` |
| Race conditions | `bug-bounty-bootcamp/chapters/ch09-race-conditions.md` | `web-security-academy/chapters/ch15-logic-race-api-llm.md` | `payloads-all-the-things/chapters/ch15-logic-misc.md` |
| Business logic | `bug-bounty-bootcamp/chapters/ch14-logic-access-control.md` | `owasp-wstg/chapters/ch10-business-logic.md`; `web-app-hackers-handbook/chapters/ch10-application-logic.md`; `web-hacking-101/chapters/ch04-application-logic.md` | `payloads-all-the-things/chapters/ch15-logic-misc.md` |
| Access control | `web-app-hackers-handbook/chapters/ch07-access-controls.md` | `bug-bounty-bootcamp/chapters/ch14-logic-access-control.md`; `owasp-wstg/chapters/ch05-authorization.md`; `web-security-academy/chapters/ch14-access-authn-info.md` | `payloads-all-the-things/chapters/ch15-logic-misc.md` |
| Info disclosure / errors | `bug-bounty-bootcamp/chapters/ch18-info-disclosure.md` | `web-app-hackers-handbook/chapters/ch14-information-disclosure.md`; `web-security-academy/chapters/ch14-access-authn-info.md`; `owasp-wstg/chapters/ch08-error-handling.md`; `owasp-api-security-top-10/chapters/ch03-excessive-data-exposure.md` | — |
| Subdomain takeover | `bug-bounty-playbook/chapters/ch03-github-subdomain-takeover.md` | `web-hacking-101/chapters/ch07-subdomain-takeover.md` | — |
| Exposed secrets / public repos | `hacking-the-cloud/chapters/ch03-found-iam-credentials.md` | `bug-bounty-playbook/chapters/ch04-exposed-databases.md`; `recon-pipeline/chapters/01-stages.md` (stage 6 code-leak hunting) | — |
| Cloud — metadata / IAM / storage | `hacking-the-cloud/chapters/ch02-metadata-services-ssrf.md` + `ch03-found-iam-credentials.md` + `ch04-storage-buckets-snapshots.md` | `hacking-the-cloud/chapters/ch05-privesc-misconfigured-policies.md` + `ch08-multicloud-general.md`; `bug-bounty-bootcamp/chapters/ch10-ssrf.md` (the usual way in) | `payloads-all-the-things/chapters/ch05-ssrf.md` |
| Mobile (Android/iOS) | `owasp-mas/chapters/ch01-methodology-setup.md` then `ch02-storage.md`-`ch08-resilience.md` | `bug-bounty-bootcamp/chapters/ch20-android.md`; `tbhm-methodology/chapters/ch08-mobile-aux-checklist.md` | — |
| LLM / GenAI app surface | `web-security-academy/chapters/ch15-logic-race-api-llm.md` (OWASP GenAI LLM + Agentic Top 10 2026 mapping; selected AI Testing Guide v1 methods) | `owasp-api-security-top-10/chapters/ch08-injection.md` + `ch09-improper-assets-management.md`; `hacking-apis/chapters/ch12-breaches-bounties-checklist.md` | `payloads-all-the-things/chapters/ch15-logic-misc.md` |
| API misc / inventory | `owasp-api-security-top-10/chapters/ch09-improper-assets-management.md` | `hacking-apis/chapters/ch03-discovering-apis.md`; `owasp-wstg/chapters/ch12-api-testing.md` | — |

## How to use the index

1. Find the row, open the **primary** chapter, run its hunting loop verbatim
   before improvising.
2. If the primary's bypasses bounce off an observed defense, open an
   **alternate** — different authors document different bypass families.
3. When the technique is settled and only the string is missing, open the
   **payload** column — `payloads-all-the-things` answers "which payload for
   this context"; `xss-cheat-sheet` is the deeper XSS-only reference
   (`chapters/ch01-basics.md` through `ch05-miscellaneous.md`).
4. API targets: always cross-check the `owasp-api-security-top-10` row entry —
   the same bug class has an API-specific shape (BOLA vs IDOR is the canonical
   example). That skill now carries the **2023 edition** naming (API1 BOLA,
   API3 BOPLA, API6 sensitive business flows, API7 SSRF, API10 unsafe API
   consumption); rows above cite chapters by filename, not edition year.
5. Found something the index misses? It is probably mapping back to one of:
   `owasp-wstg/chapters/ch02-configuration-deployment.md` (misconfig),
   `web-app-hackers-handbook/chapters/ch16-application-architecture.md` /
   `ch17-application-server.md` (platform bugs), or
   `bug-bounty-playbook/chapters/ch01-known-vulnerabilities.md` /
   `ch02-cms.md` (known-CVE and CMS checks).

## Coverage note — all 16 companion skills

Every companion skill appears above or in `01-phase-map.md`: the 14 book
skills plus the two ops skills (`recon-pipeline`, `report-writing`). Classes
with no dedicated chapter anywhere route to the nearest row's alternates —
when in doubt, the `owasp-wstg` and `web-app-hackers-handbook` columns cover
the long tail.
