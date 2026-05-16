# Long-Context Strategies

## The Problem

Models now support 128K-2M token context windows. But using all of it is:
1. **Expensive**: 128K input tokens at GPT-4o = $0.32 per query
2. **Slow**: More tokens = longer generation time
3. **Inaccurate**: "Lost in the middle" — information in the middle gets ignored
4. **Degraded quality**: Models perform worse on very long inputs

## Strategies

### 1. Sliding Window Attention
```python
class SlidingWindow:
    """Only attend to the last N tokens."""
    def __init__(self, window_size: int = 4096):
        self.window = window_size
        self.history = deque(maxlen=window_size)

    def add(self, token_ids: list[int]):
        self.history.extend(token_ids)

    def get_context(self) -> list[int]:
        return list(self.history)
```

### 2. Hierarchical Summarization
```python
class HierarchicalSummarizer:
    """Summarize chunks, then summarize summaries."""
    def __init__(self, llm, chunk_size: int = 4000):
        self.llm = llm
        self.chunk_size = chunk_size

    async def summarize(self, long_text: str) -> str:
        # Level 1: summarize each chunk
        chunks = self._chunk(long_text)
        summaries = []
        for chunk in chunks:
            s = await self.llm.generate(f"Summarize:\n{chunk}")
            summaries.append(s)

        # Level 2: summarize the summaries
        if len(summaries) > 1:
            combined = "\n".join(summaries)
            return await self.llm.generate(f"Summarize these summaries:\n{combined}")
        return summaries[0]
```

### 3. RAG over Long Documents
Instead of putting the entire document in context, use RAG within the document:
```python
class DocumentRAG:
    """Use RAG on a single long document."""
    def __init__(self, chunk_size: int = 1000):
        self.chunks = []
        self.chunk_size = chunk_size

    async def load(self, document: str):
        self.chunks = self._chunk(document)
        self.embeddings = await self._embed(self.chunks)

    async def query(self, question: str) -> str:
        relevant = await self._search(question, k=5)
        context = "\n\n".join(relevant)
        return await self.llm.generate(f"Context:\n{context}\nQuestion: {question}")
```

### 4. Context Compression
```python
class ContextCompressor:
    """Compress context to its essential information."""
    async def compress(self, context: str, query: str, ratio: float = 0.3) -> str:
        prompt = f"""Compress the following context to {int(ratio * 100)}% of its original size.
Keep ALL factual information relevant to answering the question.
Remove redundancies, examples, and tangential details.

Question: {query}
Context: {context}
Compressed:"""
        return await self.llm.generate(prompt)
```

## 🔴 Senior: When to Use Each

| Strategy | Use Case | Tradeoff |
|----------|----------|----------|
| RAG over doc | Most cases | Need chunking + retrieval quality |
| Hierarchical summary | Need full understanding | Loses detail at each level |
| Sliding window | Real-time conversation | Loses early context |
| Context compression | Tight budget | May lose critical info |
| Full context | Short docs (<10K) | Expensive but complete |
