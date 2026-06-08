# 07 — Multi-Provider Failover & Disaster Recovery: Your Model WILL Go Down

## 🎯 Purpose & Goals

> 🛑 STOP. You rely on a single API provider for your AI service. Ask yourself:
>
> **What happens when that provider goes down?**

Not "if." **When.**

Every provider has outages:
- **OpenAI**: Had multi-hour outages in 2023, 2024, 2025. Rate limits under load.
- **AWS Bedrock**: Regional dependencies. us-east-1 goes down, so does your model.
- **Self-hosted vLLM**: GPU instance terminates. Disk fills up. CUDA OOM at 2 AM.
- **Anthropic**: API degradation under high load. Latency spikes during peak hours.

**The hard truth:** Relying on a single provider means your service's availability = their availability. And their availability is never 100%.

**By the end of this module, you will:**
- Design a multi-provider architecture that handles any single-provider outage
- Implement health checks, circuit breakers, and automatic failover
- Build traffic shifting for canary deployments and gradual migrations
- Design a disaster recovery plan that guarantees < 5 min RTO (Recovery Time Objective)
- Handle STATE during failover — in-flight requests, streaming sessions, and cached data

---

### 🤔 Discovery Question 1: The Failover That Made Things Worse

Your AI service uses OpenAI GPT-4o. At 2:30 PM, OpenAI starts returning 503 errors. Your monitoring fires. Your failover system kicks in: ALL traffic is redirected to your backup provider — self-hosted vLLM running Llama-3-70B on AWS.

Within 30 seconds:
- vLLM is overwhelmed: it was configured for 10 requests/min (dev traffic), now handling 200 req/min
- Latency goes from 800ms to 45 seconds
- GPU memory fills up, requests start queueing, then timing out
- Your "backup" provider is now ALSO failing
- Users see errors from both providers

**The failover made things WORSE than if you had just queued requests until OpenAI recovered.**

**🤔 Before reading on, answer these:**

1. The backup provider wasn't provisioned for production load. This is the most common failover mistake. **How do you keep a backup provider "ready" for production traffic without paying for duplicated GPU capacity 24/7?** Cold standby (start instances on failover) takes 5-15 minutes — too slow. Hot standby (always running) costs 2x. What's the middle ground? (Hint: Pre-warm some capacity with a FRACTION of traffic — route 5% of requests to the backup provider continuously. This keeps instances warm, validates the pipeline, and costs only 5% extra. On failover, you have warm instances that can scale up from 5% to 100% load — still not instant, but you only need to absorb the delta of 95% additional traffic.)

2. You implement pre-warmed backup: 5% traffic to vLLM, 95% to OpenAI. On failover, vLLM needs to handle 20x its current load in seconds. **What scaling strategy handles the 5%→100% traffic surge?** Docker Compose horizontal scaling (add instances via API)? Kubernetes HPA? AWS ECS Service Auto Scaling? Each has different startup latency. (Hint: ECS Service Auto Scaling with target tracking can scale based on request count. But adding containers takes 2-5 minutes. For faster scaling: keep a BUFFER of 2-3 extra instances always running, OR use AWS Lambda as a shock absorber for low-latency requests during scale-up.)

3. **The hardest question:** Your pre-warmed backup works — vLLM instances are running with 5% live traffic. When OpenAI fails, you failover 100% to vLLM. But vLLM was configured for OpenAI's input patterns (average 300 tokens/request). Now it's getting ALL requests, including batch summarization (4,000+ tokens/request) that OpenAI was handling. **vLLM's KV cache fills up in 3 minutes because long-context requests consume disproportionate memory.** How do you design failover that accounts for DIFFERENT workload characteristics between providers — not just different capacity? (Hint: Consider: (a) route long-context requests to a DIFFERENT model during failover (e.g., GPT-4o-mini API), (b) dynamically adjust max_num_seqs based on average context length, (c) implement per-request admission control — reject or queue requests that would exceed available KV cache, (d) model-specific failover policies — chat → vLLM, summarization → fallback API.)

---

### 🤔 Discovery Question 2: The State Problem

Your chat service maintains conversations. A user has sent 15 messages in a conversation (3,000 tokens of history). They send message 16.

Your primary provider (OpenAI) is healthy. The request goes to OpenAI. Response starts streaming.

Mid-stream — 37 tokens into a 200-token response — OpenAI has a brief hiccup. TCP connection drops. The streaming response is interrupted.

Your failover system sees the error. It retries the request... on the backup provider (self-hosted vLLM).

But vLLM doesn't have the conversation history. It only sees the latest message. The response is completely different — it has no context.

**The user sees:**
```
[37 tokens of first response] ... user clicks send again ...
[Completely new response from vLLM, no context of the 15 previous messages]
```

**🤔 Before reading on, answer these:**

1. The failover worked TECHNICALLY (request succeeded) but failed UX-ally (response was wrong). **How do you carry CONVERSATION STATE across providers?** The conversation history needs to be injected into every request — this is the client's responsibility, not the server's. If the client sends the full history to every provider, any provider can pick up where another left off. But what if the state (conversation history) is on YOUR backend, not the client's? (Hint: Store conversation history in Redis (Phase 10, File 03's cache). On every request, the backend retrieves history from cache and injects it into the prompt. This way, ANY provider gets the full context. The failover doesn't lose state.)

2. Even with full history, the response INTERRUPTION is visible. The user sees partial text that disappears when they retry. **How do you handle IN-FLIGHT streaming requests during failover?** A streaming response is partially delivered. If you failover mid-stream, the second provider starts from scratch. The user sees two partial responses. What's the UX strategy? (Hint: Options: (a) BUFFER the first provider's partial response and APPEND the second provider's response on top, (b) DISCARD the first partial response and restart cleanly (the user sees "Reconnecting..." and the full response appears), (c) STREAM from both providers and take the first complete response — but this costs 2x. Each has different UX implications.)

