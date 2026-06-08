# 07 — Production RAG: Caching, Streaming, Monitoring & Optimization

## 🎯 Purpose & Goals

You've built a RAG pipeline. You've evaluated it. It works. Now you need to **make it work in production** — where users expect answers in under 2 seconds, costs can spiral out of control, and failures need to be detected before users notice them.

This module is about the engineering layer between "my RAG system works" and "my RAG system works reliably, cost-effectively, and observably at scale." Most RAG tutorials stop at the pipeline. Production engineering is what separates a demo from a product.

**By the end of this module, you will:**

1. Design a multi-level caching strategy that reduces latency by 60-80% without sacrificing answer quality
2. Implement streaming responses that show answers word by word while maintaining citation integrity
3. Build monitoring dashboards that track the metrics that actually matter (not just latency and cost)
4. Identify and eliminate the top 3 latency bottlenecks in any RAG pipeline
5. Implement cost controls that automatically manage provider spend across query types
6. Design error handling that degrades gracefully instead of failing hard

---

### ⏹ STOP. Answer these questions before proceeding.

**Question 1 — The Latency Budget**

*You have a RAG pipeline with these components:*
- *Query rewriting: 300ms*
- *Dense retrieval (vector DB): 150ms*
- *Sparse retrieval (BM25): 100ms*
- *Reranking: 200ms*
- *Context assembly: 50ms*
- *LLM generation (100 tokens): 800ms*
- *Total: 1,600ms*

*Your SLO is p95 < 2,000ms. You're at 1,600ms for the happy path, but p95 hits 3,500ms because of LLM cold starts and retrieval timeouts. A competitor launches with sub-second response times.*

*Where do you get the biggest latency improvements? Is it in the components you can see (the ones listed above) or somewhere else? What's the cheapest optimization (least engineering effort for most latency gain)? What's the hardest but most impactful? Can you cache anything here? If the LLM takes 800ms to generate 100 tokens, how much of that is actual computation vs. network overhead?*

**Question 2 — The Cost Spiral**

*You launch your RAG system. It's popular. Traffic grows 5x in the first month. Your LLM costs grow 5x. Your vector DB costs grow 5x. Your total infrastructure cost is now $12,000/month and growing.*

*Your CTO asks: "Can we cache repeat questions? How many of our queries are unique vs. repeated?" You check: 40% of queries are exact repeats, 25% are semantic repeats (same intent, different wording), 35% are truly novel.*

*Design a caching strategy that captures the 40% exact repeats trivially and at least half of the 25% semantic repeats. What's the cache hit rate you can achieve? What's the cost savings? What's the risk of serving stale answers from cache? How do you invalidate the cache when documents change?*

**Question 3 — The Monitoring Blind Spot**

*Your RAG system has 3 dashboards:*
1. *Latency dashboard (p50/p95/p99 response times)*
2. *Error dashboard (5xx rates, timeout counts)*
3. *Cost dashboard (LLM spend per day)*

*Everything looks green. Latency is good. Errors are below 0.1%. Cost is within budget.*

*But user satisfaction is dropping. The dashboards are green while users are unhappy. What are you NOT measuring? What's the metric that would have caught the decline before users started complaining? What leading indicators (predictive, not lagging) should you track?*

*Dig deeper: If your answer quality drops from 0.92 to 0.85 in the evaluation pipeline, would your monitoring catch it? If not, how do you build an "answer quality" SLO that runs continuously, not just during evaluation sprints?*

**After answering, proceed to the deep-dive. Production engineering is a different skill than pipeline building.**

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 15 min |
| Caching Strategy | 35 min |
| Streaming Responses | 30 min |
| Latency Optimization | 35 min |
| Cost Management | 25 min |
| Monitoring & Observability | 30 min |
| Error Handling & Resilience | 25 min |
| Scaling & Concurrency | 20 min |
| Code: Production RAG Server | 60 min |
| Code: Cache Layer | 30 min |
| Code: Monitoring Setup | 30 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~6 hours** |

---

## 📖 Concept Deep-Dive

### 1. Caching Strategy

Caching is the single highest-leverage optimization for production RAG. A good cache can reduce p95 latency by 70% and cut costs by 50%.

#### Cache Level 1: Exact Query Cache

**What:** Cache the exact (query → answer) mapping.

**When to use:** For frequently asked questions — "What's your return policy?" asked 1000 times/day.

**Implementation:**

```python
"""
cache_layer.py — Multi-level caching for RAG systems.
"""

import hashlib
import json
import time
from typing import Optional, Dict, Any, List
from dataclasses import dataclass
from datetime import datetime, timedelta
import redis.asynced as redis


@dataclass
class CacheEntry:
    """A cached response with metadata."""
    query: str
    answer: str
    contexts_used: List[str]  # IDs of chunks used to generate this answer
    created_at: datetime
    ttl_seconds: int
    hit_count: int = 0
    last_accessed: Optional[datetime] = None


class ExactQueryCache:
    """
    Cache for exact query matches.

    Fastest cache (O(1) lookup). Only matches identical queries.
    """

    def __init__(self, redis_client, default_ttl: int = 3600):
        self.redis = redis_client
        self.default_ttl = default_ttl
        self.prefix = "rag_cache:exact:"

    def _key(self, query: str) -> str:
        """Generate cache key from query."""
        normalized = query.lower().strip()
        return self.prefix + hashlib.sha256(
            normalized.encode()
        ).hexdigest()

    async def get(self, query: str) -> Optional[Dict]:
        """Get cached response for exact query match."""
        key = self._key(query)
        data = await self.redis.get(key)

        if data is None:
            return None

        entry = json.loads(data)

        # Update access stats (fire and forget)
        await self.redis.hincrby("rag_cache:stats", "exact_hits", 1)

        return {
            "answer": entry["answer"],
            "cached": True,
            "cache_level": "exact",
            "created_at": entry["created_at"]
        }

    async def set(
        self,
        query: str,
        answer: str,
        contexts_used: List[str],
        ttl: Optional[int] = None
    ):
        """Cache a response."""
        key = self._key(query)
        entry = {
            "query": query,
            "answer": answer,
            "contexts_used": contexts_used,
            "created_at": datetime.now().isoformat(),
            "ttl_seconds": ttl or self.default_ttl
        }

        await self.redis.setex(
            key,
            ttl or self.default_ttl,
            json.dumps(entry)
        )

    async def invalidate(self, query: str):
        """Invalidate a specific cached response."""
        key = self._key(query)
        await self.redis.delete(key)

    async def invalidate_by_context(self, context_id: str):
        """
        Invalidate all cached responses that used a specific context chunk.

        Called when a document is updated — all answers that cited that
        document become potentially stale.
        """
        # This requires a reverse index: context_id -> queries
        # Maintained separately (see below)
        queries_key = f"rag_cache:context_index:{context_id}"
        query_keys = await self.redis.smembers(queries_key)

        if query_keys:
            await self.redis.delete(*query_keys)
            await self.redis.delete(queries_key)
```

#### Cache Level 2: Semantic Query Cache

**What:** Cache similar queries to the same answer.

**When to use:** For queries that are semantically identical but phrased differently — "What's the refund policy?" vs. "Can I get my money back?"

**How it works:** Embed the query, find similar cached queries, and if similarity is above threshold, return the cached answer.

