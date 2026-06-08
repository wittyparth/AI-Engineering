# ⚜️ GATE 5: THE SAGE'S SANCTUM

```
[SYSTEM] ─────────────────────────────────────
  Gate 5 is opening.

  "You've learned to retrieve.
   Now learn to understand.
   The Sage's Sanctum holds techniques
   that turn good RAG into great RAG."

  ─────────────────────────────────────

  Rank Required:  B-5 or higher
  Dungeons:       7
  Boss Battle:    1 (Advanced RAG Engine)
  Est. Time:      16-18 days
  XP Available:   ~3,200
  Skill Unlock:   🗡️ HyDE Pierce
  Stat Focus:     💨 Agility · 👁️ Perception
─────────────────────────────────────────────
```

---

## The Gate

In Solo Leveling, the Sage's Sanctum was where Jinwoo learned to combine his abilities — not just using them one at a time, but weaving them into techniques greater than the sum of their parts.

This is your advanced RAG training. Everything you've built so far — embeddings, vector search, chunking, basic RAG — now you learn the techniques that separate production systems from prototypes.

---

## 🏰 DUNGEON 5.1: HyDE & Query Transformation

```
[SYSTEM] Entering Dungeon 5.1...
─────────────────────────────────────────────
  Difficulty:  C
  Est. Time:   2 sprints (1 day)
  XP:          +200
  Stats:       💨 AGI +4 | 🧠 INT +3
─────────────────────────────────────────────
```

### The Intuition

HyDE (Hypothetical Document Embeddings) is a clever trick: instead of searching with the user's query, ask an LLM to generate a "hypothetical perfect document" that would answer the query, then search with THAT document's embedding.

Why? Because the query "Tell me about Python async" is far from the document "Python async uses asyncio and await keywords." But a hypothetical document about async Python would be much closer.

### Sprint 1 (90 min) — Build First

```
☀️ BUILD FIRST
───────────────────────────────────────────
  Time-box: 90 min. Implement HyDE from scratch.

  Requirements:
  • Take a user query
  • Ask the LLM to generate a hypothetical document
    (not an answer — a document that would contain the answer)
  • Embed the hypothetical document
  • Search with the hypothetical embedding
  • Compare results with direct query search

  STARTER:
  ┌──────────────────────────────────────┐
  │ def hyde_search(query, k=5):        │
  │     # Step 1: Generate hypothetical │
  │     hyp_doc = llm.generate(          │
  │         f"Write a paragraph that    │
  │          would answer: {query}"     │
  │     )                               │
  │     # Step 2: Embed the hypothesis  │
  │     hyp_embed = embed(hyp_doc)     │
  │     # Step 3: Search                │
  │     return vector_store.search(     │
  │         hyp_embed, k)               │
  └──────────────────────────────────────┘

  PREDICTION:
  "HyDE will improve recall by ___% because..."
  ┌──────────────────────────────────────┐
  │                                      │
  └──────────────────────────────────────┘

  GO → 90:00
```

### Sprint 2 — Deep Dive

```
🌅 DEEP DIVE
───────────────────────────────────────────
  Read: `05-Advanced-RAG/01-HyDE-Query-Transformation.md`

  THE TWIST:
  Implement 3 query transformation strategies:
  1. HyDE (hypothetical document)
  2. Query expansion (generate 3 query variants)
  3. Step-back queries (generalize: "What is X?" → "What are the principles of X?")

  Compare all 3 on the same test set:
  ┌──────────────────────────────────────┐
  │ Strategy     │ Recall@5 │ Latency    │
  │──────────────│──────────│────────────│
  │ Direct query │          │            │
  │ HyDE         │          │            │
  │ Expansion    │          │            │
  │ Step-back    │          │            │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] HyDE implemented and working
- [ ] Query expansion implemented
- [ ] Step-back queries implemented
- [ ] I understand WHEN to use each strategy

---

## 🏰 DUNGEON 5.2: Contextual Retrieval

```
[SYSTEM] Entering Dungeon 5.2...
─────────────────────────────────────────────
  Difficulty:  C
  Est. Time:   2 sprints (1 day)
  XP:          +250
  Stats:       🧠 INT +4 | ⚔️ STR +4
