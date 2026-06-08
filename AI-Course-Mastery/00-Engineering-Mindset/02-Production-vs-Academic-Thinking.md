# Production vs Academic Thinking

## 🎯 Purpose & Goals

Most AI resources are written by academics or researchers. They teach you elegant theory, clever architectures, and benchmark-chasing. Production engineering is the **opposite** — it's boring, pragmatic, and obsessed with edge cases.

**By the end of this file, you will:**
- Recognize the 80/20 trap and why it kills production AI systems
- Understand the 5 things that separate demos from production apps
- Be able to spot "academic thinking" in architecture decisions
- Have a repeatable framework for productionizing any AI feature

**⏱ Time Budget:** 1.5 hours

---

## 📖 The 80/20 Trap

```python
# ACADEMIC THINKING:
# "Let's implement the latest SOTA RAG technique with Graph-based retrieval,
#  multi-hop reasoning, and dynamic query decomposition."

# Time spent: 3 weeks of implementation
# Result: 85% accuracy on benchmarks
# Problem: Takes 6 seconds per query, costs $0.50 per query, 
#          breaks on 15% of edge cases, and you can't debug it

# PRODUCTION THINKING:
# "Let's start with naive RAG — embed, retrieve, generate."
# "Then measure where it fails."
# "Then fix the specific failures one at a time."

# Time spent: 1 day
# Result: 80% accuracy
# Problem identified: chunking is bad for legal text
# Fix: semantic chunking → 87% accuracy
# Next problem: retrieval misses relevant docs
# Fix: HyDE → 92% accuracy
# Total time: 1 week. Cost per query: $0.002. P95 latency: 800ms.
```

**The 80/20 Rule for AI Systems:**
- Getting from 0% → 80% quality takes 20% of the total time
- Getting from 80% → 95% takes the remaining 80% of time
- That final 80% is pure engineering — eval, debug, fix, repeat

**Senior engineers know this and plan for it.**
They build the simplest possible system first, measure it thoroughly,
and then iteratively improve the specific weak points — not the whole system.

---

## 📖 The Production Gap — 5 Things Demos Skip

### 1. Evaluation

```python
# ACADEMIC: "We tested on 3 examples and it looked good."
# PRODUCTION: 

from evals import EvalSuite, EvalCase

# Build a golden dataset FIRST
suite = EvalSuite("customer-support-bot")

suite.add_case(EvalCase(
    input="What is your return policy?",
    expected_contains=["return", "30 days", "receipt"],
    expected_not_contains=["I don't know"]
))

suite.add_case(EvalCase(
    input="I want a refund for my order #12345",
    expected_contains=["order", "refund", "investigate"],
))

# 50+ test cases covering:
# - Happy paths (standard questions)
# - Edge cases (empty input, very long input, non-English)
# - Failure modes (offensive input, jailbreak attempts)
# - Regression (previously fixed bugs)

results = suite.run()
# Results include: pass rate, per-metric scores, failed cases
# CI fails if pass rate < 0.85
```

### 2. Observability

```python
# ACADEMIC: print("response:", response)
# PRODUCTION:

from langfuse.decorators import observe, langfuse_context
from openai import OpenAI

client = OpenAI()

@observe(name="rag-retrieval")
def retrieve_context(query: str) -> list[str]:
    """Every call is traced: input, output, latency, cost"""
    results = vector_store.search(query, top_k=5)
    
    langfuse_context.update_current_observation(
        input=query,
        output={"num_chunks": len(results), "chunk_ids": [r["id"] for r in results]},
        metadata={"top_k": 5, "search_type": "semantic"},
    )
    return [r["text"] for r in results]

@observe(name="rag-generation")  
def generate_answer(query: str, context: list[str]) -> str:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Answer using provided context."},
            {"role": "user", "content": f"Context: {' '.join(context)}\n\nQuery: {query}"}
        ]
    )
    answer = response.choices[0].message.content
    
    # Log token usage
    langfuse_context.update_current_observation(
        usage={
            "input": response.usage.prompt_tokens,
            "output": response.usage.completion_tokens,
        },
        metadata={"model": "gpt-4o-mini", "context_length": len(context)}
    )
    return answer

@observe(name="rag-pipeline")
def rag_pipeline(query: str) -> str:
    context = retrieve_context(query)
    answer = generate_answer(query, context)
    return answer

# Now you can:
# - See every query in a dashboard
# - Filter by model, latency, cost
# - Find failing queries by score
# - Debug specific traces
```

