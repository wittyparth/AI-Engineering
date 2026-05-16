# Similarity Search & Metrics

## Distance Metrics

### Cosine Similarity (Most Common)
```python
def cosine_similarity(a: list[float], b: list[float]) -> float:
    dot = sum(x * y for x, y in zip(a, b))
    norm_a = sum(x * x for x in a) ** 0.5
    norm_b = sum(y * y for y in b) ** 0.5
    return dot / (norm_a * norm_b)
# Range: [-1, 1] or [0, 1] for normalized vectors
```
Best for: normal-length texts, well-normalized embeddings.

### Euclidean Distance (L2)
```python
def euclidean_distance(a: list[float], b: list[float]) -> float:
    return sum((x - y) ** 2 for x, y in zip(a, b)) ** 0.5
# Range: [0, ∞)
```
Best for: spatial similarity, when magnitude matters.

### Dot Product
```python
def dot_product(a: list[float], b: list[float]) -> float:
    return sum(x * y for x, y in zip(a, b))
# Range: (-∞, ∞)
```
Best for: normalized vectors (equivalent to cosine), faster to compute.

## Which Metric to Use

| Use Case | Metric | Why |
|----------|--------|-----|
| Text search | Cosine | Length-independent semantic matching |
| Image search | Cosine | Visual feature similarity |
| Recommendation | Dot product | User-item interaction strength |
| Anomaly detection | Euclidean | Distance from expected cluster |
| Normalized vectors | Dot product | Cosine without the division |

## 🔴 Senior: Performance Reality

```
Brute force:     O(n × d)  — accurate, slow for large datasets
KD-tree:         O(log n × d) — only works for low dimensions (d < 20)
HNSW:            O(log n)  — best for high dimensions, approximate
IVF:             O(n/centroids) — good balance, less accurate than HNSW
```

For d > 100 (which is all text embeddings), HNSW is the default choice.