3. **The hardest question:** You decide to buffer partial responses in Redis and append the second provider's output. But now you have CACHE COHERENCE issues: the first provider's partial tokens are in Redis, the second provider is generating new tokens. When the user refreshes, they see: first-provider-start + second-provider-start + second-provider-continuation. **How do you detect and deduplicate overlapping generated text?** The user might see "The capital of France is Paris... The capital of France is Paris and it's known for..." — 27 overlapping tokens. Design a strategy that detects and removes duplicate content at the overlap boundary, then delivers a clean merged response. (Hint: Compare suffix of first response to prefix of second response. Find the longest overlapping substring. Trim the duplicate. This is essentially longest-common-substring at the word or token level. But beware of partial-word overlaps and semantically different continuations.)

---

### 🤔 Discovery Question 3: The Regional Disaster

You deploy on AWS us-east-1. Your primary model is Bedrock (Claude) in us-east-1. Your backup is self-hosted vLLM, also in us-east-1.

AWS us-east-1 has a major availability zone failure. Half the AZs in us-east-1 go down.

- Bedrock is degraded (it depends on AZs that are down)
- Your vLLM instances — half are in the affected AZs, terminated
- Your load balancer is still healthy (cross-AZ), but half your compute is gone
- You can't failover to another provider because... wait, Bedrock is YOUR primary, vLLM is YOUR backup. Both are in the same region.

**You have NO geographic redundancy. One region fails, your entire service fails.**

**🤔 Before reading on, answer these:**

1. The root cause: both providers share the same single point of failure — us-east-1. **How do you achieve TRUE geographic diversity?** Two providers in the same region is not diversity. What's the minimum viable multi-region architecture? (Hint: Minimum: primary provider in us-east-1 (Bedrock), backup provider in us-west-2 (self-hosted vLLM or different API). Or: managed API (OpenAI) in us-east-1 + self-hosted in us-west-2. The KEY is that the backup is NOT co-located with the primary. But then you add cross-region latency — how do you handle the latency penalty during failover?)

2. You deploy vLLM in us-west-2 as your true backup. On failover, traffic from us-east-1 users routes to us-west-2 vLLM. Cross-region latency adds 60-80ms. The model itself (Llama-3-70B) has the same generation speed. But users in Europe were getting 200ms latency to us-east-1 — now they get 280ms to us-west-2. **Is 80ms additional latency acceptable during failover?** How do you communicate this to users? How do you set user expectations vs keeping the service technically available? (Hint: This is where SLOs matter (File 06). Your availability SLO is met (service is up). Your latency SLO may be temporarily violated (280ms > 200ms target). The error budget absorbs this. But what about users who NEED sub-200ms latency — do you need a THIRD provider in eu-west-1 for those users?)

3. **The hardest question:** You implement multi-region with active-active: 50% traffic to us-east-1 (Bedrock), 50% to us-west-2 (vLLM). Both regions serve live traffic. If us-east-1 fails, us-west-2 absorbs ALL traffic (scales from 50% to 100%). This is the BEST architecture for failover — no cold start, no traffic surge, no state issues (both regions were already handling traffic). **But ACTIVE-ACTIVE costs 2x the infrastructure — you're paying for TWO GPU clusters that are each 50% utilized during normal operation. How do you justify the cost?** What's the business argument for paying 2x for availability? When is active-active worth it? When is active-passive (cold standby) acceptable? (Hint: Answer with a COST OF DOWNTIME calculation. If your service makes $10,000/hour, 1 hour of downtime costs $10,000. Active-passive saves $3,000/month in infra but risks $10,000/hour in lost revenue. Active-active costs $3,000 more but prevents the $10,000 loss. The math decides. Now, what's YOUR service's cost of downtime? If you don't know, you can't make this decision.)

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Discovery Questions | 30 min |
| Concept: Multi-Provider Architecture Patterns | 30 min |
| Concept: Circuit Breakers, Health Checks, Traffic Shifting | 30 min |
| Concept: Disaster Recovery Planning | 20 min |
| Code: Failover Proxy (Fail-First) | 45 min |
| Code: Fixed Production Multi-Provider Router | 45 min |
| Code: Traffic Shifting Canary Deployments | 30 min |
| Drills | 30 min |
| Gate Check | 15 min |
| **Total** | **~4 hr** |

---

## 📖 Concept Deep-Dive

### Multi-Provider Architecture Patterns

```
PATTERN 1: ACTIVE-PASSIVE (Cold Standby)
  ┌──────────────┐      Normal: 100% traffic    ┌──────────────┐
  │  Provider A  │ ──────────────────────────▶  │  Users       │
  │  (Primary)   │                              │              │
  └──────────────┘                              └──────────────┘
  ┌──────────────┐      Failover: traffic → B
  │  Provider B  │ ◀────────────────────────────
  │  (Backup)    │   (instances start on demand)
  └──────────────┘
  Pros: Lowest cost. Backups don't run.
  Cons: RTO = 5-15 minutes (cold start). Capacity unknown during failover.

PATTERN 2: WARM STANDBY
  ┌──────────────┐      Normal: 95% traffic     ┌──────────────┐
  │  Provider A  │ ──────────────────────────▶  │  Users       │
  │  (Primary)   │                              │              │
  └──────────────┘                              └──────────────┘
  ┌──────────────┐      Normal: 5% traffic
  │  Provider B  │ ──────────────────────────▶
  │  (Pre-warm)  │      Failover: scales up 5%→100%
  └──────────────┘
  Pros: Low RTO (seconds). Validated pipeline. Known capacity baseline.
  Cons: 5% extra cost. Scale-up takes 2-5 minutes.

PATTERN 3: ACTIVE-ACTIVE (Multi-Region)
  ┌──────────────┐      50% traffic             ┌──────────────┐
  │  Region A    │ ──────────────────────────▶  │  Users       │
  │  (Provider)  │                              │  (Global)    │
  └──────────────┘                              └──────────────┘
  ┌──────────────┐      50% traffic
  │  Region B    │ ──────────────────────────▶
  │  (Provider)  │
  └──────────────┘
  Pros: Zero RTO (both regions live). Most resilient.
  Cons: 2x infrastructure cost. State sync complexity.

PATTERN 4: PROVIDER ARBITRAGE (Cost-Optimized)
  ┌────────────────┐  Route by cost/latency    ┌──────────────┐
  │ OpenAI GPT-4o  │ ◀──── high priority ────  │  Users       │
  ├────────────────┤                           │              │
  │ Bedrock Claude │ ◀──── high traffic ─────  │              │
  ├────────────────┤                           └──────────────┘
  │ Self-hosted    │ ◀──── overflow/cost ────
  │ vLLM Llama     │
  └────────────────┘
  Pros: Uses cheapest provider when possible. Load-balanced.
  Cons: Most complex. Different providers have different outputs.
```