### 3. Error Handling (Not Just Retries)

```python
# ACADEMIC:
try:
    result = call_llm(prompt)
except Exception:
    result = "Sorry, something went wrong."

# PRODUCTION — Multi-Layer Error Strategy:
from enum import Enum
from dataclasses import dataclass

class FailureMode(Enum):
    API_DOWN = "api_down"
    RATE_LIMITED = "rate_limited" 
    TIMEOUT = "timeout"
    BAD_OUTPUT = "bad_output"
    CONTEXT_OVERFLOW = "context_overflow"

@dataclass
class FallbackPlan:
    """Every AI call needs a fallback plan — not just a try/except"""
    model_tier: list[str]  # Try these models in order
    retry_count: int
    circuit_breaker_threshold: int
    fallback_response: str  # What to say if everything fails
    alert_on_failure: bool

def call_with_fallback(prompt: str, plan: FallbackPlan) -> str:
    """Try multiple models with circuit breaker pattern"""
    failures = 0
    
    for model in plan.model_tier:
        for attempt in range(plan.retry_count):
            try:
                response = client.chat.completions.create(
                    model=model,
                    messages=[{"role": "user", "content": prompt}],
                    timeout=10.0
                )
                return response.choices[0].message.content
                
            except (RateLimitError, APITimeoutError):
                failures += 1
                if failures >= plan.circuit_breaker_threshold:
                    # Circuit open — stop trying for a while
                    cache.set("circuit_open", True, ex=60)  # 1 minute cooldown
                    break
                time.sleep(2 ** attempt)
                
            except APIError:
                continue  # Try next model
    
    if plan.alert_on_failure:
        send_alert(f"All models failed for prompt: {prompt[:100]}")
    
    return plan.fallback_response
```

### 4. Graceful Degradation

```python
# ACADEMIC:
# Feature either works perfectly or crashes.

# PRODUCTION — Multiple Quality Tiers:

class ServiceLevel(Enum):
    FULL = "full"         # All features, best model
    DEGRADED = "degraded" # Cheaper model, fewer features  
    FALLBACK = "fallback" # Rule-based only, no AI
    OFFLINE = "offline"   # Clear message to user

class AIService:
    def __init__(self):
        self.current_level = ServiceLevel.FULL
        self.failure_count = 0
    
    def process_request(self, query: str) -> dict:
        """Automatically degrade service quality under stress"""
        
        if self.current_level == ServiceLevel.OFFLINE:
            return {"status": "offline", "message": "Service temporarily unavailable"}
        
        if self.current_level == ServiceLevel.FALLBACK:
            return self.rule_based_response(query)  # No AI at all
        
        try:
            if self.current_level == ServiceLevel.FULL:
                response = self.llm_complete(query, model="gpt-4o")
            elif self.current_level == ServiceLevel.DEGRADED:
                response = self.llm_complete(query, model="gpt-4o-mini")
            
            self.failure_count = 0
            return {"status": "ok", "response": response}
            
        except Exception:
            self.failure_count += 1
            
            if self.failure_count > 10:      # 10 failures
                self.current_level = ServiceLevel.DEGRADED
            if self.failure_count > 50:      # 50 failures
                self.current_level = ServiceLevel.FALLBACK  
            if self.failure_count > 100:     # 100 failures
                self.current_level = ServiceLevel.OFFLINE
            
            return self.process_request(query)  # Retry with new level
```

### 5. Cost Governance

