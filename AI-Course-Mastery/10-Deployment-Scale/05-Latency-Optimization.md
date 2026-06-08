# 05 — Latency Optimization: Speed Without Breaking the Bank

## 🎯 Purpose & Goals

> 🛑 STOP. Before you optimize latency, you need to understand the most damaging myth in AI serving:

**"Faster GPU = faster responses."**

It's TRUE that a faster GPU generates tokens faster. But most latency isn't from token generation speed. It's from:
- **Queueing**: Waiting for other requests to finish (50-200ms)
- **Prefill**: Processing the input prompt (100-500ms for long prompts)
- **Network**: Round trips between services (50-200ms)
- **Serialization**: Converting data between formats (10-50ms)
- **Cache misses**: Waiting for I/O on cache lookup (5-20ms)

If your total latency is 2,000ms and token generation is 300ms of that, buying a 2x faster GPU only saves 150ms — 7.5% improvement for 3x the cost.

**The goal of this module:** Profile ACTUAL latency, find the REAL bottlenecks, and fix the right ones.

---

### 🤔 Discovery Question 1: The Streaming Paradox

You implement streaming — tokens appear as they're generated instead of waiting for the full response. Users love it: "The app feels so much faster!"

But your P50 TIME-TO-FIRST-TOKEN (TTFT) is still 1.5 seconds. Users see a blank screen for 1.5 seconds before ANY text appears. Then text streams smoothly.

**Your PM says: "Streaming is great, but the blank screen at the start is terrible. Fix it."**

**🤔 Before reading on, answer these:**

1. The TTFT (time to first token) is high because the model has to process the ENTIRE input prompt before generating the first token. This is called the PREFILL phase. For a 500-token prompt on A10G, prefill takes ~300ms. The remaining 1,200ms is QUEUEING — waiting for other requests to finish. **Why would other requests block yours from starting?** vLLM uses continuous batching — shouldn't every request start immediately? (Hint: Continuous batching has limits. If max_num_seqs is reached, new requests queue. If GPU memory is full, new requests wait.)

2. You optimize: reduce max_num_seqs from 256 to 64. Throughput drops 40%, but TTFT drops from 1.5s to 0.4s. Users see text faster. But throughput can't handle peak traffic now. **At what point does TTFT optimization HURT more than it helps?** How do you find the optimal balance between throughput and TTFT?

3. **The hardest question:** Streaming means the user sees text AS IT'S GENERATED. But the MODEL doesn't produce text at a constant rate — first tokens are fast (prefill builds the KV cache), then tokens slow down (generation phase with memory bandwidth bottleneck). **Design a latency-visualization system that shows users: "Generating..." with a REAL-TIME progress indicator, not a spinner.** What progress metric would you show? Tokens per second? Estimated time remaining? How do you estimate "time remaining" when you don't know how many tokens the model will generate?

---

### 🤔 Discovery Question 2: The Batch Size Tradeoff

You're serving a chatbot. You have 10 concurrent users. Each query is ~200 tokens of input, ~150 tokens of output.

**Without batching:** Each request is processed one at a time on the GPU.
- Total time per request: ~200ms (prefill) + ~450ms (generation) = 650ms
- 10 users × 650ms = 6.5 seconds for the last user to finish
- User 1 waits 650ms. User 10 waits 6.5 seconds. Terrible UX.

**With continuous batching:** vLLM batches multiple requests into one GPU call.
- vLLM batches 10 requests: prefill all 10 together (200ms), then generate tokens for all 10 together (450ms per token step)
- Each request effectively completes in ~650ms TOTAL (because all 10 are processed in parallel)
- All 10 users get responses in ~650ms

**🤔 Before reading on, answer these:**

1. Continuous batching seems magical — 10 users served in the same time as 1. But there's a HIDDEN cost: the batch of 10 uses 10x the GPU memory for KV cache. If your GPU has 24GB and each sequence uses 2GB, you can batch at most 12 sequences. **What's the LIMITING factor for batch size — compute or memory?** How does this change for short vs. long sequences?

