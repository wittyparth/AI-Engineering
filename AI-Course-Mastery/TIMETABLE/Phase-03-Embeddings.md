# Phase 3 — Embeddings & Vector Databases (Days 33-43)

**Goal:** Deep understanding of embeddings, build semantic search from scratch, master vector databases (ChromaDB, Qdrant, pgvector).
**Files:** 8 concept + 1 drill + 1 project (10 files total)
**Total days:** 11 study days + 1 phase-end review

---

### Day 33 — Module Overview + Embeddings Deep Dive

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Module overview + Embeddings deep dive | `00-Module-Overview.md`, `01-Embeddings-Deep-Dive.md` |
| 🌤️ Afternoon | Code: generate embeddings, explore vector properties | Type all code examples — play with embedding similarity |
| 🌙 Evening | Flashback | Write: "What is an embedding vector and what makes a good one?" |

**Key Concepts:** Embedding as semantic representation, cosine similarity, embedding dimensions, pooling strategies
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** How embeddings capture meaning, cosine vs dot product vs euclidean, embedding model comparison

---

### Day 34 — Semantic Search From Scratch (Full Day)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Build semantic search from scratch | `02-Semantic-Search-From-Scratch.md` (concepts + first half) |
| 🌤️ Afternoon | Complete the implementation | Type the full search engine code — indexing + querying |
| 🌙 Evening | Flashback | Write: "How would I explain semantic search to a junior dev in 3 paragraphs?" |

**Key Concepts:** Search architecture, index building, query embedding, similarity ranking
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Dense vs sparse retrieval, indexing strategies, search quality metrics

---

### Day 35 — Embedding Model Selection

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Embedding Model Selection | `03-Embedding-Model-Selection.md` |
| 🌤️ Afternoon | Code: benchmark 3 embedding models on a real task | Compare OpenAI `text-embedding-3-small`, `text-embedding-3-large`, and a local model |
| 🌙 Evening | Flashback | Write: "What criteria would I use to choose an embedding model for a project?" |

**Key Concepts:** Model size vs quality tradeoff, MTEB benchmark, domain-specific embedding models
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** MTEB benchmark interpretation, model selection decision tree, cost vs quality tradeoff

---

### Day 36 — ChromaDB Deep Dive

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | ChromaDB | `04-ChromaDB.md` |
| 🌤️ Afternoon | Code: build a full ChromaDB application | Implement CRUD operations, collection management, filtering |
| 🌙 Evening | Flashback | Write: "When would I use ChromaDB vs building from scratch?" |

**Key Concepts:** ChromaDB architecture, collections, metadata filtering, persistence
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** ChromaDB limitations, when to migrate to production-grade DB

---

### Day 37 — Qdrant & Pinecone (Production Vector DBs)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Qdrant & Pinecone | `05-Qdrant-Pinecone.md` |
| 🌤️ Afternoon | Code: deploy Qdrant locally + perform searches | Type all examples, compare API differences |
| 🌙 Evening | Flashback | Write: "Compare ChromaDB, Qdrant, and Pinecone — when would you choose each?" |

**Key Concepts:** Qdrant architecture, payload indexing, filtering, sharding, cloud vs self-hosted
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Qdrant vs Pinecone feature comparison, cost analysis, self-hosting decision

---

### Day 38 — pgvector (PostgreSQL Integration)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | pgvector | `06-pgvector.md` |
| 🌤️ Afternoon | Code: implement pgvector search with SQL + Postgres | Type all examples — create indexes, query with distances |
| 🌙 Evening | Flashback | Write: "When would pgvector be the best choice over a dedicated vector DB?" |

**Key Concepts:** pgvector architecture, index types (IVFFlat, HNSW), hybrid SQL + vector queries
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** pgvector vs dedicated vector DB, HNSW vs IVFFlat tradeoffs, operational simplicity

---

### Day 39 — Advanced Vector Search

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Advanced Vector Search | `07-Advanced-Vector-Search.md` |
| 🌤️ Afternoon | Code: hybrid search + filtering | Implement hybrid (dense + sparse/BM25), metadata filtering, multi-vector search |
| 🌙 Evening | Flashback | Write: "What does 'advanced' vector search look like beyond simple similarity?" |

**Key Concepts:** Hybrid search, multi-vector retrieval, filtered search, late interaction
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Hybrid search tradeoffs, late interaction vs early fusion

---

### Day 40 — Embeddings Drill

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Embeddings Drill | `08-Drills-Embeddings.md` |
| 🌤️ Afternoon | Complete all drill exercises | Work through every exercise in the drill file |
| 🌙 Evening | Flashback | What was the hardest drill? What did I learn from it? |

**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | All drill exercises completed ☐ | Log updated ☐

---

### Day 41 — Project Part 1: Knowledge Search Engine Setup

**All day = 🏗️ Project**

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Read project spec + design architecture | `09-Project-Knowledge-Search-Engine.md` |
| 🌤️ Afternoon | Implement ingestion pipeline | Document ingestion → chunking → embedding → storing |
| 🌙 Evening | Flashback + commit | Architecture decisions documented → git commit |

**Checklist:**
- [ ] Project spec read and architecture diagram sketched
- [ ] Ingestion pipeline working (docs → chunks → vectors)
- [ ] Code pushed to GitHub

---

### Day 42 — Project Part 2: Search + Polish

**All day = 🏗️ Project**

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Implement search endpoints | Semantic search, hybrid search, filtered search APIs |
| 🌤️ Afternoon | Testing + docs + Gate Check | 🎯 Tests, README, run gate checks |
| 🌙 Evening | Final commit + reflection | What surprised me about working with vector search? |

**🎯 Gate Check — Can you:**
- [ ] Ingest documents, chunk them, generate embeddings, and store them?
- [ ] Perform semantic search with cosine similarity?
- [ ] Perform hybrid search (vector + keyword)?
- [ ] Filter results by metadata?
- [ ] Compare results across 2 different embedding models?
- [ ] Explain the tradeoffs between ChromaDB, Qdrant, and pgvector?
- [ ] Benchmark search quality for different chunk sizes?

---

### Day 43 — Phase 3 Review

**🔄 Phase-End + Weekly Review combined**

| Session | Focus | Activities |
|---------|-------|------------|
| ☀️ Morning | Scan + recall | Re-read headings of ALL 10 files. Re-type semantic search from scratch. |
| 🌤️ Afternoon | Mind map + weak spots | ✍️ Mind map Phase 3 from memory. Work through fuzzy concepts. |
| 🌙 Evening | 🧪 Self-test | Quiz yourself. |

**🧪 Self-Test Questions:**
1. What is an embedding and how does it represent meaning?
2. Walk through the architecture of semantic search from scratch
3. Compare ChromaDB, Qdrant, and pgvector — pick one for a production app and justify
4. What is cosine similarity and when would you NOT use it?
5. How does hybrid search work? Why would you need it?
6. What's the difference between IVFFlat and HNSW indexes?
7. How do you choose an embedding model? What tradeoffs exist?
8. What metadata filtering strategies work with vector search?
9. How does chunk size affect search quality?
10. When would pgvector be a bad choice?

**Self-Assessment:**
- Total score (out of 10): ________
- Weakest area: ________
