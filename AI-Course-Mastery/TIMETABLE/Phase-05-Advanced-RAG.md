# Phase 5 — Advanced RAG (Days 55-69)

**Goal:** Master advanced RAG techniques — HyDE, contextual retrieval, agentic RAG, multimodal, GraphRAG — build an advanced RAG engine.
**Files:** 8 concept + 1 drill + 1 project (10 files — many large, up to 2832 lines)
**Total days:** 14 study days + 1 phase-end review (then Milestone Review)

---

### Day 55 — HyDE & Query Transformation

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Module overview + HyDE & Query Transformation | `00-Module-Overview.md`, `01-HyDE-Query-Transformation.md` |
| 🌤️ Afternoon | Code: implement HyDE + query expansion | Build hypothetical document generation, query expansion, query rewriting |
| 🌙 Evening | Flashback | Write: "How does HyDE improve retrieval and when would it hurt?" |

**Key Concepts:** Hypothetical Document Embeddings, query expansion, query rewriting, step-back prompting
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** HyDE vs standard retrieval, query transformation techniques, when to use each

---

### Day 56 — Contextual Retrieval (Full Day — Large File)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Contextual Retrieval | `02-Contextual-Retrieval.md` (first half — concepts) |
| 🌤️ Afternoon | Code: implement contextual retrieval pipeline | Build the full implementation including context enrichment |
| 🌙 Evening | Flashback | Write: "What makes contextual retrieval different from standard chunk+embed?" |

**Key Concepts:** Contextual chunk headers, context enrichment strategies, contextual embedding
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Contextual retrieval vs HyDE, when contextual retrieval fails

---

### Day 57 — Reranking Deep Dive

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Reranking Deep Dive | `03-Reranking-Deep-Dive.md` |
| 🌤️ Afternoon | Code: implement cross-encoder reranking | Build two-stage retrieval: bi-encoder retrieval → cross-encoder reranking |
| 🌙 Evening | Flashback | Write: "Compare bi-encoder vs cross-encoder — when would you add a reranking stage?" |

**Key Concepts:** Cross-encoder reranking, two-stage retrieval, Cohere rerank API, latency-cost-quality tradeoff
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Reranking tradeoffs (latency added vs quality gained), when reranking doesn't help

---

### Day 58 — Agentic RAG (Full Day — Large File)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Agentic RAG | `04-Agentic-RAG.md` (concepts + architecture) |
| 🌤️ Afternoon | Code: build an agentic RAG loop | Retrieval → evaluate → re-retrieve → generate (iterative loop) |
| 🌙 Evening | Flashback | Write: "What makes RAG 'agentic' vs standard RAG? When is it worth the complexity?" |

**Key Concepts:** Agentic retrieval, self-evaluation, iterative refinement, tool use in RAG
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Agentic RAG vs standard RAG, self-correction mechanisms, cost of iteration

---

### Day 59 — Multimodal RAG

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Multimodal RAG | `05-Multimodal-RAG.md` |
| 🌤️ Afternoon | Code: implement multimodal retrieval | Images → embeddings → retrieve + generate with multimodal LLM |
| 🌙 Evening | Flashback | Write: "How does multimodal RAG differ from text-only RAG? What new challenges appear?" |

**Key Concepts:** Multimodal embedding models, image+text retrieval, multimodal LLM integration
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Multimodal vs text-only tradeoffs, image embedding quality, cost implications

---

### Day 60 — GraphRAG (Full Day — Large File)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | GraphRAG | `06-GraphRAG.md` (concepts + theory) |
| 🌤️ Afternoon | Code: implement GraphRAG pipeline | Entity extraction → knowledge graph → graph traversal + retrieval |
| 🌙 Evening | Flashback | Write: "What types of questions is GraphRAG better at than vector RAG?" |

**Key Concepts:** Knowledge graphs in RAG, entity extraction, relationship traversal, graph+vector fusion
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** GraphRAG vs vector RAG vs hybrid, when GraphRAG complexity is justified

---

### Day 61 — Advanced Evaluation

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Advanced Evaluation | `07-Advanced-Evaluation.md` |
| 🌤️ Afternoon | Code: build advanced RAG eval pipeline | Multi-metric eval, comparative eval across techniques, ablation studies |
| 🌙 Evening | Flashback | Write: "How do you evaluate which advanced RAG technique is worth the complexity?" |

**Key Concepts:** Ablation studies, comparative evaluation, cost-quality-latency tradeoff quantification
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Ablation study design, metrics for comparing RAG techniques

---

