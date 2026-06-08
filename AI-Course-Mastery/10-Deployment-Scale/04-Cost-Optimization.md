# 04 — Cost Optimization: Cutting GPU Bills by 80%

## 🎯 Purpose & Goals

> 🛑 STOP. Before you spend money on GPU instances, you need to understand the single biggest cost trap in AI serving:

**Your GPU is IDLE most of the time.**

Not "your GPU is computing slowly." IDLE. Wasting money. Doing nothing.

Average GPU utilization in production AI services: **15-30%**.

The GPU is waiting for:
- Data to be loaded from CPU memory (PCIe bandwidth bottleneck)
- The next request to arrive (idle between requests)
- Other GPUs to finish synchronizing (communication overhead)
- The next batch to fill up (batching inefficiency)
- Cache lookups to complete (I/O wait)

Every minute your GPU spends waiting is a minute you're paying for but NOT getting value from.

**The goal of this module:** Identify every source of GPU cost waste and eliminate it — without degrading quality.

---

### 🤔 Discovery Question 1: The Spot Instance Gambit

You run your AI service on 4x A10G on-demand instances. Cost: $6/hour ($4,320/month).

Your cloud provider offers SPOT instances: same GPU, 70% cheaper, but can be terminated with 2 minutes notice.

You switch to spot instances. Cost drops to $1.80/hour. You're saving $3,024/month.

**Day 1-5:** No interruptions. You think "spot instances are fine."
**Day 6:** AWS reclaims 2 of your 4 spot instances. Your service capacity drops 50%. Latency spikes. Some requests time out.
**Day 7:** Another interruption. 3 instances reclaimed. Service is running on 1 GPU — barely handling traffic.
**Day 8:** ALL instances reclaimed during peak hours. Service DOWN for 12 minutes while on-demand replacements spin up.

**🤔 Before reading on, answer these:**

1. Spot instances saved 70% but caused a 12-minute outage. For a customer-facing service, 12 minutes of downtime might cost MORE than the savings (lost revenue, damaged reputation). **How do you calculate whether the savings are worth the risk?** What's the formula that compares "70% cost reduction" vs. "X minutes of expected downtime per month"?

2. You could run a MIX: 3 spot + 1 on-demand per region. If all 3 spots are reclaimed, the 1 on-demand handles minimum traffic while new spots spin up. **What's the optimal spot/on-demand ratio that minimizes cost while guaranteeing N-1 capacity?** How does this change if your traffic is predictable vs. spiky?

3. **The hardest question:** Spot interruption is RANDOM — you can't predict when instances will be reclaimed. But you CAN prepare: (a) pre-download model weights to EFS so new instances start fast, (b) pre-warm replacement instances, (c) drain connections before termination. **Design a "spot instance survival kit" — the minimum infrastructure needed to survive spot interruptions with <30 seconds of impact per interruption.** What's the architecture?

---

### 🤔 Discovery Question 2: The Provider Arbitrage Trap

You use OpenAI's API. Cost: $10/1M input tokens, $30/1M output tokens. Monthly bill: $5,000.

You think: "I'll run my own fine-tuned model on a GPU. It'll be cheaper."

You set up:
- 1x A10G on-demand ($1.50/hr) → $1,080/month
- Model: Mistral-7B (fine-tuned, quantized)
- Throughput: ~50 req/s

**Monthly cost:** $1,080 vs. $5,000 OpenAI. You're saving 78%!

**But 3 months later:**
- GPU maintenance: 8 hours/month troubleshooting crashes, OOMs, driver updates
- Model updates: 2 days/month of engineering time for retraining
- Scaling: need 2 more GPUs for peak traffic → $3,240/month

**Total real cost:** $1,080 (GPU) + $1,600 (engineering at $100/hr) + $200 (maintenance) = $2,880/month. Plus you're on call.

**🤔 Before reading on, answer these:**

1. The direct GPU cost was $1,080/month vs. $5,000 OpenAI. But the HIDDEN costs (engineering time, maintenance, on-call) added $1,800/month. **What's the "total cost of ownership" (TCO) formula for self-hosted vs. API-based inference?** At what request volume does self-hosting become cheaper when you include engineering time?

