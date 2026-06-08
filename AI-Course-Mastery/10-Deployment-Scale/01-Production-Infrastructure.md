# 01 — Production Infrastructure Foundation: From Docker to AWS

## 🎯 Purpose & Goals

> 🛑 STOP. Before we talk about Dockerfiles and ECS task definitions, you need to understand the single most important principle of production infrastructure:

**Your dev environment is a LIE.**

Everything works differently in production:
- Your Docker container that starts in 2 seconds locally? On a cold EC2 instance, it takes 3 minutes to pull and start.
- Your database that handles 100 queries/second locally? Under 10,000 QPS in production, connection pooling breaks.
- Your single-server app that "never crashes"? A load balancer health check fails when one request hangs.
- Your "it works on my machine" setup? The production server has different CUDA drivers, different network latency, different memory limits.

**The goal of this module:** Build infrastructure that SURVIVES production — not just works on your laptop.

---

### 🤔 Discovery Question 1: The Silent Resource Leak

You deploy your AI service in a Docker container on a t3.medium EC2 instance (2 vCPU, 4GB RAM).

Day 1: Memory usage is 45%. Everything looks fine.
Day 3: Memory usage is 72%. Still looks fine.
Day 7: Memory usage is 91%. The instance is swapped to disk. Response times go from 500ms to 8 seconds.
Day 8: OOM killer terminates your container. Service down.

**Your monitoring** showed memory usage growing linearly. But you didn't set an alert because "it's not at 90% yet."

**🤔 Before reading on, answer these:**

1. If memory grows linearly from Day 1 (45%) to Day 7 (91%), you can compute the DAILY growth rate. It's about 6.6% per day. At this rate, you know it will hit 100% on Day 9. **Given that you can PREDICT the failure date, what action do you take on Day 3 — not Day 8?** Do you restart the container daily? Find the leak? Add a memory limit?

2. You add a container memory limit. Docker's `--memory=3.5g` means the container gets OOM-killed when it exceeds 3.5GB. But now your container restart EVERY 3-4 days. Users see a brief outage each time. **What's the right tradeoff between "let the container grow until OOM" (longer uptime, unpredictable crash) vs. "hard limit with frequent restarts" (predictable behavior, frequent brief outages)?** Does your answer change if you have a load balancer that can route around a restarting container?

