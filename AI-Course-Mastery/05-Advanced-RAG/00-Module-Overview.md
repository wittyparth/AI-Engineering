# Phase 5: Advanced RAG — Beyond Naive Retrieval

## 🎯 Phase Purpose

Phase 4 taught you to build RAG. You learned chunking, retrieval, context integration, evaluation, and production deployment. You built a Customer Support Bot that answers questions from your documents.

But if you actually ran that bot in production at scale, you'd discover its limits:

- Your retrieval fails when the query uses DIFFERENT WORDS than the documents
- Your system can't answer questions that require connecting information across MULTIPLE documents
- Your bot gives the SAME answer for "What's the API limit?" whether the user is on the free tier or enterprise
- Your system can't handle questions about IMAGES or DIAGRAMS in your documentation
- Your retrieval doesn't understand ENTITIES and RELATIONSHIPS — it treats all text as bags of vectors

Phase 5 fixes ALL of these. This is where we move from "RAG that works" to "RAG that works for hard problems."

---

### ⏹ STOP. Three scenarios. Think before reading on.

**Scenario 1 — The Vocabulary Mismatch**

*Your company's internal document says: "The depreciation schedule for capital assets follows MACRS methodology with a 7-year recovery period." A user asks: "How long until I can write off that new server I bought?"*

*Your keyword search won't match — "write off" ≠ "depreciation", "server" ≠ "capital asset", "how long" ≠ "7-year recovery period." Your dense embeddings might connect some of these, but probably not all. The query is about the same concept as the document, but uses completely different vocabulary.*

*How can you bridge this gap? What if you could turn the user's question into a document-like representation before searching? What if you could know, before searching, whether the question is asking about a concept your documents cover?*

**Scenario 2 — The Multi-Hop Problem**

*Your company has two documents:*

- *Doc A: "Employees with manager-level or above are eligible for the executive benefits package."*
- *Doc B: "The executive benefits package includes supplemental life insurance, deferred compensation, and an annual executive health screening."*

*A user asks: "I'm a senior manager — do I get an executive health screening?"*

*This requires TWO retrieval hops:*
1. *Find Doc A → establishes that senior manager qualifies as "manager-level" → identifies "executive benefits package"*
2. *Find Doc B → establishes that "executive health screening" is part of the package*

*Standard RAG retrieves top-K chunks. It might get both docs, or it might get only one. If it only gets Doc A, the LLM can see "executive benefits package" but not what's IN it. If it only gets Doc B, the LLM can see the screening exists but not whether the user qualifies.*

*How would you design a retrieval system that can perform this multi-hop reasoning? What if the chain requires 3, 4, or 5 hops?*

**Scenario 3 — The Relationship Blindness**

*Your documents mention:*
- *"Project Alpha was led by Dr. Sarah Chen"*
- *"Dr. Sarah Chen is the Director of AI Research"*
- *"The AI Research team published 15 papers in 2024"*
- *"Papers by the AI Research team focus on NLP and computer vision"*

*A user asks: "What research areas does Project Alpha's leader focus on?"*

*This requires the system to understand: Project Alpha → led by → Dr. Sarah Chen → Director of → AI Research → focuses on → NLP and computer vision.*

*A standard vector search sees 4 disconnected chunks. It doesn't understand the entity relationship graph. With dense retrieval, you MIGHT retrieve all 4 chunks if the embedding similarity is high enough, but you have no GUARANTEE that the relationship chain is intact.*

*How would you build a RAG system that explicitly models and traverses entity relationships? What does retrieval look like when it's guided by a knowledge graph rather than vector similarity?*

---

**By the end of this phase, you will:**

- Implement **HyDE (Hypothetical Document Embeddings)** to bridge vocabulary mismatches
- Build **contextual retrieval** that enriches chunks with surrounding context
- Master **reranking** with cross-encoders to dramatically improve precision
- Design **agentic RAG** systems that perform multi-hop retrieval reasoning
- Build **multimodal RAG** that retrieves and processes images, diagrams, and text
- Implement **GraphRAG** that explicitly models entity relationships
- Evaluate systems with advanced metrics that catch subtle errors
- Ship an **Advanced RAG Engine** as your portfolio project

---

## 🧠 The Advanced RAG Spectrum

RAG techniques exist on a spectrum from "simple improvement" to "fundamentally different architecture":

```
Simple improvements ────────────────────────────────────► Major architecture shifts

   HyDE         Contextual        Reranking         Agentic          Multimodal        GraphRAG
 (rewrite       (enrich          (precision        (multi-hop        (vision +        (knowledge
  query →        chunks           boost)            retrieval)        text)             graphs)
  embed)         with context)
  
   Low complexity ────────────────────────────────────► High complexity
   Low latency cost ───────────────────────────────────► Higher latency
   Incremental change ────────────────────────────────► Architectural change
```

