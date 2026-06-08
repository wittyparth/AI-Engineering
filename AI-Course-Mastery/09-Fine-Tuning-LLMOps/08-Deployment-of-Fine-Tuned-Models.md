# 08 — Deployment of Fine-Tuned Models: Serving at Scale

## 🎯 Purpose & Goals

> 🛑 STOP. Before you serve your fine-tuned model to real users, you need to understand a critical truth:

**A model that works on your laptop will FAIL in production.**

Not because the model is bad. Because production serving has constraints your laptop doesn't:
- Your laptop has 24GB GPU memory. Production serves 1,000 concurrent users.
- Your laptop has no latency requirements. Production needs p99 < 2 seconds.
- Your laptop loads ONE model. Production swaps between models per request.
- Your laptop is idle when you're not using it. Production costs $2-10/hour per GPU.
- Your laptop has one user (you). Production has users who WILL find the edge cases.

**The goal of this module: Learn how to deploy fine-tuned models to production — with the right serving infrastructure, quantization strategy, scaling approach, and cost optimization.**

---

### 🤔 Discovery Question 1: The Silent Throughput Drop

You deploy your fine-tuned 7B model with vLLM on a single A10G GPU. For the first day, everything works:
- Latency: 1.2s p50, 2.1s p99
- Throughput: 50 requests/minute
- No errors

**Day 2, 2:00 PM**: Latency starts climbing.
- 2:00 PM: p99 = 2.5s
- 2:15 PM: p99 = 3.8s
- 2:30 PM: p99 = 6.2s
- 2:45 PM: OOM errors start

You haven't changed anything. User traffic is the same as yesterday. Why is it slowing down?

**🤔 Before reading on, answer these:**

1. Your GPU memory is fixed (24GB). Your request rate is the same. Why would latency increase over time? (Hint: Think about what ACCUMULATES during serving. KV cache. Each request generates KV cache entries that persist across turns. If your model handles multi-turn conversations, every active conversation holds KV cache in memory. As conversations accumulate, less memory is available for new requests → queuing → latency spikes.)

2. You check `nvidia-smi` and see GPU memory usage is at 95% (was 60% on Day 1). What's filling the memory? The model weights are static (~4GB with 4-bit quantization). So the memory growth must be from something DYNAMIC. What serving mechanisms allocate memory that is NOT freed between requests? (Hint: KV cache blocks, request state, continuous batching slots.)

3. **The hardest question:** You're using vLLM which has a KV cache manager. The cache manager should evict old entries when memory is full. But why is it NOT evicting — or why is eviction not enough? **What could cause the KV cache to grow unbounded, and what configuration would fix it?** (Hint: max_num_seqs, max_model_len, gpu_memory_utilization, and whether the client is reusing connections.)

---

### 🤔 Discovery Question 2: The Quantization Cliff

You want to reduce costs. Your 7B model runs on 1x A10G (24GB). You could:
1. Quantize to 8-bit → runs on A10G, but 2x throughput
2. Quantize to 4-bit → could run on T4 (16GB), 4x cost reduction
3. Quantize to 2-bit → could run on CPU, minimal cost

You try 4-bit quantization. Model loads. Inference runs. But:

| Metric | FP16 | 4-bit | Change |
|--------|------|-------|--------|
| ROUGE-L | 0.47 | 0.43 | **-8.5%** |
| BERTScore | 0.85 | 0.82 | **-3.5%** |
| Perplexity | 2.8 | 3.4 | **+21%** |
| Exact Match | 0.41 | 0.36 | **-12%** |

You try 8-bit:
| ROUGE-L | 0.47 | 0.46 | **-2%** |
| BERTScore | 0.85 | 0.845 | **-0.6%** |

**🤔 Before reading on, answer these:**

1. 8-bit quantization costs 2% ROUGE-L drop. 4-bit costs 8.5% drop. The relationship is NOT linear — 2x compression costs 4x the quality loss. Why? What changes between 8-bit and 4-bit that causes a QUALITY CLIFF? (Hint: 8-bit has 256 quantization levels. 4-bit has 16. At 16 levels, each quantization bucket is very wide. Values near zero lose precision disproportionately.)

2. You notice that the 4-bit model performs WORSE on legal citations (exact match drops 12%) but ALMOST THE SAME on open-ended legal analysis (BERTScore drops 3.5%). Why does quantization affect DIFFERENT TASKS differently? (Hint: Exact match requires precise token-level predictions. Open-ended analysis allows many valid responses. Quantization noise matters more for precision tasks.)

3. **The hardest question:** You have 3 deployment targets with different quantization requirements:
   - Edge (mobile app): Needs 4-bit, runs on phone NPU
   - Web (real-time): Needs 8-bit, runs on T4 GPU
   - Batch (offline processing): FP16, runs on A100
   
   **How do you ensure that your EVALUATION captures quantization-aware quality?** If you evaluate only the FP16 model, you don't know how 4-bit performs on edge. If you evaluate all 3, you're spending 3x the eval budget. What's the minimum evaluation that gives you confidence in quantized deployment quality?