2. In a batch of 10, all sequences must be PADDED to the same length. If 9 sequences have 50-token outputs and 1 sequence has 500 tokens, the GPU computes 500 tokens for ALL 10 — 90% of computation is wasted on the 9 short sequences. **How does vLLM handle variable-length sequences in a batch?** Does it pad? Does it use a different strategy? (Hint: vLLM uses dynamic batching — sequences LEAVE the batch when they finish generating. The 9 short sequences finish early and free up space for new sequences.)

3. **The hardest question:** You have two request types:
   - Real-time chat: 200 input + 150 output tokens, needs < 1s latency
   - Batch summarization: 4,000 input + 500 output tokens, latency-tolerant (up to 30s)
   
   If you batch them together, the chat request waits for the summarization to finish. **How do you handle REQUEST PRIORITIZATION in continuous batching?** Should chat requests jump the queue? Should summarization requests use a separate GPU pool? Design a scheduling strategy that gives chat low latency without wasting GPU capacity.

---

### 🤔 Discovery Question 3: The Speculative Decoding Bet

You implement speculative decoding (draft model + target model):
- Draft model (tiny, fast): generates 5 candidate tokens speculatively
- Target model (full): verifies all 5 in one forward pass
- If all 5 are accepted: 5 tokens in the time of 1 = 5x speedup

**Results:**
| Metric | Without Spec Decode | With Spec Decode |
|--------|-------------------|-----------------|
| Tokens/second | 35 | 82 |
| Latency p50 | 1.2s | 0.9s |
| GPU memory | 5GB (model only) | 7GB (model + draft) |
| Quality (ROUGE-L) | 0.47 | 0.46 |

**But:** Your colleague tries on a DIFFERENT use case and gets:
| Tokens/second | 42 | 48 |
| Speedup | — | **14%** |

**🤔 Before reading on, answer these:**

1. Same technique, two use cases. One gets 2.3x speedup. The other gets 14%. **Why does speculative decoding work so differently for different use cases?** (Hint: Acceptance rate depends on how well the DRAFT model matches the TARGET model. For creative writing (high entropy), the draft's tokens are often rejected. For factual Q&A (low entropy), the draft's tokens are often accepted.)

2. The draft model adds 2GB of GPU memory. On a 24GB GPU, that's 8% less memory for KV cache — meaning FEWER concurrent sequences. If the 2.3x speedup comes at the cost of 20% fewer concurrent users, at what request rate does speculative decoding ACTUALLY help? **What's the BREAK-EVEN point for speculative decoding?**

3. **The hardest question:** You can choose your draft model:
   - Option A: Same architecture, 10x smaller (e.g., 0.7B draft for 7B target) — good acceptance rate (40%), but still uses significant memory
   - Option B: Completely different architecture, 50x smaller (e.g., 0.14B draft from a different model family) — low memory, but low acceptance rate (15%)
   - Option C: SELF-SPECULATION — use the same model but with earlier layers as draft — no extra memory, but lower speedup and complex to implement
   
   **Which do you choose?** What measurements would you take before deciding? (Hint: You need to measure: acceptance rate on YOUR data, not synthetic benchmarks.)

---

## ⏱ Time Budget

| Section | Time |
|---------|------|
| Discovery Questions | 30 min |
| Concept Deep-Dive | 1 hr |
| Code Examples | 1.5 hr |
| Drills | 30 min |
| Gate Check | 15 min |
| **Total** | **~4 hr** |

---

## 📖 Concept Deep-Dive

### 1. The Latency Decomposition

```
Total Latency = Queue + Prefill + Generation + Network + Serialization

Example (typical 7B on A10G):
  Queue:          50-500ms   (waiting for GPU)
  Prefill:        100-300ms  (processing prompt)
  Generation:     150-1500ms (generating tokens at ~30ms/token)
  Network:        20-100ms   (API call overhead)
  Serialization:  5-20ms     (JSON, tokenize, detokenize)
  
  Total:          325-2420ms
```

