# Tier 2: First Real Clients — RAG & Production Systems

> *"Real business stakes, but Maya is very present. You'll make mistakes but catch them before they reach the client."*

## Why Tier 2 Exists

Tier 1 was your flight simulator. Tier 2 is your **first solo flight with an instructor in the cockpit.**

You're now building systems for actual clients with actual business consequences. The law firm has a board demo. The e-commerce brand has real customers. The recruiting firm has 1,000 CVs coming in every week.

**But Maya is still here.** She won't let you ship something broken. She'll catch you before you go off a cliff. Rohan's bar goes up — he now checks all 7 points, not just functional correctness.

## What Changes From Tier 1

| Dimension | Tier 1 | Tier 2 |
|---|---|---|
| **Projects** | Internal tools | Real client systems |
| **Stakes** | "Would be nice to have" | "Board demo on Friday" |
| **Maya's presence** | Step-by-step guidance | Goal + hints, you figure approach |
| **Rohan's bar** | Points 1-2 (works + edge cases) | Points 1-4 (adds cost + observability) |
| **Core new skill** | API + structured outputs | **RAG** — the most-used AI pattern in production |

## The 4 Projects

| Project | What You Build | The Real Lesson |
|---|---|---|
| **4** | Legal Document Q&A | RAG from scratch. Chunking. Hybrid search. The foundation of ~70% of production AI systems. |
| **5** | E-commerce Support Bot | Multi-tenant RAG. Conversation memory. Confidence thresholds. The "I don't know" problem. |
| **6** | CV Parsing Pipeline | Batch structured extraction at scale. Pydantic validation with retry. Cost optimization at volume. |
| **7** | Multi-Model API Gateway | Model routing. Cost optimization. LiteLLM. The economic layer of AI engineering. |

## What You'll Be Able to Do After Tier 2

- ✅ Build a complete RAG pipeline from scratch (chunk → embed → store → retrieve → generate)
- ✅ Implement hybrid search (dense + sparse) with Qdrant
- ✅ Measure and optimize retrieval quality (Recall@k, MRR, RAGAS/DeepEval metrics)
- ✅ Handle multi-tenant document isolation and "I don't know" responses
- ✅ Build batch extraction pipelines that process 1,000+ documents reliably
- ✅ Implement model routing for cost optimization (40%+ savings)
- ✅ Think in terms of **cost per query**, **latency budgets**, and **observability**

**Portfolio:** 4 client-ready projects with evaluation reports demonstrating measurable quality

## The Golden Rule of Tier 2

> **Every RAG system in production is Tier 2 Project 3 (Semantic Search) plus one step (generation with context).**

If you understood embeddings, vector similarity, and chunking in Project 3, you already understand the engine. Tier 2 adds the "G" in RAG — generation grounded in retrieved context. Everything else is engineering around it.

Let's go. The law firm is waiting.