---

### 🤔 Discovery Question 3: The Cold Start Problem

You deploy your model on a serverless GPU platform (like AWS SageMaker, Banana, Replicate). The platform auto-scales: spins up new GPU instances when traffic increases, spins down when idle.

At 10:00 AM, a user sends a request. The platform has 0 active instances (idle scaling). It starts a new instance.

- 10:00:00 — Request received
- 10:00:05 — GPU instance starts (OS boot, Docker pull, etc.)
- 10:00:45 — Model loading starts (download weights from S3)
- 10:01:15 — Model loaded into GPU memory
- 10:01:20 — First token generated
- 10:01:25 — Response complete

**Total time: 85 seconds. The user's request timed out at 30 seconds.**

**🤔 Before reading on, answer these:**

1. Of the 85 seconds, only 5 seconds were actual inference. The other 80 seconds were: instance startup (40s) + model loading (30s) + network overhead (10s). **What strategies would reduce this cold start time?** Consider: pre-warmed instances, snapshot loading, model store location, keeping a minimum number of idle instances.

2. You add keep-alive: always keep 1 idle instance running. Cold starts are eliminated. But now you're paying $150/month for an idle GPU that does nothing 80% of the time. Your peak traffic is only 2 concurrent users. Is the $150/month worth it for eliminating a 30-second wait? At what point does it become worth it? **How do you calculate the cost-benefit of pre-warmed instances?**

3. **The hardest question:** A large enterprise customer does a DEMO of your product at 2:00 PM. At 1:55 PM, your auto-scaler scaled down to 0 instances (no traffic for 2 hours). The demo request triggers a cold start. The customer waits 85 seconds. They leave. You lost the deal.

   **Your CTO says: "We need to predict when customers will arrive and pre-warm instances."**
   **Your CFO says: "We need to eliminate GPU waste."**
   
   **Design a PREDICTIVE PRE-WARMING system that balances both concerns.** What signals would you use to predict upcoming demand? (Scheduled demos? Time-of-day patterns? User behavior signals?) How early would you warm? What's the acceptable false-positive rate (warming when no request comes)?

---

## ⏱ Time Budget

| Section | Time |
|---------|------|
| Discovery Questions | 30 min |
| Concept Deep-Dive | 1 hr |
| Code Examples (2) | 1.5 hr |
| Drills (3) | 45 min |
| Gate Check | 15 min |
| **Total** | **~4 hr** |

---

## 📖 Concept Deep-Dive

### 1. Serving Architecture Overview

```
                         ┌─────────────────┐
                         │  Load Balancer  │
                         │  (NGINX/ALB)    │
                         └────────┬────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
              ┌─────▼─────┐             ┌───────▼──────┐
              │ Inference  │             │  Inference   │
              │ Server 1   │             │  Server 2    │
              │ (vLLM)     │             │  (vLLM)      │
              │ GPU: A10G  │             │  GPU: A10G   │
              │ Model: FT  │             │  Model: FT   │
              └─────┬─────┘             └──────┬───────┘
                    │                          │
              ┌─────▼──────────────────────────▼───────┐
              │           Model Registry               │
              │    (stores model artifacts)            │
              └─────────────────────────────────────────┘
```

### 2. Serving Frameworks Comparison

| Framework | Pros | Cons | Best For |
|-----------|------|------|----------|
| **vLLM** | Fastest inference, PagedAttention, continuous batching, OpenAI-compatible API | Requires specific GPU/arch | Production serving of single model |
| **TGI (HF)** | Easy deployment, HF ecosystem integration | Slower than vLLM | Quick deploys, HF ecosystem users |
| **Llama.cpp** | CPU inference, all quantization types | Slower than GPU frameworks | Edge/local deployment |
| **Ollama** | Easiest setup, model management | Limited production features | Local development |
| **Ray Serve** | Multi-model, multi-GPU, auto-scaling | Complex setup | Multi-model serving |
| **SageMaker** | Managed, auto-scaling, monitoring | Vendor lock-in, expensive | Enterprise production |

### 3. Quantization Strategies for Deployment

| Method | Bit Width | Quality Impact | Speed Up | Use Case |
|--------|-----------|---------------|----------|----------|
| FP16 | 16 | None (training default) | 1x | Baseline |
| INT8 (GPTQ) | 8 | Minimal (~1-2%) | 1.5-2x | Cost-sensitive production |
| INT4 (GPTQ/AWQ) | 4 | Moderate (~3-8%) | 3-4x | Edge/mobile deployment |
| NF4 (QLoRA) | 4 | Good for FT models | 3-4x | When you have QLoRA weights |
| GGUF (llama.cpp) | 2-8 | Variable | Variable | CPU/local deployment |

**Key insight**: Quantize AFTER fine-tuning, not before. If you quantize before FT, the quantization noise interacts with training noise in unpredictable ways.

### 4. A/B Testing for Model Deployments

