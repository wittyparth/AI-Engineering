# ⛩️ GATE 10 — Architect's Spire (Production Deployment)

**Rank Requirement:** A
**Theme:** Deploying LLM applications to production — architecture, scaling, cost optimization, and reliability
**Total Days:** 7 Dungeons + 1 Boss Battle
**Core Skills:** Production architecture, containerization, scaling, cost management, CI/CD, incident response

---

## Dungeon 1 — Production Architecture

**Day 1 | Concept:** A production LLM system is not just an API call. It's a distributed system with caching, queues, fallbacks, and monitoring.

**Reference Architecture:**
```
                   ┌──────────┐
                   │   CDN    │
                   └────┬─────┘
                        │
                   ┌────▼─────┐
                   │  Load    │
                   │ Balancer │
                   └────┬─────┘
                        │
              ┌─────────┼─────────┐
              │         │         │
         ┌────▼──┐ ┌───▼───┐ ┌───▼───┐
         │  App  │ │  App  │ │  App  │
         │ Svr 1 │ │ Svr 2 │ │ Svr 3 │
         └───┬───┘ └───┬───┘ └───┬───┘
             │         │         │
        ┌────┴─────────┴─────────┴────┐
        │         Redis Cache         │
        └─────────────┬───────────────┘
                      │
        ┌─────────────┴───────────────┐
        │         LLM Gateway         │
        │  (rate-limit, retry, route)  │
        └────┬──────────┬─────────────┘
             │          │
        ┌────▼──┐  ┌────▼────┐
        │ LLM   │  │ Vector  │
        │ API   │  │   DB    │
        └───────┘  └─────────┘
```

**🗡️ Dungeon Mastery:** Design the architecture for a customer support chatbot. Identify: single point of failure, scaling bottleneck, cost driver, and fallback mechanism.

**💻 Code — Architecture Blueprint:**
```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class Service:
    name: str
    replicas: int
    cpu: str
    memory: str
    dependencies: list[str] = field(default_factory=list)

@dataclass
class Architecture:
    services: list[Service] = field(default_factory=list)
    cache: Optional[str] = None
    database: Optional[str] = None
    message_queue: Optional[str] = None

    def add_service(self, service: Service):
        self.services.append(service)

    def total_resources(self) -> dict:
        total_cpu = sum(int(s.cpu.replace("c", "")) for s in self.services)
        total_mem = sum(int(s.memory.replace("Gi", "")) for s in self.services)
        return {"cpu_cores": total_cpu, "memory_gb": total_mem}

    def identify_single_points_of_failure(self) -> list[str]:
        spofs = []
        if not any(s.replicas > 1 for s in self.services):
            spofs.append("All services have single replica")
        if not self.cache:
            spofs.append("No cache layer — DB will be bottleneck")
        return spofs


# Example: Customer support bot architecture
chatbot_arch = Architecture(cache="Redis", database="PostgreSQL", message_queue="RabbitMQ")
chatbot_arch.add_service(Service("web-app", replicas=3, cpu="1c", memory="2Gi", dependencies=["api-gateway"]))
chatbot_arch.add_service(Service("api-gateway", replicas=3, cpu="1c", memory="2Gi", dependencies=["llm-service", "rag-service"]))
chatbot_arch.add_service(Service("llm-service", replicas=2, cpu="4c", memory="16Gi", dependencies=[]))
chatbot_arch.add_service(Service("rag-service", replicas=2, cpu="2c", memory="8Gi", dependencies=["vector-db"]))
```

**👹 Boss Lesson:** Design for failure. Assume every component will fail and build redundancy, fallbacks, and graceful degradation.

---

## Dungeon 2 — Containerization & Orchestration

**Day 2 | Concept:** Docker for packaging, Kubernetes for orchestration. Every LLM service should be containerized.

**Docker Best Practices for LLM Services:**
- Multi-stage builds (dev image ≠ prod image)
- Pre-download models in build step (not at runtime)
- Health checks that actually check model ready
- Graceful shutdown (finish in-flight requests)
- Resource limits (OOM kills are silent failures)

