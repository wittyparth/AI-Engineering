# Cost Optimization

## Where the Money Goes

For a typical RAG system serving 1M queries/month:

| Component | Cost/Month | % |
|-----------|-----------|----|
| LLM API calls (GPT-4o-mini) | $150-500 | 40% |
| Embedding API | $30-100 | 8% |
| Vector DB (hosted) | $50-200 | 15% |
| Reranking API | $30-100 | 8% |
| Compute (ECS/Render) | $50-150 | 13% |
| Redis (cache) | $20-50 | 5% |
| PostgreSQL + pgvector | $30-100 | 8% |
| Monitoring (Langfuse/etc) | $10-30 | 3% |

**Total**: $370-1,230/month for a modest production system.

## Optimization Strategies

### 1. Model Selection (Biggest Impact)
```python
# Cost per 1M tokens
gpt4o = {"input": 2.50, "output": 10.00}
gpt4o_mini = {"input": 0.15, "output": 0.60}
claude_haiku = {"input": 0.25, "output": 1.25}

# Use cheaper model for simple queries
def select_model(complexity: str) -> str:
    if complexity == "simple":
        return "gpt-4o-mini"    # $0.15/M input
    return "gpt-4o"              # $2.50/M input
```

### 2. Caching
- Exact match: 5-15% hit rate → saves 5-15% of LLM costs
- Semantic cache: 20-40% hit rate → saves 20-40%
- Session cache: repeated queries in same session → saves additional 5-10%

### 3. Prompt Optimization
```python
# Before: 500 token prompt
# After: 200 token prompt
# Savings: 60% on input tokens
```

### 4. Batch Processing
```python
# Instead of 100 individual API calls:
# Batch into 10 calls with 10 items each
# OpenAI supports batching with 50% discount
```

## Cost Tracking Dashboard

```python
class CostTracker:
    def __init__(self):
        self.metrics = {"total_cost": 0, "per_model": defaultdict(float)}

    def track(self, model: str, input_tokens: int, output_tokens: int):
        cost = self._calculate_cost(model, input_tokens, output_tokens)
        self.total_cost += cost
        self.per_model[model] += cost

    def get_report(self) -> dict:
        return {
            "total_cost": self.total_cost,
            "cost_by_model": dict(self.per_model),
            "projected_monthly": self.total_cost * 30,
            "top_queries": self._most_expensive_queries(),
        }
```

## 🔴 Senior: The 80/20 of Cost Optimization

1. **Model selection** (60% impact): Use the cheapest model that passes evals
2. **Caching** (20% impact): Semantic cache for common patterns
3. **Prompt optimization** (10% impact): Shorter prompts, fewer tokens
4. **Batching** (5% impact): Discounted batch API calls
5. **Everything else** (5% impact): Rate limiting, compression, etc.

Start with model selection. Don't optimize caching until you have real traffic patterns.
