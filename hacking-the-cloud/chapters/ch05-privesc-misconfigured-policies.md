# Ch5: Privilege Escalation & Misconfigured Policies

Source: `aws/exploitation/` (iam_privilege_escalation, Misconfigured_Resource-Based_Policies/*, route53_modification, local_ec2_priv_esc_through_user_data, obfuscated_admin_policy, cognito_user_self_signup), `gcp/exploitation/*`, `azure/run-command-abuse.md`. Requires an existing foothold (leaked creds, SSRF creds, Cognito pool). For bounty work: prove the *policy condition* exists, don't execute the escalation.

## AWS IAM privesc — the canonical families

**Policy-write self-elevation** (attach or overwrite a better policy on yourself):
`iam:AttachUserPolicy`/`AttachGroupPolicy`/`AttachRolePolicy` (attach AdministratorAccess), `iam:PutUserPolicy`/`PutGroupPolicy`/`PutRolePolicy` (inline admin), `iam:CreatePolicyVersion` + `iam:SetDefaultPolicyVersion` (upgrade existing policy), `iam:AddUserToGroup`, `iam:UpdateLoginProfile`/`CreateLoginProfile` (console password takeover), `iam:CreateAccessKey` (mint keys for a *more privileged user* — also lateral movement), `iam:UpdateAssumeRolePolicy` (open a trusted role to yourself), `iam:Delete*`/`Put*PermissionsBoundary` and `Detach*`/`Delete*Policy` (strip constraints).

**PassRole + compute = run as someone else** — `iam:PassRole` plus a service that instantiates a role: `ec2:RunInstances` (then read the box's IMDS creds), `lambda:CreateFunction`+`InvokeFunction`/`CreateEventSourceMapping`/`AddPermission`, `glue:CreateDevEndpoint`/`CreateJob`/`UpdateJob`, `ecs:RunTask`/`StartTask`+`RegisterContainerInstance`+`DeregisterContainerInstance`, `cloudformation:CreateStack`, `datapipeline:*`, `codestar:CreateProject`+`AssociateTeamMember` (Owner → broad List* perms), autoscaling launch configs/templates, `bedrock-agentcore:CreateCodeInterpreter`+`InvokeCodeInterpreter`.

**Mutate an existing workload**: `lambda:UpdateFunctionCode`/`UpdateFunctionConfiguration`, `glue:UpdateDevEndpoint` (replace SSH key → host creds), `ec2:ModifyInstanceAttribute` on userData (below).

**User-data escalation**: instance must be stopped → `aws ec2 modify-instance-attribute --instance-id i-x --attribute userData --value file://script.b64` → runs as root at next boot (`#cloud-boothook` or multipart `cloud_final_modules:[scripts-user, always]` for every-reboot). Softer variant: poison an S3-hosted script the user data pulls (`aws s3 cp s3://boot/start_script.sh`) — hits every ASG instance too.

**Route53 privesc**: hosted-zone record modification → NS/A/MX control → domain takeover, cert issuance, mail interception.

## Resource-based policy misconfigs (the externals-facing half)

- **Wildcard principal**: `Principal: {"AWS":"*"}` in a trust or resource policy = *every AWS account*, not just yours/org — common fatal misunderstanding; anyone can `AssumeRole`. Obfuscated forms (`"AWS": ["*", "arn:…"]`, conditions that look restrictive but aren't) hide it.
- **Not-elements + Allow**: `NotPrincipal`+Allow = everyone *except* listed entity (public); `NotAction`+Allow = all actions except listed; `NotResource`+Allow = every resource except listed (a typo silently grants the real one).
- **Evaluation quirk**: a resource-policy Allow beats an *implicit* identity deny within the same account — principals can act with no identity grant at all (cross-account still needs identity-side allow). ECR twist: `ecr:GetAuthorizationToken` must come from an identity policy regardless of the repo policy.
- **OIDC/federated trusts**: GitLab `AssumeRoleWithWebIdentity` without `gitlab.com:sub` condition → any gitlab.com user assumes it; Terraform Cloud OIDC missing org/workspace conditions; Amplify service roles predictable by convention. Findable from outside via the session-policy probe (ch01) or `get-role`/Access Analyzer.
- **Obfuscated admin**: `Action:"*:*"` ≡ `"*"`; `?` single-char wildcards (`?am:*`); `NotAction` inversions — all evade literal-string detection. Lint dumped policies with `aws-lint-iam-policies`/Access Analyzer validation.
- **Cognito self-signup → role**: signup enabled (ch01) → ID token → identity-pool creds → whatever IAM role the pool trusts.

## GCP & Azure parallels

- **GCP privesc table** (~24 paths): `iam.serviceAccountKeys.create` (mint SA key), `implicitDelegation`, `getAccessToken`, `signBlob`/`signJwt`, `iam.roles.update`, `setIamPolicy`; compute-creation paths — `compute.instances.create`, `cloudfunctions.*create/update`, `cloudbuild.builds.create`, `run.services.create`, `dataflow/dataproc.jobs`, `deploymentmanager.deployments.create`, `composer.environments.get`, `serviceusage.apiKeys.*`, `storage.hmacKeys.create`, `orgpolicy.policy.set` (strip org constraints). Rhino GCP-IAM-Privilege-Escalation has per-permission scripts.
- **Tag-bindings privesc**: `tagUser` + a conditional grant (`resource.matchTag('env','sandbox')`) → attach the satisfying tag to the target resource → condition passes; splitting tag-bind and privileged call across sessions evades correlation rules.
- **Azure Run Command**: `Microsoft.Compute/virtualMachines/runCommands/action` → script runs as SYSTEM/root on any VM via portal/CLI/PS — execution + lateral movement primitive.

**Bounty-safe validation**: collect the *policy text* (trust policy JSON, bucket policy, IAM list output) as evidence and describe the escalation path — do not create keys, roles, login profiles, or modify userData on a bounty account. iam-vulnerable / CloudGoat reproduce every path in your own lab.
