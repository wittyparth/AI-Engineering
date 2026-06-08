# 06 — Production Observability: Seeing Your AI System in the Wild

## 🎯 Purpose & Goals

> 🛑 STOP. Before we talk about observability tools, you need to understand why most monitoring setups give you FALSE confidence.

Everything up to this point has been PRE-PRODUCTION evaluation:
- You evaluated your system against curated datasets (File 04)
- You judged outputs with LLM judges (File 02)
- You ran CI/CD pipelines before deployment (File 05)

But here's the hard truth: **pre-production evaluation tells you what your system CAN do. It doesn't tell you what it DOES do.**

Production observability is the difference between:
- "Our eval scores are 94%" → (measured in a controlled environment)
- "Our users are getting 78% useful answers, and we can show you exactly which queries fail" → (measured in production)

And the gap between these two numbers is where the real problems live.

---

### 🤔 Discovery Question 1: The Black Box Deployment

Your team deploys a new version of your RAG system. Pre-production eval: 94% faithfulness, 91% relevance. The PM is happy. You ship.

Three days later, support tickets spike. Users are getting wrong answers. You have NO data about what's happening in production. The system is a black box.

You spend 2 days adding logging, re-deploy, and wait. Now you can see that the system is retrieving DIFFERENT chunks than it did in evaluation — because production queries don't match your eval dataset distribution.

**🤔 Before reading on, answer these:**

1. Why does retrieval DIFFER between eval and production, even when using the SAME retrieval code? Give at least 3 concrete reasons related to the queries themselves, not the code.

2. Your pre-production eval used 500 curated queries covering 5 categories. But production traffic is: 40% "billing" (you have 20% in eval), 35% "technical" (you have 30%), 15% "account" (you have 30%), 10% "security" (you have 20%). Your eval dataset over-represents account queries and under-represents billing queries. The system performs WORSE on billing queries. Your eval score was inflated because you tested fewer billing queries. The question: **How would you have DETECTED this distribution mismatch WITHOUT production observability?** What signal would have told you your eval dataset was misaligned?

3. You add production tracing. Now you can see every query, every retrieval, every LLM call, every response. That's 10,000 queries/day × 5 LLM calls per query = 50,000 traces per day. Each trace takes ~2KB of storage. That's 100MB/day — 3GB/month. Storage is cheap. But the TRACE DATA itself contains user queries, PII, and business-sensitive information. **How do you balance observability with privacy?** What data do you redact? How do you ensure trace data doesn't create a security vulnerability?

---

### 🤔 Discovery Question 2: The Metric Explosion

Your observability platform captures:

```
Per query: latency, token count, model used, cost, retrieval latency, 
           generation latency, judge score, user feedback (if any)
           
Aggregated: p50/p95/p99 latency, avg cost, error rate, token throughput,
            cache hit rate, retrieval recall (estimated), user satisfaction proxy
```

Your dashboard has 47 metrics. You check it daily. You look at the "overall health" widget, which is green. Life is good.

Except one day, the VP of Product asks: "How's the AI system doing?" and you can't answer concisely. You show her 47 metrics. She walks away without a clear answer.

**🤔 Before reading on, answer these:**

1. **The "one number" problem:** You need to give the VP ONE number that captures system health. It can't be an average of all 47 metrics (that hides problems). It can't be a single metric (that misses context). Design a "System Health Score" (0-100) that combines the most important signals. What goes into it? How is it weighted? How do you ensure a good score genuinely means a healthy system?

2. Your SRE (Site Reliability Engineer) needs DIFFERENT information than your PM. The SRE needs p99 latency, error rates, and cost. The PM needs user satisfaction, feature adoption, and response quality. **Design TWO dashboards** — one for SREs, one for product managers. Each should have NO MORE than 5 metrics. Which 5 for each? Defend your choices.

3. **The alert threshold problem:** You set an alert for "p95 latency > 5s." It fires 2-3 times per day. Your team investigates each time. It's always the same cause: one slow model (Claude 3.5 Sonnet) on specific query types. You can't fix it (model latency is an API issue, not your code). The alerts are noise. But if you DISABLE the alert, you won't notice if the latency issue SPREADS to other models. How do you design alerts that catch REAL incidents without desensitizing your team to NOISE?

---

### 🤔 Discovery Question 3: The "It Worked in Eval" Surprise

Your pre-production eval showed 96% faithfulness. Two weeks into production, you run a production trace analysis on 1,000 random queries. Production faithfulness: 84%.

The gap is 12 points. Something that happens in production is NOT happening in eval.

**🤔 Before reading on, list all the possible reasons for the gap:**

