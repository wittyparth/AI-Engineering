# Phase 5: Build an Advanced Knowledge System

**The Problem**: Simple RAG fails on complex questions. "Compare our Q4 revenue to last year" requires multi-hop reasoning across documents. You need a system that chooses the right retrieval strategy per query.

**What You'll Build**: An agentic research assistant that uses HyDE, multi-query, self-RAG, Graph RAG, and LlamaIndex Workflows to answer questions that simple RAG can't handle.

**Difficulty**: Hard | **Time**: ~15 hours

## The Build Path

```
1. PROBLEM: "Simple RAG fails on multi-hop, comparative, and analytical queries"
2. BUILD RAW: HyDE, multi-query, self-RAG — build each pattern from scratch
3. ADD FRAMEWORK: LlamaIndex Workflows — multi-hop retrieval, agentic RAG
4. BUILD RAW: Graph RAG — knowledge graphs for complex reasoning
5. PRODUCTIONIZE: Semantic caching, context window management
6. EVALUATE: Multi-hop accuracy, retrieval precision, cache hit rate
7. DEBUG: When RAG fails — systematic failure analysis
```

## Key Concepts

- HyDE (Hypothetical Document Embeddings) — retrieve by what the answer WOULD look like
- Multi-query — ask the same question 5 ways, retrieve more comprehensively
- Self-RAG / Corrective RAG — system evaluates its own retrieval and retries
- Graph RAG — knowledge graphs for multi-hop questions
- Agentic RAG — AI decides WHEN and HOW to retrieve
- Semantic caching — cache by meaning, not exact match
- Context window management — dynamic packing, sliding windows

## Framework Integration

- **LlamaIndex Workflows** — Multi-hop retrieval, query decomposition, and agentic RAG workflows using LlamaIndex's workflow engine

## The Project: Agentic Research Assistant

A system that:
- Breaks complex questions into sub-queries
- Chooses retrieval strategy per sub-query (vector, web, code, SQL)
- Evaluates retrieved info before generating (self-RAG)
- Iterates if first retrieval wasn't sufficient
- Produces a structured research report with citations
- Caches semantically similar queries (40%+ cost reduction)

## 🔴 Senior: RAG Strategy Selection

Senior engineers don't use one RAG strategy. They build systems that SELECT the right strategy per query. This phase teaches you to think in strategies, not just implementations.

## Gate Check

- [ ] Agentic RAG outperforms vanilla RAG on complex multi-part questions
- [ ] Graph RAG improves accuracy for questions needing multi-hop reasoning
- [ ] Semantic caching reduces costs by 40%+ on repeated query patterns
- [ ] Self-RAG catches and corrects at least 3 retrieval failures
- [ ] LlamaIndex Workflow handles 5+ step reasoning chains