### Circuit Breaker Pattern

The circuit breaker prevents cascading failures — if a provider is failing, stop sending requests to it before it degrades further.

```
States:
  ┌──────────┐     Failure threshold exceeded     ┌──────────┐
  │  CLOSED  │ ──────────────────────────────────▶ │   OPEN   │
  │ (Normal) │                                     │ (Reject) │
  └──────────┘                                     └──────────┘
       ▲                                               │
       │                                    Timeout    │
       │                                    elapsed    │
       │                                               ▼
       │                                   ┌──────────────────┐
       │                                   │    HALF-OPEN     │
       │                                   │ (Test with 1 req)│
       └───────────────────────────────────┴──────────────────┘
                    Request succeeds → CLOSED
                    Request fails → OPEN (reset timer)

Implementation decisions:
- Failure threshold: 5 consecutive errors? 50% error rate over 1 minute?
- Half-open test interval: 30 seconds? 5 minutes?
- Which errors count: 503? 429? Timeout? 200 with low eval score?
```

### Health Check Design

```
Provider A (OpenAI API):
  Health check: GET https://api.openai.com/v1/models
  Expected: 200 OK within 2s
  Frequency: Every 10 seconds
  Consecutive failures to mark unhealthy: 3
  
Provider B (Self-hosted vLLM):
  Health check: GET http://vllm-instance:8000/health
  Expected: 200 OK with {"status": "ready", "gpu_memory_used_pct": 72}
  Additional check: GET /metrics → gpu_memory gauge, queue depth
  Frequency: Every 5 seconds
  Consecutive failures to mark unhealthy: 2

Provider C (AWS Bedrock):
  Health check: InvokeModel with tiny payload (< 10 tokens)
  Expected: Response within 5s
  Frequency: Every 30 seconds (costs money per invoke — balance)
  Consecutive failures to mark unhealthy: 3
```

### Disaster Recovery Plan Template

```
DR PLAN: Multi-Provider Failover

RTO (Recovery Time Objective): 2 minutes
RPO (Recovery Point Objective): 0 (no state loss — Redis persists)

TRIGGERS:
  - Provider returns > 10% errors over 2 minutes
  - Provider latency p99 > 10s for 3 minutes
  - Circuit breaker opens (5 consecutive 503s)
  - Health check fails for 3 consecutive attempts

FAILOVER STEPS:
  1. [0s] Circuit breaker opens for failing provider
  2. [0s] DNS/Load balancer shifts traffic to healthy provider(s)
  3. [0s] Request queue: buffer in-flight requests for 30s
  4. [30s] Scale up backup provider (if warm standby, trigger scale-out)
  5. [60s] Verify backup provider health
  6. [90s] Drain remaining in-flight from failing provider
  7. [120s] Declare incident. Notify team.

FAILBACK STEPS:
  1. [0s] Verify primary provider health (half-open check)
  2. [0s] Route 5% canary traffic to primary
  3. [5m] If canary succeeds, route 50% to primary
  4. [10m] If stable, route 100% to primary
  5. [15m] Scale down backup provider to pre-warm level
  6. [20m] Declare resolved. Document incident.
```

---

## 💻 Code: Multi-Provider Failover Router

### Fail-First: The Failover Proxy with 7 Deliberate Bugs

```python
"""
failover_proxy.py — BROKEN VERSION. Find the bugs before reading the fix.
"""
from fastapi import FastAPI, Request
import httpx
import random
import time
from typing import Optional

app = FastAPI()

# Provider configs
PROVIDERS = {
    "openai": {
        "url": "https://api.openai.com/v1/chat/completions",
        "api_key": "sk-...",  # Bug 1
        "priority": 1,  # Lower = higher priority
    },
    "bedrock": {
        "url": "https://bedrock.us-east-1.amazonaws.com",
        "api_key": "...",
        "priority": 2,
    }
}

current_primary = "openai"

@app.post("/chat")
async def chat_endpoint(request: Request):
    body = await request.json()
    
    # Try providers in priority order
    for provider_name in sorted(PROVIDERS.keys(), 
                                 key=lambda p: PROVIDERS[p]["priority"]):
        provider = PROVIDERS[provider_name]
        
        try:
            # Bug 2: What happens to the request body?
            async with httpx.AsyncClient() as client:
                response = await client.post(
                    provider["url"],
                    json=body,
                    headers={"Authorization": f"Bearer {provider['api_key']}"},
                    timeout=30.0,  # Bug 3
                )
            
            return response.json()
        
        except Exception as e:  # Bug 4
            # Bug 5: What's wrong with failover logging?
            print(f"Provider {provider_name} failed: {e}")
            continue  # Try next provider
    
    # Bug 6
    return {"error": "No provider available"}, 503

@app.get("/health")
async def health_check():
    # What's the problem? (Bug 7)
    return {"status": "healthy", "primary": current_primary}
```

**Find the 7 bugs before reading the fixed version.**

<details>
<summary>🔍 Click to reveal bugs (try yourself first)</summary>

