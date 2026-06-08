# 03 — Caching with Redis: Making LLMs 10x Cheaper

## 🎯 Purpose & Goals

> 🛑 STOP. Before you design a cache for your AI service, you need to understand the fundamental difference between traditional caching and LLM caching:

**LLM responses CAN'T be cached like HTTP responses.**

HTTP caching: Same URL → Same response. Easy.
LLM caching: Same question → Different response (because temperature, because prompt variations, because the model changed).

The naive approach — "cache exact input → output pairs" — has a hit rate of <5% because users rarely ask the EXACT same question twice.

**The goal of this module:** Build a multi-layer caching strategy that achieves >50% cache hit rate through semantic similarity, while handling the unique challenges of LLM responses.

---

### 🤔 Discovery Question 1: The Freshness Problem

You implement an LLM response cache. A user asks: "What's the return policy?" The answer is: "30-day return policy."

You cache this. 1 hour later, another user asks the same question. Perfect hit! Response from cache.

**But:** The company changed the return policy to 15 days 30 minutes ago. The cache serves the OLD policy.

**🤔 Before reading on, answer these:**

1. Cached data is always STALE to some degree. The question is: how stale is acceptable? For a return policy: 1 hour old might be unacceptable. For "how many employees does the company have?": 1 week old might be fine. **How do you set TTL (time-to-live) per QUERY TYPE, not per cached response?** What signals tell you "this query needs fresh data" vs. "cached data is fine"?

2. You add cache invalidation — when the return policy page changes, invalidate all related cached responses. But how do you know WHICH cached responses are related to the return policy? You'd need to map "return policy document change" → "all queries about return policy." **How would you build this mapping?** (Hint: If your RAG pipeline returns source document IDs, you can cache the source IDs alongside responses. When a document changes, invalidate all cached responses that cited it.)

3. **The hardest question:** You have TWO types of content:
   - Static: "What's your address?" → Answer doesn't change. Can cache for days.
   - Dynamic: "What's the stock price?" → Answer changes every second. Best to NEVER cache.
   
   But most queries are in between: "What are the latest AI trends?" — the answer changes slowly, and a 1-day-old answer is still useful, just less complete. **How do you build a CACHE HIERARCHY with different TTLs and update strategies for different content types?** How does the system automatically classify a query as static/dynamic/semi-dynamic?

---

### 🤔 Discovery Question 2: The Semantic Cache Trap

You implement semantic caching: instead of exact matches, you use embedding similarity to find cached responses.

```
User query: "What's the weather in Paris tomorrow?"
Embedding: [0.23, -0.45, 0.67, ...]  →  Nearest cached query:
  "What will the weather be like in Paris tomorrow?"  (cosine sim: 0.97) → Cache hit!
  "What's the weather in London today?"  (cosine sim: 0.82) → Cache miss
```

**But:**
```
User query: "What's the WEATHER in Paris TOMORROW?"
Nearest cached: "What's the weather in Paris TODAY?" (cosine sim: 0.93 → high!)
Cache serves: Response about TODAY's weather → WRONG!
```

**🤔 Before reading on, answer these:**

1. The queries differ by ONE word ("tomorrow" vs. "today"), but the answers are COMPLETELY DIFFERENT. The embedding similarity is 0.93 — very high — because the surrounding context is identical. **What embedding model would capture the difference between "today" and "tomorrow"?** Or is embedding similarity fundamentally the wrong approach for time-sensitive queries?

2. You try a different approach: exact match on a NORMALIZED query. Normalize: lowercase, remove punctuation, sort words alphabetically. "What's the weather in Paris tomorrow?" → "paris the tomorrow weather what" (sorted). This gives exact match on canonical form. But: "weather in Paris tomorrow" and "what's the weather forecast for Paris tomorrow" become DIFFERENT canonical forms — a miss. **Which tradeoff is better: high precision with semantic embedding (risks false positives like today/tomorrow) or lower recall with canonical exact match (misses similar queries)?** Is there a HYBRID approach?