1. **Data distribution shift** — production queries are different from eval queries
2. **Context drift** — the documents being retrieved have changed
3. **Latency pressure** — production has timeouts, eval doesn't
4. **Caching effects** — production uses cached responses, eval doesn't
5. **User behavior** — users ask follow-ups, abandon mid-conversation
6. **Adversarial inputs** — production gets spam, abuse, edge cases

For EACH reason above:
a) How would you DETECT it from production traces?
b) How would you FIX it (or mitigate it)?
c) How would you UPDATE your eval dataset to catch it pre-deployment next time?

**The hardest one:** Reason #6 (adversarial inputs) is almost NEVER in eval datasets. Eval datasets contain nice, clean queries. Production contains people trying to break your system. How do you build an eval dataset that includes adversarial inputs WITHOUT biasing your eval toward worst-case scenarios?

---

> **Observability is not just dashboards and alerts. It's the FEEDBACK LOOP that closes the gap between what you think your system does and what it actually does.**

By the end of this module, you will:
- Implement production tracing with Langfuse or OpenTelemetry
- Design dashboards for different stakeholders (SRE, PM, engineer)
- Build alerting that catches real incidents without noise
- Set up cost tracking and budget alerts
- Implement user feedback collection that isn't biased
- Design a production-to-eval feedback loop that improves your eval datasets

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Discovery Questions (above) | 25 min |
| Observability vs. Monitoring | 15 min |
| Traces, Spans, and Events | 20 min |
| Langfuse Architecture | 20 min |
| Dashboard Design Principles | 20 min |
| Alert Design & Noise Reduction | 20 min |
| Code: Langfuse Tracing Integration | 35 min |
| Code: Custom Span Attributes | 20 min |
| Code: Cost Tracking Dashboard | 25 min |
| Code: User Feedback Collection | 20 min |
| Code: Production-to-Eval Feedback Loop | 25 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 10 min |
| **Total** | **~5.5 hours** |

---

## 📖 Concept Deep-Dive

### 1. Observability vs. Monitoring

These terms are often used interchangeably. They're NOT the same.

| Monitoring | Observability |
|-----------|---------------|
| "Is the system healthy?" | "WHY is the system unhealthy?" |
| Pre-defined metrics and thresholds | Ad-hoc investigation capability |
| Answers known questions | Enables discovering UNKNOWN unknowns |
| Tells you SOMETHING is wrong | Tells you WHAT is wrong |
| Aggregated data | Individual request-level data |

**Analogy:** Monitoring is a car's "check engine" light. It tells you something is wrong. Observability is the diagnostic tool that reads the engine's error codes — it tells you WHICH sensor failed, when, and under what conditions.

For AI systems, monitoring checks: "Is the error rate below 1%?" Observability enables: "Show me all queries from the last hour where the retrieval returned fewer than 3 chunks AND the user left negative feedback."

### 2. The Three Pillars of Observability

**Traces:** The path of a single request through your system.
- For a RAG system: User query → retrieval → context assembly → LLM call → response
- For an agent: Task → decompose → search → analyze → cite → synthesize
- Each trace has SPANS (individual operations) with timing, metadata, and relationships

**Logs:** Discrete events with timestamps.
- "Retrieval returned 5 chunks from index 'products_v3'"
- "LLM call to gpt-4o-mini completed in 1,234ms, 456 tokens"
- "User feedback received: thumbs_down on response #abc123"

**Metrics:** Aggregated numerical measurements over time.
- Average latency, p99 latency, error rate, token throughput, cost per query

For AI systems, you need ALL THREE. Traces tell you WHAT happened to a specific query. Logs give you DETAIL about that query. Metrics tell you whether TODAY is different from YESTERDAY.

---

## 💻 Code Examples

### Example 1: Langfuse Tracing for a RAG System

This shows how to instrument a RAG system with production tracing.