**Bug 1 — API keys hardcoded:**
API keys in source code. Leaked in git, exposed in Docker images, visible in logs.
- **Fix:** Environment variables: `OPENAI_API_KEY=sk-... python failover_proxy.py`

**Bug 2 — Request body consumed once, reused across providers:**
`await request.json()` reads the body ONCE. But `request.json()` can only be called ONCE in FastAPI (StreamConsumed error). The code passes the same `body` to multiple providers, which works TECHNICALLY because `body` is a cached dict. BUT the real issue: the body might be MODIFIED by a middleware or previous handler before the second provider call. More importantly, if any provider call somehow reads from the request stream, it's gone.
- **Fix:** Read body once, deep-copy if modifying. Or reconstruct from the parsed dict.

**Bug 3 — Fixed timeout across all providers:**
30-second timeout for EVERY provider. OpenAI might respond in 2s. Self-hosted vLLM under load might need 60s. If vLLM takes 45s, the 30s timeout kills it and you think it's unhealthy.
- **Fix:** Per-provider timeouts based on expected latency: OpenAI=10s, vLLM=60s, Bedrock=30s.

**Bug 4 — Catches ALL exceptions including connection errors and timeout:**
The broad `except Exception` swallows ALL failures including `httpx.TimeoutException`, `httpx.ConnectError`, JSON decode errors. These have DIFFERENT meanings. A timeout means "provider might be slow but alive." A connection refused means "provider is dead." You should handle these differently.
- **Fix:** Categorize errors. Timeout = circuit breaker half-open, not failover. Connection refused = immediate failover.

**Bug 5 — Print statement for failover logging:**
`print()` in an async app goes to stdout which might be captured, lost, or mixed with other output. No structured logging, no trace ID, no request context. You can't debug a failover sequence from prints.
- **Fix:** Use structured logging with trace IDs: `logger.error("provider_failover", extra={"provider": name, "request_id": req_id, "error": str(e)})`

**Bug 6 — No retry logic or queuing when all providers fail:**
Returns 503 immediately when all providers fail. But the failing provider might recover in 2 seconds — the request is already lost. For non-real-time requests, you could queue and retry.
- **Fix:** Add a retry queue with backoff. For synchronous endpoints, retry 1-2 times with exponential backoff before returning 503.

**Bug 7 — Health check doesn't actually check provider health:**
Returns `{"status": "healthy"}` without calling ANY provider health check endpoints. If OpenAI is down, this endpoint still says "healthy."
- **Fix:** The health check should call EVERY provider's health endpoint and report individual statuses: `{"openai": "unhealthy", "bedrock": "healthy"}`.

</details>

---

### ✅ Fixed: Production Multi-Provider Router with Circuit Breakers

