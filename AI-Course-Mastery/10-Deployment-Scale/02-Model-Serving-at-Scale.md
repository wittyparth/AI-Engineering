# 02 — Model Serving at Scale: vLLM Deep Dive

## 🎯 Purpose & Goals

> 🛑 STOP. Before you scale your model serving from 1 GPU to N GPUs, you need to understand the single biggest bottleneck:

**GPU utilization is NEVER 100%.**

Most engineers assume: "My model uses the GPU → GPU is busy → GPU utilization is 100%." This is WRONG. In reality, most of the time is spent:
- Loading input data from CPU to GPU memory (transfer bottleneck)
- Waiting for the next request to arrive (batching inefficiency)
- Computing attention for PADDING tokens (wasted computation)
- Re-computing KV cache for shared prefixes (redundant work)
- Synchronizing across GPUs (communication overhead)

**A single-GPU deployment at 20% utilization is normal. Getting to 50% is an achievement. Getting to 80% means you've mastered serving.**

**The goal of this module:** Understand what vLLM does under the hood and how to configure it for MAXIMUM throughput.

---

### 🤔 Discovery Question 1: The Batch Size Illusion

You're serving a 7B model on a single A10G (24GB). You configure vLLM:

```
max_num_seqs = 256     # Max 256 concurrent sequences
max_model_len = 4096   # Max 4096 tokens per sequence
```

You expect: "256 sequences can be processed in parallel → high throughput."

**Reality:**
- GPU memory: 24GB
- Model weights (AWQ 4-bit): ~4GB
- KV cache per token: ~2MB (FP16, 32 layers, 32 heads, 128 dim per head)
- KV cache for 256 sequences × 4096 tokens: 256 × 4096 × 2MB = ~2.1GB
- Remaining memory for activations, overhead: enough... barely

But here's the problem: with 256 sequences at 4096 tokens each, the TOTAL sequence length is 256 × 4096 = **1,048,576 tokens**. The attention mechanism computes ALL pairwise interactions within each sequence — that's O(n²) per sequence.

**256 sequences at 4096 tokens each is 256 × 4096² = 4.3 BILLION attention computations per layer.**

**🤔 Before reading on, answer these:**

1. You have 24GB GPU memory. Model weights take 4GB. KV cache takes ~2.1GB at max capacity. That leaves ~18GB for activations and intermediate computations. But the attention computation for 256 sequences generates HUGE intermediate tensors. **What actually runs out first — memory or compute?** If you profile and find GPU compute utilization is only 35%, what's the bottleneck? (Hint: Memory bandwidth. The GPU spends most of its time loading weights from HBM to SRAM, not computing.)

2. You reduce max_num_seqs from 256 to 64. Throughput DROPS by 30% (fewer sequences in each batch). You set max_num_seqs to 512. The server OOMs at peak traffic. **What's the OPTIMAL max_num_seqs for your model, GPU, and workload?** How would you find it without trial-and-error? (Hint: Monitor KV cache utilization. If it's below 80%, you can increase. If it's at 95%, you're one spike away from OOM.)

3. **The hardest question:** Your colleague says: "Batch size is a LIE. vLLM doesn't use fixed batch sizes — it uses CONTINUOUS BATCHING. Every step, the scheduler decides which sequences to include in the next batch based on which have pending tokens. The 'max_num_seqs' is just an UPPER BOUND, not a target." **If continuous batching means the effective batch size varies every step, how do you predict throughput for capacity planning?** What's the formula that relates max_num_seqs, average request rate, and average response length to expected throughput?

---

### 🤔 Discovery Question 2: The Prefix Caching Paradox

You're serving a chatbot that always starts with a 1,500-token system prompt. Every request includes the same system prompt + a short user query.

- System prompt: 1,500 tokens (same for every request)
- User query: ~50 tokens (varies)
- Total: 1,550 tokens per request

Without prefix caching:
- Each request computes KV cache for 1,550 tokens
- 1,500 tokens are REDUNDANT (same system prompt)
- Wasted computation: 1,500/1,550 = 97% of all KV cache computation

vLLM has `enable_prefix_caching=true`. It caches the KV cache for common prefixes. When a new request arrives with the same prefix, it REUSES the cached KV cache instead of recomputing.

**But you enable it and see only 20% improvement, not the 97% you expected.**