```python
"""
langfuse_tracing.py

Instruments a RAG system with Langfuse tracing.
Every query gets a trace with spans for retrieval, context, and generation.

DESIGN DECISION: What level of detail to trace?
- Option A: Trace EVERYTHING (every chunk retrieval, every model call)
  → Maximum debug capability, maximum storage cost
- Option B: Trace only the top-level (query → response)
  → Minimal overhead, can't debug internals
- Option C: Trace fully but SAMPLE (trace 10% of queries fully, 
  rest get top-level only)
  → Balance of cost and debuggability

We implement Option C. You need to decide the sampling rate.
"""

from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context
import os
import time
import random


class ObservableRAG:
    """
    RAG system with Langfuse tracing.
    
    Traces every query with detailed spans for:
    - Query receipt
    - Retrieval (with chunk count and scores)
    - Context assembly (with token usage)
    - LLM generation (with model, latency, tokens)
    - Response delivery
    """
    
    def __init__(
        self,
        trace_sample_rate: float = 0.1,  # Trace 10% of queries in FULL detail
    ):
        self.langfuse = Langfuse(
            public_key=os.getenv("LANGFUSE_PUBLIC_KEY"),
            secret_key=os.getenv("LANGFUSE_SECRET_KEY"),
            host=os.getenv("LANGFUSE_HOST", "https://cloud.langfuse.com"),
        )
        self.trace_sample_rate = trace_sample_rate
    
    @observe(name="rag_query")
    async def answer(self, query: str, user_id: str = "") -> str:
        """
        Answer a user query with full observability.
        
        The @observe decorator automatically captures:
        - Input/output
        - Duration
        - Token counts (if using Langfuse's LLM call tracking)
        - Errors
        """
        # Determine sampling level
        full_trace = random.random() < self.trace_sample_rate
        
        # Add user/session context
        langfuse_context.update_current_trace(
            user_id=user_id,
            session_id=f"session_{user_id}",
            metadata={
                "query_length": len(query),
                "has_attachments": "@" in query,
                "full_trace": full_trace,
            },
        )
        
        try:
            # SPAN 1: Retrieval
            retrieval_result = await self._retrieve(query, full_trace)
            
            # SPAN 2: Context assembly
            context = self._assemble_context(retrieval_result, full_trace)
            
            # SPAN 3: LLM generation
            answer = await self._generate(query, context, full_trace)
            
            # Record score (if we have a judge)
            score = await self._score_response(query, answer, full_trace)
            
            return answer
            
        except Exception as e:
            langfuse_context.update_current_trace(
                metadata={"error": str(e), "error_type": type(e).__name__},
            )
            raise
    
    @observe(name="retrieval")
    async def _retrieve(self, query: str, full_trace: bool) -> list:
        """
        Retrieve relevant chunks.
        
        If full_trace is True, log each chunk individually.
        Otherwise, log only aggregate stats.
        """
        chunks = []  # Your retrieval logic here
        
        # Simulate retrieval
        await asyncio.sleep(0.1)
        chunks = [
            {"id": "chunk_1", "score": 0.89, "content": "..."},
            {"id": "chunk_2", "score": 0.76, "content": "..."},
            {"id": "chunk_3", "score": 0.62, "content": "..."},
        ]
        
        if full_trace:
            # Log each chunk as a separate event
            for chunk in chunks:
                langfuse_context.update_current_span(
                    output={"chunk_id": chunk["id"], "score": chunk["score"]},
                )
        else:
            # Log aggregate only
            langfuse_context.update_current_span(
                metadata={
                    "num_chunks": len(chunks),
                    "avg_score": sum(c["score"] for c in chunks) / len(chunks),
                    "top_score": chunks[0]["score"],
                },
            )
        
        return chunks
    
    @observe(name="context_assembly")
    def _assemble_context(self, chunks: list, full_trace: bool) -> str:
        """Assemble retrieved chunks into a context string."""
        context = "\n\n".join(c["content"] for c in chunks)
        
        total_tokens = len(context.split())  # Rough token count
        langfuse_context.update_current_span(
            usage={"input": total_tokens},
            metadata={
                "context_length_chars": len(context),
                "context_length_tokens": total_tokens,
                "num_chunks": len(chunks),
            },
        )
        
        return context
    
    @observe(name="llm_generation")
    async def _generate(self, query: str, context: str, full_trace: bool) -> str:
        """Generate answer using LLM."""
        # In real implementation, this calls OpenAI/Anthropic
        # Langfuse auto-tracks token usage if you use their integrations
        
        answer = "Here is the answer based on the provided context..."
        
        langfuse_context.update_current_span(
            model="gpt-4o-mini",
            model_parameters={"temperature": 0.1, "max_tokens": 500},
            metadata={
                "prompt_template": "rag_v2",
                "context_first_200": context[:200],
            },
        )
        
        return answer


# === USAGE ===
rag = ObservableRAG(trace_sample_rate=0.1)
# answer = await rag.answer("What is the return policy?", user_id="user_123")
```

**🤔 The sampling tradeoff:** You set `trace_sample_rate = 0.1`, meaning only 10% of queries get full detail. The other 90% get only top-level timing and pass/fail.

What happens when a bug affects ONLY the 90% of queries that aren't traced fully? You can see the error rate go up (from aggregate metrics), but you can't see WHICH retrieval step failed (no detail spans). You know SOMETHING is wrong but not WHAT.

How would you design an ADAPTIVE sampling strategy that:
- Traces only 5% normally (low cost)
- Automatically increases to 50% when error rate spikes (high detail during incidents)
- Decreases back to 5% when the incident is resolved

This is called "dynamic sampling" and it's the production standard for observability.

---

### Example 2: Cost Tracking Dashboard