### Day 62 — Advanced RAG Drills

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Advanced Drills | `08-Advanced-Drills.md` |
| 🌤️ Afternoon | Complete all drill exercises | Debug broken advanced RAG, optimize HyDE, compare techniques |
| 🌙 Evening | Flashback | Which advanced technique was hardest to implement? Why? |

**Checklist:** ☀️ ☐ 🌤️ ☐ All drills completed ☐ | Log updated ☐

---

### Day 63-65 — Project: Advanced RAG Engine (3 days)

**All days = 🏗️ Project** — `09-Project-Advanced-RAG-Engine.md`

#### Day 63 — Project: Foundation + Multiple Retrieval Strategies

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Read project spec + design | Architecture plan: which techniques to implement, comparison strategy |
| 🌤️ Afternoon | Build base + 2 retrieval strategies | Standard RAG baseline + HyDE + contextual retrieval |
| 🌙 Evening | Flashback + commit | Architecture decisions documented → git commit |

**Checklist:**
- [ ] Project spec read and architecture designed
- [ ] Standard RAG baseline working
- [ ] HyDE retrieval working
- [ ] Contextual retrieval working
- [ ] Code pushed to GitHub

#### Day 64 — Project: Advanced Features

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Add reranking + agentic RAG | Two-stage retrieval, agentic retry loop |
| 🌤️ Afternoon | Multimodal + GraphRAG | At least one of: multimodal retrieval OR GraphRAG |
| 🌙 Evening | Flashback + commit | What was the hardest technique to implement? → git commit |

**Checklist:**
- [ ] Reranking working
- [ ] Agentic RAG working (at minimum: auto-retry on low confidence)
- [ ] At least one advanced technique (multimodal or GraphRAG) working
- [ ] Code pushed to GitHub

#### Day 65 — Project: Evaluation + Polish + Gate Check

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Comparative evaluation | Run ablation: compare all techniques on same questions, measure quality + cost + latency |
| 🌤️ Afternoon | Docs + Gate Check | 🎯 Results table, README, gate checks |
| 🌙 Evening | Final commit + reflection | Which technique gave the biggest quality boost? Was it worth the cost? |

**🎯 Gate Check — Can you:**
- [ ] Run at least 4 different retrieval strategies on the same query?
- [ ] Compare results quantitatively (retrieval quality, generation quality, latency, cost)?
- [ ] Implement HyDE and explain when it helps vs hurts?
- [ ] Implement cross-encoder reranking?
- [ ] Show an agentic RAG loop (retrieve → evaluate → re-retrieve)?
- [ ] Demonstrate multimodal retrieval OR GraphRAG?
- [ ] Produce a comparison table of techniques with metrics?
- [ ] Give a recommendation: which technique for which use case?

---

### Day 66-67 — Phase 5 Review

**Note: this review is combined with the beginning of the Milestone Review. After Phase 5, you'll do a full cumulative review of Phases 0-5. See [Review-System.md](./Review-System.md) for the 3-day Milestone Review plan.**

#### Day 66 — Phase 5 Individual Review

| Session | Focus | Activities |
|---------|-------|------------|
| ☀️ Morning | Scan + recall | Re-read headings of ALL 10 files. Re-type HyDE pipeline from memory. |
| 🌤️ Afternoon | Mind map + weak spots | ✍️ Mind map Phase 5. Re-read sections on weakest technique. |
| 🌙 Evening | 🧪 Self-test | Phase 5-specific questions below |

**🧪 Phase 5 Self-Test Questions:**
1. How does HyDE work? When would it reduce retrieval quality?
2. What problem does contextual retrieval solve? How does it differ from chunking improvements?
3. Explain the bi-encoder → cross-encoder two-stage retrieval architecture
4. What makes RAG "agentic"? When would you avoid agentic RAG?
5. How does multimodal RAG work? What embedding strategy does it use?
6. What is GraphRAG and what types of questions is it best for?
7. How do you run an ablation study comparing 4 retrieval techniques?
8. What metrics should you track when comparing RAG techniques?
9. When is reranking NOT worth the added latency?
10. What cost do you incur with agentic RAG loops?

---

#### Day 67 — Phase 5 Completion + Milestone Prep

| Session | Focus | Activities |
|---------|-------|------------|
| ☀️ Morning | Phase 5 summary document | Write a one-page summary of ALL advanced RAG techniques you now know |
| 🌤️ Afternoon | Prepare for Milestone Review | Collect your Phase 1-5 logs, notes, and project repos |
| 🌙 Evening | Rest + mental prep | Tomorrow starts the 3-day Milestone Review of everything from Phase 0 to Phase 5 |

**Checklist:**
- [ ] Phase 5 summary written
- [ ] All Phase 1-5 notes organized
- [ ] All Phase 1-5 projects pushed to GitHub
- [ ] Logs ready for review