```python
# ACADEMIC: Never considers cost.
# PRODUCTION:

import time
from dataclasses import dataclass

@dataclass
class CostMetrics:
    """Track every penny spent on AI inference"""
    model: str
    prompt_tokens: int
    completion_tokens: int
    cost_per_1k_input: float
    cost_per_1k_output: float
    
    @property
    def total_cost(self) -> float:
        input_cost = (self.prompt_tokens / 1000) * self.cost_per_1k_input
        output_cost = (self.completion_tokens / 1000) * self.cost_per_1k_output
        return input_cost + output_cost
    
    @property
    def cost_per_query(self) -> float:
        return self.total_cost

# COST MATRIX (2025 prices):
# Model                    Input ($/1M)  Output ($/1M)
# GPT-4o-mini              $0.15        $0.60
# GPT-4o                   $2.50        $10.00
# Claude 3.5 Sonnet        $3.00        $15.00
# Claude 3 Haiku           $0.25        $1.25
# Gemini 1.5 Flash         $0.075       $0.30

def estimate_monthly_cost(
    avg_input_tokens: int,
    avg_output_tokens: int,
    queries_per_day: int,
    model_input_cost: float,
    model_output_cost: float
) -> dict:
    """Estimate monthly AI costs BEFORE writing any code"""
    daily_cost = (
        (avg_input_tokens / 1000) * (model_input_cost / 1000) +
        (avg_output_tokens / 1000) * (model_output_cost / 1000)
    ) * queries_per_day
    
    monthly = daily_cost * 30
    
    return {
        "daily": round(daily_cost, 2),
        "monthly": round(monthly, 2),
        "yearly": round(monthly * 12, 2),
        "annual_per_user": round(monthly * 12 / 1000, 2) if queries_per_day > 1000 else "N/A"
    }

# Example: Customer support bot
costs = estimate_monthly_cost(
    avg_input_tokens=2000,    # user query + system prompt + context
    avg_output_tokens=300,    # response
    queries_per_day=10000,    # 10K conversations
    model_input_cost=2.50,    # GPT-4o
    model_output_cost=10.00,
)
print(costs)
# {'daily': 80.0, 'monthly': 2400.0, 'yearly': 28800.0, 'annual_per_user': 28.8}
# 
# SENIOR NOTE: $28,800/year for one AI feature.
# Is the feature worth $28,800? If not, use a cheaper model or optimize.
```

---

## ✅ The Production Checklist

For EVERY AI feature you build, ask these questions BEFORE writing code:

- [ ] **Do you have evals?** What's the baseline pass rate? How will you measure improvement?
- [ ] **What's the fallback?** If the model fails, what happens? If the API is down?
- [ ] **How will you trace this?** Can you see every model call in a dashboard?
- [ ] **What's the cost budget?** How much are you willing to spend per query? Per day?
- [ ] **What's the latency budget?** How long is too long for the user?
- [ ] **How will you detect degradation?** What metric triggers an alert?
- [ ] **What's the degradation plan?** What do users see when quality drops?
- [ ] **Who's responsible when it breaks?** Is there on-call rotation?

---

## 🧪 Drill: Productionize This Demo

**Time: 30 minutes**

Below is an "academic" implementation — works in a notebook, will fail in production.

```python
from openai import OpenAI

client = OpenAI()

def answer_from_docs(question: str, documents: list[str]) -> str:
    context = "\n".join(documents)
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "Answer using the context."},
            {"role": "user", "content": f"Context: {context}\n\nQuestion: {question}"}
        ]
    )
    return response.choices[0].message.content
```

**Your task:** List at least 10 things that will break in production. For each one, state the fix.

<details>
<summary>Sample answer (check after writing yours)</summary>

1. **No error handling** → API failures crash the app. Fix: try/except with retries.
2. **No timeout** → Hanging requests tie up resources. Fix: timeout=15.
3. **No input validation** → Empty questions, 100K token questions. Fix: validate length.
4. **No context size check** → Context might exceed model's limit. Fix: truncate or chunk.
5. **No output validation** → Model might return garbage. Fix: Pydantic validation.
6. **No cost tracking** → $28K/year surprise bill. Fix: log and monitor costs.
7. **No rate limit handling** → Multiple requests cause 429 errors. Fix: backoff.
8. **No fallback model** → If GPT-4o fails, system is down. Fix: model cascade.
9. **No observability** → Can't debug why a response was bad. Fix: Langfuse tracing.
10. **No eval** → Can't tell if changes improve quality. Fix: eval suite with golden dataset.
11. **Hardcoded model** → Can't switch without code change. Fix: config/env var.
12. **No caching** → Same question costs every time. Fix: Redis cache for common queries.
13. **Single-threaded** → Blocking call under load. Fix: async.
14. **No guardrails** → Model might hallucinate or produce harmful content. Fix: output guardrails.
15. **No CI/CD** → Can't test changes before deploy. Fix: eval as CI gate.
</details>

---

## 🚦 Gate Check

- [ ] Can you explain the 80/20 trap and how to avoid it?
- [ ] Can you name all 5 things demos skip that production requires?
- [ ] Can you list at least 10 things that break in production from a "simple" AI feature?
- [ ] Can you estimate the monthly cost of an AI feature before writing code?

**Proceed when all four are YES.**
