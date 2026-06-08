# Cost Awareness Culture

## 🎯 Purpose & Goals

In production AI, **cost is a feature**. Every model call, every retrieved document, every token in the context window costs money. Senior AI engineers develop an intuitive sense for cost that informs every architectural decision.

**By the end of this file, you will:**
- Know the cost of every major model and embedding service
- Be able to estimate monthly AI costs BEFORE writing code
- Understand the 6 levers for cost optimization
- Have a cost-tracking system in your projects

**⏱ Time Budget:** 1.5 hours

---

## 📖 The Token Economics You Must Internalize

### The Raw Numbers

```python
# As of 2025 — MASTER THESE NUMBERS:

MODEL_PRICING = {
    # OpenAI
    "gpt-4o": {"input": 2.50, "output": 10.00},    # per 1M tokens
    "gpt-4o-mini": {"input": 0.15, "output": 0.60}, # per 1M tokens
    "o1": {"input": 15.00, "output": 60.00},         # per 1M tokens (!)
    "o1-mini": {"input": 3.00, "output": 12.00},     # per 1M tokens
    
    # Anthropic
    "claude-3-5-sonnet": {"input": 3.00, "output": 15.00},
    "claude-3-haiku": {"input": 0.25, "output": 1.25},
    
    # Google
    "gemini-1.5-pro": {"input": 1.25, "output": 5.00},
    "gemini-1.5-flash": {"input": 0.075, "output": 0.30},  # Cheapest
    "gemini-2.0-flash": {"input": 0.10, "output": 0.40},
    
    # Embeddings
    "text-embedding-3-small": 0.02,    # per 1M tokens
    "text-embedding-3-large": 0.13,    # per 1M tokens
}

def format_cost(cost_per_1m: float) -> str:
    """Make costs intuitive"""
    if cost_per_1m >= 10:
        return f"${cost_per_1m:.2f}/1M tokens"
    elif cost_per_1m >= 1:
        return f"${cost_per_1m:.2f}/1M (${cost_per_1m/1000:.4f}/1K)"
    elif cost_per_1m >= 0.1:
        return f"${cost_per_1m:.3f}/1M (${cost_per_1m/1000:.6f}/1K)"
    else:
        return f"${cost_per_1m:.4f}/1M (${cost_per_1m/1000:.8f}/1K)"

# Print for reference:
for model, pricing in MODEL_PRICING.items():
    if isinstance(pricing, dict):
        print(f"{model:25} | Input:  {format_cost(pricing['input'])}")
        print(f"{'':25} | Output: {format_cost(pricing['output'])}")
    else:
        print(f"{model:25} | {format_cost(pricing)}")
```

### Real Costs for Common Scenarios

```python
class CostScenario:
    """Make costs tangible by calculating real scenarios"""
    
    @staticmethod
    def chatbot_conversation():
        """
        10-turn conversation with context
        - System prompt: 500 tokens
        - Each user msg: 100 tokens
        - Each assistant response: 300 tokens
        - Total: 500 + 10*(100 + 300) = 4500 tokens
        """
        model = "gpt-4o-mini"
        total_tokens = 4500
        input_cost = (total_tokens / 1_000_000) * 0.15
        output_cost = (total_tokens / 1_000_000) * 0.60  # rough
        total = input_cost + output_cost
        
        return {
            "per_conversation": total,
            "daily_1000_convos": total * 1000,
            "monthly_1000_convos": total * 1000 * 30,
        }
    
    @staticmethod
    def rag_query():
        """
        RAG query with 10 retrieved chunks
        - Query embedding: ~100 tokens
        - 10 chunks: ~5K tokens total
        - Generated answer: ~300 tokens
        - Total context: ~5.4K tokens
        """
        model = "gpt-4o-mini"
        input_tokens = 5400
        output_tokens = 300
        total_cost = (input_tokens / 1_000_000) * 0.15 + (output_tokens / 1_000_000) * 0.60
        
        return {
            "per_query": round(total_cost, 5),
            "monthly_100K_queries": round(total_cost * 100000, 2),
            "yearly_100K_queries": round(total_cost * 100000 * 12, 2),
        }
    
    @staticmethod
    def document_embedding():
        """Embedding 100K documents for RAG"""
        model = "text-embedding-3-small"
        avg_doc_tokens = 1000
        cost_per_doc = (avg_doc_tokens / 1_000_000) * 0.02
        
        return {
            "cost_per_doc": cost_per_doc,
            "cost_100K_docs": cost_per_doc * 100000,
            "cost_1M_docs": cost_per_doc * 1000000,
        }

# Run the scenarios
chat = CostScenario.chatbot_conversation()
rag = CostScenario.rag_query()
embed = CostScenario.document_embedding()

print(f"Chatbot: ${chat['per_conversation']:.4f}/convo → ${chat['monthly_1000_convos']:.2f}/month")
print(f"RAG: ${rag['per_query']:.5f}/query → ${rag['monthly_100K_queries']:.2f}/month")
print(f"Embedding: ${embed['cost_per_doc']:.6f}/doc → ${embed['cost_100K_docs']:.2f}/100K docs")
```

