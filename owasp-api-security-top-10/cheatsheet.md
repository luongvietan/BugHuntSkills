# Cheatsheet — OWASP API Security Top 10 2023: bounded tester decision table

Use the 2023 taxonomy and IDs below. The parent `SKILL.md` and crosswalk
retain the 2019 source-book chapter paths; that historical structure does not
change the current risk names.

## Entry gate — apply before every check

1. Verify the exact host and path against the current program scope contract.
   A discovered host, subdomain, or path is a lead until scope is confirmed.
2. Use an endpoint observed in the authorized application flow or supplied
   program documentation. Do not guess routes or sweep endpoint lists.
3. Verify that the exact technique and HTTP method are allowed. Policy
   silence or ambiguity means `blocked-by-policy`; ask the program before
   continuing.
4. Use only researcher-controlled accounts and records. Never access
   real-user data or affect shared, financial, or third-party resources.
5. Choose the smallest permitted check and stop at unexpected data, impact,
   errors, or service degradation.

## Current model — API Security Top 10:2023

| Current risk | Start only from this verified surface | Bounded first check |
|---|---|---|
| **API1:2023 Broken Object Level Authorization (BOLA)** | An observed object request on a listed asset; its method is allowed. | Compare only objects created by two accounts you control, and only if cross-account checks are permitted. Never enumerate or use identifiers belonging to other people. |
| **API2:2023 Broken Authentication** | An observed login, session, reset, or token flow covered by the policy. | Review the flow using your own accounts. Run a low-volume auth check only when the relevant technique and rate are allowed; no spraying or lockout testing by assumption. |
| **API3:2023 Broken Object Property Level Authorization (BOPLA)** | An observed response or mutation for a researcher-owned record. | Compare fields available to your own roles. Test a harmless property write only when that exact mutation method is allowed; do not try privilege, balance, or verification changes without explicit permission. |
| **API4:2023 Unrestricted Resource Consumption** | An observed operation with a documented or directly observed resource limit. | Prefer policy/docs review. Make no burst, large-payload, exhaustion, or degradation test. A minimal request is eligible only if the exact check and rate are permitted. |
| **API5:2023 Broken Function Level Authorization (BFLA)** | An observed function and roles you control on a listed asset. | Compare access using your own authorized roles and only an allowed method. Do not guess admin paths or try unlisted methods. |
| **API6:2023 Unrestricted Access to Sensitive Business Flows** | An observed business flow and a policy-permitted way to test it. | Use only controlled accounts and non-impacting records. Do not repeat purchases, bookings, votes, redemptions, or other side effects unless the program explicitly permits that exact check. |
| **API7:2023 Server Side Request Forgery (SSRF)** | An observed URL-fetch feature on a listed asset; callback testing is permitted. | Use a researcher-controlled canary only. Do not probe internal addresses, cloud metadata, or third-party destinations. |
| **API8:2023 Security Misconfiguration** | A listed asset and an observed route or program-documented configuration surface. | Review supplied documentation and ordinary responses. Do not enumerate hidden files, methods, or routes unless explicitly allowed. |
| **API9:2023 Improper Inventory Management** | Program-listed assets or versions and their supplied inventory. | Compare only versions already in scope. Newly found hosts or versions remain leads until the scope contract confirms them; do not rotate hostnames or version paths. |
| **API10:2023 Unsafe Consumption of APIs** | An observed integration the target consumes. | Assess the target's documented or observable handling. Do not contact, modify, or test the third-party provider; use a controlled integration only if the program permits it. |

The label “API Top 10” is a triage aid, not proof of impact or a severity
rating. For injection classes that are not separate 2023 risks, use the
specific class name and its specialist reference; do not force a 2023 ID.

## Severity and reporting

- Base priority on reproducible, in-scope impact, affected data or actions,
  required privileges, and the program's rating rubric—not the risk-class
  name.
- Demonstrate impact with controlled accounts and records. Do not read,
  change, or delete real-user or business data to strengthen a report.
- For resource-consumption issues, describe the control gap without causing
  load or disruption.
- For a newly observed host, route, or version, record a lead and resolve
  scope before testing or reporting it as an in-scope asset.

## Historical reference — OWASP API Security Top 10 2019 scores

The following values are the legacy 2019 table retained for historical
reference. They predate the 2023 categories and must not be used to rank
current risks, assign severity, or choose a live test.

| Historical 2019 risk | E (exploitability) | P (prevalence) | D (detectability) | T (technical impact) |
|---|---:|---:|---:|---:|
| API1 Broken Object Level Authorization | 3 | 3 | 2 | 3 |
| API2 Broken User Authentication | 3 | 2 | 2 | 3 |
| API3 Excessive Data Exposure | 3 | 2 | 2 | 2 |
| API4 Lack of Resources & Rate Limiting | 2 | 3 | 3 | 2 |
| API5 Broken Function Level Authorization | 3 | 2 | 1 | 2 |
| API6 Mass Assignment | 2 | 2 | 2 | 2 |
| API7 Security Misconfiguration | 3 | 3 | 3 | 2 |
| API8 Injection | 3 | 2 | 3 | 3 |
| API9 Improper Assets Management | 3 | 3 | 2 | 2 |
| API10 Insufficient Logging & Monitoring | 2 | 3 | 1 | 2 |

## Working rules

- A listed host does not put its subdomains, vendors, or shared infrastructure
  in scope.
- A route being observed does not authorize every method on it; re-check the
  method and technique for each hypothesis.
- Two accounts are useful only when both are controlled by the researcher and
  the policy permits that comparison.
- Treat unknown permissions as blockers, not as missing details to infer.
- Stop on unexpected personal data, a state change outside the planned test,
  service degradation, or a scope boundary.
