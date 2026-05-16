# Caching & Semantic Caching

## Why Cache in AI Systems

| Cache Type | Hit Rate | Latency Savings | Cost Savings |
|-----------|----------|----------------|-------------|
| Exact match | 5-15% | 2-10s | 100% |
| Semantic (similar) | 20-40% | 2-10s | 100% |
| Prefix (same start) | 10-20% | 2-10s | 100% |
| Tiered (all three) | 35-55% | 2-10s | 100% on hits |

## Exact Match Cache (Redis)

```python
import hashlib
import redis.asyncio as redis

class ExactCache:
    def __init__(self):
        self.client = redis.Redis(host="localhost", port=6379, decode_responses=True)

    def _key(self, query: str, model: str) -> str:
        return f"rag:{model}:{hashlib.sha256(query.encode()).hexdigest()}"

    async def get(self, query: str, model: str) -> str | None:
        return await self.client.get(self._key(query, model))

    async def set(self, query: str, model: str, response: str, ttl: int = 3600):
        await self.client.setex(self._key(query, model), ttl, response)
```

## Semantic Cache

```python
class SemanticCache:
    def __init__(self, embedder, threshold: float = 0.95):
        self.embedder = embedder
        self.threshold = threshold
        self.cache: list[dict] = []  # In production: vector DB

    async def get(self, query: str) -> str | None:
        query_emb = await self.embedder.embed(query)
        best_score = 0
        best_entry = None

        for entry in self.cache:
            score = cosine_similarity(query_emb, entry["embedding"])
            if score > best_score:
                best_score = score
                best_entry = entry

        if best_score >= self.threshold:
            return best_entry["response"]
        return None

    async def set(self, query: str, response: str):
        embedding = await self.embedder.embed(query)
        self.cache.append({
            "query": query,
            "embedding": embedding,
            "response": response,
        })
```

## Cache-Aside Pattern

```python
class CachedRAG:
    def __init__(self, rag: RAGPipeline, exact_cache, semantic_cache):
        self.rag = rag
        self.exact = exact_cache
        self.semantic = semantic_cache

    async def answer(self, query: str) -> str:
        # 1. Try exact match cache
        cached = await self.exact.get(query)
        if cached:
            return cached

        # 2. Try semantic cache
        cached = await self.semantic.get(query)
        if cached:
            return cached

        # 3. Generate fresh
        response = await self.rag.answer(query)

        # 4. Cache the result
        await self.exact.set(query, response)
        await self.semantic.set(query, response)

        return response
```

## 🔴 Senior: Cache Invalidation Strategy

| Strategy | Works For | Problem |
|----------|-----------|---------|
| TTL | Most cases | Stale data within TTL window |
| Event-based | Known updates | Complex event handling |
| Version-based | Versioned knowledge | Requires version tracking |
| Write-through | Real-time systems | Latency on writes |
| Never expire | Static knowledge | Content can't change |

In production: use TTL for most caches (1 hour default), event-based for known updates.

## Project: Semantic Caching Layer

Extend your Phase 4 RAG project with:
1. Exact match cache (Redis)
2. Semantic cache (vector similarity threshold)
3. Cache hit/miss analytics
4. Configurable TTL per content type
5. Cache warming (pre-cache common queries)