**The key insight:** You don't need ALL of these for every use case. You need to know which technique solves WHICH problem and combine them strategically.

---

## 🔗 Connecting to Phase 4

Phase 4 gave you a solid RAG foundation. Phase 5 addresses each of its limitations:

| Phase 4 Limitation | Phase 5 Solution | File |
|-------------------|-------------------|------|
| Query-document vocabulary mismatch | HyDE: generate hypothetical document from query, embed that instead | 01 |
| Chunks lose surrounding context | Contextual retrieval: prepend chunk context | 02 |
| Naive retrieval ranks poorly | Cross-encoder reranking, listwise ranking | 03 |
| Single-shot retrieval misses multi-hop info | Agentic RAG: self-query, multi-hop, iterative | 04 |
| Text-only RAG can't handle images | Multimodal RAG: vision + text retrieval | 05 |
| No entity/relationship understanding | GraphRAG: knowledge graphs for retrieval | 06 |
| Basic eval metrics miss subtle failures | Advanced evaluation: ARES, RGB, human eval | 07 |

---

## 📖 Phase Map

```
PHASE 5: ADVANCED RAG
═══════════════════════════════════════════════════════════════════════

Foundations (Phase 4): chunk → embed → retrieve → augment → generate → evaluate

Advanced RAG (Phase 5):
                    ┌──────────────────────────────────────────┐
                    │          QUERY TRANSFORMATION             │
                    │  ┌────────┐  ┌──────────┐  ┌──────────┐ │
                    │  │  HyDE  │  │  Step-   │  │  Multi-  │ │
                    │  │        │  │  Back    │  │  Query   │ │
                    │  └────────┘  └──────────┘  └──────────┘ │
                    └──────────────────────────────────────────┘
                                    │
                    ┌───────────────▼──────────────────────────┐
                    │         ENHANCED RETRIEVAL                │
                    │  ┌──────────┐  ┌──────────┐  ┌────────┐ │
                    │  │Contextual│  │  Cross-  │  │  Graph │ │
                    │  │Retrieval │  │ Encoder  │  │Traverse│ │
                    │  └──────────┘  └──────────┘  └────────┘ │
                    └──────────────────────────────────────────┘
                                    │
                    ┌───────────────▼──────────────────────────┐
                    │        AGENTIC ORCHESTRATION              │
                    │  ┌──────────┐  ┌──────────┐  ┌────────┐ │
                    │  │  Self-   │  │  Multi-  │  │Iterative│ │
                    │  │  Query   │  │  Hop     │  │  RAG   │ │
                    │  └──────────┘  └──────────┘  └────────┘ │
                    └──────────────────────────────────────────┘
                                    │
                    ┌───────────────▼──────────────────────────┐
                    │         MULTIMODAL EXTENSION              │
                    │  ┌──────────┐  ┌──────────┐  ┌────────┐ │
                    │  │  Multi-  │  │  Vision  │  │Diagram │ │
                    │  │  modal   │  │  RAG     │  │ Parsing│ │
                    │  │ Embedding│  │          │  │        │ │
                    │  └──────────┘  └──────────┘  └────────┘ │
                    └──────────────────────────────────────────┘
                                    │
                    ┌───────────────▼──────────────────────────┐
                    │         ADVANCED EVALUATION               │
                    │  ┌──────────┐  ┌──────────┐  ┌────────┐ │
                    │  │  RAGAS   │  │  ARES    │  │  Human │ │
                    │  │  Advanced│  │          │  │  Eval  │ │
                    │  └──────────┘  └──────────┘  └────────┘ │
                    └──────────────────────────────────────────┘

FILE MAP:
┌─────────────────────────────────────────────────────────────────────┐
│ 00: Module Overview          ← YOU ARE HERE                          │
│                                                                      │
│ 01: HyDE & Query Transform   → Hypothetical docs, step-back, rewrite │
│ 02: Contextual Retrieval     → Enriched chunks, window retrieval     │
│ 03: Reranking Deep Dive      → Cross-encoders, listwise, colBERT     │
│ 04: Agentic RAG              → Self-query, multi-hop, iterative      │
│ 05: Multimodal RAG           → Images + text, vision LLMs            │
│ 06: GraphRAG                 → Knowledge graphs, entity extraction   │
│ 07: Advanced Evaluation      → RAGAS advanced, ARES, RGB, human eval │
│ 08: Advanced Drills          → Multi-hop, cross-modal, GraphRAG fix  │
│ 09: Project                  → Advanced RAG Engine portfolio piece   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📋 File-by-File Breakdown

| # | File | Core Question It Answers | Time |
|---|------|-------------------------|------|
| **00** | Module Overview | What's beyond basic RAG? What's the map? | — |
| **01** | HyDE & Query Transformation | How do I retrieve when the user's words don't match the docs' words? | 3h |
| **02** | Contextual Retrieval | How do I stop chunks from losing their surrounding meaning? | 2.5h |
| **03** | Reranking Deep Dive | How do I prioritize the BEST chunks from a good but noisy set? | 3h |
| **04** | Agentic RAG | How do I answer questions that need 3, 5, or 10 pieces of evidence? | 4h |
| **05** | Multimodal RAG | How do I retrieve and process images, diagrams, and screenshots? | 3.5h |
| **06** | GraphRAG | How do I retrieve based on entity relationships, not just text similarity? | 4h |
| **07** | Advanced Evaluation | How do I catch subtle errors that basic metrics miss? | 2.5h |
| **08** | Advanced Drills | 8 hard challenges combining all techniques | 4h |
| **09** | Project | Advanced RAG Engine — multi-strategy, self-improving | 10-12h |

**⏱ Phase Budget:** 20-25 days (2-3 hours/day)
**Portfolio Project:** Advanced RAG Engine with multi-strategy retrieval
**Tech Stack Additions:** Cross-encoders, Knowledge Graphs (NetworkX/Neo4j), Multi-modal embeddings, LangGraph (for agentic RAG)

---

## 🛑 The Three Big Questions — Revisit After Each File

1. **Which queries need WHICH advanced technique?**  
   *Not every query needs HyDE. Not every query needs multi-hop. How do you detect complexity and route to the right strategy?*

2. **What's the cost-quality-latency tradeoff for EACH advanced technique?**  
   *HyDE adds ~500ms and ~$0.001/query. Reranking adds ~200ms and ~$0.002/query. Multi-hop adds seconds and variable cost. When is each worth it?*

3. **How do these techniques COMPOSE?**  
   *HyDE + Reranking + Agentic RAG is powerful but slow and expensive. Can you use them in a tiered system where simple queries get simple RAG and complex queries get advanced RAG?*

---

## 🔧 What You'll Install

```bash
# Cross-encoder reranking
pip install sentence-transformers  # Already have this, need cross-encoder models
pip install cohere  # Optional: Cohere Rerank API

