# Hybrid Search (BM25 + Vector)

## Why Hybrid?

Pure vector search misses exact keyword matches. Pure BM25 misses semantic matches. Hybrid gets both.

```
Query: "How do I reset my iPhone password?"
Vector search: captures "reset password" semantically → returns manual for password reset
BM25: captures "iPhone password" → returns Apple's password policy page
Hybrid: returns BOTH → user picks the right one
```

## BM25 (Keyword Search)

```python
from rank_bm25 import BM25Okapi

class BM25Search:
    def __init__(self, documents: list[str]):
        tokenized = [doc.lower().split() for doc in documents]
        self.bm25 = BM25Okapi(tokenized)

    def search(self, query: str, k: int = 10):
        tokenized_query = query.lower().split()
        scores = self.bm25.get_scores(tokenized_query)
        top_k = np.argsort(scores)[-k:][::-1]
        return top_k, [scores[i] for i in top_k]
```

## Dense (Vector) Search

```python
class DenseSearch:
    def __init__(self, documents: list[str], embedder):
        self.documents = documents
        self.embeddings = embedder.embed(documents)

    def search(self, query: str, k: int = 10):
        query_emb = self.embedder.embed([query])[0]
        scores = cosine_similarity(query_emb, self.embeddings)
        top_k = np.argsort(scores)[-k:][::-1]
        return top_k, [scores[i] for i in top_k]
```

## Fusion: Reciprocal Rank Fusion (RRF)

```python
def rrf(rankings: list[list[int]], k: int = 60) -> list[int]:
    """Combine multiple rankings using RRF."""
    scores = {}
    for ranking in rankings:
        for rank, doc_id in enumerate(ranking):
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank + 1)
    return sorted(scores, key=scores.get, reverse=True)
```

## When to Use Each

| Scenario | BM25 | Dense | Hybrid |
|----------|------|-------|--------|
| Exact phrase match | Best | Poor | Best |
| Synonym matching | Poor | Best | Best |
| New/rare terms | Best | Poor | Best |
| Long queries | Poor | Best | Best |
| Domain-specific terms | Poor | Mixed | Best |
| Low resource | Fast | Needs GPU | Balanced |

## 🔴 Senior: The Cold Start Problem

When you launch a new system, you have no user queries to learn from. Start with hybrid search (equal weight BM25 + dense). As you collect query data, optimize the weighting. Most production systems end up at 70-80% dense, 20-30% BM25.