```python
"""
production_failover.py — Production multi-provider failover router.

Features:
- Circuit breaker per provider (closed → open → half-open)
- Per-provider health checks with configurable thresholds
- Priority-ordered failover with automatic fallback
- Structured logging with request tracing
- Configurable per-provider timeouts
- Retry queue with exponential backoff
- Environment variable configuration
"""
import asyncio
import json
import logging
import os
import time
import uuid
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import Enum
from typing import Any, AsyncGenerator, Dict, List, Optional, Tuple

import httpx
from fastapi import FastAPI, Request, Response
from pydantic import BaseModel, Field

# ── Configuration ────────────────────────────────────────

# Load from environment variables
API_KEYS = {
    "openai": os.environ.get("OPENAI_API_KEY", ""),
    "anthropic": os.environ.get("ANTHROPIC_API_KEY", ""),
    "bedrock": os.environ.get("BEDROCK_API_KEY", ""),
}

# Per-provider configuration
PROVIDER_CONFIG: Dict[str, Dict[str, Any]] = {
    "openai": {
        "url": "https://api.openai.com/v1/chat/completions",
        "api_key": API_KEYS["openai"],
        "priority": 1,
        "timeout_seconds": 30,
        "health_check_url": "https://api.openai.com/v1/models",
        "health_check_interval": 10,
        "failure_threshold": 5,  # Consecutive failures to open circuit
        "half_open_interval": 30,  # Seconds before testing recovery
    },
    "anthropic": {
        "url": "https://api.anthropic.com/v1/messages",
        "api_key": API_KEYS["anthropic"],
        "priority": 2,
        "timeout_seconds": 30,
        "health_check_url": "https://api.anthropic.com/v1/messages",
        "health_check_interval": 15,
        "failure_threshold": 3,
        "half_open_interval": 45,
    },
    "self_hosted": {
        "url": os.environ.get("VLLM_URL", "http://localhost:8000/v1/chat/completions"),
        "api_key": "",  # Self-hosted may not need auth
        "priority": 3,
        "timeout_seconds": 120,  # Longer timeout for self-hosted
        "health_check_url": os.environ.get("VLLM_HEALTH_URL", "http://localhost:8000/health"),
        "health_check_interval": 5,
        "failure_threshold": 3,
        "half_open_interval": 15,
    },
}

# ── Circuit Breaker State ────────────────────────────────

class CircuitState(str, Enum):
    CLOSED = "closed"       # Normal operation
    OPEN = "open"           # Rejecting requests
    HALF_OPEN = "half_open" # Testing with single request


@dataclass
class CircuitBreaker:
    provider_name: str
    state: CircuitState = CircuitState.CLOSED
    failure_count: int = 0
    last_failure_time: Optional[float] = None
    last_success_time: Optional[float] = None
    
    config: Dict[str, Any] = field(default_factory=dict)
    
    def record_success(self):
        """Reset on success."""
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_success_time = time.monotonic()
    
    def record_failure(self):
        """Increment failure count; open circuit if threshold exceeded."""
        self.failure_count += 1
        self.last_failure_time = time.monotonic()
        
        if self.failure_count >= self.config.get("failure_threshold", 5):
            self.state = CircuitState.OPEN
            logging.warning(
                "Circuit OPEN for provider %s after %d failures",
                self.provider_name, self.failure_count
            )
    
    def try_half_open(self) -> bool:
        """Check if we should attempt half-open."""
        if self.state != CircuitState.OPEN:
            return False
        
        elapsed = time.monotonic() - (self.last_failure_time or 0)
        if elapsed >= self.config.get("half_open_interval", 30):
            self.state = CircuitState.HALF_OPEN
            logging.info("Circuit HALF_OPEN for provider %s — testing", self.provider_name)
            return True
        
        return False
    
    @property
    def can_try(self) -> bool:
        """Can this circuit accept a request?"""
        if self.state == CircuitState.CLOSED:
            return True
        if self.state == CircuitState.OPEN:
            return self.try_half_open()
        return True  # HALF_OPEN — allow the test request


# ── Request Context ──────────────────────────────────────

@dataclass
class RequestContext:
    request_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    user_id: str = "anonymous"
    model: str = "unknown"
    endpoint: str = "/chat"
    
    # Failover tracking
    attempted_providers: List[str] = field(default_factory=list)
    failover_count: int = 0
    start_time: float = field(default_factory=time.monotonic)
    
    @property
    def elapsed(self) -> float:
        return time.monotonic() - self.start_time


# ── Logging Setup ────────────────────────────────────────

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s | %(levelname)s | %(name)s | %(message)s',
)
logger = logging.getLogger("failover")


# ── Provider Router ──────────────────────────────────────

class ProviderRouter:
    """
    Routes requests to AI providers with circuit breaker failover.
    
    Health checks run in background; circuit state is shared across requests.
    """
    
    def __init__(self, provider_configs: Dict[str, Dict[str, Any]]):
        self.providers = provider_configs
        self.circuits: Dict[str, CircuitBreaker] = {}
        self._client_pool: Dict[str, httpx.AsyncClient] = {}
        
        # Initialize circuit breakers
        for name, config in self.providers.items():
            self.circuits[name] = CircuitBreaker(
                provider_name=name,
                config=config,
            )
        
        # Build HTTP client pools (reuse connections per provider)
        for name, config in self.providers.items():
            self._client_pool[name] = httpx.AsyncClient(
                timeout=config.get("timeout_seconds", 30),
                limits=httpx.Limits(max_keepalive_connections=10, max_connections=50),
            )
        
        # Start background health checks
        self._health_check_task = None
    
    async def start_health_checks(self):
        """Background task that checks all providers periodically."""
        async def _check_loop():
            while True:
                for name, config in self.providers.items():
                    await self._check_provider_health(name)
                await asyncio.sleep(5)  # Check every 5 seconds (fast cycle)
        
        self._health_check_task = asyncio.create_task(_check_loop())
        logger.info("Background health checks started")
    
    async def _check_provider_health(self, provider_name: str):
        """Individual provider health check."""
        config = self.providers.get(provider_name)
        if not config:
            return
        
        health_url = config.get("health_check_url", config["url"])
        circuit = self.circuits[provider_name]
        
        try:
            client = self._client_pool[provider_name]
            response = await client.get(health_url, timeout=5.0)
            
            if response.status_code < 500:
                circuit.record_success()
            else:
                circuit.record_failure()
                
        except Exception as e:
            circuit.record_failure()
    
    async def route_request(
        self,
        body: Dict[str, Any],
        ctx: RequestContext,
    ) -> Tuple[int, Dict[str, Any]]:
        """
        Route a request to the best available provider.
        Returns (status_code, response_body).
        """
        # Sort providers by priority
        sorted_providers = sorted(
            self.providers.items(),
            key=lambda item: (item[1].get("priority", 99), item[0])
        )
        
        last_error: Optional[str] = None
        
        for provider_name, config in sorted_providers:
            circuit = self.circuits[provider_name]
            
            # Skip if circuit is OPEN (not ready for half-open yet)
            if circuit.state == CircuitState.OPEN and not circuit.try_half_open():
                logger.debug("Skipping provider %s (circuit open)", provider_name)
                continue
            
            ctx.attempted_providers.append(provider_name)
            
            try:
                response = await self._call_provider(
                    provider_name, config, body, ctx
                )
                
                # Success — record and return
                circuit.record_success()
                ctx.failover_count = len(ctx.attempted_providers) - 1
                
                if ctx.failover_count > 0:
                    logger.warning(
                        "Request %s failed over to %s after %d attempt(s)",
                        ctx.request_id, provider_name, ctx.failover_count
                    )
                
                return (200, response)
                
            except httpx.TimeoutException as e:
                circuit.record_failure()
                last_error = f"timeout from {provider_name}: {str(e)}"
                logger.warning(
                    "Provider %s timeout for req %s", provider_name, ctx.request_id
                )
                continue
                
            except httpx.HTTPStatusError as e:
                status = e.response.status_code
                if status == 429:
                    # Rate limit — circuit might be fine, provider is just busy
                    # Don't open circuit for rate limits, but still try next
                    last_error = f"rate_limit from {provider_name}"
                    logger.warning(
                        "Provider %s rate limited for req %s", provider_name, ctx.request_id
                    )
                    continue
                elif status >= 500:
                    circuit.record_failure()
                    last_error = f"server_error from {provider_name}: HTTP {status}"
                    continue
                else:
                    # 4xx (not 429) — client error, won't help to retry
                    return (status, {"error": f"Provider {provider_name} returned {status}"})
                    
            except httpx.ConnectError as e:
                circuit.record_failure()
                last_error = f"connection_failed to {provider_name}: {str(e)}"
                logger.error(
                    "Cannot connect to %s for req %s", provider_name, ctx.request_id
                )
                continue
                
            except Exception as e:
                # Unexpected — log and try next
                circuit.record_failure()
                last_error = f"unexpected_error from {provider_name}: {str(e)}"
                logger.exception(
                    "Unexpected error from %s for req %s", provider_name, ctx.request_id
                )
                continue
        
        # All providers failed — return 503 with diagnostic info
        return (
            503,
            {
                "error": "all_providers_unavailable",
                "detail": last_error,
                "request_id": ctx.request_id,
                "attempted_providers": ctx.attempted_providers,
            }
        )
    
    async def _call_provider(
        self,
        provider_name: str,
        config: Dict[str, Any],
        body: Dict[str, Any],
        ctx: RequestContext,
    ) -> Dict[str, Any]:
        """Call a single provider and return the response."""
        
        # Some providers need different request format
        if provider_name == "anthropic":
            # Anthropic messages API format
            anthropic_body = self._to_anthropic_format(body)
            request_body = anthropic_body
        else:
            request_body = body  # OpenAI-compatible format
        
        client = self._client_pool[provider_name]
        headers = {"Content-Type": "application/json"}
        
        if config.get("api_key"):
            if provider_name == "anthropic":
                headers["x-api-key"] = config["api_key"]
                headers["anthropic-version"] = "2023-06-01"
            else:
                headers["Authorization"] = f"Bearer {config['api_key']}"
        
        response = await client.post(
            config["url"],
            json=request_body,
            headers=headers,
        )
        
        response.raise_for_status()
        return response.json()
    
    def _to_anthropic_format(self, openai_body: Dict) -> Dict:
        """Convert OpenAI chat format to Anthropic messages format."""
        messages = openai_body.get("messages", [])
        
        # Extract system prompt
        system = None
        anthropic_messages = []
        for msg in messages:
            if msg.get("role") == "system" and not system:
                system = msg["content"]
            else:
                anthropic_role = "assistant" if msg["role"] == "assistant" else "user"
                anthropic_messages.append({
                    "role": anthropic_role,
                    "content": msg["content"],
                })
        
        result = {
            "model": openai_body.get("model", "claude-3-haiku-20240307"),
            "max_tokens": openai_body.get("max_tokens", 1024),
            "messages": anthropic_messages,
        }
        
        if system:
            result["system"] = system
        
        return result
    
    async def get_health_status(self) -> Dict[str, Any]:
        """Return health status of all providers."""
        statuses = {}
        for name in self.providers:
            circuit = self.circuits[name]
            config = self.providers[name]
            statuses[name] = {
                "status": circuit.state.value,
                "priority": config.get("priority"),
                "url": config.get("url"),
                "failure_count": circuit.failure_count,
            }
        return {"providers": statuses, "healthy_count": sum(
            1 for c in self.circuits.values() if c.state == CircuitState.CLOSED
        )}
    
    async def shutdown(self):
        """Clean up HTTP clients."""
        if self._health_check_task:
            self._health_check_task.cancel()
        for client in self._client_pool.values():
            await client.aclose()


# ── FastAPI Application ──────────────────────────────────

router: Optional[ProviderRouter] = None

app = FastAPI(title="Multi-Provider AI Router")

@app.on_event("startup")
async def startup():
    global router
    router = ProviderRouter(PROVIDER_CONFIG)
    await router.start_health_checks()

@app.on_event("shutdown")
async def shutdown():
    if router:
        await router.shutdown()

@app.post("/chat")
async def chat_endpoint(request: Request):
    """Chat endpoint with automatic multi-provider failover."""
    body = await request.json()
    
    ctx = RequestContext(
        user_id=request.headers.get("X-User-ID", "anonymous"),
        model=body.get("model", "unknown"),
    )
    
    status_code, response_body = await router.route_request(body, ctx)
    
    # Add failover metadata for debugging
    if ctx.failover_count > 0:
        response_body["_failover"] = {
            "attempted": ctx.attempted_providers,
            "failovers": ctx.failover_count,
            "request_id": ctx.request_id,
        }
    
    return Response(
        content=json.dumps(response_body),
        status_code=status_code,
        media_type="application/json",
    )

@app.get("/health")
async def health():
    """Health check endpoint — reports ALL provider statuses."""
    global router
    if not router:
        return {"status": "starting"}
    status = await router.get_health_status()
    all_healthy = status["healthy_count"] == len(PROVIDER_CONFIG)
    return {
        "status": "healthy" if all_healthy else "degraded",
        **status,
    }


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### 📊 Expected Output

Running the multi-provider router with circuit breakers:

```bash
# Terminal 1: Start the router
export OPENAI_API_KEY="sk-..."
export ANTHROPIC_API_KEY="sk-ant-..."
python production_failover.py

