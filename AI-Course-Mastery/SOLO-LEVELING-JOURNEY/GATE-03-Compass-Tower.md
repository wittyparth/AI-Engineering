# ⚜️ GATE 3: THE COMPASS TOWER

```
[SYSTEM] ─────────────────────────────────────
  Gate 3 is opening.

  "Words are powerful, Jinwoo.
   But meaning lives in the spaces between them.
   Learn to navigate the geometry of knowledge."

  ─────────────────────────────────────

  Rank Required:  C-5 or higher
  Dungeons:       7
  Boss Battle:    1 (Knowledge Search Engine)
  Est. Time:      14-16 days
  XP Available:   ~2,800
  Skill Unlock:   🏹 Embedding Arrow
  Stat Focus:     🧠 Intelligence · 💨 Agility
─────────────────────────────────────────────
```

---

## The Gate

In Solo Leveling, the Tower of Trials tested Jinwoo's ability to navigate the unknown. Each floor presented a different challenge, forcing him to adapt.

This is your Tower of Trials. You're entering the world of **embeddings and vector search** — the technology that powers semantic understanding in every modern AI system.

Embeddings are the foundation of RAG, semantic search, clustering, and recommendation systems. Every AI engineer must understand them deeply — not just how to call an API, but what's happening geometrically.

---

## 🏰 DUNGEON 3.1: Embedding Intuition

```
[SYSTEM] Entering Dungeon 3.1...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   1 sprint (90 min)
  XP:          +100
  Stats:       🧠 INT +4
─────────────────────────────────────────────
```

### The Intuition

Embeddings are vectors (lists of numbers) that represent the **meaning** of text in a multi-dimensional space. Similar meanings are geometrically close. Different meanings are far apart.

"King" and "Queen" are close. "King" and "Apple" are far. This is NOT keyword matching — it's semantic understanding.

### Sprint 1 (90 min)

```
☀️ THE MISSION
───────────────────────────────────────────
  Read: `03-Embeddings-Vector-DBs/01-Embeddings-Deep-Dive.md`

  As you read, write down answers to:
  1. What does "dimension" mean in embedding space?
  2. Why does "King - Man + Woman ≈ Queen" work?
  3. What makes a good embedding model vs a bad one?

  THE TWIST CHALLENGE:
  After reading, close the file.
  Write the "King - Man + Woman ≈ Queen" analogy
  from scratch — explain WHY this works geometrically.

  ┌──────────────────────────────────────┐
  │                                      │
  │                                      │
  │                                      │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] I understand what embeddings represent geometrically
- [ ] I can explain the analogy without looking
- [ ] I know what "dimension" means in this context

---

## 🏰 DUNGEON 3.2: Semantic Search from Scratch

```
[SYSTEM] Entering Dungeon 3.2...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   2 sprints (1 day)
  XP:          +200
  Stats:       ⚔️ STR +5 | 🧠 INT +3
─────────────────────────────────────────────
```

### The Intuition

Before using any vector database, you need to understand how similarity search works under the hood. Cosine similarity is just math — but it's the math that powers every AI search system.

### Sprint 1 (90 min) — Build First

```
☀️ BUILD FIRST
───────────────────────────────────────────
  Time-box: 90 min. Implement cosine similarity
  and semantic search from scratch.

  NO NUMPY. Pure Python.

  Requirements:
  • Implement cosine_similarity(vec_a, vec_b) → float
  • Implement euclidean_distance(vec_a, vec_b) → float
  • Build a simple search: embed 10 sentences, find top-3 similar
  • Compare cosine vs euclidean — when are they different?

  STARTER (type this):
  ┌──────────────────────────────────────┐
  │ import math                          │
  │                                      │
  │ def cosine_similarity(a, b):        │
  │     dot = sum(x*y for x,y in        │
  │                zip(a, b))            │
  │     norm_a = math.sqrt(              │
  │         sum(x*x for x in a))        │
  │     norm_b = math.sqrt(              │
  │         sum(y*y for y in b))        │
  │     return dot / (norm_a * norm_b)   │
  │     if norm_a * norm_b > 0 else 0   │
  └──────────────────────────────────────┘

  PREDICTION before running:
  "For the sentences 'I love cats' and 'I love dogs',
  cosine similarity will be approximately..."
  ┌──────────────────────────────────────┐
  │                                      │
  └──────────────────────────────────────┘

  GO → 90:00
