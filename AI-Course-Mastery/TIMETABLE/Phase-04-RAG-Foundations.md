# Phase 4 — RAG Foundations (Days 44-55)

**Goal:** Master the full RAG pipeline — chunking, retrieval, context integration, failure modes — build a production customer support bot.
**Files:** 8 concept + 1 drill + 1 project (10 files total)
**Total days:** 12 study days + 1 phase-end review

---

### Day 44 — What Is RAG? + Module Overview

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Module overview + What Is RAG | `00-Module-Overview.md`, `01-What-Is-RAG.md` |
| 🌤️ Afternoon | Code: build a naive RAG pipeline end-to-end | Simplest possible RAG: retrieve chunks → stuff into context → generate answer |
| 🌙 Evening | Flashback | Write: "What problem does RAG solve that prompting alone can't?" |

**Key Concepts:** RAG architecture (retrieve → augment → generate), naive RAG limitations, when RAG is the right solution
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** RAG vs fine-tuning vs pure prompting decision, RAG building blocks

---

### Day 45 — Chunking Strategies (Full Day)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Chunking Strategies | `02-Chunking-Strategies.md` (concepts + theory) |
| 🌤️ Afternoon | Code: implement and compare 5 chunking methods | Semantic chunking, recursive character, token-based, document-based, sliding window |
| 🌙 Evening | Flashback | Write: "What chunking strategy would I use for: (a) legal docs, (b) code, (c) conversational data?" |

**Key Concepts:** Chunk size tradeoffs, chunk overlap, semantic chunking, chunking for different document types
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Chunking strategy selection matrix, overlap ratio impact, cost vs quality tradeoff

---

### Day 46 — Retrieval Deep Dive

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Retrieval Deep Dive | `03-Retrieval-Deep-Dive.md` |
| 🌤️ Afternoon | Code: compare retrieval strategies | Dense vs sparse vs hybrid — measure recall@k for each |
| 🌙 Evening | Flashback | Write: "What affects retrieval quality beyond the embedding model?" |

**Key Concepts:** Retrieval quality metrics (recall@k, MRR, NDCG), dense vs sparse, hybrid fusion, multi-stage retrieval
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Retrieval strategy comparison, fusion methods (CC, RRF), query expansion

---

### Day 47 — Context Integration

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Context Integration | `04-Context-Integration.md` |
| 🌤️ Afternoon | Code: build context window manager | Implement context stuffing, dynamic truncation, relevance filtering |
| 🌙 Evening | Flashback | Write: "How do you fit the right amount of context when you have too many retrieval results?" |

**Key Concepts:** Context window budget, dynamic truncation, relevance ranking, context formatting strategies
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Context management strategies, lost-in-the-middle phenomenon, summary-based context

---

### Day 48 — RAG Failure Modes

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | RAG Failure Modes | `05-RAG-Failure-Modes.md` |
| 🌤️ Afternoon | Code: create failure test suite | Deliberately create each failure type — missing context, hallucination, irrelevant retrieval |
| 🌙 Evening | Flashback | Write: "What are the 5 failure modes of RAG and how would I detect each?" |

**Key Concepts:** Missing context, hallucination, irrelevant retrieval, contradictory chunks, recency bias
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Failure mode detection strategies, mitigation approaches per failure mode

---

### Day 49 — Evaluating RAG

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Evaluating RAG | `06-Evaluating-RAG.md` |
| 🌤️ Afternoon | Code: implement RAG evaluation metrics | Faithfulness, answer relevance, context precision/recall |
| 🌙 Evening | Flashback | Write: "How do you measure whether a RAG system is actually working?" |

**Key Concepts:** RAG-specific metrics (faithfulness, answer relevance, context precision), evaluation dataset creation
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** RAGAS metrics, LLM-as-judge for RAG, human eval vs automated

---

### Day 50 — Production RAG Considerations

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Production RAG | `07-Production-RAG.md` |
| 🌤️ Afternoon | Code: add caching, monitoring, fallbacks | Implement response caching, rate limiting, fallback strategies |
| 🌙 Evening | Flashback | Write: "What 3 things would break first in a RAG system at production scale?" |

**Key Concepts:** Caching strategies for RAG, monitoring, cost optimization, fallback chains
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Cache invalidation, latency considerations, cost per query analysis

---

### Day 51 — RAG Drills

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | RAG Drills | `08-Drills-RAG.md` |
| 🌤️ Afternoon | Complete all drills | Work through every exercise — debug broken RAG, optimize chunking, etc. |
| 🌙 Evening | Flashback | What did the drills reveal that the theory didn't? |

**Checklist:** ☀️ ☐ 🌤️ ☐ All drills completed ☐ | Log updated ☐

---

### Day 52 — Project Part 1: Customer Support Bot

**All day = 🏗️ Project**

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Read project spec + design | `09-Project-Customer-Support-Bot.md` — design ingestion pipeline + retrieval |
| 🌤️ Afternoon | Build ingestion + retrieval | Document ingestion, chunking, embedding, basic search |
| 🌙 Evening | Flashback + commit | Architecture decision: why this chunking strategy? → git commit |

**Checklist:**
- [ ] Project spec read and architecture designed
- [ ] Ingestion pipeline working
- [ ] Basic retrieval working
- [ ] Code pushed to GitHub

---

### Day 53 — Project Part 2: Generation + Evaluation

**All day = 🏗️ Project**

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Build generation with context | Implement answer generation with retrieved context |
| 🌤️ Afternoon | Evaluation + polish + Gate Check | 🎯 Add eval metrics, tests, documentation |
| 🌙 Evening | Final commit + reflection | What would break first in production? |

**🎯 Gate Check — Can you:**
- [ ] Ingest documents with configurable chunking strategy?
- [ ] Retrieve relevant chunks for any query?
- [ ] Generate answers with proper context citation?
- [ ] Handle the case where no relevant context is found?
- [ ] Measure and report faithfulness and answer relevance?
- [ ] Cache responses to avoid redundant LLM calls?
- [ ] Identify and handle at least 2 RAG failure modes?
- [ ] Compare at least 2 chunking strategies with quantitative results?

---

### Day 54 — Phase 4 Review

**🔄 Phase-End + Weekly Review combined**

| Session | Focus | Activities |
|---------|-------|------------|
| ☀️ Morning | Scan + recall | Re-read headings of ALL 10 files. Re-type RAG evaluation code from memory. |
| 🌤️ Afternoon | Mind map + weak spots | ✍️ Mind map Phase 4 from memory. Deep-dive on failure modes. |
| 🌙 Evening | 🧪 Self-test | Quiz yourself. |

**🧪 Self-Test Questions:**
1. What problem does RAG solve? When would you NOT use RAG?
2. Walk through a complete RAG pipeline step by step
3. Compare 5 chunking strategies — when would you use each?
4. What's the difference between recall@k, MRR, and NDCG for retrieval evaluation?
5. What is the "lost-in-the-middle" problem and how do you mitigate it?
6. Name 5 RAG failure modes and a detection strategy for each
7. How does RAGAS evaluate RAG quality? Walk through faithfulness vs answer relevance
8. What caching strategies work for RAG in production?
9. How do you handle "no relevant context found" gracefully?
10. What's the cost breakdown of a single RAG query in production?

**Self-Assessment:**
- Total score (out of 10): ________
- Weakest area: ________
