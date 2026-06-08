# 06 — Monitoring & Alerting: Knowing Before Your Users Tell You

## 🎯 Purpose & Goals

> 🛑 STOP. Before you deploy another AI service, answer this honestly:
>
> **How long would it take you to notice your model is serving garbage responses?**

If your answer is "a user would complain" or "we'd check the dashboard tomorrow morning," this module is for you.

**The hard truth about production AI:** Your model WILL degrade silently. It will:
- Start returning worse answers without changing error rates
- Drift as user queries shift over weeks
- Get slower as KV cache accumulates
- Cost more as usage patterns change
- Break in ways that don't crash — they just produce subtly wrong outputs

Traditional monitoring (CPU, memory, disk) doesn't catch these. You need **AI-aware monitoring** — metrics, dashboards, and alerts designed for the failure modes of intelligent systems.

**By the end of this module, you will:**
- Design a complete monitoring architecture with Prometheus, Grafana, and Loki
- Instrument your AI service with the FOUR golden signals (latency, traffic, errors, saturation)
- Build intelligent alerting that catches real problems without waking you up for noise
- Set SLOs for AI systems and measure burn rate
- Monitor model-specific metrics: token throughput, cache hit rate, embedding drift
- Build a cost-per-request dashboard that catches budget blowups early

---

### 🤔 Discovery Question 1: The Silent Quality Degradation

You deploy a summarization service. For 3 months, everything works. Your dashboards show:
- **Latency**: p50 = 800ms, p99 = 2.1s — stable
- **Error rate**: 0.3% — well below 1% threshold
- **Throughput**: 45 requests/min — steady
- **GPU memory**: 72% — healthy

Then your biggest customer emails: "Your summaries have been getting worse for 2 weeks. Key information is missing. We're considering alternatives."

You check all your dashboards. Every metric is green. The model hasn't changed. The code hasn't changed. **Everything looks fine but the output quality is degrading.**

**🤔 Before reading on, answer these:**

