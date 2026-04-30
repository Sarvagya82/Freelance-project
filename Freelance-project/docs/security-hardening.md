# Security Hardening

## IAM Least-Privilege Roles

### CI Role (GitHub Actions user/role)

Scoped to only what deployment needs:

- `ecr:GetAuthorizationToken`
- `ecr:BatchCheckLayerAvailability`
- `ecr:InitiateLayerUpload`
- `ecr:UploadLayerPart`
- `ecr:CompleteLayerUpload`
- `ecr:PutImage`
- Optional: constrained `ec2:DescribeInstances` for deployment diagnostics

### EC2 Instance Role

Scoped runtime permissions:

- `secretsmanager:GetSecretValue` for specific secret ARNs only
- `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer`
- `logs:CreateLogStream`, `logs:PutLogEvents` where centralized logs are enabled

No wildcard admin permissions are granted.

## AWS Secrets Manager

Runtime secrets are never committed into repository files.

- App fetches secrets at container start or process bootstrapping.
- Sensitive values include `DATABASE_URL`, API signing keys, and integration tokens.
- Rotation policy:
  - Automatic rotation enabled for database credentials.
  - App and worker secrets rotated on scheduled cadence with staged rollout.

This approach avoids secret sprawl and keeps revocation paths operationally clean.

## RDS in Private Subnet

Database is deployed with `PubliclyAccessible=false`.

Security Group policy:

- **Inbound 5432** allowed **only** from EC2 application security group.
- No CIDR-based public access on database port.
- Outbound restricted to required AWS services where possible.

This blocks direct internet database access and enforces service-to-service trust boundaries.

## CloudWatch Alarms

Monitored controls and rationale:

- **EC2 CPUUtilization** catches sustained load that can degrade API response.
- **EC2 StatusCheckFailed** catches host-level failures quickly.
- **RDS CPUUtilization** tracks DB compute saturation.
- **RDS FreeStorageSpace** prevents storage exhaustion incidents.
- **RDS DatabaseConnections** catches pool leaks and sudden traffic spikes.

Alarm actions route to SNS for on-call notification and operational response.

## Credential Hygiene

- No hardcoded credentials in source files, compose files, workflow scripts, or shell scripts.
- All runtime secrets are resolved from AWS-managed stores.
- GitHub Actions uses encrypted repository secrets for CI-time credentials only.