# Knowledge Graphs
pip install networkx  # Graph algorithms
pip install pyvis     # Graph visualization
pip install neo4j     # Optional: Neo4j graph DB
pip install pypdf     # PDF parsing for richer documents

# Multimodal
pip install pillow    # Image processing
pip install ftfy      # Unicode handling
pip install torch     # For CLIP models (or use API-based)

# Agentic/LangGraph
pip install langgraph # State machine for agentic RAG

# Evaluation
pip install ragas     # Now with deeper understanding
pip install deepeval  # Alternative eval framework
pip install arize-phoenix  # LLM observability
```

---

## 🏗️ How This Connects to Phase 4

Your Phase 4 Customer Support Bot is the FOUNDATION. Phase 5 doesn't replace it — it extends it:

```
Phase 4 Bot:                  Phase 5 Engine:

[User Query]                  [User Query]
     │                              │
     ▼                              ▼
[Basic Retrieval]             [Query Classifier]
     │                         ┌────┴────┐
     ▼                         ▼         ▼
[Basic Context]          [Simple Q]  [Complex Q]
     │                         │         │
     ▼                         ▼         ▼
[LLM Generation]          [HyDE]    [Agentic]
     │                         │         │
     ▼                    [Rerank]   [Multi-Hop]
[Simple Answer]                │         │
                          [Context] [Context]
                               │         │
                          [Generate] [Generate]
                               │         │
                               ▼         ▼
                          [Simple]   [Complex]
                           Answer      Answer
```

Phase 4 gave you one retrieval strategy. Phase 5 gives you a TOOLKIT of strategies and the intelligence to choose the right one.

---

## 🚦 Phase Gate

Before moving to Phase 6 (AI Agents):

- [ ] Can you implement HyDE and explain when it helps vs. hurts?
- [ ] Have you built a reranking pipeline and measured the precision improvement?
- [ ] Can you design a multi-hop retrieval flow for a 3-document question?
- [ ] Have you processed at least one multimodal query (text + image)?
- [ ] Can you extract entities from a document and build a simple knowledge graph?
- [ ] Have you evaluated your Advanced RAG Engine with at least 4 metrics?
- [ ] **Can you take a failing Phase 5 query, diagnose WHY it fails, and apply the right technique to fix it?**

---

> **The three scenarios from the top:**
>
> 1. **The Vocabulary Mismatch** — HyDE (File 01) is your answer. It reformulates the user's query into document language before embedding.
> 2. **The Multi-Hop Problem** — Agentic RAG (File 04) handles this with iterative retrieval guided by what's already been found.
> 3. **The Relationship Blindness** — GraphRAG (File 06) models entities and relationships explicitly, giving you guaranteed relationship chains.
>
> **Next:** File 01 — HyDE & Query Transformation. Turn queries into documents.
