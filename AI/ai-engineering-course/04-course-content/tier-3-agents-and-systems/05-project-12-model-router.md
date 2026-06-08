# Project 12: Cost-Aware Model Router (Production)

- **Tier:** 3 — Real Constraints
- **Project #:** 12 of 12 (final project of Tier 3)
- **Tech Stack:** LiteLLM (Router), Redis (caching + rate limiting), Langfuse (observability), FastAPI, Docker Compose
- **Concepts:** Production model routing, semantic caching, fallback chains, cost optimization, observability, A/B testing, CI/CD integration
- **Quality Gate:** ✅ APPROVED when 40%+ cost savings vs always-expensive model with < 3% quality degradation measured by comprehensive eval suite

---

## Phase 1: Brief (Priya)

> *Priya: "Remember Project 7? Rohan loved the savings. Now he wants the PRODUCTION version. With bells on."*

**What this is:** You're rebuilding Project 7 (Multi-Model API Gateway) as a production-grade internal infrastructure service.

**What changed from Project 7:**
- More traffic (1000+ requests/day from 5+ internal apps)
- Need for persistent caching (Redis)
- Need for semantic caching (similar queries get cached results)
- Need for A/B testing (compare routing strategies)
- Need for CI/CD pipeline (changes go through automated testing)
- Need for proper deployment (Docker Compose, health checks, graceful shutdown)

**Non-negotiable:**
- 40%+ cost savings vs always using the best model (same as Project 7)
- Quality degradation < 3% (measured by automated eval suite)
- Sub-500ms median latency (caching helps here)
- 99.5% uptime (fallback chains ensure this)
- CI/CD pipeline that tests routing quality before deployment
- Dashboard: real-time cost, latency, quality metrics

**Deadline:** 10 days. Rohan wants this running as the default API layer before next month's billing cycle.

**Definition of done:** Internal apps switch their LLM calls from direct API to this gateway. Costs drop 40%+. Quality doesn't drop measurably. Dashboard shows live metrics.

---

## Phase 2: Learning Path (Maya)

> *Maya: "This is the capstone of Tier 3. It brings together everything: cost awareness from Project 2, API design from Project 3, routing from Project 7, evaluation from Project 4, caching from Project 8, and observability from Project 9."*

### What's New vs Project 7

| Feature | Project 7 (Prototype) | Project 12 (Production) |
|---|---|---|
| **Caching** | In-memory (volatile) | Redis (persistent, shared across instances) |
| **Cache type** | Exact-match only | Exact + semantic (similar queries) |
| **Routing strategy** | Hardcoded | Configurable + A/B testable |
| **Monitoring** | Console logs | Dashboard (Langfuse + custom) |
| **Deployment** | Python script | Docker Compose + CI/CD |
| **Quality checks** | Manual eval | Automated eval suite in CI |
| **Fallback** | Simple retry | Multi-layer: retry → different model → different provider |

### Learning Order

**Step 1: Redis caching (exact match).**
Replace your in-memory cache with Redis. Same logic, persistent storage. Cache survives restarts. Shared across instances.

**Step 2: Semantic caching.**
Hash the query embedding instead of the raw text. Similar queries (cosine similarity > 0.95) return cached results. This catches 15-25% more cache hits.

**Step 3: Multi-strategy routing.**
Support multiple routing strategies (cost-based, latency-based, manual). Make them configurable. Add A/B testing: compare two strategies on live traffic.

**Step 4: Eval suite in CI.**
Every change to routing config triggers: run eval suite → compare quality metrics → approve/deploy.

**Step 5: Dashboard.**
Real-time metrics: cost per model, latency P50/P95/P99, cache hit rate, quality score over time.

### First Concrete Step

> "Install Redis locally (`docker run -p 6379:6379 redis`). Write a script that connects, sets a key, gets it, and deletes it. Then wrap your exact-match cache logic around Redis."

---

## Phase 3: The Build