**🗡️ Dungeon Mastery:** Write a Dockerfile for a FastAPI LLM service. Multi-stage build: dev stage with hot reload, prod stage with gunicorn. Include health check.

**💻 Code — Dockerfile for LLM Service:**
```dockerfile
# === Build stage ===
FROM python:3.11-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# === Production stage ===
FROM python:3.11-slim AS production

WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*

COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY ./app ./app
COPY ./models ./models

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000

# Graceful shutdown — finish in-flight requests
STOPSIGNAL SIGTERM

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]


# Kubernetes deployment spec (conceptual):
"""
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-service
spec:
  replicas: 3
  strategy:
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    spec:
      containers:
      - name: llm
        image: llm-service:latest
        resources:
          requests:
            memory: "12Gi"
            cpu: "2"
          limits:
            memory: "16Gi"
            cpu: "4"
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          periodSeconds: 60
"""
```

**👹 Boss Lesson:** Containerization is table stakes. If your LLM service isn't containerized, it's not ready for production.

---

## Dungeon 3 — Caching Strategies

**Day 3 | Concept:** LLM inference is expensive and slow. Cache aggressively. Every cache hit saves money and latency.

**Caching Levels:**
| Level | Cache Key | TTL | Hit Rate |
|-------|-----------|-----|----------|
| Exact query | Exact user input | 1 hour | Low (~10%) |
| Semantic | Embedding similarity | 1 hour | Medium (~30%) |
| Prefix | Query prefix match | 5 min | Medium (~20%) |
| Template | Prompt template hash | 1 day | High (~70%) |

**🗡️ Dungeon Mastery:** Implement an LLM response cache with: exact match, semantic similarity threshold (0.95), TTL, and cache invalidation on model update.

**💻 Code — LLM Response Cache:**
```python
import hashlib
import json
import time
from sentence_transformers import SentenceTransformer
import numpy as np

class LLMCache:
    def __init__(self, similarity_threshold: float = 0.97, default_ttl: int = 3600):
        self.cache: dict[str, dict] = {}
        self.threshold = similarity_threshold
        self.ttl = default_ttl
        self.stats = {"hits": 0, "misses": 0, "semantic_hits": 0}

    def _exact_key(self, prompt: str, params: dict) -> str:
        content = prompt + json.dumps(params, sort_keys=True)
        return hashlib.md5(content.encode()).hexdigest()

    def get(self, prompt: str, params: dict = None) -> str | None:
        params = params or {}

        # 1. Exact match
        key = self._exact_key(prompt, params)
        if key in self.cache:
            entry = self.cache[key]
            if time.time() < entry["expires_at"]:
                self.stats["hits"] += 1
                return entry["response"]
            else:
                del self.cache[key]

        self.stats["misses"] += 1
        return None

    def set(self, prompt: str, response: str, params: dict = None, ttl: int = None):
        params = params or {}
        key = self._exact_key(prompt, params)
        self.cache[key] = {
            "response": response,
            "expires_at": time.time() + (ttl or self.ttl),
            "prompt": prompt,
            "params": params,
        }

    def invalidate_all(self):
        self.cache.clear()

    def size(self) -> int:
        return len(self.cache)

    def hit_rate(self) -> float:
        total = self.stats["hits"] + self.stats["misses"]
        return self.stats["hits"] / total if total > 0 else 0.0


class SemanticCache(LLMCache):
    """Cache that also checks embedding similarity."""
    def __init__(self, model_name: str = "all-MiniLM-L6-v2", **kwargs):
        super().__init__(**kwargs)
        self.embedder = SentenceTransformer(model_name)
        self.embeddings: dict[str, np.ndarray] = {}

    def get(self, prompt: str, params: dict = None) -> str | None:
        params = params or {}

        # 1. Try exact match first
        exact = super().get(prompt, params)
        if exact:
            return exact

        # 2. Try semantic match
        query_emb = self.embedder.encode([prompt])[0]
        best_score = 0
        best_key = None

        for key, entry in self.cache.items():
            if time.time() >= entry["expires_at"]:
                continue
            cached_emb = self.embeddings.get(key)
            if cached_emb is not None:
                score = np.dot(query_emb, cached_emb)
                if score > best_score and score >= self.threshold:
                    best_score = score
                    best_key = key

        if best_key:
            self.stats["semantic_hits"] += 1
            self.stats["hits"] += 1
            return self.cache[best_key]["response"]

        return None

    def set(self, prompt: str, response: str, params: dict = None, ttl: int = None):
        super().set(prompt, response, params, ttl)
        key = self._exact_key(prompt, params or {})
        self.embeddings[key] = self.embedder.encode([prompt])[0]
```