2. OpenAI reduces its prices by 20% every 6 months. Your GPU costs stay FLAT. After 2 years, OpenAI's price is $6.40/1M tokens. Your self-hosted cost is still $2,880/month. **Is self-hosting still cheaper?** At what point does the managed API become cheaper than self-hosting? How do you model price TRENDS in your decision?

3. **The hardest question:** You use MULTIPLE providers:
   - OpenAI for simple queries (cheaper per token)
   - Self-hosted GPU for fine-tuned model (fixed cost, zero marginal cost)
   - Anthropic for complex reasoning (better quality, higher cost)
   
   Each request should go to the CHEAPEST provider that can handle it adequately. **Design a ROUTER that classifies each request and sends it to the optimal provider based on complexity, cost, and latency requirements.** What signals does the router use to decide?

---

### 🤔 Discovery Question 3: The Quantization Cost/Benefit Puzzle

You're deciding between quantization levels for cost optimization:

| Quantization | GPU | Hourly Cost | Quality (ROUGE-L) | Throughput |
|-------------|-----|-------------|-------------------|------------|
| FP16 | A10G (24GB) | $1.50 | 0.47 (baseline) | 45 req/s |
| INT8 (GPTQ) | T4 (16GB) | $0.50 | 0.46 (-2%) | 52 req/s |
| INT4 (AWQ) | T4 (16GB) | $0.50 | 0.43 (-8.5%) | 68 req/s |
| INT4 (AWQ) | L4 (24GB) | $0.75 | 0.43 (-8.5%) | 85 req/s |

**Cost per 1,000 requests:**
| Config | Cost | Quality Loss |
|--------|------|-------------|
| A10G FP16 | $0.0093 | 0% |
| T4 INT8 | $0.0027 | +2% drop |
| T4 INT4 | $0.0020 | +8.5% drop |
| L4 INT4 | $0.0025 | +8.5% drop |

**🤔 Before reading on, answer these:**

1. T4 INT8 costs 71% less per 1,000 requests than A10G FP16, with only 2% quality loss. That seems like an easy win. But T4 has 16GB memory vs. A10G's 24GB. With 16GB, you can serve FEWER concurrent sequences before OOM. **What's the HIDDEN cost of lower GPU memory?** How do you calculate the tradeoff between per-request cost and concurrency capacity?

2. T4 INT4 costs 78% less than A10G FP16 but loses 8.5% quality. For some use cases (creative writing, brainstorming), 8.5% quality loss is acceptable. For others (legal analysis, medical advice), it's not. **How do you build a QUALITY-AWARE cost optimizer that routes requests to the cheapest GPU that meets the quality bar for THAT request type?**

3. **The hardest question:** You run ALL configurations in a multi-tier GPU pool:
   - Simple queries → T4 INT4 ($0.002/1K) — 50% of traffic
   - Medium queries → T4 INT8 ($0.0027/1K) — 30% of traffic
   - Complex queries → A10G FP16 ($0.0093/1K) — 20% of traffic
   
   Your weighted average cost: 0.5 × $0.002 + 0.3 × $0.0027 + 0.2 × $0.0093 = **$0.0036/1K** — 61% cheaper than all-on-A10G.
   
   **But:** You need a CLASSIFIER that categorizes each request as simple/medium/complex BEFORE running inference. If the classifier costs $0.001/request (an API call) and makes mistakes 5% of the time, at what point does the classification overhead EAT the savings?

---

## ⏱ Time Budget

| Section | Time |
|---------|------|
| Discovery Questions | 30 min |
| Concept Deep-Dive | 30 min |
| Code Examples | 1 hr |
| Drills | 30 min |
| Gate Check | 15 min |
| **Total** | **~3 hr** |

---

## 📖 Concept Deep-Dive

### 1. GPU Cost Breakdown

```
Monthly GPU Cost = Hours × (Instance Cost + Storage Cost + Data Transfer)

Instance Cost:
  On-demand: $0.50-4.00/hr per GPU
  Spot: 60-80% cheaper, but interruptible
  Reserved: 30-60% cheaper, 1-3 year commitment

Hidden Costs:
  Engineering time: $50-150/hr for debugging/maintenance
  Storage: EBS/EFS for model weights, logs ($0.08-0.30/GB-month)
  Data transfer: $0.09/GB outbound (cross-region, internet)
  Monitoring: CloudWatch, Grafana Cloud, Datadog
```