---

## 📖 The 6 Cost Optimization Levers

### Lever 1: Model Selection (Easiest, Biggest Impact)

```python
# SAME TASK, 3 MODEL CHOICES:
def summarize_text(text: str, model: str) -> str:
    response = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": f"Summarize: {text}"}],
        max_tokens=200
    )
    return response.choices[0].message.content

# Input: ~2000 tokens, Output: ~200 tokens
# GPT-4o:       (2000 * 2.50 + 200 * 10.00) / 1M = $0.007  per query
# GPT-4o-mini:  (2000 * 0.15 + 200 * 0.60) / 1M  = $0.00042 per query  
# Gemin Flash:  (2000 * 0.075 + 200 * 0.30) / 1M = $0.00021 per query

# At 100K queries/day:
# GPT-4o:       $700/day  → $21,000/month ← CATASTROPHIC
# GPT-4o-mini:  $42/day   → $1,260/month   ← REASONABLE
# Gemini Flash: $21/day   → $630/month     ← CHEAP

# LESSON: Always try the CHEAPEST model first.
# Only upgrade if eval scores fail quality thresholds.
```

### Lever 2: Prompt Compression

```python
# BEFORE: Inefficient prompt (using more tokens than needed)
bad_prompt = f"""
You are a customer support assistant for Acme Corp.
Acme Corp was founded in 2010 by John Smith and Mary Johnson.
The company has 500 employees across offices in San Francisco, New York, and London.
Our return policy allows returns within 30 days of purchase with original receipt.
Our shipping policy offers free shipping on orders over $50.
Our premium membership costs $99/year and includes free shipping and priority support.
[50 more lines of company background...]

Question: {user_query}
"""
# ~2000 tokens in system context

# AFTER: Compressed prompt (only what's needed)
good_prompt = f"""
CONTEXT (return policy, shipping, premium):
- Returns: 30 days, original receipt required
- Shipping: Free over $50, 3-5 business days
- Premium: $99/yr, free shipping + priority support

Question: {user_query}
"""
# ~100 tokens

# SAVINGS: 95% token reduction for the same task
```

### Lever 3: Context Window Management

```python
class ContextManager:
    """
    The context window is your most expensive resource.
    Every token you put in costs money. Every token you DON'T put in saves money.
    """
    
    def __init__(self, max_context_tokens: int = 8000):
        self.max_tokens = max_context_tokens
        self.system_prompt_tokens = 0
        self.history_tokens = 0
        self.rag_chunks_tokens = 0
    
    def build_context(self, 
                     system_prompt: str,
                     conversation_history: list,
                     retrieved_chunks: list[str],
                     user_query: str) -> list[dict]:
        """Intelligently manage context within budget"""
        
        # 1. Always include system prompt (highest priority)
        messages = [{"role": "system", "content": system_prompt}]
        remaining = self.max_tokens - count_tokens(system_prompt)
        
        # 2. Add RAG chunks (with priority ordering)
        chunks_used = []
        for chunk in retrieved_chunks:
            chunk_tokens = count_tokens(chunk)
            if remaining - chunk_tokens > 200:  # Leave room for user query
                chunks_used.append(chunk)
                remaining -= chunk_tokens
            else:
                break
        
        if chunks_used:
            messages.append({
                "role": "system", 
                "content": f"Use this context:\n{'---'.join(chunks_used)}"
            })
        
        # 3. Add conversation history (trim oldest first if needed)
        history_to_include = []
        for msg in reversed(conversation_history):
            msg_tokens = count_tokens(msg["content"])
            if remaining - msg_tokens > 100:
                history_to_include.insert(0, msg)
                remaining -= msg_tokens
            else:
                break
        
        messages.extend(history_to_include)
        
        # 4. Add user query (guaranteed)
        messages.append({"role": "user", "content": user_query})
        
        return messages
        
    def get_stats(self) -> dict:
        """Expose context window utilization for monitoring"""
        total = self.system_prompt_tokens + self.history_tokens + self.rag_chunks_tokens
        return {
            "total_tokens": total,
            "system_prompt_tokens": self.system_prompt_tokens,
            "history_tokens": self.history_tokens,
            "rag_chunks_tokens": self.rag_chunks_tokens,
            "utilization_pct": (total / self.max_tokens) * 100,
            "estimated_cost": (total / 1_000_000) * 2.50,  # GPT-4o price
        }
```