─────────────────────────────────────────────
```

### Sprint 1 (90 min)

```
☀️ BUILD IT
───────────────────────────────────────────
  Build contextual retrieval pipeline.

  The problem: chunks lose context. A chunk that says
  "she pressed the button" has no meaning if we don't
  know the previous chunk was about "Sarah standing
  at the control panel."

  Requirements:
  • Implement contextual chunking: add surrounding context
    to each chunk (previous N sentences, section header, doc title)
  • Compare: plain chunk vs context-enriched chunk retrieval
  • Measure: does context enrichment improve retrieval accuracy?

  Read: `05-Advanced-RAG/02-Contextual-Retrieval.md`

  GO → 90:00
```

### Sprint 2

```
🌅 THE TWIST
───────────────────────────────────────────
  Add windowing to your contextual retrieval:
  • Sliding window of 3 chunks with overlap
  • Each search returns the best window, not just the best chunk
  • This gives the LLM more context to work with

  Compare single-chunk vs window retrieval:
  ┌──────────────────────────────────────┐
  │ Method        │ Faithfulness │        │
  │───────────────│──────────────│        │
  │ Single chunk  │              │        │
  │ Window (3)    │              │        │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] Contextual chunking implemented
- [ ] Sliding window retrieval works
- [ ] Measured improvement in faithfulness
- [ ] I understand when context enrichment helps

---

## 🏰 DUNGEON 5.3: Reranking Deep Dive

```
[SYSTEM] Entering Dungeon 5.3...
─────────────────────────────────────────────
  Difficulty:  C
  Est. Time:   2 sprints (1 day)
  XP:          +250
  Stats:       👁️ PER +5 | 💨 AGI +4
─────────────────────────────────────────────
```

### The Intuition

Initial retrieval gives you a broad set of candidates. Reranking takes those candidates and re-orders them with a more expensive but more accurate model. This is the "two-stage retrieval" pattern used in production.

### Sprint 1 (90 min) — Build First

```
☀️ BUILD FIRST
───────────────────────────────────────────
  Time-box: 90 min. Implement cross-encoder
  reranking from scratch.

  NO libraries. Pure Python + API calls.

  Requirements:
  • Take 10 candidate documents from initial search
  • For each candidate, compute relevance score:
    score = llm.rate_relevance(query, document)
    (Use a simple prompt that returns a number 0-10)
  • Re-rank candidates by score
  • Return top-3 after reranking

  PREDICTION:
  "Reranking will change the top-3 results by ___% because..."
  ┌──────────────────────────────────────┐
  │                                      │
  └──────────────────────────────────────┘

  GO → 90:00
```

### Sprint 2 — Deep Dive

```
🌅 DEEP DIVE
───────────────────────────────────────────
  Read: `05-Advanced-RAG/03-Reranking-Deep-Dive.md`

  THE TWIST:
  Compare pre-retrieval vs post-retrieval reranking:
  • Pre-retrieval: rerank queries (query expansion)
  • Post-retrieval: rerank retrieved documents (cross-encoder)

  When does each make sense?
  ┌──────────────────────────────────────┐
  │ Pre-retrieval works best when:       │
  │                                      │
  │ Post-retrieval works best when:      │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] Cross-encoder reranking implemented
- [ ] Pre vs post retrieval comparison done
- [ ] I understand the cost-quality tradeoff of reranking
- [ ] Measured improvement in top-3 accuracy

---

## 🏰 DUNGEON 5.4: Agentic RAG

```
[SYSTEM] Entering Dungeon 5.4...
─────────────────────────────────────────────
  Difficulty:  C
  Est. Time:   2 sprints (1 day)
  XP:          +300
  Stats:       ⚔️ STR +5 | 🧠 INT +5
