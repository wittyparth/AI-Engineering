# ⚜️ GATE 4: THE ARCHIVE

```
[SYSTEM] ─────────────────────────────────────
  Gate 4 is opening.

  "You've learned to navigate the space of knowledge.
   Now learn to retrieve what matters most.
   The Archive holds answers — but only if
   you know how to ask."

  ─────────────────────────────────────

  Rank Required:  C-3 or higher
  Dungeons:       7
  Boss Battle:    1 (Customer Support Bot)
  Est. Time:      16-18 days
  XP Available:   ~3,000
  Skill Unlock:   🛡️ RAG Barrier
  Stat Focus:     🛡️ Endurance · 🧠 Intelligence
─────────────────────────────────────────────
```

---

## The Gate

In Solo Leveling, the Archive was a hidden dungeon filled with ancient knowledge — but it was guarded by monsters that fed on ignorance.

RAG (Retrieval Augmented Generation) is the Archive of AI Engineering. It's the technique that turns LLMs from "trained on data up to 2024" into systems that answer questions about YOUR specific documents, in real-time.

This Gate is where everything clicks. Embeddings (Gate 3) + prompts (Gate 2) + Gateway (Gate 1) = RAG. Every piece builds on the last.

---

## 🏰 DUNGEON 4.1: What Is RAG?

```
[SYSTEM] Entering Dungeon 4.1...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   1 sprint (90 min)
  XP:          +100
  Stats:       🧠 INT +4
─────────────────────────────────────────────
```

### The Intuition

RAG is simple: when the user asks a question, first SEARCH your knowledge base for relevant documents, then FEED those documents to the LLM as context, then LET the LLM answer based on what you gave it.

```
User Question → Search Documents → Find Relevant Chunks
    ↓
Feed Chunks + Question to LLM
    ↓
LLM Answers Based on Context
```

No training. No fine-tuning. Just smart retrieval + smart prompting.

### Sprint 1 (90 min)

```
☀️ THE MISSION
───────────────────────────────────────────
  Read: `04-RAG-Foundations/01-What-Is-RAG.md`

  THE TWIST CHALLENGE:
  After reading, close the file.
  Draw the RAG architecture from memory:

  ┌──────────────────────────────────────┐
  │                                      │
  │                                      │
  │                                      │
  │                                      │
  │                                      │
  │                                      │
  │                                      │
  └──────────────────────────────────────┘

  Now compare with the actual architecture.
  What did you miss?
  ┌──────────────────────────────────────┐
  │ I missed:                             │
  │                                      │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] I can draw RAG architecture from memory
- [ ] I understand the 3 stages: retrieve, augment, generate
- [ ] I understand why RAG is better than fine-tuning for most use cases

---

## 🏰 DUNGEON 4.2: Chunking Strategies

```
[SYSTEM] Entering Dungeon 4.2...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   2 sprints (1 day)
  XP:          +200
  Stats:       🧠 INT +4 | ⚔️ STR +3
─────────────────────────────────────────────
```

### The Intuition

RAG is only as good as your chunks. If you chunk too small, you lose context. If you chunk too big, you waste tokens. If you chunk at sentence boundaries, you break thoughts.

Chunking is the most underrated skill in RAG engineering.

### Sprint 1 (90 min) — Build First

```
☀️ BUILD FIRST
───────────────────────────────────────────
  Time-box: 90 min. Build a chunking library
  from scratch.

  Implement 3 chunking methods:
  1. Character-based: split every N characters with overlap
  2. Recursive: split on paragraph → sentence → word boundaries
  3. Semantic: split when topic changes (use embedding similarity)

  STARTER:
  ┌──────────────────────────────────────┐
  │ def chunk_character(text, size=1000, │
  │                   overlap=200):      │
  │     chunks = []                      │
  │     start = 0                        │
  │     while start < len(text):        │
  │         end = start + size           │
  │         chunks.append(               │
  │             text[start:end])         │
  │         start += size - overlap      │
  │     return chunks                    │
  └──────────────────────────────────────┘

  PREDICTION before running:
  "For a 5000-word document, recursive chunking
  will produce ___ chunks because..."
  ┌──────────────────────────────────────┐
  │                                      │
  └──────────────────────────────────────┘

  GO → 90:00
```

### Sprint 2 — Deep Dive

```
🌅 DEEP DIVE
───────────────────────────────────────────
  Read: `04-RAG-Foundations/02-Chunking-Strategies.md`

  THE TWIST:
  Compare all 3 chunking methods on the SAME document.
  For each method, measure:
  • Number of chunks
  • Average chunk length
  • How often a single concept is split across chunks

  ┌──────────────────────────────────────┐
  │ Method    │ Chunks │ Avg Len │ Splits│
  │──────────│────────│─────────│───────│
  │ Character│        │         │       │
  │ Recursive│        │         │       │
  │ Semantic │        │         │       │
  └──────────────────────────────────────┘

  WRITE YOUR FINDING:
  "I would use ___ chunking for a production RAG because..."
  ┌──────────────────────────────────────┐
  │                                      │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] 3 chunking methods implemented