```python
class SemanticQueryCache:
    """
    Cache for semantically similar queries.

    Embeds queries and finds similar ones in a vector store.
    More complex than exact cache but captures more reuse.
    """

    def __init__(
        self,
        embedding_model,
        similarity_threshold: float = 0.95,
        max_results: int = 1
    ):
        self.embedding_model = embedding_model
        self.similarity_threshold = similarity_threshold
        self.max_results = max_results
        self.cache_store = []  # In production, use a vector DB

    async def get(self, query: str) -> Optional[Dict]:
        """Find a semantically similar cached query."""
        query_emb = await self.embedding_model.embed(query)

        best_match = None
        best_similarity = 0.0

        for entry in self.cache_store:
            similarity = self._cosine_sim(query_emb, entry["embedding"])
            if similarity > best_similarity:
                best_similarity = similarity
                best_match = entry

        if best_match and best_similarity >= self.similarity_threshold:
            return {
                "answer": best_match["answer"],
                "cached": True,
                "cache_level": "semantic",
                "matched_query": best_match["query"],
                "similarity": best_similarity
            }

        return None

    async def set(
        self,
        query: str,
        embedding: List[float],
        answer: str
    ):
        """Store a new entry in the semantic cache."""
        self.cache_store.append({
            "query": query,
            "embedding": embedding,
            "answer": answer,
            "created_at": datetime.now()
        })

        # In production: cap size, implement eviction
        if len(self.cache_store) > 10000:
            self.cache_store = self.cache_store[-5000:]

    def _cosine_sim(self, a, b):
        import numpy as np
        a, b = np.array(a), np.array(b)
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
```

**Important nuance**: The semantic similarity threshold needs tuning. Too high (0.99) → almost never matches. Too low (0.80) → serves wrong answers for different but similarly-worded questions.

#### Cache Level 3: Context Cache

**What:** Cache the (query → retrieved_chunks) mapping, but regenerate the LLM answer each time.

**When to use:** When retrieval is expensive but generation is cheap (or when you want fresher answers while skipping the retrieval step).

```python
class RetrieverCache:
    """
    Cache retrieval results only. Regenerate answers each time.

    This is a good middle ground:
    - Saves 200-500ms on retrieval per query
    - Still gets fresh LLM answers (so document updates are reflected)
    - Doesn't need invalidation on document changes (only on embedding model changes)
    """

    async def get(self, query: str) -> Optional[List[Chunk]]:
        """Get cached retrieval results."""
        key = f"retrieval:{self._hash(query)}"
        data = await self.redis.get(key)
        if data:
            return [Chunk(**c) for c in json.loads(data)]
        return None

    async def set(self, query: str, chunks: List[Chunk]):
        """Cache retrieval results."""
        key = f"retrieval:{self._hash(query)}"
        data = [c.to_dict() for c in chunks]
        await self.redis.setex(key, 3600, json.dumps(data))
```

#### Cache Level 4: Prefix Cache

**What:** Cache the LLM's KV cache for common prefix text (system prompts + common context).

**When to use:** When your system prompt is large (500+ tokens) and shared across many queries.

**Note:** This only works with LLM providers that support KV caching (most do not expose it via API). For self-hosted models with vLLM or TensorRT-LLM, prefix caching is extremely effective.

```python
# Pseudo-code for prefix caching with vLLM
# Not all providers support this
response = await llm.complete(
    prompt=prompt,
    prefix_pos=len(system_prompt),  # Cache the system prompt tokens
    # vLLM automatically caches and reuses the system prompt's KV cache
)
```

#### Cache Invalidation Strategy

The hardest part of caching. Here are your options:

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| **TTL-based** | Cache expires after fixed time (e.g., 1 hour) | Dynamic content (pricing, policies) |
| **Event-based** | Cache invalidated when source document changes | Static content (documentation) |
| **Version-based** | Cache tagged with document version; new version → new cache | Versioned documentation |
| **No invalidation** | Cache forever; serve "as of" timestamps | Historical data, reference material |

**The context index pattern:**

```python
class ContextAwareCache:
    """
    Cache that tracks which context chunks were used for each answer.

    Enables event-based invalidation:
    When a document is updated, ALL cached answers that cited it
    are automatically invalidated.
    """

    async def _index_contexts(self, cache_key: str, chunks: List[Chunk]):
        """Build reverse index: context_id -> cache_keys."""
        for chunk in chunks:
            index_key = f"rag_cache:ctx_idx:{chunk.id}"
            await self.redis.sadd(index_key, cache_key)
            # Set expiry on index too (cleanup old entries)
            await self.redis.expire(index_key, self.default_ttl * 2)

    async def on_document_updated(self, document_id: str):
        """Invalidate all caches that used this document."""
        # Find all chunk IDs for this document
        chunk_ids = await self.redis.smembers(
            f"rag_cache:doc_chunks:{document_id}"
        )

        for chunk_id in chunk_ids:
            # Find all cache entries that used this chunk
            cache_keys = await self.redis.smembers(
                f"rag_cache:ctx_idx:{chunk_id}"
            )
            if cache_keys:
                await self.redis.delete(*cache_keys)

            # Clean up index
            await self.redis.delete(f"rag_cache:ctx_idx:{chunk_id}")
```

---

### 2. Streaming Responses

Users perceive a system as faster if it starts responding immediately, even if total time is the same. Streaming is the #1 UX optimization for LLM applications.

#### The Citation Problem in Streaming

Streaming works fine for plain text. But RAG needs citations. If the model says "The rate limit is 1000 requests per minute [SOURCE 3]," the citation needs to appear at the right time without breaking the stream.

**Solution: Citation chunking**

```python
"""
streaming_rag.py — Streaming RAG response with citations.
"""

import asyncio
import json
from typing import AsyncGenerator, List, Dict
from pydantic import BaseModel


class StreamEvent(BaseModel):
    """Events emitted during streaming RAG response."""
    event_type: str  # "token", "citation", "done", "error"
    data: Dict


class StreamingRAGResponse:
    """
    Streaming RAG response handler.

    Yields tokens incrementally. Citations are sent as separate
    events so the frontend can render them inline.
    """

    def __init__(self, rag_pipeline, cache_layer=None):
        self.rag = rag_pipeline
        self.cache = cache_layer

    async def stream_answer(self, query: str) -> AsyncGenerator[str, None]:
        """
        Stream a RAG answer with structured events.

        Yields JSON-encoded StreamEvent objects that the frontend
        can parse and render progressively.
        """

        # Check cache first (non-streaming path for cached answers)
        if self.cache:
            cached = await self.cache.get(query)
            if cached:
                yield self._event("cache_hit", {"answer": cached["answer"]})
                # Still stream tokens for UX consistency
                for token in cached["answer"].split():
                    yield self._event("token", {"token": token + " "})
                    await asyncio.sleep(0.01)  # Simulate streaming
                yield self._event("done", {"cached": True})
                return

        # Phase 1: Retrieve (send status event)
        yield self._event("status", {
            "phase": "retrieving",
            "message": "Searching documents..."
        })

        chunks = await self.rag.retrieve(query)

        if not chunks:
            yield self._event("status", {
                "phase": "error",
                "message": "No relevant documents found."
            })
            yield self._event("done", {"error": "no_documents"})
            return

        # Phase 2: Build context (send source info)
        yield self._event("status", {
            "phase": "context",
            "message": f"Found {len(chunks)} relevant documents."
        })

        # Send source metadata for frontend display
        sources = [
            {
                "id": i + 1,
                "title": c.document_title,
                "relevance": c.score,
                "preview": c.text[:100]
            }
            for i, c in enumerate(chunks)
        ]
        yield self._event("sources", {"sources": sources})

        context = self.rag.build_context(chunks, query)

        # Phase 3: Generate (stream tokens)
        yield self._event("status", {
            "phase": "generating",
            "message": "Generating answer..."
        })

        # Track citations seen during generation
        citations_seen = set()
        full_text = ""

        async for event in self.rag.generate_stream(context, query):
            if event["type"] == "token":
                full_text += event["token"]
                yield self._event("token", {"token": event["token"]})

            elif event["type"] == "citation":
                # Deduplicate citations (same source cited multiple times)
                source_id = event["source_id"]
                if source_id not in citations_seen:
                    citations_seen.add(source_id)
                    yield self._event("citation", {
                        "source_id": source_id,
                        "title": sources[source_id - 1]["title"]
                            if source_id <= len(sources) else "Unknown"
                    })

        # Phase 4: Done
        yield self._event("done", {
            "full_text": full_text,
            "citations": list(citations_seen),
            "chunks_used": len(chunks),
            "token_count": len(full_text.split())
        })

        # Cache for future
        if self.cache:
            await self.cache.set(query, full_text, [c.id for c in chunks])

    def _event(self, event_type: str, data: Dict) -> str:
        """Format a stream event as SSE."""
        event = StreamEvent(event_type=event_type, data=data)
        return f"data: {json.dumps(event.model_dump())}\n\n"


# ============================================================
# FastAPI Server with Streaming
# ============================================================

from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from sse_starlette.sse import EventSourceResponse

app = FastAPI()
rag_streamer = StreamingRAGResponse(rag_pipeline)


@app.post("/api/chat/stream")
async def chat_stream(request: Request):
    """Streaming RAG chat endpoint."""
    body = await request.json()
    query = body["query"]

    return EventSourceResponse(
        rag_streamer.stream_answer(query)
    )


@app.post("/api/chat")
async def chat_sync(request: Request):
    """Non-streaming RAG endpoint (for backward compatibility)."""
    body = await request.json()
    query = body["query"]

    answer, sources = await rag_pipeline.answer(query)

    return {
        "answer": answer,
        "sources": [
            {"id": i, "title": c.document_title, "score": c.score}
            for i, c in enumerate(sources)
        ]
    }
```