### 2. Cost Optimization Levers (Ranked by Impact)

| Lever | Savings | Complexity | Risk | Implementation Time |
|-------|---------|------------|------|-------------------|
| Caching (File 03) | 40-70% | Medium | Quality degradation | 2-3 days |
| Spot instances | 60-70% | Low | Interruption risk | 1 day |
| Quantization | 50-75% | Low | Quality loss (2-8%) | 1 day |
| Provider arbitrage | 30-50% | Medium | Latency increase | 1 week |
| Auto-scaling | 20-40% | Medium | Cold starts | 2 days |
| Model distillation | 40-60% | High | Training cost | 2-4 weeks |
| Batch processing | 50-80% | Medium | Latency increase | 1 day |
| Reserved instances | 30-60% | Low | 1-year commitment | Negotiation |

### 3. The Cost Efficiency Formula

```
Cost per good request = Total monthly cost / (Total requests × Quality-adjusted factor)

Where:
- Total monthly cost = GPU + storage + data transfer + engineering + monitoring
- Quality-adjusted factor = 1.0 for baseline quality, 0.92 for 8% quality loss

Example:
  A10G FP16: $4,320/month for 500K requests → $0.0086/request (baseline quality)
  T4 INT4: $1,440/month for 400K requests (less memory) at 0.92 quality → $0.0039/quality-adjusted-request
```

---

## 💻 CODE EXAMPLES

### Example 1: The Broken Cost Calculator (Full of Bugs)

```python
"""
WHAT NOT TO DO — A cost calculator that misses HALF the costs.
"""

# BUG 1: Only counting GPU instance cost
# Misses: storage, data transfer, engineering, monitoring

def calculate_monthly_cost(gpu_hourly_rate: float, num_gpus: int) -> float:
    """Calculate monthly GPU cost."""
    hours_per_month = 24 * 30
    return gpu_hourly_rate * num_gpus * hours_per_month

# BUG 2: Assuming 100% utilization
cost_per_hour = calculate_monthly_cost(1.50, 4) / (24 * 30)
cost_per_request = cost_per_hour / 3600  # Assuming 3600 req/hour
# Reality: GPU is idle 70% of the time → actual cost per request is 3x higher

# BUG 3: Not accounting for spot interruptions
# Spot instances are 70% cheaper, but you need failover capacity
# If you run 4 spot + 1 on-demand backup, cost isn't 4 × $0.45
# It's 4 × $0.45 + 1 × $1.50

# BUG 4: Not modeling traffic patterns
# Peak traffic: 6 hours at 5,000 req/hour
# Off-peak: 18 hours at 500 req/hour
# Average: 1,625 req/hour
# But you need capacity for 5,000 req/hour at peak
# So you're paying for peak capacity but using it 25% of the time

# BUG 5: No cost per quality-adjusted request
# If quantization costs 8% quality, you need MORE requests
# to achieve the same effective quality
```

### Example 2: Production Cost Optimizer (Fixed)