```
Request → Router ──50%──→ Model A (current production)
                 └──50%──→ Model B (candidate)
                              │
                         ┌────▼────┐
                         │  Eval   │
                         │ Logger  │
                         └─────────┘
                              │
                    Metrics Dashboard
                    ┌───┬───┬───┬───┐
                    │ A │ B │ Δ │ ✓ │
                    ├───┼───┼───┼───┤
                    │   │   │   │   │
                    └───┴───┴───┴───┘
```

**Proper A/B test requirements:**
1. **Stratified split** — Split traffic by user segment (not random) to avoid confounding
2. **Minimum sample size** — 385+ users per variant for 5% effect at 95% power
3. **Guardrail metrics** — Monitor latency, error rate, cost PER VARIANT
4. **Duration** — At least 1 full business cycle (include weekends if usage varies)
5. **Stopping rule** — Pre-commit to duration; don't stop early because results look good

### 5. Cost Optimization

| Strategy | Cost Reduction | Complexity | Risk |
|----------|---------------|------------|------|
| Quantization | 4x | Low | Quality loss |
| Batch inference | 10x (offline) | Low | Latency increase |
| GPU pooling | 2-3x | Medium | Cold starts |
| Spot instances | 2-3x | High | Instance termination |
| Model distillation | 2-5x | High | Training cost, quality loss |

---

## 💻 CODE EXAMPLES

### Example 1: The Broken Deployment Configuration (Full of Bugs)

```yaml
# WHAT NOT TO DO — A vLLM deployment config with 10+ bugs

model: ./ft-legal-model  # BUG 1: Local path — won't work in distributed deployment
                          # Should use S3/HF path or containerized model

# BUG 2: No quantization specified
# Defaults to FP16 — uses 14GB for 7B model
# Should use 4-bit or 8-bit for production

# BUG 3: No tensor_parallel_size
# Single GPU only — can't scale to larger models or multi-GPU

# BUG 4: No max_model_len
# Defaults to model's max (often 4096)
# If your use case needs 8192, this silently truncates

# BUG 5: No gpu_memory_utilization
# Defaults to 0.9 — but you don't know if that's optimal for your workload
# Too high → OOM under load. Too low → wasted memory.

serving:
  port: 8000
  
  # BUG 6: No authentication
  # Anyone who can reach this port can use the model
  # Should have API key or token-based auth
  
  # BUG 7: No rate limiting
  # A single user can DOS the service with many requests
  
  # BUG 8: No request queuing configuration
  # vLLM's default queue might be unbounded
  
  # BUG 9: No timeout
  # Requests hang forever if model hangs
  
  # BUG 10: No health check
  # Load balancer can't detect if service is unhealthy
  
  # BUG 11: No metrics endpoint
  # No way to monitor latency, throughput, errors
  
  # BUG 12: No logging
  # Can't debug failures
```

#### 🔍 All 12+ Bugs

| # | Bug | Impact | Fix |
|---|-----|--------|-----|
| 1 | Local model path | Only works on one machine | Use S3/HF model ID or containerize |
| 2 | No quantization | 2x memory usage, 2x cost | Set quantize=gptq or load 4-bit |
| 3 | No tensor_parallel | Can't use multi-GPU | Set tensor_parallel_size=2 for 2 GPUs |
| 4 | No max_model_len | Silent truncation | Set max_model_len based on your data |
| 5 | No gpu_memory_util | Suboptimal memory use | Set 0.85-0.95 based on workload |
| 6 | No auth | Anyone can use model | Add API key auth |
| 7 | No rate limit | DOS vulnerability | Add rate limiting |
| 8 | No queue config | Unbounded queuing | Set max_num_seqs, max_waiting |
| 9 | No timeout | Hanging requests | Set request_timeout |
| 10 | No health check | Blind deployment | Add /health endpoint |
| 11 | No metrics | No observability | Add Prometheus metrics |
| 12 | No logging | Can't debug | Add structured logging |

---

### Example 2: Production vLLM Deployment

