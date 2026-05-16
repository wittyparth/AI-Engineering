# Context Window Management

## The Problem

You have 20 retrieved chunks (10K tokens) + conversation history (5K tokens) + system prompt (1K tokens) + user query. Total: 18K+ tokens. Most models have 128K context, but:

1. **Cost**: 128K tokens = $0.32 per query (GPT-4o)
2. **Latency**: More tokens = slower generation
3. **Lost in the middle**: Information in the middle gets ignored
4. **Recency bias**: Models focus on the last ~10% of context

## Strategies

### 1. Dynamic Context Selection
```python
class DynamicContext:
    def __init__(self, max_tokens: int = 8000):
        self.max_tokens = max_tokens

    def build(self, system_prompt: str, chunks: list[Chunk], history: list, query: str):
        budget = self.max_tokens

        # Fixed costs
        budget -= count_tokens(system_prompt)
        budget -= count_tokens(query)

        # Conversation history (most recent first, truncate oldest)
        history_tokens = count_tokens(history)
        if history_tokens > budget * 0.2:
            history = self._truncate_history(history, int(budget * 0.2))
        budget -= count_tokens(history)

        # Remaining budget for chunks
        selected_chunks = []
        for chunk in chunks:
            chunk_tokens = count_tokens(chunk.content)
            if chunk_tokens <= budget:
                selected_chunks.append(chunk)
                budget -= chunk_tokens
            else:
                break

        return self._build_prompt(system_prompt, history, selected_chunks, query)
```

### 2. Sliding Window
For long conversations, keep a fixed-size window:

```python
class SlidingWindow:
    def __init__(self, window_size: int = 10):
        self.window_size = window_size
        self.history = []

    def add(self, message: dict):
        self.history.append(message)
        if len(self.history) > self.window_size:
            # Summarize the oldest messages
            oldest = self.history[:-self.window_size]
            summary = self._summarize(oldest)
            self.history = [{"role": "system", "content": f"Previous summary: {summary}"}] + self.history[-self.window_size:]
```

### 3. Contextual Compression (LLMLingua)
```python
from llmlingua import PromptCompressor

compressor = PromptCompressor()
compressed_prompt = compressor.compress(
    context,
    instruction="Answer the question based on the context.",
    rate=0.5,  # Compress to 50% of original
)
```

### 4. Structured Context with Priority Levels
```python
class PrioritizedContext:
    PRIORITIES = {
        "critical": 0,    # Always include (system prompt, user query)
        "high": 1,        # Include if possible (top-3 retrieved chunks)
        "medium": 2,      # Include if space (remaining chunks)
        "low": 3,         # Include last (chat history old turns)
    }
```

## 🔴 Senior: The 80/20 Rule of Context Budget

```
80% of budget → Retrieved chunks (these are why you're using RAG)
10% of budget → Conversation history (recent 2-3 turns only)
10% of budget → System prompt + query (keep system prompt tight)
```

Most people do the opposite: 50% system prompt, 40% history, 10% chunks.
