# Tokenizers Deep Dive — The Atomic Unit of Everything

## 🎯 Purpose & Goals

Tokens are the **atoms** of AI engineering. Everything — cost, context limits, latency, model capability — is measured in tokens. Most AI engineers don't understand tokenizers deeply, and it costs them money and quality.

**By the end of this file, you will:**
- Understand exactly what a token is and how tokenizers work
- Know why "gpt" costs 1 token but "GPT-4o" costs 4 tokens
- Be able to estimate token counts for any text (critical for cost estimation)
- Know why different models count tokens differently
- Be able to write token-aware code that avoids expensive surprises

**⏱ Time Budget:** 1.5 hours

---

## 📖 What Is a Token? (The Real Answer)

```python
"""
A token is a chunk of text that the model processes as a unit.

• Common words → 1 token:    "the", "is", "and", "hello"
• Common phrases → 1 token:  " because", " really", " GPT"
• Uncommon words → 2+ tokens: "tokenization" → "token" + "ization"
• Code has special rules:    "    " (4 spaces) might be 1 token in Python
• Numbers get split:         "1234567" → "123" + "4567"
• Non-English uses more:     "你好" → 4-5 tokens (vs "hello" = 1 token)

THIS IS CRITICAL:
English text:           ~1 token per 0.75 words
Code:                   ~1 token per 0.5 words (denser)
Non-English text:       ~1 token per 0.3 words (more expensive!)
Special characters:     ~1 token per 1-2 characters (emojis!)
"""
```

> 🤔 **Think about this:** If you're building an AI product for India with Hindi, Tamil, and Bengali queries, how much MORE expensive will it be compared to English-only?

**Answer:** Indian languages typically use 2-5x more tokens per word than English. A Hindi query might cost 3x what the same English query costs. This is NOT negligible — it changes your pricing model.

### Let's See It In Code:

```python
import tiktoken  # OpenAI's tokenizer (open source)

# Different models use different encodings:
cl100k_base = tiktoken.get_encoding("cl100k_base")  # GPT-4, GPT-4o, GPT-3.5
o200k_base = tiktoken.get_encoding("o200k_base")    # GPT-4o (newer)
p50k_base = tiktoken.get_encoding("p50k_base")      # GPT-3 (legacy)

texts = [
    "Hello, how are you today?",
    "The quick brown fox jumps over the lazy dog",
    "I love programming in Python!",
    "tokenization is fascinating",
    "1234567890",
    "    " * 10,  # 40 spaces
    "नमस्ते, आप कैसे हैं?",  # Hindi
    "你好，你今天怎么样？",  # Chinese  
    "హలో, మీరు ఎలా ఉన్నారు?",  # Telugu
    "🔥🔥🔥",  # Emojis
]

for text in texts:
    tokens = cl100k_base.encode(text)
    print(f"Tokens: {len(tokens):3d} | '{text[:50]}'")
    if len(tokens) < 10:
        print(f"        Decoded: {[cl100k_base.decode([t]) for t in tokens]}")
```

**Run this yourself.** The results will surprise you.

---

## 📖 The Cost Implications (Where Most Engineers Get It Wrong)