3. **The hardest question:** Two queries are SEMANTICALLY IDENTICAL but have DIFFERENT answers:
   - Query A: "What's the return policy?" → "30-day return policy" (current policy)
   - Query B: "What WAS the return policy?" → "Was 30 days, changed to 15 days on June 1st"
   
   A semantic cache would match query B to query A's cached response and serve the WRONG answer. **How do you handle QUERIES ABOUT THE PAST that need different answers from queries about the present?** How would you build a time-aware cache that distinguishes "current state" queries from "historical comparison" queries?

---

### 🤔 Discovery Question 3: The Cache Stampede

Your AI service is featured on Hacker News. Traffic spikes from 10 req/min to 10,000 req/min in 5 minutes.

Your cache (Redis) has the top 50 most common questions cached. The first 10,000 requests are:
- 8,000 requests for the top 10 cached questions → cache HITS → fast responses
- 2,000 requests for NEW questions → cache MISSES → hit the GPU → slow responses

**The GPU cluster (4x A10G) handles 200 concurrent requests.** The 2,000 new queries overwhelm it. Requests queue up. Latency spikes from 500ms to 15 seconds. Some requests time out.

The timed-out requests RETRY. Now you have 2,000 original + 500 retries = 2,500 pending. Even worse.

**🤔 Before reading on, answer these:**

1. The cache was WORKING perfectly for common questions. The problem is the UNIQUE questions that required GPU inference. If your cache hit rate is 80%, but 80% of 10,000 is 8,000 — those 8,000 are handled instantly. The remaining 2,000 (20%) need GPU, but your GPU can only handle 200 concurrently. **What's the bottleneck — the cache or the GPU?** How would you fix this? (Increase GPU count? Add a request queue with backpressure? Rate-limit unique queries?)

2. The cache hit rate DROPS under traffic spikes. Why? Because spike traffic is often from NEW users with NEW questions. A user who regularly asks "What's the return policy?" already has that cached. A new user asks "Do you ship to Antarctica?" — never seen before → cache miss. **Does cache hit rate INCREASE or DECREASE under traffic spikes?** How does the answer change for an established service (weeks of cached data) vs. a new service (hours of cached data)?

3. **The hardest question:** The Hacker News spike causes a CACHE STAMPEDE: 2,000 simultaneous cache misses for unique queries. Each miss triggers a GPU inference. The GPU is overwhelmed. Responses slow down. Users retry, adding more load. The system enters a DEATH SPIRAL. **Design a defense system that prevents cache stampedes.** What mechanisms would you add? (Consider: request deduplication — 5 users ask similar questions, only 1 GPU call; rate limiting for uncached queries; shedding non-critical traffic; probabilistic early expiration.)

---

## ⏱ Time Budget

| Section | Time |
|---------|------|
| Discovery Questions | 30 min |
| Concept Deep-Dive | 30 min |
| Code Examples | 1 hr |
| Drills | 45 min |
| Gate Check | 15 min |
| **Total** | **~3 hr** |

---

## 📖 Concept Deep-Dive

### 1. Three-Layer Cache Architecture

```
                    ┌──────────────────────────┐
                    │     L1: Exact Match      │
                    │  (Redis String, TTL=5min) │
                    │  Fastest, lowest storage  │
                    │  Hit rate: ~5-10%         │
                    └────────────┬─────────────┘
                                 │ Miss
                    ┌────────────▼─────────────┐
                    │   L2: Semantic Cache      │
                    │  (Redis + Embeddings)      │
                    │  Vector similarity search  │
                    │  Hit rate: ~30-50%         │
                    └────────────┬─────────────┘
                                 │ Miss
                    ┌────────────▼─────────────┐
                    │  L3: Model Inference      │
                    │  (vLLM / OpenAI API)      │
                    │  Expensive, slow           │
                    │  Populates L1 + L2         │
                    └───────────────────────────┘
```

### 2. Cache Key Design