**Frontend handling:**

```javascript
// Client-side streaming handler
const eventSource = new EventSource('/api/chat/stream');

const answerDiv = document.getElementById('answer');
const sourcesDiv = document.getElementById('sources');
let answerText = '';

eventSource.onmessage = (event) => {
    const payload = JSON.parse(event.data);

    switch (payload.event_type) {
        case 'token':
            answerText += payload.data.token;
            answerDiv.innerHTML = renderMarkdown(answerText);
            break;

        case 'citation':
            // Add citation badge
            const badge = document.createElement('sup');
            badge.textContent = `[${payload.data.source_id}]`;
            badge.title = payload.data.title;
            answerDiv.appendChild(badge);
            break;

        case 'sources':
            // Display source documents in sidebar
            renderSources(payload.data.sources, sourcesDiv);
            break;

        case 'done':
            eventSource.close();
            break;

        case 'error':
            showError(payload.data.message);
            eventSource.close();
            break;
    }
};
```

---

### 3. Latency Optimization

Every RAG pipeline has a latency profile. You need to know where the time goes before you can optimize.

#### The Latency Breakdown

```python
"""
latency_profiler.py — Profile RAG pipeline latency.
"""

import time
import asyncio
from contextlib import asynccontextmanager
from dataclasses import dataclass, field
from typing import Dict


@dataclass
class LatencyProfile:
    """Latency breakdown for a single RAG query."""
    total_ms: float = 0.0
    query_rewrite_ms: float = 0.0
    dense_retrieval_ms: float = 0.0
    sparse_retrieval_ms: float = 0.0
    reranking_ms: float = 0.0
    context_assembly_ms: float = 0.0
    llm_first_token_ms: float = 0.0  # Time to first token (TTFT)
    llm_generation_ms: float = 0.0   # Total generation time
    llm_tokens_per_second: float = 0.0

    def summary(self) -> str:
        """Print latency breakdown."""
        lines = [f"Total: {self.total_ms:.0f}ms"]
        lines.append(f"  Query rewrite: {self.query_rewrite_ms:.0f}ms "
                     f"({self.query_rewrite_ms/self.total_ms*100:.0f}%)")
        lines.append(f"  Dense retrieval: {self.dense_retrieval_ms:.0f}ms")
        lines.append(f"  Sparse retrieval: {self.sparse_retrieval_ms:.0f}ms")
        lines.append(f"  Reranking: {self.reranking_ms:.0f}ms")
        lines.append(f"  Context assembly: {self.context_assembly_ms:.0f}ms")
        lines.append(f"  LLM TTFT: {self.llm_first_token_ms:.0f}ms")
        lines.append(f"  LLM generation: {self.llm_generation_ms:.0f}ms "
                     f"({self.llm_tokens_per_second:.0f} tok/s)")
        return "\n".join(lines)


class LatencyProfiler:
    """Profile and log RAG pipeline latency."""

    def __init__(self, enable_profile: bool = True):
        self.enable_profile = enable_profile

    @asynccontextmanager
    async def profile(self, label: str, profile: LatencyProfile, attr: str):
        """Time a block and store in profile."""
        if not self.enable_profile:
            yield
            return

        start = time.perf_counter()
        try:
            yield
        finally:
            elapsed = (time.perf_counter() - start) * 1000
            setattr(profile, attr, elapsed)

    async def run_profiled(self, query: str, rag_pipeline) -> tuple:
        """Run RAG pipeline with full profiling."""
        profile = LatencyProfile()
        start = time.perf_counter()

        async with self.profile("rewrite", profile, "query_rewrite_ms"):
            rewritten = await rag_pipeline.rewrite_query(query)

        async with self.profile("dense", profile, "dense_retrieval_ms"):
            dense_results = await rag_pipeline.dense_retrieve(rewritten)

        async with self.profile("sparse", profile, "sparse_retrieval_ms"):
            sparse_results = await rag_pipeline.sparse_retrieve(rewritten)

        async with self.profile("rerank", profile, "reranking_ms"):
            chunks = await rag_pipeline.rerank(
                dense_results + sparse_results, query
            )

        async with self.profile("context", profile, "context_assembly_ms"):
            context = rag_pipeline.build_context(chunks, query)

        # LLM timing: measure TTFT and generation separately
        llm_start = time.perf_counter()
        first_token = True

        # This depends on your LLM client supporting streaming with timing
        async for event in rag_pipeline.generate_stream(context, query):
            if first_token:
                profile.llm_first_token_ms = (
                    time.perf_counter() - llm_start
                ) * 1000
                first_token = False

        profile.llm_generation_ms = (
            time.perf_counter() - llm_start
        ) * 1000

        profile.total_ms = (time.perf_counter() - start) * 1000
        profile.llm_tokens_per_second = (
            len(context.split()) / (profile.llm_generation_ms / 1000)
            if profile.llm_generation_ms > 0 else 0
        )

        return chunks, context, profile
```

#### The Optimization Hierarchy

Optimize in this order (highest impact first):

**Level 1: Caching (40-60% latency reduction)**
- Exact query cache: 5-10ms lookup vs. 1500ms full pipeline
- Semantic cache: 50ms (embed + search) vs. full pipeline
- Implementation: Simple. Add a cache check at the start of the pipeline.

**Level 2: Parallelization (20-30% reduction)**
- Run dense + sparse retrieval in parallel (not sequential)
- Context assembly overlaps with retrieval

```python
async def parallel_retrieve(query: str) -> List[Chunk]:
    """Run dense and sparse retrieval in parallel."""
    dense_task = asyncio.create_task(dense_retriever.retrieve(query))
    sparse_task = asyncio.create_task(sparse_retriever.retrieve(query))

    dense_results, sparse_results = await asyncio.gather(
        dense_task, sparse_task
    )

    return fuse_results(dense_results, sparse_results)
```