> *Production infrastructure. Every decision has operational consequences.*

### Milestone 1: Redis Caching Layer

```python
import redis
import json
import hashlib

class RedisCache:
    def __init__(self, redis_url: str, default_ttl: int = 3600):
        self.client = redis.from_url(redis_url)
        self.default_ttl = default_ttl
    
    def exact_cache_key(self, model: str, messages: list) -> str:
        """Hash the exact request for cache lookup."""
        content = json.dumps({"model": model, "messages": messages}, sort_keys=True)
        return f"exact:{hashlib.md5(content.encode()).hexdigest()}"
    
    def get_exact(self, model: str, messages: list) -> dict | None:
        key = self.exact_cache_key(model, messages)
        cached = self.client.get(key)
        return json.loads(cached) if cached else None
    
    def set_exact(self, model: str, messages: list, response: dict, ttl: int = None):
        key = self.exact_cache_key(model, messages)
        self.client.setex(key, ttl or self.default_ttl, json.dumps(response))

    def semantic_cache_key(self, query: str) -> str:
        """Use embedding hash for semantic similarity matching."""
        embedding = embedding_model.encode(query)
        # Use first 8 bytes of embedding as rough hash
        rough_hash = hashlib.md5(embedding.tobytes()).hexdigest()[:16]
        return f"semantic:{rough_hash}"
```

### Milestone 2: Semantic Caching

```python
class SemanticCache:
    """Caches responses for semantically similar queries."""
    
    SIMILARITY_THRESHOLD = 0.95
    
    def __init__(self, redis_client: RedisCache):
        self.redis = redis_client
    
    def find_similar(self, query: str) -> dict | None:
        """Check if a semantically similar query has been cached."""
        query_embedding = embedding_model.encode(query)
        
        # Get candidate cache keys (same rough hash bucket)
        candidate_key = self.redis.semantic_cache_key(query)
        candidates = self.redis.client.smembers(f"semantic_bucket:{candidate_key}")
        
        for candidate in candidates:
            candidate_data = json.loads(candidate)
            candidate_embedding = np.array(candidate_data["embedding"])
            
            similarity = cosine_similarity(query_embedding, candidate_embedding)
            if similarity >= self.SIMILARITY_THRESHOLD:
                return candidate_data["response"]
        
        return None
    
    def store(self, query: str, response: dict):
        """Store query+response for future semantic matching."""
        query_embedding = embedding_model.encode(query)
        candidate_key = self.redis.semantic_cache_key(query)
        
        entry = json.dumps({
            "query": query,
            "embedding": query_embedding.tolist(),
            "response": response
        })
        
        self.redis.client.sadd(f"semantic_bucket:{candidate_key}", entry)
        self.redis.client.expire(f"semantic_bucket:{candidate_key}", 3600)
```

**Expected stuck point:** Semantic caching adds latency. Embedding the query + similarity search against candidates takes 50-100ms. For a cache that only hits 15% of the time, the latency cost might outweigh the benefit.

**Maya's Socratic question:**
> *"Your semantic cache adds 80ms to every request but only hits 12% of the time. The math says this is a net latency INCREASE for most users. When would semantic caching actually be worth it?"**

> They should: (1) measure carefully before deploying, (2) only do semantic cache lookup if exact cache misses (optimistic approach), (3) use a very fast embedding model (all-MiniLM-L6-v2, ~5ms), (4) consider that cache hit saves 1-5s of LLM time, so even 80ms overhead is worth it.

### Milestone 3: A/B Testing Framework

