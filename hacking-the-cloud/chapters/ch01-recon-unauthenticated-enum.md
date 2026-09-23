# Ch1: Recon & Unauthenticated Enumeration

Source: `aws/enumeration/`, `gcp/enumeration/`, `azure/enum_email_addresses.md`. Surprising amount of AWS structure is discoverable with zero or near-zero access — most techniques log to *your* account, not the target's. The goal: turn a company name / domain into account IDs, principal names, and exposed resources.

> **Scope label:** unauthenticated enum hits *public* endpoints (resource
> policies, public snapshots, email-validity oracles) and is default-safe.
> Anything run *with* credentials is a different scope class — it requires
> credentialed cloud scope in the policy (a test account the program gave
> you), and "the key I found" is never that account (see ch03).

## Account ID discovery

- **From an access key** — `aws sts get-access-key-info --access-key-id AKIA...` returns the account ID and logs only to *your* account. Offline alternative: the account ID is encoded in the key itself — strip the 4-char prefix, base32-decode, mask `0x7fffffffff80`, shift right 7 bits (Aidan Steele / Tal Be'ery method).
- **From a public S3 bucket** — the `s3:ResourceAccount` policy condition supports wildcards, so you can binary-search the 12-digit ID by progressively tightening the condition in a policy on *your* bucket/role. Tool: `s3-account-search <your-role-arn> <bucket>`. Needs your own AWS account + assumable role with `s3:GetObject`/`s3:ListBucket`.
- **From an EC2/SSRF foothold** — `http://169.254.169.254/latest/dynamic/instance-identity/document` returns `accountId`, region, AZ, instance ID (see ch02).
- **From error messages** — AccessDenied responses to calls like `sqs:ListQueues` leak the caller's ARN *and* account ID (see ch03).

## Enumerating principals & existence (no auth needed)

- **IAM users/roles via resource policies** — reference a guessed principal in a policy on *your* resource: a role trust policy `Principal: arn:aws:iam::<acct>:role/<name>` saves only if the role exists; a bucket-policy `Deny` on `...:user/<name>` likewise. Scale with **quiet-riot** (also does Azure AD / Google Workspace email enum), **pacu `iam__enum_roles`**, or the `enumerate_iam_using_bucket_policy` script. Noise lands in your account's CloudTrail (`UpdateAssumeRolePolicy`/`PutBucketPolicy`) — use your own credentials, never target creds.
- **Service-linked roles as service fingerprint** — the same trick enumerates `AWSServiceRoleFor*` roles, revealing whether the account uses GuardDuty, Organizations, ECS/EKS, etc.
- **Unique ID → ARN** — a found `AIDA…`/`AROA…` unique ID pasted as `Principal` in your own role trust policy resolves to the full ARN on save (revealing account ID + role name).
- **Root email via console** — the AWS sign-in page with "Root user" selected tells you whether an email owns an account ("An AWS account with that sign-in information does not exist" vs. password prompt).
- **Verbose error oracle (session-policy probe)** — assume your own role with a deny-all session policy, then call the target resource: `"deny in a session policy"` = resource policy is public; `"deny in a resource-based policy"`/`"no resource-based policy allows"` = private/not-public; `"no identity-based policy allows"` = ambiguous. Works on SNS, SQS, Lambda, KMS, ECR, EventBridge (sns-buster automates SNS); SCPs/RCPs can mask results; S3 cross-org still returns generic Access Denied.

## Cognito discovery (very bounty-relevant)

- **User Pool Client ID is not secret** — find it in JS (`AWSCognito.config.update({UserPoolId, ClientId})`) or decompiled mobile apps.
- **User enum bypass** — `Prevent user existence errors` covers `initiate-auth` but *not* `cognito-idp:SignUp`: existing username → `UsernameExistsException`; new → signup JSON. Detection angle for reports: spike in `Unconfirmed` users; CloudTrail hides username/userAttributes.
- **Self-signup enabled** — if "Admin Only" signup isn't set, `aws cognito-idp sign-up --client-id ...` creates an account even with no signup UI → leads to identity-pool creds (ch03).
- **Identity pools** — `aws cognito-identity get-id` + `get-credentials-for-identity` mint AWS creds; unauthenticated pools need no login at all. Permissions = whatever IAM role the pool maps to.

## Exposed-resource sweeps

- **Public EBS snapshots** — `aws ec2 describe-snapshots --restorable-by-user-ids all --owner-ids <acct>` lists every snapshot the account made public (tens of thousands exist globally); restore → mount → mine for keys/source. Making one public fires `ec2:ModifySnapshotAttribute` with `groups:all`.
- **Public AMIs** — `aws ec2 describe-images --owners <acct> --include-deprecated` per region; launch a t2.micro in *your* account and scan (`find / -name credentials -o -name id_rsa`, truffleHog/gitleaks).
- **AWS Backup inventory** — with any `backup:List*`/`Describe*`, `aws backup list-protected-resources` + recovery-point/job listing reveals which resources the org *cares about*, tagging/naming conventions, and backup cadence — quieter than per-service Describe* calls.
- **GCP/Azure emails** — quiet-riot / `enum_email_addresses` techniques validate Workspace/AAD addresses → feed root-email and phishing-surface findings.

**Bounty-safe validation**: existence ≠ access. Prove the oracle works (one known-bad + one known-good guess), document the confirmed name, stop. Enumerating *content* of public snapshots/AMIs you don't own is out-of-bounds for most programs — a listing screenshot plus policy evidence is the report.
