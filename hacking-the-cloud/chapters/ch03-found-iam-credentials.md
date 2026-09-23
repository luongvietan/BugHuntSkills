# Ch3: Found IAM Credentials — Identify, Validate, Scope

Source: `aws/general-knowledge/using_stolen_iam_credentials.md`, `iam-key-identifiers.md`, `aws/enumeration/whoami.md`, `get-account-id-from-keys.md`, `brute_force_iam_permissions.md`, `aws/post_exploitation/` (console session, creds-from-console, role-chain-juggling, GetFederationToken), `aws/exploitation/cognito_identity_pool_excessive_privileges.md`. Leaked keys turn up in JS, mobile apps, repos, public AMIs/snapshots — the playbook is: identify → minimal validation → permission mapping.

## Anatomy of a key

| Prefix | Type | Lifespan | Notes |
|---|---|---|---|
| `AKIA` | Long-term IAM user key | Until revoked | Highest report value — durable access |
| `ASIA` | STS temporary creds | 15 min – hours | Needs `AWS_SESSION_TOKEN` too; validate fast |
| `AIDA`/`AROA` | Unique IDs (user/role) | — | Reverse to ARN via trust policy (ch01) |

Other prefixes: AGPA group, AIPA instance profile, ANPA managed policy, ANVA policy version, APKA public key, ASCA cert, ABIA STS bearer token, ACCA context cred.

## Using & validating

```bash
export AWS_ACCESS_KEY_ID=ASIA… AWS_SECRET_ACCESS_KEY=… AWS_SESSION_TOKEN=… AWS_REGION=us-east-1
aws sts get-caller-identity          # always works (can't be denied); returns ARN + account ID
```

- `get-caller-identity` logs to CloudTrail — may be alerted on first-time callers. Quieter whoami via *unlogged* calls whose AccessDenied leaks the identity anyway: `aws sqs list-queues`, `aws pinpoint-sms-voice send-voice-message`, `aws timestream-query describe-endpoints` (SQS/pinpoint/timestream data-plane actions don't write CloudTrail management events).
- Account ID only: `aws sts get-access-key-info --access-key-id AKIA…` (logs to your account) or offline base32 decode (ch01).
- Check for dangling-but-scope-relevant context: is the account ID in program scope *before* further calls?

## Mapping permissions

- **enumerate-iam** — brute-forces Get/List/Describe across services: `./enumerate-iam.py --access-key … --secret-key … --session-token …`. Non-destructive by design (no mutating calls) but **extremely noisy** — thousands of CloudTrail events; last resort or own-account only. Refresh its API list from `aws-sdk-js` for newer services.
- Cheaper situational awareness: service-linked role enum (ch01) reveals which services exist; `backup:List*` shows what the org protects; the session-policy probe distinguishes "allowed" from "denied by resource policy" without touching data.

## Converting access forms

- **Keys → console**: `aws-vault login` (`-s` prints URL). ASIA creds work directly; AKIA needs `sts:GetFederationToken` or `sts:AssumeRole` first. Generates `ConsoleLogin` + browser user-agent anomalies — loud.
- **Console → keys**: browser session → CloudShell → `localhost:1338` metadata endpoint (ch02). Scenario: cookies found on a dev box but no `.aws` creds.
- **Refresh expiry**: role-chain juggling — assumed-role creds can `AssumeRole` again (incl. the same role or mutual trust pairs), refreshing the expiration each hop; AWSRoleJuggler automates.
- **Outlive key deletion**: `sts:GetFederationToken --name x --duration-seconds 129600` mints ASIA creds that remain valid after the parent key is deleted. Optional `--policy-arns`/inline policy scopes the result to the *intersection* of your permissions and the passed policy — a real attacker attaches the broadest plausible policy to capture the identity's full access (and a highly privileged or very long-lived session is itself a detection signal). Limitation: no IAM/STS calls via API — *except through a console session*.
- **Cognito pools → AWS creds**: `aws cognito-identity get-id --identity-pool-id … --logins <provider>:<id_token>` then `get-credentials-for-identity` — works unauthenticated if the pool allows it. Pool IAM role privileges decide impact: worst case = account takeover; least-privilege pools = read-only demo.

**Bounty-safe validation**: a leaked `AKIA` + secret + one `get-caller-identity` screenshot is a complete, reportable PoC — redact the secret in the report, note key age if determinable. Do not enumerate resources, read buckets, or escalate; describe *potential* impact (what the role name implies) rather than proving it. `ASIA` creds: validate once immediately — they may be gone in an hour.
