# RAG Architecture

## The Full Pipeline

```
                    ┌─────────────────────┐
                    │   Document Source    │
                    │ (PDF, HTML, Notion)  │
                    └─────────┬───────────┘
                              │
                    ┌─────────▼───────────┐
                    │   Document Parser   │
                    │ (extract text + MD) │
                    └─────────┬───────────┘
                              │
                    ┌─────────▼───────────┐
                    │     Chunker         │
                    │ (split into pieces) │
                    └─────────┬───────────┘
                              │
               ┌──────────────┼──────────────┐
               ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Embedder │  │  BM25    │  │Metadata  │
        │ (vector) │  │ (lexical)│  │ Extractor│
        └────┬─────┘  └────┬─────┘  └────┬─────┘
             │             │             │
             ▼             ▼             ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Vector   │  │ Inverted │  │Metadata  │
        │ Database │  │ Index    │  │ Store    │
        └──────────┘  └──────────┘  └──────────┘
                              │
                    ┌─────────▼───────────┐
                    │   Query Processor   │
                    │  (rewrite, expand)  │
                    └─────────┬───────────┘
                              │
               ┌──────────────┼──────────────┐
               ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ BM25     │  │ Vector   │  │ Metadata │
        │ Search   │  │ Search   │  │ Filter   │
        └────┬─────┘  └────┬─────┘  └────┬─────┘
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                    ┌──────────┐
                    │  RRF     │
                    │ (fusion) │
                    └────┬─────┘
                         ▼
                    ┌──────────┐
                    │ Reranker │
                    └────┬─────┘
                         ▼
                    ┌──────────┐
                    │  Prompt  │
                    │ Builder  │
                    └────┬─────┘
                         ▼
                    ┌──────────┐
                    │   LLM    │
                    └──────────┘
```

## 🔴 Senior: The RAG Tax

| Component | Latency | Cost per query |
|-----------|---------|---------------|
| Query embedding | ~100ms | $0.00001 |
| Vector search | ~10ms | ~$0 |
| BM25 search | ~5ms | ~$0 |
| Reranking | ~200ms | $0.0001 |
| LLM generation | ~2-10s | $0.01-0.10 |

Total: ~2.5-10s per query, $0.01-0.10 per query.

RAG is NOT free. Every component adds latency and cost. Optimize where it matters.

## 🔧 Framework Integration: When to Use LangChain vs LlamaIndex

You'll build RAG from scratch first (Phase 4 drill). Once you understand every component, you add frameworks. Here's the decision framework:

| Need | LangChain | LlamaIndex |
|------|-----------|------------|
| Rapid prototyping | ✅ Best choice | ⚠️ Good |
| 100+ data connectors | ⚠️ Good (1000+ integrations total) | ✅ Best choice (100+ connectors) |
| Pre-built RAG chains | ✅ Best choice | ⚠️ Good (query engines) |
| Custom retrieval logic | ⚠️ Chain abstraction can fight you | ✅ Best choice (composable) |
| Multi-modal RAG | ⚠️ Limited | ✅ Best choice |
| Agentic RAG | ✅ LangGraph | ✅ Workflow engine |
| Learning curve | Steep (many abstractions) | Moderate |
| Production maturity | ✅ Klarna, Uber, Vodafone | ⚠️ Growing |

**Deep dives**: See `08-RAG-with-LangChain-LlamaIndex.md` for code comparison, and `Project-Cornerstone-Comparison-RAG-Framework-vs-Raw.md` for the side-by-side build.
