# Ch7: Exfiltration, Persistence Mechanics & Detection Surface

Source: `aws/exploitation/` (s3-bucket-replication-exfiltration, s3_server_access_logs, s3_streaming_copy), `aws/post_exploitation/` (iam_persistence*, lambda_persistence, iam_roles_anywhere, iam_rogue_oidc, s3_acl, user_data_script, codebuild runner, survive_access_key_deletion, network-firewall bypass, download_tools), `aws/avoiding-detection/*`, `gcp/avoid-detection/apps-script-abuse.md`. Two uses: (a) describe what an attacker *could* do next for impact narratives; (b) tell the blue team exactly which telemetry your PoC generated.

## Data-egress channels worth naming

- **S3 replication backdoor** — `PutReplicationConfiguration` + assumable role silently mirrors current and future objects cross-account; Batch Operations sweeps existing objects. Evidence to cite: replication config in `get-bucket-replication`.
- **Streaming copy** — CLI pipe victim→attacker bucket (ch04); blocked by VPC-endpoint `principalOrgId` policies; cross-region destinations may dodge the endpoint.
- **Server access logs** — denied `GetObject` key names/User-Agents still land in the attacker's logging bucket; 1024-byte key limit; VPC endpoints without the attacker bucket allowed will drop it.
- **`--endpoint-url` LOLBin** — AWS CLI ships on Amazon Linux; pointing it at an attacker MinIO moves tools/data under a benign-looking process (egress filtering via Network Firewall SNI/Host is spoofable — ch06).

## Persistence mechanics (report as "attacker-could" — never implement)

Access keys on existing users (`iam:CreateAccessKey`), login profiles, `GetFederationToken` creds surviving key deletion, rogue OIDC IdP + backdoored role, IAM Roles Anywhere trust anchor (attacker CA → cert-based assume-role from outside), Lambda runtime bootstrap backdoor, user-data script edits/S3-script poisoning, S3 ACL grants to external accounts (survives Public Access Block since account-targeted sharing isn't "public"), CodeBuild runner loop, eventual-consistency window (~4 s between control-plane change and authz propagation lets an attacker dodge containment), GCP Apps Script `sys-…` projects hidden from the console project list (free SAs even unbilled).

## The detection surface — what your actions reveal

**GuardDuty findings to name in reports**
- `PenTest:IAMUser/{Kali,Parrot,Pentoo}Linux` — fires on the CLI/SDK *User-Agent*; bypass = proxy rewrite (Burp match/replace on `^User-Agent`) or botocore session patch. Defender note: UA is trivially spoofed — don't rely on it; baseline-normal-UAs detection is better.
- `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.{OutsideAWS,InsideAWS}` — EC2 role creds used off-instance / off-account. VPC endpoints historically bypassed it (SneakyEndpoints); since Oct-2024 GuardDuty uses CloudTrail *network activity events* — coverage now spans ~26 services, shrinking the bypass. Lambda creds have **no** equivalent finding.
- `UnauthorizedAccess:EC2/TorClient` — Tor guard connections; bridges/obfs4 evade.
- **GuardDuty tampering as evidence** — `UpdateDetector --no-enable`, removing S3Logs/Kubernetes data sources, `finding-publishing-frequency SIX_HOURS`, adding attacker IPs to Trusted IP lists (DNS findings exempt): each is a strong compromise IoC to flag.

**CloudTrail blind spots**
- Data events (S3 object-level, Lambda invoke, `sqs:ListQueues`, `pinpoint-sms-voice`, `timestream-query`) aren't logged by default → whoami-without-logs (ch03) and quiet reads.
- SSM command text, Cognito sign-up username/attributes → `HIDDEN_DUE_TO_SECURITY_REASONS`.
- enumerate-iam = CloudTrail flood; brute-force perms last.
- `sts:GetCallerIdentity` from a first-time caller is a classic tripwire — defenders alert on it.
- Console sessions mint `ConsoleLogin` + browser UAs — obvious in logs.
- User-data modifications require `StopInstances` — a stoppable production instance is itself a finding.

**Bounty-safe validation**: run nothing in this chapter against a target. Value = accuracy of the narrative: "these creds can (a) read X, (b) evade Y detection because finding Z doesn't cover it" is a stronger report than an executed exfil. Map every PoC action you *did* take to its CloudTrail event so triage can confirm.
