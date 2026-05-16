# Indexing Strategies

## The Recall-Speed-Memory Tradeoff

```
              High Recall
                 ↑
                 │
        Brute Force (100%)
                 │
           HNSW (95-99%)
                 │
            IVF (90-95%)
                 │
                 │
    Low Memory ←──────────→ High Memory
```

## HNSW Parameters

| Parameter | Low Value | High Value | Effect |
|-----------|-----------|------------|--------|
| M (connections) | 4-8 | 16-64 | Higher = better recall, more memory |
| ef_construct | 100 | 500 | Higher = better index quality, slower build |
| ef_search | 50 | 500 | Higher = better recall per query, slower |

```python
# Fast, less accurate
HnswConfigDiff(m=8, ef_construct=100)
# Query time: set ef_search = 100

# Accurate, slower, more memory
HnswConfigDiff(m=32, ef_construct=400)
# Query time: set ef_search = 300
```

## IVF Parameters

| Parameter | Low Value | High Value | Effect |
|-----------|-----------|------------|--------|
| nlist | 100 | 10000 | More = finer cells, slower build |
| nprobe | 1 | nlist | More = more cells searched, more accurate |

## Quantization

Reduce memory by reducing precision:

| Type | Precision | Memory Saved | Recall Loss |
|------|-----------|-------------|-------------|
| Float32 | 32-bit | 0% | 0% |
| Float16 | 16-bit | 50% | <0.5% |
| Int8 | 8-bit | 75% | 1-3% |
| Binary | 1-bit | 96% | 5-15% |

```python
# Qdrant scalar quantization
QuantizationConfig(
    scalar=ScalarQuantization(
        type=ScalarType.INT8,
        always_ram=True  # Keep quantized data in RAM
    )
)
```

## 🔴 Senior: Production Index Strategy

For a typical production setup (1M+ vectors):

1. **Index type**: HNSW (best all-around)
2. **M**: 16 (good balance)
3. **ef_construct**: 200
4. **ef_search**: Start at 100, tune based on recall requirements
5. **Quantization**: Int8 (3-5x memory reduction, minimal recall loss)
6. **Payload indexing**: Index all filter fields

Monitor: recall@10, query latency (p50/p95/p99), memory usage.

## Drill: Index Performance Benchmark

Build a benchmark that:
1. Creates vector collections of different sizes (10K, 100K, 1M)
2. Tests different index types (brute force, HNSW, IVF) with varying parameters
3. Measures: recall@10, latency p50/p95/p99, memory, build time
4. Recommends optimal parameters for your hardware