─────────────────────────────────────────────
```

### The Intuition

Basic RAG: always retrieve, then generate. Agentic RAG: the model DECIDES whether to retrieve, what to search for, and whether the results are sufficient.

### Sprint 1 (90 min)

```
☀️ BUILD IT
───────────────────────────────────────────
  Build an agent that decides when to retrieve.

  Requirements:
  • The agent has access to a search tool
  • For each query, it decides:
    - "I can answer this without searching" → answer directly
    - "I need more information" → search
    - "The search results aren't sufficient" → re-phrase and search again
  • Max 3 search iterations

  Read: `05-Advanced-RAG/04-Agentic-RAG.md`

  GO → 90:00
```

### Sprint 2

```
🌅 THE TWIST
───────────────────────────────────────────
  Add tool selection: the agent can choose between
  3 different search strategies:
  1. Quick search (top-3, cheap embedding)
  2. Deep search (top-10, expensive embedding, reranking)
  3. Web search (simulated — return "I'd search the web")

  Log the agent's decisions:
  ┌──────────────────────────────────────┐
  │ Query │ Strategy │ Iterations │ Success │
  │───────│──────────│────────────│─────────│
  │       │          │            │         │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] Agent decides when to retrieve
- [ ] Multi-tool search strategy works
- [ ] Agent handles insufficient results gracefully
- [ ] I understand when agentic RAG is worth the complexity

---

## 🏰 DUNGEON 5.5: Multimodal RAG

```
[SYSTEM] Entering Dungeon 5.5...
─────────────────────────────────────────────
  Difficulty:  C
  Est. Time:   2 sprints (1 day)
  XP:          +300
  Stats:       💨 AGI +5 | 👁️ PER +4
─────────────────────────────────────────────
```

### Sprint 1 (90 min)

```
☀️ BUILD IT
───────────────────────────────────────────
  Build multimodal RAG (text + image retrieval).

  Requirements:
  • Create a document set with mixed content
    (paragraphs with diagrams, screenshots, charts)
  • For each document, extract both text and image descriptions
  • Embed text descriptions + image captions together
  • Retrieve relevant chunks for a text query
  • Return: relevant text + relevant images

  Read: `05-Advanced-RAG/05-Multimodal-RAG.md`

  GO → 90:00
```

### Sprint 2

```
🌅 THE TWIST
───────────────────────────────────────────
  Handle documents with mixed content:
  • A document has text, a diagram, and a table
  • Query: "What's the architecture shown in the diagram?"
  • Your RAG should find the image even though the query
    is about visual content

  Test with 5 queries about visual content:
  ┌──────────────────────────────────────┐
  │ Query │ Found Image? │ Relevance │    │
  │───────│──────────────│───────────│    │
  │       │              │           │    │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] Multimodal RAG works for text+image
- [ ] Mixed content handling works
- [ ] I understand how to handle different media types

---

## 🏰 DUNGEON 5.6: GraphRAG

```
[SYSTEM] Entering Dungeon 5.6...
─────────────────────────────────────────────
  Difficulty:  C
  Est. Time:   2 sprints (1 day)
  XP:          +300
  Stats:       🧠 INT +6 | ⚔️ STR +4
─────────────────────────────────────────────
```

### Sprint 1 (90 min)

```
☀️ BUILD IT
───────────────────────────────────────────
  Build a simple GraphRAG implementation.

  Requirements:
  • Extract entities from documents (people, places, concepts)
  • Extract relationships between entities
  • Build a knowledge graph (start with NetworkX)
  • For a query, find relevant entities + traverse relationships
  • Return context: entity descriptions + connected entities

  Read: `05-Advanced-RAG/06-GraphRAG.md`

  GO → 90:00
```

### Sprint 2

```
🌅 THE TWIST
───────────────────────────────────────────
  Compare GraphRAG vs standard RAG on multi-hop queries:
  "Who worked with Person X at Company Y?"

  ┌──────────────────────────────────────┐
  │ Query Type │ Std RAG │ GraphRAG │    │
  │────────────│─────────│──────────│    │
  │ Simple     │         │          │    │
  │ Multi-hop  │         │          │    │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] Knowledge graph built from documents
