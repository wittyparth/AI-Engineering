# Drill: Embedding Benchmark

**Time**: 45 min  **Difficulty**: Medium

## Task

Compare 5 different embedding models on YOUR data:

```python
import time
import numpy as np
from sentence_transformers import SentenceTransformer
import openai

models = {
    "all-MiniLM-L6-v2": SentenceTransformer("all-MiniLM-L6-v2"),
    "bge-large-en-v1.5": SentenceTransformer("BAAI/bge-large-en-v1.5"),
    "nomic-embed-text": SentenceTransformer("nomic-ai/nomic-embed-text-v1.5"),
    "text-embedding-3-small": "openai",  # special handling
    "cohere-embed-v3": "cohere",        # special handling
}

# Your test data
test_queries = [
    "How do I deploy FastAPI to AWS ECS?",
    "What's the difference between RAG and fine-tuning?",
    "Explain vector database indexing",
    # Add 10-20 more domain-specific queries
]

test_docs = [
    # Add 50-100 documents relevant to your domain
]

def benchmark(models: dict, queries: list, docs: list):
    doc_embeddings = {}
    query_embeddings = {}
    latencies = {}

    for name, model in models.items():
        start = time.time()
        doc_embeddings[name] = model.encode(docs) if not isinstance(model, str) else None
        latencies[name] = {"docs": time.time() - start}

    # Measure search quality
    results = {}
    for name in models:
        correct = 0
        for query, relevant_docs in zip(queries, ground_truth):
            results[name] = {"recall@10": correct / len(queries)}

    return results
```

## Report Template

```yaml
model: all-MiniLM-L6-v2
dimensions: 384
latency_per_doc: 0.002s
latency_per_query: 0.001s
recall@10: 0.78
cost_per_1k_docs: $0.00
memory_usage: 200MB

model: text-embedding-3-small
dimensions: 1536
latency_per_doc: 0.015s
latency_per_query: 0.005s
recall@10: 0.91
cost_per_1k_docs: $0.02
memory_usage: 0MB (API)

# Recommendation: text-embedding-3-small for production,
# all-MiniLM-L6-v2 for development
```
