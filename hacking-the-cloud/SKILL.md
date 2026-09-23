---
name: hacking-the-cloud
description: Cloud attack encyclopedia distilled from Hacking the Cloud (hackingthe.cloud). Use when a web/API finding touches cloud infrastructure — SSRF reaching an instance metadata service, leaked AWS access keys in JS or repos, exposed S3/GCS/Azure storage, Cognito misconfigurations, dangling CloudFront or DNS takeovers — or when validating found IAM credentials and scoping cloud impact safely. Covers unauthenticated AWS enumeration (account IDs, IAM principals, public snapshots/AMIs), metadata-service abuse across AWS/GCP/Azure/Lambda, credential handling and permission discovery, IAM privilege escalation and misconfigured resource policies (wildcard principals, OIDC federation), S3/EBS/AMI/blob exposure, lateral movement via SSM/console/Organizations, exfiltration and detection notes, plus GCP/Azure/Terraform equivalents. Emphasis on the SSRF → metadata → IAM → storage kill chain and minimal-impact, bug-bounty-safe validation. Verify asset scope and method permission before live use.
---

# Hacking the Cloud — Cloud Attack Encyclopedia

Knowledge base distilled from [Hacking the Cloud](https://hackingthe.cloud) — an open encyclopedia of offensive cloud techniques (mostly AWS, with GCP/Azure/Terraform sections). The source organizes ~85 articles by domain: enumeration, exploitation, post-exploitation, detection evasion. This skill reorganizes them around the hunter's question: *"I found a cloud-shaped thing — what can I safely prove with it?"*

The signature chain: **SSRF → instance metadata → IAM credentials → cloud API → storage/secrets.** Every hop has its own chapter, and every chapter answers: how do I confirm this hop exists, and what is the *minimal* proof that demonstrates impact?

Mental model: cloud resources are governed by policies, not by the app that fronts them. A public bucket, a wildcard trust policy, or a leaked access key each bypass the web layer entirely — your job is to document the misconfiguration with the least intrusive proof possible.

Related skills: `bug-bounty-bootcamp` ch10 (SSRF hunting) and ch18 (info disclosure — where leaked keys get found), `zseano-methodology` (recon that surfaces cloud assets), `owasp-wstg` (CONF-10 cloud storage test), `web-hacking-101` (case studies).

## How to use

**The chain, hop by hop**
- Unauthenticated recon: account IDs, IAM principals, public snapshots/AMIs, Cognito/Cognito-identity-pool discovery → `chapters/ch01-recon-unauthenticated-enum.md`
- Metadata services & the SSRF → IAM hop (AWS/GCP/Azure/Lambda/Terraform) → `chapters/ch02-metadata-services-ssrf.md`
- Found or stolen credentials: identify offline, report by default; single gated identity check only when policy authorizes → `chapters/ch03-found-iam-credentials.md`
- Storage exposure: S3, EBS snapshots, public AMIs, GCS buckets, Azure blobs → `chapters/ch04-storage-buckets-snapshots.md`

**Once inside (credentialed scope only)**
- IAM privilege escalation + misconfigured resource/trust policies (wildcard, OIDC) → `chapters/ch05-privesc-misconfigured-policies.md`
- Lateral movement: SSM, console sessions, Organizations pivots, container breakouts → `chapters/ch06-lateral-movement.md`
- Exfiltration channels, persistence mechanics, GuardDuty/CloudTrail detection surface → `chapters/ch07-exfiltration-detection.md`

**Everything else**
- Multi-cloud general knowledge, GCP/Azure/Terraform specifics, labs → `chapters/ch08-multicloud-general.md`
- Terms → `glossary.md`; heuristics → `patterns.md`; quick ref → `cheatsheet.md`

## Doctrine

1. **Every cloud artifact is a policy question.** A bucket, key, role, or endpoint is only "vulnerable" relative to its policy. Read the policy (or the error that reveals it) before claiming a finding.
2. **Metadata is the crown jewel — and the most bountied hop.** SSRF that reaches `169.254.169.254` and returns IAM credentials is the canonical high-impact cloud finding. Know IMDSv2's limits (PUT token, no X-Forwarded-For, hop-limit 1) before concluding "not vulnerable."
3. **Validate up the ladder, not down.** Read-only proof first (list, describe, identity document), then a canary write you own and delete, and stop. Never browse real customer data to "prove" exposure — a directory listing or one test object is enough.
4. **Found credentials are radioactive.** `AKIA` keys live until revoked; `ASIA` keys expire in minutes-to-hours. **Default = identify offline + report** (masked prefix, leak location). A single `get-caller-identity` is permitted only when the policy *explicitly* authorizes credential validation AND the key is confirmed to belong to the program — every API call beyond that is scope risk and CloudTrail noise.
5. **Your own account is the lab.** Cross-account enumeration tricks (role/user existence, account-ID-from-bucket, session-policy probing) generate logs in *your* AWS account, not the target's — but they still need authorization against the target.
6. **Defaults are the bug.** Public-access-block gaps, Cognito self-signup, IMDSv1, OIDC trusts missing `sub` conditions, non-retroactive org policies — most cloud findings are insecure defaults, not clever exploits.
7. **Detection knowledge is report quality.** Knowing which GuardDuty finding / CloudTrail event your PoC generates lets you tell the program exactly what their blue team should have seen — that is impact.

## Scope & ethics

Before live testing, verify the exact asset and technique against current program rules. This skill grants no authorization; if scope or permission is missing or unclear, stop and re-check with the program. Restraint rules for cloud work: use your own AWS account for enumeration primitives; report found keys without using them — a single identity check only under explicit policy authorization; prove storage exposure with metadata/canaries, never data reads; never modify policies, create users/roles/keys, or run persistence techniques against a bounty target — document the *capability* (policy evidence) instead; treat post-exploitation chapters as "what an attacker could do" narrative for reports, not a to-do list. AWS's own policies additionally constrain testing against AWS infrastructure — check program rules for cloud-in-scope language. Metadata/version context: `sources.md`.