### Lever 4: Caching

```python
import redis
import json
from hashlib import sha256

class AICache:
    """
    Cache LLM responses. HUGE cost savings for common queries.
    In production: 30-50% of queries are repeats (same question, same context).
    """
    
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.cache = redis.Redis.from_url(redis_url)
        self.default_ttl = 3600  # 1 hour
    
    def _make_key(self, model: str, messages: list) -> str:
        """Create deterministic cache key from request"""
        # Only cache for temperature=0.0 (deterministic)
        serialized = json.dumps({"model": model, "messages": messages}, sort_keys=True)
        return f"llm:{sha256(serialized.encode()).hexdigest()}"
    
    def get_or_compute(self, model: str, messages: list, compute_fn: callable) -> str:
        """Return cached response or compute and cache"""
        cache_key = self._make_key(model, messages)
        
        # Try cache first
        cached = self.cache.get(cache_key)
        if cached:
            return json.loads(cached)
        
        # Compute and cache
        response = compute_fn()
        self.cache.setex(cache_key, self.default_ttl, json.dumps(response))
        
        return response

# Impact analysis:
# Without cache: 100K queries/day × $0.001 = $100/day = $3,000/month
# With cache (40% hit rate): 60K queries/day × $0.001 = $60/day = $1,800/month
# SAVINGS: $1,200/month. One Redis instance costs ~$15/month.
```

### Lever 5: Batch Processing

```python
class BatchProcessor:
    """
    Batch API is 50% cheaper than real-time for async processing.
    Use for: background tasks, document processing, data enrichment.
    """
    
    def create_batch(self, prompts: list[str]) -> str:
        """Submit batch job for 50% cost reduction"""
        import json
        
        requests = []
        for i, prompt in enumerate(prompts):
            requests.append({
                "custom_id": f"req-{i}",
                "method": "POST",
                "url": "/v1/chat/completions",
                "body": {
                    "model": "gpt-4o-mini",
                    "messages": [{"role": "user", "content": prompt}],
                    "max_tokens": 500,
                }
            })
        
        # Write JSONL
        with open("batch_input.jsonl", "w") as f:
            for req in requests:
                f.write(json.dumps(req) + "\n")
        
        # Upload and submit
        batch_file = client.files.create(
            file=open("batch_input.jsonl", "rb"),
            purpose="batch"
        )
        
        batch = client.batches.create(
            input_file_id=batch_file.id,
            endpoint="/v1/chat/completions",
            completion_window="24h"
        )
        
        return batch.id
    
    # Cost comparison:
    # Real-time: 100K queries × $0.001 = $100
    # Batch: 100K queries × $0.0005 = $50
    # Latency tradeoff: 2 seconds vs 24 hours
```

### Lever 6: Token Budget Governance

```python
class TokenBudget:
    """
    Set hard limits on token usage per user, per session, per day.
    This prevents bill shock from abusive users or runaway agents.
    """
    
    def __init__(self):
        self.daily_budgets = {}  # user_id → tokens used today
    
    def check_budget(self, user_id: str, estimated_tokens: int) -> bool:
        """Check if this request would exceed the user's daily budget"""
        
        DAILY_TOKEN_LIMIT = 100000  # ~$0.015 at GPT-4o-mini prices
        
        today = datetime.now().date().isoformat()
        user_key = f"{user_id}:{today}"
        
        used = self.daily_budgets.get(user_key, 0)
        
        if used + estimated_tokens > DAILY_TOKEN_LIMIT:
            return False  # "Daily limit reached. Try again tomorrow."
        
        self.daily_budgets[user_key] = used + estimated_tokens
        return True
```

---

## ✅ Good Cost Analysis Example