```python
"""
Production-grade vLLM deployment with comprehensive configuration.

This is the COMPLETE serving setup:
1. vLLM server configuration
2. Docker container setup
3. Health checks and monitoring
4. Load testing
5. A/B testing infrastructure
"""

# ──────────────────────────────────────────────
# 1. vLLM SERVER CONFIGURATION
# ──────────────────────────────────────────────

# config.yaml — vLLM server configuration
"""
model: s3://models/legal-summarization/v2/  # Remote model path
tokenizer: s3://models/legal-summarization/v2/tokenizer/

# Quantization
quantization: awq                    # AWQ 4-bit quantization
dtype: auto                          # Auto-detect dtype from config

# Memory management
max_model_len: 8192                  # Max context length
gpu_memory_utilization: 0.90         # Reserve 10% for KV cache overhead
max_num_seqs: 256                    # Max concurrent sequences
max_num_batched_tokens: 8192         # Max tokens per batch

# Performance
tensor_parallel_size: 1              # Single GPU (use 2+ for 13B+ models)
block_size: 16                       # PagedAttention block size
enable_prefix_caching: true          # Cache common prefixes (system prompts)
use_v2_block_manager: true           # Use v2 block manager (better memory)

# Serving
port: 8000
host: 0.0.0.0

# API
api_key: ${VLLM_API_KEY}             # Auth required
served_model_name: legal-summarization-v2

# Scheduling
max_num_seqs: 256
max_waiting_tokens: 20               # Max tokens to wait for batching
scheduler: max_throughput             # Optimize for throughput (vs. latency)

# Logging
log_level: info
log_requests: true                    # Log all requests for debugging

# Trust remote code (for custom model architectures)
trust_remote_code: false
"""

# ──────────────────────────────────────────────
# 2. DOCKER DEPLOYMENT
# ──────────────────────────────────────────────

# Dockerfile
"""
FROM vllm/vllm-openai:latest

# Install additional dependencies
RUN pip install awscli boto3 prometheus-client

# Copy model download script
COPY download_model.py /app/download_model.py
RUN python /app/download_model.py

# Health checks
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \\
    CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000

ENTRYPOINT ["python3", "-m", "vllm.entrypoints.openai.api_server"]
CMD ["--config", "/app/config.yaml"]
"""

# docker-compose.yaml
"""
version: '3.8'

services:
  model-server:
    build: .
    ports:
      - "8000:8000"
    environment:
      - VLLM_API_KEY=${VLLM_API_KEY}
      - HF_TOKEN=${HF_TOKEN}
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
    volumes:
      - model_cache:/root/.cache/huggingface
      - ./config.yaml:/app/config.yaml
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GF_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  model_cache:
  grafana_data:
"""

# ──────────────────────────────────────────────
# 3. DEPLOYMENT SCRIPT
# ──────────────────────────────────────────────

#!/usr/bin/env python3
"""
Deploy a model from the registry to production.

Usage:
    python deploy.py --model-id legal-summarization-v2-a3f8c2e --stage canary
"""

import json
import subprocess
import time
import argparse
import logging
from pathlib import Path
from typing import Optional
import requests

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class ModelDeployer:
    """
    Deploy models from registry to serving infrastructure.
    """
    
    def __init__(self, registry_db: str = "model_registry.db"):
        self.registry_db = registry_db
    
    def get_model_config(self, model_id: str) -> dict:
        """Get model configuration from registry."""
        import sqlite3
        
        with sqlite3.connect(self.registry_db) as conn:
            conn.row_factory = sqlite3.Row
            cursor = conn.execute(
                "SELECT * FROM models WHERE model_id = ?", (model_id,)
            )
            row = cursor.fetchone()
            if not row:
                raise ValueError(f"Model {model_id} not found in registry")
            
            return {
                "model_id": row["model_id"],
                "model_path": row["model_path"],
                "tokenizer_path": row["tokenizer_path"],
                "model_name": row["model_name"],
                "version": row["version"],
                "eval_metrics": json.loads(row["eval_metrics"]),
            }
    
    def health_check(self, url: str) -> bool:
        """Check if the serving endpoint is healthy."""
        try:
            response = requests.get(f"{url}/health", timeout=5)
            return response.status_code == 200
        except Exception as e:
            logger.warning(f"Health check failed: {e}")
            return False
    
    def wait_for_ready(
        self, url: str, timeout_seconds: int = 300, interval: int = 10
    ) -> bool:
        """Wait for the serving endpoint to become healthy."""
        start = time.time()
        while time.time() - start < timeout_seconds:
            if self.health_check(url):
                logger.info(f"Model ready at {url}")
                return True
            logger.info(f"Waiting for model... ({int(time.time() - start)}s)")
            time.sleep(interval)
        
        logger.error(f"Model not ready after {timeout_seconds}s")
        return False
    
    def test_inference(self, url: str, api_key: str) -> dict:
        """Run a test inference to verify the model works."""
        headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
        }
        
        payload = {
            "model": "default",
            "messages": [
                {"role": "user", "content": "What is the legal definition of negligence?"}
            ],
            "max_tokens": 100,
            "temperature": 0.1,
        }
        
        try:
            response = requests.post(
                f"{url}/v1/chat/completions",
                headers=headers,
                json=payload,
                timeout=30,
            )
            if response.status_code == 200:
                data = response.json()
                return {
                    "success": True,
                    "latency_ms": data.get("usage", {}).get("total_tokens", 0),
                    "response_preview": data["choices"][0]["message"]["content"][:100],
                }
            else:
                return {"success": False, "error": response.text}
        except Exception as e:
            return {"success": False, "error": str(e)}
    
    def deploy_model(
        self,
        model_id: str,
        stage: str,
        docker_compose_file: str = "docker-compose.yaml",
    ) -> bool:
        """
        Deploy a model to the specified stage.
        
        This script:
        1. Gets model config from registry
        2. Updates docker-compose with correct model path
        3. Starts the model server
        4. Waits for it to become healthy
        5. Runs test inference
        6. Reports status back to registry
        """
        
        logger.info(f"Deploying {model_id} to {stage}")
        
        # 1. Get model config
        config = self.get_model_config(model_id)
        logger.info(f"Model: {config['model_name']} v{config['version']}")
        logger.info(f"Path: {config['model_path']}")
        
        # 2. Update serving config
        # In production, this would update Kubernetes manifests,
        # docker-compose, Terraform configs, etc.
        self._update_serving_config(config)
        
        # 3. Deploy
        logger.info("Starting model server...")
        result = subprocess.run(
            ["docker-compose", "-f", docker_compose_file, "up", "-d"],
            capture_output=True,
            text=True,
        )
        
        if result.returncode != 0:
            logger.error(f"Deploy failed: {result.stderr}")
            return False
        
        # 4. Wait for health
        url = "http://localhost:8000"
        if not self.wait_for_ready(url):
            logger.error("Deployment failed: model not healthy")
            self.rollback(model_id)
            return False
        
        # 5. Test inference
        test_result = self.test_inference(url, "test-key")
        if not test_result["success"]:
            logger.error(f"Test inference failed: {test_result.get('error')}")
            self.rollback(model_id)
            return False
        
        logger.info(f"Test inference: {test_result['response_preview']}...")
        
        # 6. Update registry stage
        if stage != "dev":
            self._update_registry_stage(model_id, stage)
        
        logger.info(f"Deployment of {model_id} to {stage} SUCCESS")
        return True
    
    def _update_serving_config(self, config: dict):
        """Update serving configuration for the new model."""
        # In production, this would:
        # - Update Kubernetes ConfigMap
        # - Update docker-compose environment
        # - Update Terraform variables
        # - Update load balancer routing
        logger.info(f"Updated serving config for {config['model_id']}")
    
    def _update_registry_stage(self, model_id: str, new_stage: str):
        """Update model stage in registry."""
        import sqlite3
        
        with sqlite3.connect(self.registry_db) as conn:
            conn.execute(
                "UPDATE models SET stage = ? WHERE model_id = ?",
                (new_stage, model_id),
            )
            conn.execute("""
                INSERT INTO stage_transitions 
                (model_id, from_stage, to_stage, changed_by, changed_at, reason)
                VALUES (?, 'deploying', ?, 'deployer', ?, ?)
            """, (
                model_id, new_stage,
                time.strftime("%Y-%m-%dT%H:%M:%S"),
                f"Deployed via deployment pipeline to {new_stage}",
            ))
        
        logger.info(f"Updated registry: {model_id} → {new_stage}")
    
    def rollback(self, model_id: str):
        """Rollback a failed deployment."""
        logger.warning(f"Rolling back deployment of {model_id}")
        
        # Stop the failed server
        subprocess.run(["docker-compose", "down"], capture_output=True)
        
        # Restore previous deployment
        # In production: redeploy previous model version
        logger.info("Rollback complete")


# ──────────────────────────────────────────────
# 4. LOAD TESTING
# ──────────────────────────────────────────────

import asyncio
import aiohttp
from dataclasses import dataclass
from statistics import mean, median, pstdev

@dataclass
class LoadTestResult:
    """Results from a load test."""
    total_requests: int
    successful: int
    failed: int
    latency_p50_ms: float
    latency_p95_ms: float
    latency_p99_ms: float
    throughput_req_per_sec: float
    error_rate: float
    duration_seconds: float


class LoadTester:
    """
    Load test a model serving endpoint.
    
    Simulates production traffic to validate:
    - Throughput under load
    - Latency distribution (p50, p95, p99)
    - Error rate
    - Memory stability (no OOM over time)
    """
    
    def __init__(self, url: str, api_key: str, concurrency: int = 10):
        self.url = url
        self.api_key = api_key
        self.concurrency = concurrency
    
    async def send_request(
        self, session: aiohttp.ClientSession, prompt: str
    ) -> dict:
        """Send a single inference request."""
        headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json",
        }
        
        payload = {
            "model": "default",
            "messages": [
                {"role": "user", "content": prompt}
            ],
            "max_tokens": 200,
            "temperature": 0.1,
        }
        
        start = time.time()
        try:
            async with session.post(
                f"{self.url}/v1/chat/completions",
                headers=headers,
                json=payload,
                timeout=aiohttp.ClientTimeout(total=60),
            ) as response:
                latency = (time.time() - start) * 1000  # ms
                
                if response.status == 200:
                    return {"success": True, "latency_ms": latency}
                else:
                    return {"success": False, "latency_ms": latency, "error": await response.text()}
        except Exception as e:
            return {"success": False, "latency_ms": 0, "error": str(e)}
    
    async def run_load_test(
        self,
        prompts: list,
        duration_seconds: int = 60,
    ) -> LoadTestResult:
        """Run a load test for a specified duration."""
        
        connector = aiohttp.TCPConnector(limit=self.concurrency)
        async with aiohttp.ClientSession(connector=connector) as session:
            
            start_time = time.time()
            tasks = []
            results = []
            request_count = 0
            
            while time.time() - start_time < duration_seconds:
                # Fan out concurrent requests
                batch_tasks = []
                for _ in range(self.concurrency):
                    prompt = prompts[request_count % len(prompts)]
                    batch_tasks.append(self.send_request(session, prompt))
                    request_count += 1
                
                batch_results = await asyncio.gather(*batch_tasks)
                results.extend(batch_results)
                
                # Small delay to control rate
                await asyncio.sleep(0.1)
            
            actual_duration = time.time() - start_time
            
            # Process results
            latencies = [
                r["latency_ms"] for r in results
                if r["success"] and r["latency_ms"] > 0
            ]
            successful = sum(1 for r in results if r["success"])
            failed = len(results) - successful
            
            latencies.sort()
            
            return LoadTestResult(
                total_requests=len(results),
                successful=successful,
                failed=failed,
                latency_p50_ms=median(latencies) if latencies else 0,
                latency_p95_ms=latencies[int(len(latencies) * 0.95)] if latencies else 0,
                latency_p99_ms=latencies[int(len(latencies) * 0.99)] if latencies else 0,
                throughput_req_per_sec=successful / actual_duration,
                error_rate=failed / len(results) if results else 1.0,
                duration_seconds=actual_duration,
            )


# ──────────────────────────────────────────────
# 5. A/B TESTING INFRASTRUCTURE
# ──────────────────────────────────────────────

class ABTestRouter:
    """
    Route requests between model variants for A/B testing.
    
    Features:
    - Configurable traffic split
    - User-consistent routing (same user → same variant)
    - Metrics collection per variant
    - Automatic rollback on metric degradation
    """
    
    def __init__(self, registry_db: str = "model_registry.db"):
        self.registry_db = registry_db
        self.variants = {}  # variant_name → endpoint_url
    
    def register_variant(
        self, name: str, endpoint_url: str, weight: float = 0.5
    ):
        """Register a model variant for A/B testing."""
        self.variants[name] = {
            "url": endpoint_url,
            "weight": weight,
        }
    
    def get_variant(self, user_id: str) -> str:
        """
        Determine which variant a user gets.
        
        Uses consistent hashing: same user_id → same variant
        (unless weights change).
        """
        import hashlib
        
        # Hash user_id to a deterministic value
        hash_val = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
        normalized = hash_val / (2 ** 128)  # 0.0 to 1.0
        
        # Assign based on cumulative weights
        cumulative = 0.0
        for name, config in self.variants.items():
            cumulative += config["weight"]
            if normalized <= cumulative:
                return name
        
        return list(self.variants.keys())[-1]
    
    def track_metric(
        self,
        variant: str,
        metric_name: str,
        value: float,
        user_id: str,
    ):
        """Log a metric for A/B test analysis."""
        # In production: write to Prometheus, CloudWatch, or custom DB
        logger.info(f"A/B METRIC: variant={variant}, metric={metric_name}, value={value:.4f}")


# ──────────────────────────────────────────────
# USAGE
# ──────────────────────────────────────────────

if __name__ == "__main__":
    """
    Complete deployment workflow.
    
    Usage:
        python deploy.py --model-id legal-summarization-v2-a3f8c2e --stage canary --load-test
    
    This demonstrates:
    1. Deploy model from registry
    2. Health check
    3. Test inference
    4. Load test
    5. A/B test setup
    """
    
    parser = argparse.ArgumentParser()
    parser.add_argument("--model-id", required=True)
    parser.add_argument("--stage", default="canary", choices=["dev", "staging", "canary", "production"])
    parser.add_argument("--load-test", action="store_true", help="Run load test after deploy")
    parser.add_argument("--concurrency", type=int, default=10, help="Load test concurrency")
    parser.add_argument("--duration", type=int, default=30, help="Load test duration (seconds)")
    args = parser.parse_args()
    
    # 1. Deploy
    deployer = ModelDeployer()
    success = deployer.deploy_model(args.model_id, args.stage)
    
    if not success:
        logger.error("Deployment failed!")
        exit(1)
    
    # 2. Load test (optional)
    if args.load_test:
        tester = LoadTester(
            url="http://localhost:8000",
            api_key="test-key",
            concurrency=args.concurrency,
        )
        
        test_prompts = [
            "What is the legal definition of negligence?",
            "Summarize this contract clause: [CLause text]",
            "What are the elements of a valid contract?",
            "Explain the difference between tort and criminal law.",
            "What is the statute of limitations for breach of contract?",
        ] * 20
        
        result = asyncio.run(
            tester.run_load_test(test_prompts, duration_seconds=args.duration)
        )
        
        print(f"\n=== Load Test Results ===")
        print(f"Total requests: {result.total_requests}")
        print(f"Successful: {result.successful}")
        print(f"Failed: {result.failed}")
        print(f"Error rate: {result.error_rate:.2%}")
        print(f"Throughput: {result.throughput_req_per_sec:.1f} req/s")
        print(f"p50 latency: {result.latency_p50_ms:.0f}ms")
        print(f"p95 latency: {result.latency_p95_ms:.0f}ms")
        print(f"p99 latency: {result.latency_p99_ms:.0f}ms")
        print(f"Duration: {result.duration_seconds:.1f}s")
    
    print(f"\n✅ Deployment of {args.model_id} to {args.stage} complete!")
```

