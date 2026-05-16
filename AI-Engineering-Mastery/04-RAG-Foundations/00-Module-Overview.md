# Phase 4: Build a Production RAG Engine

**The Problem**: LLMs don't know YOUR data. They hallucinate answers about your product, your documentation, your internal knowledge. You need to ground LLM responses in real data — reliably, at scale.

**What You'll Build**: A production-grade RAG engine implemented 3 ways — from scratch, with LangChain, and with LlamaIndex — so you understand EVERY layer. This is the single most in-demand AI engineering skill in 2026.

**Difficulty**: Medium-Hard | **Time**: ~15 hours

## The Build Path

```
1. PROBLEM: "LLMs hallucinate because they don't know my data"
2. BUILD RAW: RAG from scratch — chunking, embedding, retrieval, generation
3. PAIN EMERGES: Boilerplate, edge cases, missing features (reranking, query transform)
4. ADD FRAMEWORK: LangChain (ecosystem breadth, fast prototyping)
5. ADD FRAMEWORK: LlamaIndex (data-centric RAG, 100+ connectors)
6. FRAMEWORK COMPARISON: Same RAG task, 3 implementations — raw vs LangChain vs LlamaIndex
7. PRODUCTIONIZE: Reranking, hybrid search, caching, streaming
8. EVALUATE: RAGAS — faithfulness, answer relevancy, context precision
9. DEBUG: 5 common RAG failure modes — find and fix them
```

## Concepts (Built Into the Flow)

| # | Step | What You Learn | Why It Matters |
|---|------|---------------|----------------|
| 1 | Problem | RAG architecture, the retrieval-generation gap | Foundation of production AI |
| 2 | Build Raw | Chunking strategies, document ingestion | Single most impactful RAG parameter |
| 3 | Build Raw | Embedding, retrieval, generation | Understand every component |
| 4 | Add Framework | LangChain RAG | 1000+ integrations, pre-built chains |
| 5 | Add Framework | LlamaIndex RAG | Data-centric, 100+ data connectors |
| 6 | Decision | Framework comparison: same task, 3 implementations | KNOW when to use what |
| 7 | Productionize | Reranking, hybrid search, query transformation | The difference between demo and prod |
| 8 | Evaluate | RAGAS metrics | If you can't measure it, it's not done |
| 9 | Debug | Common failure modes | Retrieval misses, bad chunks, hallucination |

## Frameworks Integrated Here

- **LangChain** — Rapid prototyping, pre-built chains, 70+ LLM providers, 1000+ integrations
- **LlamaIndex** — Data connectors (100+), advanced indexing, query engines

**Why both?** LangChain excels at prototyping speed and ecosystem breadth. LlamaIndex excels at data ingestion and advanced RAG patterns. Senior engineers know BOTH and choose based on the problem.

## The Projects

### Project 1: Framework Comparison (Learning)
Build the SAME RAG system 3 ways — raw Python, LangChain, LlamaIndex. Compare code complexity, flexibility, performance, and debuggability. This is how you build the intelligence to choose the right tool for any situation.

### Project 2: Production RAG System (Portfolio)
A FastAPI service with:
- Multi-strategy chunking (configurable)
- Hybrid search (semantic + keyword)
- Reranking via cross-encoder
- Query rewriting (multi-query, HyDE)
- RAGAS evaluation pipeline (CI gate)
- Streaming responses
- Cost tracking per query

**This is the system companies ask you to build in AI engineer interviews.**

## 🔴 Senior: RAG is NOT Search

Maximize throughput with continuous batching, manage KV cache efficiently, implement quantization (FP8, AWQ) to fit larger models on fewer GPUs, and benchmark P50/P95/P99 latency under load.

The retrieval quality defines an UPPER bound on generation quality. Bad retrieval = bad generation, even with GPT-5. Most RAG failures trace back to chunks, not the LLM.

## Gate Check

- [ ] Built RAG from scratch (no frameworks)
- [ ] Can explain how chunk size affects retrieval quality
- [ ] Reranking improves recall@5 by at least 10%
- [ ] RAGAS faithfulness score > 0.8 on test set
- [ ] Framework comparison completed — can articulate when to use each
- [ ] Streaming generation works end-to-end
- [ ] Debug challenge: identified all 5 RAG failure modes