- [ ] Comparison table completed
- [ ] I understand the tradeoffs of each method
- [ ] Can recommend a strategy for any document type

---

## 🏰 DUNGEON 4.3: Retrieval Deep Dive

```
[SYSTEM] Entering Dungeon 4.3...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   2 sprints (1 day)
  XP:          +250
  Stats:       ⚔️ STR +5 | 💨 AGI +3
─────────────────────────────────────────────
```

### Sprint 1 (90 min)

```
☀️ BUILD IT
───────────────────────────────────────────
  Time-box: 90 min. Implement retrieval from scratch.

  Requirements:
  • Build a simple VectorStore class
  • Methods: add(doc, embedding), search(query_emb, k)
  • Implement cosine similarity search
  • Return documents + similarity scores

  Read: `04-RAG-Foundations/03-Retrieval-Deep-Dive.md`

  PREDICTION:
  "When I search for a query, the top result will
  have a similarity score of ___ if..."
  ┌──────────────────────────────────────┐
  │                                      │
  └──────────────────────────────────────┘

  GO → 90:00
```

### Sprint 2 — The Twist

```
🌅 THE TWIST
───────────────────────────────────────────
  Add HYBRID RETRIEVAL to your VectorStore:
  1. Dense retrieval (embedding similarity)
  2. Sparse retrieval (keyword/BM25)
  3. Hybrid: weighted combination

  Compare precision@5 for dense vs sparse vs hybrid:
  ┌──────────────────────────────────────┐
  │ Method  │ Precision@5 │ Recall@10    │
  │─────────│─────────────│──────────────│
  │ Dense   │             │              │
  │ Sparse  │             │              │
  │ Hybrid  │             │              │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] VectorStore class works end-to-end
- [ ] Hybrid retrieval implemented
- [ ] Performance comparison completed
- [ ] I understand when dense beats sparse and vice versa

---

## 🏰 DUNGEON 4.4: Context Integration

```
[SYSTEM] Entering Dungeon 4.4...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   1 sprint (90 min)
  XP:          +150
  Stats:       🧠 INT +3 | 👁️ PER +3
─────────────────────────────────────────────
```

### Sprint 1 (90 min)

```
☀️ BUILD IT
───────────────────────────────────────────
  Build a Context Window Manager.

  Requirements:
  • Takes retrieved chunks + user query
  • Manages token budget (e.g., max 4000 tokens)
  • Prioritizes most relevant chunks
  • Handles overflow when chunks exceed budget
  • Returns the assembled prompt

  Read: `04-RAG-Foundations/04-Context-Integration.md`

  THE TWIST:
  Implement "Lost in the Middle" mitigation:
  Put the MOST important chunk at the beginning AND end,
  less important ones in the middle.

  Test: Does reordering chunks change the answer quality?
  ┌──────────────────────────────────────┐
  │ Ordering │ Relevance Score │          │
  │──────────│─────────────────│          │
  │ Original │                 │          │
  │ Best-First-Last │          │          │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] Context window manager handles token budget
- [ ] Lost-in-the-middle mitigation implemented
- [ ] Tested reordering effect on answer quality

---

## 🏰 DUNGEON 4.5: RAG Failure Modes

```
[SYSTEM] Entering Dungeon 4.5...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   1 sprint (90 min)
  XP:          +150
  Stats:       👁️ PER +5
─────────────────────────────────────────────
```

### The Intuition

RAG fails in predictable ways. Knowing these failure modes is more important than knowing how RAG works — because you'll spend more time debugging than building.

### Sprint 1 (90 min)

```
☀️ THE MISSION
───────────────────────────────────────────
  Read: `04-RAG-Foundations/05-RAG-Failure-Modes.md`

  Then BUILD a RAG failure detector:
  • Given a query, retrieved docs, and LLM response
  • Detect these failure modes:
    - No relevant context (retrieval failed)
    - Context ignored (LLM didn't use retrieved docs)
    - Hallucination (LLM made up facts)
    - Conflicting context (chunks contradict each other)

  For each failure, return: detected? + confidence score.

  ┌──────────────────────────────────────┐
  │ Failure detected:                     │
  │ - No context: ☐                       │
  │ - Context ignored: ☐                  │
  │ - Hallucination: ☐                    │
  │ - Contradiction: ☐                    │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] I know the 4 main RAG failure modes
- [ ] Failure detector works on test cases
- [ ] I can diagnose why a RAG system produced a bad answer

---

## 🏰 DUNGEON 4.6: Evaluating RAG

```
[SYSTEM] Entering Dungeon 4.6...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   2 sprints (1 day)
  XP:          +200
  Stats:       👁️ PER +4 | 🧠 INT +3
