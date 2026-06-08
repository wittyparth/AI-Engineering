# 00 — Module Overview: Deployment & Scale

## 🎯 Phase Purpose

> **🛑 STOP. Three scenarios. Think about each before reading on.**

### 🤔 Scenario 1: The 2 AM Pager

You deploy your AI service. It works for 2 weeks. Then at 2:47 AM on a Saturday, your phone buzzes:

**ALERT: p99 latency > 10s (threshold: 3s)**
**ALERT: Error rate 23% (threshold: 1%)**
**ALERT: GPU memory 97% (threshold: 90%)**

You open your laptop, SSH into the server, and see:

- The model is responding — but VERY slowly
- GPU memory is nearly full — KV cache consumed everything
- Requests are queueing up — the server is processing them one at a time
- The error rate is from TIMEOUTS — requests waiting too long in the queue

**Your first instinct:** Restart the server. It fixes the problem temporarily. But 12 hours later, it happens again.

**🤔 Before reading on, answer these:**

1. A restart fixes it temporarily but it recurs after 12 hours. What's ACCUMULATING over time? Is it the KV cache for long-lived connections? Is it a memory leak in your application code? Is it the model itself accumulating internal state? How do you DISTINGUISH between these causes without guessing?

2. Your monitoring showed GPU memory at 60% after deployment, growing to 97% over 12 hours. If you knew this pattern from DAY 1, you could have prevented the 2 AM page. What monitoring would you set up on DAY 1 — not after the incident — that would have shown you this memory growth trend? What metric would you track? What threshold would you alert on?

3. **The hardest question:** The root cause is that vLLM's KV cache manager isn't evicting old sequences because clients aren't closing connections properly. But fixing the SERVER (increasing eviction) is a band-aid. The real fix is on the CLIENT side — closing connections after each conversation. **How do you design your system so that a client-side bug (not closing connections) doesn't bring down the entire serving infrastructure?** What's the server-side defense against misbehaving clients?

---

### 🤔 Scenario 2: The $10,000 Mistake

Your AI service runs on 4x A10G GPUs (24GB each) in AWS. Cost: $6/hour. At 1,000 requests/hour, that's $0.006/request in GPU cost — acceptable.

Your CTO says: "We're launching in 3 new regions. Each region needs its own GPU cluster for low latency. Our GPU bill will be $24/hour ($17,280/month). That's too much."

**The CTO wants you to REDUCE GPU costs by 50%.**

**Options you consider:**
- **A)** Switch to T4 GPUs ($2/hour instead of $6) — 1/3 the cost, less memory
- **B)** Quantize models to 4-bit — use 1/4 the memory, might lose quality
- **C)** Use spot instances — 60-80% cheaper, but can be terminated any time
- **D)** Multi-region with a single GPU region + caching — only 1 GPU region, edge cache

**🤔 Before reading on, answer these:**

1. You run the numbers. Option C (spot instances) saves the most money — 70% reduction. But spot instances can be terminated with 2 minutes notice. When your GPU instances are terminated mid-request, users see errors. **What architecture handles spot instance terminations gracefully?** How do you drain connections, requeue requests, and fail over without dropping user requests?

2. Option D (single GPU region + caching) seems clever. Cache responses in each region so repeated queries don't need GPU. But if your cache hit rate is only 30% (most queries are unique), the cost savings are only 30% — not 50%. **What caching strategy would give you >50% cache hit rate?** Is it possible for LLM responses? (Hint: Semantic caching — cache based on meaning, not exact text. Two different phrasings that mean the same thing can reuse the same cached response.)

3. **The hardest question:** You implement ALL 4 options together — T4 GPUs, 4-bit quantization, spot instances, and semantic caching. The deployment complexity is enormous. If something breaks at 2 AM, you won't be able to debug it because the system is too complex. **At what point does cost optimization become COUNTERPRODUCTIVE?** How do you measure the "operational cost" of complexity against the "infrastructure cost" of GPUs? What's the minimum viable cost optimization that gives you 40% savings without 4x the complexity?

---

### 🤔 Scenario 3: The One-Second Wall

Your AI service serves a chatbot that users interact with in REAL TIME. Your research shows:
- Response time < 500ms: Users feel it's instant
- Response time 500ms-2s: Users notice but accept it
- Response time > 2s: Users start to leave
- Response time > 5s: 50% of users abandon mid-response

Your current setup:
- Model: Llama-3-8B (AWQ 4-bit)
- GPU: A10G
- Average response: 1,200ms

You need to get under 500ms.