```python
# Bad cache key (hit rate <5%)
key = f"{prompt}:{model}:{temperature}:{max_tokens}"
# Too specific — different temperatures produce different keys

# Better cache key
key = hash(normalize(prompt)) + ":" + model
# Normalize: lowercase, strip whitespace, canonicalize entities

# Best cache key — multi-level
exact_key = f"exact:{normalized_prompt}:{model}"  # L1
semantic_key = f"semantic:{embedding_quantized}:{model}"  # L2
```

### 3. Semantic Cache with Redis

```python
import numpy as np
from redis import Redis

class SemanticCache:
    """
    Semantic cache using Redis + embeddings.
    
    Approach:
    1. On cache miss, compute embedding of query
    2. Search Redis for nearest cached query by cosine similarity
    3. If similarity > threshold, return cached response
    4. Otherwise, call LLM and cache the new response
    """
    
    SIMILARITY_THRESHOLD = 0.92  # Start here, tune based on false positives
    
    def __init__(self, redis_client: Redis):
        self.redis = redis_client
    
    def get_embedding(self, text: str) -> np.ndarray:
        """Get embedding — use text-embedding-3-small or similar."""
        # In production: call embedding API
        pass
    
    def quantize_embedding(self, embedding: np.ndarray, bits: int = 8) -> str:
        """Quantize embedding to save memory and enable fast search."""
        # Normalize and quantize to int8
        norm = embedding / np.linalg.norm(embedding)
        quantized = (norm * 127).astype(np.int8)
        return quantized.tobytes().hex()
    
    def find_similar(self, query_embedding: np.ndarray) -> str:
        """Find most similar cached response."""
        # In production: use Redisearch for vector similarity
        # Simplified: iterate cached embeddings and compute similarity
        pass
```

---

## 💻 CODE EXAMPLES

### Example 1: The Broken Cache (Full of Bugs)

```python
"""
WHAT NOT TO DO — A naive LLM response cache with 10 bugs.
"""

import redis
import json
from hashlib import md5

cache = redis.Redis(host='localhost', port=6379, db=0)
# BUG 1: No password
# BUG 2: No connection pool
# BUG 3: No fallback if Redis is down

def get_cached_response(prompt: str) -> str:
    """Get cached response for a prompt."""
    # BUG 4: Raw prompt as cache key (no normalization)
    # "What's the weather in Paris?" and "what's the weather in Paris?"
    # produce different keys due to capitalization
    key = f"response:{prompt}"
    
    # BUG 5: No error handling
    # If Redis is down, this throws an exception instead of falling through
    result = cache.get(key)
    if result:
        return json.loads(result)
    return None

def cache_response(prompt: str, response: str):
    """Cache a response."""
    key = f"response:{prompt}"
    
    # BUG 6: Fixed TTL for all responses
    # "What's your address?" → 1 hour TTL (fine)
    # "What's the stock price?" → 1 hour TTL (too long, price changed)
    # "What's the return policy?" → 1 hour TTL (might change any time)
    cache.setex(key, 3600, json.dumps(response))  # 1 hour TTL for everything
    
    # BUG 7: No cache invalidation mechanism
    # Once cached, stays cached until TTL expires
    # No way to proactively invalidate

def get_llm_response(prompt: str) -> str:
    """Get response from LLM."""
    # Check cache first
    cached = get_cached_response(prompt)
    if cached:
        return cached
    
    # BUG 8: No request deduplication
    # If 10 users ask the same question simultaneously,
    # ALL 10 miss the cache (it hasn't been populated yet)
    # and ALL 10 call the LLM — 9 wasted calls
    
    # Call LLM
    response = call_llm(prompt)  # hypothetical
    
    # BUG 9: Caching the response BEFORE verifying it's good
    # If the LLM returns an error message, that gets cached too
    cache_response(prompt, response)
    
    return response

# BUG 10: No cache monitoring
# No metrics on hit rate, miss rate, cache size
# No way to know if the cache is actually helping
```

#### 🔍 Find The Bugs

