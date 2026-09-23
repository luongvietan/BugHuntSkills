# Ch3: Found IAM Credentials — Identify, Validate, Scope

Source: `aws/general-knowledge/using_stolen_iam_credentials.md`, `iam-key-identifiers.md`, `aws/enumeration/whoami.md`, `get-account-id-from-keys.md`, `brute_force_iam_permissions.md`, `aws/post_exploitation/` (console session, creds-from-console, role-chain-juggling, GetFederationToken), `aws/exploitation/cognito_identity_pool_excessive_privileges.md`. Leaked keys turn up in JS, mobile apps, repos, public AMIs/snapshots.

> **Default = identify + report, never use.** A found credential is reported,
> not exercised: record key type (prefix table below), where it leaked, and a
> masked prefix (`AKIA…AB12`) — never the secret value. **Credential use of
> any kind requires the policy to explicitly authorize credential
> validation**, plus confirmation the key belongs to the program (offline
> account-ID decode, key context). The maximum authorized check is a single
> `get-caller-identity` — everything else on this page is *impact narrative*
> for the report, not a procedure to run on a found key.

## Anatomy of a key

| Prefix | Type | Lifespan | Notes |
|---|---|---|---|
| `AKIA` | Long-term IAM user key | Until revoked | Highest report value — durable access |
| `ASIA` | STS temporary creds | 15 min – hours | Needs `AWS_SESSION_TOKEN` too; validate fast |
| `AIDA`/`AROA` | Unique IDs (user/role) | — | Reverse to ARN via trust policy (ch01) |

Other prefixes: AGPA group, AIPA instance profile, ANPA managed policy, ANVA policy version, APKA public key, ASCA cert, ABIA STS bearer token, ACCA context cred.

## Validating — gated, single-check maximum

```bash
export AWS_ACCESS_KEY_ID=ASIA… AWS_SECRET_ACCESS_KEY=… AWS_SESSION_TOKEN=… AWS_REGION=us-east-1
aws sts get-caller-identity          # always works (can't be denied); returns ARN + account ID
```

- **Gate:** run this only when the policy explicitly authorizes credential
  validation AND offline evidence says the key is the program's (account ID
  decode below, leak context). Otherwise the masked-prefix report IS the PoC.
- `get-caller-identity` logs to CloudTrail — may be alerted on first-time
  callers. (Stealth variants exist — unlogged data-plane calls whose
  AccessDenied leaks the identity — but evading the target's telemetry is
  not bounty behavior; use the logged, honest call.)
- Account ID only, offline: base32-decode the key ID (ch01) — the safe
  ownership check that makes zero calls. `aws sts get-access-key-info` logs
  to *your* account as an alternative.
- Check for dangling-but-scope-relevant context: is the account ID in
  program scope *before* further calls? A key belonging to a third party
  (vendor, SaaS, acquired-but-out-of-scope org) is reported as such and
  never validated.

## Mapping permissions — narrative, not procedure

For a *found* key, permission mapping ends at the single identity check.
The techniques below describe what an attacker would do next — cite them in
the report's impact section, run them only against an account you own.

- **enumerate-iam** — brute-forces Get/List/Describe across services: `./enumerate-iam.py --access-key … --secret-key … --session-token …`. Non-destructive by design (no mutating calls) but **extremely noisy** — thousands of CloudTrail events; own-account use only, never on a found key. Refresh its API list from `aws-sdk-js` for newer services.
- Cheaper situational awareness: service-linked role enum (ch01) reveals which services exist; `backup:List*` shows what the org protects; the session-policy probe distinguishes "allowed" from "denied by resource policy" without touching data.

## Converting access forms — escalation narrative

- **Keys → console**: `aws-vault login` (`-s` prints URL). ASIA creds work directly; AKIA needs `sts:GetFederationToken` or `sts:AssumeRole` first. Generates `ConsoleLogin` + browser user-agent anomalies — loud.
- **Console → keys**: browser session → CloudShell → `localhost:1338` metadata endpoint (ch02). Scenario: cookies found on a dev box but no `.aws` creds.
- **Refresh expiry**: role-chain juggling — assumed-role creds can `AssumeRole` again (incl. the same role or mutual trust pairs), refreshing the expiration each hop; AWSRoleJuggler automates.
- **Outlive key deletion**: `sts:GetFederationToken --name x --duration-seconds 129600` mints ASIA creds that remain valid after the parent key is deleted. Optional `--policy-arns`/inline policy scopes the result to the *intersection* of your permissions and the passed policy — a real attacker attaches the broadest plausible policy to capture the identity's full access (and a highly privileged or very long-lived session is itself a detection signal). Limitation: no IAM/STS calls via API — *except through a console session*.
- **Cognito pools → AWS creds**: `aws cognito-identity get-id --identity-pool-id … --logins <provider>:<id_token>` then `get-credentials-for-identity` — works unauthenticated if the pool allows it. Pool IAM role privileges decide impact: worst case = account takeover; least-privilege pools = read-only demo.

**Bounty-safe validation (revised default)**: the report's PoC is the leak
itself — file location, key prefix/type, masked prefix, and (when authorized)
one `get-caller-identity` screenshot proving the account belongs to the
program. Do not enumerate resources, read buckets, or escalate; describe
*potential* impact (what the role name implies) rather than proving it.
`ASIA` creds expire in hours — if validation isn't authorized by policy,
report the leak immediately without it.
