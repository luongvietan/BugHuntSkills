# Ch8: Multi-Cloud General Knowledge, GCP/Azure & Labs

Source: `aws/general-knowledge/` (aws_cli_tips, aws_organizations_defaults, block-expensive-actions-with-scps, why_recreating_an_iam_role, connection-tracking covered in ch06), `gcp/general-knowledge/*`, `gcp/enumeration/enumerate_all_permissions.md`, `gcp/avoid-detection/apps-script-abuse.md`, `azure/*`, `terraform/*`, `*/capture_the_flag/*`. Provider-agnostic model: control-plane identity → resource policies → data-plane access. Details differ; the hunting loop doesn't.

## AWS general knowledge worth internalizing

- **Organizations model** — management account owns the org; member accounts carry `OrganizationAccountAccessRole` (admin, trusts mgmt) — the built-in lateral path (ch06). Invited accounts may lack it.
- **SCPs as guardrails (yours)**: attach a "block expensive/destructive" SCP to your *own* research org — blocks accidental `Create*` on costly services. Never assume a policy blocks everything; it's a seatbelt not a wall.
- **Recreated-role gotcha**: delete + recreate an IAM role/user with the same name → new unique ID → trust policies referencing the ARN silently break (console renders the stale principal as the old unique ID). Explains "why did cross-account access die?" and avoids identity-spoofing via name reuse.
- **CLI tricks**: `-` = stdin/stdout (`aws s3 cp s3://b/k -`, `echo x | aws s3 cp - s3://b/k` — no local files); `AWS_EXECUTION_ENV` env var alters the CloudTrail user-agent; `--endpoint-url` retargets to any S3-compatible store.

## GCP model & enumeration

- Hierarchy: org → folders → projects (≈AWS accounts); org policies inherit down and are **mostly non-retroactive** — existing resources keep violating settings created later (`compute.vmExternalIpAccess`, `vmCanIpForward`, `requireShieldedVm`, `iam.disableServiceAccountKeyCreation`); a same-named new VM can satisfy a stale allowlist.
- **Default service accounts** — `<name>@<project>.iam.gserviceaccount.com`, `<project>@appspot.gserviceaccount.com` (App Engine), `<project-number>-compute@developer.gserviceaccount.com` (Compute default). Key files hide as `serviceaccount.json`, `sa-private-key.json`, `service-account-file.json`; activate via `GOOGLE_APPLICATION_CREDENTIALS`.
- **Permission discovery**: `gcloud`/`enumerate_all_permissions` workflow — test `iam.serviceAccounts.getIamPolicy`, iterate roles; service-account permission sets map to the privesc table (ch05). Quiet-riot enumerates Workspace emails (ch01).
- **Apps Script shadow projects** — every deployed Apps Script makes a hidden `sys-<26-digit>` project under `system-gsuite/apps-script`, invisible in the console project list (visible via `gcloud`); attackers mimic the naming to hide SAs/compute — detection = reconcile project list API vs console.

## Azure model

- Managed Identities (system vs user-assigned) attach roles to resources — scope propagates down subscription → RG → resource; over-scoped identity + one RCE/SSRF = whole-subscription read (ch02 for token theft flow: `$IDENTITY_ENDPOINT`/`$IDENTITY_HEADER` → `Connect-AzAccount` → `Get-AzResource` → `Get-AzStorageAccountKey`).
- Storage: account → containers → blobs; three anon levels (Private/Blob/Container), `?restype=container&comp=list` listing (ch04); SAS URI = scoped bearer URL; connection string = full account power; soft-delete retains 7 days by default.
- Run Command (`runCommands/action`) = SYSTEM/root script on VMs; email enum via quiet-riot/enum_email_addresses for target mapping.

## Terraform supply chain

- **TFE metadata pivot** — stolen `.atlasv1.` token → org/workspace API → remote backend → `external`/`local-exec` provider executes during `plan` → TFE's internal metadata → cloud creds (ch02).
- **ANSI escape evasion** — malicious modules hide `local-exec` payloads in plan output via `\033[2K`/`\033[A` line-clear sequences; visible only in raw logs — review .tf output as text, not rendered terminal.

## Practice labs & CTFs (safe proving grounds)

CloudGoat scenarios (`vulnerable_cognito` etc.), iam-vulnerable (all 21+ privesc paths), SneakyEndpoints (VPC-endpoint lab), GCP Goat, Thunder CTF, cicdont, Stratus Red Team technique library (detection validation). Reproduce chains in your own account before claiming them on a target.

**Bounty-safe validation**: nothing in this chapter is target-actionable on its own — it's context for reading findings and lab material for practicing the chains in ch02–ch07. Validate every technique in your own account (SCP-guardrailed) or a listed lab before writing it into a report; organizational/structural claims about a target (org membership, service-linked roles, default role presence) still need authorization — confirm via the unauth-enum oracles in ch01 and document rather than probe further.