```python
"""
cost_tracker.py

Tracks and alerts on LLM API costs.
Because cost surprises are the #1 reason observability gets shut down.
"""

from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Optional
from collections import defaultdict
import json


@dataclass
class CostRecord:
    """A single LLM call's cost."""
    timestamp: str
    model: str
    provider: str
    prompt_tokens: int
    completion_tokens: int
    cost_usd: float
    query_id: str
    system_component: str  # "retrieval" | "generation" | "judge" | "agent"


class CostTracker:
    """
    Tracks LLM API costs with per-component breakdown.
    
    Why per-component?
    - If total cost spikes, you need to know WHICH component caused it
    - Is retrieval more expensive? (more chunks, bigger context)
    - Is generation more expensive? (longer answers)
    - Is evaluation more expensive? (more LLM judges)
    """
    
    def __init__(self, daily_budget_usd: float = 100.0):
        self.records: list[CostRecord] = []
        self.daily_budget = daily_budget_usd
    
    def record_call(
        self,
        model: str,
        prompt_tokens: int,
        completion_tokens: int,
        query_id: str,
        system_component: str = "generation",
    ):
        """Record an LLM API call with its cost."""
        cost = self._compute_cost(model, prompt_tokens, completion_tokens)
        
        self.records.append(CostRecord(
            timestamp=datetime.now().isoformat(),
            model=model,
            provider=model.split("-")[0] if "-" in model else model,
            prompt_tokens=prompt_tokens,
            completion_tokens=completion_tokens,
            cost_usd=cost,
            query_id=query_id,
            system_component=system_component,
        ))
    
    def _compute_cost(self, model: str, prompt_tokens: int, completion_tokens: int) -> float:
        """Compute cost based on model pricing."""
        # Simplified pricing (update with actual rates)
        pricing = {
            "gpt-4o": {"prompt": 0.00001, "completion": 0.00003},
            "gpt-4o-mini": {"prompt": 0.0000015, "completion": 0.000006},
            "gpt-4": {"prompt": 0.00003, "completion": 0.00006},
            "claude-3-sonnet": {"prompt": 0.000003, "completion": 0.000015},
            "claude-3-haiku": {"prompt": 0.0000025, "completion": 0.0000125},
        }
        
        model_key = model
        if model_key not in pricing:
            # Fallback: use gpt-4o-mini pricing
            model_key = "gpt-4o-mini"
        
        p = pricing[model_key]
        return (prompt_tokens * p["prompt"]) + (completion_tokens * p["completion"])
    
    def daily_summary(self, date: Optional[str] = None) -> dict:
        """Get cost summary for a specific day."""
        if date is None:
            date = datetime.now().strftime("%Y-%m-%d")
        
        day_records = [
            r for r in self.records
            if r.timestamp.startswith(date)
        ]
        
        total = sum(r.cost_usd for r in day_records)
        by_model = defaultdict(float)
        by_component = defaultdict(float)
        by_query = defaultdict(float)
        
        for r in day_records:
            by_model[r.model] += r.cost_usd
            by_component[r.system_component] += r.cost_usd
            by_query[r.query_id] += r.cost_usd
        
        # Find most expensive queries
        top_queries = sorted(by_query.items(), key=lambda x: x[1], reverse=True)[:5]
        
        return {
            "date": date,
            "total_cost": total,
            "budget": self.daily_budget,
            "budget_remaining": self.daily_budget - total,
            "budget_pct": (total / self.daily_budget) * 100,
            "by_model": dict(by_model),
            "by_component": dict(by_component),
            "total_queries": len(set(r.query_id for r in day_records)),
            "most_expensive_queries": top_queries,
        }
    
    def alert_if_over_budget(self) -> Optional[str]:
        """Check if we're on track to exceed daily budget."""
        today = self.daily_summary()
        
        # If we've used >80% of budget by noon, alert
        if today["budget_pct"] > 80:
            hour = datetime.now().hour
            expected_usage_pct = (hour / 24) * 100
            
            if today["budget_pct"] > expected_usage_pct * 1.5:
                return (
                    f"⚠️ Cost alert: {today['budget_pct']:.0f}% of daily budget used "
                    f"({hour}:00, expected ~{expected_usage_pct:.0f}%). "
                    f"On track to exceed budget."
                )
        
        return None


# === USAGE ===
if __name__ == "__main__":
    tracker = CostTracker(daily_budget_usd=50.0)
    
    # Simulate a day of usage
    tracker.record_call("gpt-4o-mini", prompt_tokens=500, completion_tokens=200,
                        query_id="q1", system_component="generation")
    tracker.record_call("gpt-4o", prompt_tokens=800, completion_tokens=300,
                        query_id="q1", system_component="judge")
    tracker.record_call("gpt-4o-mini", prompt_tokens=1500, completion_tokens=600,
                        query_id="q2", system_component="generation")
    
    summary = tracker.daily_summary()
    print(f"Daily cost: ${summary['total_cost']:.4f}")
    print(f"By component: {summary['by_component']}")
    print(f"Most expensive queries: {summary['most_expensive_queries']}")
```