- [ ] Graph traversal search works
- [ ] Compared with standard RAG on multi-hop queries

---

## 🏰 DUNGEON 5.7: Advanced RAG Evaluation

```
[SYSTEM] Entering Dungeon 5.7...
─────────────────────────────────────────────
  Difficulty:  C
  Est. Time:   1 sprint (90 min)
  XP:          +150
  Stats:       👁️ PER +5 | 🧠 INT +3
─────────────────────────────────────────────
```

### Sprint 1 (90 min)

```
☀️ BUILD IT
───────────────────────────────────────────
  Build a comprehensive RAG evaluation suite.

  Requirements:
  • Test each RAG variant you've built in this gate:
    - Standard RAG (from Gate 4)
    - HyDE-enhanced RAG
    - Agentic RAG
    - GraphRAG
  • For each, measure: faithfulness, answer relevance, context precision
  • Generate a comparison report

  Read: `05-Advanced-RAG/07-Advanced-Evaluation.md`

  ┌──────────────────────────────────────┐
  │ Variant    │ Faith │ Rel  │ Prec    │
  │───────────│───────│──────│─────────│
  │ Standard  │       │      │         │
  │ HyDE      │       │      │         │
  │ Agentic   │       │      │         │
  │ GraphRAG  │       │      │         │
  └──────────────────────────────────────┘

  GO → 90:00
```

### Gate Check
- [ ] All RAG variants evaluated on same test set
- [ ] Comparison report generated
- [ ] I know which variant to use for which problem

---

## 👑 BOSS BATTLE: Advanced RAG Engine

```
████████████████████████████████████████████
  BOSS BATTLE — ADVANCED RAG ENGINE
  Difficulty:  B
  XP:          +500
  Stats:       ⚔️ STR +8 | 💨 AGI +8 | 🧠 INT +4
  Skill:       🗡️ HyDE Pierce → Lv. 3
  Unlocks:     Gate 6: The Conductor's Hall
████████████████████████████████████████████
```

### The Mission

Build an Advanced RAG Engine that combines everything from this gate: HyDE, query expansion, contextual retrieval, reranking, and agentic retrieval — with a comprehensive evaluation suite.

### Project Plan

**Day 1:** Core architecture — modular RAG pipeline with pluggable strategies
**Day 2:** Query transformation module — HyDE, expansion, step-back
**Day 3:** Retrieval + reranking — dense, sparse, hybrid, cross-encoder
**Day 4:** Agentic RAG — decide when/what to retrieve
**Day 5:** Evaluation suite — compare all combinations
**Day 6:** Documentation + Gate Check

### Gate Pass Requirements

- [ ] Modular RAG pipeline with pluggable strategies
- [ ] HyDE, query expansion, step-back implemented
- [ ] Reranking improves top-3 accuracy
- [ ] Agentic RAG works end-to-end
- [ ] Evaluation suite compares all variants
- [ ] Recommendation: which strategy for which use case
- [ ] I typed every line (no copy-paste)

### Shadow Extraction

```
BOSS SHADOW: Advanced RAG Engine
───────────────────────────────────────────
Explain the differences between standard, HyDE, agentic, and GraphRAG:
_________________________________________
_________________________________________
_________________________________________
```

---

## Rewards

```
═══════════════════════════════════════════
  GATE 5 CLEARED — THE SAGE'S SANCTUM

  Rewards:
  • +~3,200 XP total
  • Stats: ⚔️ STR +26 | 🧠 INT +27
            👁️ PER +14 | 🛡️ END +0  | 💨 AGI +17
  • Skills: 🗡️ HyDE Pierce (Lv. 3)
  • Shadow Army: +8 new soldiers
  • Unlocked: Gate 6 — The Conductor's Hall

  Advanced RAG is mastered.
  Now learn to command armies of agents.
═══════════════════════════════════════════
```

---

*Next: [GATE-06-Conductors-Hall.md](./GATE-06-Conductors-Hall.md)*