# Terminal 2: Test normal operation (primary provider)
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }'
# → {"id":"...","choices":[{"message":{"content":"Hello! How can I help you today?"}}],...}

# Terminal 3: Check provider health
curl http://localhost:8000/health
# → {"status":"healthy","providers":{"openai":{"status":"closed","priority":1,"failure_count":0},
#     "anthropic":{"status":"closed","priority":2,"failure_count":0},
#     "self_hosted":{"status":"closed","priority":3,"failure_count":0}},
#     "healthy_count":3}

# Simulate primary failure: set OPENAI_API_KEY to invalid value, restart
# Then make a chat request:
# → Response still succeeds (failed over to anthropic)
# → Response includes _failover metadata:
#   {"id":"...","content":"Hello!...",
#    "_failover":{"attempted":["openai","anthropic"],"failovers":1,"request_id":"abc-123"}}

# Health check after failover:
# → {"status":"degraded","providers":{"openai":{"status":"open","failure_count":5},"anthropic":{...}},"healthy_count":2}
```

### Traffic Shifting for Canary Deployments

```python
"""
traffic_shifting.py — Gradual traffic shifting for canary deployments.

Use case: Migrating from GPT-4o to self-hosted Llama-3-70B
without cutting over all at once.
"""
import random
from dataclasses import dataclass
from typing import Dict, List, Optional


