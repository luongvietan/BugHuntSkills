# Patterns — Reusable Heuristics from Hacking the Cloud

Cross-cutting cloud heuristics. Per-domain mechanics live in `chapters/`; web-layer hunting lives in `bug-bounty-bootcamp`/`owasp-wstg`.

## The master chain

```
Web bug (SSRF/XXE/LFI) → metadata endpoint → IAM creds → permission map →
resource policies (S3/SNS/SQS/KMS/ECR) → data or hosted-content impact
```

Each hop multiplies severity: blind SSRF alone is medium; SSRF returning ASIA creds for a role that can read production buckets is critical. Document every hop you confirmed plus the *next* hop the creds enable — that second half is the impact statement.

## Universal cloud heuristics

- **The resource is the attack surface.** Apps front policies; policies front resources. The same bucket/queue/key is reachable from the web app, the CLI, and any creds — test the policy, not the page.
- **Error messages are an oracle.** AccessDenied leaks caller ARN+account; deny-all session policies classify public-vs-private resource policies; sign-up endpoints leak user existence when login is hardened; `NoSuchBucket` used to leak takeover targets. Whenever AWS/Azure/GCP answers *why* it refused, that's a recon primitive.
- **Existence checks don't need auth.** Role/user name enum via your own trust policy, root email via console errors, account ID via bucket/key/or error — assemble the target map before touching it.
- **Defaults are the vuln class.** Cognito self-signup, IMDSv1, unauthenticated identity pools, wildcard principals, missing OIDC `sub` conditions, non-retroactive org policies, soft-delete retention — hunt the knobs nobody turned.
- **Temp creds have a clock.** ASIA tokens die in 15min–hours: if the policy authorizes validation, one `get-caller-identity` immediately — otherwise report the leak now; re-checking later may show "dead creds" that were live during the PoC window. Say so in the report.
- **Your account is the clean room.** s3-account-search, quiet-riot, session-policy probes, trust-policy enum all log to *your* CloudTrail. Set up a research account with an SCP blocking expensive/destructive calls before hunting.
- **Naming is intelligence.** `OrganizationAccountAccessRole`, `AWSServiceRoleFor*`, GCP `…-compute@developer.gserviceaccount.com`, Apps Script `sys-*` projects, bucket naming conventions, backup tags — defaults and conventions turn guessing into enumeration.
- **Deleted ≠ gone.** Dangling CNAME→bucket takeovers, soft-deleted blobs (7 days), public snapshots/AMIs of retired systems, recreated-role ARN staleness — always check the graveyard.
- **Write beats read for impact, read beats write for safety.** A writable public bucket hosting JS = supply-chain bug (report the capability); but your PoC should be a marker file you delete, not a defaced asset.
- **Every cloud has a metadata endpoint** — AWS `169.254.169.254`, GCP `metadata.google.internal` + `Metadata-Flavor: Google`, Azure `169.254.169.254` + `Metadata: true`, Lambda env/`/proc`, CloudShell `localhost:1338`, TFE via remote-exec. Found SSRF? Try them all — header requirements are the only gate.
- **PassRole is the privesc pivot.** `iam:PassRole` + any service that runs as a role (EC2, Lambda, ECS, Glue, CodeBuild, CloudFormation…) = execute as a more privileged principal. In GCP the analog is "actAs"/SA-key/`signBlob` paths.
- **Logging gaps define the quiet path.** Data events off by default, `HIDDEN_DUE_TO_SECURITY_REASONS` fields, first-call `GetCallerIdentity` alerts, UA-based PenTest findings — know which of your calls write logs to plan PoCs and to write credible detection guidance.
- **Containment has seams.** Eventual-consistency ~4s window, connection-tracking persistence through SG lockdown, federated-token survival past key deletion, GuardDuty tampering — reportable as resilience-of-compromise evidence.

## Workflow patterns

- **Validate up the ladder**: free/anonymous checks → error-oracle probes → (gated) one credential validation if authorized → read-only list/describe → canary write+delete → stop. Never descend into data to "see how bad it is."
- **Chain accounting for cloud**: SSRF→metadata→creds→`get-caller-identity`→(role name implies permissions)→storage. Write the whole hypothetical chain; confirm the hops you can.
- **Lab-first for dangerous steps**: CloudGoat/iam-vulnerable/Stratus reproduce privesc, exfil, and detection-evasion paths in your own account — cite the reproduction in the report instead of running it on the target.
- **Map every action to its trail**: before each PoC call, know the CloudTrail event it writes; include the list in your report so triage can verify and defenders can alert.

## Anti-patterns

- Dumping bucket contents or reading personal data to prove exposure — a listing suffices.
- Running enumerate-iam or persistence techniques on a target — CloudTrail flood, real damage, policy violations.
- Reporting "SSRF found" without testing the metadata hop (or explaining why IMDSv2 blocks it) — the hop *is* the finding.
- Treating `ASIA`-credential expiry in retest as "not vulnerable."
- Claiming privesc you didn't verify — "the policy permits X" with evidence beats "I ran X."