```python
"""
Production cost optimizer — calculates true TCO and recommends optimal config.
"""

from dataclasses import dataclass
from typing import Dict, List, Optional
from enum import Enum


class GPUType(Enum):
    T4 = "t4"
    L4 = "l4"
    A10G = "a10g"
    A100 = "a100"


GPU_SPECS = {
    GPUType.T4: {"memory_gb": 16, "on_demand": 0.50, "spot": 0.15, "throughput_multiplier": 0.7},
    GPUType.L4: {"memory_gb": 24, "on_demand": 0.75, "spot": 0.23, "throughput_multiplier": 1.0},
    GPUType.A10G: {"memory_gb": 24, "on_demand": 1.50, "spot": 0.45, "throughput_multiplier": 1.0},
    GPUType.A100: {"memory_gb": 80, "on_demand": 4.00, "spot": 1.20, "throughput_multiplier": 2.5},
}


@dataclass
class DeploymentConfig:
    """A deployment configuration to cost-optimize."""
    model_size_b: int = 7
    quantization: str = "awq"
    gpu_type: GPUType = GPUType.A10G
    num_gpus: int = 4
    use_spot: bool = False
    spot_backup_count: int = 0  # On-demand backups for spot
    avg_requests_per_hour: int = 5000
    peak_requests_per_hour: int = 15000  # 3x peak
    avg_prompt_tokens: int = 500
    avg_generation_tokens: int = 200
    engineering_hourly_rate: float = 100
    maintenance_hours_per_month: int = 20
    quality_multiplier: float = 1.0  # 0.92 for 8% quality loss


@dataclass
class CostBreakdown:
    """Detailed cost breakdown for a configuration."""
    gpu_cost: float
    storage_cost: float
    data_transfer_cost: float
    engineering_cost: float
    monitoring_cost: float
    total_monthly: float
    cost_per_1k_requests: float
    cost_per_quality_adjusted_1k: float
    effective_throughput: int
    risk_factors: List[str]


class CostOptimizer:
    """Calculate true TCO for AI inference configurations."""
    
    def estimate_throughput(self, config: DeploymentConfig) -> int:
        """Estimate requests per hour for this configuration."""
        gpu = GPU_SPECS[config.gpu_type]
        
        # Base throughput for 7B model on A10G: ~10,000 req/hour
        base_throughput = 10000
        
        # Scale by model size
        model_factor = (7 / config.model_size_b) ** 1.5  # Larger = slower
        
        # Scale by GPU throughput
        gpu_factor = gpu["throughput_multiplier"]
        
        # Scale by GPUs (diminishing returns)
        gpu_count_factor = config.num_gpus ** 0.8  # Sub-linear scaling
        
        # Quantization factor
        quantization_speedup = {"fp16": 1.0, "int8": 1.8, "awq": 3.0, "gptq": 2.8}
        quant_factor = quantization_speedup.get(config.quantization, 1.0)
        
        # Quality adjustment (more throughput needed if quality is lower)
        quality_factor = 1.0 / config.quality_multiplier
        
        return int(
            base_throughput
            * model_factor
            * gpu_factor
            * gpu_count_factor
            * quant_factor
            * quality_factor
        )
    
    def calculate(self, config: DeploymentConfig) -> CostBreakdown:
        """Calculate complete cost breakdown."""
        gpu = GPU_SPECS[config.gpu_type]
        
        # GPU cost
        gpu_hourly = gpu["spot"] if config.use_spot else gpu["on_demand"]
        gpu_hourly_total = gpu_hourly * config.num_gpus
        
        if config.use_spot and config.spot_backup_count > 0:
            # Add on-demand backup costs
            backup_hourly = gpu["on_demand"] * config.spot_backup_count
            gpu_hourly_total += backup_hourly
        
        gpu_cost = gpu_hourly_total * 24 * 30
        
        # Storage cost
        model_weight_size_gb = config.model_size_b * {
            "fp16": 2, "int8": 1, "awq": 0.5, "gptq": 0.5
        }.get(config.quantization, 2) / 1.024
        
        storage_cost = model_weight_size_gb * 0.10 * 30  # $0.10/GB-month for EFS
        
        # Data transfer cost ($0.09/GB outbound)
        tokens_per_request = config.avg_prompt_tokens + config.avg_generation_tokens
        bytes_per_request = tokens_per_request * 4  # ~4 bytes per token
        gb_per_request = bytes_per_request / (1024 ** 3)
        monthly_requests = config.avg_requests_per_hour * 24 * 30
        data_transfer_gb = gb_per_request * monthly_requests
        data_transfer_cost = data_transfer_gb * 0.09
        
        # Engineering cost
        engineering_cost = config.engineering_hourly_rate * config.maintenance_hours_per_month
        
        # Monitoring cost
        monitoring_cost = config.num_gpus * 50  # $50/GPU for basic monitoring
        
        # Total
        total_monthly = gpu_cost + storage_cost + data_transfer_cost + engineering_cost + monitoring_cost
        
        # Cost per request
        effective_throughput = self.estimate_throughput(config)
        monthly_capacity = effective_throughput * 24 * 30
        actual_requests = min(monthly_requests, monthly_capacity)
        
        cost_per_1k = (total_monthly / actual_requests * 1000) if actual_requests > 0 else float('inf')
        cost_per_quality_1k = cost_per_1k / config.quality_multiplier
        
        # Risk factors
        risks = []
        if config.use_spot:
            risks.append(f"Spot interruption: {config.spot_backup_count} backup GPUs may not cover full traffic")
        if config.quality_multiplier < 0.95:
            risks.append(f"Quality loss: {1 - config.quality_multiplier:.0%} quality degradation may impact user satisfaction")
        if config.num_gpus > 4 and not config.use_spot:
            risks.append("High cost concentration: failure of 1 GPU loses significant capacity")
        
        return CostBreakdown(
            gpu_cost=round(gpu_cost, 2),
            storage_cost=round(storage_cost, 2),
            data_transfer_cost=round(data_transfer_cost, 2),
            engineering_cost=round(engineering_cost, 2),
            monitoring_cost=round(monitoring_cost, 2),
            total_monthly=round(total_monthly, 2),
            cost_per_1k_requests=round(cost_per_1k, 4),
            cost_per_quality_adjusted_1k=round(cost_per_quality_1k, 4),
            effective_throughput=effective_throughput,
            risk_factors=risks,
        )
    
    def find_optimal(self, config: DeploymentConfig) -> List[tuple]:
        """Try multiple configurations and find the most cost-effective."""
        results = []
        
        for gpu in GPUType:
            for quant in ["awq", "int8", "fp16"]:
                for use_spot in [True, False]:
                    # Skip T4 FP16 (too little memory for 7B at FP16)
                    if gpu == GPUType.T4 and quant == "fp16":
                        continue
                    
                    cfg = DeploymentConfig(
                        model_size_b=config.model_size_b,
                        quantization=quant,
                        gpu_type=gpu,
                        num_gpus=config.num_gpus,
                        use_spot=use_spot,
                        spot_backup_count=1 if use_spot else 0,
                        avg_requests_per_hour=config.avg_requests_per_hour,
                        peak_requests_per_hour=config.peak_requests_per_hour,
                        avg_prompt_tokens=config.avg_prompt_tokens,
                        avg_generation_tokens=config.avg_generation_tokens,
                        engineering_hourly_rate=config.engineering_hourly_rate,
                        maintenance_hours_per_month=config.maintenance_hours_per_month,
                        quality_multiplier={
                            "fp16": 1.0, "int8": 0.98, "awq": 0.915
                        }.get(quant, 0.95),
                    )
                    
                    cost = self.calculate(cfg)
                    results.append((cost.cost_per_quality_adjusted_1k, cfg, cost))
        
        results.sort(key=lambda x: x[0])
        return results[:5]  # Top 5 cheapest


if __name__ == "__main__":
    optimizer = CostOptimizer()
    
    base = DeploymentConfig()
    print(f"\n{'='*60}")
    print(f"COST OPTIMIZATION ANALYSIS")
    print(f"{'='*60}")
    print(f"Model: {base.model_size_b}B, Quant: {base.quantization}")
    print(f"Traffic: {base.avg_requests_per_hour}/hr avg, {base.peak_requests_per_hour}/hr peak")
    print()
    
    best_configs = optimizer.find_optimal(base)
    
    print(f"{'Config':<40} {'$/1K req':<12} {'$/mo':<12}")
    print(f"{'-'*64}")
    
    for cost_per_1k, cfg, cost in best_configs[:3]:
        label = f"{cfg.gpu_type.value} {cfg.quantization} {'spot' if cfg.use_spot else 'ondemand'} {cfg.num_gpus}gpu"
        print(f"{label:<40} ${cost_per_1k:<10.4f} ${cost.total_monthly:<10.2f}")
    
    print(f"\n{'='*60}")
    print("RECOMMENDATION")
    best = best_configs[0]
    print(f"Cheapest config: {best[1].gpu_type.value} {best[1].quantization} ({'spot' if best[1].use_spot else 'on-demand'})")
    print(f"Cost: ${best[0]:.4f}/1K quality-adjusted requests")
    print(f"Monthly: ${best[2].total_monthly:.2f}")
    print(f"Risks: {', '.join(best[2].risk_factors) if best[2].risk_factors else 'None'}")
```