```python
"""
HERE'S WHERE ENGINEERS MAKE MULTI-THOUSAND-DOLLAR MISTAKES:
"""

# MISTAKE 1: Assuming English token ratios
# "My document has 1000 words, that's ~1333 tokens"
# WRONG if it contains code, tables, or non-English text

# MISTAKE 2: Not counting system prompt every time
# System prompt: 500 tokens → included in EVERY query
# At 10K queries/day: 500 × 10K = 5M tokens/day just for system prompt
# With GPT-4o: 5M × $2.50/1M = $12.50/day = $375/month SYSTEM PROMPT ALONE

# MISTAKE 3: Thinking output tokens cost the same as input
# GPT-4o: Input = $2.50/1M, Output = $10.00/1M (4x more expensive!)
# Long outputs cost disproportionately more

class TokenCostAnalyzer:
    """Real tool for estimating token costs before building"""
    
    def __init__(self, model: str = "gpt-4o"):
        self.model = model
        self.pricing = {
            "gpt-4o": {"input": 2.50, "output": 10.00},
            "gpt-4o-mini": {"input": 0.15, "output": 0.60},
            "claude-3-5-sonnet": {"input": 3.00, "output": 15.00},
            "claude-3-haiku": {"input": 0.25, "output": 1.25},
        }
    
    def estimate_conversation_cost(self, turns: int, system_prompt_tokens: int = 500,
                                   user_msg_avg_tokens: int = 100, 
                                   assistant_avg_tokens: int = 300) -> dict:
        """Estimate cost of a multi-turn conversation"""
        pricing = self.pricing.get(self.model, self.pricing["gpt-4o"])
        
        # Each turn includes: system prompt + user message + assistant response
        # But system prompt is included ONCE, not per-turn
        total_input = system_prompt_tokens + turns * user_msg_avg_tokens
        total_output = turns * assistant_avg_tokens
        
        cost = (total_input / 1_000_000 * pricing["input"]) + \
               (total_output / 1_000_000 * pricing["output"])
        
        return {
            "total_input_tokens": total_input,
            "total_output_tokens": total_output,
            "cost_per_conversation": round(cost, 4),
            "cost_per_1000_conversations": round(cost * 1000, 2),
            "cost_per_100K_conversations": round(cost * 100000, 2),
        }
    
    def compare_models(self, input_tokens: int, output_tokens: int) -> list[dict]:
        """Compare cost across models for the same workload"""
        results = []
        for model, pricing in self.pricing.items():
            cost = (input_tokens / 1_000_000 * pricing["input"]) + \
                   (output_tokens / 1_000_000 * pricing["output"])
            results.append({"model": model, "cost": round(cost, 4)})
        
        return sorted(results, key=lambda x: x["cost"])

# Example: 5-turn conversation, 10K conversations/month
analyzer = TokenCostAnalyzer("gpt-4o-mini")
costs = analyzer.estimate_conversation_cost(turns=5)

print(f"Per conversation: ${costs['cost_per_conversation']:.4f}")
print(f"Per 1000 conversations: ${costs['cost_per_1000_conversations']:.2f}")
print(f"Per 100K conversations: ${costs['cost_per_100K_conversations']:.2f}")

# Compare models
print("\nModel comparison for 2000-in, 500-out tokens:")
for result in analyzer.compare_models(2000, 500):
    print(f"  {result['model']:25} ${result['cost']:.4f}")
```

---

## 📖 Token-Aware Engineering Patterns