**👹 Boss Lesson:** Caching has two benefits: cost (fewer API calls) and latency (instant responses). Design your cache strategy before you launch.

---

## Dungeon 4 — Cost Optimization

**Day 4 | Concept:** LLM costs scale with usage. A successful application can bankrupt itself on API costs without careful optimization.

**Cost Levers:**
| Lever | Impact | Effort |
|-------|--------|--------|
| Model selection (GPT-4o vs GPT-4o-mini) | 20x cost difference | Low |
| Prompt compression | 30-50% token reduction | Medium |
| Caching | 30-70% request reduction | Medium |
| Batch processing | 50% cost reduction | Low |
| Fine-tuning smaller model | 10x inference cost reduction | High |

**Cost Math:**
```
GPT-4o: $5/1M input tokens, $15/1M output tokens
One RAG query: ~2000 input tokens + ~300 output tokens = ($5 × 2000/1M) + ($15 × 300/1M) = $0.0145/query
10K queries/day: $145/day = $4,350/month
→ Switch to GPT-4o-mini: $0.15/1M input, $0.60/1M output
→ 10K queries: $4.20/day = $126/month (97% reduction)
```

**🗡️ Dungeon Mastery:** Calculate monthly cost for a chatbot doing 50K queries/day: GPT-4 (current) vs GPT-4o-mini (proposed). Input: 1500 tokens. Output: 400 tokens. Cache hit rate: 20%. Show savings.

**💻 Code — Cost Calculator:**
```python
from dataclasses import dataclass

@dataclass
class ModelPricing:
    input_per_million: float  # $/1M input tokens
    output_per_million: float  # $/1M output tokens

MODELS = {
    "gpt-4o": ModelPricing(5.0, 15.0),
    "gpt-4o-mini": ModelPricing(0.15, 0.60),
    "claude-3.5-sonnet": ModelPricing(3.0, 15.0),
    "claude-3-haiku": ModelPricing(0.25, 1.25),
    "llama-3.1-8b": ModelPricing(0.05, 0.05),  # Self-hosted (approx)
}

class CostOptimizer:
    def __init__(self, model_name: str = "gpt-4o"):
        self.model = MODELS.get(model_name, MODELS["gpt-4o"])

    def query_cost(self, input_tokens: int = 1500, output_tokens: int = 300) -> float:
        input_cost = (input_tokens / 1_000_000) * self.model.input_per_million
        output_cost = (output_tokens / 1_000_000) * self.model.output_per_million
        return input_cost + output_cost

    def daily_cost(self, queries: int, input_tokens: int = 1500, output_tokens: int = 300, cache_hit_rate: float = 0.0) -> float:
        uncached = queries * (1 - cache_hit_rate)
        return uncached * self.query_cost(input_tokens, output_tokens)

    def monthly_cost(self, **kwargs) -> float:
        return self.daily_cost(**kwargs) * 30

    def compare_models(self, queries_per_day: int, input_tokens: int = 1500, output_tokens: int = 300, cache_rate: float = 0.0):
        results = {}
        for name, pricing in MODELS.items():
            self.model = pricing
            results[name] = {
                "daily": round(self.daily_cost(queries_per_day, input_tokens, output_tokens, cache_rate), 2),
                "monthly": round(self.monthly_cost(queries=queries_per_day, input_tokens=input_tokens, output_tokens=output_tokens, cache_hit_rate=cache_rate), 2),
                "per_query": round(self.query_cost(input_tokens, output_tokens), 5),
            }
        return results

    def savings_recommendation(self, current_model: str, proposed_model: str, queries_per_day: int, **kwargs):
        current = CostOptimizer(current_model)
        proposed = CostOptimizer(proposed_model)
        current_monthly = current.monthly_cost(queries=queries_per_day, **kwargs)
        proposed_monthly = proposed.monthly_cost(queries=queries_per_day, **kwargs)
        savings = current_monthly - proposed_monthly
        return {
            "current_cost_monthly": round(current_monthly, 2),
            "proposed_cost_monthly": round(proposed_monthly, 2),
            "monthly_savings": round(savings, 2),
            "annual_savings": round(savings * 12, 2),
            "reduction_pct": round((1 - proposed_monthly / current_monthly) * 100, 1) if current_monthly > 0 else 0,
        }
```