---

### 🔍 Critical Questions

1. **Quality multiplier**: The optimizer uses fixed quality multipliers (FP16=1.0, INT8=0.98, AWQ=0.915). But quality impact VARIES by task — a model might lose 2% on summarization but 12% on exact match. **How would you make the quality multiplier TASK-DEPENDENT instead of global?**

2. **Spot instance risk**: The optimizer suggests spot instances as cheaper but lists risk factors. What's missing from the risk analysis? (Hint: Spot interruption is CORRELATED — not random. AWS reclaims instances in AZs with high demand. If your region's spot market dries up during peak hours, ALL instances may be reclaimed simultaneously.)

3. **Engineering cost**: The model uses a flat $100/hr for 20 hrs/month. But engineering cost FIXED — whether you spend 20 hours or 200 hours, you pay the same salary. **Should engineering cost be modeled as FIXED (part of baseline ops) or VARIABLE (only the incremental time spent on this deployment)?** How does this assumption change the cost comparison?

---

## ✅ Good Output Examples

### What an Optimized Cost Report Looks Like

```
MONTHLY COST REPORT — AI Inference
====================================
Current cost: $4,320.00 (4× A10G on-demand)
Optimized cost: $1,157.80 (4× T4 spot + 1× L4 on-demand backup)
Savings: 73.2%

Breakdown:
  GPU (spot):         $432.00  (4 × $0.15/hr × 720h)
  GPU (backup):       $540.00  (1 × $0.75/hr × 720h)
  Storage (models):   $24.80
  Data transfer:      $81.00
  Engineering:        $0.00    (automated ops)
  Monitoring:         $80.00   (Datadog, 4 hosts)
  
Cost per 1K requests: $0.0029 (current: $0.0108)
Cost per quality-adjusted: $0.0032 (INT8 quantization: 98% quality)

Risk assessment:
  - Spot interruption: Protected by 1 on-demand backup (50% capacity)
  - Quality loss: 2% (acceptable for this workload per A/B test)
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Freezing at the First Bill

**Problem**: First GPU bill arrives ($4,320/month). Panic. Switch everything to cheapest config without testing quality impact. Users notice degradation. Churn increases.

**Fix**: A/B test cost optimizations. Roll out to 10% of traffic first. Measure quality metrics AND user satisfaction BEFORE full rollout.

### Antipattern 2: Ignoring Engineering Cost

**Problem**: Switch to spot instances to save $3,000/month. Spend 40 hours/month dealing with spot interruptions, pre-warming, and failover. Engineering cost: $4,000/month. Net loss: $1,000/month.

**Fix**: Include engineering time in TCO calculations. If the savings from a cost optimization are less than the engineering time to implement it, skip it.

### Antipattern 3: Over-Optimizing for Cost

**Problem**: 4-bit quantization + spot instances + aggressive auto-scaling → cost is minimal. But: quality is degraded, service is unreliable, and on-call engineers hate the system. Users leave. Revenue drops. The "cost optimization" destroyed the business.

**Fix**: Define MINIMUM ACCEPTABLE QUALITY and MINIMUM ACCEPTABLE RELIABILITY before optimizing cost. Never optimize below these thresholds.

### Common Cost Failures

| Failure | Symptom | Fix |
|---------|---------|-----|
| Idle GPU waste | GPU utilization < 20% | Auto-scaling, batch processing |
| Over-provisioned | Paying for peak capacity during off-peak | Spot + on-demand mix |
| Hidden data transfer | Unexpected $500+ bill | Cache responses, minimize cross-region traffic |
| Quantization quality loss | User complaints increase | A/B test before full rollout |
| Engineering time sink | Team spends all time on infra | Buy managed services for non-differentiating work |

---

## 🧪 Drills & Challenges

### Drill 1: Calculate Your True Cost (30 min)

**Task**: Pick one deployment scenario (4× A10G, 7B model, 10K req/hr) and calculate the TRUE monthly cost including all hidden costs: storage, data transfer, engineering time, monitoring, incident response.

**Then**: Find the optimal cost-optimized configuration that reduces cost by 50%+ while keeping quality above 95% of baseline and uptime above 99.9%.

---

### Drill 2: Design a Provider Router (45 min)

**Goal**: Build a router that sends each request to the cheapest provider that can handle it.

**Task**: Design the routing logic for 3 providers:
- OpenAI: $10/1M input, $30/1M output — good for everything
- Self-hosted vLLM (7B): $1.50/hr GPU — good for fine-tuned tasks, cheap per request at volume
- Anthropic: $15/1M input, $75/1M output — best for complex reasoning

The router must classify each request as:
1. **Simple** (factual Q&A, extraction) → route to self-hosted
2. **Standard** (summarization, generation) → route to OpenAI
3. **Complex** (reasoning, analysis, code) → route to Anthropic

**Design the classifier** and calculate: at 50% simple, 30% standard, 20% complex, what's the cost savings vs. using only OpenAI?

---

### Drill 3: Build an Auto-Scaling Cost Model (30 min)

**Goal**: Model the cost of auto-scaling vs. always-on GPU instances.

**Scenario**: Traffic varies: 500 req/hr at night (12 AM-8 AM), 2,000 req/hr during business hours (8 AM-6 PM), 5,000 req/hr peak (6 PM-12 AM).

**Task**: Compare costs for:
1. **Always-on**: 4 GPUs, 24/7
2. **Auto-scale**: Scale from 1 GPU (off-peak) to 4 GPUs (peak), 5 min to spin up
3. **Spot + on-demand**: 2 spot (always) + 2 on-demand (auto-scale during peak)

**Calculate**: Monthly cost for each, and the savings from auto-scaling. At what traffic volume does auto-scaling stop being worth it (because the cold start cost outweighs the savings)?

---

## 🚦 Gate Check

Before moving to File 05 (Latency Optimization), verify you can:

1. **Calculate true TCO** — For a deployment with given GPU, model, traffic, and engineering time: calculate monthly cost with ALL hidden costs included

2. **Compare spot vs. on-demand** — Given a traffic pattern with predictable peaks: design a spot/on-demand mix with: minimum cost, guaranteed minimum capacity, survivable spot interruption

3. **Quantization cost/benefit** — Given quality metrics for 3 quantization levels and GPU costs: calculate cost per quality-adjusted request and recommend the best config

4. **Design provider routing** — For a deployment using OpenAI, self-hosted, and Anthropic: design routing rules, expected cost savings, risk analysis

5. **Avoid cost traps** — Identify 3 "cost optimization" strategies that SAVE money in the short term but COST more in the long term (engineering time, reliability, quality)

---

## 📚 Resources

### GPU Pricing
- **[AWS GPU Pricing](https://aws.amazon.com/ec2/pricing/on-demand/)** — Current EC2 GPU pricing
- **[Spot Instance Best Practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html)** — AWS guide to spot usage

### Cost Optimization
- **[ML Cost Optimization at Scale](https://www.anyscale.com/blog/how-to-reduce-your-gpu-costs-by-up-to-70-using-anyscale)** — Strategies for GPU cost reduction
- **[LLM Inference Cost Guide](https://biriukov.dev/docs/cloud-computing/cloud-pricing-ec2-spot-the-cheapest-gpu-instances/)** — Comparison of GPU options

### Phase Cross-References
- **Phase 10, File 03 (Caching)** → Caching is the #1 cost optimization lever. Implement caching BEFORE optimizing GPU costs.
- **Phase 10, File 05 (Latency)** → Cost and latency are TRADE-OFFS. Fast GPUs cost more. Lazy GPUs save money but increase latency.
- **Phase 9 (Fine-Tuning, File 08)** → Deploying your own fine-tuned model changes the cost equation vs. API-based models.

> **Next up: File 05 — Latency Optimization.** Users leave when responses take too long. You'll learn every technique to reduce latency: streaming, continuous batching, speculative decoding, prefix caching, and quantization-aware latency tuning. Each has a cost tradeoff — you'll learn when each one pays off.