1. What could cause output quality to degrade when NO infrastructure metric has changed? Think about what COULD change in a deployed AI system without changing code or infrastructure. (Hint: The user queries changed — your customer started asking about a new product category. The model wasn't trained on this category. Your infrastructure metrics are fine but your MODEL doesn't know what it doesn't know. This is called **data drift** — the input distribution shifted away from the training distribution.)

2. Your infrastructure metrics show nothing wrong. Your model is still responding. Responses are still fast. Error rate is unchanged. **What metric would you track that captures "quality" in production?** How do you measure output quality without a human reading every response? (Hint: You need an automated eval — LLM-as-judge scoring each response in the request-response path. This is what Phase 7's evals infrastructure is for. But running an LLM judge on EVERY response costs money. What's the sampling strategy?)

3. **The hardest question:** You decide to run LLM-as-judge on 5% of responses in production. The judge scores each summary on: completeness, accuracy, conciseness. You track the average score over 7-day rolling windows. After setting this up, you detect the degradation that the customer noticed — the average score dropped from 8.7/10 to 7.1/10 over 2 weeks. **But the customer noticed BEFORE your monitoring did.** Your 7-day rolling window smoothed out the decline. How would you design an alert that catches quality degradation FASTER — within hours or days, not weeks — without triggering false alarms on normal variance? (Hint: Rolling averages lag. Consider: week-over-week comparisons, rate of change alerts, or per-customer cohort tracking instead of global averages.)

---

### 🤔 Discovery Question 2: The Alert That Destroyed Trust

You set up alerts for your AI service. Your pager is quiet for a month. Then, at 3:14 PM on a Tuesday:

**Page: Error rate > 5% (current: 18%)**

You drop everything. Investigate. Look at logs. Check the model. Check the API provider. 45 minutes later you find it: one user is sending 10,000 malformed requests per minute from a buggy script. Your error rate is 18% because 18% of ALL requests are from that one user's broken client. The OTHER 100 real users are fine.

The incident wasn't real. But you spent 45 minutes of focus time on it.

The next week: same alert. Same investigation. Same result — a different user with a broken script.

Your team starts ignoring the alert. "It's probably just another bad client." Then one day the error rate spikes to 25% because the model is actually returning 500 errors. Nobody reacts. The alert has been trained out of them. **Real incident, no response.**

**🤔 Before reading on, answer these:**

1. The problem: your error rate alert counts ALL errors together — 500s from the model, 400s from bad client requests, timeouts, rate limits. **How do you categorize errors so a single user's buggy script doesn't trigger a global alert?** What's the minimum error categorization you'd need? (Hint: At minimum: distinguish CLIENT errors (4xx — user's fault) from SERVER errors (5xx — your fault). More granular: timeout vs. model error vs. rate limit. Each gets its own alert threshold.)

2. A single user's bad client causes 90% of your 4xx errors. That's still "noise" — not a real incident. But what if the LARGEST legitimate user starts seeing a 10% error rate that's still a small fraction of your total error count? **How do you detect per-user or per-customer degradation before it becomes a global problem?** (Hint: Track error rates per user/API key. Alert when ANY customer exceeds threshold, not just when global average does. But then you need to handle 100+ customers — how do you avoid 100 alert rules?)

3. **The hardest question:** You implement categorization: 4xx = client, 5xx = server. You set 5xx alert threshold at 1%. Then one day the model starts returning subtly wrong answers — HTTP 200 with garbage content. Zero server errors. Zero client errors. Your alerts are silent. The users are unhappy. **How do you monitor for "successful but wrong" responses?** The HTTP status is 200. The latency is normal. No infrastructure metric is triggered. But the output is useless. What's the monitoring strategy for this class of failure? (Hint: This connects to Phase 7 — you need content-level evals running on sampled production traffic, not just HTTP-level metrics. But what's the sampling strategy that catches this within minutes, not hours?)

---

### 🤔 Discovery Question 3: The Metric You're Not Tracking Is Costing You

You have a beautiful Grafana dashboard:
- CPU, memory, GPU utilization
- Request latency (p50, p95, p99)
- Error rate by endpoint
- Token throughput (input + output)

Green lights everywhere. Your team is proud of the monitoring setup.

The CFO emails: "Our AI infrastructure bill was $47,000 last month. That's 3x higher than the month before. What changed?"

You check your dashboards. Traffic increased 2x month-over-month. That explains 2x. But the bill went 3x.

**🤔 Before reading on, answer these:**

1. Traffic went up 2x, cost went up 3x. **What metric explains the gap between 2x traffic and 3x cost?** Think about what makes AI costs non-linear with traffic. (Hint: Average output token count per request increased from 200 to 450 tokens. The model generates more tokens per request now because... wait, why would that happen without changing the model or prompt? Did users start asking harder questions? Did your prompt template change?)

2. You dig deeper: output tokens per request doubled because your prompt template was changed to include more context (more input tokens = more prefill compute + more likely long outputs). **The CFO shouldn't need to tell you cost is up. What cost-per-request dashboard would have shown this on DAY 1?** What granularity? Per-model? Per-endpoint? Per-user? (Hint: Track: cost per request, cost per token, tokens per request — per endpoint, per model, per user. A dashboard showing cost/request trending up over 30 days would have caught this in week 1, not month 3.)

3. **The hardest question:** You now track cost per request. You see it trending up. You investigate and find: users are pasting longer documents into the chat, which increases input token count, which increases prefill cost, which increases response length. **Should you LIMIT input length to control costs?** What happens if you truncate inputs? Some users lose functionality. Is there a BETTER way to handle long inputs that doesn't blindly truncate? (Hint: Consider: (a) tiered pricing — long inputs cost more, (b) smart truncation — summarize the middle not the beginning/end, (c) model routing — short inputs use GPT-4o, long inputs use a cheaper long-context model, (d) caching — semantically similar long inputs reuse previous responses. Each has different UX and cost implications.)

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Discovery Questions | 30 min |
| Concept: Four Golden Signals for AI | 30 min |
| Concept: SLOs, SLIs, Error Budgets | 30 min |
| Concept: Intelligent Alerting Design | 30 min |
| Code: Instrumenting a FastAPI Service (Fail-First) | 45 min |
| Code: Fixed Production Monitoring Stack | 45 min |
| Code: Grafana Dashboard JSON + Alert Rules | 30 min |
| Drills | 45 min |
| Gate Check | 15 min |
| **Total** | **~4 hr** |

---

## 📖 Concept Deep-Dive

### The Four Golden Signals — Adapted for AI

Google's SRE book defines four golden signals. Here's how they apply to AI services:

| Signal | Traditional | AI-Specific |
|--------|------------|-------------|
| **Latency** | HTTP response time | TTFT, tokens/sec, generation time vs queue time |
| **Traffic** | Requests per second | Requests per second + tokens per second + context length distribution |
| **Errors** | HTTP 5xx rate | 5xx + 4xx + **quality errors** (200 OK but wrong answer) |
| **Saturation** | CPU/memory/disk | GPU memory (KV cache), VRAM utilization, batch utilization |

**The critical insight:** Traditional monitoring catches INFRASTRUCTURE problems. AI monitoring catches MODEL problems. You need BOTH.

### Monitoring Architecture for AI Services

```
┌─────────────────────────────────────────────────────────┐
│                    Grafana Dashboards                    │
│  (Latency, Traffic, Errors, Saturation + Token Metrics  │
│   Cost/Request, Cache Hit Rate, Eval Scores)            │
└──────────────┬────────────────────┬─────────────────────┘
               │                    │
               ▼                    ▼
┌───────────────────────┐  ┌──────────────────┐
│    Prometheus         │  │      Loki        │
│  (Metrics DB)         │  │  (Logs DB)       │
│  - Counters/Gauges    │  │  - Request logs  │
│  - Histograms         │  │  - Error traces  │
│  - Alert evaluation   │  │  - Eval results  │
└───────┬───────────────┘  └───────┬──────────┘
        │                          │
        │  /metrics endpoint       │  structured logs
        ▼                          ▼
┌──────────────────────────────────────────────────────────┐
│                 AI Service (instrumented)                │
│  - Prometheus client metrics                             │
│  - Structured logging to stdout (picked up by Loki)      │
│  - OpenTelemetry traces for request flow                 │
└──────────────────────────────────────────────────────────┘
```

### SLOs, SLIs, Error Budgets

**Service-Level Indicator (SLI)** — What you measure:
- p99 latency < 3s
- Error rate < 1%
- Token throughput > 30 tok/s
- Eval score > 7/10

**Service-Level Objective (SLO)** — The target you commit to:
- "p99 latency < 3s over 30-day rolling window" — 99.9% of the time
- "Error rate < 1% over 7-day window"
- "Eval score > 7/10 on 95% of sampled responses"

**Error Budget** — How much failure you allow:
- If SLO = 99.9% uptime → error budget = 0.1% = ~43 minutes/month
- If you exceed error budget → STOP shipping new features. Fix reliability first.
- This creates a DISINCENTIVE to set unrealistic SLOs (you'll stop shipping)

**For AI services, error budget is more nuanced:**
- INFRASTRUCTURE error budget (downtime, 5xx) — traditional SLO
- QUALITY error budget (output score below threshold) — AI-specific
- COST error budget (cost per request exceeds target) — business SLO

### Alerting Design: The Four Alert Types

| Alert Type | Example | Action |
|-----------|---------|--------|
| **Page** | p99 latency > 5s for 5 min | Wake someone up |
| **Ticket** | Error rate trending up 50% WoW | Investigate during business hours |
| **Log** | Model switching to fallback | Log it, monitor trend |
| **Dashboard** | Cache hit rate dropping | Visualize, no action needed |

**The critical rule:** Every page-worthy alert must be ACTIONABLE, RARE, and REAL.
- Actionable: You can do something about it (restart, rollback, scale up)
- Rare: < 1 page per shift per person (otherwise you learn to ignore)
- Real: Detects actual incidents, not noise

### AI-Specific Metrics You Must Track

| Category | Metric | Why |
|----------|--------|-----|
| **Token Economics** | Input tokens/min, Output tokens/min, Tokens/request | Cost tracking, usage patterns |
| **Generation Quality** | TTFT (time to first token), Tokens/sec, Generation time | User experience |
| **Cache Efficiency** | Cache hit rate, Cache savings ($), Cache eviction rate | Cost optimization |
| **Batch Efficiency** | Average batch size, GPU utilization, Memory per sequence | Serving efficiency |
| **Fallback Rate** | Primary model failure count, Fallback model usage, Degradation time | Reliability |
| **Quality Evals** | Eval score (LLM-as-judge), Drift score, Per-category scores | Output quality |
| **Cost per Request** | Cost/request by model, by user, by endpoint | Business alignment |

---

## 💻 Code: Instrumenting a FastAPI AI Service

### Fail-First: The Monitoring Setup with 8 Deliberate Bugs

Below is a monitoring setup with deliberate bugs a junior engineer might write. **Before reading the fixed version, find all the bugs.**

Read the code carefully. Ask yourself: "What would go wrong in production with this monitoring code?"

```python
from fastapi import FastAPI, Request
from prometheus_client import Counter, Histogram, generate_latest, REGISTRY
from prometheus_client.exposition import CONTENT_TYPE_LATEST
from starlette.responses import Response
import time
import random

app = FastAPI()

# Bug 1: What's wrong with this metric naming?
inference_counter = Counter("inference", "Number of inference requests")
latency_histogram = Histogram("latency", "Request latency")  # Bug 2
error_counter = Counter("errors", "Number of errors")
token_counter = Counter("tokens", "Tokens generated")

@app.get("/metrics")
def metrics():
    return Response(content=generate_latest(REGISTRY), media_type=CONTENT_TYPE_LATEST)

@app.post("/chat")
async def chat_endpoint(request: Request):
    body = await request.json()
    prompt = body.get("prompt", "")
    model = body.get("model", "gpt-4o")
    
    start = time.time()
    
    try:
        # Simulate inference
        # Bug 3: How does this affect latency measurement?
        await simulate_inference(prompt, model)
        
        # Bug 4: When should we observe latency?
        latency = time.time() - start
        latency_histogram.observe(latency)
        
        # Track tokens
        output_tokens = len(prompt.split()) * 3  # Bug 5
        token_counter.inc(output_tokens)  # Bug 6
        
        inference_counter.inc()
        
        return {"response": f"Generated response for: {prompt[:50]}...", "tokens": output_tokens}
    
    except Exception as e:
        error_counter.inc()
        return {"error": str(e)}, 500

async def simulate_inference(prompt: str, model: str):
    """Simulate LLM inference with variable latency"""
    latency = random.uniform(0.1, 2.0)
    time.sleep(latency)
```

**Find the 8 bugs before reading the fixed version.**

<details>
<summary>🔍 Click to reveal bugs (try yourself first)</summary>

**Bug 1 — Generic metric names without labels:**
The counters `inference_counter`, `error_counter`, `token_counter` have NO labels. You can't filter by model, endpoint, user, or error type. Every counter looks the same — you can't distinguish "GPT-4o errors" from "Claude errors" or "chat errors" from "summarization errors."
- **Fix:** Add labels like `Counter("inference_requests_total", "Total requests", ["model", "endpoint"])`

**Bug 2 — Histogram without buckets:**
`Histogram("latency", "Request latency")` uses Prometheus DEFAULT buckets: [0.005, 0.01, 0.025, 0.05, ... 10.0]. These are designed for HTTP microservice latency (milliseconds to seconds). For AI latency where responses take 1-30 seconds, these buckets are WRONG — most requests fall into the last oversized bucket and you lose granularity.
- **Fix:** Custom buckets: `Histogram("latency_seconds", "...", buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0, 60.0])`

**Bug 3 — Measuring latency with await BEFORE start measurement ends:**
`simulate_inference` is awaited and it uses `time.sleep()` (blocking) instead of `asyncio.sleep()`. The simulated latency is correct, but in production, if you use blocking calls in async endpoints, you'll measure REQUEST QUEUE time as part of LATENCY because async event loop is blocked for ALL concurrent requests.
- **Fix:** Use `asyncio.sleep()` or ensure inference runs in a thread pool. Better: separate QUEUE time (time before inference starts) from GENERATION time (time spent in inference).

**Bug 4 — Observing latency after inference but before error handling:**
The code observes latency AFTER successful inference. But if inference FAILS (exception before line 36), `latency_histogram.observe(latency)` is NEVER called. Failed requests are invisible in latency metrics, making your latency dashboards look better than reality.
- **Fix:** Use a try/finally or measure latency in a wrapper that records BOTH success and failure latency separately.

**Bug 5 — Token count estimation is garbage:**
`len(prompt.split()) * 3` is NOT how LLM tokenization works. Token count is NOT word count × 3. This can be off by 2-10x depending on the language. You're tracking fake numbers.
- **Fix:** Use the actual tokenizer to count tokens: `len(tokenizer.encode(prompt))`, or use the API's returned `usage` field.

**Bug 6 — Using `.inc(value)` on a Counter for a dynamic value:**
`token_counter.inc(output_tokens)` increments the counter by `output_tokens`. This is VALID usage. But the problem is semantic: you're adding tokens to a SINGLE counter across ALL requests. If you ever reset or need per-request token counts, you've lost granularity.
- **Fix:** Use labels per model/endpoint OR use a Histogram for tokens per request bucketing.

**Bug 7 — No TTFT histogram:**
The code measures total latency but NOT time-to-first-token (TTFT). TTFT is the most important UX metric for streaming applications. Without TTFT, you can't tell if users are staring at a blank screen or seeing text flow immediately.
- **Fix:** Separate `ttft_histogram` that records time until first token is generated, separate from total generation time.

**Bug 8 — No request tracking across endpoints:**
Every endpoint in a real service should track its OWN metrics. This code only instruments `/chat`. What about `/summarize`, `/embed`, `/classify`? Each endpoint has different latency, error, and throughput profiles. Tracking them all in one metric hides per-endpoint problems.
- **Fix:** Use middleware-level instrumentation that automatically tracks ALL endpoints, or create per-endpoint metrics manually.

</details>

---

### ✅ Fixed: Production Monitoring Stack

```python
"""
production_monitoring.py — Complete monitoring instrumentation for AI services.

Install:
    pip install prometheus-client fastapi uvicorn httpx

Usage:
    python production_monitoring.py
    
    Then visit http://localhost:8000/metrics for Prometheus scrape endpoint.
"""

import asyncio
import time
import uuid
from contextlib import asynccontextmanager
from dataclasses import dataclass, field
from enum import Enum
from typing import AsyncGenerator, Optional

from fastapi import FastAPI, Request, Response
from fastapi.middleware.base import BaseHTTPMiddleware
from prometheus_client import Counter, Histogram, Gauge, generate_latest, REGISTRY
from prometheus_client.exposition import CONTENT_TYPE_LATEST
import httpx


# ── Metric Definitions ────────────────────────────────────
# Naming convention: <namespace>_<subsystem>_<metric>_<unit>
# Example: ai_service_inference_duration_seconds

# --- Latency Metrics ---
# Custom buckets for AI workloads (0.1s to 120s)
AI_LATENCY_BUCKETS = [0.05, 0.1, 0.25, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0, 60.0, 120.0]

request_duration = Histogram(
    "ai_request_duration_seconds",
    "Request latency by endpoint and model",
    ["endpoint", "model", "status"],  # Can slice by any dimension
    buckets=AI_LATENCY_BUCKETS,
)

ttft_duration = Histogram(
    "ai_ttft_seconds",
    "Time to first token by endpoint and model",
    ["endpoint", "model"],
    buckets=[0.05, 0.1, 0.25, 0.5, 1.0, 2.0, 5.0, 10.0],
)

generation_duration = Histogram(
    "ai_generation_duration_seconds",
    "Generation phase duration (after first token)",
    ["endpoint", "model"],
    buckets=AI_LATENCY_BUCKETS,
)

# --- Throughput Metrics ---
requests_total = Counter(
    "ai_requests_total",
    "Total inference requests by endpoint, model, status",
    ["endpoint", "model", "status"],
)

tokens_input_total = Counter(
    "ai_tokens_input_total",
    "Total input tokens by model and endpoint",
    ["endpoint", "model"],
)

tokens_output_total = Counter(
    "ai_tokens_output_total",
    "Total output tokens by model and endpoint",
    ["endpoint", "model"],
)

tokens_per_request = Histogram(
    "ai_tokens_per_request",
    "Tokens per request by type (input/output)",
    ["endpoint", "model", "token_type"],  # token_type = "input" or "output"
    buckets=[50, 100, 250, 500, 1000, 2000, 4000, 8000, 16000],
)

# --- Error Metrics ---
errors_total = Counter(
    "ai_errors_total",
    "Errors by endpoint, model, error_type",
    ["endpoint", "model", "error_type"],  # error_type: "timeout", "rate_limit", "model_error", "invalid_request", "internal"
)

# --- Saturation Metrics ---
gpu_memory_usage = Gauge(
    "ai_gpu_memory_usage_percent",
    "GPU memory utilization",
    ["gpu_id"],
)

batch_size = Gauge(
    "ai_vllm_batch_size",
    "Current vLLM batch size",
    ["instance"],
)

kv_cache_usage = Gauge(
    "ai_kv_cache_usage_percent",
    "KV cache utilization",
    ["instance"],
)

# --- Cache Metrics ---
cache_hits = Counter(
    "ai_cache_hits_total",
    "Cache hits by cache layer",
    ["cache_layer"],  # "response_cache", "semantic_cache", "prefix_cache"
)

cache_misses = Counter(
    "ai_cache_misses_total",
    "Cache misses by cache layer",
    ["cache_layer"],
)

# --- Cost Metrics ---
cost_per_request = Histogram(
    "ai_cost_per_request_usd",
    "Estimated cost per request by model",
    ["endpoint", "model"],
    buckets=[0.0001, 0.0005, 0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0],
)

cost_total = Counter(
    "ai_cost_total_usd",
    "Total estimated cost by model",
    ["model"],
)

# --- Quality / Eval Metrics ---
eval_score = Gauge(
    "ai_eval_score",
    "LLM-as-judge quality score (sampled, averaged)",
    ["endpoint", "eval_criteria"],  # eval_criteria: "accuracy", "helpfulness", "safety"
)

# --- Consumer / User Metrics ---
per_user_errors = Counter(
    "ai_per_user_errors_total",
    "Errors by user_id (high cardinality — use carefully)",
    ["user_id", "error_type"],
)


# ── Data Models ───────────────────────────────────────────

class ErrorType(str, Enum):
    TIMEOUT = "timeout"
    RATE_LIMIT = "rate_limit"
    MODEL_ERROR = "model_error"
    INVALID_REQUEST = "invalid_request"
    INTERNAL = "internal"


class StatusType(str, Enum):
    SUCCESS = "success"
    CLIENT_ERROR = "client_error"  # 4xx
    SERVER_ERROR = "server_error"  # 5xx


@dataclass
class TokenUsage:
    input_tokens: int
    output_tokens: int

    @property
    def total(self) -> int:
        return self.input_tokens + self.output_tokens


@dataclass
class RequestMetrics:
    endpoint: str
    model: str
    user_id: str
    request_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    status: StatusType = StatusType.SUCCESS
    error_type: Optional[ErrorType] = None
    
    # Timing — set during request lifecycle
    queue_start: Optional[float] = None
    inference_start: Optional[float] = None
    first_token_time: Optional[float] = None
    end_time: Optional[float] = None
    
    # Tokens
    token_usage: Optional[TokenUsage] = None
    
    # Cost estimation
    estimated_cost: Optional[float] = None


# ── Monitoring Middleware ──────────────────────────────────

class MonitoringMiddleware(BaseHTTPMiddleware):
    """
    Automatic instrumentation for ALL endpoints.
    
    Captures:
    - Request duration, TTFT, generation time
    - Error categorization (4xx vs 5xx vs quality)
    - Token counts (requires integration)
    - Cost estimation
    """
    
    # Cost per 1K tokens (approximate — update with your pricing)
    COST_PER_1K_INPUT = {"gpt-4o": 0.0025, "gpt-4o-mini": 0.00015, "claude-3-haiku": 0.00025}
    COST_PER_1K_OUTPUT = {"gpt-4o": 0.01, "gpt-4o-mini": 0.0006, "claude-3-haiku": 0.00125}
    
    async def dispatch(self, request: Request, call_next):
        # Skip metrics endpoint
        if request.url.path == "/metrics":
            return await call_next(request)
        
        # Start the clock
        start_time = time.monotonic()
        
        # Determine endpoint from route
        endpoint = request.url.path
        
        # Try to get model from request body (best-effort)
        model = "unknown"
        try:
            body = await request.json()
            model = body.get("model", "unknown")
        except Exception:
            pass
        
        # Initialize metrics context
        metrics_ctx = RequestMetrics(
            endpoint=endpoint,
            model=model,
            user_id=request.headers.get("X-User-ID", "anonymous"),
        )
        
        try:
            # Call the actual endpoint
            response = await call_next(request)
            
            # Determine status category
            if response.status_code >= 500:
                metrics_ctx.status = StatusType.SERVER_ERROR
                metrics_ctx.error_type = ErrorType.INTERNAL
            elif response.status_code >= 400:
                metrics_ctx.status = StatusType.CLIENT_ERROR
                metrics_ctx.error_type = self._categorize_client_error(
                    response.status_code
                )
            
            # Record metrics
            duration = time.monotonic() - start_time
            self._record_metrics(metrics_ctx, duration)
            
            return response
            
        except Exception as exc:
            # Catch unhandled exceptions
            duration = time.monotonic() - start_time
            metrics_ctx.status = StatusType.SERVER_ERROR
            metrics_ctx.error_type = ErrorType.INTERNAL
            
            self._record_metrics(metrics_ctx, duration)
            
            raise
    
    def _categorize_client_error(self, status_code: int) -> ErrorType:
        if status_code == 429:
            return ErrorType.RATE_LIMIT
        elif status_code == 400:
            return ErrorType.INVALID_REQUEST
        else:
            return ErrorType.CLIENT_ERROR  # type: ignore
    
    def _record_metrics(self, ctx: RequestMetrics, duration: float):
        # Labels for base metrics
        endpoint = ctx.endpoint
        model = ctx.model
        
        # Request count by status
        requests_total.labels(
            endpoint=endpoint, model=model, status=ctx.status.value
        ).inc()
        
        # Duration (only for successful requests — failed ones have misleading duration)
        if ctx.status == StatusType.SUCCESS:
            request_duration.labels(
                endpoint=endpoint, model=model, status="success"
            ).observe(duration)
        
        # Errors
        if ctx.error_type:
            errors_total.labels(
                endpoint=endpoint, model=model, error_type=ctx.error_type.value
            ).inc()
            
            # Per-user error tracking
            if ctx.user_id != "anonymous":
                per_user_errors.labels(
                    user_id=ctx.user_id, error_type=ctx.error_type.value
                ).inc()
        
        # Token and cost metrics (if available)
        if ctx.token_usage:
            tokens_input_total.labels(
                endpoint=endpoint, model=model
            ).inc(ctx.token_usage.input_tokens)
            
            tokens_output_total.labels(
                endpoint=endpoint, model=model
            ).inc(ctx.token_usage.output_tokens)
            
            tokens_per_request.labels(
                endpoint=endpoint, model=model, token_type="input"
            ).observe(ctx.token_usage.input_tokens)
            
            tokens_per_request.labels(
                endpoint=endpoint, model=model, token_type="output"
            ).observe(ctx.token_usage.output_tokens)
            
            # Cost estimation
            input_cost = (ctx.token_usage.input_tokens / 1000) * self.COST_PER_1K_INPUT.get(model, 0.001)
            output_cost = (ctx.token_usage.output_tokens / 1000) * self.COST_PER_1K_OUTPUT.get(model, 0.005)
            total_cost = input_cost + output_cost
            
            cost_per_request.labels(
                endpoint=endpoint, model=model
            ).observe(total_cost)
            
            cost_total.labels(model=model).inc(total_cost)


# ── Application Setup ────────────────────────────────────

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifespan events — startup and shutdown."""
    # Startup: log that monitoring is active
    print("🚀 AI Monitoring initialized")
    print("  → /metrics endpoint available at port 8000")
    print("  → Custom AI latency buckets: ", AI_LATENCY_BUCKETS)
    yield
    # Shutdown: nothing needed


app = FastAPI(lifespan=lifespan)
app.add_middleware(MonitoringMiddleware)


# ── Metrics Endpoint ─────────────────────────────────────

@app.get("/metrics")
async def metrics():
    """Prometheus scrape endpoint."""
    return Response(
        content=generate_latest(REGISTRY),
        media_type=CONTENT_TYPE_LATEST,
    )


# ── Simulated AI Endpoints ──────────────────────────────

@app.post("/chat")
async def chat_endpoint(request: Request):
    """Simulated chat endpoint with realistic timing."""
    body = await request.json()
    prompt = body.get("prompt", "Hello")
    
    # Simulate real tokenization
    # In production: use tiktoken for OpenAI, or model's tokenizer
    input_tokens = max(1, len(prompt) // 4)  # Rough: ~4 chars per token
    
    # Simulate inference with realistic latency
    # Prefill: ~0.5ms per input token for a 7B model
    prefill_time = input_tokens * 0.0005
    
    # Generation: ~25ms per output token for a 7B model
    output_tokens = min(500, max(10, input_tokens // 2))
    gen_time = output_tokens * 0.025
    
    # Add some variance
    import random
    prefill_jitter = random.uniform(0.8, 1.2)
    gen_jitter = random.uniform(0.8, 1.2)
    
    await asyncio.sleep(prefill_time * prefill_jitter + gen_time * gen_jitter)
    
    # Estimate cost
    input_cost = (input_tokens / 1000) * 0.0025
    output_cost = (output_tokens / 1000) * 0.01
    
    return {
        "response": f"Simulated response ({output_tokens} tokens)",
        "usage": {
            "input_tokens": input_tokens,
            "output_tokens": output_tokens,
            "estimated_cost_usd": round(input_cost + output_cost, 6),
        },
    }


@app.post("/summarize")
async def summarize_endpoint(request: Request):
    """Simulated summarization endpoint — longer inputs, shorter outputs."""
    body = await request.json()
    text = body.get("text", "")
    
    input_tokens = max(1, len(text) // 4)
    output_tokens = max(10, input_tokens // 5)
    
    prefill_time = input_tokens * 0.0005
    gen_time = output_tokens * 0.025
    
    await asyncio.sleep(prefill_time + gen_time)
    
    return {
        "summary": f"Summary of {input_tokens}-token input ({output_tokens} tokens)",
        "usage": {"input_tokens": input_tokens, "output_tokens": output_tokens},
    }


@app.post("/classify")
async def classify_endpoint(request: Request):
    """Simulated classification endpoint — fast, few tokens."""
    body = await request.json()
    text = body.get("text", "")
    
    input_tokens = max(1, len(text) // 4)
    
    # Classification is fast: minimal output
    prefill_time = input_tokens * 0.0005
    gen_time = 0.01  # Tiny — usually single token or < 50 tokens
    
    await asyncio.sleep(prefill_time + gen_time)
    
    return {"category": "positive", "confidence": 0.92}


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### 📊 Expected Output

Running this instrumented service, then hitting it with traffic:

```bash
# Terminal 1: Start the instrumented service
python production_monitoring.py
# → 🚀 AI Monitoring initialized
# → /metrics endpoint available at port 8000
# → Custom AI latency buckets: [0.05, 0.1, 0.25, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0, 60.0, 120.0]

# Terminal 2: Generate traffic
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Explain quantum computing in simple terms", "model": "gpt-4o"}'

curl -X POST http://localhost:8000/summarize \
  -H "Content-Type: application/json" \
  -d '{"text": "Long document text here...", "model": "gpt-4o-mini"}'

# Terminal 2: Scrape metrics
curl http://localhost:8000/metrics
```

Metrics output (abbreviated):

```
# HELP ai_request_duration_seconds Request latency by endpoint and model
# TYPE ai_request_duration_seconds histogram
ai_request_duration_seconds_bucket{endpoint="/chat",model="gpt-4o",status="success",le="0.05"} 0
ai_request_duration_seconds_bucket{endpoint="/chat",model="gpt-4o",status="success",le="0.5"} 3
ai_request_duration_seconds_bucket{endpoint="/chat",model="gpt-4o",status="success",le="1.0"} 8
ai_request_duration_seconds_bucket{endpoint="/chat",model="gpt-4o",status="success",le="2.0"} 12
ai_request_duration_seconds_bucket{endpoint="/chat",model="gpt-4o",status="success",le="5.0"} 15
ai_request_duration_seconds_bucket{endpoint="/chat",model="gpt-4o",status="success",le="+Inf"} 15
ai_request_duration_seconds_count{endpoint="/chat",model="gpt-4o",status="success"} 15
ai_request_duration_seconds_sum{endpoint="/chat",model="gpt-4o",status="success"} 22.345

# HELP ai_errors_total Errors by endpoint, model, error_type
# TYPE ai_errors_total counter
ai_errors_total{endpoint="/chat",model="gpt-4o",error_type="timeout"} 2
ai_errors_total{endpoint="/chat",model="gpt-4o",error_type="rate_limit"} 5
ai_errors_total{endpoint="/chat",model="unknown",error_type="invalid_request"} 1

# HELP ai_cost_total_usd Total estimated cost by model
# TYPE ai_cost_total_usd counter
ai_cost_total_usd{model="gpt-4o"} 0.0873
ai_cost_total_usd{model="gpt-4o-mini"} 0.0021
```

### Grafana Dashboard JSON (Key Panels)

Here's the core dashboard structure for an AI monitoring dashboard (PromQL queries):

```json
{
  "title": "AI Service Monitoring",
  "panels": [
    {
      "title": "Request Rate by Endpoint",
      "type": "timeseries",
      "targets": [{
        "expr": "sum(rate(ai_requests_total[5m])) by (endpoint)",
        "legendFormat": "{{endpoint}}"
      }]
    },
    {
      "title": "p50 / p95 / p99 Latency",
      "type": "timeseries",
      "targets": [
        {
          "expr": "histogram_quantile(0.50, sum(rate(ai_request_duration_seconds_bucket[5m])) by (le))",
          "legendFormat": "p50"
        },
        {
          "expr": "histogram_quantile(0.95, sum(rate(ai_request_duration_seconds_bucket[5m])) by (le))",
          "legendFormat": "p95"
        },
        {
          "expr": "histogram_quantile(0.99, sum(rate(ai_request_duration_seconds_bucket[5m])) by (le))",
          "legendFormat": "p99"
        }
      ]
    },
    {
      "title": "Error Rate by Type",
      "type": "timeseries",
      "targets": [{
        "expr": "sum(rate(ai_errors_total[5m])) by (error_type)",
        "legendFormat": "{{error_type}}"
      }]
    },
    {
      "title": "Tokens Per Second (Input vs Output)",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate(ai_tokens_input_total[5m]))",
          "legendFormat": "Input"
        },
        {
          "expr": "sum(rate(ai_tokens_output_total[5m]))",
          "legendFormat": "Output"
        }
      ]
    },
    {
      "title": "Cost Per Hour by Model",
      "type": "timeseries",
      "targets": [{
        "expr": "sum(rate(ai_cost_total_usd[1h])) by (model)",
        "legendFormat": "{{model}}"
      }]
    },
    {
      "title": "Cache Hit Rate",
      "type": "timeseries",
      "targets": [{
        "expr": "sum(rate(ai_cache_hits_total[5m])) / (sum(rate(ai_cache_hits_total[5m])) + sum(rate(ai_cache_misses_total[5m]))) * 100",
        "legendFormat": "Hit Rate %"
      }]
    },
    {
      "title": "GPU Memory Utilization",
      "type": "timeseries",
      "targets": [{
        "expr": "ai_gpu_memory_usage_percent{gpu_id=~\"0|1\"}",
        "legendFormat": "GPU {{gpu_id}}"
      }]
    },
    {
      "title": "Top Users by Error Count (Last 1h)",
      "type": "barchart",
      "targets": [{
        "expr": "topk(10, sum(increase(ai_per_user_errors_total[1h])) by (user_id))",
        "legendFormat": "{{user_id}}"
      }]
    }
  ]
}
```

### Intelligent Alert Rules (Prometheus)

```yaml
# prometheus-alerts.yml
groups:
  - name: ai_service_alerts
    rules:
      # ── HIGH PRIORITY: Page immediately ──
      
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(ai_errors_total{error_type=~"model_error|internal|timeout"}[5m]))
            /
            sum(rate(ai_requests_total[5m]))
          ) > 0.05
        for: 2m
        labels:
          severity: page
          team: ai-infra
        annotations:
          summary: "Error rate {{ $value | humanizePercentage }} — above 5% threshold"
          
      - alert: HighP99Latency
        expr: |
          histogram_quantile(
            0.99,
            sum(rate(ai_request_duration_seconds_bucket[5m])) by (le)
          ) > 10.0
        for: 3m
        labels:
          severity: page
          team: ai-infra
        annotations:
          summary: "p99 latency {{ $value }}s — above 10s threshold"

      # ── MEDIUM PRIORITY: Ticket (investigate next business day) ──

      - alert: LatencyTrendingUp
        expr: |
          (
            histogram_quantile(0.95, sum(rate(ai_request_duration_seconds_bucket[1h])) by (le))
            /
            histogram_quantile(0.95, sum(rate(ai_request_duration_seconds_bucket[1h] offset 1d)) by (le))
          ) > 1.5
        labels:
          severity: ticket
          team: ai-infra
        annotations:
          summary: "p95 latency increased 50%+ compared to yesterday"

      - alert: CostSpike
        expr: |
          sum(rate(ai_cost_total_usd[1h])) > sum(rate(ai_cost_total_usd[1h] offset 7d)) * 2
        labels:
          severity: ticket
          team: ai-infra
        annotations:
          summary: "Cost rate doubled compared to last week"

      - alert: CacheHitRateDropping
        expr: |
          (
            sum(rate(ai_cache_hits_total{cache_layer="semantic_cache"}[1h]))
            /
            (sum(rate(ai_cache_hits_total{cache_layer="semantic_cache"}[1h]))
             + sum(rate(ai_cache_misses_total{cache_layer="semantic_cache"}[1h])))
          ) < 0.3
        labels:
          severity: ticket
          team: ai-infra
        annotations:
          summary: "Semantic cache hit rate below 30% — may be worth investigating"

      # ── LOW PRIORITY: Log / Dashboard ──

      - alert: PerUserErrorRate
        expr: |
          sum by (user_id) (
            rate(ai_per_user_errors_total[5m])
          ) > 0.5
        labels:
          severity: log
          team: ai-infra
        annotations:
          summary: "User {{ $labels.user_id }} has high error rate ({{ $value }}/s)"
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Dashboard Graveyard