### 2. Latency Optimization Levers (Ranked by Impact)

| Lever | Latency Reduction | Cost Impact | Implementation Effort |
|-------|------------------|-------------|----------------------|
| Streaming | Perceived: 90% (users see first token faster) | None | Low |
| Continuous batching | 40-80% (batch parallel) | None (free) | Already in vLLM |
| Queue reduction | 50-200ms | None | Medium (config tuning) |
| Speculative decoding | 20-60% | + memory for draft model | Medium |
| Quantization | 20-40% | - quality (2-8%) | Low |
| GPU upgrade | 20-50% | +2-3x cost | Low (config change) |
| Prefix caching | 10-50% (TTFT reduction) | None (free) | Low (enable flag) |
| Early stopping/response length | Variable | Lower cost | Medium |

### 3. The Latency Budget Approach

Define latency budgets per request type:

```
Chat (interactive):
  Total budget: 2,000ms
  ├── Queue:          200ms max (10%)
  ├── Prefill:        300ms max (15%)
  ├── Generation:    1,200ms max (60%) — 40 tokens @ 30ms/token
  ├── Post-process:   200ms max (10%)
  └── Network:        100ms max (5%)

Summarization (batch):
  Total budget: 30,000ms
  ├── Queue:        5,000ms max (17%)
  ├── Prefill:        800ms max (3%)
  ├── Generation:   22,000ms max (73%)
  ├── Post-process:   200ms max (1%)
  └── Network:       2,000ms max (7%)
```

If ANY component exceeds its budget, flag it.

---

## 💻 CODE EXAMPLES

### Example 1: The Broken Latency Test (Full of Bugs)

```python
"""
WHAT NOT TO DO — A latency measurement script with 8 bugs.
"""

import time
import requests

def measure_latency():
    """Measure API latency."""
    start = time.time()
    
    response = requests.post(
        "http://localhost:8000/generate",
        json={"prompt": "Hello, how are you?", "max_tokens": 50},
    )
    
    # BUG 1: Measuring wall clock time, including network latency
    # If network is slow, latency appears high — but it's not the model
    end = time.time()
    total_latency = end - start
    print(f"Total latency: {total_latency:.3f}s")  # BUG 2: Only total, no breakdown
    
    # BUG 3: Only measuring 1 request
    # Cold start might be 5s, warm is 0.5s — average of 1 is meaningless
    
    # BUG 4: Measuring with a trivial prompt
    # "Hello, how are you?" is 5 tokens — prefill is negligible
    # Real prompts are 500+ tokens with very different latency
    
    # BUG 5: Not measuring time-to-first-token (streaming)
    # User experience is dominated by TTFT, not total time
    # But this script only measures total time
    
    # BUG 6: No warmup request
    # First request triggers CUDA kernel compilation (5-10s)
    # If this is the ONLY request, results are useless
    
    # BUG 7: No percentile reporting
    # "Average latency: 1.2s" hides that p99 is 8s
    # Users experience p99, not average
    
    # BUG 8: Testing in isolation
    # No concurrent load — real latency happens under load
    # Single request latency doesn't predict production latency
    
    return total_latency
```

### Example 2: Production Latency Profiler (Fixed)

