# Reranking for Precision

## Why Reranking

Vector search returns documents that are semantically SIMILAR to the query. But "similar" ≠ "relevant" or "answers the question."

```
Query: "How do I reset my password?"
Vector search returns:
  [0.92] "Password policies require 8+ characters"
  [0.89] "Account security best practices"
  [0.85] "Reset your password by clicking 'Forgot password'"
  [0.81] "Password manager recommendations"

Reranker returns:
  [0.95] "Reset your password by clicking 'Forgot password'" ← this is what we need
  [0.72] "Password policies require 8+ characters"
  [0.55] "Account security best practices"
  [0.30] "Password manager recommendations"
```

## Cross-Encoder Reranking

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerank(query: str, documents: list[str], top_k: int = 5):
    pairs = [[query, doc] for doc in documents]
    scores = reranker.predict(pairs)

    # Sort by score descending
    scored = list(zip(documents, scores))
    scored.sort(key=lambda x: x[1], reverse=True)
    return scored[:top_k]
```

## Cohere Rerank (API)

```python
import cohere
co = cohere.Client(os.getenv("COHERE_API_KEY"))

results = co.rerank(
    model="rerank-english-v3.0",
    query=query,
    documents=documents,
    top_n=5,
)
```

## Performance Impact

| Metric | Without Reranker | With Reranker | Improvement |
|--------|-----------------|---------------|-------------|
| Recall@5 | 0.72 | 0.89 | +24% |
| Precision@5 | 0.65 | 0.85 | +31% |
| Latency | +0ms | +200ms | Added |
| Cost | $0 | $0.001/query | Added |

🔴 **Senior**: Reranking is the highest ROI improvement for RAG quality. The 200ms latency is almost always worth it. Skip reranking only if latency is absolutely critical.

## Reranking vs. Second Retrieval

| Strategy | Use Case |
|----------|----------|
| Rerank top-20 from vector search | General RAG improvement |
| Generate new query from initial results | When first retrieval missed the mark |
| Iterative retrieval (retrieve, then retrieve more) | Complex, multi-part questions |
