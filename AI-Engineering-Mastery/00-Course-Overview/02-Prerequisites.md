# Prerequisites

## Non-Negotiable

| Skill | Level | Why |
|-------|-------|-----|
| Python | Advanced typing, async, decorators, context managers | 80% of AI tooling |
| FastAPI | Routing, middleware, DI, WebSockets, background tasks | Backbone of AI services |
| Docker | Compose, multi-stage builds, networking, volumes | Everything runs in containers |
| AWS | EC2, ECS/S3, IAM, VPC basics | Production deployment target |
| PostgreSQL | Queries, indexing, JSONB, pgvector | Data + vector storage |
| Git | Branching, rebasing, PR workflows | Non-negotiable |
| CLI | bash, env management, process supervision | Daily driver |

## Strongly Recommended

| Skill | Why |
|-------|-----|
| Redis | Caching, rate limiting, pub/sub for streaming |
| CI/CD (GitHub Actions) | Automated eval + deployment pipelines |
| Kubernetes (basic) | Production scaling (covered in Phase 8) |
| TypeScript/React | Frontend for AI apps (optional but valuable) |

## Tooling Setup

- Python 3.11+ (install via pyenv)
- Docker Desktop 4.25+
- AWS CLI configured with named profile
- OpenAI API key + Anthropic API key
- VS Code with Python, Docker, YAML extensions
- Langfuse account (free tier)