**👹 Boss Lesson:** Cost optimization is not optional. Without it, success is a liability. The best time to optimize costs is before you launch.

---

## Dungeon 5 — CI/CD for LLM Applications

**Day 5 | Concept:** LLM applications need CI/CD that goes beyond unit tests. Prompt changes, eval scores, and model updates need automated gates.

**CI/CD Pipeline Stages:**
```
1. Lint + Type check → 2. Unit tests → 3. Integration tests → 
4. Eval suite (≥85% pass) → 5. Prompt diff review → 
6. Canary deploy (10%) → 7. Monitor metrics → 8. Full rollout
```

**What Makes LLM CI/CD Different:**
- **Prompt changes** are code changes (version them!)
- **Eval scores** are test results (fail build if score drops)
- **Model updates** need A/B testing (canary deployment)
- **Evil input testing** should be automated (red team in CI)

**🗡️ Dungeon Mastery:** Write a GitHub Actions workflow for an LLM app. Steps: lint, test, eval suite, build Docker, deploy to staging, run smoke tests.

**💻 Code — CI/CD Pipeline:**
```yaml
# .github/workflows/llm-pipeline.yml
"""
name: LLM CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint
        run: |
          pip install ruff mypy
          ruff check .
          mypy app/

      - name: Unit tests
        run: pytest tests/unit/ --cov=app --cov-fail-under=80

      - name: Integration tests
        run: |
          docker compose up -d
          pytest tests/integration/ --timeout=60

      - name: Eval suite
        run: |
          python -m app.evals.run --dataset evals/dataset.json \
            --threshold 0.85 --fail-on-drop

      - name: Red team checks
        run: |
          python -m app.guardrails.redteam --tests evals/redteam.json \
            --min-pass-rate 0.8

      - name: Build Docker
        if: github.ref == 'refs/heads/main'
        run: docker build -t llm-app:${{ github.sha }} .

      - name: Deploy canary
        if: github.ref == 'refs/heads/main'
        run: |
          kubectl set image deployment/llm-app \
            llm-app=llm-app:${{ github.sha }} \
            --record
          kubectl scale deployment/llm-app --replicas=1
          # Wait for canary health
          sleep 30
          kubectl scale deployment/llm-app --replicas=3
"""
```

**🗡️ Dungeon Mastery:** Write a GitHub Actions workflow for an LLM app. Include: linting, unit tests, eval suite (with threshold), Docker build, and canary deploy.

**👹 Boss Lesson:** LLM CI/CD is not optional. Without automated eval gates, you will deploy a regression. Eventually, you will be sorry.

---

## Dungeon 6 — Incident Response

**Day 6 | Concept:** LLM systems fail differently than traditional systems. Hallucinations, prompt injections, and model degradation need specific response plans.