**🤔 Before reading on, answer these:**

1. You profile the pipeline and find: 200ms networking, 100ms pre-tokenization/routing, 800ms model inference (300ms TTFT + 500ms generation), 100ms post-processing. The model inference is the bottleneck at 800ms. **Where do you optimize FIRST?** The model inference (800ms → optimize by quantization/speculative decoding)? The networking (200ms → deploy closer to users)? All of the above? How do you prioritize?

2. You're told: "Use speculative decoding to get 2x speedup." You implement it. But now your model requires TWICE the GPU memory (draft model + target model). Your A10G (24GB) can't fit both. You need an A100 (80GB, $4/hr — 2.7x more expensive). **If speculative decoding saves 400ms but costs 2.7x more, is it worth it?** At what request volume does the latency improvement justify the cost increase?

3. **The hardest question:** You CANNOT get under 500ms on a single GPU. The physics of the transformer architecture limits how fast tokens can be generated. Your competitors claim 300ms response times — you suspect they're using CACHING, not inference speed. They pre-generate responses for common queries and serve from cache. **How would you design a HYBRID system — part cached, part generated — that feels "instant" for 80% of queries while keeping real generation for the remaining 20%?** What's the detection mechanism — how do you know if a query can be served from cache before running inference?

---

**By the end of this phase, you will:**
- Deploy a complete production AI infrastructure with Docker Compose, ECS, and auto-scaling
- Serve fine-tuned models with vLLM at scale (multi-GPU, autoscaling, prefix caching)
- Implement Redis caching (response + semantic) to reduce GPU costs by 40-70%
- Optimize costs across GPU choice, quantization, spot instances, and provider arbitrage
- Optimize latency through streaming, batching, speculative decoding, and quantization
- Build a production monitoring stack (Prometheus + Grafana) with intelligent alerting
- Design a multi-provider failover strategy (Bedrock + self-hosted + managed API)
- Implement security and compliance (auth, encryption, audit logging)
- **Ship a complete Production Infrastructure project for your portfolio**

## ⏱ Phase Budget

| File | Topic | Estimated Time |
|------|-------|---------------|
| 00 | Module Overview | 30 min |
| 01 | Production Infrastructure | 4 hr |
| 02 | Model Serving at Scale | 5 hr |
| 03 | Caching with Redis | 3 hr |
| 04 | Cost Optimization | 3 hr |
| 05 | Latency Optimization | 4 hr |
| 06 | Monitoring & Alerting | 4 hr |
| 07 | Multi-Provider & Failover | 3 hr |
| 08 | Security & Compliance | 4 hr |
| 09 | Project: Production Infrastructure | 8 hr |
| **Total** | | **~38 hr** |

## 🔗 Connecting to Previous Phases

```
Phase 1 (LLM Gateway)      → You built a gateway. Now deploy it with real infra.
Phase 4 (RAG)              → Your RAG pipeline needs production infrastructure to serve users.
Phase 7 (Evals)            → Your eval platform needs monitoring integration.
Phase 8 (Guardrails)       → Guardrails run in front of deployed models — they need infra too.
Phase 9 (Fine-Tuning)      → Your fine-tuned models need vLLM serving and caching.

This phase is WHERE EVERYTHING RUNS — the infrastructure layer that all previous phases depend on.
```

## 📖 Phase Map

```
PHASE 10: DEPLOYMENT & SCALE

 00 ─ Module Overview ──────── 3 production scenarios that show why infra matters
  │
 01 ─ Production Infra ──────── Docker Compose, VPC, ECS, networking basics
  │
 02 ─ Model Serving ──────────── vLLM deep dive, multi-GPU, autoscaling, prefix caching
  │
 03 ─ Caching ───────────────── Redis, response cache, semantic cache, rate limiting
  │
 04 ─ Cost Optimization ─────── GPU pricing, spot instances, distillation, arbitrage
  │
 05 ─ Latency Optimization ──── Streaming, continuous batching, speculative decoding
  │
 06 ─ Monitoring ────────────── Prometheus, Grafana, alerts, dashboards
  │
 07 ─ Multi-Provider ────────── AWS Bedrock, multi-region, failover, DR
  │
 08 ─ Security ──────────────── Auth, encryption, audit, compliance
  │
 09 ─ PROJECT ───────────────── Deploy everything — production infrastructure
```

> **Start with File 01: Production Infrastructure.** Everything runs on infrastructure. You'll build from Docker Compose on a single machine to a production-grade deployment on AWS ECS with auto-scaling, load balancing, and zero-downtime deployments.