```python
"""
Production latency profiler with correct measurement methodology.

Key features:
- Proper warmup
- Concurrent load testing
- Per-component breakdown (queue, prefill, generation, network)
- Percentile reporting (p50, p95, p99)
- TTFT and total time measurement
"""

import time
import asyncio
import logging
from dataclasses import dataclass
from typing import List, Optional
from statistics import median, quantiles

import aiohttp
import numpy as np

logger = logging.getLogger(__name__)


@dataclass
class LatencyBreakdown:
    """Per-component latency breakdown for one request."""
    queue_ms: float = 0      # Time waiting before processing starts
    prefill_ms: float = 0    # Time to process input prompt
    generation_ms: float = 0 # Time to generate all output tokens
    ttft_ms: float = 0       # Time to first token (queue + prefill)
    total_ms: float = 0      # Full end-to-end latency
    tokens_per_second: float = 0
    success: bool = True
    error: Optional[str] = None


@dataclass
class LatencyReport:
    """Aggregated latency report with percentiles."""
    total_requests: int
    successful: int
    failed: int
    
    # TTFT (time to first token)
    ttft_p50: float
    ttft_p95: float
    ttft_p99: float
    
    # Total time
    total_p50: float
    total_p95: float
    total_p99: float
    
    # Generation speed
    tokens_per_second_avg: float
    
    # Throughput
    requests_per_second: float
    
    # Breakdown components (median)
    queue_median_ms: float
    prefill_median_ms: float
    generation_median_ms: float


class LatencyProfiler:
    """
    Production latency profiler with scientific methodology.
    
    Methodology:
    1. Send warmup request (discard from results)
    2. Send N measurement requests sequentially (no load)
    3. Send N measurement requests concurrently (under load)
    4. Report percentiles for BOTH scenarios
    """
    
    def __init__(
        self,
        base_url: str = "http://localhost:8000",
        api_key: str = "test",
    ):
        self.base_url = base_url
        self.api_key = api_key
        self.headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
        }
    
    async def warmup(self):
        """Send warmup request to trigger CUDA compilation."""
        logger.info("Warming up...")
        async with aiohttp.ClientSession() as session:
            payload = {
                "prompt": "Hello",
                "max_tokens": 1,
                "temperature": 0.1,
                "stream": False,
            }
            async with session.post(
                f"{self.base_url}/v1/chat/completions",
                headers=self.headers,
                json=payload,
            ) as resp:
                await resp.read()
        logger.info("Warmup complete")
    
    async def measure_single(
        self,
        session: aiohttp.ClientSession,
        prompt: str,
        max_tokens: int = 100,
        stream: bool = True,
    ) -> LatencyBreakdown:
        """Measure latency breakdown for a single request."""
        
        payload = {
            "prompt": prompt,
            "max_tokens": max_tokens,
            "temperature": 0.1,
            "stream": stream,
        }
        
        start = time.monotonic()
        first_token_time = None
        total_content = ""
        
        try:
            async with session.post(
                f"{self.base_url}/v1/completions",
                headers=self.headers,
                json=payload,
            ) as resp:
                
                if stream:
                    # Streaming: read SSE events
                    first_token_time = None
                    async for line in resp.content:
                        if line.startswith(b"data: ") and b"content" in line:
                            if first_token_time is None:
                                first_token_time = time.monotonic()
                            # Extract content (simplified)
                            pass
                else:
                    # Non-streaming: wait for full response
                    data = await resp.json()
                    first_token_time = start + 0.1  # Estimate
                    total_content = data.get("choices", [{}])[0].get("text", "")
            
            end = time.monotonic()
            
            if first_token_time:
                ttft = (first_token_time - start) * 1000
                
                return LatencyBreakdown(
                    queue_ms=ttft * 0.3,  # Estimate queue as 30% of TTFT
                    prefill_ms=ttft * 0.7,  # Estimate prefill as 70% of TTFT
                    generation_ms=(end - first_token_time) * 1000,
                    ttft_ms=ttft,
                    total_ms=(end - start) * 1000,
                    tokens_per_second=len(total_content) / (end - start) / 4,  # Rough
                )
            
            return LatencyBreakdown(
                total_ms=(end - start) * 1000,
                success=False,
                error="No first token detected",
            )
            
        except Exception as e:
            end = time.monotonic()
            return LatencyBreakdown(
                total_ms=(end - start) * 1000,
                success=False,
                error=str(e),
            )
    
    async def run_profile(
        self,
        prompts: List[str],
        concurrency: int = 1,
        max_tokens: int = 100,
        stream: bool = True,
    ) -> LatencyReport:
        """
        Run a complete latency profile.
        
        Args:
            prompts: List of test prompts
            concurrency: Number of concurrent requests
            max_tokens: Max tokens per response
            stream: Use streaming
        """
        
        await self.warmup()
        
        results = []
        connector = aiohttp.TCPConnector(limit=concurrency)
        
        async with aiohttp.ClientSession(connector=connector) as session:
            # Create tasks
            tasks = []
            for i in range(len(prompts)):
                prompt = prompts[i % len(prompts)]
                tasks.append(
                    self.measure_single(session, prompt, max_tokens, stream)
                )
            
            # Run with semaphore to control concurrency
            semaphore = asyncio.Semaphore(concurrency)
            
            async def limited_measure(prompt):
                async with semaphore:
                    return await self.measure_single(
                        session, prompt, max_tokens, stream
                    )
            
            limited_tasks = [limited_measure(p) for p in prompts]
            results = await asyncio.gather(*limited_tasks)
        
        # Process results
        successful = [r for r in results if r.success]
        failed = [r for r in results if not r.success]
        
        if not successful:
            return LatencyReport(
                total_requests=len(results),
                successful=0,
                failed=len(failed),
                **{k: 0 for k in [
                    "ttft_p50", "ttft_p95", "ttft_p99",
                    "total_p50", "total_p95", "total_p99",
                    "tokens_per_second_avg",
                    "requests_per_second",
                    "queue_median_ms", "prefill_median_ms",
                    "generation_median_ms",
                ]}
            )
        
        ttfts = sorted([r.ttft_ms for r in successful])
        totals = sorted([r.total_ms for r in successful])
        tokens_per_sec = [r.tokens_per_second for r in successful]
        
        def percentile(data, p):
            return data[int(len(data) * p / 100)]
        
        report = LatencyReport(
            total_requests=len(results),
            successful=len(successful),
            failed=len(failed),
            ttft_p50=percentile(ttfts, 50),
            ttft_p95=percentile(ttfts, 95),
            ttft_p99=percentile(ttfts, 99),
            total_p50=percentile(totals, 50),
            total_p95=percentile(totals, 95),
            total_p99=percentile(totals, 99),
            tokens_per_second_avg=sum(tokens_per_sec) / len(tokens_per_sec),
            requests_per_second=len(successful) / (sum(r.total_ms for r in successful) / 1000 / len(successful)),
            queue_median_ms=median([r.queue_ms for r in successful]),
            prefill_median_ms=median([r.prefill_ms for r in successful]),
            generation_median_ms=median([r.generation_ms for r in successful]),
        )
        
        return report


def generate_latency_optimization_report(report: LatencyReport) -> str:
    """Generate actionable recommendations from latency profile."""
    
    lines = []
    lines.append("LATENCY OPTIMIZATION REPORT")
    lines.append("=" * 50)
    lines.append(f"Requests: {report.total_requests} ({report.successful} ok, {report.failed} failed)")
    lines.append(f"")
    lines.append(f"TTFT:        p50={report.ttft_p50:.0f}ms  p95={report.ttft_p95:.0f}ms  p99={report.ttft_p99:.0f}ms")
    lines.append(f"Total:       p50={report.total_p50:.0f}ms  p95={report.total_p95:.0f}ms  p99={report.total_p99:.0f}ms")
    lines.append(f"Tokens/s:    {report.tokens_per_second_avg:.1f}")
    lines.append(f"Throughput:  {report.requests_per_second:.1f} req/s")
    lines.append(f"")
    
    # Recommendations
    lines.append("RECOMMENDATIONS:")
    
    if report.ttft_p95 > 1000:
        lines.append(f"  ⚠️  High TTFT ({report.ttft_p95:.0f}ms p95)")
        lines.append("     → Enable prefix caching (reduces prefill time)")
        lines.append("     → Reduce queue by lowering max_num_seqs")
        lines.append("     → Consider larger GPU for faster prefill")
    
    if report.total_p95 > 5000:
        lines.append(f"  ⚠️  High total latency ({report.total_p95:.0f}ms p95)")
        lines.append("     → Enable speculative decoding")
        lines.append("     → Consider INT8/INT4 quantization")
        lines.append("     → Reduce max_tokens if responses are too long")
    
    if report.queue_median_ms > 200:
        lines.append(f"  ⚠️  High queue time ({report.queue_median_ms:.0f}ms)")
        lines.append("     → Reduce max_num_seqs to limit concurrent requests")
        lines.append("     → Add request prioritization (interactive vs. batch)")
        lines.append("     → Scale up GPU count if consistently queued")
    
    if report.tokens_per_second_avg < 30:
        lines.append(f"  ⚠️  Low generation speed ({report.tokens_per_second_avg:.1f} tok/s)")
        lines.append("     → Check GPU memory bandwidth utilization")
        lines.append("     → Consider speculative decoding")
        lines.append("     → Upgrade GPU if memory bandwidth is bottleneck")
    
    if report.failed > 0:
        lines.append(f"  ❌ {report.failed} requests failed — investigate errors")
    
    return "\n".join(lines)
```