**🤔 Cost allocation in multi-agent systems:** Your multi-agent research system (Phase 6 capstone) uses 4 agents per query. Each agent calls LLMs. The total cost per query is distributed across supervisor, search worker, analysis worker, and citation worker.

How do you ATTRIBUTE cost to each agent? And more importantly: how do you detect that ONE agent is driving cost increases? If the search worker's cost tripled after a prompt change, you need to know it was the SEARCH worker, not the others.

Design a cost attribution system for multi-agent systems. What span attributes would you add to make cost analysis possible?

---

### Example 3: Production-to-Eval Feedback Loop

The most important observability pattern: using production data to IMPROVE your evaluation.

```python
"""
feedback_loop.py

Closes the gap between production and evaluation by:
1. Sampling production queries that had issues
2. Adding them to the eval dataset
3. Updating the eval dataset distribution to match production
"""

from dataclasses import dataclass
from typing import Optional
from collections import Counter
import random
import json


@dataclass
class ProductionObservation:
    """A single production query with observability data."""
    query: str
    response: str
    latency_ms: float
    cost_usd: float
    user_feedback: Optional[str]  # "positive" | "negative" | None
    retrieval_chunks: int
    response_length: int
    trace_id: str


class EvalDatasetUpdater:
    """
    Updates the evaluation dataset based on production observations.
    
    Strategy:
    1. Sample production queries with NEGATIVE user feedback (always add)
    2. Sample production queries with UNUSUAL characteristics (add)
    3. Sample random production queries (maintain distribution match)
    4. Remove stale eval entries that no longer match production
    """
    
    def __init__(self, max_dataset_size: int = 1000):
        self.max_size = max_dataset_size
        self.eval_entries: list[dict] = []
    
    def select_candidates_for_eval(
        self,
        observations: list[ProductionObservation],
        current_eval_distribution: dict[str, float],
        production_distribution: dict[str, float],
        samples_needed: int = 50,
    ) -> list[ProductionObservation]:
        """
        Select production observations to add to eval dataset.
        
        Three selection criteria:
        1. Negative feedback (top priority)
        2. Fill distribution gaps (match production)
        3. Random outliers (long latency, high cost, unusual patterns)
        """
        candidates = []
        
        # Priority 1: All negative feedback queries
        negative = [o for o in observations if o.user_feedback == "negative"]
        candidates.extend(negative)
        
        # Priority 2: Distribution gap fillers
        # Find categories that are UNDER-represented in eval vs production
        gaps = {}
        all_categories = set(production_distribution.keys()) | set(current_eval_distribution.keys())
        for cat in all_categories:
            prod_pct = production_distribution.get(cat, 0)
            eval_pct = current_eval_distribution.get(cat, 0)
            gap = prod_pct - eval_pct
            if gap > 0.05:  # More than 5% under-represented
                gaps[cat] = gap
        
        # Sample from under-represented categories
        for cat, gap_pct in gaps.items():
            cat_observations = [
                o for o in observations
                if self._categorize_query(o.query) == cat
            ]
            # How many to add to close the gap
            needed = int(gap_pct * self.max_size)
            sample = random.sample(
                cat_observations,
                min(needed, len(cat_observations)),
            )
            candidates.extend(sample)
        
        # Priority 3: Outliers (high cost, high latency, unusual patterns)
        latency_threshold = sorted(
            [o.latency_ms for o in observations]
        )[-int(len(observations) * 0.05)]  # Top 5% latency
        
        cost_threshold = sorted(
            [o.cost_usd for o in observations]
        )[-int(len(observations) * 0.05)]  # Top 5% cost
        
        outliers = [
            o for o in observations
            if o.latency_ms >= latency_threshold or o.cost_usd >= cost_threshold
        ]
        # Don't add duplicates (already selected via other criteria)
        existing_ids = {c.trace_id for c in candidates}
        new_outliers = [o for o in outliers if o.trace_id not in existing_ids]
        candidates.extend(random.sample(
            new_outliers,
            min(samples_needed // 3, len(new_outliers)),
        ))
        
        return candidates
    
    def _categorize_query(self, query: str) -> str:
        """Categorize a query. In production, use your classifier."""
        query_lower = query.lower()
        if any(w in query_lower for w in ["refund", "charge", "bill", "payment"]):
            return "billing"
        elif any(w in query_lower for w in ["error", "bug", "crash", "broken"]):
            return "technical"
        elif any(w in query_lower for w in ["login", "password", "account"]):
            return "account"
        else:
            return "other"
    
    def refresh_eval_dataset(
        self,
        current_eval: list[dict],
        production_observations: list[ProductionObservation],
        production_distribution: dict[str, float],
    ) -> list[dict]:
        """
        Refresh the eval dataset with production data.
        
        1. Add recent production issues (negative feedback, outliers)
        2. Remove stale entries (queries about deprecated features)
        3. Rebalance categories to match production distribution
        """
        # Analyze current eval distribution
        current_dist = Counter(
            self._categorize_query(e["query"])
            for e in current_eval
        )
        total = sum(current_dist.values()) or 1
        current_pct = {k: v/total for k, v in current_dist.items()}
        
        # Select candidates
        candidates = self.select_candidates_for_eval(
            production_observations,
            current_pct,
            production_distribution,
        )
        
        # Convert to eval entries
        new_entries = [
            {
                "query": o.query,
                "expected_response": None,  # Needs human annotation!
                "source": "production",
                "added_reason": "negative_feedback" if o.user_feedback == "negative" else "distribution_gap",
                "trace_id": o.trace_id,
            }
            for o in candidates
        ]
        
        # Combine and trim
        combined = current_eval + new_entries
        if len(combined) > self.max_size:
            # Remove oldest entries first
            combined = combined[-self.max_size:]
        
        return combined


# === USAGE ===
if __name__ == "__main__":
    updater = EvalDatasetUpdater(max_dataset_size=1000)
    
    # Current eval distribution
    current_dist = {
        "billing": 0.30,
        "technical": 0.30,
        "account": 0.25,
        "other": 0.15,
    }
    
    # Production distribution (different!)
    production_dist = {
        "billing": 0.45,  # More billing than eval!
        "technical": 0.25,
        "account": 0.20,
        "other": 0.10,
    }
    
    # Production observations (sampled from traces)
    observations = [
        ProductionObservation(
            query="Why was I charged twice?",
            response="Let me look into that...",
            latency_ms=1200,
            cost_usd=0.05,
            user_feedback="negative",
            retrieval_chunks=3,
            response_length=150,
            trace_id="t1",
        ),
        # ... more observations
    ]
    
    # The evaluator would flag billing as under-represented
    # and suggest adding billing queries from production
```

