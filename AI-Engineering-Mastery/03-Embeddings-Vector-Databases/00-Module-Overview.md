# Phase 3: Build a Semantic Search Engine

**The Problem**: Keyword search doesn't understand meaning. "Expensive laptop" returns nothing if the description says "high-end notebook." You need search that understands semantics, not just strings.

**What You'll Build**: A production-grade semantic search engine — the kind of system that powers search at companies like Uber, Notion, and Intercom.

**Difficulty**: Medium-Hard | **Time**: ~10 hours

## The Build Path

```
1. PROBLEM: "Keyword search fails for meaning-based queries"
2. BUILD RAW: Word embeddings from scratch (co-occurrence matrix + SVD)
3. BUILD RAW: Vector search from scratch (brute force → IVF)
4. ADD FRAMEWORK: FAISS / Qdrant — production vector databases
5. PRODUCTIONIZE: HNSW index, hybrid search (semantic + keyword)
6. EVALUATE: Recall@K, precision@K, MRR on labeled test set
7. DEBUG: Why did this search return irrelevant results? (5 failure modes)
```

## Key Concepts

- Embeddings from scratch (skip-gram, CBOW, sentence embeddings)
- Vector search algorithms (brute force → IVF → HNSW)
- Production vector databases (Qdrant, pgvector)
- Hybrid search (combine semantic + keyword relevance)
- Evaluation of search quality

## The Project: Semantic Search Engine

A FastAPI service with:
- Multiple embedding models (sentence-transformers, OpenAI, Cohere)
- HNSW index for fast approximate search
- Hybrid search (semantic + keyword via BM25)
- Search quality dashboard (recall@K, precision@K, latency P95)
- A/B comparison of embedding models

## 🔴 Senior: Search Quality is Measured, Not Guessed

Most RAG failures trace back to bad retrieval. Before you build RAG in Phase 4, master retrieval in this phase. If your search quality is bad, your RAG quality will be worse.

## Gate Check

- [ ] Built vector search from scratch (no FAISS)
- [ ] Understand HNSW graph construction
- [ ] Hybrid search outperforms pure semantic on your test set
- [ ] Production Qdrant deployment handles 1M+ vectors
- [ ] Search quality dashboard shows recall@K and latency P95
