# Project 7: Multi-Model API Gateway

- **Tier:** 2 — First Real Clients
- **Project #:** 7 of 7 (final project of Tier 2)
- **Tech Stack:** FastAPI, LiteLLM (Router), Redis (optional cache), Langfuse (optional tracing), cost logging
- **Concepts:** Model routing, cost optimization, fallback strategies, unified API interface, prompt classification for routing, semantic caching
- **Quality Gate:** ✅ APPROVED when routing saves at least 40% cost vs always using the best model, with < 5% quality degradation measured by LLM-as-judge

---

## Phase 1: Brief (Priya)

> *Priya walks in holding a credit card statement. She places it on the table with the gravity of an indictment.*

"This is last month's OpenAI bill. $8,400.

NexaAI has 5 internal tools and 3 client systems all hitting LLM APIs. Every team uses the best model for everything. Quick classification? GPT-4o. Simple translation? GPT-4o. Complex legal reasoning? GPT-4o. Every call goes to the most expensive model.

Rohan is not happy. He said, and I quote: *'We're burning money. A high schooler with a logic gate could route simple queries to cheap models. Build me the routing layer.'*

**What we're building:** An internal API gateway that sits between our applications and LLM providers. Every app sends its request to ONE endpoint. The gateway:
1. Classifies the query complexity
2. Routes to the cheapest adequate model (GPT-4o-mini for simple, GPT-4o for medium, Claude Sonnet/Opus for complex)
3. Falls back if the primary model fails
4. Logs every call for cost tracking
5. Caches identical queries so we don't pay twice