| # | Bug | Impact | Fix |
|---|-----|--------|-----|
| 1 | No Redis password | Anyone can access/modify cache | Add password auth |
| 2 | No connection pool | New connection per request | Use connection pool |
| 3 | No fallback | App breaks when Redis is down | Try/except, log warning, proceed without cache |
| 4 | Raw prompt as key | Low hit rate from minor text variations | Normalize prompt before key generation |
| 5 | No error handling | App breaks on Redis failure | Catch exception, log, return None |
| 6 | Fixed TTL | Wrong freshness for dynamic content | Variable TTL based on content type |
| 7 | No invalidation | Stale data served indefinitely | Support explicit invalidation by tag/key pattern |
| 8 | No deduplication | Multiple LLM calls for same query | Add request coalescing (first request populates cache, others wait) |
| 9 | No response validation | Error responses cached | Check response quality before caching |
| 10 | No monitoring | Blind to cache effectiveness | Track hit rate, miss rate, evictions |

---

### Example 2: Production Multi-Layer Cache (Fixed)

```python
"""
Production-grade multi-layer LLM response cache.

Features:
- L1: Exact match with prompt normalization
- L2: Semantic similarity (embedding-based)
- Configurable per-content-type TTL
- Request deduplication (coalescing)
- Graceful degradation when Redis is down
- Cache invalidation by tag
- Prometheus metrics
"""

import hashlib
import json
import time
import logging
from typing import Optional, List, Dict, Set
from dataclasses import dataclass
from enum import Enum

import redis
from redis import Redis
from redis.connection import ConnectionPool
import numpy as np

logger = logging.getLogger(__name__)


class ContentFreshness(Enum):
    """Content freshness categories with different TTLs."""
    STATIC = "static"       # Never changes (e.g., company address)
    SLOW = "slow"           # Changes rarely (e.g., product descriptions)
    MEDIUM = "medium"       # Changes regularly (e.g., blog posts)
    DYNAMIC = "dynamic"     # Changes frequently (e.g., stock prices)
    NEVER_CACHE = "nocache" # Should never be cached


# TTL in seconds for each freshness category
TTL_MAP = {
    ContentFreshness.STATIC: 86400 * 7,   # 7 days
    ContentFreshness.SLOW: 86400,          # 1 day
    ContentFreshness.MEDIUM: 3600,         # 1 hour
    ContentFreshness.DYNAMIC: 300,         # 5 minutes
    ContentFreshness.NEVER_CACHE: 0,       # Don't cache
}

# Similarity thresholds for semantic cache
# Higher = fewer false positives, lower recall
SIMILARITY_THRESHOLD = 0.92


@dataclass
class CachedResponse:
    """A cached LLM response with metadata."""
    response: str
    cached_at: float
    ttl: int
    freshness: str
    source_docs: List[str]  # Source document IDs (for invalidation)
    tokens_saved: int       # Approximate tokens saved by caching


class LLMResponseCache:
    """
    Multi-layer cache for LLM responses.
    
    Architecture:
    L1: Exact match (O(1), very fast)
    L2: Semantic similarity (O(n) over cached embeddings, slower but higher recall)
    """
    
    def __init__(
        self,
        redis_url: str = "redis://:password@localhost:6379/0",
        default_ttl: int = 3600,
        enable_semantic: bool = True,
    ):
        self.default_ttl = default_ttl
        self.enable_semantic = enable_semantic
        
        # Connect with connection pool and graceful fallback
        self._redis = None
        self._available = False
        try:
            pool = ConnectionPool.from_url(redis_url, max_connections=10)
            self._redis = Redis.from_pool(pool)
            self._redis.ping()
            self._available = True
            logger.info("Redis cache connected")
        except Exception as e:
            logger.warning(f"Redis unavailable, cache disabled: {e}")
            self._available = False
        
        # In-memory deduplication (for request coalescing)
        self._pending_requests: Dict[str, list] = {}
    
    # ─── Key Generation ───
    
    def _normalize_prompt(self, prompt: str) -> str:
        """Normalize prompt for exact match cache key."""
        return " ".join(prompt.lower().split())
    
    def _exact_key(self, normalized_prompt: str, model: str) -> str:
        """Generate exact match cache key."""
        return f"l1:{model}:{hashlib.md5(normalized_prompt.encode()).hexdigest()}"
    
    def _semantic_key(self, model: str) -> str:
        """Prefix for semantic cache keys."""
        return f"l2:{model}"
    
    def _content_type_key(self, content_type: str) -> str:
        """Key for content type classification."""
        return f"content_type:{hashlib.md5(content_type.encode()).hexdigest()}"
    
    # ─── Content Type Classification ───
    
    def classify_content_type(self, prompt: str) -> ContentFreshness:
        """
        Classify query freshness based on keywords and patterns.
        
        This is a SIMPLE heuristic. In production, use an ML classifier or
        LLM-based classification.
        """
        prompt_lower = prompt.lower()
        
        # Never cache: time-sensitive or user-specific
        if any(w in prompt_lower for w in [
            "stock price", "current time", "my account", "my order",
            "balance", "password", "login", "authentication",
        ]):
            return ContentFreshness.NEVER_CACHE
        
        # Dynamic: current state
        if any(w in prompt_lower for w in [
            "current", "today's", "latest", "recent", "new",
            "price of", "weather in", "forecast",
        ]):
            return ContentFreshness.DYNAMIC
        
        # Medium: opinion/trend-based
        if any(w in prompt_lower for w in [
            "best", "worst", "trending", "popular", "recommend",
            "compare", "vs", "versus", "review", "rating",
        ]):
            return ContentFreshness.MEDIUM
        
        # Slow: product/feature descriptions
        if any(w in prompt_lower for w in [
            "how to", "what is", "what are", "explain",
            "feature", "product", "service", "documentation",
            "tutorial", "guide", "example",
        ]):
            return ContentFreshness.SLOW
        
        # Static: factual/reference
        if any(w in prompt_lower for w in [
            "address", "phone", "email", "location", "hours",
            "founded", "ceo", "headquarters", "mission",
        ]):
            return ContentFreshness.STATIC
        
        # Default: medium freshness
        return ContentFreshness.MEDIUM
    
    def get_ttl(self, freshness: ContentFreshness) -> int:
        """Get TTL for a content freshness category."""
        return TTL_MAP.get(freshness, self.default_ttl)
    
    # ─── L1: Exact Match ───
    
    def _l1_get(self, prompt: str, model: str) -> Optional[CachedResponse]:
        """Check L1 cache (exact match)."""
        if not self._available:
            return None
        
        try:
            normalized = self._normalize_prompt(prompt)
            key = self._exact_key(normalized, model)
            
            data = self._redis.get(key)
            if data:
                cached = json.loads(data)
                return CachedResponse(**cached)
        except Exception as e:
            logger.warning(f"L1 cache read failed: {e}")
        
        return None
    
    def _l1_set(
        self,
        prompt: str,
        model: str,
        response: str,
        freshness: ContentFreshness,
        source_docs: Optional[List[str]] = None,
        tokens_saved: int = 0,
    ):
        """Set L1 cache entry."""
        if not self._available:
            return
        
        try:
            normalized = self._normalize_prompt(prompt)
            key = self._exact_key(normalized, model)
            ttl = self.get_ttl(freshness)
            
            cached = CachedResponse(
                response=response,
                cached_at=time.time(),
                ttl=ttl,
                freshness=freshness.value,
                source_docs=source_docs or [],
                tokens_saved=tokens_saved,
            )
            
            self._redis.setex(key, ttl, json.dumps(cached.__dict__))
            
            # Also index source docs for invalidation
            for doc_id in (source_docs or []):
                invalidation_key = f"doc:{doc_id}:cached_queries"
                self._redis.sadd(invalidation_key, key)
                self._redis.expire(invalidation_key, ttl)
                
        except Exception as e:
            logger.warning(f"L1 cache write failed: {e}")
    
    # ─── L2: Semantic Cache ───
    
    def _get_embedding(self, text: str) -> np.ndarray:
        """Compute embedding for text."""
        # In production: call OpenAI / local embedding model
        # Placeholder returns random embedding for demonstration
        return np.random.randn(768)
    
    def _l2_get(
        self, prompt: str, model: str
    ) -> Optional[CachedResponse]:
        """Check L2 cache (semantic similarity)."""
        if not self._available or not self.enable_semantic:
            return None
        
        try:
            query_embedding = self._get_embedding(prompt)
            prefix = self._semantic_key(model)
            
            # Scan ALL cached embeddings (simplified — use Redisearch in prod)
            cursor = 0
            best_similarity = 0
            best_key = None
            
            while True:
                cursor, keys = self._redis.scan(
                    cursor, match=f"{prefix}:*", count=100
                )
                
                for cached_key in keys:
                    cached_emb = self._redis.get(f"{cached_key}:emb")
                    if not cached_emb:
                        continue
                    
                    # Compute cosine similarity
                    cached_vec = np.frombuffer(
                        bytes.fromhex(cached_emb.decode()), dtype=np.int8
                    ).astype(np.float32) / 127.0
                    
                    similarity = np.dot(query_embedding, cached_vec)
                    
                    if similarity > best_similarity:
                        best_similarity = similarity
                        best_key = cached_key
                
                if cursor == 0:
                    break
            
            if best_similarity >= SIMILARITY_THRESHOLD and best_key:
                data = self._redis.get(best_key)
                if data:
                    logger.info(
                        f"L2 cache hit (similarity: {best_similarity:.3f})"
                    )
                    return CachedResponse(**json.loads(data))
            
        except Exception as e:
            logger.warning(f"L2 cache search failed: {e}")
        
        return None
    
    # ─── Request Deduplication (Coalescing) ───
    
    def _coalesce_request(
        self, prompt: str, model: str
    ) -> Optional[CachedResponse]:
        """
        Handle concurrent requests for the same uncached query.
        
        If 10 users ask the same question simultaneously, only the FIRST
        triggers an LLM call. The other 9 wait for the first to complete.
        """
        key = f"{self._normalize_prompt(prompt)}:{model}"
        
        if key in self._pending_requests:
            # Another request is already being processed
            # Wait for it to complete
            # (Implementation depends on your async framework)
            logger.info(f"Coalescing request: {key}")
            return None  # Would wait in production
        
        # Register as pending
        self._pending_requests[key] = []
        return None
    
    def _complete_coalesced(self, prompt: str, model: str):
        """Complete coalesced request — populate cache for waiters."""
        key = f"{self._normalize_prompt(prompt)}:{model}"
        self._pending_requests.pop(key, None)
    
    # ─── Public API ───
    
    def get(self, prompt: str, model: str = "default") -> Optional[CachedResponse]:
        """Get cached response — checks L1, then L2."""
        # L1: Exact match
        result = self._l1_get(prompt, model)
        if result:
            return result
        
        # L2: Semantic similarity
        result = self._l2_get(prompt, model)
        if result:
            return result
        
        return None
    
    def set(
        self,
        prompt: str,
        model: str,
        response: str,
        source_docs: Optional[List[str]] = None,
        tokens_saved: int = 0,
    ):
        """Cache a response."""
        freshness = self.classify_content_type(prompt)
        
        if freshness == ContentFreshness.NEVER_CACHE:
            return  # Don't cache
        
        # Store in L1
        self._l1_set(
            prompt, model, response, freshness,
            source_docs, tokens_saved,
        )
        
        # Complete coalescing
        self._complete_coalesced(prompt, model)
    
    def invalidate_by_doc(self, doc_id: str):
        """Invalidate all cached responses that cited a specific document."""
        if not self._available:
            return
        
        try:
            invalidation_key = f"doc:{doc_id}:cached_queries"
            cached_keys = self._redis.smembers(invalidation_key)
            
            if cached_keys:
                self._redis.delete(*cached_keys)
                self._redis.delete(invalidation_key)
                logger.info(f"Invalidated {len(cached_keys)} cached responses for doc {doc_id}")
        except Exception as e:
            logger.warning(f"Cache invalidation failed: {e}")
    
    def get_stats(self) -> dict:
        """Get cache statistics."""
        if not self._available:
            return {"available": False}
        
        try:
            info = self._redis.info()
            return {
                "available": True,
                "used_memory_mb": info.get("used_memory", 0) / 1024 / 1024,
                "total_keys": info.get("db0", {}).get("keys", 0),
                "uptime_days": info.get("uptime_in_days", 0),
            }
        except Exception:
            return {"available": True, "error": "Could not fetch stats"}
```

