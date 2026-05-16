# Embedding Model Selection

## Model Landscape (2026)

| Model | Dimensions | Max Tokens | Best For | Cost |
|-------|-----------|------------|----------|------|
| text-embedding-3-large | 3072 | 8191 | General, high-quality | $0.13/M tokens |
| text-embedding-3-small | 1536 | 8191 | General, balanced | $0.02/M tokens |
| Cohere Embed v3 | 1024 | 512 | Multilingual, search | $0.10/M tokens |
| Jina Embeddings v3 | 1024 | 8192 | Long documents | $0.05/M tokens |
| all-MiniLM-L6-v2 | 384 | 256 | Local, fast | Free (OSS) |
| BAAI/bge-large-en-v1.5 | 1024 | 512 | Open-source quality | Free (OSS) |
| nomic-embed-text-v1.5 | 768 | 8192 | OSS, long context | Free (OSS) |

## How to Choose

### Step 1: Define Your Requirements
```python
requirements = {
    "dimensions": "Not important" if fast_NN or "Performance critical",
    "quality": "MTEB score > 60",
    "latency": "<100ms per embedding",
    "throughput": "1000 docs/min",
    "cost": "$0.01 per 1000 embeddings",
    "domain": "legal/medical/code/general",
}
```

### Step 2: Benchmark on YOUR Data
```python
def benchmark_embedding_model(model_name: str, queries: list, documents: list):
    embedder = get_embedder(model_name)
    query_embs = embedder.embed(queries)
    doc_embs = embedder.embed(documents)

    results = {"model": model_name, "latency": [], "recall@k": []}
    for query, qemb in zip(queries, query_embs):
        start = time.time()
        scores = cosine_similarity(qemb, doc_embs)
        results["latency"].append(time.time() - start)

        # Recall@10: is the relevant doc in top 10?
        ranking = np.argsort(scores)[-10:]
        results["recall@k"].append(1 if relevant_idx in ranking else 0)

    return results
```

## 🔴 Senior: The Embedding Trap

Never use a model's default embedding model for production. Defaults are chosen for generality, not for your specific task. Always:
1. Define your task (semantic search, clustering, classification)
2. Collect representative data
3. Benchmark 3-5 models
4. Choose the best one (weighting quality vs. cost vs. latency)

## Drill: Embedding Benchmark

Build a benchmark that:
1. Takes a test set of query-document pairs (with relevance labels)
2. Runs 5 different embedding models
3. Measures Recall@k, Precision@k, latency, cost
4. Generates a comparison table
5. Recommends the best model for your use case