**Level 3: Model Selection (15-25% reduction)**
- Use smaller/faster models for simple queries
- GPT-4o-mini generates 2-3x faster than GPT-4o
- For self-hosted: quantization reduces latency (at cost of quality)

```python
class AdaptiveModelSelector:
    """Choose LLM based on query complexity."""

    async def select_model(self, query: str) -> str:
        complexity = await self._assess_complexity(query)

        if complexity == "simple":
            return "gpt-4o-mini"  # Fast, cheap
        elif complexity == "medium":
            return "gpt-4o"       # Balanced
        else:
            return "gpt-4o"       # Maximum capability
```

**Level 4: Connection Pooling & Keepalive (10-15% reduction)**
- Reuse HTTP connections to LLM APIs (don't open new ones per request)
- Keep database connections pooled
- LLM cold starts add 200-500ms to the first query

```python
# httpx client with connection pooling
import httpx

# Share this across your application
client = httpx.AsyncClient(
    limits=httpx.Limits(
        max_connections=100,
        max_keepalive_connections=20,
        keepalive_expiry=30.0
    ),
    timeout=httpx.Timeout(30.0)
)

# Reuse for all API calls
response = await client.post(
    "https://api.openai.com/v1/chat/completions",
    headers={"Authorization": f"Bearer {api_key}"},
    json={...}
)
```

**Level 5: Streaming (perceived latency reduction)**
- First token in 200ms → user perceives "fast" even if total time is 2s
- Show retrieval progress while generating

**Level 6: Batching (throughput, not single-query latency)**
- Batch multiple embedding queries into one API call
- Batch LLM requests when possible (non-real-time workloads)

---

### 4. Cost Management

RAG costs come from three places:
1. **LLM API calls** (60-80% of total cost)
2. **Embedding API calls** (10-20%)
3. **Infrastructure** (vector DB, server hosting) (10-20%)

#### The Cost Breakdown Calculator

```python
"""
cost_calculator.py — Estimate and track RAG costs.
"""

from dataclasses import dataclass


@dataclass
class CostConfig:
    """Pricing for various models (example rates)."""
    gpt4o_input_per_m: float = 2.50    # $2.50 per million input tokens
    gpt4o_output_per_m: float = 10.00
    gpt4o_mini_input_per_m: float = 0.15
    gpt4o_mini_output_per_m: float = 0.60
    embedding_per_m: float = 0.02       # text-embedding-3-small


class RAGCostEstimator:
    """Estimate per-query and monthly costs."""

    def __init__(self, config: CostConfig = None):
        self.config = config or CostConfig()

    def estimate_per_query(
        self,
        model: str = "gpt-4o-mini",
        input_tokens: int = 2000,   # system + context + question
        output_tokens: int = 300,    # typical answer length
        embedding_queries: int = 2,  # query embedding + optional rewrite
    ) -> dict:
        """Estimate cost for a single RAG query."""

        if model == "gpt-4o-mini":
            input_cost = (input_tokens / 1_000_000) * self.config.gpt4o_mini_input_per_m
            output_cost = (output_tokens / 1_000_000) * self.config.gpt4o_mini_output_per_m
        else:
            input_cost = (input_tokens / 1_000_000) * self.config.gpt4o_input_per_m
            output_cost = (output_tokens / 1_000_000) * self.config.gpt4o_output_per_m

        llm_cost = input_cost + output_cost

        embedding_cost = (embedding_queries * 0.0001) * self.config.embedding_per_m
        # ~0.0001M tokens per embedding query

        return {
            "model": model,
            "input_tokens": input_tokens,
            "output_tokens": output_tokens,
            "llm_cost": round(llm_cost, 6),
            "embedding_cost": round(embedding_cost, 6),
            "total_per_query": round(llm_cost + embedding_cost, 6),
        }

    def estimate_monthly(
        self,
        queries_per_day: int = 10000,
        model: str = "gpt-4o-mini",
        cache_hit_rate: float = 0.3
    ) -> dict:
        """Estimate monthly cost with caching."""

        per_query = self.estimate_per_query(model=model)
        uncached_queries = queries_per_day * (1 - cache_hit_rate)

        daily_cost = uncached_queries * per_query["total_per_query"]
        monthly_cost = daily_cost * 30

        return {
            "queries_per_day": queries_per_day,
            "cache_hit_rate": cache_hit_rate,
            "uncached_per_day": uncached_queries,
            "daily_cost": round(daily_cost, 2),
            "monthly_cost": round(monthly_cost, 2),
            "annual_cost": round(monthly_cost * 12, 2),
        }


# Usage
estimator = RAGCostEstimator()

# Per query
print(estimator.estimate_per_query())

# Monthly projection
print(estimator.estimate_monthly(queries_per_day=50000, cache_hit_rate=0.4))

# With model tiering (90% mini, 10% full)
mini = estimator.estimate_monthly(
    queries_per_day=45000, model="gpt-4o-mini", cache_hit_rate=0.4
)
full = estimator.estimate_monthly(
    queries_per_day=5000, model="gpt-4o", cache_hit_rate=0.3
)
print(f"Total monthly (tiered): ${mini['monthly_cost'] + full['monthly_cost']:.2f}")
```

#### Cost Optimization Strategies

| Strategy | Savings | Complexity | Risk |
|----------|---------|------------|------|
| **Exact query cache** | 30-50% | Low | Stale answers |
| **Semantic cache** | 10-20% additional | Medium | Slightly different answers |
| **Model tiering** | 40-60% | Medium | Wrong model for hard queries |
| **Context length reduction** | 10-30% | Low | Missing info |
| **Batch embedding** | 5-10% | Low | More latency |
| **Prompt compression** | 15-25% | Medium | Compression errors |

#### Cost Monitoring & Alerting

```python
class CostMonitor:
    """Track cost per query and alert on anomalies."""

    def __init__(self, budget_daily: float = 100.0):
        self.budget_daily = budget_daily
        self.today_spend = 0.0

    async def record_query(
        self, model: str, input_tokens: int, output_tokens: int
    ):
        """Record cost of a query."""
        cost = self._compute_cost(model, input_tokens, output_tokens)
        self.today_spend += cost

        # Alert if approaching budget
        if self.today_spend > self.budget_daily * 0.8:
            await self._send_alert(
                f"Daily spend at {self.today_spend:.2f} / {self.budget_daily:.2f} "
                f"({self.today_spend/self.budget_daily*100:.0f}%)"
            )

    def _compute_cost(self, model: str, input_tokens: int, output_tokens: int) -> float:
        rates = {
            "gpt-4o-mini": (0.15e-6, 0.60e-6),   # per token
            "gpt-4o": (2.50e-6, 10.00e-6),
        }
        input_rate, output_rate = rates.get(model, rates["gpt-4o-mini"])
        return input_tokens * input_rate + output_tokens * output_rate
```

---

### 5. Monitoring & Observability

You need three types of monitoring:

1. **Technical monitoring** — Is the system up? Is it fast?
2. **Quality monitoring** — Are the answers good?
3. **Business monitoring** — Are users getting value?

#### What to Monitor

```python
"""
monitoring.py — RAG monitoring metrics.
"""

from dataclasses import dataclass
from typing import Optional
from datetime import datetime


@dataclass
class RAGMetrics:
    """Comprehensive RAG metrics for a single query."""

    # Metadata
    timestamp: datetime
    query_id: str

    # Technical metrics
    total_latency_ms: float
    retrieval_latency_ms: float
    generation_latency_ms: float
    ttft_ms: float  # Time to first token
    tokens_per_second: float

    # Retrieval metrics
    num_chunks_retrieved: int
    max_relevance_score: float
    avg_relevance_score: float
    min_relevance_score: float
    chunks_from_cache: bool = False

    # Context metrics
    context_token_count: int
    context_truncated: bool  # Was context truncated to fit budget?

    # Cost metrics
    model_used: str
    input_tokens: int
    output_tokens: int
    estimated_cost: float

    # Quality metrics
    faithfulness_score: Optional[float] = None
    answer_relevance_score: Optional[float] = None
    user_feedback: Optional[str] = None  # "helpful", "unhelpful", None

    # Error tracking
    had_error: bool = False
    error_type: Optional[str] = None  # "timeout", "api_error", "no_results"
```

#### The Four Dashboards

**Dashboard 1: Operations (real-time)**
```
┌─────────────────────────────────────────────────┐
│ RAG Operations Dashboard                        │
├─────────────────────────────────────────────────┤
│ Queries/min: 142    Error rate: 0.3%            │
│ p50 latency: 850ms  p95 latency: 2100ms         │
│ Cost today: $34.20  Cache hit rate: 38%         │
├─────────────────────────────────────────────────┤
│ [Latency trend chart (last 24h)]                │
│ [Error rate by endpoint]                        │
│ [Cache hit rate by cache level]                 │
│ [Cost by model]                                 │
└─────────────────────────────────────────────────┘
```

**Dashboard 2: Quality (per-query evaluation sample)**
```
┌─────────────────────────────────────────────────┐
│ RAG Quality Dashboard                           │
├─────────────────────────────────────────────────┤
│ Sampled queries last hour: 50/5000 (1%)         │
│ Avg faithfulness: 0.91  ▲ 0.02 from yesterday  │
│ Avg relevance: 0.87     ▼ 0.03 from yesterday  │
│ User helpful rate: 84%  ▼ 0.05 from last week  │
├─────────────────────────────────────────────────┤
│ [Faithfulness distribution]                     │
│ [Queries with low scores]                       │
│ [Trend over last 7 days]                        │
└─────────────────────────────────────────────────┘
```

**Dashboard 3: Cost**
```
┌─────────────────────────────────────────────────┐
│ RAG Cost Dashboard                              │
├─────────────────────────────────────────────────┤
│ Daily spend: $34.20 / $100 budget (34%)         │
│ Monthly projected: $1,026                       │
├─────────────────────────────────────────────────┤
│ Cost by model:                                  │
│   GPT-4o-mini: $22.10 (65%)                     │
│   GPT-4o:       $9.60  (28%)                    │
│   Embeddings:   $2.50  (7%)                     │
├─────────────────────────────────────────────────┤
│ Cost per query (avg): $0.0024                   │
│ Cost efficiency: ▲ 12% this week (caching)      │
└─────────────────────────────────────────────────┘
```

**Dashboard 4: Business**
```
┌─────────────────────────────────────────────────┐
│ RAG Business Impact Dashboard                   │
├─────────────────────────────────────────────────┤
│ Active users: 2,340  ▲ 15% this week            │
│ Queries/user/day: 4.2                           │
│ Avg session duration: 3.2 min                   │
│ User retention (7-day): 68%                     │
├─────────────────────────────────────────────────┤
│ Top query categories:                           │
│   Billing: 28%  Technical: 35%  Product: 22%   │
├─────────────────────────────────────────────────┤
│ Escalation rate: 3.1% (user asked for human)    │
│ Issue resolution rate: 89%                      │
└─────────────────────────────────────────────────┘
```

#### Structured Logging

```python
"""
structured_logging.py — Log RAG queries in structured format.

Each query produces a single JSON line for log aggregation.
"""

import json
import logging
import uuid
from datetime import datetime


class RAGLogger:
    """Structured logger for RAG queries."""

    def __init__(self):
        self.logger = logging.getLogger("rag")
        self.handler = logging.StreamHandler()
        self.handler.setFormatter(logging.Formatter(
            "%(message)s"  # Raw JSON, no prefixes
        ))
        self.logger.addHandler(self.handler)
        self.logger.setLevel(logging.INFO)

    def log_query(self, query: str, answer: str, metrics: RAGMetrics):
        """Log a structured RAG query event."""
        entry = {
            "type": "rag_query",
            "query_id": metrics.query_id,
            "timestamp": metrics.timestamp.isoformat(),
            "query": query,
            "answer_length": len(answer),
            "answer_preview": answer[:200],
            "latency_ms": metrics.total_latency_ms,
            "model": metrics.model_used,
            "cost": metrics.estimated_cost,
            "num_chunks": metrics.num_chunks_retrieved,
            "cache_hit": metrics.chunks_from_cache,
            "error": metrics.error_type if metrics.had_error else None,
            "user_feedback": metrics.user_feedback,
        }
        self.logger.info(json.dumps(entry))

    def log_retrieval(self, query_id: str, query: str, chunks: list):
        """Log retrieval details (separate from query log for volume control)."""
        entry = {
            "type": "retrieval_detail",
            "query_id": query_id,
            "query": query,
            "chunks": [
                {
                    "id": c.id,
                    "doc": c.document_title,
                    "score": c.score,
                    "length": len(c.text)
                }
                for c in chunks
            ]
        }
        self.logger.debug(json.dumps(entry))
```

#### Alerting Thresholds

```python
# alert_rules.py

ALERT_RULES = {
    "latency_p95_high": {
        "metric": "latency_p95_ms",
        "threshold": 5000,       # Alert if p95 > 5s
        "window": "5m",
        "severity": "warning"
    },
    "error_rate_high": {
        "metric": "error_rate",
        "threshold": 0.05,       # Alert if > 5% errors
        "window": "5m",
        "severity": "critical"
    },
    "cache_hit_rate_low": {
        "metric": "cache_hit_rate",
        "threshold": 0.15,       # Alert if < 15% cache hits
        "window": "1h",
        "severity": "warning"
    },
    "cost_spike": {
        "metric": "daily_cost",
        "threshold": 150.0,      # Alert if > $150/day
        "window": "1d",
        "severity": "warning"
    },
    "zero_queries": {
        "metric": "queries_per_min",
        "threshold": 1,
        "window": "5m",
        "severity": "critical",  # No queries = system down
        "operator": "less_than"
    },
    "faithfulness_drop": {
        "metric": "avg_faithfulness",
        "threshold": 0.80,       # Alert if < 0.80 (sampled)
        "window": "1h",
        "severity": "critical"
    },
}
```

---

### 6. Error Handling & Resilience

#### The Retry Strategy

```python
"""
resilience.py — Retry, circuit breaker, and graceful degradation.
"""

import asyncio
from typing import Callable, Any
from functools import wraps


async def retry_with_exponential_backoff(
    func: Callable,
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 30.0,
    retryable_exceptions: tuple = (TimeoutError, ConnectionError)
) -> Any:
    """Retry with exponential backoff + jitter."""
    last_exception = None

    for attempt in range(max_retries):
        try:
            return await func()
        except retryable_exceptions as e:
            last_exception = e
            if attempt == max_retries - 1:
                raise

            delay = min(base_delay * (2 ** attempt), max_delay)
            import random
            jitter = random.uniform(0, delay * 0.1)
            await asyncio.sleep(delay + jitter)

    raise last_exception


class CircuitBreaker:
    """
    Circuit breaker for external services (LLM API, vector DB).

    States:
    - CLOSED: Normal operation
    - OPEN: Service is failing, fail fast
    - HALF_OPEN: Testing if service has recovered
    """

    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: float = 30.0,
        half_open_max_requests: int = 1
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.half_open_max_requests = half_open_max_requests

        self.state = "CLOSED"
        self.failure_count = 0
        self.last_failure_time = None
        self.half_open_requests = 0

    async def call(self, func: Callable, *args, **kwargs) -> Any:
        """Call a function with circuit breaker protection."""

        if self.state == "OPEN":
            if self._should_recover():
                self.state = "HALF_OPEN"
                self.half_open_requests = 0
            else:
                raise CircuitBreakerOpenError(
                    f"Circuit breaker is OPEN for {self._service_name}"
                )

        if self.state == "HALF_OPEN":
            if self.half_open_requests >= self.half_open_max_requests:
                raise CircuitBreakerOpenError("Half-open max requests reached")

        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        self.state = "CLOSED"
        self.failure_count = 0
        self.half_open_requests = 0

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = asyncio.get_event_loop().time()

        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
            self.half_open_requests = 0

    def _should_recover(self) -> bool:
        if self.last_failure_time is None:
            return True
        elapsed = asyncio.get_event_loop().time() - self.last_failure_time
        return elapsed >= self.recovery_timeout


class CircuitBreakerOpenError(Exception):
    pass
```

#### Graceful Degradation Strategies

| Situation | Degradation Strategy |
|-----------|---------------------|
| LLM API timeout | Fall back to smaller/faster model |
| Vector DB unavailable | Fall back to BM25-only search (local) |
| Embedding API fails | Use pre-computed sparse vectors |
| Document index outdated | Serve with "as of [date]" disclaimer |
| Cache backend down | Skip cache, run full pipeline |
| Rate limited | Queue requests, notify user of delay |

```python
class GracefulDegradationHandler:
    """Handle service failures with graceful degradation."""

    async def handle_llm_failure(
        self, query: str, context: str
    ) -> str:
        """If primary LLM fails, try fallback models."""
        for fallback in ["gpt-4o-mini", "claude-3-haiku"]:
            try:
                return await self.llm_client.complete(
                    prompt=f"{context}\n\nQuestion: {query}",
                    model=fallback,
                    max_tokens=200  # Shorter answer to reduce failure chance
                )
            except Exception:
                continue

        # Last resort: return retrieved chunks directly
        return (
            "I'm unable to generate an answer right now. "
            "Here are the most relevant documents I found:\n\n"
            + "\n\n".join(c.text[:300] for c in self.last_chunks[:3])
        )

    async def handle_retrieval_failure(
        self, query: str
    ) -> tuple:
        """If vector DB fails, try BM25, then return empty."""
        try:
            return await self.bm25_retriever.retrieve(query)
        except Exception:
            self.logger.error("All retrieval backends failed")
            return []  # Will trigger "no documents found" flow
```

---

### 7. The Complete Production RAG Server

Here's how all the pieces fit together:

```python
"""
production_rag_server.py — Complete production RAG FastAPI server.

Integrates:
- Request validation
- Query classification
- Multi-level cache
- Parallel retrieval
- Latency profiling
- Streaming responses
- Cost tracking
- Structured logging
- Graceful degradation
"""

import asyncio
import time
import uuid
from datetime import datetime
from typing import List, Optional
from pydantic import BaseModel, Field

from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from sse_starlette.sse import EventSourceResponse


# ============================================================
# Pydantic Models
# ============================================================

class ChatRequest(BaseModel):
    query: str = Field(..., min_length=1, max_length=2000)
    user_id: Optional[str] = None
    stream: bool = False
    model: Optional[str] = None


class Source(BaseModel):
    id: int
    title: str
    score: float
    preview: str


class ChatResponse(BaseModel):
    answer: str
    sources: List[Source]
    query_id: str
    latency_ms: float


# ============================================================
# RAG Application
# ============================================================

class RAGApplication:
    """
    Production RAG application with all optimizations.
    """

    def __init__(self, config: dict):
        self.config = config

        # Initialize components
        self.retriever = self._init_retriever()
        self.llm_client = self._init_llm()
        self.context_builder = ContextIntegrator(**config.get("context", {}))
        self.cache = self._init_cache()
        self.profiler = LatencyProfiler()
        self.cost_tracker = CostMonitor(budget_daily=config.get("daily_budget", 100))
        self.logger = RAGLogger()
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=5,
            recovery_timeout=30.0
        )

    async def answer(
        self,
        query: str,
        stream: bool = False,
        preferred_model: Optional[str] = None
    ):
        """Main answer method with full production pipeline."""
        query_id = str(uuid.uuid4())
        t0 = time.perf_counter()

        # 1. Check exact cache
        if self.cache:
            cached = await self.cache.get(query)
            if cached:
                return {
                    "answer": cached["answer"],
                    "sources": cached.get("sources", []),
                    "query_id": query_id,
                    "latency_ms": (time.perf_counter() - t0) * 1000,
                    "cached": True
                }

        # 2. Profiled retrieval pipeline
        try:
            result = await self.circuit_breaker.call(
                self._run_pipeline,
                query, preferred_model
            )
        except CircuitBreakerOpenError:
            return await self._handle_circuit_open(query, query_id)

        latency = (time.perf_counter() - t0) * 1000

        # 3. Cache the result
        if self.cache and result.get("answer"):
            asyncio.create_task(
                self.cache.set(
                    query, result["answer"],
                    [s.id for s in result.get("sources", [])]
                )
            )

        # 4. Log and track
        self.logger.log_query(query, result["answer"], RAGMetrics(
            timestamp=datetime.now(),
            query_id=query_id,
            total_latency_ms=latency,
            retrieval_latency_ms=result.get("retrieval_ms", 0),
            generation_latency_ms=result.get("generation_ms", 0),
            ttft_ms=result.get("ttft_ms", 0),
            tokens_per_second=result.get("tokens_per_sec", 0),
            num_chunks_retrieved=len(result.get("sources", [])),
            max_relevance_score=result.get("max_score", 0),
            avg_relevance_score=result.get("avg_score", 0),
            min_relevance_score=result.get("min_score", 0),
            context_token_count=result.get("context_tokens", 0),
            context_truncated=result.get("truncated", False),
            model_used=result.get("model", "unknown"),
            input_tokens=result.get("input_tokens", 0),
            output_tokens=result.get("output_tokens", 0),
            estimated_cost=result.get("cost", 0),
            had_error=result.get("error", False),
            error_type=result.get("error_type"),
        ))

        result.update({
            "query_id": query_id,
            "latency_ms": latency
        })
        return result

    async def _run_pipeline(self, query: str, preferred_model: str) -> dict:
        """Core RAG pipeline with profiling."""
        profile = LatencyProfile()

        # Retrieve with timing
        t_retrieval = time.perf_counter()
        async with self.profiler.profile("retrieval", profile, "retrieval_ms"):
            chunks, retrieval_meta = await self.retriever.retrieve(
                query, top_k=self.config.get("top_k", 5)
            )
        retrieval_ms = (time.perf_counter() - t_retrieval) * 1000

        if not chunks:
            return {
                "answer": self._no_results_message(),
                "sources": [],
                "error": False
            }

        # Build context
        async with self.profiler.profile("context", profile, "context_ms"):
            context = self.context_builder.build_context(chunks, query)

        # Generate answer
        t_gen = time.perf_counter()
        model = preferred_model or self._select_model(query)

        answer = await self.llm_client.complete(
            prompt=context,
            model=model,
            max_tokens=self.config.get("max_tokens", 500)
        )
        generation_ms = (time.perf_counter() - t_gen) * 1000

        # Build response
        sources = [
            Source(
                id=i+1,
                title=c.document_title,
                score=round(c.score, 3),
                preview=c.text[:150]
            )
            for i, c in enumerate(chunks[:5])
        ]

        return {
            "answer": answer,
            "sources": sources,
            "model": model,
            "retrieval_ms": retrieval_ms,
            "generation_ms": generation_ms,
            "latency_profile": profile.__dict__,
            "num_chunks": len(chunks),
            "max_score": max((c.score for c in chunks), default=0),
            "avg_score": sum(c.score for c in chunks) / len(chunks) if chunks else 0,
            "min_score": min((c.score for c in chunks), default=0),
            "context_tokens": len(context.split()),
            "truncated": retrieval_meta.get("truncated", False),
        }

    def _select_model(self, query: str) -> str:
        """Select model based on query complexity."""
        if len(query.split()) < 5:
            return "gpt-4o-mini"  # Simple queries
        if any(kw in query.lower() for kw in ["compare", "why", "explain"]):
            return "gpt-4o"  # Complex queries
        return "gpt-4o-mini"  # Default

    def _no_results_message(self) -> str:
        return (
            "I couldn't find any documents that address your question. "
            "Please rephrase your question or check if the information "
            "you're looking for exists in our knowledge base."
        )


# ============================================================
# FastAPI Server
# ============================================================

app = FastAPI(title="Production RAG API", version="2.0.0")
rag = RAGApplication(config={
    "top_k": 5,
    "max_tokens": 500,
    "daily_budget": 100,
    "context": {"max_context_tokens": 4000}
})


@app.post("/api/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    """Non-streaming RAG endpoint."""
    try:
        result = await rag.answer(
            query=request.query,
            preferred_model=request.model
        )
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


@app.post("/api/chat/stream")
async def chat_stream(request: ChatRequest):
    """Streaming RAG endpoint."""
    async def event_generator():
        try:
            result = await rag.answer(
                query=request.query,
                stream=True,
                preferred_model=request.model
            )
            # Stream the answer token by token
            for token in result["answer"].split():
                yield {"event": "token", "data": token + " "}
                await asyncio.sleep(0.01)

            # Send sources
            yield {"event": "sources", "data": str(result.get("sources", []))}
            yield {"event": "done", "data": result.get("query_id", "")}
        except Exception as e:
            yield {"event": "error", "data": str(e)}

    return EventSourceResponse(event_generator())


@app.get("/api/health")
async def health():
    """Health check endpoint."""
    return {
        "status": "healthy",
        "timestamp": datetime.now().isoformat(),
        "cache_size": await rag.cache.redis.dbsize() if rag.cache else 0,
    }


# ============================================================
# Startup / Shutdown
# ============================================================

@app.on_event("startup")
async def startup():
    """Initialize connections on startup."""
    await rag.retriever.initialize()
    print("RAG server started")


@app.on_event("shutdown")
async def shutdown():
    """Clean up connections on shutdown."""
    await rag.retriever.close()
    print("RAG server stopped")
```

---

## ✅ Good Output Examples

### Good Production Deployment

```
Production RAG System — Operational Summary
============================================

INFRASTRUCTURE:
  Server: FastAPI (4 workers, async)
  Vector DB: Qdrant (self-hosted, 2 nodes)
  Cache: Redis (exact + semantic, 1GB)
  Monitoring: Prometheus + Grafana
  Logging: JSON-structured, shipped to ELK

PERFORMANCE (last 7 days):
  Queries served: 345,000
  p50 latency: 980ms
  p95 latency: 2,300ms
  p99 latency: 4,100ms
  TTFT (streaming): 320ms median

CACHE PERFORMANCE:
  Exact cache hit rate: 34%
  Semantic cache hit rate: 12%
  Total queries saved from full pipeline: 46%
  Estimated cost savings: $842/month

COST:
  Daily average: $38.40
  Cost per query: $0.0032
  Monthly projected: $1,152
  Budget utilization: 38%

QUALITY (sampled, 2% of queries):
  Faithfulness: 0.91 (target: 0.85+)
  Answer relevance: 0.88 (target: 0.80+)
  User helpful rating: 83% (target: 80%+)
  Escalation rate: 2.8% (target: <5%)

ERRORS:
  LLM API errors: 0.08% (rate limited 2x, recovered on retry)
  Retrieval errors: 0.01% (Qdrant primary down 47s, BM25 fallback served)
  Cache errors: 0.00% (Redis degraded once, cache disabled for 3 min)
  0 outages impacting users
```

---

## ❌ Antipatterns & Failure Modes

### 1. Caching Without Invalidation

**What:** Caching everything with long TTLs and no invalidation strategy.

**Why it fails:** Users get stale answers. A document update won't be reflected until cache expires. For time-sensitive content (pricing, policies), this means wrong answers.

**Fix:** Implement context-aware cache invalidation or use shorter TTLs for dynamic content. At minimum, tag cached answers with document versions.

### 2. Streaming Without Citations

**What:** Streaming tokens that include citation markers like "[1]" but the frontend doesn't handle them.

**Why it fails:** Users see raw citation markers in the stream before the backend resolves them. The "citations" look like broken formatting.

**Fix:** Use structured streaming events. Citations are separate events from tokens, sent as metadata. The frontend renders them inline.

### 3. Over-Optimizing for Latency

**What:** Aggressively optimizing for p50 latency at the expense of correctness.

**Example:** Reducing chunk count to 2 to save 200ms on LLM input processing → the answer misses a critical piece of information → user gets wrong answer fast instead of correct answer slower.

**Fix:** Measure both latency AND quality. A 2-second correct answer is better than a 500ms wrong answer. Quality is your constraint; latency is your optimization target within that constraint.

### 4. Single-Point-of-Failure Architecture

**What:** All RAG components are in a single process. If any component fails, the whole system fails.

**Why it fails:** One bad LLM API call times out → user gets no answer (or a 500 error). One vector DB query hangs → all requests wait.

**Fix:** Isolate each component with timeouts, circuit breakers, and fallbacks. A retrieval failure shouldn't crash the generation layer. Each component should have a "last resort" behavior.

### 5. Logging Everything

**What:** Logging full contexts and answers for every query to storage.

**Why it fails:** At 10K queries/day with 4K tokens of context each, you're generating 40M tokens/day in log data = hundreds of GB of storage costs.

**Fix:** Log summaries for every query (latency, scores, errors). Log full details only for sampled queries (1-5%) or queries that triggered errors/alerts.

### 6. Ignoring Cold Starts

**What:** Optimizing for steady-state performance but ignoring cold start latency.

**Why it fails:** The first query of the day (or after a deployment) takes 10 seconds because connections need to be established, caches are empty, and the LLM has warm-up time.

**Fix:** Use health checks with warmup queries. Pre-warm connections on startup. Cache seeding for common queries.

---

## 🧪 Drills & Challenges

### Drill 1: Build a Multi-Level Cache (45 min)

Implement a `RAGCache` class with three levels:

1. **Level 1 (Exact)**: O(1) lookup for identical queries
2. **Level 2 (Semantic)**: Embedding-based lookup for similar queries (threshold: 0.92)
3. **Level 3 (Retrieval-only)**: Cache retrieval results only, regenerate answer

**Requirements:**
- All levels share a single Redis instance (or in-memory dict for testing)
- Cache entries have TTLs (configurable per level)
- Hit/miss statistics per level
- Invalidation by query (for Level 1) and by context (for all levels)
- Fallback: if Level 2 fails, don't crash — try Level 3 or run full pipeline

**Test with:**
```python
cache = RAGCache(redis_client)

# Simulate queries
await cache.set("What's the refund policy?", answer)
await cache.set("What's the return policy?", answer)  # Similar query

# Test exact match
result = await cache.get("What's the refund policy?")
assert result["hit"] and result["level"] == "exact"

# Test semantic match
result = await cache.get("Can I get a refund?")
assert result["hit"] and result["level"] == "semantic"

# Test different query
result = await cache.get("How do I reset my password?")
assert not result["hit"]
```

### Drill 2: The Latency Autopsy (35 min)

You're given this latency profile:

```
Total: 3,450ms
  Query rewrite: 150ms
  Dense retrieval: 320ms
  Sparse retrieval: 280ms
  Fusion + reranking: 410ms
  Context assembly: 80ms
  LLM TTFT: 1,200ms  <-- High!
  LLM generation (250 tokens): 1,010ms
```

The TTFT (Time to First Token) is 1,200ms — the largest single contributor. This means the LLM provider takes over a second to start generating.

**Tasks:**
1. What could cause high TTFT? (List at least 3 potential causes)
2. What diagnostics would you run to pinpoint the cause?
3. Propose 3 different solutions, ordered from cheapest to most expensive
4. If the cause is "model too large for the provider's instance," what's your solution?
5. If the cause is "network latency between your server and the API," what's your solution?
6. If the cause is "the provider does prompt processing during TTFT and your prompt is 5000 tokens," what's your solution?

### Drill 3: Design a Cost Optimization Strategy (40 min)

Your RAG system has these characteristics:
- 50,000 queries/day
- Average input: 3,000 tokens (including system prompt, context, question)
- Average output: 400 tokens
- Current model: GPT-4o for everything
- Current daily cost: $425/day

**Tasks:**
1. Calculate the current cost per query
2. Design a tiered model strategy that cuts costs by 60% (target: $170/day)
3. For each tier, specify:
   - Which queries go to which model?
   - What's the cost per query for each tier?
   - What's the quality risk?
   - How do you detect if the cheaper model is degrading quality?
4. Design a cache strategy that further reduces costs
5. What's your total cost after both optimizations?
6. What's the payback period if implementing these optimizations takes 2 weeks of engineering time?

### Drill 4: Build the Operations Dashboard (45 min)

Create a monitoring dashboard template for RAG (text-based or JSON config):

**Required metrics:**
- Query volume (requests/min)
- Latency distribution (p50, p95, p99)
- Error rate (by error type)
- Cache hit rate (by cache level)
- Cost (by model, hourly/daily)
- Retrieval quality (avg relevance score)
- User feedback rate

**For each metric, define:**
1. How it's computed (formula)
2. Where the data comes from (log, metric, trace)
3. The alert threshold
4. What action to take when alerted

**Bonus:** Design a "Health Score" — a single number that combines multiple metrics into an at-a-glance system health indicator.

### Drill 5: Implement Graceful Degradation (30 min)

Extend the RAG server to handle these failure scenarios gracefully:

1. **LLM API returns 429 (rate limited)**: Retry with exponential backoff, then fall back to a smaller model
2. **Vector DB connection timeout**: Fall back to BM25-only retrieval
3. **Both LLM and fallback fail**: Return the top 3 retrieved chunks as-is (no generation)
4. **Cache backend is down**: Skip cache, log warning, run full pipeline

**For each scenario:**
- Log what happened
- Add a header to the response indicating degradation (e.g., `X-RAG-Degraded: llm-fallback`)
- Track degradation metrics separately

### Drill 6: The Cache Invalidation Puzzle (30 min)

Your RAG system has been running for 3 months with a 24-hour TTL cache. Today, you updated 3 critical pricing documents. Users are still getting old pricing from cached responses.

**Tasks:**
1. Design a system that invalidates cache entries when source documents change
2. Implement the "context index" — for each cached answer, track which document chunks were used
3. Implement the invalidation endpoint: `POST /api/cache/invalidate` that accepts a document ID
4. Test: After invalidation, verify the next query for that topic gets a fresh answer
5. What happens during the brief period between document update and cache invalidation? How do you minimize this gap?

---

## 🚦 Gate Check

Before proceeding to Phase 4.08 (Drills), you must demonstrate:

1. **Benchmark your RAG latency**: Profile your complete RAG pipeline and produce a latency breakdown. Identify the top 3 bottlenecks and propose specific optimizations with expected impact.

2. **Implement caching**: Build and test a multi-level cache (exact + semantic) that measurably reduces latency. Measure: cache hit rate, latency improvement, cost savings on a test set of 100+ queries.

3. **Set up structured logging**: Add structured logging to your RAG server that outputs JSON-formatted logs for every query. Each log should include: query ID, latency breakdown, model used, token counts, cost, error status.

4. **Implement a fallback chain**: Demonstrate that your system handles at least 3 failure scenarios gracefully:
   - LLM API timeout → falls back to smaller model
   - Vector DB unavailable → falls back to BM25
   - All backends fail → returns meaningful "unavailable" message

5. **Answer the reflection questions**:
   - What's your SLO for p95 latency and how did you determine it?
   - What's the most expensive component in your pipeline and how would you optimize it?
   - How do you balance cache freshness vs. cache hit rate?
   - What's the first thing you'd add to monitoring if you saw user satisfaction dropping?

**Passing criteria:** Your RAG system is ready for production — it's observable, cost-controlled, degrades gracefully, and you know where every millisecond and cent goes.

---

## 📚 Resources

### Production Systems & Case Studies
- **Glean's RAG Infrastructure** — Blog posts on how they handle caching, latency, and cost at enterprise scale.
- **Notion AI's Architecture** — How Notion shipped AI features with production reliability.
- **Quora's Poe Platform** — Multi-model routing and caching at scale.

### Tools
- **Prometheus + Grafana** — Standard monitoring stack. Learn the basics of metric collection and dashboard building.
- **Redis** — Used for caching in virtually every production RAG system. Learn memory management and eviction policies.
- **sse-starlette** — SSE (Server-Sent Events) library for FastAPI streaming.
- **Langfuse** — Open-source observability for LLM apps. Has built-in RAG tracing.

### Infrastructure Patterns
- **vLLM** — For self-hosted LLM serving with prefix caching (dramatically reduces TTFT).
- **Qdrant on Kubernetes** — For production vector DB deployment with auto-scaling.
- **Cloudflare Workers / Vercel Edge** — For edge caching of common RAG queries.

### Your Own Code
- Apply everything in this module to your **Customer Support Bot** (the Phase 4 project). It should have: caching, streaming, monitoring logging, cost tracking, and graceful degradation.
- Your **Knowledge Search Engine (Phase 3)** could be wrapped with a production API — the caching and monitoring patterns apply equally there.

### Expert-Level Deep Dives
- **For production engineers**: The most impactful optimization is often not technical — it's reducing the number of queries that hit the LLM at all. Every percentage point of cache hit rate improvement saves real money and latency. Invest in cache infrastructure before model optimization.
- **For infrastructure engineers**: Connection pooling and keep-alive are the most overlooked optimizations. A single HTTP connection reuse can save 200-500ms per query. Profile your network overhead before your model serving.
- **For critical thinkers**: Production RAG is primarily a COST and LATENCY optimization problem, not a QUALITY optimization problem. In development, you optimize for quality. In production, you optimize for cost/latency while maintaining quality. The skill is knowing where to cut without breaking the user experience.

---

*"A RAG system in production is defined not by its best-case performance, but by its worst-case behavior. Cache misses, cold starts, API failures — these are the moments that define your user's experience. Design for the failure modes, and the happy path takes care of itself."*