**Incident Types:**
| Type | Severity | Response |
|------|----------|----------|
| Model down | Critical | Failover, fallback model |
| High latency | Major | Scale up, disable non-critical features |
| Toxic output | Critical | Block output, rollback prompt |
| Cost spike | Major | Rate limit, check for abuse |
| Eval regression | Minor | Rollback last change, investigate |
| Data leakage | Critical | Shutdown endpoint, rotate keys |

**Incident Response Steps:**
1. **Detect** — Monitoring alert or user report
2. **Triage** — Severity assessment
3. **Contain** — Block bad traffic / rollback
4. **Resolve** — Fix root cause
5. **Learn** — Add regression test, update runbook

**🗡️ Dungeon Mastery:** Write a runbook for "Toxic output detected in production." Include: detection method, immediate containment steps, root cause investigation, fix and verification.

**💻 Code — Incident Response Framework:**
```python
from enum import Enum
from datetime import datetime
from dataclasses import dataclass, field

class Severity(Enum):
    CRITICAL = "critical"  # Data breach, ongoing harm
    MAJOR = "major"        # Degraded experience
    MINOR = "minor"        # Non-urgent issue

class IncidentStatus(Enum):
    DETECTED = "detected"
    TRIAGED = "triaged"
    CONTAINED = "contained"
    RESOLVED = "resolved"
    POSTMORTEM = "postmortem"

@dataclass
class Incident:
    id: str
    title: str
    severity: Severity
    status: IncidentStatus = IncidentStatus.DETECTED
    detected_at: str = field(default_factory=lambda: datetime.now().isoformat())
    resolved_at: str = ""
    summary: str = ""
    timeline: list[str] = field(default_factory=list)
    action_items: list[str] = field(default_factory=list)

    def contain(self, action: str):
        self.status = IncidentStatus.CONTAINED
        self.timeline.append(f"[Containment] {action} at {datetime.now().isoformat()}")

    def resolve(self, action: str):
        self.status = IncidentStatus.RESOLVED
        self.resolved_at = datetime.now().isoformat()
        self.timeline.append(f"[Resolution] {action} at {self.resolved_at}")

    def postmortem(self, summary: str, action_items: list[str]):
        self.status = IncidentStatus.POSTMORTEM
        self.summary = summary
        self.action_items = action_items


# Runbook template:
TOXIC_OUTPUT_RUNBOOK = """
## Runbook: Toxic Output Detected

### Detection
- Monitor alert: Toxicity score > 0.9
- User report with screenshot
- Manual review flagged output

### Immediate Containment (within 5 minutes)
1. Block the affected user's session
2. Add specific input/output pattern to guardrail blocklist
3. If widespread: switch to fallback model with stricter settings
4. Notify on-call via incident channel

### Root Cause Investigation
1. Check guardrail logs: Was input flagged?
2. Check prompt version: Was there a recent change?
3. Check model response: Is this a known failure mode?
4. Reproduce with exact input

### Fix (within 1 hour)
1. Rollback prompt change if applicable
2. Add specific pattern to guardrails
3. Add regression test to eval suite
4. Deploy fix with canary

### Postmortem (within 24 hours)
1. Document timeline and root cause
2. Add automated detection for this pattern
3. Update runbook with lessons
4. Review: could guardrails have caught this earlier?
"""
```

**👹 Boss Lesson:** The difference between a good team and a bad team is incident response. Plan for failure before it happens.

---

## Dungeon 7 — Load Testing & Scaling

**Day 7 | Concept:** You don't know your system's limits until you test them. Load testing reveals bottlenecks before users do.

**What to Measure:**
| Metric | Goal | Warning |
|--------|------|---------|
| Requests/second | Target RPS | >80% of target |
| p50 latency | <200ms | >500ms |
| p95 latency | <500ms | >2s |
| Error rate | <0.1% | >1% |
| GPU utilization | 70-90% | >95% (bottleneck) |

**Scaling Strategies:**
- **Horizontal:** More replicas (Kubernetes HPA)
- **Vertical:** Bigger GPU (A10 → A100 → H100)
- **Optimization:** Quantization, batching, caching
- **Queuing:** Request buffer during spikes (Bull, Celery)

