# Deployment Guide

## Prerequisites

- AWS account with ECR repositories for `frontend` and `backend`.
- EC2 instance with Docker Engine and Docker Compose plugin installed.
- RDS PostgreSQL instance in private subnet with correct security group rules.
- IAM role attached to EC2 permitting:
  - `secretsmanager:GetSecretValue`
  - `ecr:GetAuthorizationToken`
  - `ecr:BatchGetImage`
  - `ecr:GetDownloadUrlForLayer`
  - CloudWatch logging permissions (optional but recommended)
- GitHub repository with configured Actions secrets.
- SSH access from GitHub runner to EC2 over port `22`.

## Step-by-Step Deployment

1. **Clone the repository**
   ```bash
   git clone https://github.com/<org>/<repo>.git
   cd Freelance-project
   ```
2. **Configure AWS-side dependencies**
   - Create ECR repositories:
     - `<ECR_REGISTRY>/frontend`
     - `<ECR_REGISTRY>/backend`
   - Ensure EC2 can pull from ECR using IAM role.
3. **Store runtime secrets in AWS Secrets Manager**
   - Database URL
   - JWT/APP secrets
   - Any integration credentials
4. **Configure GitHub Actions secrets**
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
   - `AWS_REGION`
   - `ECR_REGISTRY`
   - `EC2_HOST`
   - `EC2_USER`
   - `EC2_SSH_KEY`
5. **Push to main**
   ```bash
   git add .
   git commit -m "Deploy-ready update"
   git push origin main
   ```
6. **Observe pipeline execution**
   - Build and push stage publishes SHA-tagged images.
   - Deploy stage connects to EC2 and updates running services.

## How the EC2 Deploy Script Works

The deploy stage sends a shell script over SSH and executes it on EC2:

1. Receives `DEPLOY_SHA` from the workflow output.
2. Pulls latest ECR images tagged with that SHA.
3. Uses `sed` to update compose image tags in-place.
4. Starts services via `docker compose up -d --pull always`.
5. Verifies service health with `docker ps` and container status checks.
6. On success, writes `DEPLOY_SHA` to `LAST_STABLE_SHA`.

Secrets are **not hardcoded** in compose. App startup scripts fetch required values from AWS Secrets Manager at runtime.

## Rollback Trigger and Behavior

Rollback is triggered when deployment health checks fail (container exits, unhealthy state, or compose restart failure):

1. Script reads existing `LAST_STABLE_SHA`.
2. Rewrites compose image references to the prior stable SHA.
3. Restarts stack with `docker compose up -d --pull always`.
4. Reports rollback completion and exits non-zero for visibility in Actions.

If no `LAST_STABLE_SHA` exists (first deployment), rollback is skipped and pipeline fails hard for manual intervention.
