# Patterns & Techniques — OWASP API Security Top 10:2023

## Before selecting a pattern

The exact host and path must be in the current scope contract. The endpoint
must be observed in an authorized flow or supplied by the program. Verify the
specific technique, HTTP method, account/data conditions, rate, and impact
boundary before each check. An unobserved route, newly found asset, or method
not explicitly covered by policy is a lead or a `blocked-by-policy` card—not
an invitation to discover or try it. Use only accounts and records you
control.

## Object authorization — API1:2023 BOLA

**When to use:** An observed, in-scope object request whose exact method is
permitted, and the program allows controlled-account authorization checks.

**Bounded check:** Create an object under account A that you control. If the
program also permits cross-account testing, compare access from your own
account B using only that known object ID. Stop if any record not created by
you appears. Do not guess, increment, scrape, or enumerate identifiers.

## Function authorization — API5:2023 BFLA

**When to use:** A documented or observed function and researcher-controlled
roles exist on a listed asset.

**Bounded check:** Compare the same observed function across roles you
control, using only the method explicitly permitted for that function. Do
not mutate methods, infer administrator routes, or append guessed paths.

## Object properties — API3:2023 BOPLA

**When to use:** An observed response or mutation concerns a record you
control, and its exact read/write method is permitted.

**Bounded check:** Compare returned fields against the feature's documented
purpose and the roles you control. For write-side checks, use only a harmless
property on your own record. Do not submit privilege, balance, ownership, or
verification changes unless the program explicitly authorizes that precise
test and its impact boundary.

## Resource consumption — API4:2023

**When to use:** A program document or an observed response identifies a
limit on an observed operation.

**Bounded check:** Start with passive review of the documented limit. Do not
send bursts, oversized values, repeated expensive requests, or requests
intended to consume CPU, memory, quota, or shared capacity. A single minimal
request is allowed only when that exact test is covered by policy. Stop at
any latency, error-rate, or service-health change.

## Sensitive business flows — API6:2023

**When to use:** An observed purchase, booking, vote, redemption, or similar
flow has a documented constraint and the program permits testing that
specific side effect.

**Bounded check:** Prefer a non-committing or sandbox path. Otherwise use
only a researcher-owned account and record, and the smallest policy-permitted
action. Do not automate or repeat purchases, bookings, votes, redemptions, or
other state changes. Do not affect inventory, payments, notifications, or
other people.

## Server-side request forgery — API7:2023

**When to use:** An observed in-scope feature fetches a URL, and the program
explicitly permits callback/canary testing.

**Bounded check:** Use a researcher-controlled canary endpoint and one
minimal request. Do not target loopback, private networks, metadata services,
redirect chains to internal systems, or third-party hosts.

## Security misconfiguration — API8:2023

**When to use:** A listed asset has a supplied configuration document or an
observed route relevant to the reportable issue.

**Bounded check:** Review the supplied document and ordinary responses for
the observed route. Do not enumerate dotfiles, hidden paths, methods, or
error conditions unless the exact discovery technique is allowed. Exclude
best-practice-only findings if the program excludes them.

## Inventory management — API9:2023

**When to use:** The program lists multiple assets or API versions, or its
documentation supplies an inventory to compare.

**Bounded check:** Compare only those listed assets and versions. Do not
rotate hostname guesses, try unlisted versions, or expand from a link to a
new host. Record newly observed assets as leads and request a scope
clarification before testing them.

## Unsafe consumption of APIs — API10:2023

**When to use:** An observed target feature consumes a third-party API and
the target's handling can be reviewed without contacting or changing the
provider.

**Bounded check:** Review supplied documentation or target-side behavior in
an authorized flow. Do not test, alter, or send requests to the provider. Use
a researcher-controlled integration or canary only when the program
explicitly permits it.

## Authentication — API2:2023

**When to use:** An observed login, session, reset, or token flow is covered
by the program's authentication-testing and rate rules.

**Bounded check:** Use researcher-controlled accounts and the minimum
permitted request count. Do not spray credentials, probe other people's
accounts, trigger lockouts, or treat silence about rate as authorization.

## Other injection classes

SQL, NoSQL, command, and other injection vulnerabilities remain valid bug
classes but are not current API Top 10:2023 IDs. Start only from an observed
input and an explicitly permitted technique. Keep proofs non-destructive and
limited to data you control; never alter/delete backend data or read more
than the minimum needed to show impact. Use the relevant specialist skill
for the class-specific validation details.