**🗡️ Dungeon Mastery:** Run a load test scenario: Your app gets 1000 simultaneous users after a product launch. Current capacity: 100 RPS. Design the scaling plan (how many more replicas? what's the queue strategy?).

**💻 Code — Load Testing:**
```python
import time
import json
import requests
from concurrent.futures import ThreadPoolExecutor, as_completed
from dataclasses import dataclass
import statistics

@dataclass
class LoadTestResult:
    total_requests: int
    successful: int
    failed: int
    p50_latency: float
    p95_latency: float
    p99_latency: float
    avg_latency: float
    rps: float

class LoadTester:
    def __init__(self, endpoint: str, concurrency: int = 10):
        self.endpoint = endpoint
        self.concurrency = concurrency

    def run(self, num_requests: int, payload: dict) -> LoadTestResult:
        latencies = []
        successful = 0
        failed = 0

        def make_request():
            nonlocal successful, failed
            start = time.time()
            try:
                resp = requests.post(self.endpoint, json=payload, timeout=30)
                latency = (time.time() - start) * 1000
                if resp.status_code == 200:
                    successful += 1
                else:
                    failed += 1
                return latency
            except Exception:
                failed += 1
                return None

        with ThreadPoolExecutor(max_workers=self.concurrency) as executor:
            futures = [executor.submit(make_request) for _ in range(num_requests)]
            for future in as_completed(futures):
                latency = future.result()
                if latency is not None:
                    latencies.append(latency)

        sorted_lat = sorted(latencies)
        total_time = (sorted_lat[-1] if sorted_lat else 0) / 1000  # seconds

        return LoadTestResult(
            total_requests=num_requests,
            successful=successful,
            failed=failed,
            p50_latency=sorted_lat[len(sorted_lat) // 2] if sorted_lat else 0,
            p95_latency=sorted_lat[int(len(sorted_lat) * 0.95)] if sorted_lat else 0,
            p99_latency=sorted_lat[int(len(sorted_lat) * 0.99)] if sorted_lat else 0,
            avg_latency=statistics.mean(latencies) if latencies else 0,
            rps=successful / total_time if total_time > 0 else 0,
        )

    def ramp_up(self, start_concurrency: int = 1, max_concurrency: int = 50, step: int = 5, requests_per_step: int = 20, payload: dict = None):
        payload = payload or {"prompt": "Hello"}
        results = []
        for concurrency in range(start_concurrency, max_concurrency + 1, step):
            print(f"Testing concurrency={concurrency}...")
            self.concurrency = concurrency
            result = self.run(requests_per_step, payload)
            results.append({"concurrency": concurrency, **result.__dict__})
            if result.error_rate > 0.1:  # >10% errors
                print(f"  Breaking: error rate {result.error_rate:.1%}")
                break
        return results
```

**👹 Boss Lesson:** Load test BEFORE launch. Production is not the place to discover your system handles 10 RPS when you need 100.

---

## ⚔️ BOSS BATTLE: The Production Deployment

**Objective:** Design and document a complete production deployment for an LLM application that:
1. Architecture diagram with 4+ services
2. Containerized with Docker (multi-stage build)
3. Caching layer (semantic or exact-match)
4. Cost optimization analysis (compare 3 models)
5. CI/CD pipeline with eval gates
6. Incident response runbook for 3 scenarios
7. Load test plan with target metrics

**🗡️ Victory Conditions:**
- Architecture has no single point of failure
- Dockerfile follows production best practices
- Cache strategy is appropriate for use case
- Cost analysis shows ≥ 50% savings vs naive approach
- CI/CD includes eval gates with thresholds
- Runbooks cover detection, containment, resolution
- Load testing validates target throughput

**🏆 Rewards:**
- +500 XP
- A Rank → A+ Rank
- Skill Unlock: **Architect's Vision** — See production issues before they happen
- Title: **The Architect**

---

*"Architecture is what you build before you build. Production is where architecture meets reality."*
