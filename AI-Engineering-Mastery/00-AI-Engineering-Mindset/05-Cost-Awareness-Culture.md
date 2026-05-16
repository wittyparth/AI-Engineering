# Cost-Awareness Culture

## The Reality

A single GPT-4o call costs ~$0.01. At 1M calls/day, that's $10K/day. A prompt change that doubles output tokens doubles your bill. Senior engineers think about this BEFORE shipping.

## Cost Breakdown

| Model | Input ($/1M tokens) | Output ($/1M tokens) |
|-------|-------------------|---------------------|
| GPT-4o | $2.50 | $10.00 |
| GPT-4o-mini | $0.15 | $0.60 |
| Claude Sonnet 4 | $3.00 | $15.00 |
| Claude Haiku 3.5 | $0.80 | $4.00 |
| Llama 3.2 8B (self-hosted) | ~$0.05 | ~$0.05 |
| Llama 3.2 70B (self-hosted) | ~$0.30 | ~$0.30 |

## Cost Optimization Levers

| Lever | Impact | Effort |
|-------|--------|--------|
| Use smaller model for simple tasks | 10-20x | Low |
| Cache repeated queries | 40-80% reduction | Medium |
| Reduce output tokens (be concise) | 2-5x | Low |
| Batch non-real-time requests | 2-3x | Low |
| Fine-tune smaller model | 10-100x | High |
| Self-host with vLLM | 5-50x | High |

## The Model Cascade Pattern (Used by Klarna, Uber, etc.)

```python
async def route_query(query: str) -> str:
    """Route simple queries to cheap model, complex to expensive."""
    # Step 1: Classify query complexity (cheap model)
    complexity = await classify_complexity(query, model="gpt-4o-mini")
    
    if complexity == "simple":
        return await answer(query, model="gpt-4o-mini")  # $0.00015/query
    elif complexity == "moderate":
        return await answer(query, model="gpt-4o")  # $0.0025/query
    else:
        # Complex: use best model + search + analysis
        return await deep_answer(query, model="claude-sonnet-4")  # $0.015/query
```

## Cost Tracking (Non-Negotiable)

```python
class CostTracker:
    def __init__(self):
        self.daily_cost = 0
        self.query_count = 0
    
    def log(self, model: str, input_tokens: int, output_tokens: int):
        cost = self.calculate_cost(model, input_tokens, output_tokens)
        self.daily_cost += cost
        self.query_count += 1
    
    def alert_if_over_budget(self, threshold: float = 100.0):
        if self.daily_cost > threshold:
            slack.send(f"⚠️ Daily cost ${self.daily_cost:.2f} exceeds ${threshold}")
```

## 🔴 Senior: Cost is a Feature, Not an Afterthought

- Every query has a cost budget (just like a latency budget)
- Cost per query should be tracked in the same dashboard as latency and errors
- Alerts should fire when cost per query drifts up (indicates a problem)
- Model cascades are the standard pattern, not an optimization

## Exercise

For a RAG system serving 100K queries/day:
1. Calculate cost using GPT-4o for everything
2. Calculate cost using gpt-4o-mini for 70% of queries, GPT-4o for 30%
3. Calculate cost using a self-hosted Llama model
4. What's the annual savings between strategy 1 and strategy 3?
