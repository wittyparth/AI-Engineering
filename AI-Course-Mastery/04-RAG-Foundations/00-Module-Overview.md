# Phase 4: RAG Foundations — Making LLMs Answer from Your Data

## 🎯 Phase Purpose

> **🛑 STOP. Three scenarios. Think about each before reading on.**

### 🤔 Scenario 1: The Expiration Problem

You built a customer support chatbot using GPT-4. It's great — sounds natural, handles complex questions. But there's a problem: your company changes its return policy every quarter. The GPT-4 training data is 18 months old. It keeps telling customers about the OLD policy.

**The obvious fix:** Fine-tune GPT-4 on your new policy every quarter.

**The not-so-obvious problem:** Fine-tuning costs $500+ per run. And next quarter, you'll need to do it again. And for EVERY policy change. And for EVERY new product. And your company has 200 products.

**Your challenge before reading on:** What's a cheaper, faster alternative to fine-tuning every time your data changes? How can you give the LLM "fresh" information WITHOUT retraining it?

---

### 🤔 Scenario 2: The 10,000 Page Problem

Your company has 10,000 pages of internal documentation. An employee asks: "What's the procedure for requesting a PTO during the holiday blackout period?"

The answer is in one paragraph, buried on page 7,432 of the employee handbook.

**You can either:**
- A) Send ALL 10,000 pages as context to GPT-4 (cost: ~$80 per query, and it won't fit in the context window)
- B) Find the RIGHT 3 pages and send only those
- C) Fine-tune GPT-4 on all 10,000 pages (cost: $1,000+, must repeat monthly)

**Which approach makes sense? And HOW would you find the right 3 pages out of 10,000?**

*(Hint: What did you build in Phase 3?)*

---

### 🤔 Scenario 3: The Hallucination Trap

You deploy a RAG system. It works great — 95% of answers are correct and cite sources. But there's a pattern you notice: when the retrieved documents DON'T contain the answer, the LLM makes something up that sounds plausible.

**Example:**
> User: "Do you offer a student discount?"
> Retrieved docs: [general pricing page, FAQ about payment methods]
> LLM answer: "Yes! We offer a 15% student discount with a valid .edu email. Just use code STUDENT15 at checkout."
> Reality: Your company has NO student discount. The LLM invented it.

**Your challenge:** The retrieval phase (Phase 3) did its job — it found the most relevant documents. They just didn't contain the answer. The LLM, being helpful, filled in the gap with a hallucination.

**How would you detect and prevent this?** Think about it before reading the solutions in this phase.

---

**By the end of this phase, you will:**
- Build a complete RAG system from scratch — your Phase 3 search engine + an LLM
- Understand why naive RAG fails and how to fix each failure mode
- Master 5 chunking strategies and know when to use each
- Implement hybrid search + re-ranking specifically for RAG (different from Phase 3)
- Build evaluation pipelines that catch hallucinations before users do
- Ship a Production Customer Support Bot as your portfolio project

**⏱ Phase Budget:** 14-18 days (2-3 hours/day)
**Portfolio Project:** Production Customer Support Bot
**Tech Stack:** FastAPI, LangChain (prototyping), LlamaIndex (data-centric), ChromaDB/Qdrant, OpenAI, Anthropic, RAGAS

---

## 🔗 Connecting to Previous Phases

```
Phase 1 (LLM Gateway)         → You'll use your gateway to call LLMs
Phase 2 (Prompt Engineering)   → RAG prompts are DIFFERENT — you'll learn why
Phase 3 (Embeddings & Vector) → This is the RETRIEVAL layer RAG depends on!

This phase is where everything converges:
   Phase 1 (LLM API) + Phase 3 (Vector Search) + Phase 2 (Prompts) = RAG
```

## 📖 Phase Map

```
PHASE 4: RAG FOUNDATIONS
═══════════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│                  THE RAG PIPELINE (Mental Model)                 │
│                                                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │  User    │───→│ Retrieve │───→│  Augment │───→│ Generate │  │
│  │  Query   │    │ (Vector) │    │ (Prompt) │    │  (LLM)   │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│                        │              │              │          │
│                        ▼              ▼              ▼          │
│                  ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│                  │ Phase 3 │    │  Chunk   │    │  Prompt  │  │
│                  │ Vector  │    │ Strategy │    │ Template │  │
│                  │ Search  │    │          │    │          │  │
│                  └──────────┘    └──────────┘    └──────────┘  │
└─────────────────────────────────────────────────────────────────┘

FILE MAP:
┌────────────────────────────────────────────────────────────────────┐
│ 00: Module Overview ← YOU ARE HERE                                  │
│                                                                    │
│ 01: What Is RAG?            → Build naive RAG from scratch         │
│ 02: Chunking Strategies     → How to split documents (5 methods)    │
│ 03: Retrieval Deep Dive     → Sparse, dense, hybrid for RAG        │
│ 04: Context Integration     → Prompt templates, windowing, ordering │
│ 05: RAG Failure Modes       → 7 ways RAG breaks in production      │
│ 06: Evaluating RAG          → RAGAS, faithfulness, relevance        │
│ 07: Production RAG          → Caching, streaming, monitoring       │
│ 08: Drills                  → Debug broken RAG, optimize queries    │
│ 09: Project                 → Production Customer Support Bot      │
└────────────────────────────────────────────────────────────────────┘
```