```

### Sprint 2 — Deep Dive + The Twist

```
🌅 DEEP DIVE
───────────────────────────────────────────
  Read: `03-Embeddings-Vector-DBs/02-Semantic-Search-From-Scratch.md`

  THE TWIST:
  Add BM25 (keyword-based) search alongside your embedding search.
  Then implement HYBRID SEARCH:
  score = 0.3 * bm25_score + 0.7 * embedding_score

  Compare pure embedding vs hybrid on 5 test queries:
  ┌──────────────────────────────────────┐
  │ Query │ Embedding │ Hybrid │ Winner  │
  │───────│───────────│────────│─────────│
  │       │           │        │         │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] Cosine similarity implemented from scratch
- [ ] Semantic search works end-to-end
- [ ] BM25 + hybrid search implemented
- [ ] I understand when embedding search is better than keyword

---

## 🏰 DUNGEON 3.3: Embedding Model Selection

```
[SYSTEM] Entering Dungeon 3.3...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   1 sprint (90 min)
  XP:          +100
  Stats:       🧠 INT +3 | 👁️ PER +2
─────────────────────────────────────────────
```

### Sprint 1 (90 min)

```
☀️ THE MISSION
───────────────────────────────────────────
  Read: `03-Embeddings-Vector-DBs/03-Embedding-Model-Selection.md`

  BUILD: Compare 3 embedding models on the same search task:
  1. text-embedding-3-small (OpenAI)
  2. text-embedding-3-large (OpenAI)
  3. nomic-embed-text (Ollama, local)

  For each, measure:
  • Quality: does it return relevant results?
  • Cost per 1K documents
  • Latency per query
  • Dimensions (affects storage cost)

  ┌──────────────────────────────────────┐
  │ Model │ Cost │ Latency │ Quality │    │
  │───────│──────│─────────│─────────│    │
  │       │      │         │         │    │
  └──────────────────────────────────────┘

  PREDICTION:
  "text-embedding-3-large will be better than small by..."
  ┌──────────────────────────────────────┐
  │                                      │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] Compared 3 embedding models quantitatively
- [ ] I understand the cost-quality-latency tradeoff
- [ ] I know which model to use for which scenario

---

## 🏰 DUNGEON 3.4: ChromaDB

```
[SYSTEM] Entering Dungeon 3.4...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   2 sprints (1 day)
  XP:          +200
  Stats:       ⚔️ STR +4 | 💨 AGI +3
─────────────────────────────────────────────
```

### Sprint 1 (90 min) — Build First

```
☀️ BUILD FIRST
───────────────────────────────────────────
  Time-box: 90 min. Build a vector store with ChromaDB.

  Requirements:
  • Install chromadb (pip install chromadb)
  • Create a collection
  • Add 20+ documents with metadata
  • Query by semantic similarity
  • Filter by metadata

  STARTER:
  ┌──────────────────────────────────────┐
  │ import chromadb                      │
  │ from chromadb.config import Settings │
  │                                      │
  │ client = chromadb.Client(Settings(   │
  │     persist_directory="./chroma_db"  │
  │ ))                                   │
  │                                      │
  │ collection = client.create_collection│
  │     ("my_docs")                      │
  │                                      │
  │ collection.add(                      │
  │     documents=[...],                  │
  │     metadatas=[...],                  │
  │     ids=[...]                        │
  │ )                                    │
  └──────────────────────────────────────┘

  GO → 90:00