---

### 🔍 Critical Questions

1. **Health check vs. readiness probe**: The health check tests if the server is running. But a server can be "healthy" and still serve BAD responses (e.g., model loaded corrupted weights). What additional checks would you add to a readiness probe beyond "server responds to /health"?

2. **Load test distribution**: The load test uses SIMPLE prompts (what is legal definition of...). But real traffic has VARIABLE prompt lengths (some 10 tokens, some 4,000 tokens). How would you design a load test that reflects REAL traffic patterns, not just worst-case or average-case?

3. **Concurrent deploys**: The deployment script assumes EXCLUSIVE access to the target environment. What happens if someone else deploys at the same time? How would you add deployment locking to prevent concurrent deployments?

---

## ✅ Good Output Examples

### What a Healthy Production Deployment Looks Like

```
=== Deployment Status: legal-summarization-v2 ===

Infrastructure:
  - Server: vLLM 0.4.2 on A10G (24GB)
  - Quantization: AWQ 4-bit
  - Max concurrency: 256 sequences
  - Max context: 8192 tokens

Performance (24h):
  ┌──────────────┬──────────┬──────────┬──────────┐
  │ Metric       │ p50      │ p95      │ p99      │
  ├──────────────┼──────────┼──────────┼──────────┤
  │ Latency      │ 0.8s     │ 1.5s     │ 2.1s     │
  │ TTFT         │ 0.3s     │ 0.6s     │ 0.9s     │
  │ Throughput   │ 45 req/s │ 62 req/s │ 78 req/s │
  └──────────────┴──────────┴──────────┴──────────┘

Errors (24h):
  - Total errors: 3 / 64,512 requests (0.005%)
  - Timeouts: 0
  - OOM: 0
  - Invalid responses: 0

GPU Utilization:
  ┌─────────────────────┬────────┐
  │ GPU Memory Usage    │ 87%    │
  │ GPU Compute Usage   │ 72%    │
  │ KV Cache Usage      │ 45%    │
  └─────────────────────┴────────┘

Cost:
  - GPU: $1.50/hr (A10G on-demand)
  - Total: $36/day
  - Cost per request: $0.00056
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Production = Laptop Config

**Problem**: The same config that works on your dev machine is used in production. Dev machine has 1 user (you). Production has 1,000 concurrent users. Everything breaks.

**Fix**: Create a SEPARATE production config with: higher concurrency, lower GPU memory utilization (reserve headroom), health checks, timeouts, and auto-scaling.

### Antipattern 2: No Load Testing Before Deployment

**Problem**: Model passes unit tests (1 request) so it's deployed. First spike kills the server because KV cache fills up and OOM.

**Fix**: Run load tests at 50%, 100%, and 150% of expected peak traffic BEFORE deploying.

### Antipattern 3: One Model, One Server

**Problem**: Fine-tuned model A and fine-tuned model B each run on separate GPU servers. Each server is 40% utilized. You're paying for 2 GPUs when 1 could handle both.

**Fix**: Use multi-model serving (vLLM supports model multiplexing) or route requests to a shared pool of GPUs.

### Antipattern 4: Ignoring Cold Starts

**Problem**: Serverless deployment auto-scales down to 0. First request of the day takes 85 seconds. Users leave.

**Fix**: Keep minimum N warm instances (N=1 for low traffic, N=expected peak / instance capacity for high traffic). Calculate the cost of idle instances vs. cost of lost users.

### Common Deployment Failures

| Failure | Symptom | Fix |
|---------|---------|-----|
| KV cache OOM | Latency spikes, then errors under sustained load | Reduce max_num_seqs, enable prefix caching |
| Model loading timeout | Server starts but never becomes healthy | Pre-download models, use snapshot loading |
| Quantization quality loss | Production metrics lower than eval metrics | Run quantization-aware evaluation |
| Request timeout | Slow queries fail silently | Set appropriate timeouts, implement streaming |
| Connection pool exhaustion | New connections fail under load | Increase connection pool size, reuse connections |
| GPU memory fragmentation | Memory free but allocation fails | Restart server periodically (scheduled) |

---

## 🧪 Drills & Challenges

### Drill 1: Build a Cost Calculator (30 min)

**Goal**: Calculate the monthly cost of serving a fine-tuned model based on usage patterns.

**Task**: Write a function that calculates monthly serving cost:

```python
def calculate_monthly_cost(
    model_size_billions: int,
    quantization_bits: int,
    requests_per_day: int,
    avg_tokens_per_request: int,
    gpu_hourly_cost: float,  # e.g., $1.50 for A10G
    target_latency_p99_ms: int,  # e.g., 2000
) -> dict:
    """
    Calculate monthly serving cost.
    
    Returns:
    - Recommended GPU type
    - Number of GPUs needed
    - Monthly GPU cost
    - Cost per request
    - Whether target latency is achievable
    """
    pass