**🤔 Before reading on, answer these:**

1. Why would prefix caching only give 20% improvement when 97% of tokens are redundant? (Hint: KV cache caching operates at the BLOCK level — 16 tokens per block. What if the system prompt crosses a block boundary? What if the system prompt isn't perfectly aligned with the block size?)

2. You check the vLLM logs and see: "Prefix cache hit rate: 12%." But 97% of tokens SHOULD be cached. What could cause such a low hit rate? (Hint: What if user queries AREN'T all after the exact same prefix? What if some requests include session-specific variables IN the system prompt? Even a 1-token difference in the prefix invalidates the entire cache for that position.)

3. **The hardest question:** Your system prompt includes the USER'S NAME: "You are helping {user_name} with their legal documents." Since each user has a different name, the system prompt is DIFFERENT for each user — prefix caching helps NOTHING. **How would you restructure your prompt template to MAXIMIZE prefix cache utilization while still personalizing responses?** (Hint: Move the user name to AFTER the system prompt, in the user message. Or use a placeholder that gets substituted AFTER KV cache computation. Or split the prompt into "shared system" + "user-specific" segments.)

---

### 🤔 Discovery Question 3: The Multi-GPU Scaling Wall

You add a second GPU. You expect 2x throughput.

You configure:
```
tensor_parallel_size = 2
```

**Results:**
- 1 GPU: 50 requests/second
- 2 GPUs: 72 requests/second (only 1.44x improvement, not 2x)

**🤔 Before reading on, answer these:**

1. Where did the 0.56x throughput go? (28% loss from ideal scaling). The model is split across 2 GPUs — each GPU holds half the layers. Every forward pass requires ALL-TO-ALL communication between GPUs (each GPU sends its activations to the other GPU). **What's the bottleneck — compute or communication?** How much throughput would you lose to PCIe bandwidth vs. NVLink bandwidth? (NVLink is 10x faster than PCIe.)

2. You try tensor_parallel_size=4 on 4 GPUs. Throughput: 95 req/s — only 1.9x of single GPU, and LESS than 2-GPU throughput per GPU (18.75 req/s/GPU vs 36 req/s/GPU). The communication overhead grows with the SQUARE of the number of GPUs (each GPU communicates with ALL others). **At what point does adding more GPUs DECREASE throughput per GPU?** Is there an optimal number of GPUs for a given model size?

3. **The hardest question:** You have 4 GPUs. You can either:
   - **Option A**: Tensor parallelism across all 4 (1 model, 4 GPUs) — low latency per request, lower throughput per GPU
   - **Option B**: Two independent model instances on 2 pairs (2 models, 2 GPUs each) — higher throughput per model instance, but each instance handles half the traffic
   
   **Which gives higher TOTAL throughput?** What if latency is the priority vs. throughput? What if your workload is batch-heavy (offline) vs. latency-sensitive (real-time)?

---

## ⏱ Time Budget

| Section | Time |
|---------|------|
| Discovery Questions | 30 min |
| Concept Deep-Dive | 1.5 hr |
| Code Examples | 1.5 hr |
| Drills | 45 min |
| Gate Check | 15 min |
| **Total** | **~4.5 hr** |

---

## 📖 Concept Deep-Dive

### 1. How vLLM Actually Works

```
Request arrives → Tokenizer → Scheduler → Block Manager → Model Execution → Output
                        │              │            │
                        │         ┌────┘      ┌────┘
                        │         │           │
                   ┌────▼─────────▼────────────▼──────┐
                   │         Core Concepts             │
                   │                                   │
                   │  Continuous Batching:              │
                   │  Every step, scheduler picks which │
                   │  sequences to process next.        │
                   │  No fixed batch size.              │
                   │                                   │
                   │  PagedAttention:                   │
                   │  KV cache stored in fixed-size     │
                   │  blocks (16 tokens). Allocated     │
                   │  on-demand, not pre-allocated.     │
                   │                                   │
                   │  Prefix Caching:                   │
                   │  KV cache blocks are hash-addressed.│
                   │  Shared prefixes reuse blocks.     │
                   └───────────────────────────────────┘
```

### 2. PagedAttention — The Key Innovation

Traditional KV cache: Pre-allocate max_seq_len × num_layers × num_heads × head_dim for each sequence. Wastes memory because most sequences are shorter than max_seq_len.

PagedAttention: Allocate KV cache in BLOCKS (usually 16 tokens each). A sequence with 100 tokens uses 7 blocks (96 tokens + 4 in the 7th block). A sequence with 4,000 tokens uses 250 blocks.

**This matters because:**
- Short sequences don't waste memory on unused slots
- Blocks can be shared between sequences (prefix caching)
- No fragmentation — blocks are the same size and can be freely allocated/freed

**Memory formula:**
```
KV cache size = num_blocks_in_use × block_size × num_layers × 2 × head_dim × num_kv_heads

For Llama-3-8B with block_size=16 and 8192 context:
  KV heads = 8, layers = 32, head_dim = 128
  Bytes per token = 2 × 32 × 128 × 8 × 2 bytes = 131,072 bytes = 128KB
  Per sequence (8192 tokens) = 8192 × 128KB = 1GB
  With gpu_memory_utilization=0.9 → available KV cache memory = 24GB × 0.9 - 4GB(weights) = 17.6GB
  Max concurrent sequences = 17.6GB / 1GB ≈ 17 (but with variable length, actual is higher)
```

### 3. Tensor Parallelism vs. Pipeline Parallelism

| | Tensor Parallelism | Pipeline Parallelism |
|---|---|---|
| **How it works** | Split EACH layer across GPUs | Split LAYERS across GPUs |
| **Communication** | ALL-TO-ALL every layer (heavy) | Only at layer boundaries (light) |
| **Latency** | Lower (no waiting for previous layers) | Higher (sequential pipeline) |
| **Best for** | Latency-sensitive serving | Throughput-maximizing batch |
| **Scaling efficiency** | ~80% at 2 GPUs, ~50% at 8 GPUs | ~90%+ regardless of GPU count |
| **When to use** | Need low latency per request | Processing large batches offline |

### 4. vLLM Configuration Guide

| Parameter | What It Does | Tuning Guidance |
|-----------|-------------|-----------------|
| `max_num_seqs` | Max concurrent sequences | Start: 256. Monitor KV cache usage. Increase if utilization < 80%. |
| `max_model_len` | Max tokens per sequence | Set to your ACTUAL max (check your data). Higher = more KV cache per sequence. |
| `gpu_memory_utilization` | % of GPU memory for KV cache | Start: 0.90. Increase to 0.95 if stable. Decrease if OOM. |
| `block_size` | KV cache block size (tokens) | Default 16. Smaller = more flexible but more overhead. |
| `enable_prefix_caching` | Cache KV cache for repeated prefixes | Enable if requests share prefixes (system prompts). |
| `tensor_parallel_size` | GPUs per model instance | Start: 1. Only increase if single GPU is latency-bound. |
| `max_num_batched_tokens` | Max tokens in one batch | Default matches max_model_len. Lower = more scheduling flexibility. |

---

## 💻 CODE EXAMPLES

### Example 1: The Broken vLLM Deployment (Full of Bugs)

```python
"""
WHAT NOT TO DO — A vLLM server configuration with 10+ hidden bugs.
"""

from vllm import LLM, SamplingParams

# BUG 1: Loading with default parameters
# Default max_model_len is the MODEL's max (often 4096 or 8192)
# But your USE CASE only needs 2048 — you're wasting KV cache on unused capacity
llm = LLM(model="mistralai/Mistral-7B-Instruct-v0.2")

# BUG 2: No quantization specified
# Defaults to FP16 — 7B model uses 14GB just for weights
# Should use AWQ or GPTQ 4-bit

# BUG 3: No tensor_parallel_size
# Single GPU only — even if 2 GPUs are available
# Leaves half the hardware unused

# BUG 4: Using default gpu_memory_utilization (0.9)
# This is a GOOD default, but should be adjusted based on workload
# If you KNOW max_model_len=2048 and requests are short,
# you can increase to 0.95 for more concurrent sequences

# BUG 5: No enable_prefix_caching
# If requests share system prompts, this wastes 50%+ of compute
# Should be True for most chatbots

sampling_params = SamplingParams(
    temperature=0.7,
    top_p=0.9,
    max_tokens=512,
)

# BUG 6: Generating one at a time
# This defeats vLLM's continuous batching
# vLLM is designed to BATCH multiple requests
for prompt in prompts:
    output = llm.generate(prompt, sampling_params)
    process_output(output)

# BUG 7: No error handling
# If the GPU runs out of memory, the entire process crashes
# Should catch exceptions and retry with reduced batch size

# BUG 8: No warmup
# First request is SLOW because CUDA kernels need to compile
# Should send a dummy request on startup to trigger compilation

# BUG 9: No monitoring
# No way to know: KV cache utilization, request queue depth, throughput
# Should expose metrics endpoint

# BUG 10: Using vLLM directly, not the OpenAI-compatible server
# The vLLM library API is lower-level and harder to use in production
# The OpenAI-compatible API server handles: HTTP routing, auth, metrics
```

#### 🔍 Find The Bugs

| # | Bug | Impact | Fix |
|---|-----|--------|-----|
| 1 | Default max_model_len | Wastes KV cache on unused capacity | Set max_model_len to actual max needed |
| 2 | No quantization | 2-4x memory usage | Add quantization=awq to LLM constructor |
| 3 | No tensor_parallel_size | Only 1 GPU used | Set tensor_parallel_size=N if N GPUs available |
| 4 | Default gpu_memory_utilization | May be suboptimal | Tune based on workload (0.85-0.95) |
| 5 | No prefix caching | 50%+ compute wasted | enable_prefix_caching=True |
| 6 | Sequential generation | No batching | Collect prompts, generate in batch |
| 7 | No error handling | Process crashes on OOM | Wrap with try/except, reduce batch on OOM |
| 8 | No warmup | Slow first request | Send warmup request on startup |
| 9 | No monitoring | Blind serving | Add Prometheus metrics |
| 10 | Using library API directly | Missing production features | Use OpenAI-compatible API server |

---

### Example 2: Production vLLM Configuration (Fixed)

```python
"""
Production vLLM deployment with optimal configuration.

This covers:
1. Loading the model with optimal settings
2. Batching for maximum throughput
3. Monitoring and metrics
4. Graceful error handling
5. Warm-up and health checks
"""

import time
import logging
from typing import List, Optional, AsyncGenerator
from contextlib import asynccontextmanager

from vllm import LLM, SamplingParams
from vllm.engine.arg_utils import AsyncEngineArgs
from vllm.engine.async_llm_engine import AsyncLLMEngine
from vllm.outputs import RequestOutput
from vllm.utils import random_uuid

logger = logging.getLogger(__name__)


# ──────────────────────────────────────────────
# 1. OPTIMAL MODEL LOADING
# ──────────────────────────────────────────────

def create_llm_engine(
    model_name: str = "mistralai/Mistral-7B-Instruct-v0.2",
    quantization: str = "awq",
    max_model_len: int = 4096,
    gpu_memory_utilization: float = 0.90,
    tensor_parallel_size: int = 1,
    max_num_seqs: int = 256,
    enable_prefix_caching: bool = True,
    block_size: int = 16,
) -> AsyncLLMEngine:
    """
    Create vLLM engine with production-optimized settings.
    
    Key decisions explained:
    - quantization='awq': 4-bit AWQ has better quality than GPTQ at same bit-width
    - max_model_len=4096: Set based on actual data (check 95th percentile)
    - gpu_memory_utilization=0.90: Reserve 10% for KV cache fluctuations
    - enable_prefix_caching=True: Essential for chatbot system prompts
    - block_size=16: Good balance of flexibility vs. overhead
    """
    
    # Validate configuration
    if tensor_parallel_size > 1:
        import torch
        visible_devices = torch.cuda.device_count()
        if visible_devices < tensor_parallel_size:
            raise ValueError(
                f"Requested {tensor_parallel_size} GPUs but only "
                f"{visible_devices} available"
            )
    
    engine_args = AsyncEngineArgs(
        model=model_name,
        quantization=quantization,
        max_model_len=max_model_len,
        gpu_memory_utilization=gpu_memory_utilization,
        tensor_parallel_size=tensor_parallel_size,
        max_num_seqs=max_num_seqs,
        enable_prefix_caching=enable_prefix_caching,
        block_size=block_size,
        # Disable unused features for memory savings
        disable_log_requests=True,
        # Enable metrics
        disable_log_stats=False,
    )
    
    engine = AsyncLLMEngine.from_engine_args(engine_args)
    logger.info(
        f"vLLM engine created: {model_name}, "
        f"quant={quantization}, "
        f"max_len={max_model_len}, "
        f"tensor_parallel={tensor_parallel_size}"
    )
    
    return engine


# ──────────────────────────────────────────────
# 2. BATCHED INFERENCE
# ──────────────────────────────────────────────

class BatchedInferenceEngine:
    """
    Handles batched inference with vLLM.
    
    Instead of generating one-by-one (Bug 6 from the broken example),
    this collects requests and processes them in batches.
    """
    
    def __init__(self, engine: AsyncLLMEngine):
        self.engine = engine
        self._warmup_done = False
    
    async def warmup(self):
        """Send a warmup request to trigger CUDA kernel compilation."""
        if self._warmup_done:
            return
        
        logger.info("Warming up vLLM engine...")
        start = time.time()
        
        result = await self.generate(
            prompt="Hello",  # Minimal prompt for fast warmup
            max_tokens=1,    # Single token
            temperature=0.1,
        )
        
        elapsed = time.time() - start
        logger.info(f"Warmup complete in {elapsed:.2f}s")
        self._warmup_done = True
    
    async def generate(
        self,
        prompt: str,
        max_tokens: int = 512,
        temperature: float = 0.7,
        top_p: float = 0.9,
        **kwargs,
    ) -> str:
        """Generate a single response."""
        
        sampling_params = SamplingParams(
            temperature=temperature,
            top_p=top_p,
            max_tokens=max_tokens,
            **kwargs,
        )
        
        request_id = random_uuid()
        
        try:
            results = []
            async for output in self.engine.generate(
                prompt, sampling_params, request_id
            ):
                results.append(output)
            
            if results:
                return results[-1].outputs[0].text
            return ""
            
        except Exception as e:
            logger.error(f"Generation failed: {e}")
            raise
    
    async def batch_generate(
        self,
        prompts: List[str],
        max_tokens: int = 512,
        temperature: float = 0.7,
    ) -> List[str]:
        """
        Generate responses for multiple prompts in parallel.
        
        vLLM's engine handles batching internally — we just need to
        submit all requests and collect results.
        """
        
        sampling_params = SamplingParams(
            temperature=temperature,
            top_p=0.9,
            max_tokens=max_tokens,
        )
        
        # Submit all requests
        request_ids = []
        tasks = {}
        
        for prompt in prompts:
            request_id = random_uuid()
            request_ids.append(request_id)
            
            # Collect results for this request
            results = []
            async for output in self.engine.generate(
                prompt, sampling_params, request_id
            ):
                results.append(output)
            tasks[request_id] = results
        
        # Collect results in order
        outputs = []
        for rid in request_ids:
            results = tasks[rid]
            if results:
                outputs.append(results[-1].outputs[0].text)
            else:
                outputs.append("")
        
        return outputs


# ──────────────────────────────────────────────
# 3. PRODUCTION API SERVER
# ──────────────────────────────────────────────

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import prometheus_client

# Prometheus metrics
REQUESTS_TOTAL = prometheus_client.Counter(
    'model_requests_total', 'Total inference requests'
)
REQUESTS_FAILED = prometheus_client.Counter(
    'model_requests_failed', 'Failed inference requests'
)
LATENCY_HISTOGRAM = prometheus_client.Histogram(
    'model_request_duration_seconds', 'Request latency in seconds',
    buckets=[0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)
KV_CACHE_USAGE = prometheus_client.Gauge(
    'model_kv_cache_usage_blocks', 'KV cache block usage'
)

class GenerateRequest(BaseModel):
    prompt: str
    max_tokens: int = 512
    temperature: float = 0.7

class GenerateResponse(BaseModel):
    text: str
    tokens_generated: int
    latency_ms: float


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup and shutdown events."""
    # Create engine
    app.state.engine = create_llm_engine()
    app.state.inference = BatchedInferenceEngine(app.state.engine)
    
    # Warm up
    await app.state.inference.warmup()
    
    yield
    
    # Cleanup
    del app.state.engine

app = FastAPI(lifespan=lifespan)


@app.get("/health")
async def health():
    """Health check — used by load balancer."""
    return {"status": "healthy"}


@app.get("/metrics")
async def metrics():
    """Prometheus metrics endpoint."""
    return prometheus_client.generate_latest()


@app.post("/generate", response_model=GenerateResponse)
async def generate(request: GenerateRequest):
    """Generate a response."""
    REQUESTS_TOTAL.inc()
    
    start = time.time()
    try:
        text = await app.state.inference.generate(
            prompt=request.prompt,
            max_tokens=request.max_tokens,
            temperature=request.temperature,
        )
        
        latency = (time.time() - start) * 1000
        LATENCY_HISTOGRAM.observe(latency / 1000)
        
        # Rough token count (4 chars ≈ 1 token for English)
        tokens = len(text) // 4
        
        return GenerateResponse(
            text=text,
            tokens_generated=tokens,
            latency_ms=latency,
        )
        
    except Exception as e:
        REQUESTS_FAILED.inc()
        logger.error(f"Generation failed: {e}")
        raise HTTPException(status_code=500, detail=str(e))


# ──────────────────────────────────────────────
# 4. CAPACITY PLANNING TOOL
# ──────────────────────────────────────────────

def estimate_capacity(
    model_size_b: int = 7,
    quantization: str = "awq",
    gpu_memory_gb: int = 24,
    gpu_memory_utilization: float = 0.90,
    avg_prompt_tokens: int = 500,
    avg_generation_tokens: int = 200,
    avg_concurrent_users: int = 10,
    target_latency_p99_ms: int = 3000,
) -> dict:
    """
    Estimate serving capacity for a given configuration.
    
    This is a BACK-OF-ENVELOPE calculation — actual results vary.
    But it's good enough for capacity planning.
    """
    
    # Model weight size
    weight_bytes_per_param = {
        "fp16": 2,
        "int8": 1,
        "awq": 0.5,   # 4-bit
        "gptq": 0.5,  # 4-bit
        "fp32": 4,
    }
    weight_size_gb = model_size_b * weight_bytes_per_param.get(quantization, 2) / 1024
    
    # Available memory for KV cache
    total_memory = gpu_memory_gb * gpu_memory_utilization
    kv_cache_memory = total_memory - weight_size_gb
    
    # KV cache per token (for Llama architecture)
    # 2 (K and V) × layers × kv_heads × head_dim × 2 bytes (FP16)
    layers = {7: 32, 8: 32, 13: 40, 70: 80}.get(model_size_b, 32)
    kv_heads = {7: 8, 8: 8, 13: 8, 70: 8}.get(model_size_b, 8)
    head_dim = 128
    bytes_per_token = 2 * layers * kv_heads * head_dim * 2
    
    # Total KV cache for one sequence
    total_tokens = avg_prompt_tokens + avg_generation_tokens
    kv_cache_per_seq_gb = total_tokens * bytes_per_token / (1024 ** 3)
    
    # Maximum concurrent sequences
    max_concurrent = int(kv_cache_memory / kv_cache_per_seq_gb)
    
    # Throughput estimation (very rough)
    # Assumes: 1 token ≈ 2x model inference time in ms
    # For 7B on A10G: ~30ms per token
    ms_per_token = {7: 30, 8: 25, 13: 45, 70: 150}.get(model_size_b, 30)
    generation_time_ms = avg_generation_tokens * ms_per_token
    prompt_time_ms = avg_prompt_tokens * ms_per_token * 0.3  # Prefill is faster
    total_time_per_request = prompt_time_ms + generation_time_ms
    
    # Theoretical max throughput (batched)
    throughput_per_gpu = 3600 / (total_time_per_request / 1000) * max_concurrent
    
    # Latency estimation
    # At low concurrency: latency ≈ total_time_per_request
    # At high concurrency: queueing adds latency
    utilization = avg_concurrent_users / max_concurrent
    queueing_factor = 1 / (1 - utilization) if utilization < 1 else float('inf')
    estimated_p50 = total_time_per_request * queueing_factor
    
    return {
        "model_size_b": model_size_b,
        "quantization": quantization,
        "weight_size_gb": round(weight_size_gb, 1),
        "kv_cache_memory_gb": round(kv_cache_memory, 1),
        "max_concurrent_sequences": max_concurrent,
        "estimated_throughput_req_per_hour": int(throughput_per_gpu),
        "estimated_p50_latency_ms": int(estimated_p50),
        "is_feasible": estimated_p50 < target_latency_p99_ms,
        "gpu_recommendation": (
            "Single GPU sufficient"
            if max_concurrent >= avg_concurrent_users * 2
            else "Consider multi-GPU or larger GPU"
        ),
    }


if __name__ == "__main__":
    # Example capacity planning
    result = estimate_capacity(
        model_size_b=7,
        quantization="awq",
        gpu_memory_gb=24,
        avg_prompt_tokens=500,
        avg_generation_tokens=200,
        avg_concurrent_users=10,
    )
    
    print(f"\n{'='*50}")
    print(f"CAPACITY ESTIMATION")
    print(f"{'='*50}")
    for k, v in result.items():
        print(f"{k}: {v}")
```

---

### 🔍 Critical Questions

1. **The capacity estimator** uses a linear model for throughput (doubling sequences doubles throughput). But real vLLM throughput is NOT linear with sequence count — at high concurrency, the GPU memory bandwidth saturates and throughput FLATTENS. **How would you model the REAL throughput curve** (power law? logistic?)? What data would you collect to calibrate the model?

2. **Prefix caching** works by hashing KV cache blocks. But what if two requests have the SAME prefix but DIFFERENT sampling parameters (temperature, top_p)? The prefix KV cache is the same, but the generated tokens will differ after the prefix. **Can prefix caching still help when sampling parameters differ?** What about when the system prompt is identical but the user messages start at different token positions?

3. **The warmup request** sends "Hello" with max_tokens=1. This triggers CUDA kernel compilation for a SHORT sequence. But the FIRST real request might be 4,000 tokens — which triggers DIFFERENT kernel optimizations. **Does a 1-token warmup actually warm up the kernels needed for 4,000-token sequences?** What would be a better warmup strategy?

---

## ✅ Good Output Examples

### What a Healthy vLLM Deployment Looks Like

```
vLLM Status:
├── Model: Mistral-7B-Instruct-v0.2 (AWQ 4-bit)
├── GPU: 1x A10G (24GB)
├── KV Cache: 7,891 blocks used / 10,240 available (77% utilization)
├── Max Sequences: 198 / 256
├── Throughput: 45.2 req/s
│
├── Latency:
│   ├── p50: 0.8s
│   ├── p95: 1.9s
│   └── p99: 2.7s
│
├── GPU Utilization:
│   ├── Compute: 68%
│   ├── Memory: 89%
│   └── NVMem BW: 72%
│
└── Errors (24h): 12 / 3,891,204 requests (0.0003%)
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Overcommitting GPU Memory

**Problem**: Set gpu_memory_utilization=0.99 or skip KV cache limits entirely. The server works for 5 minutes, then OOMs when enough concurrent requests accumulate.

**Fix**: Start at 0.85, monitor KV cache utilization for 24 hours, increase slowly. Never exceed 0.95 for production.

### Antipattern 2: Maximum Context for Everyone

**Problem**: Set max_model_len=32768 because "it's better to have more context." Every sequence allocates KV cache for 32K tokens. Most requests use 1K tokens. 97% of KV cache is wasted.

**Fix**: Check your actual data. What's the 95th percentile of total tokens (prompt + generation)? Set max_model_len to that value + 20% buffer.

### Antipattern 3: Ignoring Prefix Cache Alignment

**Problem**: System prompt is 1,501 tokens. Block size is 16. The system prompt uses 93 full blocks (1,488 tokens) + 1 partial block (13 tokens). Any user query that starts differently by even 1 token is a cache miss for the last block.

**Fix**: Pad system prompts to a multiple of block_size. Add filler tokens to align.

### Antipattern 4: Multi-GPU = Better

**Problem**: Add more GPUs expecting linear scaling. Communication overhead kills throughput per GPU.

**Fix**: Always measure throughput PER GPU. If adding a GPU increases total throughput by < 40% of the previous, stop adding. Consider pipeline parallelism instead.

### Common vLLM Failures

| Failure | Symptom | Fix |
|---------|---------|-----|
| OOM under load | Process crashes after N requests | Reduce gpu_memory_utilization or max_num_seqs |
| Low GPU utilization | GPU compute < 30% | Increase max_num_seqs, enable prefix caching |
| High TTFT (time to first token) | Users wait before response starts | Reduce max_num_seqs, enable prefill optimization |
| Slow prompt processing | Long prompts take disproportionate time | Enable prefix caching, check block alignment |
| Multi-GPU scaling wall | Adding GPUs doesn't help linearly | Switch from tensor to pipeline parallelism |

---

## 🧪 Drills & Challenges

### Drill 1: Build a Capacity Planner (45 min)

**Goal**: Extend the capacity estimator to be more accurate and include cost.

**Task**: Build a function that:
1. Takes: model size, quantization, GPU type, expected request rate, avg prompt/generation tokens, target latency
2. Returns: recommended GPU count, estimated throughput, estimated monthly cost, and whether target latency is achievable

**Add these factors:**
- GPU memory bandwidth (A10G: 600 GB/s, A100: 2,000 GB/s, T4: 320 GB/s)
- Communication overhead for multi-GPU (10% per additional GPU for NVLink, 30% for PCIe)
- Burst factor (peak traffic is 2x average)

---

### Drill 2: Profile Your vLLM Deployment (45 min)

**Goal**: Identify bottlenecks in a running vLLM deployment.

**Run these diagnostics:**
1. GPU compute utilization — is the GPU busy or waiting?
2. KV cache utilization — how close to OOM?
3. Request queue depth — are requests piling up?
4. P99 latency over 24h — is there a pattern?

**For each metric, answer:**
1. What's a healthy range?
2. What threshold triggers an alert?
3. What configuration change fixes it if it's unhealthy?

---

### Drill 3: Design a Multi-GPU Strategy (30 min)

**Scenario**: You have 8x A10G GPUs available. You need to serve a 70B model.

**Options:**
- A) Tensor parallelism across all 8 GPUs (low latency, max throughput per GPU ~60% of single)
- B) Pipeline parallelism across 8 GPUs (higher throughput per GPU, higher latency per request)
- C) 2 groups of 4 GPUs tensor-parallel (balance of latency and throughput)
- D) 4 groups of 2 GPUs each, load balanced (max total throughput, worst latency)

**Task**: For EACH option, estimate:
1. Expected throughput (req/s)
2. P50 latency
3. Failure domain (what happens if 1 GPU fails in each configuration?)
4. Cost efficiency (throughput per dollar)

**Which option do you choose, and why?**

---

## 🚦 Gate Check

Before moving to File 03 (Caching with Redis), verify you can:

1. **Size the deployment** — For a 13B model serving 500 concurrent users, average prompt=1K tokens, average generation=500 tokens, target p99=3s: recommend GPU type, count, quantization, and configuration parameters

2. **Diagnose a bottleneck** — Given: GPU compute 25%, KV cache 95%, throughput 12 req/s on A10G. The GPU is barely computing but near OOM. What's the bottleneck? How do you fix it?

3. **Design for prefix caching** — Your system prompt is 2,047 tokens. Block size is 16. What's the cache efficiency? How would you restructure to hit 100%?

4. **Choose parallelism strategy** — For a 7B model (fits on 1 GPU) with 2 GPUs available: tensor parallelism OR 2 independent instances? Which gives better throughput? Which gives better latency per request?

5. **Plan capacity** — Given: 8K requests/hour during peak, avg 300 prompt + 150 generation tokens, target p99 < 2s on A10G. How many GPUs? What max_num_seqs? Is prefix caching critical?

---

## 📚 Resources

### vLLM Official
- **[vLLM Documentation](https://docs.vllm.ai/en/latest/)** — Comprehensive configuration guide
- **[vLLM Performance Tuning](https://docs.vllm.ai/en/latest/performance/)** — Official tuning guide

### Serving Architecture
- **[Continuous Batching Blog](https://www.anyscale.com/blog/continuous-batching-llm-inference)** — How continuous batching works
- **[PagedAttention Paper](https://arxiv.org/abs/2309.06180)** — The paper that made vLLM possible

### Phase Cross-References
- **Phase 9 (Fine-Tuning, File 08)** → Deployment with vLLM was introduced there. This file goes DEEPER into vLLM internals.
- **Phase 1 (Gateway)** → Your Phase 1 gateway routes to the vLLM server. Wire them together.
- **Phase 8 (Guardrails)** → Guardrails sit IN FRONT OF the model server. The vLLM server generates, guardrails validate.

> **Next up: File 03 — Caching with Redis.** GPU inference is expensive ($0.50-4/hr). Caching reduces GPU calls. A good cache strategy can cut your GPU bill by 60%+ without reducing quality. But caching LLM responses is NOT like caching HTTP responses — semantic similarity matters, not exact match.