You build 25 beautiful Grafana dashboards. After 2 weeks, nobody looks at them. They're too noisy, too complex, or not relevant.

**Fix:** Start with ONE "p0" dashboard — the 8 panels listed above. Add more only when you actually need them for debugging.

### Antipattern 2: Alert on Everything

Every metric has an alert. You get 50 pages per day. You learn to ignore them. Real incidents blend into noise.

**Fix:** Every alert must have a documented RUNBOOK. If you can't write a runbook for it, it shouldn't page.

### Antipattern 3: Averaging Away Problems

You track "average latency" as 1.2s. Looks fine. But p99 is 15s and growing. The average is hiding a growing tail.

**Fix:** Always track percentiles (p50, p95, p99), never averages.

### Antipattern 4: No Quality Metrics

All your metrics are green. But the model has been returning garbage for 3 days. You only find out when the customer churns.

**Fix:** Implement LLM-as-judge eval on sampled production traffic (Phase 7 integration). Track eval score as a P0 metric alongside latency and errors.

### Antipattern 5: Cost Blindness

You optimize latency perfectly. But the cost per request has doubled. Nobody noticed until the bill arrives.

**Fix:** Cost per request on the main dashboard — next to latency and errors. It's as important as performance.

### Common Monitoring Failures

| Failure | Symptom | Cause | Fix |
|---------|---------|-------|-----|
| False alarm fatigue | Pages ignored | Threshold too sensitive | Use rate-of-change, not absolute; require sustained breach for N minutes |
| Silent degradation | No alert, user complaints | No quality monitoring | Add eval score tracking (Phase 7) |
| Dashboard blindness | Nobody checks | Too many dashboards | Single P0 dashboard, everything else on-demand |
| Metric explosion | Prometheus performance issues | High-cardinality labels (user_id, request_id) in high-volume metrics | Limit cardinality; use exemplars or separate log-based tracking |
| Cost surprise | CFO notices first | No cost tracking | Add cost/request to every metric label |
| Alert on symptom, not cause | "Latency high" page but root cause is GPU memory | Alert on EFFECT not CAUSE | Alert on GPU memory saturation DIRECTLY; latency is the symptom |
| Gap in coverage | Missing metrics for new endpoint | No instrumentation of new code paths | Use middleware-level instrumentation (catches ALL endpoints automatically) |