**🤔 The annotation bottleneck:** The feedback loop above adds production queries to the eval dataset. But they need HUMAN ANNOTATION before they can be used as eval cases (you need the "expected answer"). Each annotation costs time and money.

How do you prioritize which production queries to annotate FIRST? The system adds 50 new candidates per day, but you can only afford 10 annotations per day. What's your selection criteria? How do you ensure the 10 you annotate give you the MOST signal about system quality?

---

## ✅ Good Output Examples

### What Good Observability Looks Like

```
=== Production Health Dashboard (last 24h) ===

TRAFFIC:             10,234 queries (+3% vs yesterday)
ACTIVE USERS:        3,456

LATENCY:
  p50: 1.2s (target: <2s) ✅
  p95: 3.8s (target: <5s) ✅ 
  p99: 7.2s (target: <10s) ⚠️ (tail needs investigation)

COST:
  Total: $42.15 (budget: $50/day) ✅
  Avg per query: $0.004
  By component: generation $28, retrieval $10, judge $4.15

QUALITY (sampled, n=200):
  Positive feedback: 78%
  Negative feedback: 8%
  No feedback: 14%
  Estimated faithfulness: 0.91 (from LLM judge on 10% sample)

ALERTS:
  No active alerts ✅

TOP ISSUES (from negative feedback):
  1. "refund status" queries — 23% of negatives
  2. Multi-language queries — 12% of negatives
  3. Technical jargon queries — 8% of negatives
```

### What Poor Observability Looks Like

```
=== System Health Dashboard ===

Error Rate: 1.2% ← (but what KIND of errors? no breakdown)
Avg Latency: 2.1s ← (p50 or p99? unclear)
Cost: Unknown ← (cost tracking not implemented)
User Satisfaction: 4.2/5 ← (from 50 survey responses, selection bias)

No traces available for debugging.
No alerting configured.
No production-to-eval feedback loop.
```

---

## ❌ Antipatterns & Failure Modes

### ❌ Antipattern 1: Dashboard Overload

You build a dashboard with 50+ metrics. It's comprehensive. No one looks at it because it's overwhelming.

**🔧 Fix:** Build THREE targeted dashboards:
- SRE dashboard (5 metrics: error rate, p50/p95/p99 latency, cost)
- PM dashboard (5 metrics: user satisfaction, feature adoption, query volume, top failure modes, trend vs last week)
- Debug dashboard (50 metrics, only looked at during incidents)

