# Healthcare ERP - AWS Cloud Migration & Deployment

![CI/CD](https://img.shields.io/github/actions/workflow/status/your-org/healthcare-erp/deploy.yml?branch=main&label=CI%2FCD&style=for-the-badge)
![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20RDS%20%7C%20ECR-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Private%20RDS-336791?style=for-the-badge&logo=postgresql&logoColor=white)

Migrated a healthcare ERP to AWS EC2 with Docker Compose, automated CI/CD via GitHub Actions, private RDS PostgreSQL, and hardened production security.

## AWS Architecture

Architecture diagram source: [`architecture/aws-architecture.md`](architecture/aws-architecture.md)

```text
See: architecture/aws-architecture.md
```

## Key Outcomes

| Metric | Result |
| --- | --- |
| Deployment effort reduced | 70% |
| Release cycle | Under 5 minutes |
| DB internet exposure | 0% (private subnet) |
| Monitoring blind spots | None (CloudWatch alarms) |

## Tech Stack

- AWS EC2
- RDS PostgreSQL
- Amazon ECR
- Docker Compose
- Nginx
- GitHub Actions
- AWS Secrets Manager
- Amazon CloudWatch
- AWS IAM

## How It Works

Production runs on a single EC2 host in a public subnet. Nginx terminates inbound traffic and routes requests to two internal containers:

- **Frontend container** serves the web application on port `3000`.
- **Backend container** exposes the API on port `8000`.

The backend connects to a PostgreSQL RDS instance deployed in a private subnet. Direct internet access to the database is blocked by design. Only the EC2 security group is allowed to connect to RDS on `5432`.

## CI/CD Pipeline

Pushes to `main` trigger GitHub Actions. The workflow builds frontend and backend images, tags both with the immutable git SHA, and pushes them to ECR. Deployment then runs over SSH on EC2 and updates `docker-compose.yml` image tags to the new SHA.

Deployment behavior includes:

- git-SHA image versioning for deterministic rollback.
- ECR pull and rolling container update through `docker compose up -d --pull always`.
- Automatic rollback if health checks fail, using `LAST_STABLE_SHA`.
- Promotion of successful releases by updating `LAST_STABLE_SHA` on-host.

## Security Design

- IAM roles scoped with least privilege for EC2 runtime and CI deployment actions.
- Secrets Manager used for runtime secret retrieval and scheduled rotation.
- RDS stays in private subnets with no public endpoint exposure.
- Security groups strictly separate internet-facing and internal traffic.
- CloudWatch metric alarms monitor EC2 and RDS health, performance, and capacity.

## Lessons Learned

- Release reliability improved only after enforcing immutable image tags and rollback by default.
- Keeping secrets out of compose files required robust runtime retrieval and startup validation.
- Alert quality mattered more than alert quantity; threshold tuning reduced noise and improved response.
- Database access boundaries were easiest to maintain using security-group-to-security-group rules.

---

Infrastructure decommissioned post-project engagement. All configs and pipeline code reflect the production setup.