---

## 🧪 Drills & Challenges

### Drill 1: Instrument Your Own AI Endpoint (45 min)

Take an existing AI endpoint (any project you've built) and instrument it:
1. Add the 4 golden signal metrics (latency, requests, errors, tokens)
2. Create custom latency buckets for AI workloads
3. Add cost-per-request estimation
4. Expose a /metrics endpoint and verify Prometheus can scrape it

**Expected output:** A running service with `/metrics` endpoint returning properly labeled Prometheus metrics.

### Drill 2: Design an Alert for Quality Degradation (30 min)

You can't run LLM-as-judge on every response (too expensive). Design a sampling strategy:
- Sample 1% of responses for eval
- What's the minimum sample size to detect a 10% quality drop within 1 hour?
- What statistical test do you use to decide if the drop is significant?
- How do you alert without false alarms from normal variance?

**Expected output:** A written strategy with: sample rate, statistical method, alert threshold, and expected detection latency.

### Drill 3: Build a Cost Dashboard (30 min)

You have these models in production:
- GPT-4o: $0.0025/1K input, $0.01/1K output — 500 req/min, avg 300 input + 150 output tokens
- GPT-4o-mini: $0.00015/1K input, $0.0006/1K output — 2,000 req/min, avg 400 input + 200 output tokens
- Claude-3-Haiku: $0.00025/1K input, $0.00125/1K output — 300 req/min, avg 1,000 input + 300 output tokens

Design a Prometheus metric + Grafana panel that shows:
- Cost per hour by model
- Cost per request by model
- Projected monthly cost
- Cost trend (compare to last week)

**Expected output:** PromQL queries and Grafana panel JSON for each visualization.

### Drill 4: The Incident Postmortem (30 min)

You get paged at 3 AM. p99 latency is 25s (threshold: 5s). Error rate is 12% (threshold: 1%).

Write a real postmortem covering:
1. **What metrics would you check to diagnose the root cause?** (In order)
2. **What dashboard would tell you in 30 seconds whether it's:**
   - GPU memory saturation (KV cache full)
   - Network issue (API provider degraded)
   - Traffic spike (2x normal load)
   - Model regression (new deployment)
3. **What runbook action would you take for each possible cause?**

**Expected output:** A postmortem template with your diagnostic flow and runbook steps.

---

## 🚦 Gate Check

Before moving to File 07 (Multi-Provider Failover), verify you can:

1. **Design a monitoring architecture** — For a service with 3 endpoints (chat, summarize, classify), specify: what metrics you'd track, which are counter vs histogram vs gauge, and what labels each needs

2. **Write PromQL queries** — Given a `ai_request_duration_seconds` histogram with `endpoint` and `model` labels, write queries for: p50 latency by endpoint, p95 latency for chat endpoint only, rate of requests > 5s

3. **Design intelligent alerts** — For a quality degradation scenario (model returning wrong answers at 200 OK), design: what metric to track, how to sample, what threshold, what alert severity, and what runbook action

4. **Set SLOs for an AI service** — Given: chatbot with 50ms TTFT SLO, 2s total latency SLO, 99.5% uptime, 8/10 quality score. Calculate the error budget for each SLO. What happens when any budget is exhausted?

5. **Debug a monitoring gap** — A team has "green" dashboards but users complain about errors. Walk through the diagnostic process: what metrics to check, what's likely missing, and what new metric would catch the issue

---

## 📚 Resources

### Monitoring Fundamentals
- **[Google SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)** — The four golden signals, best practices for alerting
- **[Prometheus Documentation](https://prometheus.io/docs/prometheus/latest/querying/basics/)** — PromQL basics, metric types, recording rules
- **[Grafana Fundamentals](https://grafana.com/tutorials/)** — Dashboard design, transformations, alerting

### AI-Specific Monitoring
- **[Why Observability Is Critical for LLM Apps](https://www.honeycomb.io/blog/observability-llm-applications)** — High-cardinality tracing for AI workloads
- **[Langfuse Documentation](https://langfuse.com/docs)** — Open-source tracing for LLM apps (complements Prometheus metrics with detailed traces)
- **[Weights & Biases Prompts](https://wandb.ai/site/prompts)** — Monitoring and versioning prompts in production

### Alerting Design
- **[Alerting Best Practices](https://prometheus.io/docs/practices/alerting/)** — Prometheus alerting rules, runbooks, silences
- **[My Philosophy on Alerting](https://docs.google.com/document/d/199PqyG3UsyXlwiee9FgGMiJ1sxVsdHm3i1QjWzWFAW4/edit)** — The "worth paging" framework (Rob Ewaschuk)

### Phase Cross-References
- **Phase 7 (Evals & Observability)** → LLM-as-judge eval metrics integrate directly with Grafana dashboards for quality monitoring
- **Phase 10, File 03 (Caching)** → Cache hit rate monitoring is critical for cost optimization dashboards
- **Phase 10, File 04 (Cost)** → Cost-per-request dashboards connect to spot instance pricing and arbitrage strategies
- **Phase 10, File 05 (Latency)** → Latency dashboards decompose into TTFT, generation time, and queue time — corresponding to the optimization techniques in File 05

> **Next up: File 07 — Multi-Provider Failover & Disaster Recovery.** Your monitoring caught a problem — your primary model provider is down. Now what? You'll build a multi-provider failover system that switches between AWS Bedrock, self-hosted vLLM, and managed APIs without dropping user requests.