### ❌ Antipattern 2: Alert Fatigue

You set alerts for every metric. 50 alerts fire per day. Your team ignores all of them.

**🔧 Fix:** Alert only on ACTIONABLE signals:
- Alert when p95 latency exceeds threshold for 5+ consecutive minutes (not 1 spike)
- Alert when error rate doubles compared to same time yesterday (not static threshold)
- Alert when cost exceeds 120% of budget projection (not when it crosses budget)
- Don't alert on things the team can't fix (model latency, third-party API issues)

### ❌ Antipattern 3: PII in Traces

Your trace data contains full user queries, including credit card numbers, addresses, and passwords. Storing this creates a security liability.

**🔧 Fix:** Implement PII redaction at the trace COLLECTION point, not at query time. Use regex patterns or an LLM to detect and redact: emails, phone numbers, credit cards, SSNs, API keys. Test your redaction with adversarial inputs.

### ❌ Antipattern 4: No Sampling Strategy

You trace EVERY query in full detail. Storage costs explode. Querying traces becomes slow. The team stops looking at traces because there's too much data.

**🔧 Fix:** Implement intelligent sampling:
- 100% trace for errors
- 10% trace for successful queries (random sample, rotated daily)
- 50% trace for queries from VIP users
- 1% trace for routine, low-value queries

### ❌ Antipattern 5: The One-Week Retention

You keep production traces for only 1 week (to save storage). A regression that started 10 days ago — you have no data to diagnose it.

**🔧 Fix:** Tiered retention:
- Full traces: 30 days
- Aggregated metrics: 1 year
- Sampled traces (1%): 1 year
- Error traces: 1 year

---

## 🧪 Drills & Challenges

### Drill 1: Design the Tracing Schema (25 min)

Your multi-agent research system (Phase 6 capstone) needs production observability. Design the tracing schema.

For each SPAN in the system, define:
1. Span name
2. Parent span
3. What attributes to capture
4. What "error" means for this span
5. Cost attribution (how to assign cost to this span)

Spans to cover:
- `supervisor.decompose` — LLM call to break down the question
- `search_worker.execute` — Web search tool call
- `analysis_worker.synthesize` — LLM call to analyze findings
- `citation_worker.verify` — Source verification tool call
- `supervisor.synthesize` — Final answer generation

**🤔 Extra:** Your tracing shows that 80% of total latency is in `search_worker.execute` (web search API calls). But you can't make the web search faster (it's an external API). Should you trace something INSIDE that span to understand WHY it's slow? What sub-spans would you add?

---

### Drill 2: The "Drift Detection" Dashboard (30 min)