```python
class ABTestRouter:
    """Routes traffic between two routing strategies for comparison."""
    
    def __init__(self, strategy_a: Router, strategy_b: Router, split: float = 0.5):
        self.strategy_a = strategy_a
        self.strategy_b = strategy_b
        self.split = split  # 0.5 = 50/50 split
        self.results = []
    
    def route(self, request: dict) -> dict:
        """Route to A or B based on split."""
        import random
        use_a = random.random() < self.split
        
        router = self.strategy_a if use_a else self.strategy_b
        start = time.time()
        response = router.completion(**request)
        latency = (time.time() - start) * 1000
        
        self.results.append({
            "strategy": "A" if use_a else "B",
            "latency_ms": latency,
            "model": response.model if hasattr(response, 'model') else "unknown",
            "cost": response.cost if hasattr(response, 'cost') else 0,
            "timestamp": datetime.now().isoformat()
        })
        
        return response
    
    def get_comparison(self) -> dict:
        """Compare A vs B performance."""
        a_results = [r for r in self.results if r["strategy"] == "A"]
        b_results = [r for r in self.results if r["strategy"] == "B"]
        
        return {
            "strategy_a": {
                "avg_latency_ms": mean(r["latency_ms"] for r in a_results),
                "avg_cost": mean(r["cost"] for r in a_results),
                "sample_size": len(a_results)
            },
            "strategy_b": {
                "avg_latency_ms": mean(r["latency_ms"] for r in b_results),
                "avg_cost": mean(r["cost"] for r in b_results),
                "sample_size": len(b_results)
            },
            "winner": "A" if mean(r["cost"] for r in a_results) < mean(r["cost"] for r in b_results) else "B"
        }
```

### Milestone 4: CI/CD Eval Pipeline

```python
# In your CI config (GitHub Actions):
"""
name: Routing Quality Check
on:
  pull_request:
    paths:
      - 'routing_config.yaml'

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run routing eval
        run: python eval_routing.py --config ${{ github.head_ref }}.yaml
      - name: Compare quality
        run: python compare_quality.py --baseline main.yaml --candidate ${{ github.head_ref }}.yaml
      - name: Fail if degradation > 3%
        run: |
          DEGRADATION=$(cat quality_report.json | jq '.degradation_percentage')
          if (( $(echo "$DEGRADATION > 3.0" | bc -l) )); then
            echo "Quality degradation $DEGRADATION% exceeds 3% threshold"
            exit 1
          fi
"""

def eval_routing_config(config_path: str, test_set_path: str) -> dict:
    """Run quality evaluation on a routing config."""
    with open(config_path) as f:
        config = yaml.safe_load(f)
    
    router = Router.from_config(config)
    test_queries = json.load(open(test_set_path))
    
    results = []
    for test in test_queries:
        response = router.completion(
            model="auto",  # router decides
            messages=[{"role": "user", "content": test["query"]}]
        )
        
        # Compare with expensive model reference
        reference = call_expensive_model(test["query"])
        quality = judge_quality(test["query"], response.content, reference)
        
        results.append({
            "query": test["query"],
            "routed_cost": getattr(response, "cost", 0),
            "reference_cost": reference_cost,
            "quality_score": quality,
        })
    
    degradation = calculate_degradation(results)
    cost_savings = calculate_savings(results)
    
    return {
        "degradation_percentage": degradation,
        "cost_savings_percentage": cost_savings,
        "num_queries": len(results),
        "pass": degradation < 3.0 and cost_savings > 40.0
    }
```

### Rohan's Mid-Build Interruption

> *Rohan appears during A/B testing.*

"I see you're A/B testing two routing strategies. But your A/B framework doesn't account for DAY OF WEEK effects — traffic patterns on Monday are different from Saturday. Your 24-hour A/B test will be contaminated by day-of-week bias.

Run both strategies for 7 full days (Mon-Sun) before declaring a winner. Or better: use a time-aware randomization that stratifies by day/hour."

### Priya's Requirement Change

> *Priya: "The product team wants to use your gateway but they need it to support CUSTOM routing rules per team. Team Alpha wants ALL their traffic to go through GPT-4o for quality. Team Beta wants cheapest possible. Can you add per-team routing config?"*

This forces: multi-tenant routing with per-team configuration, team-level API keys and rate limits, team-level cost tracking in the dashboard.

---

## Phase 4: Review (Rohan)

