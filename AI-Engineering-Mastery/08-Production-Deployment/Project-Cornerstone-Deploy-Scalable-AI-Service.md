# Cornerstone Project: Deploy Scalable AI Service

**Time**: 3-4 days  **Difficulty**: Medium-Hard  **Portfolio**: Yes

## The Problem

Take your Phase 4 RAG system or Phase 6 Multi-Agent system and deploy it to production on AWS with proper infrastructure.

## Requirements

### Must Have
- [ ] Docker Compose for local development (AI service + all dependencies)
- [ ] Multi-stage Docker build for production
- [ ] Deployed to AWS ECS Fargate (or EKS)
- [ ] Application Load Balancer with HTTPS
- [ ] Auto-scaling (CPU/memory based)
- [ ] Secrets management (AWS Secrets Manager for API keys)
- [ ] CloudWatch logging and metrics
- [ ] CI/CD pipeline (GitHub Actions build → test → eval → deploy)

### Should Have
- [ ] Blue/green deployment strategy
- [ ] Canary testing (route 5% traffic to new version)
- [ ] CloudFront CDN for static assets
- [ ] RDS + pgvector for production database
- [ ] ElastiCache (Redis) for caching
- [ ] Bedrock integration (instead of direct API calls)

### Nice to Have
- [ ] Multi-region deployment
- [ ] Cost allocation tags per service
- [ ] Guardrails (rate limiting, input validation, PII detection)
- [ ] Dashboard (Grafana or CloudWatch dashboard)
- [ ] Alerting (PagerDuty or Slack for production issues)

## Architecture

```
User → CloudFront → ALB (HTTPS)
                      ↓
               ECS Fargate (auto-scaling)
                ├── API Service (2-10 tasks)
                ├── Worker Service (async tasks)
                │
Service Mesh:    ├── ElastiCache (Redis)
                 ├── RDS (PostgreSQL + pgvector)
                 └── Bedrock (LLM API)
```

## Infrastructure as Code

Use AWS CDK (Python) or Terraform to define:
1. VPC with public/private subnets
2. ECS cluster + task definitions + services
3. ALB with HTTPS certificate
4. RDS + ElastiCache
5. IAM roles and policies
6. CloudWatch dashboards and alarms
7. ECR repositories

## 🔴 Senior: Production Readiness Checklist

```
Before going live:
  [ ] Health endpoints on all services
  [ ] Startup probes (not just liveness)
  [ ] Graceful shutdown handling
  [ ] No hardcoded secrets (env vars from Secrets Manager)
  [ ] Logging to centralized location
  [ ] Metrics exported (latency, errors, throughput)
  [ ] Alerts configured for p95 latency > 5s
  [ ] Cost budget alerts configured
  [ ] Security scan passed (container vulnerabilities)
  [ ] Load test passed (100 concurrent users)
  [ ] Rollback plan documented
  [ ] Runbook for common failures
```

## Evaluation

| Metric | Target |
|--------|--------|
| Uptime | 99.9% |
| p95 latency | < 3s |
| Error rate | < 0.1% |
| Auto-scaling | 0→10 tasks in < 2 min |
| Deployment time | < 10 min |
| Cost | < $500/month |

## Design Decisions

1. **Fargate vs EKS vs EC2**: Serverless vs Kubernetes vs control?
2. **Single vs multi-service**: One big service or many small ones?
3. **Database choice**: Aurora Serverless vs RDS vs DynamoDB?
4. **Cache strategy**: Redis vs ElastiCache vs DAX?
5. **LLM access**: Direct API vs Bedrock vs self-hosted?
6. **Observability**: CloudWatch vs Datadog vs Grafana?

## Reflection

1. What was the hardest part of the production deployment?
2. How did you handle secrets management?
3. What would break at 10x the traffic?
4. How would you recover from a failed deployment?
5. What's the cost breakdown of your deployed system?