Design a dashboard (can be text/ASCII, doesn't need real code) that answers:

1. **Is our eval dataset still representative of production?**
   - Show: production query distribution vs eval query distribution
   - Show: production latency distribution vs eval latency distribution
   - Flag: categories where gap > 10%

2. **Is our system quality changing over time?**
   - Show: weekly average faithfulness score (from sampled production LLM judge)
   - Show: user feedback positive rate over time
   - Flag: 3-week trend (up, flat, down)

3. **Is our cost predictable?**
   - Show: daily cost vs budget
   - Show: cost per query trend
   - Flag: day-over-day increase > 20%

**For each widget, write:**
- The query to generate the data (pseudocode or SQL-like)
- The visualization (chart type, axes, color coding)
- The alert threshold (when does this widget turn red?)
- Who should look at this widget (SRE? PM? Engineer?)

---

### Drill 3: The Production Incident Postmortem (25 min)

You receive this alert at 2:00 PM:

```
🚨 ALERT: User negative feedback rate spiked from 8% to 23% in the last hour.
🚨 ALERT: Cost tracking shows $120 spent in last hour (budget: $5/hour).
🚨 ALERT: Average response length increased 3x (from 150 to 450 tokens).
```

All three alerts fired at the same time. They're related.

**Investigate using observability data:**

1. What's the MOST LIKELY root cause given these three signals? (Longer responses + more expensive + users hate it = ?)

2. You check the traces and find that 90% of the affected queries used `gpt-4o` instead of `gpt-4o-mini`. Someone changed the model routing. What else would you check?

3. The model routing change was in a configuration file, not in code — it slipped through the CI/CD eval pipeline because the pipeline doesn't test configuration changes. How do you fix the pipeline to catch config changes?

4. You revert the model routing. Negative feedback drops back to 8% within 2 hours. Write a postmortem (3-5 bullet points) that covers: what happened, why it wasn't caught, and what systemic fix prevents recurrence.

---

### Drill 4: The Sampling Strategy Decision (20 min)

Your AI system handles 100,000 queries/day. Full tracing would cost $500/month in storage and slow down the system by 2% (tracing overhead).

You need to decide on a sampling strategy:

| Strategy | Cost/Month | Debug Capability | Risk |
|----------|-----------|-----------------|------|
| A: Trace all | $500 | Full visibility | 2% overhead, storage cost |
| B: Trace 10% | $50 | Partial visibility | May miss rare issues |
| C: Trace errors only | $10 | Error visibility only | Miss silent degradation |
| D: Adaptive (10% normal, 100% during incidents) | $100 | Good balance | Complex to implement |

**Which do you choose? Defend your answer with specific reasoning about:**
- What kinds of issues would each strategy miss?
- How quickly could you debug a production incident with each strategy?
- How would your choice differ for a safety-critical system vs. a recommendation system?

---

### Drill 5: The User Feedback Pipeline (25 min)

You need a user feedback system that collects satisfaction signals WITHOUT introducing bias.

**Problem:** You could add a "thumbs up / thumbs down" button. But:
- Only 5% of users click it (selection bias)
- Users who click are more likely to be unhappy (negativity bias)
- The button is visual clutter on mobile (product team hates it)

**Design an ALTERNATIVE feedback collection system that:**
1. Collects signal from >20% of queries (not 5%)
2. Doesn't rely on explicit user actions (no buttons)
3. Doesn't violate privacy
4. Is available within 1 hour of the query (not delayed surveys)

**Think about IMPLICIT signals:**
- Does the user ask a FOLLOW-UP question? (They might not be satisfied)
- Does the user REPEAT their query with different wording? (They didn't get what they wanted)
- Does the user CLICK A LINK in the response? (They engaged = possibly helped)
- Does the user ABANDON the conversation? (No response for 5+ minutes = ?)
- Does the user ESCALATE to a human? (Clear signal of dissatisfaction)

For each signal: Is it a positive or negative signal? How reliable is it? What are its blind spots?

---

## 🚦 Gate Check

Before moving to File 07, verify you can:

1. **☐** Explain the difference between monitoring and observability (and why both matter)
2. **☐** Design a tracing schema for a complex AI system (RAG, agent, or multi-agent)
3. **☐** Implement cost tracking with per-component breakdown
4. **☐** Design targeted dashboards for different stakeholders (SRE, PM, engineer)
5. **☐** Set up alerts that are actionable (not noise)
6. **☐** Implement a production-to-eval feedback loop
7. **☐** Design a sampling strategy that balances cost and coverage

### 🛑 Stop and Think

**Question 1 — The Observability ROI**

Your CTO asks: "We're spending $300/month on Langfuse. What are we getting for that?"

Quantify the ROI of observability. Think about:
- How much time does observability save during incident investigation? (Before observability: 4 hours to find root cause. After: 30 minutes.)
- How many incidents does observability prevent? (By detecting trends before they become incidents)
- How much does observability improve eval datasets? (By providing production data to the feedback loop)

**Question 2 — The Multi-System Observability Problem**

Your company has 5 AI systems: a chatbot, a code assistant, a content moderator, a recommendation engine, and a data extractor. Each uses different models, different frameworks, different deployment patterns.

Your VP wants ONE observability dashboard that shows the health of ALL 5 systems. But each system has different metrics (chatbot cares about answer quality, recommendation engine cares about CTR, content moderator cares about precision/recall).

How do you design a UNIFIED dashboard that's useful for each system team AND gives the VP an "overall AI health" view?

**Question 3 — Cross-Phase: Observability into Phases 1-5**

Choose ONE system from a previous phase (Phase 1's LLM gateway, Phase 3's semantic search, Phase 4's RAG bot, or Phase 5's advanced RAG). 

Design an observability plan for that system:
1. What are the 3 most important things to trace?
2. What are the 3 most important metrics?
3. What are the 3 most important alerts?
4. Which phase's eval concepts would you validate using production observability?

---

## 📚 Resources

- **"Observability for AI Systems"** (Langfuse, 2025) — The official Langfuse documentation. Their Python SDK documentation is excellent and covers tracing, spans, and cost tracking.
- **"Practical Observability for LLM Applications"** (Honeycomb, 2025) — Guide on applying observability patterns (high-cardinality, event-based) to LLM systems. Chapter 4 on sampling strategies is directly relevant.
- **"The Three Pillars of Observability"** (CNCF, 2021) — The original definition of traces, logs, and metrics. Essential foundation reading.
- **"LLM Application Costs in Production"** (Anthropic, 2025) — Analysis of real production costs across different AI system architectures. Their cost breakdown by component informed Example 2.

**Next Module:**
→ **07-Regression-Detection.md**: Detecting quality regressions with statistical methods, diagnosing root causes, and building automated rollback systems.