```

### Sprint 2 — Deep Dive

```
🌅 DEEP DIVE
───────────────────────────────────────────
  Read: `03-Embeddings-Vector-DBs/04-ChromaDB.md`

  THE TWIST:
  Add metadata filtering to your search:
  • Filter by category, date range, or author
  • Combine semantic search + metadata filter
  • Measure how filters affect recall
```

### Gate Check
- [ ] ChromaDB collection created with documents
- [ ] Semantic search + metadata filtering works
- [ ] I understand ChromaDB's architecture
- [ ] Collection persisted and reloads correctly

---

## 🏰 DUNGEON 3.5: Qdrant & Pinecone

```
[SYSTEM] Entering Dungeon 3.5...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   2 sprints (1 day)
  XP:          +200
  Stats:       ⚔️ STR +5 | 🛡️ END +3
─────────────────────────────────────────────
```

### Sprint 1 (90 min)

```
☀️ BUILD IT
───────────────────────────────────────────
  Time-box: 90 min. Build a production vector
  search pipeline with Qdrant.

  Requirements:
  • Spin up Qdrant (Docker: docker run -p 6333:6333 qdrant/qdrant)
  • Create collection with proper config
  • Insert 50+ documents with payloads
  • Search with filters
  • Measure query latency

  PREDICTION:
  "Qdrant queries will be ___ than ChromaDB because..."
  ┌──────────────────────────────────────┐
  │                                      │
  └──────────────────────────────────────┘

  GO → 90:00
```

### Sprint 2 — Deep Dive

```
🌅 DEEP DIVE
───────────────────────────────────────────
  Read: `03-Embeddings-Vector-DBs/05-Qdrant-Pinecone.md`

  THE TWIST:
  Benchmark query latency for 1, 10, 100, 1000 concurrent queries.
  Plot the results:
  ┌──────────────────────────────────────┐
  │ QPS  │ Avg Latency │ P99 │           │
  │──────│─────────────│─────│           │
  │ 1    │             │     │           │
  │ 10   │             │     │           │
  │ 100  │             │     │           │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] Qdrant running with documents
- [ ] Filtered search works
- [ ] Benchmark completed
- [ ] I understand when to use Qdrant vs Pinecone vs ChromaDB

---

## 🏰 DUNGEON 3.6: pgvector

```
[SYSTEM] Entering Dungeon 3.6...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   1 sprint (90 min)
  XP:          +150
  Stats:       ⚔️ STR +4 | 🧠 INT +2
─────────────────────────────────────────────
```

### The Intuition

You already use PostgreSQL. Now add vector search to it. pgvector lets you store and query embeddings alongside your relational data — no separate vector database needed for many use cases.

### Sprint 1 (90 min)

```
☀️ BUILD IT
───────────────────────────────────────────
  Time-box: 90 min. Set up pgvector with Docker.

  Requirements:
  • Docker compose with PostgreSQL + pgvector
  • Create a table with a vector column
  • Insert documents with embeddings
  • Query by cosine similarity using SQL
  • Add a GiST index for performance

  docker-compose.yml starter:
  ┌──────────────────────────────────────┐
  │ version: '3.8'                       │
  │ services:                            │
  │   db:                                │
  │     image: pgvector/pgvector:pg16    │
  │     environment:                     │
  │       POSTGRES_DB: vectordb          │
  │       POSTGRES_USER: user            │
  │       POSTGRES_PASSWORD: pass        │
  │     ports:                           │
  │       - "5432:5432"                  │
  └──────────────────────────────────────┘

  Read: `03-Embeddings-Vector-DBs/06-pgvector.md`

  GO → 90:00
```

### Gate Check
- [ ] pgvector container running
- [ ] Vector column created and queried
- [ ] I understand the SQL interface for vector search
- [ ] I know when to use pgvector vs dedicated vector DB

---

## 🏰 DUNGEON 3.7: Advanced Vector Search

