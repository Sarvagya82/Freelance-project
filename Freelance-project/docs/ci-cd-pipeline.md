# CI/CD Pipeline

## Pipeline Stages

The delivery pipeline is split into two jobs:

1. **build-and-push**
   - Checks out source from `main`.
   - Authenticates to AWS and logs into Amazon ECR.
   - Builds frontend and backend images.
   - Tags images with the immutable commit SHA.
   - Pushes both images to ECR.
2. **deploy**
   - Executes only after successful build.
   - Connects to EC2 over SSH.
   - Pulls images for the exact SHA produced in the previous stage.
   - Restarts services and validates runtime health.
   - Rolls back automatically if deployment checks fail.

## git-SHA Versioning Strategy

Every release is tied to a single git commit hash:

- Backend image: `<ECR_REGISTRY>/backend:<GIT_SHA>`
- Frontend image: `<ECR_REGISTRY>/frontend:<GIT_SHA>`

Benefits:

- Immutable and traceable releases.
- Exact rollback target without guessing tags.
- Fast incident triage by mapping running containers to commit history.

## Rollback Mechanism

`LAST_STABLE_SHA` is persisted on EC2 as the last verified release.

During deploy:

- Candidate SHA is deployed first.
- Health checks confirm containers are running and stable.
- If checks pass, `LAST_STABLE_SHA` is updated.
- If checks fail, compose file is reverted to `LAST_STABLE_SHA` and services are restarted.

This design minimizes downtime and provides deterministic rollback in under a minute.

## ECR Image Lifecycle

ECR stores a history of SHA-tagged releases for rollback and audit.

Recommended lifecycle policy:

- Keep last 50 production images per service.
- Expire untagged images after 7 days.
- Optionally retain release-candidate tags for 14 days.

This keeps storage efficient while preserving enough history for operational recovery and compliance review.