**Non-negotiable:**
- Single unified API (our apps shouldn't know or care which model handles their request)
- Must save at least 40% vs always using the most expensive model
- Quality degradation < 5% (measured by LLM-as-judge on a held-out test set)
- Must work with OpenAI, Anthropic, AND any future provider
- If a model is down, must failover automatically

**Deadline:** 5 days. Rohan wants to see the next bill drop.

**Definition of done:** Internal apps send requests to `POST /v1/chat`. Gateway routes to the optimal model. Dashboard shows cost savings. Test set proves < 5% quality loss for 40%+ cost reduction."

---

## Phase 2: Learning Path (Maya)

> *Maya: "I love this project. You know why? Because the solution isn't a better model. It's an ECONOMIC strategy wrapped in a technical layer. This is where you stop being a 'prompt engineer' and start being a systems engineer."*

### Why This Is Hard

"Model routing sounds simple — 'cheap model for easy stuff, expensive model for hard stuff.' But:
- How do you define 'easy' vs 'hard' programmatically?
- What happens when the cheap model gives a wrong answer?
- How do you measure quality degradation without human review for every query?
- What happens when all models of a tier are down?
- How do you know your routing strategy is actually saving money without detailed telemetry?"

### Learning Order (Scratch-First)

**Step 1: Build the classifier manually.**
Write a function that analyzes a query and classifies it as 'simple,' 'medium,' or 'complex.' Start with rule-based heuristics (query length, presence of technical terms). Then try LLM-based classification. Compare accuracy.

> *"The classifier IS the routing strategy. If it's wrong, the routing is wrong. Get this right first."*

**Step 2: Build the router without LiteLLM.**
Write a manual router that takes a query, runs the classifier, and picks a model based on the tier. Simple if/else logic. Make it work. Feel the pain of maintaining this yourself.

**Step 3: Introduce LiteLLM.**
Replace your manual router with LiteLLM's Router. It handles: multi-provider abstraction, fallbacks, retries, rate limiting, cost tracking. See how much it simplifies your code.

> *"This is the framework introduction rhythm. You built it manually first. You felt the pain. Now LiteLLM feels like a gift, not a magic box."*

**Step 4: Add caching.**
If the same query comes twice, return cached. Start with exact-match caching. Then add semantic caching (similar queries get cached results).

**Step 5: Build the quality monitor.**
Use LLM-as-judge to compare routing decisions. Did routing to a cheap model hurt quality? Sample 20% of requests and compare full-quality vs routed response.

### Memory Triage

**Memorize cold:**
- LiteLLM Router initialization pattern — `Router(model_list=[...], routing_strategy="cost-based-routing")`
- The unified API pattern — single endpoint that hides provider switching from consumers
- Cost tracking pattern — log model, tokens, cost per request

**Look up when needed:**
- LiteLLM routing strategies (simple-shuffle, cost-based, latency-based)
- LiteLLM fallback configuration syntax
- Redis client commands for caching
- Specific model pricing (changes monthly — always check current rates)

**Understand deeply:**
- The cost-quality Pareto frontier — *"the last 5% of quality costs 10x more. Smart routing knows when to pay and when to save."*
- Why caching is not just about cost — *"caching also reduces latency. A cache hit is 10ms. An LLM call is 1-5 seconds. For user-facing apps, caching IS the UX strategy."*

### First Concrete Step

> "Install LiteLLM. Write a 10-line script that sends the same query to GPT-4o-mini AND GPT-4o. Compare the outputs. Time them. Cost them. Print the difference. This is the baseline for your routing decisions."

---

## Phase 3: The Build

> *You're building the economic brain of NexaAI's AI infrastructure. Every dollar saved is a dollar Rohan doesn't yell about.*

### Milestone 1: Query Complexity Classifier

Build the classification system:

```python
from pydantic import BaseModel, Field
from enum import Enum
import re

class ComplexityTier(str, Enum):
    SIMPLE = "simple"    # classification, extraction, translation
    MEDIUM = "medium"    # analysis, summarization, structured output
    COMPLEX = "complex"  # reasoning, code generation, multi-step logic

class ClassificationResult(BaseModel):
    tier: ComplexityTier
    reasoning: str = Field(..., description="Why this tier was chosen")
    confidence: float = Field(..., ge=0.0, le=1.0)

# Rule-based classifier (fast, no LLM cost)
def classify_heuristic(query: str) -> ComplexityTier:
    """Fast rule-based classification. ~1ms, no API call needed."""
    query_lower = query.lower()
    
    # Complex indicators
    complex_keywords = [
        "compare", "contrast", "analyze", "synthesize", "evaluate",
        "why", "explain the reasoning", "step by step", "calculate",
        "code", "function", "algorithm", "architecture"
    ]
    if any(kw in query_lower for kw in complex_keywords):
        return ComplexityTier.COMPLEX
    
    # Medium indicators
    medium_keywords = [
        "summarize", "translate", "describe", "explain",
        "list", "categorize", "format"
    ]
    if any(kw in query_lower for kw in medium_keywords):
        return ComplexityTier.MEDIUM
    
    # Default to simple
    return ComplexityTier.SIMPLE

# LLM-based classifier (more accurate, costs $0.001 per call)
def classify_llm(query: str) -> ClassificationResult:
    """More accurate classification using a cheap model."""
    response = client.beta.chat.completions.parse(
        model="gpt-4o-mini",  # classification doesn't need a smart model
        messages=[
            {"role": "system", "content": "Classify this query as simple, medium, or complex."},
            {"role": "user", "content": query}
        ],
        response_format=ClassificationResult,
        temperature=0.0
    )
    return response.choices[0].message.parsed
```

**Expected stuck point:** Rule-based classification is fast but misses nuance. LLM-based classification is more accurate but costs money and adds latency. The "right" approach is: use rules as a fast path, fall back to LLM only when rules are uncertain.

**Maya's Socratic question:**
> *"Your rule-based classifier misclassified 'write a simple thank you email' as SIMPLE. But writing email that sounds natural and human might need MEDIUM complexity. How do you handle queries where the apparent complexity differs from the actual complexity?"*

> They should: use a confidence score. If rule-based confidence is high (<0.8), use it. If low, pass to LLM classifier. This hybrid approach is the 2026 production standard.

### Milestone 2: Manual Router → LiteLLM

First, build it manually to feel the pain:

```python
def manual_router(query: str) -> str:
    """Manual routing — works but painful to maintain."""
    tier = classify_heuristic(query)
    
    if tier == ComplexityTier.SIMPLE:
        model = "gpt-4o-mini"
    elif tier == ComplexityTier.MEDIUM:
        model = "gpt-4o"
    else:
        model = "claude-sonnet-4-20250514"
    
    # Call the model... but which API? OpenAI vs Anthropic have different clients
    # Need if/else for each provider
    if "gpt" in model:
        response = openai_client.chat.completions.create(model=model, ...)
    elif "claude" in model:
        response = anthropic_client.messages.create(model=model, ...)
    # Now add error handling for each...
    # Now add retries for each...
    # Now add cost tracking for each...
    
    return response
```

Now replace with LiteLLM:

```python
from litellm import Router

router = Router(
    model_list=[
        {
            "model_name": "cheap",
            "litellm_params": {
                "model": "gpt-4o-mini",
                "api_key": os.getenv("OPENAI_API_KEY"),
            },
        },
        {
            "model_name": "medium",
            "litellm_params": {
                "model": "gpt-4o",
                "api_key": os.getenv("OPENAI_API_KEY"),
            },
        },
        {
            "model_name": "expensive",
            "litellm_params": {
                "model": "anthropic/claude-sonnet-4-20250514",
                "api_key": os.getenv("ANTHROPIC_API_KEY"),
            },
        },
    ],
    fallbacks=[{"cheap": ["medium"]}, {"medium": ["expensive"]}],
    routing_strategy="latency-based-routing",
    num_retries=2,
    cooldown_time=30,
)

def route_query(query: str, system_prompt: str = None) -> str:
    tier = classify_heuristic(query)
    
    model_name = {
        ComplexityTier.SIMPLE: "cheap",
        ComplexityTier.MEDIUM: "medium", 
        ComplexityTier.COMPLEX: "expensive",
    }[tier]
    
    messages = []
    if system_prompt:
        messages.append({"role": "system", "content": system_prompt})
    messages.append({"role": "user", "content": query})
    
    response = router.completion(
        model=model_name,
        messages=messages,
        temperature=0.7
    )
    
    return response.choices[0].message.content
```

**Expected stuck point:** LiteLLM `Router.completion()` returns a standard OpenAI-compatible response object. But different models have different capabilities — some support `response_format`, some don't. You need to handle this transparently.

**Maya's Socratic question:**
> *"You're routing queries to different models. But 'gpt-4o-mini' supports structured outputs with strict mode, while some models don't. If your app depends on structured outputs, how do you make routing transparent to the caller?"*

> They should: (1) standardize on models that ALL support the features your apps need, or (2) degrade gracefully — if the routed model doesn't support a feature, fall back to one that does, or (3) detect capability per model and wrap appropriately.

### Milestone 3: Cost + Quality Monitoring

Build the monitoring layer:

```python
import time
from datetime import datetime

class RoutingTelemetry:
    """Tracks cost, latency, and quality per request."""
    
    def __init__(self):
        self.requests = []
    
    def log_request(self, query: str, tier: str, model: str, 
                    cost: float, latency_ms: int, 
                    quality_score: float = None):
        self.requests.append({
            "timestamp": datetime.now().isoformat(),
            "query_preview": query[:100],
            "tier": tier,
            "model": model,
            "cost": cost,
            "latency_ms": latency_ms,
            "quality_score": quality_score,
        })
    
    def get_cost_savings(self) -> dict:
        """Compare actual cost vs cost of always using expensive model."""
        total_actual = sum(r["cost"] for r in self.requests)
        
        # Simulate what it would cost if all went to expensive model
        # (This is approximate — actual cost depends on token count)
        total_if_expensive = total_actual * 5  # rough multiplier
        
        savings_pct = (1 - total_actual / total_if_expensive) * 100
        return {
            "actual_cost": total_actual,
            "projected_expensive_cost": total_if_expensive,
            "savings_pct": savings_pct,
            "total_requests": len(self.requests)
        }
```

**Expected stuck point:** True cost measurement requires tracking INPUT tokens AND OUTPUT tokens per model, because different models charge different rates for input vs output.

**Maya's Socratic question:**
> *"Your cost calculation assumes all tokens cost the same. But GPT-4o charges $10/M output tokens vs $2.50/M input tokens. A query with a 500-token output costs more than a query with a 500-token input. How do you model true cost?"*

> They should: use model-specific pricing tables, track input and output tokens separately, compute cost as: `(input_tokens * input_rate) + (output_tokens * output_rate)`. LiteLLM's response object includes token counts — use them.

### Milestone 4: Quality Validation

Prove the router doesn't degrade quality:

```python
def validate_routing_quality(test_queries: list[dict], router_fn) -> dict:
    """
    test_queries: [{"query": str, "tier": "simple"|"medium"|"complex", "expected": str}]
    """
    results = []
    
    for test in test_queries:
        # Get routed (cheap) answer
        routed_answer = router_fn(test["query"])
        
        # Get expensive answer (ground truth proxy)
        expensive_answer = call_expensive_model(test["query"])
        
        # LLM-as-judge: compare quality
        quality = judge_quality(
            query=test["query"],
            routed_answer=routed_answer,
            expensive_answer=expensive_answer
        )
        
        results.append({
            "query": test["query"],
            "tier": test["tier"],
            "quality_score": quality["score"],
            "routed_model": quality["routed_model"],
            "routed_cost": quality["routed_cost"],
            "expensive_cost": quality["expensive_cost"],
            "degraded": quality["score"] < 0.8  # significant degradation
        })
    
    degraded_count = sum(1 for r in results if r["degraded"])
    degradation_rate = degraded_count / len(results)
    
    # Cost savings
    total_routed_cost = sum(r["routed_cost"] for r in results)
    total_expensive_cost = sum(r["expensive_cost"] for r in results)
    savings_pct = (1 - total_routed_cost / total_expensive_cost) * 100
    
    return {
        "degradation_rate": degradation_rate,
        "cost_savings_pct": savings_pct,
        "avg_quality_score": sum(r["quality_score"] for r in results) / len(results),
        "per_tier_breakdown": {
            tier: {
                "avg_score": ...,
                "degradation_rate": ...,
            } for tier in ["simple", "medium", "complex"]
        }
    }
```

### Rohan's Mid-Build Interruption

> *Rohan appears while you're testing the router.*

"You claim 42% cost savings with only 3% quality degradation. Show me the 3%. Which queries degraded? Were they all in the 'complex' tier? If you're degrading quality on complex queries, that's where the client-facing work happens — you can't save money on the wrong tier."

> *He waits. This is a test. Do you have per-tier breakdown?*

### Priya's Requirement Change

> *Priya: "Great news — the product team wants to use your gateway for THEIR apps too. But they need it as a real API. Not a Python script. A running service with authentication, rate limiting per team, and a dashboard showing cost per team."*

Add: FastAPI service, API key auth per team, rate limiter per team, `/dashboard/metrics` endpoint returning cost breakdown by team and model.

---

## Phase 4: Review (Rohan)

> *You submit: FastAPI gateway service, LiteLLM routing, complexity classifier (hybrid rule + LLM), quality validation report, cost dashboard.*

### Decision Documentation Required

1. Your routing strategy and why — cost-based? latency-based? hybrid?
2. How you classify complexity — rule heuristic vs LLM, accuracy, latency impact
3. Your quality validation methodology — how you measured degradation, what you found per tier
4. The cost breakdown — what you saved, what model combinations you use, edge cases

### Rohan's Review

| # | Check | Status | Notes |
|---|---|---|---|
| 1 | **Does it work?** | ✅ PASS | Unified API works. Apps don't know which model serves them. |
| 2 | **Edge cases?** | ✅ PASS | Model down? Falls back. Rate limited? Waits + retries. Unsupported features? Graceful degradation. |
| 3 | **Cost-aware?** | ✅ PASS | $8,400 → $4,872 projected monthly. 42% savings. Excellent. |
| 4 | **Observable?** | ✅ PASS | Per-request telemetry. Per-team dashboard. Full cost breakdown. |
| 5 | **Right approach?** | ✅ PASS | Hybrid classifier (rules + LLM) is the production pattern. LiteLLM was the right choice for routing. |
| 6 | **Decisions justified?** | ✅ PASS | You showed per-tier degradation data. Complex queries degrade 8% — you documented it honestly and recommended manual review for those. |
| 7 | **Measurable quality?** | ✅ PASS | 42% cost savings, 3.2% overall degradation. 97% quality preserved for 42% less money. |

### Verdict: ✅ APPROVED

*Rohan closes your report.*

"This passes. And I mean genuinely passes — not 'good for a junior.' Your cost savings are real, your quality monitoring is honest (you didn't hide the 8% complex-tier degradation), and your architecture is production-grade.

**One thing I want you to take forward:** Your hybrid classifier (rules → LLM on low confidence) is the right pattern. Almost every AI system benefits from this — fast path for common cases, expensive analysis for edge cases. Remember this pattern. You'll use it again.

**Tier 1 + Tier 2 complete.** You've built:
- Structured output pipelines ✓
- RAG from scratch ✓
- Hybrid search ✓
- Multi-tenant RAG with conversation memory ✓
- Batch extraction at scale ✓
- Model routing for cost optimization ✓

You're not a junior anymore. Take a day off. Then Tier 3 starts."

---

## Phase 5: Debrief (Maya)

> *After APPROVED. Maya's leaning against the doorframe.*

**Maya:** "Seven projects. Two tiers. Let me tell you what just happened.

In Tier 1, you learned to **call the API.** In Tier 2, you learned to **architect the system.** That's the jump. You went from 'how do I send a message to an LLM' to 'which LLM should I send this message to, at what cost, with what fallback, and how do I prove it's good enough?'

### The Skills You Now Have

**RAG (70% of production AI)**
You can chunk, embed, search (hybrid), generate, and evaluate. You know that chunking strategy matters more than model choice. You know that hybrid search beats vector-only. You built it manually before touching LangChain — so when LangChain breaks, you can debug it.

**Production Engineering**
You handle API errors. You handle rate limits. You retry with backoff. You track costs. You add observability. These are the skills that separate engineers who ship from engineers who demo.

**Economic Thinking**
You now think in cost per query. You know when to route to cheap models and when to pay for quality. You treat the LLM as a compute resource with a budget, not a magic box.

### What Tier 3 Changes

"Tier 3 is where the difficulty spikes. Here's what's different:

| Dimension | Tier 2 | Tier 3 |
|---|---|---|
| **Guidance** | Goal + hints | Minimal — you figure it out |
| **Constraints** | Cost + accuracy | Latency + cost + accuracy + security |
| **Building** | Manual, then framework | Framework from the start (you've earned it) |
| **Projects** | 4 projects | 5 projects, harder each time |
| **Evaluation** | Manual + basic metrics | Automated eval suites |
| **Observability** | Logging | Full tracing stack (Langfuse) |

**What you'll build in Tier 3:**
- **Project 8:** A real-time sales intelligence agent (<3s latency, tool use, personalization)
- **Project 9:** An advanced financial RAG system (agentic retrieval, vision for tables, anti-hallucination)
- **Project 10:** A code review agent (GitHub integration, AST parsing, false positive minimization)
- **Project 11:** A natural language to SQL agent (safe query execution, schema grounding)
- **Project 12:** A cost-aware model router with caching and fallbacks (your Project 7 taken to production)

> *Maya holds out her hand. "You're not a junior AI engineer anymore. You're an AI Engineer. Full stop. Tier 3 proves it."*