─────────────────────────────────────────────
```

### Sprint 1 (90 min)

```
☀️ BUILD IT
───────────────────────────────────────────
  Time-box: 90 min. Build RAG evaluation pipeline.

  Requirements:
  • Create a test set of 10+ Q&A pairs
  • For each query, measure:
    - Context relevance: are retrieved docs relevant?
    - Answer faithfulness: does answer come from docs?
    - Answer relevance: does answer address the question?
  • Use LLM-as-judge for evaluation
  • Generate a report

  Read: `04-RAG-Foundations/06-Evaluating-RAG.md`

  GO → 90:00
```

### Sprint 2 — Deep Dive

```
🌅 THE TWIST
───────────────────────────────────────────
  Integrate RAGAS metrics:
  • Faithfulness
  • Answer Relevance
  • Context Precision
  • Context Recall

  Compare your LLM-as-judge scores vs RAGAS scores.
  Where do they disagree?
  ┌──────────────────────────────────────┐
  │ Query │ My Judge │ RAGAS │ Agree?    │
  │───────│──────────│───────│───────────│
  │       │          │       │           │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] Evaluation pipeline works end-to-end
- [ ] RAGAS metrics computed
- [ ] I understand the difference between evaluation methods

---

## 🏰 DUNGEON 4.7: Production RAG

```
[SYSTEM] Entering Dungeon 4.7...
─────────────────────────────────────────────
  Difficulty:  C
  Est. Time:   2 sprints (1 day)
  XP:          +200
  Stats:       🛡️ END +5 | ⚔️ STR +4
─────────────────────────────────────────────
```

### Sprint 1 (90 min)

```
☀️ BUILD IT
───────────────────────────────────────────
  Build a production RAG pipeline that includes:
  • Caching (cache query results to avoid re-embedding)
  • Fallback logic (if Qdrant is down, use ChromaDB)
  • Monitoring (log latency, token usage, retrieval stats)
  • Error handling (graceful degradation)

  Read: `04-RAG-Foundations/07-Production-RAG.md`

  GO → 90:00
```

### Sprint 2

```
🌅 THE TWIST
───────────────────────────────────────────
  Simulate failures:
  1. Kill Qdrant → does the fallback work?
  2. Send bad queries → does it fail gracefully?
  3. Cache miss → does caching improve latency?

  Document each scenario:
  ┌──────────────────────────────────────┐
  │ Failure Test │ Result │ Fix Needed? │
  │──────────────│────────│─────────────│
  │ Qdrant down  │        │             │
  │ Bad query    │        │             │
  │ Cache test   │        │             │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] Production RAG pipeline built
- [ ] Caching works and improves latency
- [ ] Fallbacks work when primary fails
- [ ] Monitoring data is logged

---

## 👑 BOSS BATTLE: Production Customer Support Bot

```
████████████████████████████████████████████
  BOSS BATTLE — CUSTOMER SUPPORT BOT
  Difficulty:  C
  XP:          +500
  Stats:       🛡️ END +8 | ⚔️ STR +6 | 🧠 INT +4
  Skill:       🛡️ RAG Barrier → Lv. 3
  Unlocks:     Gate 5: The Sage's Sanctum
████████████████████████████████████████████
```

### The Mission

Build a production customer support bot that answers questions about your product documentation using RAG. This is your most production-ready project yet.

**Tech Stack:** FastAPI + Qdrant + OpenAI + RAG pipeline

**Day 1:** Data ingestion — chunk product docs, embed, store in Qdrant
**Day 2:** Search + retrieval — hybrid search, reranking
**Day 3:** Answer generation — context integration, prompt engineering
**Day 4:** Evaluation — build test set, measure RAGAS metrics
**Day 5:** Production polish — caching, fallbacks, error handling, monitoring
**Day 6:** Documentation + Gate Check

### Gate Pass Requirements

- [ ] Ingests documentation and answers questions about it
- [ ] Hybrid search (dense + sparse) implemented
- [ ] Reranking improves result quality
- [ ] Caching reduces latency
- [ ] Graceful failure handling (no crash on missing docs)
- [ ] RAGAS evaluation scores reported
- [ ] README with architecture + deployment guide
- [ ] I typed every line (no copy-paste)

### Shadow Extraction

```
BOSS SHADOW: Production Customer Support Bot
───────────────────────────────────────────
Explain RAG pipeline architecture end-to-end:
_________________________________________
_________________________________________
_________________________________________
```

---

## Rewards

```
═══════════════════════════════════════════
  GATE 4 CLEARED — THE ARCHIVE

  Rewards:
  • +~3,000 XP total
  • Stats: ⚔️ STR +22 | 🧠 INT +21
            👁️ PER +16 | 🛡️ END +13 | 💨 AGI +3
  • Skills: 🛡️ RAG Barrier (Lv. 3)
  • Shadow Army: +8 new soldiers
  • Unlocked: Gate 5 — The Sage's Sanctum

  You've mastered the basics of RAG.
  Now it's time to push beyond.
═══════════════════════════════════════════
```

---

*Next: [GATE-05-Sages-Sanctum.md](./GATE-05-Sages-Sanctum.md)*