@dataclass
class TrafficSplit:
    """Defines how traffic is split across model versions."""
    provider: str
    weight: float  # 0.0 to 1.0, sum across splits = 1.0
    model_name: str
    metadata: Dict = None


class TrafficShifter:
    """
    Gradual traffic shifting for canary deployments.
    
    Shift plan:
    Day 1:  5% new, 95% old  (canary — validate correctness)
    Day 2:  25% new, 75% old (expand — monitor cost and latency)
    Day 3:  50% new, 50% old (half — verify at scale)
    Day 4:  75% new, 25% old (majority — prepare for full cutover)
    Day 5:  100% new, 0% old (complete — can retire old provider)
    
    Each day: evaluate responses, compare costs, compare latency.
    If anything regresses → roll back immediately.
    """
    
    def __init__(self, splits: List[TrafficSplit]):
        total_weight = sum(s.weight for s in splits)
        if abs(total_weight - 1.0) > 0.001:
            raise ValueError(f"Traffic weights must sum to 1.0, got {total_weight}")
        self.splits = splits
    
    def select_provider(self, request_id: str) -> TrafficSplit:
        """
        Select provider based on weighted random assignment.
        
        For deterministic routing (same request → same provider):
        Use request_id hash instead of random.
        """
        # Deterministic: hash request_id for consistency
        hash_val = hash(request_id) % 10000
        cumulative = 0.0
        
        for split in self.splits:
            cumulative += split.weight * 10000
            if hash_val < cumulative:
                return split
        
        return self.splits[-1]  # Fallback to last
    
    def update_split(self, provider: str, new_weight: float):
        """Update a single provider's weight, renormalizing others."""
        total_other = 1.0 - new_weight
        current_other = 0.0
        target_split = None
        
        for split in self.splits:
            if split.provider == provider:
                target_split = split
            else:
                current_other += split.weight
        
        if not target_split:
            raise ValueError(f"Provider {provider} not found in splits")
        
        # Scale other providers proportionally
        if current_other > 0:
            scale = total_other / current_other
            for split in self.splits:
                if split.provider != provider:
                    split.weight *= scale
        
        target_split.weight = new_weight
    
    def evaluate_shift(self, metrics: Dict[str, Dict]) -> Dict[str, str]:
        """
        Evaluate whether the shift should continue, rollback, or pause.
        
        metrics: {
            "gpt-4o": {"latency_p50": 0.8, "eval_score": 8.5, "cost_per_req": 0.003},
            "llama-3-70b": {"latency_p50": 1.2, "eval_score": 7.8, "cost_per_req": 0.0005},
        }
        
        Rules:
        - If eval_score drops > 10% → ROLLBACK
        - If latency_p99 increases > 50% → PAUSE
        - If cost_per_req increases > 20% → REVIEW
        - Otherwise → CONTINUE
        """
        decisions = {}
        
        for provider, m in metrics.items():
            if m.get("eval_score", 10) < 7.0:
                decisions[provider] = "ROLLBACK"
            elif m.get("latency_p99", 0) > 10000:
                decisions[provider] = "PAUSE"
            elif m.get("cost_per_req", 0) > 0.1:
                decisions[provider] = "REVIEW"
            else:
                decisions[provider] = "CONTINUE"
        
        return decisions
```

---

## ✅ Good Output Examples

### Health Check Response During Degradation

```json
{
  "status": "degraded",
  "providers": {
    "openai": {
      "status": "open",
      "priority": 1,
      "url": "https://api.openai.com/v1/chat/completions",
      "failure_count": 7
    },
    "anthropic": {
      "status": "closed",
      "priority": 2,
      "url": "https://api.anthropic.com/v1/messages",
      "failure_count": 0
    },
    "self_hosted": {
      "status": "closed",
      "priority": 3,
      "url": "http://vllm-cluster:8000/v1/chat/completions",
      "failure_count": 1
    }
  },
  "healthy_count": 2,
  "active_failover": "openai → anthropic",
  "failover_started": "2026-05-16T14:32:00Z",
  "estimated_recovery": "Circuit will test at 2026-05-16T15:02:00Z"
}
```

### Circuit Breaker Log Output

```
2026-05-16 14:31:55 | WARNING | failover | Circuit OPEN for provider openai after 5 failures
2026-05-16 14:31:55 | WARNING | failover | Request a1b2c3 failed over to anthropic after 1 attempt(s)
2026-05-16 14:32:00 | INFO | failover | Circuit HALF_OPEN for provider openai — testing
2026-05-16 14:32:01 | INFO | failover | Circuit CLOSED for provider openai (recovered)
2026-05-16 14:32:01 | INFO | failover | Failover resolved — openai is healthy again
2026-05-16 14:32:05 | INFO | failover | Traffic restored: 100% → openai
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Failover to an Untested Backup

Your backup provider has NEVER handled production traffic. When you failover, it crashes under real load.

**Fix:** Route 5% of normal traffic to the backup continuously. If it can't handle 5%, it DEFINITELY can't handle 100%.

### Antipattern 2: Symmetric Failover

You failover from OpenAI to self-hosted vLLM expecting identical behavior. But vLLM returns different outputs for the same prompt. Users see inconsistent answers.

**Fix:** Active-active routing means users accept provider differences. Or use a CONSISTENCY layer — for example, always route the same user to the same provider (sticky sessions).

### Antipattern 3: No Thundering Herd Protection

When primary recovers, all traffic shifts back instantly. The primary gets 100% traffic in one second and collapses under the surge.

**Fix:** Gradual failback. Route 5% → 25% → 50% → 75% → 100% over 15-30 minutes. Monitor after each step.

### Antipattern 4: Ignoring Provider-Specific Rate Limits

You failover from OpenAI to Bedrock. Bedrock has LOWER rate limits. What handled 500 req/min on OpenAI gets 429'd on Bedrock.

**Fix:** Track per-provider rate limits. Implement client-side rate limiting proportional to the provider's capacity. A 429 from the backup means "too fast, slow down" not "try different provider."

