# Vector DB Deep Dive

## Indexing Algorithms

### HNSW (Hierarchical Navigable Small World)
The default for almost all vector databases.

**How it works:**
1. Build a multi-layer graph
2. Top layer: few nodes, long connections
3. Bottom layer: all nodes, short connections
4. Search: start at top, greedily descend to bottom

**Parameters**
```python
from qdrant_client import QdrantClient

client.create_collection(
    collection_name="my_collection",
    vectors_config=VectorParams(
        size=1536,
        distance=Distance.COSINE,
        hnsw_config=HnswConfigDiff(
            m=16,           # Connections per node (higher = more accurate, more memory)
            ef_construct=200,  # Dynamic list size during construction
            full_scan_threshold=10000,  # Threshold for full scan vs HNSW
        )
    )
)
```

### IVF (Inverted File Index)
Older approach. Divides space into Voronoi cells.

**Parameters**
- `nlist`: Number of centroids (more = finer search, slower build)
- `nprobe`: Number of cells to search (more = more accurate, slower)

## Vector Database Comparison

| Feature | Qdrant | Pinecone | Weaviate | Chroma | pgvector |
|---------|--------|----------|----------|--------|----------|
| Self-host | Yes | No | Yes | Yes | Yes |
| Filtering | Strong | Strong | Strong | Weak | Strong |
| Hybrid search | Yes | No | Yes | No | Limited |
| HNSW | Default | Default | Default | Default | Only IVFFlat |
| Disk-based | Yes | No | Yes | Limited | Yes |
| Open source | Yes | No | Yes | Yes | Yes |
| Speed | Fast | Fast | Moderate | Fast | Slow for large |

## 🔴 Senior: You Probably Don't Need a Vector DB

For <100K vectors, an in-memory numpy array + brute force search is:
- Faster (no network call)
- Cheaper (no infrastructure)
- Simpler (no dependencies)
- More accurate (exact search, not approximate)

```python
import numpy as np
from typing import List

class InMemoryVectorDB:
    def __init__(self):
        self.vectors: List[np.ndarray] = []
        self.documents: List[str] = []
        self.metadata: List[dict] = []

    def search(self, query: np.ndarray, k: int = 10) -> List[dict]:
        similarities = [
            np.dot(query, vec) / (np.linalg.norm(query) * np.linalg.norm(vec))
            for vec in self.vectors
        ]
        top_k = np.argsort(similarities)[-k:][::-1]
        return [
            {"document": self.documents[i], "score": similarities[i],
             "metadata": self.metadata[i]}
            for i in top_k
        ]
```

Only reach for a full vector DB when:
- >1M vectors
- Need persistence across restarts
- Need advanced filtering
- Need distributed access
- Need CRUD operations on vectors