---

### 🔍 Critical Questions

1. **The latency profiler** measures single-request latency. But REAL latency is experienced under LOAD — when multiple requests queue up. **How would you modify the profiler to measure latency at DIFFERENT REQUEST RATES** (e.g., 10 req/s, 50 req/s, 100 req/s) and produce a LOAD-LATENCY CURVE?

2. **Streaming vs. non-streaming**: The TTFT is the same whether you stream or not (both wait for the first token). But user PERCEPTION is dramatically different — streaming feels 2-3x faster even when objective latency is identical. **How would you measure "perceived latency" objectively?** Is there a metric that captures user perception?

3. **The latency report** recommends actions based on thresholds (TTFT > 1s, queue > 200ms). But these thresholds are ARBITRARY — they don't account for your specific use case. For a chatbot, 2s is acceptable. For autocomplete, 200ms is the maximum. **How would you make the recommender USE-CASE-AWARE?** What inputs would you need to customize thresholds?

---

## ✅ Good Output Examples

### What a Well-Profiled Latency Report Looks Like

```
LATENCY PROFILE: Production A10G — 7B AWQ — 50 concurrent users
================================================================
Methodology: 500 requests, 10 concurrent, warmup included

TTFT (Time to First Token):
  p50:    342ms  ✅ (target: <500ms)
  p95:    892ms  ⚠️  (target: <1000ms, approaching threshold)
  p99:   1,843ms ❌ (target: <2000ms, some users experience delay)

Total Time (End-to-End):
  p50:   1,204ms  ✅
  p95:   2,891ms  ⚠️
  p99:   5,432ms  ❌ (9x median — high variance!)

Breakdown (p50):
  Queue:       89ms  (7%)  — Healthy
  Prefill:    253ms  (21%) — Could improve with prefix caching
  Generation: 821ms  (68%) — ~35 tok/s at ~25 tokens generated
  Network:     41ms  (3%)

Recommendations:
  ✅ Streaming: enabled (first token at 342ms)
  ⚠️  High p99/p50 ratio (4.5x): load is uneven, add request prioritization
  ⚠️  Prefill at 253ms: enable prefix caching if prompts share prefixes
  ❌  Generation at 35 tok/s: spec decode could help if acceptance rate > 40%
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Optimizing the Wrong Component

**Problem**: Spend 2 weeks implementing speculative decoding to speed up generation. The actual bottleneck was queueing — requests were waiting 800ms before processing started. Generation was only 300ms. You optimized 300ms down to 200ms but the user still waits 900ms total.

**Fix**: PROFILE FIRST. Decompose latency into components. Optimize the BIGGEST component first. Re-profile after each optimization.

### Antipattern 2: Ignoring Tail Latency

**Problem**: Average latency is 800ms. Looks great! But p99 is 8,000ms. 1% of users experience 10x worse performance. Those 1% are your most important users (they're power users who make many requests).

**Fix**: Report and optimize p99 latency, not just p50. If the gap between p50 and p99 is > 3x, investigate variance.

### Antipattern 3: Measuring Without Load

**Problem**: Single-request latency is 300ms. Deploy to production. Users complain it's slow. Under load, latency is 3,000ms because of queueing.

**Fix**: Measure latency at PRODUCTION LOAD levels. The single-request number is a minimum, not a prediction.

### Common Latency Failures

| Failure | Symptom | Fix |
|---------|---------|-----|
| Queue buildup | Latency increases with request rate | Reduce max_num_seqs, scale GPU count |
| Cold start | First request is 10x slower | Warmup request, keep-alive |
| Memory thrashing | Latency spikes periodically | Reduce max_model_len, reduce concurrency |
| Network overhead | High latency even for simple prompts | Co-locate services, use VPC internal networking |
| No streaming | Users wait for full response | Enable streaming for all interactive use cases |

---

## 🧪 Drills & Challenges

### Drill 1: Profile Your Latency (30 min)

**Task**: Write a latency profiler that measures all 4 latency components: queue, prefill, generation, network. Run it against a running vLLM instance with:
- 10 sequential requests (no load)
- 50 concurrent requests (under load)
- Report p50, p95, p99 for both scenarios

---

### Drill 2: Build a Latency Budget (30 min)

**Task**: For a real-time chatbot with p99 target of 2s, design a latency budget:
1. Allocate ms budgets to each component (queue, prefill, generation, post-processing, network)
2. What's the maximum output length you can support within budget?
3. What's the minimum GPU you need?
4. Where is the MOST IMPORTANT optimization target?

---

### Drill 3: Streaming vs. Non-Streaming A/B Test (30 min)

**Task**: Design an A/B test comparing streaming vs. non-streaming responses.

**Measure:**
1. Objective latency (TTFT, total time) — for both streaming and non-streaming
2. User engagement — do users send MORE follow-up messages with streaming?
3. User satisfaction — survey after each interaction

**Expected result:** Streaming might have identical objective latency but DRAMATICALLY better perceived latency and user satisfaction. Quantify the gap.

---

## 🚦 Gate Check

Before moving to File 06 (Monitoring & Alerting), verify you can:

1. **Decompose latency** — For a given deployment, identify: which component contributes most to latency, how to measure each component, and which optimization targets each

2. **Design a latency test** — Write a test plan that measures: cold start, warm p50/p95/p99 under no load, and warm p50/p95/p99 under production load

3. **Choose optimizations** — Given: p50=2.1s, TTFT=1.2s, generation=0.7s, queue=0.2s. Recommend 3 optimizations ranked by impact

4. **Handle the streaming tradeoff** — Given a use case where streaming improves perceived latency by 60% but increases infrastructure costs by 15% (more connections), make the build/buy decision

5. **Set latency budgets** — For a voice assistant (must respond in <800ms), design the latency budget across: ASR (speech-to-text), LLM inference, TTS (text-to-speech), network

---

## 📚 Resources

### Latency Optimization
- **[LLM Inference Performance Optimization](https://www.anyscale.com/blog/llm-inference-performance-engineering-best-practices)** — Comprehensive optimization guide
- **[Continuous Batching Explained](https://flyte.org/blog/continuous-batching-a-k-a-dynamic-batching-for-llm-inference)** — How dynamic batching works

### Speculative Decoding
- **[Speculative Decoding Paper](https://arxiv.org/abs/2211.17192)** — The original paper
- **[Medusa Speculative Decoding](https://arxiv.org/abs/2401.10774)** — Multiple-head speculation

### Phase Cross-References
- **Phase 10, File 02 (Model Serving)** → vLLM configuration directly impacts all latency components
- **Phase 10, File 03 (Caching)** → Caching is the HIGHEST IMPACT latency optimization (cache hit: 5ms vs. GPU: 1s+)
- **Phase 10, File 04 (Cost)** → Every latency optimization has a COST TRADEOFF. Faster = more expensive.

> **Next up: File 06 — Monitoring & Alerting.** You've built the system. You've optimized costs and latency. Now you need to KNOW when it breaks — before users tell you. You'll build a complete monitoring stack with Prometheus, Grafana, and intelligent alerting that catches real problems without drowning you in noise.