```python
import tiktoken
from typing import List

class TokenManager:
    """
    PRODUCTION PATTERN: Token-aware context management.
    Every AI engineer needs this in their toolkit.
    """
    
    def __init__(self, model: str = "gpt-4o"):
        self.encoding = tiktoken.encoding_for_model(model)
        self.max_tokens = self._get_model_limit(model)
    
    def _get_model_limit(self, model: str) -> int:
        limits = {
            "gpt-4o": 128000,
            "gpt-4o-mini": 128000,
            "gpt-4-turbo": 128000,
            "gpt-3.5-turbo": 16385,
        }
        return limits.get(model, 128000)
    
    def count_tokens(self, text: str) -> int:
        """Accurate token count for the specific model"""
        return len(self.encoding.encode(text))
    
    def count_messages(self, messages: list[dict]) -> int:
        """Count tokens in an OpenAI-formatted messages list"""
        total = 0
        for msg in messages:
            total += 4  # Per-message overhead
            for key, value in msg.items():
                total += self.count_tokens(str(value))
                if key == "name":
                    total -= 1  # Name is cheaper
        total += 2  # Assistant reply prefix
        return total
    
    def truncate_to_limit(self, text: str, max_tokens: int) -> str:
        """Truncate text to stay within token budget"""
        tokens = self.encoding.encode(text)
        if len(tokens) <= max_tokens:
            return text
        return self.encoding.decode(tokens[:max_tokens])
    
    def fit_context(self, system_prompt: str, chunks: List[str], 
                   user_query: str, reserve_output: int = 1000) -> List[str]:
        """
        Fit as many context chunks as possible within token budget.
        Uses "lost in the middle" awareness — prioritizes first and last chunks.
        """
        # Calculate available space
        system_tokens = self.count_tokens(system_prompt)
        query_tokens = self.count_tokens(user_query)
        available = self.max_tokens - system_tokens - query_tokens - reserve_output
        
        if available <= 0:
            return []
        
        # Start adding chunks, prioritizing first and last
        chunk_tokens = [self.count_tokens(c) for c in chunks]
        
        # Greedy: add from start and end, skip middle
        selected = []
        start_idx, end_idx = 0, len(chunks) - 1
        tokens_used = 0
        
        # Always try to include first chunk
        if start_idx <= end_idx and chunk_tokens[start_idx] <= available:
            selected.append((start_idx, chunks[start_idx]))
            tokens_used += chunk_tokens[start_idx]
            start_idx += 1
        
        # Try last chunk
        while start_idx <= end_idx and tokens_used + chunk_tokens[end_idx] <= available:
            selected.append((end_idx, chunks[end_idx]))
            tokens_used += chunk_tokens[end_idx]
            end_idx -= 1
        
        # Fill middle if space remains
        while start_idx <= end_idx and tokens_used + chunk_tokens[start_idx] <= available:
            selected.append((start_idx, chunks[start_idx]))
            tokens_used += chunk_tokens[start_idx]
            start_idx += 1
        
        # Sort by original position
        selected.sort(key=lambda x: x[0])
        return [s[1] for s in selected]

# Usage
manager = TokenManager("gpt-4o")
print(f"'{'Hello, world!'}' = {manager.count_tokens('Hello, world!')} tokens")
print(f"Max context: {manager.max_tokens} tokens")
```

> 🤔 **Question:** The code above uses a "greedy" approach to fit chunks into context. When would this fail? What kind of chunk distribution would break this algorithm?

---

## 🧪 Debug Challenge: The Token Overflow Bug

**Scenario:** You're debugging a production issue. Users report that sometimes the AI responds with nonsense halfway through a conversation. You suspect token overflow.

```python
# BUGGY CODE — find the 3 bugs:
from openai import OpenAI
import tiktoken

client = OpenAI()

def chat_with_history(user_message: str, history: list[dict], system_prompt: str = "You are helpful.") -> str:
    """
    Maintain a conversation with history.
    Users report weird behavior after ~50 messages.
    """
    messages = [{"role": "system", "content": system_prompt}]
    messages.extend(history)
    messages.append({"role": "user", "content": user_message})
    
    # Bug 1: ??? 
    # Bug 2: ???
    # Bug 3: ???
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages,
        temperature=0.7
    )
    
    ai_response = response.choices[0].message.content
    history.append({"role": "user", "content": user_message})
    history.append({"role": "assistant", "content": ai_response})
    
    return ai_response
```

Find the bugs BEFORE looking at the answer.

<details>
<summary>🔍 Click to reveal bugs</summary>

**Bug 1:** No token counting. After ~50 messages of 500 tokens each = 25K tokens + system prompt + response. The context window (128K for GPT-4o-mini) seems fine, BUT the model's effective attention drops in the middle. The history is growing unbounded.

**Bug 2:** No truncation strategy. When the history gets long, older messages should be summarized or dropped. The code just keeps appending forever.

**Bug 3:** Temperature = 0.7 for a conversation. This introduces randomness. Over many turns, the model might "drift" — starting with good answers, gradually getting weirder. Lower temperature for factual conversations.

