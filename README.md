# Ghostfolio CI/CD — Docker · ECR · EC2

This repository demonstrates a **production-style CI/CD pipeline** for deploying a containerised application to AWS.
The application used as a deployment target is [Ghostfolio](https://github.com/ghostfolio/ghostfolio) — an open-source wealth management tool.
The `Dockerfile` is part of the original Ghostfolio project. Everything else — `docker-compose.yml`, environment configuration, and the full CI/CD pipeline — was written from scratch as part of this infrastructure work.

## Stack

| Layer | Technology |
|---|---|
| Application | Node.js 20 / NestJS (Ghostfolio) |
| Containerisation | Docker — multi-stage build |
| Container Registry | AWS ECR |
| Runtime | AWS EC2 (Amazon Linux 2) |
| Orchestration | Docker Compose |
| Logging | AWS CloudWatch Logs (awslogs driver) |
| CI/CD | GitHub Actions |

## Pipeline Overview

```
[Run workflow] (manual trigger)
        │
        ▼
┌─────────────────────┐
│  build-and-push     │
│  • docker build     │
│  • push :SHA tag    │
│  • push :latest tag │
└────────┬────────────┘
         │ needs:
         ▼
┌─────────────────────┐
│  deploy             │
│  • scp compose file │
│  • ssh → ECR login  │
│  • docker compose   │
│    pull && up -d    │
│  • image prune      │
└─────────────────────┘
```

## Workflows

### `deploy.yml` — Build and Deploy
Triggered **manually** via GitHub Actions → Run workflow.

1. **Build** — multi-stage Docker build using the existing [`Dockerfile`](./Dockerfile) from Ghostfolio
2. **Tag** — image tagged with short commit SHA + `latest`
3. **Push** — pushed to private AWS ECR repository
4. **SCP** — [`docker-compose.yml`](./docker-compose.yml) (written from scratch) copied to EC2
5. **SSH** — ECR login on EC2, `docker compose pull && up -d`, image cleanup

### `draft-release.yml` — Release Drafter
Triggered on any `v*.*.*` tag push. Creates a draft GitHub Release with auto-generated release notes.

## Required GitHub Secrets

| Secret | Description |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key |
| `EC2_HOST` | Public IP or DNS of EC2 instance |
| `EC2_SSH_KEY` | Private SSH key for `ec2-user` |
| `ECR_REGISTRY` | ECR registry URL (e.g. `123456789.dkr.ecr.us-east-1.amazonaws.com`) |

## Local Development

```bash
# Copy env template
cp .env.example .env

# Start with Docker Compose
docker compose up -d
```

App available at `http://localhost:3333`

## Environment Variables

See [`.env.example`](./.env.example) for all required variables including:
- `DATABASE_URL` — PostgreSQL connection string
- `REDIS_URL` — Redis connection string
- `JWT_SECRET_KEY`, `ACCESS_TOKEN_SALT` — app secrets
- `AWS_REGION`, `AWS_LOG_GROUP_APP` — CloudWatch logging config
- `ECR_REGISTRY` — your ECR registry URL
