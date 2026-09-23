# Glossary — Hacking the Cloud

Cloud terms as used across the encyclopedia and this skill's chapters.

## Identity & credentials

- **ARN** — `arn:aws:service:region:account:resource`; the universal resource address; leaks in AccessDenied errors.
- **AKIA / ASIA** — access-key prefixes: long-lived IAM-user key / short-lived STS key (needs session token). Other unique-ID prefixes: AIDA user, AROA role, AGPA group, AIPA instance profile, ANPA managed policy, ANVA policy version, APKA public key, ASCA cert, ABIA STS bearer token, ACCA context cred.
- **Session token** — third component of ASIA creds (base64, long); required in `AWS_SESSION_TOKEN`.
- **Principal** — entity a policy grants to: account root, user, role, role session, AWS service, federated IdP, or `*` (everyone).
- **Trust policy** — resource-based policy on a role defining who may `AssumeRole`/`AssumeRoleWithWebIdentity`.
- **Identity-based vs resource-based policy** — attached to a principal vs attached to the resource (bucket policy, SNS access policy, ECR repo policy, role trust).
- **Implicit vs explicit deny** — no Allow statement vs an actual `Deny`; resource-policy Allow overrides implicit identity deny same-account.
- **NotPrincipal / NotAction / NotResource** — inverted policy elements; catastrophic with `Allow`.
- **Permissions boundary** — ceiling on what a principal's policies can grant; deletable with the right `iam:` permission.
- **SCP / RCP** — organization-level Service/Resource Control Policy ceilings over member accounts/resources.
- **Service-linked role** — AWS-managed `AWSServiceRoleFor*` role; its existence fingerprints which services an account uses.
- **OrganizationAccountAccessRole** — default admin role in Org-created member accounts trusting the management account.
- **sts:AssumeRole / role chaining** — swap identity for temp creds; chaining refreshes expiry.
- **sts:GetFederationToken** — IAM-user → temp creds that outlive parent-key deletion; API can't call IAM/STS (console bypass).
- **OIDC provider / AssumeRoleWithWebIdentity** — federate GitHub/GitLab/Terraform Cloud into AWS; missing `sub` conditions = public role assumption.
- **Cognito User Pool vs Identity Pool** — user auth directory vs AWS-credential vending machine (JWT → ASIA creds via mapped IAM role).
- **Managed Identity (Azure)** — identity attached to a compute resource; tokens from `$IDENTITY_ENDPOINT` or IMDS; system- (dies with resource) vs user-assigned.
- **Service account (GCP)** — `<name>@<project>.iam.gserviceaccount.com`; key JSON files (`serviceaccount.json`…) or metadata-server tokens; `…-compute@developer.gserviceaccount.com` = Compute default.
- **tagBinding** — GCP link attaching a tag value to a resource; satisfies `resource.matchTag` IAM conditions → privesc path.

## Metadata & compute

- **IMDS / IMDSv1 / IMDSv2** — EC2 metadata at `169.254.169.254` (IPv6 `fd00:ec2::254`); v2 needs a PUT-fetched token, drops X-Forwarded-For, hop-limit 1 default.
- **User data** — EC2 bootstrap script at `/latest/user-data/`; runs as root on first boot (or every boot with cloud-config tricks); common secret spill.
- **Instance identity document** — `/latest/dynamic/instance-identity/document`: accountId, region, AZ, IPs — credential-free reachability proof.
- **Lambda runtime interface** — `http://$AWS_LAMBDA_RUNTIME_API/2018-06-01/runtime/invocation/next` returns invocation event; creds live in `/proc/self/environ`.
- **CloudShell credential endpoint** — `localhost:1338` IMDS-alike converting console session → API creds.
- **SSM Agent / EC2 Messages** — host agent polls SSM; SendCommand/StartSession are the exec primitives; spoofable to intercept sessions.
- **VPC endpoint** — private AWS-API routing; historically dodged credential-exfil findings; policies can pin `principalOrgId`.
- **Connection tracking** — SG statefulness: established flows survive later rule lockdowns.
- **SNI / Host-header egress bypass** — Network Firewall domain rules trust TLS SNI or HTTP Host, never resolve DNS → spoofable.

## Storage & data

- **Public Access Block** — S3 account/bucket switch blocking *public* grants; doesn't stop sharing to a named external account.
- **AUTHENTICATED USERS / ANY USER** — S3 ACL grantees: any AWS principal / truly anonymous.
- **EBS snapshot / AMI** — disk backup / machine image; both can be made public and listed API-wide (`--restorable-by-user-ids all`, `describe-images --owners`).
- **SAS token / connection string** — Azure scoped bearer URI vs full-account credential string.
- **Soft delete** — Azure blob recovery window (default 7 days).
- **GCS `allUsers`/`allAuthenticatedUsers`** — GCP bucket anonymous grants.

## Telemetry & tooling

- **CloudTrail management vs data events** — control-plane logged by default; high-volume data-plane (object reads, invokes, some list calls) not.
- **GuardDuty finding families** — `PenTest:IAMUser/*` (UA), `InstanceCredentialExfiltration.*`, `EC2/TorClient`, S3/K8s protection sets.
- **`HIDDEN_DUE_TO_SECURITY_REASONS`** — CloudTrail redaction for SSM params, Cognito fields.
- **enumerate-iam / Pacu / Prowler / quiet-riot / s3-account-search / MicroBurst / aws-vault / SneakyEndpoints / Stratus Red Team** — perms brute-force, AWS exploitation framework, CSPM auditor, cross-cloud enum, bucket→account-ID, Azure blob enum, keys→console, VPC-endpoint lab, attack-technique detection library.
- **CloudGoat / iam-vulnerable / GCP Goat / Thunder CTF** — practice environments.