---

## 📋 File-by-File Breakdown

| # | File | What You'll Build/Discover | Time |
|---|------|---------------------------|------|
| **00** | Module Overview | Phase map, 3 big questions, connecting all prior phases | — |
| **01** | What Is RAG? | Connect your Phase 3 search engine to an LLM. Build naive RAG end-to-end | 2.5h |
| **02** | Chunking Strategies | 5 chunking methods. Discover how chunk size affects answer quality | 2h |
| **03** | Retrieval Deep Dive | Sparse (BM25), dense (vector), hybrid for RAG. Query transformation | 2.5h |
| **04** | Context Integration | Prompt templates for RAG, windowing, re-ranking retrieved chunks | 2h |
| **05** | RAG Failure Modes | 7 ways RAG breaks. Detection, mitigation, monitoring | 2h |
| **06** | Evaluating RAG | RAGAS, faithfulness, answer relevance, context precision | 2.5h |
| **07** | Production RAG | Caching, streaming, monitoring, latency optimization | 2h |
| **08** | Drills | Debug 5 broken RAG systems, optimize retrieval, build evaluation | 1.5h |
| **09** | Project | Full Production Customer Support Bot with evaluation | 8-10h |

---

## 🛑 The Three Big Questions — Revisit After Each File

Keep these in your mind throughout every file:

1. **Where does RAG fail that's NOT obvious from a demo?**  
   *Demos show RAG working perfectly. Production shows RAG failing in 7+ distinct ways. Can you predict them before they happen?*

2. **How do you know if the retrieved context is ACTUALLY useful?**  
   *The LLM might ignore the context. It might hallucinate despite correct context. It might have enough knowledge without retrieval. How do you measure "usefulness"?*

3. **What's the cost-quality-latency tradeoff for each RAG component?**  
   *More chunks = better recall but higher cost. Re-ranking = better precision but higher latency. Bigger LLM = better answers but higher cost. How do you optimize the WHOLE system?*

---

## 🔧 What You'll Install

```bash
# From Phase 3 (keep these)
pip install chromadb qdrant-client sentence-transformers

# New for Phase 4
pip install langchain langchain-community  # RAG prototyping
pip install llama-index  # Data-centric RAG
pip install ragas  # RAG Evaluation
pip install datasets  # For evaluation datasets
pip install tiktoken  # Token counting
pip install unstructured  # Document parsing (PDF, HTML, etc.)
pip install nltk  # For chunking improvements

# LLM APIs (you should have these from Phase 1)
pip install openai anthropic
```

---

## 🏗️ How This Connects to Phase 3

The search engine you built in Phase 3 is the **R** in **RAG**:

```
Your Phase 3 Search Engine (Retrieval):
  ingest() → chunks → embeddings → vector DB
  search() → query → embedding → nearest neighbors

Phase 4 adds:
  generate(query, retrieved_chunks) → LLM response
  evaluate(query, retrieved_chunks, response) → quality score
  
The question Phase 4 answers: Once you have the right documents,
HOW do you feed them to an LLM to get a good answer?
```

---

## 🚦 Phase Gate

Before moving to Phase 5 (Advanced RAG):

- [ ] Can you explain RAG to a non-technical person in 2 sentences?
- [ ] Have you built naive RAG from scratch (no frameworks)?
- [ ] Can you compare 3 chunking strategies and choose one for a given document type?
- [ ] Can you name 5 failure modes of RAG and how to detect each?
- [ ] Have you evaluated your RAG system with RAGAS or a similar framework?
- [ ] Did you ship the Customer Support Bot project?
- [ ] **Can you find a query where your RAG system gives a wrong answer, diagnose WHY, and fix it?**

---

> **Keep the 3 scenarios from the top in mind as you go through each file.**
>
> 1. **The Expiration Problem** — RAG solves this. By the end of this phase, you'll know when it works and when it doesn't.
> 2. **The 10,000 Page Problem** — This is what you built in Phase 3. Now you'll add the "G" (Generation) to your search engine.
> 3. **The Hallucination Trap** — This is the #1 production RAG failure. File 05 and File 06 will teach you how to catch it.
>
> **Next:** File 01 — What Is RAG? Building your first end-to-end RAG system.