> *You submit: Production gateway with Redis caching, semantic caching, A/B testing, CI/CD pipeline, dashboard, per-team routing.*

### Decision Documentation Required

1. Your caching strategy — exact vs semantic, hit rates, latency impact, cost savings
2. Your routing strategy comparison — A/B test results, which strategy won and why
3. Your CI/CD quality gate — how you measure quality degradation, threshold, what happens on failure
4. Your architecture — how the components (Router, Cache, DB, Dashboard) fit together

### Rohan's Review

| # | Check | Status | Notes |
|---|---|---|---|
| 1 | **Does it work?** | ✅ PASS | Gateway handles 1000+ requests/day. 99.7% uptime over test period. |
| 2 | **Edge cases?** | ✅ PASS | Handles all failure modes: model down → fallback, cache miss → fresh, rate limit → retry. |
| 3 | **Cost-aware?** | ✅ PASS | 43.2% cost savings. $4,800/month saved on projected $11,200 baseline. |
| 4 | **Observable?** | ✅ PASS | Langfuse traces every request. Dashboard shows cost/latency/quality by model and team. |
| 5 | **Right approach?** | ✅ PASS | Semantic caching is aggressive (0.95 threshold) but measured correctly. A/B testing framework is sound. CI/CD quality gate prevents regressions. |
| 6 | **Decisions justified?** | ✅ PASS | Every component choice is documented and measured. |
| 7 | **Measurable quality?** | ✅ PASS | 2.1% quality degradation (under 3% gate). 43.2% cost savings (over 40% gate). |

### Verdict: ✅ APPROVED

*Rohan closes your dashboard.*

"This passes. Genuinely. This is the kind of infrastructure that makes a company's AI operation efficient instead of expensive.

**You've completed Tier 3.** That means:

- You can build agents with tool use and multi-step reasoning
- You can implement advanced RAG with agentic retrieval
- You can design and evaluate production AI systems with real constraints
- You can build cost-aware infrastructure with observability

Tier 4 is waiting. And Tier 4 is not like anything you've done before."

---

## Phase 5: Debrief (Maya)

> *After APPROVED. Maya is sitting on the edge of your desk.*

**Maya:** "Twelve projects. Three tiers. Here's the truth:

**In Tier 1, you learned to call the API.** You learned structured outputs, batch processing, and embeddings.

**In Tier 2, you learned to architect production systems.** RAG, hybrid search, multi-tenant RAG, batch extraction, model routing.

**In Tier 3, you learned to build agents** that think, act, observe, and decide. You built sales agents, financial RAG systems, code reviewers, text-to-SQL engines, and production infrastructure.

### What You Are Now

You're not a junior AI engineer. You're not even a mid-level. You're an **AI Engineer** who can:
- Identify when to use RAG vs agents vs simple prompting
- Design and evaluate complex AI systems
- Optimize for cost, latency, and quality simultaneously
- Build production infrastructure that survives real traffic

### What Tier 4 Changes

| Dimension | Tier 3 | Tier 4 |
|---|---|---|
| **Systems** | Single agent / single pipeline | Multi-agent orchestration |
| **Complexity** | One AI decision per step | Multiple AI systems coordinating |
| **Stakes** | Production constraints | Portfolio-level, job-interview-grade |
| **Autonomy** | Minimal guidance | You design the architecture |
| **Human-in-loop** | Simple escalation | Complex human-in-the-loop workflows |

### The 4 Projects

| Project | What You Build |
|---|---|
| **13** | Deep Research Agent — autonomous research with 50+ sources |
| **14** | Multi-Agent Content Pipeline — 5 specialized agents coordinating |
| **15** | Autonomous Customer Support System — 80% automation at 1000+ tickets/day |
| **16** | AI-Powered Code Generation Platform — NL spec → code → test → iterate |

> *Maya holds out her hand. "Tier 4 is the endgame. These are the projects that go on your resume and prove you can architect anything."*