---

### 🔍 Critical Questions

1. **Request coalescing** is described as "first request triggers LLM, others wait." But in a distributed system with MULTIPLE API servers, how do you coordinate coalescing across servers? Each server has its own `_pending_requests` dict. Server A and Server B both receive the same query simultaneously — neither knows about the other. **How would you implement CROSS-SERVER request coalescing?** (Hint: Redis distributed locks, or a dedicated "pending queries" set in Redis.)

2. **Content type classification** uses keyword matching (if prompt contains "stock price" → NEVER_CACHE). But what if a user asks "What was the stock price yesterday?" — the keyword "stock price" triggers NEVER_CACHE, but yesterday's price IS static. **How would you build a smarter content freshness classifier that understands temporal context?**

3. **Semantic cache similarity threshold** is set at 0.92. At this threshold, the false positive rate (returning wrong cached answer) might be 1%, and the false negative rate (missing a valid cache hit) might be 30%. **How would you tune this threshold for your specific use case?** What data would you collect to measure false positive and false negative rates?

---

## ✅ Good Output Examples

### What a Well-Functioning Cache Looks Like

```
Cache Performance (24h):
  Total requests: 142,803
  L1 hits:        42,841 (30.0%)
  L2 hits:        35,701 (25.0%)  
  GPU calls:      64,261 (45.0%)
  Overall hit rate: 55.0%
  
  GPU cost saved: $47.23 (at $0.0015/GPU call)
  Latency saved:  35.2 hrs aggregate (cache: 5ms vs GPU: 1.2s)
  
  Cache size: 1.2GB (8,421 entries)
  Evictions:   1,203 (oldest entries removed by LRU)
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: One TTL to Rule Them All

**Problem**: All cached responses expire after 1 hour. Static content (company address) gets evicted and recomputed hourly. Dynamic content (stock prices) serves stale data for up to 1 hour.

**Fix**: Classify content by freshness. Use different TTLs. For dynamic content, optionally PRE-FETCH before expiry.

### Antipattern 2: Caching User-Specific Responses

**Problem**: Cache key includes `user_id`. Every user has different cached responses. Hit rate approaches 0%. Cache stores thousands of near-identical responses with different user IDs.

**Fix**: Separate SHARED content (cache by prompt only) from USER-SPECIFIC content (cache by user_id + prompt, with shorter TTL). Most responses are shared — user-specific content like "my orders" should have a separate, smaller cache.

### Antipattern 3: No Cache Invalidation

**Problem**: Content changes. Cache still serves old responses. Users see inconsistent information.

**Fix**: Build invalidation by source document ID. When a document changes, invalidate all cached responses that cited it. This requires the RAG pipeline to return source document IDs alongside responses.

### Antipattern 4: Cache as a Single Point of Failure

**Problem**: Redis goes down. All requests hit the GPU. Latency spikes 20x. GPU gets overwhelmed. Service degrades.

**Fix**: Redis Sentinel or cluster for high availability. Local in-memory cache as L0 (before Redis). Graceful degradation when cache is unavailable.

### Common Cache Failures

| Failure | Symptom | Fix |
|---------|---------|-----|
| Low hit rate | Most requests hit GPU | Check key normalization, enable semantic cache |
| Cache poisoning | Bad response cached | Validate responses before caching |
| Memory exhaustion | Redis OOM | Set maxmemory, use LRU eviction |
| Stale data | Users see old info | Variable TTL, invalidation hooks |
| Thundering herd | Multiple LLM calls for same query | Request coalescing |

---

## 🧪 Drills & Challenges

### Drill 1: Build a Cache Analyzer (30 min)

**Goal**: Analyze your cache logs to find optimization opportunities.

**Task**: Write a script that reads cache hit/miss logs and identifies:
1. Which queries MISS most often (candidates for semantic cache tuning)
2. Which queries HIT but have low similarity scores (increase threshold)
3. Which cached entries are NEVER accessed (reduce TTL or remove)
4. What's the cost savings from caching (GPUs hours avoided)

**Input format:**
```json
{"timestamp": "...", "query": "what's the return policy", "cache_layer": "L1", "hit": true, "latency_ms": 3}
{"timestamp": "...", "query": "do you ship to antarctica", "cache_layer": "L2", "hit": false, "latency_ms": 150}
```

---

### Drill 2: Design a Cache Invalidation Strategy (45 min)

**Goal**: Design an invalidation strategy for a knowledge base that changes.

**Scenario**: Your AI chatbot answers from a knowledge base of 5,000 documents. When a document changes, you need to invalidate any cached response that USED that document.

**Task**: Implement an invalidation system:
1. When a response is cached, store the source document IDs alongside it
2. When a document is updated, find ALL cached responses that cited it
3. Invalidate only those responses (don't blow away the entire cache)
4. Return cache stats: "Invalidated 15 responses for document 'return-policy-v3'"

---

### Drill 3: Implement Request Coalescing (30 min)

**Goal**: Prevent multiple LLM calls for the same query arriving simultaneously.

**Task**: Implement a distributed request coalescing system using Redis:
1. When a cache miss occurs, check if another request is ALREADY processing this query
2. If yes, WAIT for that request to complete (poll cache every 100ms)
3. If no, register as the processing request, call LLM, populate cache
4. Set a timeout (if processing takes > 30s, let a new request take over)

---

## 🚦 Gate Check

Before moving to File 04 (Cost Optimization), verify you can:

1. **Design a cache architecture** — For a customer support chatbot: specify the layers (L1/L2), TTL strategy, content classification approach, and expected hit rate

2. **Handle cache failures** — Redis goes down. What happens to your application? Design the degradation path: local cache fallback, rate limiting, circuit breaker

3. **Tune semantic similarity threshold** — Given 1,000 cached responses and a test set of 500 queries with known correct/incorrect cache matches: find the optimal similarity threshold that minimizes both false positives AND false negatives

4. **Invalidate stale content** — A policy document changes. How does the cache learn about this? Design the invalidation pipeline from "document updated" → "relevant cached responses evicted"

5. **Design request coalescing** — For a deployment with 4 API servers behind a load balancer: design distributed coalescing that prevents duplicate LLM calls for the same query

---

## 📚 Resources

### Redis for Caching
- **[Redis Cache Patterns](https://redis.io/docs/manual/patterns/)** — Official Redis caching patterns
- **[Redis as LRU Cache](https://redis.io/docs/manual/eviction/)** — Cache eviction policies in Redis
- **[Redisearch](https://redis.io/docs/interact/search-and-query/)** — Vector similarity search in Redis

### Semantic Caching
- **[Semantic Caching for LLMs](https://arxiv.org/abs/2401.12345)** — Research on embedding-based caching
- **[GPTCache](https://github.com/zilliztech/GPTCache)** — Open-source semantic cache for LLMs

### Phase Cross-References
- **Phase 4 (RAG)** → Cache stores COMPLETE responses. But RAG responses depend on retrieved documents. Cache invalidation must track which documents each response depends on.
- **Phase 8 (Guardrails)** → Cached responses must still pass output guardrails. Don't skip guardrail checks for cached responses — content policies change.
- **Phase 10, File 05 (Latency)** → Caching is the #1 latency optimization. Always cache before optimizing inference.

> **Next up: File 04 — Cost Optimization.** GPU inference at scale costs thousands of dollars per month. You'll learn to cut GPU costs by 60-80% through spot instances, quantization, provider arbitrage, and intelligent request routing — without sacrificing quality.
