# sources.md — hacking-the-cloud

## Sources

- **Primary**: Hacking the Cloud — https://hackingthe.cloud — open-source
  cloud attack encyclopedia (community-maintained; each chapter header
  lists its upstream section paths verbatim).
- **Provider docs** (authority for service behavior, IMDS versions, IAM
  semantics): AWS docs — sts `GetCallerIdentity`, IMDSv1/v2, S3/EBS/AMI
  sharing models, Cognito identity pools; GCP metadata server; Azure IMDS /
  managed identities.
- Methods-refresh source register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
  (BugBounty workspace).

## Provenance & restraint policy

- Content is a distilled attack encyclopedia — techniques are documented
  *as attackers use them*; per-chapter callouts mark which parts are
  bounty-safe probes vs. report-narrative (post-exploitation) material.
- Credential handling follows the orchestrator's scope contract: found
  credentials are identified offline and reported — never exercised — unless
  the policy explicitly authorizes credential validation and ownership is
  confirmed; even then, a single identity check is the maximum.
- Provider service details (IMDS versions, service names, API calls) drift
  with AWS/GCP/Azure releases — verify against current provider docs when a
  check depends on exact behavior.

## Review history

- 2026-09-23: credential-validation reframed to non-use-by-default (single
  gated identity check); storage/IAM evidence moved to metadata+canary;
  exfil/persistence marked narrative-only; edge scenarios added (ch08);
  sources.md created (methods-refresh T6).