```python
# BAD cost analysis:
"GPT-4o-mini is cheap so we'll use it for everything."

# GOOD cost analysis:
"""
COST ANALYSIS: Document Summarization Feature
===============================================

ASSUMPTIONS:
- Average document size: ~5K tokens
- Average summary: ~300 tokens  
- Expected usage: 10,000 documents/month
- Steady-state growth: 5%/month

MODEL OPTIONS:
┌─────────────────┬──────────┬──────────┬───────────┬───────────┐
│ Model           │ Per Doc  │ Monthly  │ Monthly   │ Year 1    │
│                 │          │ (10K)    │ (50K)     │ Total     │
├─────────────────┼──────────┼──────────┼───────────┼───────────┤
│ GPT-4o          │ $0.0155 │ $155     │ $775      │ $5,580    │
│ GPT-4o-mini     │ $0.0012 │ $12      │ $60       │ $432      │
│ Gemini Flash    │ $0.0005 │ $5       │ $25       │ $180      │
│ Batch (mini)    │ $0.0006 │ $6       │ $30       │ $216      │
└─────────────────┴──────────┴──────────┴───────────┴───────────┘

RECOMMENDATION: Start with GPT-4o-mini.
- 97% cost reduction vs GPT-4o
- Eval shows 0.92 quality score (above 0.85 threshold)
- If quality issues arise, implement cascade: mini → 4o for hard cases
- Expected monthly cost: $12 → negligible

MONITORING:
- Set alert if monthly cost exceeds $50 (4x expected)
- Run weekly eval to detect quality degradation
- Review every quarter for new model options
"""
```

---

## 🧪 Drill: Cost Analysis

**Time: 30 minutes**

**Scenario:** You're building a customer support chatbot for an e-commerce company with 50K daily conversations. Each conversation averages 4 turns (user → AI → user → AI → user → AI → user → AI). Each turn includes 500 tokens of context + 200 tokens of response.

**Task:** Calculate the monthly cost for 3 scenarios:
1. All GPT-4o
2. All GPT-4o-mini
3. Cascade: 90% mini + 10% escalate-to-4o

Add a recommendation with cost savings.

<details>
<summary>Sample Cost Analysis</summary>

```python
conversations_per_day = 50000
turns_per_conversation = 4
context_per_turn = 500
response_per_turn = 200

total_daily_turns = conversations_per_day * turns_per_conversation

# Scenario 1: All GPT-4o
daily_input_tokens = total_daily_turns * context_per_turn
daily_output_tokens = total_daily_turns * response_per_turn

scenario1_daily_cost = (
    (daily_input_tokens / 1_000_000) * 2.50 +
    (daily_output_tokens / 1_000_000) * 10.00
)
# = (100M input / 1M * 2.50) + (40M output / 1M * 10.00)
# = 250 + 400 = $650/day = $19,500/month

# Scenario 2: All GPT-4o-mini
scenario2_daily_cost = (
    (daily_input_tokens / 1_000_000) * 0.15 +
    (daily_output_tokens / 1_000_000) * 0.60
)
# = 15 + 24 = $39/day = $1,170/month

# Scenario 3: Cascade (90% mini, 10% GPT-4o)
cascade_cost = (
    scenario2_daily_cost * 0.9 +
    scenario1_daily_cost * 0.1
)
# = $35.1 + $65 = $100.1/day = $3,003/month

print("Monthly Costs:")
print(f"1. All GPT-4o:            ${scenario1:,.0f}")
print(f"2. All GPT-4o-mini:       ${scenario2:,.0f}")
print(f"3. Cascade (90/10):       ${scenario3:,.0f}")
print(f"4. Mini + Cache (40% hit): ${scenario2 * 0.6:,.0f}")

# RECOMMENDATION:
# Start with Scenario 4 (mini + cache): $702/month
# Monitor quality with evals
# If quality issues arise, switch to Scenario 3 (cascade): $3,003/month
# Only use Scenario 1 if the business demands GPT-4o quality: $19,500/month
# (and if so, question whether this feature is worth $234K/year)
```
</details>

---

## 🚦 Gate Check

- [ ] Can you recite the cost per 1M tokens of GPT-4o, GPT-4o-mini, Claude Sonnet, and Gemini Flash from memory?
- [ ] Can you estimate the monthly cost of an AI feature given user count and query patterns?
- [ ] Can you name all 6 cost optimization levers?
- [ ] Do you understand why GPT-4o can cost 16x more than GPT-4o-mini for the same task?

**Proceed when all four are YES.**