```
[SYSTEM] Entering Dungeon 3.7...
─────────────────────────────────────────────
  Difficulty:  C
  Est. Time:   2 sprints (1 day)
  XP:          +250
  Stats:       🧠 INT +5 | 👁️ PER +4
─────────────────────────────────────────────
```

### Sprint 1 (90 min)

```
☀️ BUILD IT
───────────────────────────────────────────
  Time-box: 90 min. Build hybrid search
  (dense + sparse embeddings).

  Requirements:
  • Generate dense embeddings (text-embedding-3-small)
  • Generate sparse embeddings (SPLADE or BM25)
  • Combine both with weighted scoring
  • Implement multi-vector search (query expansion)

  Read: `03-Embeddings-Vector-DBs/07-Advanced-Vector-Search.md`

  GO → 90:00
```

### Sprint 2 — Deep Dive

```
🌅 THE TWIST
───────────────────────────────────────────
  Implement multi-vector search:
  For a single query, generate 3 semantically similar
  variants. Search all 3. Merge results.

  Compare single-query vs multi-query recall:
  ┌──────────────────────────────────────┐
  │ Single query recall: ___%            │
  │ Multi-query recall:  ___%            │
  │ Improvement:         ___%            │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] Hybrid search (dense + sparse) implemented
- [ ] Multi-vector search implemented
- [ ] I understand dense vs sparse embeddings
- [ ] Measured recall improvement

---

## 👑 BOSS BATTLE: Personal Knowledge Search Engine

```
████████████████████████████████████████████
  BOSS BATTLE — KNOWLEDGE SEARCH ENGINE
  Difficulty:  C
  XP:          +500
  Stats:       ⚔️ STR +8 | 🧠 INT +6 | 💨 AGI +6
  Skill:       🏹 Embedding Arrow → Lv. 3
  Unlocks:     Gate 4: The Archive
████████████████████████████████████████████
```

### The Mission

Build a Personal Knowledge Search Engine that takes a set of your documents and makes them semantically searchable. This will be the foundation of your RAG systems in Gates 4-5.

### Project Plan

**Tech Stack:** FastAPI + Qdrant + OpenAI embeddings + your choice of frontend (CLI or simple web)

**Day 1:** Data ingestion pipeline — chunk documents, generate embeddings, store in Qdrant
**Day 2:** Search API — semantic search, hybrid search, metadata filtering
**Day 3:** Reranking — add cross-encoder reranking to improve results
**Day 4:** Evaluation — build test set, measure recall@k, MRR
**Day 5:** Production polish — error handling, caching, config
**Day 6:** Documentation + Gate Check

### Gate Pass Requirements

- [ ] Ingests documents and creates searchable index
- [ ] Semantic search returns relevant results
- [ ] Hybrid search (dense + sparse) implemented
- [ ] Reranking improves result quality
- [ ] Evaluation metrics measured and reported
- [ ] README with architecture and usage
- [ ] I typed every line (no copy-paste)

### Shadow Extraction

```
BOSS SHADOW: Knowledge Search Engine
───────────────────────────────────────────
Explain how embedding search works end-to-end:
_________________________________________
_________________________________________
_________________________________________
```

---

## Rewards

```
═══════════════════════════════════════════
  GATE 3 CLEARED — THE COMPASS TOWER

  Rewards:
  • +~2,800 XP total
  • Stats: ⚔️ STR +26 | 🧠 INT +27
            👁️ PER +6  | 🛡️ END +3 | 💨 AGI +9
  • Skills: 🏹 Embedding Arrow (Lv. 3)
  • Shadow Army: +8 new soldiers
  • Unlocked: Gate 4 — The Archive

  You've mastered the geometry of meaning.
  Now learn to retrieve what matters.
═══════════════════════════════════════════
```

---

*Next: [GATE-04-The-Archive.md](./GATE-04-The-Archive.md)*