3. **The hardest question:** You find the memory leak — it's in a third-party Python library that caches tokenizer outputs indefinitely. You can't fix the library (it's maintained by another team). **Design a defense strategy that prevents this memory leak from taking down your service.** Options: (a) restart containers on a schedule, (b) limit cache size via environment variable, (c) patch the library at import time. Evaluate each — which minimizes downtime while keeping the system running?

---

### 🤔 Discovery Question 2: The Staging-Is-Production Trap

You have two environments:
- **Staging**: Single Docker container on a t3.large. 32GB disk, 8GB RAM, no load balancer.
- **Production**: Auto-scaling group of 4 containers behind an ALB. 100GB disk, 16GB RAM each.

You deploy to staging. Everything passes. You deploy to production.

**Production immediately breaks:**
- The ALB health check fails — it expects a 200 response at `/health`, but your app returns 200 at `/api/health`
- Requests that worked in staging now TIME OUT — the production containers take 8 seconds to load the model into GPU memory, but the ALB health check timeout is 5 seconds
- The database connection pool has 10 connections — fine for one container, but 4 containers consume all 40 connections and the database rejects new connections

**🤔 Before reading on, answer these:**

1. Your staging and production are IDENTICAL in code but DIFFERENT in configuration (ALB, connection pooling, timeout settings). The code passed staging because the config differences masked the bugs. **Should staging be an EXACT replica of production?** Or is it acceptable to have a smaller, cheaper staging that catches MOST bugs but misses SOME? At what cost does "identical staging" stop being worth it?

2. The database connection pool bug: each container opens 10 connections. With 4 containers, that's 40 connections. The database max is 30. When the 4th container starts, connections 31-40 fail. But the ALB distributes traffic evenly — each container handles ~25% of requests. Container 4's requests ALSO fail because it can't query the database. **What's the fix?** Reduce per-container pool to 7 (4 × 7 = 28 < 30)? Increase database max connections? Add connection pooling at the database level (PgBouncer)?

3. **The hardest question:** You fix all the staging-production discrepancies. But your next deployment breaks PRODUCTION AGAIN — this time due to a RATE LIMIT on the OpenAI API. Staging uses a test API key with higher rate limits. Production uses the real key with lower limits. **How do you design a staging environment that catches rate limit issues WITHOUT consuming production quota?** Can you simulate rate limits in staging?

---

### 🤔 Discovery Question 3: The Deployment That Went Sideways

You have a CI/CD pipeline:
1. Push to `main` → Build Docker image → Push to ECR → Update ECS service

One day you push a change that has a bug — your app crashes on startup (import error from a missing dependency).

**The deployment pipeline:**
- Build: ✅ (Docker builds fine, the import error only happens at runtime)
- Push: ✅
- ECS service update: ✅ (starts new task)
- New task starts → crashes immediately → ECS marks it as UNHEALTHY
- ECS stops the new task → starts ANOTHER new task → same crash → UNHEALTHY
- ECS does this 3 times (the default deployment circuit breaker) → marks deployment as FAILED
- Old task is still running — but the ECS service is now in a "failed deployment" state

**Result:** The old version is still serving traffic. No outage! You think you're safe.

**But:** The ECS service has FAILED_DEPLOYMENT status. Your auto-scaling can't scale because the service is in a bad state. And when the old task eventually crashes (OOM, instance failure, etc.), ECS tries to start a NEW task — using the FAILED task definition. The new task crashes. No tasks running. Outage.

**🤔 Before reading on, answer these:**

1. The failed deployment kept the OLD version running. But the AUTO-SCALER uses the NEW (failed) task definition when scaling up. **Why would ECS use the failed task definition for new tasks instead of the last known good one?** Is this a design flaw in ECS, or is there a reason for this behavior?

2. You want to protect against this: "If a deployment fails, ensure all future tasks use the LAST SUCCESSFUL task definition until a successful deployment completes." **How would you implement this?** Options: manual rollback after failed deployment, CloudFormation stack with rollback trigger, custom Lambda that monitors deployment status. Which approach is most reliable?

3. **The hardest question:** The root cause was a MISSING DEPENDENCY. Your Docker build should have caught this (pip install should fail if a dependency can't be resolved). But the dependency was installed at RUNTIME (a conditional import inside a function that's only called when a specific API endpoint is hit). **How would you design a test that catches runtime import errors BEFORE deployment?** What's the minimum test that gives you confidence the app will actually start?

---

## ⏱ Time Budget

| Section | Time |
|---------|------|
| Discovery Questions | 30 min |
| Concept Deep-Dive | 1 hr |
| Code Examples | 1.5 hr |
| Drills | 45 min |
| Gate Check | 15 min |
| **Total** | **~4 hr** |

---

## 📖 Concept Deep-Dive

### 1. The Production Infrastructure Stack

```
                    ┌─────────────────────────┐
                    │    Load Balancer (ALB)   │
                    │  Routes traffic, health  │
                    │  checks, SSL termination │
                    └───────────┬─────────────┘
                                │
            ┌───────────────────┴───────────────────┐
            │           Auto-Scaling Group          │
            │  ┌────────┐ ┌────────┐ ┌────────┐    │
            │  │Container│ │Container│ │Container│   │
            │  │ vLLM   │ │  API    │ │  Redis  │   │
            │  │ Server │ │ Gateway │ │  Cache  │   │
            │  └────────┘ └────────┘ └────────┘    │
            └───────────────────┬───────────────────┘
                                │
            ┌───────────────────┴───────────────────┐
            │           Data Layer                   │
            │  ┌────────┐ ┌────────┐ ┌────────┐    │
            │  │Postgres│ │  S3    │ │  EFS   │    │
            │  │(State) │ │(Models)│ │(Shared)│    │
            │  └────────┘ └────────┘ └────────┘    │
            └─────────────────────────────────────────┘
```

### 2. Key Infrastructure Decisions

| Decision | Dev | Production |
|----------|-----|------------|
| Container orchestration | Docker Compose | ECS / EKS / Fargate |
| GPU access | --gpus all | ECS GPU task, EKS node pool |
| Model storage | Local disk | S3 / EFS (shared across instances) |
| Secrets | .env file | AWS Secrets Manager / Parameter Store |
| Logging | docker logs | CloudWatch / ELK / Grafana Loki |
| Networking | bridge | VPC with public/private subnets |
| Auto-scaling | N/A | CPU/memory/request-based scaling |
| SSL/TLS | N/A | ACM certificate on ALB |

### 3. Docker Production Checklist

```dockerfile
# ❌ Dev Dockerfile
FROM python:3.11
COPY requirements.txt .
RUN pip install -r requirements.txt  # No cache cleanup
COPY . .
CMD ["uvicorn", "app:app", "--reload"]  # --reload is dev-only

# ✅ Production Dockerfile
FROM python:3.11-slim  # Smaller image
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*  # Clean apt cache

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt  # No pip cache

# Create non-root user
RUN useradd -m -u 1000 appuser
USER appuser

COPY --chown=appuser:appuser . .

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 4. ECS vs. EKS vs. Fargate Decision Matrix

| Factor | ECS | EKS | Fargate |
|--------|-----|-----|---------|
| Setup complexity | Low | High | Low |
| GPU support | ✅ Yes (EC2) | ✅ Yes (node pools) | ❌ No |
| Cost model | Pay for EC2 | Pay for EC2 + control plane | Pay per task |
| Scaling granularity | Service-level | Node-level | Task-level |
| Operational overhead | Medium | High | Low |
| When to choose | Most teams | Need K8s ecosystem | No GPU, want serverless |

---

## 💻 CODE EXAMPLES

### Example 1: The Broken Docker Compose Setup (Full of Bugs)

```yaml
# WHAT NOT TO DO — A docker-compose.yaml with 12+ hidden bugs

version: '3.8'

services:
  api:
    build: .
    ports:
      - "8000:8000"
    # BUG 1: No restart policy
    # Container exits on crash → stays down
    # BUG 2: No health check
    # Load balancer can't detect if service is healthy
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      # BUG 3: Hardcoded database URL
      # Points to a local database that doesn't exist in production
      - DATABASE_URL=postgres://localhost:5432/mydb
      # BUG 4: No resource limits
      # Container can consume ALL host memory
      # BUG 5: Using .env without validation
      # If OPENAI_API_KEY is missing, app starts but fails on first request
    volumes:
      # BUG 6: Mounting entire codebase
      # In production, code should be IN the image, not mounted
      - .:/app
    # BUG 7: No depends_on with healthcheck
    # API starts before database is ready → connection errors

  redis:
    image: redis:7-alpine
    # BUG 8: No password
    # Anyone who reaches this port can access Redis
    # BUG 9: No persistence
    # All cache lost on restart
    # BUG 10: No memory limit
    # Redis can consume all host memory and get OOM-killed

  # BUG 11: No model-server service
  # The API is supposed to use a fine-tuned model via vLLM,
  # but there's no GPU service defined
  # BUG 12: No monitoring
  # No Prometheus, no Grafana, no way to observe what's happening
```

#### 🔍 Find The Bugs

| # | Bug | Impact | Fix |
|---|-----|--------|-----|
| 1 | No restart policy | Container stays down after crash | restart: unless-stopped |
| 2 | No health check | Load balancer routes to dead containers | Add HEALTHCHECK in Dockerfile or docker-compose healthcheck |
| 3 | Hardcoded database URL | Different envs use different DBs | Use environment variable with sensible default |
| 4 | No resource limits | Container OOMs the host | mem_limit, mem_reservation, cpus |
| 5 | .env without validation | Missing key causes runtime error | Validate env vars at startup |
| 6 | Mounting codebase | Hot-reload in prod = unpredictable | Code in image, volumes only for data |
| 7 | No depends_on healthcheck | Race condition on startup | depends_on with condition: service_healthy |
| 8 | No Redis password | Anyone can access cache | Add requirepass |
| 9 | No Redis persistence | Cache = empty after restart | Add save config or AOF |
| 10 | No Redis memory limit | Redis OOMs the host | maxmemory setting |
| 11 | No model server | API depends on missing service | Add vLLM service with GPU config |
| 12 | No monitoring | Can't detect problems | Add Prometheus + Grafana, health endpoints |

---

### Example 2: Production Docker Compose (Fixed)

```yaml
# Production-grade Docker Compose deployment
# 
# Architecture:
#   - ALB → API Gateway (FastAPI) → vLLM (GPU) + Redis (cache)
#   - Prometheus + Grafana for monitoring
#   - All services have health checks, resource limits, restart policies

version: '3.8'

x-logging: &default-logging
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"

x-healthcheck: &default-healthcheck
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 60s

services:
  # ─── API Gateway ───
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    restart: unless-stopped
    healthcheck:
      <<: *default-healthcheck
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
    environment:
      # Env vars — validated at startup
      - OPENAI_API_KEY=${OPENAI_API_KEY:?Must set OPENAI_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY:-}  # Optional
      - DATABASE_URL=${DATABASE_URL:-postgres://user:pass@db:5432/app}
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
      - MODEL_SERVER_URL=http://model-server:8001
      - LOG_LEVEL=${LOG_LEVEL:-info}
      - ENVIRONMENT=${ENVIRONMENT:-production}
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G
    depends_on:
      redis:
        condition: service_healthy
      model-server:
        condition: service_started  # Graceful degradation if model not ready
    logging: *default-logging
    networks:
      - app-net

  # ─── Model Server (vLLM) ───
  model-server:
    image: vllm/vllm-openai:latest
    restart: unless-stopped
    healthcheck:
      <<: *default-healthcheck
      test: ["CMD", "curl", "-f", "http://localhost:8001/health"]
    environment:
      - MODEL_NAME=${MODEL_NAME:-mistralai/Mistral-7B-Instruct-v0.2}
      - QUANTIZATION=${QUANTIZATION:-awq}
      - MAX_MODEL_LEN=${MAX_MODEL_LEN:-8192}
      - GPU_MEMORY_UTILIZATION=${GPU_MEMORY_UTILIZATION:-0.90}
      - NUM_GPU=${NUM_GPU:-1}
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: ${NUM_GPU:-1}
              capabilities: [gpu]
    volumes:
      - model-cache:/root/.cache/huggingface  # Cache model downloads
    ports:
      - "8001:8001"
    command:
      - "--model"
      - "${MODEL_NAME}"
      - "--port"
      - "8001"
      - "--max-model-len"
      - "${MAX_MODEL_LEN}"
      - "--gpu-memory-utilization"
      - "${GPU_MEMORY_UTILIZATION}"
      - "--tensor-parallel-size"
      - "${NUM_GPU}"
    logging: *default-logging
    networks:
      - app-net

  # ─── Redis Cache ───
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    command:
      - "redis-server"
      - "--requirepass"
      - "${REDIS_PASSWORD:?Must set REDIS_PASSWORD}"
      - "--maxmemory"
      - "2gb"
      - "--maxmemory-policy"
      - "allkeys-lru"
      - "--appendonly"
      - "yes"
      - "--appendfsync"
      - "everysec"
    volumes:
      - redis-data:/data
    deploy:
      resources:
        limits:
          memory: 2.5G
    logging: *default-logging
    networks:
      - app-net

  # ─── Prometheus ───
  prometheus:
    image: prom/prometheus:latest
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=30d"
      - "--storage.tsdb.retention.size=50GB"
    ports:
      - "9090:9090"
    logging: *default-logging
    networks:
      - app-net

  # ─── Grafana ───
  grafana:
    image: grafana/grafana:latest
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD:-admin}
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    logging: *default-logging
    networks:
      - app-net

volumes:
  model-cache:
  redis-data:
  prometheus-data:
  grafana-data:

networks:
  app-net:
    driver: bridge
```

---

### Example 3: Startup Validation Script

```python
"""
Startup validation — run BEFORE the app starts accepting traffic.

This script:
1. Validates required environment variables
2. Checks database connectivity
3. Pings the model server
4. Verifies API keys with a test call
5. FAILS FAST — exits with non-zero code if anything is wrong

Fail fast is critical: it's better to crash at startup (caught by deployment pipeline)
than to fail on the first user request.
"""

import os
import sys
import json
from typing import List, Dict, Optional
import httpx


class StartupValidator:
    """Validate all dependencies before the app starts."""
    
    REQUIRED_ENV_VARS = [
        "OPENAI_API_KEY",
        "DATABASE_URL",
        "REDIS_URL",
    ]
    
    OPTIONAL_ENV_VARS = [
        "ANTHROPIC_API_KEY",
        "MODEL_SERVER_URL",
        "LOG_LEVEL",
    ]
    
    def __init__(self):
        self.errors: List[str] = []
        self.warnings: List[str] = []
    
    def validate_env_vars(self):
        """Check required environment variables are set."""
        for var in self.REQUIRED_ENV_VARS:
            if not os.getenv(var):
                self.errors.append(f"Missing required env var: {var}")
        
        for var in self.OPTIONAL_ENV_VARS:
            if not os.getenv(var):
                self.warnings.append(f"Optional env var not set: {var}")
    
    async def check_database(self, url: str) -> bool:
        """Check database connectivity."""
        try:
            # Extract host and port from URL
            # In production, use your DB driver's ping method
            async with httpx.AsyncClient() as client:
                response = await client.get(
                    f"{url}/ping", timeout=5.0
                )
                return response.status_code == 200
        except Exception as e:
            self.errors.append(f"Database connection failed: {e}")
            return False
    
    async def check_model_server(self, url: str) -> bool:
        """Check model server health."""
        try:
            async with httpx.AsyncClient() as client:
                response = await client.get(
                    f"{url}/health", timeout=10.0
                )
                if response.status_code == 200:
                    return True
                else:
                    self.warnings.append(
                        f"Model server health check returned {response.status_code}"
                    )
                    return False
        except Exception as e:
            self.warnings.append(f"Model server not reachable: {e}")
            return False
    
    async def check_api_key(self, provider: str, key: str) -> bool:
        """Verify an API key with a minimal test call."""
        try:
            if provider == "openai":
                async with httpx.AsyncClient() as client:
                    response = await client.get(
                        "https://api.openai.com/v1/models",
                        headers={"Authorization": f"Bearer {key}"},
                        timeout=10.0,
                    )
                    if response.status_code == 200:
                        return True
                    else:
                        self.errors.append(
                            f"OpenAI API key validation failed: {response.status_code}"
                        )
                        return False
            return True  # Skip validation for unknown providers
        except Exception as e:
            self.warnings.append(f"Could not validate {provider} API key: {e}")
            return False
    
    async def validate_all(self) -> bool:
        """Run all validations. Returns True if OK, False if fatal errors."""
        
        # 1. Environment variables
        self.validate_env_vars()
        
        # 2. Database
        db_url = os.getenv("DATABASE_URL", "")
        if db_url:
            await self.check_database(db_url)
        
        # 3. Model server
        model_url = os.getenv("MODEL_SERVER_URL", "")
        if model_url:
            await self.check_model_server(model_url)
        
        # 4. API keys
        openai_key = os.getenv("OPENAI_API_KEY", "")
        if openai_key:
            await self.check_api_key("openai", openai_key)
        
        # Report
        print(f"\n{'='*50}")
        print(f"STARTUP VALIDATION REPORT")
        print(f"{'='*50}")
        
        if self.errors:
            print(f"\n❌ FATAL ERRORS ({len(self.errors)}):")
            for err in self.errors:
                print(f"   • {err}")
        
        if self.warnings:
            print(f"\n⚠️  WARNINGS ({len(self.warnings)}):")
            for warn in self.warnings:
                print(f"   • {warn}")
        
        if not self.errors and not self.warnings:
            print("\n✅ All checks passed!")
        
        print(f"\n{'='*50}")
        
        return len(self.errors) == 0


# Run on startup
if __name__ == "__main__":
    import asyncio
    
    validator = StartupValidator()
    success = asyncio.run(validator.validate_all())
    
    sys.exit(0 if success else 1)
```

---

### 🔍 Critical Questions

1. **Fail fast vs. graceful degradation**: The validation script fails if the database is unreachable. But what if the database is only needed for some endpoints (e.g., user auth) while other endpoints (e.g., model inference) work fine without it? **Should the app START with degraded functionality instead of refusing to start at all?** How do you communicate degraded status to the load balancer?

2. **Resource limits**: The docker-compose sets CPU/memory limits. But what happens when a request exceeds the CPU limit? The container gets throttled. A throttled container handles requests slower → requests queue up → latency spikes. **Is it better to have NO CPU limit (let the container burst) but risk starving other containers?** When would you set CPU limits vs. not?

3. **GPU resource reservations**: The vLLM service requests 1 GPU via `deploy.resources.reservations.devices`. But if Docker can't allocate a GPU, the service starts WITHOUT GPU acceleration — it silently falls back to CPU, which is 50x slower. **How would you detect and alert on "GPU-accelerated service running on CPU"?**

---

## ✅ Good Output Examples

### What a Production-Ready Deployment Looks Like

```
Health Check: ✅ All services healthy

Service Status:
┌────────────────┬─────────┬────────┬─────────┬──────────┐
│ Service        │ Status  │ CPU    │ Memory  │ Uptime   │
├────────────────┼─────────┼────────┼─────────┼──────────┤
│ api            │ healthy │ 23%    │ 1.2/4GB │ 14d 3h   │
│ model-server   │ healthy │ 67%    │ 14/22GB │ 14d 3h   │
│ redis          │ healthy │ 5%     │ 0.8/2GB │ 14d 3h   │
│ prometheus     │ healthy │ 12%    │ 1.5/4GB │ 14d 3h   │
│ grafana        │ healthy │ 3%     │ 0.3/1GB │ 14d 3h   │
└────────────────┴─────────┴────────┴─────────┴──────────┘

Startup Validation: ✅ All checks passed
  ✓ Environment variables: 5/5 present
  ✓ Database: ping 12ms
  ✓ Model server: healthy (vLLM 0.4.2, 8B model, AWQ 4-bit)
  ✓ API keys: OpenAI ✅, Anthropic ✅

Deployment Info:
  Version: v2.3.1 (git: a3f8c2e)
  Deployed: 2024-04-15 14:30 UTC
  Built: 2024-04-15 14:25 UTC
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: The Fat Container

**Problem**: One container runs everything — FastAPI, Celery workers, background tasks, scheduled jobs. A memory leak in a background task OOMs the entire service.

**Fix**: SEPARATE CONCERNS. API server, background workers, and scheduled jobs should be different services with independent resource limits.

### Antipattern 2: Configuration Drift

**Problem**: Staging and production configurations diverge over time. A change is tested on staging (with old config) and deployed to production (with different config). It breaks.

**Fix**: Infrastructure as Code. All configuration is in VERSION CONTROL. Staging and production use the same templates with different variable values.

### Antipattern 3: No Graceful Shutdown

**Problem**: When a container is stopped (deployment, scaling, failure), in-flight requests are DROPPED. Users see connection errors.

**Fix**: Implement SIGTERM handling. On shutdown: (1) mark health check as failed (stop new traffic), (2) wait for in-flight requests to complete (with timeout), (3) then exit.

### Antipattern 4: Blind Deployment

**Problem**: Deploy and hope. No health checks, no metrics, no rollback. If the deployment breaks, you find out when users complain.

**Fix**: Automated canary deployments + health checks + metrics-based rollback. Deploy to 10% of traffic, monitor for 5 minutes, then roll out to 100% if healthy.

### Common Infra Failures

| Failure | Symptom | Fix |
|---------|---------|-----|
| Out of disk | Container crashes, can't write logs | Monitor disk usage, set log rotation |
| Connection pool exhaustion | Database errors under load | Set max connections per service, add PgBouncer |
| DNS resolution failure | Services can't find each other | Use Docker internal DNS, add retry logic |
| Docker socket issues | Can't deploy new containers | Separate Docker socket per service |
| Network latency between services | Slow response times | Co-locate dependent services, use internal networking |
| Clock skew | Auth tokens fail, metrics misaligned | Use NTP on all hosts, monitor clock drift |

---

## 🧪 Drills & Challenges

### Drill 1: Build a Deployment Rollback Script (30 min)

**Goal**: Given a failed deployment, automatically roll back to the last known good version.

**Task**: Write a script that:
1. Checks if the current deployment is healthy (all services passing health checks)
2. If unhealthy: get the last SUCCESSFUL deployment from ECS/CloudFormation
3. Redeploy that version
4. Verify health after rollback
5. Report success/failure

```python
def rollback_to_last_good(service_name: str, cluster: str) -> bool:
    """
    Find the last successful task definition and redeploy.
    
    Steps:
    1. List ECS task definition revisions for this service
    2. Find the last revision that had HEALTHY status
    3. Update the service with that task definition
    4. Wait for deployment to stabilize
    5. Verify health
    """
    pass
```

**Questions to answer:**
1. How do you define "last successful"? By time? By deployment status? By health check results?
2. What if the rollback ALSO fails? Do you keep trying older versions?
3. How do you NOTIFY the team about the rollback?

---

### Drill 2: Containerize a Multi-Service App (45 min)

**Goal**: Write a complete docker-compose.yaml for a real AI application.

**Requirements:**
- FastAPI app with 2 endpoints (chat, health)
- vLLM model server (GPU-accelerated)
- Redis cache
- Prometheus metrics
- Gather + Grafana for monitoring
- All services have: health checks, resource limits, restart policy, logging
- Network isolation (internal services not exposed to internet)

**Your docker-compose will be evaluated on:**
- Does every service have a health check?
- Are resource limits set appropriately?
- Is there a restart policy?
- Is logging configured?
- Are secrets passed securely (not hardcoded)?
- Can the app start without race conditions?

---

### Drill 3: Design a CI/CD Pipeline (45 min)

**Goal**: Design a complete CI/CD pipeline for an AI service.

**Requirements:**
- Build → Test → Deploy to staging → Integration tests → Deploy to production
- Production deployment: canary (10% for 5 min) → full rollout
- Rollback: automatic if error rate > 1% or latency > threshold
- All steps must complete within 30 minutes total

**Pipeline stages:**
1. Build Docker image (5 min)
2. Run unit tests (2 min)
3. Run integration tests against staging (5 min)
4. Deploy to staging (2 min)
5. Run smoke tests (3 min)
6. Deploy canary to production (2 min)
7. Monitor canary (5 min)
8. Full rollout (2 min)
9. Post-deployment monitoring (ongoing)

**Questions to answer:**
1. Can you parallelize any stages? Which ones depend on which?
2. What triggers the pipeline? Push to main? Tag? Manual approval?
3. How do you handle infrastructure changes (new services, config changes) differently from code changes?

---

## 🚦 Gate Check

Before moving to File 02 (Model Serving at Scale), verify you can:

1. **Design a deployment architecture** — For an AI service with: FastAPI, vLLM (GPU), Redis, Postgres, describe the docker-compose setup, resource requirements, and scaling strategy

2. **Write a health check** — Design a health endpoint that checks: database connectivity, model server status, API key validity, disk space. What should it return? What status code for degraded vs. healthy?

3. **Plan a rollback** — Given a deployment that causes 500 errors: describe the rollback procedure, how you identify the last good version, how you verify the rollback succeeded, and how you notify the team

4. **Design staging** — For a service with GPU requirements, OpenAI API calls, and a Postgres database: how do you create a staging environment that catches production issues without spending as much? What do you SHARE with production vs. what do you keep separate?

5. **Secure a docker-compose** — For a 5-service deployment, identify: which services should be exposed to the internet, which should be internal-only, which need authentication between them, and how to manage secrets

---

## 📚 Resources

### Docker & Containerization
- **[Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)** — Official Docker production guidance
- **[Docker Compose Production](https://docs.docker.com/compose/production/)** — Moving from dev to production

### AWS Infrastructure
- **[ECS Best Practices](https://docs.aws.amazon.com/AmazonECS/latest/bestpractices/)** — AWS ECS production patterns
- **[VPC Design](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-example-private-subnets-nat.html)** — Public/private subnet architecture

### Phase Cross-References
- **Phase 1 (Gateway)** → Your Phase 1 gateway runs ON this infrastructure. Containerize it.
- **Phase 8 (Guardrails)** → Guardrails are a service in this stack — add them as another container.
- **Phase 9 (Fine-Tuning)** → Your fine-tuned model replaces the model-server container's model.
- **All previous phases** → This is the INFRASTRUCTURE that runs everything you've built.

> **Next up: File 02 — Model Serving at Scale.** Docker Compose handles one GPU. What happens when you need 8 GPUs, 100 concurrent requests, and automatic scaling? vLLM has features most engineers never use — tensor parallelism, prefix caching, automatic scaling. You'll discover why your single-GPU deploy breaks at scale.