**The real production fix:**
```python
def chat_with_history(user_message: str, history: list[dict], 
                      system_prompt: str = "You are helpful.",
                      max_history_tokens: int = 20000) -> str:
    
    token_manager = TokenManager("gpt-4o-mini")
    
    # Count and trim history to budget
    history_tokens = sum(token_manager.count_tokens(m["content"]) for m in history)
    
    if history_tokens > max_history_tokens:
        # Strategy: keep last N messages that fit in budget
        trimmed_history = []
        tokens_used = 0
        for msg in reversed(history):
            msg_tokens = token_manager.count_tokens(msg["content"])
            if tokens_used + msg_tokens > max_history_tokens:
                break
            trimmed_history.insert(0, msg)
            tokens_used += msg_tokens
        history = trimmed_history
    
    messages = [{"role": "system", "content": system_prompt}]
    messages.extend(history)
    messages.append({"role": "user", "content": user_message})
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages,
        temperature=0.0  # Deterministic for factual conversations
    )
    
    ai_response = response.choices[0].message.content
    history.append({"role": "user", "content": user_message})
    history.append({"role": "assistant", "content": ai_response})
    
    return ai_response
```
</details>

---

## ✅ Real Production Examples

**Scenario: Processing customer support emails**

```python
# What a SENIOR engineer does BEFORE writing any code:

# 1. Sample 100 emails, measure token distribution
sample_emails = [...]  # 100 real emails
token_counts = [manager.count_tokens(email) for email in sample_emails]

import statistics
print(f"Mean: {statistics.mean(token_counts):.0f} tokens")
print(f"Median: {statistics.median(token_counts):.0f} tokens")
print(f"P95: {sorted(token_counts)[int(len(token_counts)*0.95)]:.0f} tokens")
print(f"Max: {max(token_counts)} tokens")

# 2. Estimate costs before writing code
avg_input_tokens = int(statistics.mean(token_counts)) + 500  # + system prompt
avg_output_tokens = 200  # typical response length

daily_queries = 10000
cost_per_query = (avg_input_tokens / 1_000_000 * 0.15) + (avg_output_tokens / 1_000_000 * 0.60)
daily_cost = cost_per_query * daily_queries
monthly_cost = daily_cost * 30

print(f"\nCost Analysis:")
print(f"Per query: ${cost_per_query:.4f}")
print(f"Daily: ${daily_cost:.2f}")
print(f"Monthly: ${monthly_cost:.2f}")

# 3. Decision: is the cost worth it?
# If monthly > $500, consider:
#   - GPT-4o-mini instead of 4o (16x cheaper)
#   - Caching common queries (40-60% cache hit rate for support)
#   - Shorter system prompt
#   - Summarize long emails before processing
```

---

## 📚 Resources

- 🎥 [OpenAI Tokenizer Tool](https://platform.openai.com/tokenizer) *(Play with this — essential for intuition)*
- 📄 [tiktoken Documentation](https://github.com/openai/tiktoken)
- 📄 [Anthropic Token Counting](https://docs.anthropic.com/en/docs/build-with-claude/token-counting)
- 🛠 [Tokenizer Perplexity](https://tiktokenizer.vercel.app/) *(Interactive tokenizer explorer)*

---

## 🚦 Gate Check

- [ ] I can estimate token count for any text within ~20% accuracy
- [ ] I understand why non-English text costs more
- [ ] I can calculate the cost of any AI request before making it
- [ ] I know how to implement token-aware context truncation
- [ ] I can identify and fix token overflow bugs

**Action item:** Go to [platform.openai.com/tokenizer](https://platform.openai.com/tokenizer) and paste 5 different types of text (code, Hindi, emoji, Python, long document). Write down the token counts. Internalize the patterns.

---

**→ Continue to `03-Context-Windows-Attention.md`**
