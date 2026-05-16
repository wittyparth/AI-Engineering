# Cornerstone Project: Semantic Search Engine

**Time**: 3-4 days  **Difficulty**: Medium-Hard  **Portfolio**: Yes

## The Problem

You have 10,000+ documents (internal docs, code comments, or research papers). Users need to find the right information. Full-text search misses synonyms. Pure vector search misses exact matches. Build a production hybrid search engine.

## Requirements

### Must Have
- [ ] FastAPI service with `/search` and `/index` endpoints
- [ ] Document ingestion pipeline (PDF, Markdown, plain text)
- [ ] Multiple embedding model support (selectable per request)
- [ ] Hybrid search (BM25 + vector with RRF fusion)
- [ ] Configurable chunking strategy
- [ ] Metadata filtering (date, author, category, tags)
- [ ] Performance benchmark mode (recall@k, latency, throughput)

### Should Have
- [ ] Qdrant (or pgvector) as vector storage
- [ ] Batch indexing (handle 10K+ documents)
- [ ] Incremental indexing (add docs without re-indexing all)
- [ ] Search analytics dashboard (popular queries, zero-result queries)
- [ ] Caching for frequent queries

### Nice to Have
- [ ] Relevance tuning UI (adjust BM25 vs vector weighting)
- [ ] Custom reranking model (Cohere Rerank or cross-encoder)
- [ ] A/B test different embedding models in production
- [ ] Search suggestions / auto-complete

## Architecture

```
Documents → Parser → Chunker → Embedder → Vector DB
                               ↘ BM25 Index → Search Engine
                                               ↓
User Query → Query Processor → Hybrid Search → RRF → Ranked Results
```

## Corpus Options

Pick one that interests you:
1. **Technical docs** — FastAPI, Docker, or LangChain documentation
2. **Research papers** — arXiv papers on AI/ML (100+ papers)
3. **Code repository** — Your own projects with README + docstrings
4. **Legal/medical documents** — Public domain texts
5. **Internal wiki** — Export from Notion/Confluence

## 🔴 Senior: Production Search Pipeline

```python
class SearchPipeline:
    def __init__(self, bm25_weight: float = 0.3, vector_weight: float = 0.7):
        self.bm25_weight = bm25_weight
        self.vector_weight = vector_weight

    async def search(self, query: str, filters: dict, k: int = 10):
        # 1. Query expansion (optional)
        expanded = await self.expand_query(query)

        # 2. Parallel search
        bm25_results, vector_results = await asyncio.gather(
            self.bm25_search(expanded, filters, k * 2),
            self.vector_search(expanded, filters, k * 2),
        )

        # 3. RRF fusion
        fused = rrf([bm25_results, vector_results])

        # 4. Rerank top results
        reranked = await self.reranker.rerank(query, fused[:k])

        # 5. Apply diversity filter
        diverse = self.diversify(reranked)

        return diverse
```

## Design Decisions

1. **Chunking strategy**: Fixed size? Semantic? Recursive? (Affects everything downstream)
2. **Embedding refresh**: When do you re-embed documents? (Versioning matters)
3. **Filter-first vs search-then-filter**: Which order for metadata filtering?
4. **Caching strategy**: What constitutes a "repeat query"? Similarity threshold?
5. **Scoring**: Normalize BM25 scores before fusion? How?

## Code Review Rubric

- Search quality: recall@10 > 0.85 on test set
- Latency: p50 < 200ms, p99 < 1s for 100K documents
- Ingestion: 100+ docs/min throughput
- Hybrid outperforms pure BM25 and pure vector
- Metadata filtering works correctly
- Graceful handling of empty queries, missing embeddings
- Tests cover search, indexing, and edge cases

## Reflection

1. How did chunking strategy affect search quality?
2. What was the optimal BM25/vector weight for your corpus?
3. Where did search fail? What kind of queries?
4. How would this change at 10M documents vs 10K?
5. What monitoring would you add in production?
