# Drill: Build Vector Search From Scratch

**Time**: 30 min  **Difficulty**: Medium

## Task

Build a vector search engine using ONLY Python standard library + numpy:

```python
import numpy as np
from typing import List, Dict

class VectorSearch:
    def __init__(self, dim: int = 384):
        self.dim = dim
        self.vectors: List[np.ndarray] = []
        self.documents: List[str] = []
        self.metadata: List[Dict] = []

    def add(self, vector: np.ndarray, document: str, metadata: Dict = None):
        # Normalize vector (important for cosine similarity)
        normalized = vector / (np.linalg.norm(vector) + 1e-10)
        self.vectors.append(normalized)
        self.documents.append(document)
        self.metadata.append(metadata or {})

    def search(self, query_vector: np.ndarray, k: int = 10) -> List[Dict]:
        # Normalize query
        qnorm = query_vector / (np.linalg.norm(query_vector) + 1e-10)

        # Compute similarities (vectorized)
        similarities = np.dot(self.vectors, qnorm)

        # Get top-k indices
        top_k = np.argsort(similarities)[-k:][::-1]

        return [
            {
                "document": self.documents[i],
                "score": float(similarities[i]),
                "metadata": self.metadata[i],
            }
            for i in top_k
        ]
```

## Test It

1. Create 20 "documents" (short phrases)
2. Use sentence-transformers or any embedding model
3. Add all documents to your VectorSearch
4. Search with 5 different queries
5. Manually judge relevance of top-3 results
6. Where did it fail? Why?

## Challenge Extension
Add BM25 hybrid search (use `rank_bm25` package) and implement RRF fusion.