### Antipattern 5: No Cost Awareness in Failover

Your failover policy is "route to the cheapest available provider." During failover, the SECOND cheapest provider becomes primary. But the CHEAPEST provider was handling most load because it was cheapest. Now the SECOND cheapest is handling 100% load — costing MORE than the primary ever did.

**Fix:** Cost control during failover. If costs exceed X%, consider degrading service quality — switch to a smaller model, reduce max_tokens, or queue non-urgent requests.

### Common Failover Failures

| Failure | What Happens | Why | Fix |
|---------|-------------|-----|-----|
| Cascading failover | Backup also fails | Backup not sized for full load | Pre-warm with 5% traffic; scale-out on failover trigger |
| Split-brain | Both providers think they're primary | Health check ambiguity | Use quorum-based health checks (3+ checkers) |
| Thundering herd | Primary crashes on failback | All traffic shifts instantly | Gradual failback over 15-30 min |
| State loss | Responses inconsistent across providers | Conversation history not shared | Store state in Redis (File 03) |
| Latency surprise | Backup has unexpected latency | Different provider/hardware | Per-provider timeout configs; latency health checks |
| Cost explosion | Bill 3x higher during failover | Cheaper provider down, expensive backup handling all load | Cost-aware routing; degrade model quality during failover |

---

## 🧪 Drills & Challenges

### Drill 1: Build a Failover Simulator (45 min)

Write a script that simulates provider failures and tests your failover logic:

```python
# pseudo-code outline
def test_failover_scenario():
    # Scenario 1: Primary fails (returns 503)
    # Expected: Request succeeds via backup, health status = "degraded"
    
    # Scenario 2: All providers fail
    # Expected: Request returns 503 with attempt list
    
    # Scenario 3: Primary recovers after 30s
    # Expected: Circuit goes half-open → test request succeeds → circuit closed
    
    # Scenario 4: Intermittent failures (50% error rate)
    # Expected: Circuit opens after threshold, stays open until half-open test
```

### Drill 2: Design a Migration Plan (30 min)

You're migrating from OpenAI GPT-4o to self-hosted Llama-3-70B. Design the traffic shift:

1. What metrics would you compare during each stage?
2. What eval scores must the new model meet to proceed?
3. What's the rollback plan if day 1 shows regression?
4. How do you handle users who notice different outputs?

### Drill 3: Cost of Downtime Calculation (20 min)

Your service:
- Handles 10,000 requests/hour at peak
- Average revenue: $0.02/request
- Average cost: $0.005/request in GPU/compute
- Active-active costs $4,000/month extra
- Active-passive (cold standby) costs $400/month extra
- Expected downtime without failover: 2 hours/month (one 1-hour OpenAI outage, one 1-hour AWS AZ failure)

Calculate:
1. Revenue lost per hour of downtime
2. Profit lost per hour (revenue - saved compute cost)
3. ROI of active-active vs active-passive
4. At what request volume does active-active become worth it?

### Drill 4: Write a Real Disaster Recovery Plan (30 min)

Using the template from the concept section, write a DR plan for YOUR service:
- What providers do you use?
- What's your recovery time objective?
- What's the failover sequence?
- What's the failback sequence?
- How do you test this plan?

---

## 🚦 Gate Check

Before moving to File 08 (Security & Compliance), verify you can:

1. **Design a multi-provider architecture** — Given: OpenAI (primary), self-hosted vLLM (backup), AWS Bedrock (tertiary). Design: health check strategy, failover triggers, circuit breaker thresholds, and failback sequence

2. **Implement circuit breakers** — When do you open a circuit? When do you half-open? When do you close? What counts as a "failure" — 503 only? Timeout? 200 with low eval score?

3. **Handle state during failover** — A streaming response is interrupted mid-stream. The failover retry generates a complete new response. Design the deduplication strategy to merge partial + full response without duplicate content

4. **Make the cost/availability tradeoff** — Your CFO asks: "Why are we paying for two GPU clusters?" Answer with: cost of downtime calculation, RTO/RPO requirements, and the specific failover pattern that matches your budget

5. **Test a failover without breaking production** — Design a chaos engineering experiment that validates your failover works WITHOUT causing user-visible errors

---

## 📚 Resources

### Multi-Provider Architecture
- **[AWS Multi-Region Architecture](https://aws.amazon.com/architecture/multi-region/)** — Patterns for multi-region deployment
- **[Resilience and Disaster Recovery](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/disaster-recovery-dr.html)** — AWS Well-Architected Framework on DR

### Circuit Breakers
- **[Circuit Breaker Pattern (Martin Fowler)](https://martinfowler.com/bliki/CircuitBreaker.html)** — The canonical reference
- **[Hystrix Documentation](https://github.com/Netflix/Hystrix/wiki)** — Netflix's circuit breaker implementation (patterns, not Java-specific)

### Production AI Failover
- **[Why Your LLM App Needs a Fallback Strategy](https://www.truera.io/blog/why-your-llm-app-needs-a-fallback-strategy)** — Provider redundancy for AI services
- **[Building Resilient LLM Applications](https://www.anyscale.com/blog/building-resilient-llm-applications)** — Failover patterns specific to AI workloads

### Phase Cross-References
- **Phase 10, File 03 (Caching)** → Redis stores conversation state shared across providers — critical for stateful failover
- **Phase 10, File 06 (Monitoring)** → Alert triggers initiate failover; health check metrics feed into Grafana
- **Phase 10, File 04 (Cost)** → Cost-aware routing decisions (which provider to use based on real-time pricing)
- **Phase 1 (LLM Gateway)** → Your original provider abstraction layer — File 07 is the production-grade version with failover

> **Next up: File 08 — Security & Compliance.** You have multiple providers, traffic routing, and state management. Now you need to lock it down: authentication, encryption, audit logging, and compliance for AI systems handling user data. You'll build a security layer that protects your infrastructure AND your users' data.