```

**Use these reference costs:**
| GPU | Memory | Hourly Cost (on-demand) |
|-----|--------|------------------------|
| T4 | 16GB | $0.50 |
| L4 | 24GB | $0.75 |
| A10G | 24GB | $1.50 |
| A100 | 80GB | $4.00 |

**Questions:**
1. At what request volume does it become cheaper to use a single A100 instead of 2x A10G?
2. Does quantization (4-bit) reduce cost enough to justify the quality loss?
3. How does prompt length affect cost differently from completion length?

---

### Drill 2: Monitoring Dashboard Setup (45 min)

**Goal**: Configure Prometheus metrics collection and a Grafana dashboard for model serving.

**Task**: Write a Prometheus config and Grafana dashboard JSON that monitors:

1. Request latency (p50, p95, p99) — per model variant
2. Throughput (requests/second) — per model variant
3. Error rate (%) — per model variant
4. GPU memory utilization (%)
5. KV cache utilization (%)
6. Queue depth (pending requests)

**Prometheus config (prometheus.yml):**
```yaml
scrape_configs:
  - job_name: 'vllm'
    scrape_interval: 15s
    static_configs:
      - targets: ['model-server:8000']
```

**Questions:**
1. What alert thresholds would you set for each metric?
2. What's the difference between alerting on p95 vs p99 latency?
3. How do you distinguish "traffic spike" from "model degradation"?

---

### Drill 3: Design a Multi-Region Deployment (45 min)

**Goal**: Design a deployment strategy for users across the US and Europe.

**Scenario**:
- US users: 70% of traffic, mostly from East Coast
- Europe users: 30% of traffic, from London/Frankfurt
- Latency requirement: p99 < 3s for ALL users
- Budget: $2000/month for GPU compute

**Task**: Design a deployment architecture that:
1. Minimizes latency for both regions
2. Stays within budget
3. Handles failover (if US region goes down, EU handles traffic)
4. Maintains model consistency across regions

**Hints:**
- Model weights can be stored in S3 (US) and replicated to Europe
- Each region needs its own GPU instances
- Global load balancer (Route53/Cloudflare) routes users to nearest region
- Consider: do you deploy identical models in both regions, or regional-specific models?

---

## 🚦 Gate Check

Before moving to File 09 (Project: Custom Fine-Tuned Model), verify you can:

1. **Design a serving architecture** — For a 13B fine-tuned model serving 500 requests/minute with p99 < 2s: specify GPU type, number of GPUs, quantization, expected cost, scaling strategy

2. **Plan a deployment pipeline** — From "model registered in registry" to "serving production traffic": design the CI/CD pipeline with: staging deployment, canary testing, production promotion, rollback procedure

3. **Design load test** — For a summarization model with prompt lengths from 100-4,000 tokens: design a load test that represents REAL traffic (not just average), specify: test prompts, concurrency levels, duration, pass/fail criteria

4. **Calculate cost** — Given: 7B model, AWQ 4-bit, 10,000 requests/day, average 1,500 tokens/request, target p99=2s. Calculate: required GPU, monthly cost, cost per request, and whether the target is achievable

5. **Debug a production issue** — Given: latency suddenly increased from 1.2s to 8.5s p99, no traffic change, no deployment change. What do you check? (List in order of priority.)

---

## 📚 Resources

### Serving Frameworks
- **[vLLM Documentation](https://docs.vllm.ai/en/latest/)** — Production inference serving
- **[TGI (Text Generation Inference)](https://huggingface.co/docs/text-generation-inference/en/index)** — HuggingFace's serving solution
- **[Llama.cpp](https://github.com/ggerganov/llama.cpp)** — CPU/edge inference

### Deployment Patterns
- **[Blue/Green Deployment](https://martinfowler.com/bliki/BlueGreenDeployment.html)** — Zero-downtime deployments
- **[Canary Releases](https://martinfowler.com/bliki/CanaryRelease.html)** — Gradual traffic shifting
- **[Kubernetes for ML](https://kubernetes.io/blog/2022/08/23/machine-learning-model-serving-kubernetes/)** — Orchestrating model deployments

### Monitoring
- **[Prometheus + Grafana for ML](https://prometheus.io/docs/visualization/grafana/)** — Metrics collection and visualization
- **[OpenTelemetry](https://opentelemetry.io/)** — Distributed tracing for ML systems

### Phase Cross-References
- **Phase 10 (Infrastructure)** → This module sets the stage for Phase 10. Full deployment infrastructure (VPC, ECS, RDS, Redis) is covered in Phase 10.
- **Phase 7 (Evals)** → Deployed models need CONTINUOUS evaluation. Wire your Phase 7 eval platform to monitor production model quality.
- **Phase 8 (Guardrails)** → Deployed models run behind guardrails. Your Phase 8 guardrails intercept inputs and outputs BEFORE they reach/leave the deployed model.
- **Phase 1 (Gateway)** → Your Phase 1 LLM Gateway can be extended to route to your fine-tuned vLLM endpoint instead of OpenAI/Anthropic.

> **Next up: File 09 — Project: Custom Fine-Tuned Model.** Everything comes together. You'll fine-tune a real model for a real use case, evaluate it, register it, and deploy it. This is your portfolio project for Phase 9.
