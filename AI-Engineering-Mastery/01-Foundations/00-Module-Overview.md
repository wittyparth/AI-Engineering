# Phase 1: Build an LLM Gateway

**The Problem**: Your app needs to call multiple LLMs (GPT-4o, Claude, Llama) but each has a different API, rate limits, error formats, and cost structure. Hard-coding each provider creates a tangled mess.

**What You'll Build**: A production-grade multi-provider LLM gateway service — exactly what companies like Klarna, Uber, and LinkedIn use internally to manage their AI infrastructure.

**Difficulty**: Easy-Medium | **Time**: ~10 hours

## The Build Path

```
1. PROBLEM: "I need to call multiple LLMs reliably without duplicating code"
2. BUILD RAW: Call OpenAI, Anthropic, Ollama APIs directly — feel the pain
3. ADD FRAMEWORK: LiteLLM — provider abstraction, routing, fallback
4. PRODUCTIONIZE: Rate limiting, retry logic, cost tracking, streaming
5. EVALUATE: Latency budgets (P50/P95/P99), cost per query, error rates
6. DEBUG: 3 bugs in a streaming implementation — find and fix them
```

## Concepts (Built Into the Flow)

| # | Step | What You Learn | Why It Matters |
|---|------|---------------|----------------|
| 1 | Problem | How LLMs work, transformer intuition, context windows | Mental model for everything that follows |
| 2 | Build Raw | Tokenizers, model families, API integration patterns, structured outputs | You understand EVERY layer before abstracting |
| 3 | Build Raw | Streaming (SSE/WebSockets) | Production chat UX requires streaming |
| 4 | Add Framework | LiteLLM (provider abstraction, router, proxy) | 100+ providers, one API, cost tracking built-in |
| 5 | Productionize | Rate limiting vs backpressure, retry strategies, fallbacks | This is what separates demos from production |
| 6 | Evaluate | Latency budgets, cost tracking, error budgets | Measure before you optimize |

## The Project: Multi-Provider LLM Gateway

A FastAPI service that:
- Routes to OpenAI, Anthropic, and local Ollama models
- Implements LiteLLM Router with automatic fallback
- Streams responses via SSE
- Counts and logs tokens per request
- Enforces cost budgets per endpoint
- Exposes a unified OpenAI-compatible API

**This is NOT a tutorial chatbot.** This is infrastructure that AI teams deploy internally.

## 🔴 Senior: The Gateway Pattern

Every company with >3 AI features builds an internal LLM gateway. It's the single most important piece of AI infrastructure because:
- You can swap models without changing app code
- You enforce security and cost policies in ONE place
- You get observability across ALL AI usage automatically

## Gate Check

- [ ] Can explain how attention works to a non-technical PM
- [ ] Multi-provider gateway handles OpenAI outage gracefully (falls back)
- [ ] Streaming works end-to-end with <500ms first-token latency
- [ ] All 3 structured output strategies implemented and compared
- [ ] Cost tracking shows per-query and per-model spend
- [ ] Debug challenge: fixed all 3 streaming bugs
