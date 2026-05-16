# Cornerstone Project: Production RAG System

**Time**: 4-5 days  **Difficulty**: Hard  **Portfolio**: Yes

## The Problem

Build a production-grade RAG system over a substantial document corpus (100+ documents, 50K+ tokens). This is the project that will demonstrate you understand the full RAG stack.

## Requirements

### Must Have
- [ ] FastAPI service with streaming RAG endpoint
- [ ] Multi-strategy chunking (3+ strategies, configurable)
- [ ] Hybrid search (BM25 + vector with RRF)
- [ ] Cross-encoder reranking
- [ ] Query rewriting
- [ ] Document ingestion pipeline with Airflow or Celery
- [ ] RAGAS evaluation pipeline (faithfulness, relevancy, precision)
- [ ] Streaming response (SSE)
- [ ] Docker Compose setup (API, vector DB, worker, LLM)

### Should Have
- [ ] PDF, Markdown, HTML ingestion support
- [ ] Metadata filtering (by date, source, category)
- [ ] Caching (Redis: exact + semantic cache)
- [ ] Langfuse tracing (every step logged)
- [ ] Cost tracking per query
- [ ] Feedback collection (thumbs up/down on answers)

### Nice to Have
- [ ] Admin dashboard (query analytics, top-failed queries)
- [ ] A/B testing between RAG configurations
- [ ] Automated eval on every ingestion
- [ ] Guardrails (out-of-domain detection)
- [ ] Multi-modal support (images in documents)

## Architecture

```
                     ┌──────────────────────────────┐
                     │     Docker Compose            │
                     │  ┌──────────┐ ┌──────────┐   │
                     │  │  FastAPI  │ │  Airflow  │   │
                     │  │  Service  │ │  Worker   │   │
                     │  └────┬─────┘ └────┬─────┘   │
                     │       │            │         │
                     │  ┌────▼────────────▼─────┐   │
                     │  │      PostgreSQL        │   │
                     │  │   (metadata + pgvector)│   │
                     │  └───────────────────────┘   │
                     │  ┌──────────┐ ┌──────────┐   │
                     │  │  Redis   │ │  Qdrant   │   │
                     │  │ (cache)  │ │ (vectors) │   │
                     │  └──────────┘ └──────────┘   │
                     │  ┌──────────┐                 │
                     │  │ Ollama   │                 │
                     │  │ (local)  │                 │
                     │  └──────────┘                 │
                     └──────────────────────────────┘
```

## Corpus Options

1. **Your own project documentation** (real, meaningful, you know the content)
2. **FastAPI + Starlette docs** (excellent docs, know them cold afterwards)
3. **arXiv papers on RAG** (you'll become a RAG expert)
4. **Docker + AWS docs** (leverages your existing expertise)
5. **A book (public domain)** like "Clean Code" or "Designing Data-Intensive Applications"

## Evaluation Thresholds

Your system must achieve:
- Faithfulness: >0.85
- Answer relevancy: >0.90
- Context precision: >0.80
- End-to-end latency: <3s p95
- Cost: <$0.05 per query (using GPT-4o-mini or local model)

## Design Decisions

1. **Chunking strategy**: Which one? Why? (This is the most important decision)
2. **Top-K**: How many chunks? (Too few = missing info, too many = context overflow)
3. **Prompt template**: Single-stage or multi-stage? (Direct answer vs. "think step by step")
4. **Reranking**: Always or conditional? (Every query or only when ambiguous?)
5. **Streaming**: Token-level or chunk-level? (UX vs. complexity tradeoff)
6. **LLM choice**: Local, API, or hybrid? (Cost vs. quality vs. latency)
7. **Caching policy**: TTL-based? Similarity-based? Both?
8. **Scaling strategy**: How does this handle 100 concurrent users?

## Code Review Rubric

| Criteria | Pass | Fail |
|----------|------|------|
| Chunking strategy | Justified with data | Random choice |
| Retrieval quality | Recall@5 > 0.85 | < 0.70 |
| Generation quality | Faithfulness > 0.85 | < 0.75 |
| Streaming | Works end-to-end | Doesn't stream or breaks |
| Error handling | Graceful on all failure modes | Silently fails |
| Docker | Single compose up, all services start | Manual steps needed |
| Tests | Unit + integration + eval | No tests |
| Documentation | README explains architecture + setup | Can't run without asking |

## Debug Challenge

Given a broken version of your RAG system:
- Chunking produces overlapping inconsistent chunks
- Reranker scores are inverted (lowest score returned first)
- Streaming drops every 3rd chunk
- Evals report 0.99 faithfulness but answers are wrong (eval bug)

Find and fix all 4 bugs.

## Reflection

1. What was the hardest part of getting RAG to work well?
2. What surprised you about retrieval quality?
3. If you were to rebuild from scratch, what would you do differently?
4. How would you scale this to 1M documents?
5. What monitoring would you add in production?
