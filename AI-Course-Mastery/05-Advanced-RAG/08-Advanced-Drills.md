# 08 — Advanced Drills: Phase 5 Challenge Suite

## 🎯 Purpose

Phase 5 taught you 6 advanced RAG techniques:
1. **HyDE & Query Transformation** — rewriting queries for better retrieval
2. **Contextual Retrieval** — enriching chunks with surrounding context
3. **Reranking** — using cross-encoders to boost precision
4. **Agentic RAG** — multi-hop, self-query, corrective retrieval
5. **Multimodal RAG** — images, diagrams, vision LLMs
6. **GraphRAG** — knowledge graphs, entity traversal, community summaries

This module presents **8 challenges** that combine these techniques in production scenarios. Each challenge is harder than the last.

**How these work:**
- Each challenge gives you a FAILING system with a specific problem
- You must diagnose the problem, pick the right Phase 5 technique(s), and fix it
- No step-by-step instructions — you decide the approach
- Each challenge lists the techniques you'll likely need, but the COMBINATION is up to you

---

## ⏱ Time Budget

| Challenge | Estimated Time | Techniques Used |
|-----------|---------------|-----------------|
| 1. The Vocabulary Mismatch Fix | 35 min | HyDE, query expansion |
| 2. The Lost Context Recovery | 35 min | Contextual retrieval, sentence-window |
| 3. The Reranking Tuning Puzzle | 40 min | Cross-encoder, threshold optimization |
| 4. The Agent Debugging Nightmare | 50 min | Agentic RAG, CRAG, gap analysis |
| 5. The Screenshot That Can't Be Found | 45 min | Multimodal RAG, CLIP, vision LLM |
| 6. The Mis-merged Entity Disaster | 45 min | GraphRAG, entity resolution |
| 7. The Cost-Quality Crisis | 40 min | All techniques, tiered routing |
| 8. The Full-Stack Post-Mortem | 60 min | All techniques, evaluation, diagnosis |
| **Bonus:** The Zero-to-One Build | 120 min | Full pipeline, all techniques |

---

## Challenge 1: The Vocabulary Mismatch Fix

### Scenario

Your customer support bot handles questions about a SaaS product. The knowledge base uses formal technical language. Users ask in casual language.

**Your current system:** Standard RAG with `text-embedding-3-small` and `gpt-4o-mini`.

**The failing query:** *"How do I get my money back if I don't like the product?"*

**Expected correct answer:** *"Full refunds are available within 30 days of purchase. Contact support with your order ID to initiate the refund process. Enterprise customers have a 60-day refund window."*

**What actually happens:** The retriever finds chunks about "payment methods accepted" and "subscription billing cycles" — neither relevant to refunds. The system answers with generic information about payment processing.

**The knowledge base contains (but doesn't retrieve):**
- "Refund Policy: Customers may request a full refund within 30 days of initial purchase. Enterprise agreements have a 60-day evaluation period with full refund eligibility."
- "To initiate a refund: Contact customer support via the help center with your order ID. Refunds are processed within 5-7 business days."

### Your Mission

The root problem: the user's query uses "get my money back" and "don't like the product" — vocabulary that doesn't exist in the knowledge base (which uses "refund" and "evaluation period").

**Fix this using HyDE + query expansion.**

### Success Criteria

1. The query "How do I get my money back if I don't like the product?" retrieves the refund policy chunk in the top-3
2. Standard retrieval (without HyDE) fails for the same query — show the comparison
3. Your solution handles at least 3 other vocabulary-mismatch queries (invent them based on the knowledge base)
4. You measure the lift: recall@5 with vs. without HyDE

### Debugging Hints

- Run `SelfQueryBenchmark.compare()` to measure the difference
- Try generating 3 hypothetical documents (not 1) and merging with RRF
- Check if `hyde_weight = 0.7` is optimal for your data
- The failure pattern: high query perplexity (the query uses words the embedding model rarely sees together)

---

## Challenge 2: The Lost Context Recovery

### Scenario

Your document chunker splits at 512-token boundaries. One of your chunks reads:

> *"The rate limit is 1000 requests per hour. If you exceed this limit, your requests will be throttled with a 429 status code. For..."*

The next chunk starts with:

> *"... enterprise customers, the limit is 10000 requests per hour. Contact your account manager to request a higher limit."*

**The failing query:** *"What's the rate limit for enterprise customers?"*

**What happens:** The second chunk starts with "...enterprise customers, the limit is 10000..." — the embedding for this chunk emphasizes "enterprise" and "limit" but the prefix "..." and dangling context make it rank lower than the first chunk (which has cleaner language about "rate limit"). The system retrieves the FREE TIER chunk and gives the wrong answer.

### Your Mission

Fix this using **contextual chunk enrichment** and/or **sentence-window retrieval**.

### The Data

```python
chunks = [
    {"id": "c1", "text": "The rate limit is 1000 requests per hour. If you exceed this limit, your requests will be throttled with a 429 status code. For...", "source": "docs/pricing.md"},
    {"id": "c2", "text": "...enterprise customers, the limit is 10000 requests per hour. Contact your account manager to request a higher limit.", "source": "docs/pricing.md"},
    # ... 50 more chunks from the same document
]
```

### Success Criteria

1. Before fix: query "enterprise rate limit" returns c1 (wrong tier) or c2 with low score
2. After fix: query returns c2 with the "enterprise" context preserved → LLM gives correct answer
3. The fix does NOT break standard queries ("what's the free tier limit?" should still work)
4. You measure precision improvement on a set of 10 tier-ambiguous queries

### Hints

- Use the `ContextualChunkEnricher` pattern from File 02
- Try prepending "Section: Pricing, Tier: Enterprise" before embedding c2
- Test both prepended context AND sentence-window retrieval
- The most robust fix: combine BOTH enrichment AND window retrieval

---

## Challenge 3: The Reranking Tuning Puzzle

### Scenario

Your two-stage retrieval system:
1. Stage 1 (bi-encoder): Retrieve top-20 chunks using `text-embedding-3-small`
2. Stage 2 (cross-encoder): Rerank top-20 using `cross-encoder/ms-marco-MiniLM-L-6-v2`

**The problem:** Reranking is NOT improving your results. In fact, it's making them WORSE.

**Investigation findings:**
- Before reranking: recall@5 = 0.82, precision@5 = 0.45
- After reranking: recall@5 = 0.71, precision@5 = 0.43

Reranking improved precision by only 2% but DROPPED recall by 11%. The cross-encoder is pushing out relevant chunks and keeping irrelevant ones.

### Your Mission

Diagnose and fix the reranking pipeline.

### Debugging Steps (find the problem)

1. **Check score distributions:** Are the top-20 bi-encoder scores clustered tightly (all between 0.78-0.82)?
   - If YES: the bi-encoder can't distinguish quality, and the cross-encoder is working with a flat field
   - Solution: increase initial retrieval to 50+ to give the cross-encoder more diverse candidates

2. **Check relevance inversion:** For 5 random queries, manually inspect the top-3 from bi-encoder vs. cross-encoder
   - Is the cross-encoder promoting chunks with HIGHER lexical overlap but LOWER semantic relevance?
   - Cross-encoders trained on MS MARCO favor keyword-match answers over conceptual relevance

3. **Check cross-encoder model:** MS MARCO is a web search dataset. Your domain is technical documentation.
   - The model might be ranking "how-to" style chunks higher than "reference" style chunks
   - Try: fine-tune or use a domain-specific cross-encoder

### Success Criteria

1. After fix: reranking improves BOTH precision AND recall over bi-encoder baseline
2. precision@5 > 0.60 AND recall@5 > 0.80
3. You identify the root cause (from the 3 options above, or find your own)
4. You document: what was wrong, what you changed, and by how much metrics improved

### Production Reality

In production, this exact scenario happens all the time. Teams add reranking expecting a magic bullet, but the cross-encoder's training distribution doesn't match their domain. The fix is almost never "tune the model" — it's "understand what the model was trained on and adjust your pipeline accordingly."

---

## Challenge 4: The Agent Debugging Nightmare

### Scenario

Your agentic RAG system with CRAG (Corrective RAG) is misbehaving. It has three bugs:

**Bug A — Infinite Retrieval Loop:**
```
Query: "Tell me about the company's products"
→ Retrieve: 3 product chunks
→ Evaluate: "Need more information about pricing"
→ Retrieve: 2 pricing chunks
→ Evaluate: "Need more information about support"
→ Retrieve: 1 support chunk
→ Evaluate: "Need more information about..."
→ (loops until max iterations, returns a generic answer)
```

**Bug B — CRAG False Positives:**
```
Query: "What's the API key format?"
→ Retrieved chunk: "API keys follow the format sk-... (starts with sk_)"
→ Relevance evaluator: "IRRELEVANT (confidence: 0.85)"
→ Refines query, retrieves again, gets the SAME chunk, rejects it again
→ Falls back: "I cannot find information about API key format"
```

The evaluation says this chunk is irrelevant because it doesn't contain the EXACT phrase "API key format" — it contains "format" in a different syntactic structure.

**Bug C — Gap Analysis Circularity:**
```
Query: "Who manages the engineering team and what's their background?"
→ Hop 1: Find "engineering team manager" → finds "VP of Engineering: Jane Doe"
→ Gap analysis: "I know the manager but not their background. Need background."
→ Hop 2: Find "Jane Doe background" → finds "Jane Doe, 15 years experience"
→ Gap analysis: "I know the background but need to confirm she manages engineering"
→ Hop 3: Find "Jane Doe manages engineering" → finds same chunk as Hop 1
→ Gap analysis: "Same as before. Need..."
→ (circular — keeps re-finding the same information)
```

### Your Mission

Fix ALL THREE bugs. You can modify the gap analyzer, relevance evaluator, and stopping conditions.

### For Each Bug, Provide

1. **Root cause diagnosis:** Why is this happening? (1-2 sentences)
2. **The fix:** What code change fixes it? (Show the code)
3. **Verification:** Run the same query and show it no longer loops/rejects incorrectly

### Hints

- **Bug A fix:** Add a diminishing-returns check. If the last N retrievals added < M% new information, stop.
- **Bug B fix:** The relevance evaluator is too strict. Add a "fuzzy relevance" mode that checks semantic similarity, not exact phrase match.
- **Bug C fix:** Track which information has already been found. Before adding a "gap," check if the gap was already filled by a previous hop. If the same gap reappears 3 times, force-generate.

---

## Challenge 5: The Screenshot That Can't Be Found

### Scenario

You have a bug tracker with 2,000 screenshots embedded in bug reports. Each screenshot shows an error message. The error messages contain text like "Error 0x80070005 — Access Denied" and "Connection timeout after 30 seconds."

**The failing query:** A user asks in text: *"What does error 0x80070005 mean and how do I fix it?"*

**What happens:** Standard text-only RAG can't find the screenshots because:
- The screenshots' OCR text was never extracted
- The surrounding text says "See screenshot" — useless for retrieval
- No text chunk contains the error code

**What should happen:** The system should find the screenshot containing "Error 0x80070005," extract the error code via OCR/vision, find the associated text (the fix), and answer.

### Your Mission

Build a multimodal retrieval pipeline that:
1. Indexes all screenshots with OCR extraction
2. CLIP-embeds screenshots for visual similarity search
3. For the failing query: retrieves the correct screenshot, reads the error code, finds the fix text
4. Returns: "Error 0x80070005 is an 'Access Denied' error. Fix: [from the associated text]"

### The Data

Create 5 mock bug reports:

```python
bug_reports = [
    {
        "id": "BUG-001",
        "text": "User reports cannot access admin panel. See screenshot. This is affecting all admin users.",
        "screenshot_path": "screenshots/bug001_error.png",
        "screenshot_text": "Error 0x80070005\nAccess Denied\nYou do not have permission to access this resource.",
        "fix": "The user's role was set to 'viewer' instead of 'admin'. Reset role via admin console > Users > [user] > Role > Admin."
    },
    # ... create 4 more similar
]
```

### Success Criteria

1. The text query "Error 0x80070005" correctly retrieves the screenshot (via CLIP or OCR text embedding)
2. The vision pipeline extracts the error code from the screenshot
3. The system finds the associated fix text
4. The final answer correctly states the error AND the fix
5. You show what happens WITHOUT multimodal retrieval (text-only fails)

### Extension

Make it harder: what if the user query is ALSO a screenshot? (They upload a photo of the error on their phone.) Now you need image-to-image retrieval — find screenshots that look SIMILAR to the query image using CLIP visual similarity.

---

## Challenge 6: The Mis-merged Entity Disaster

### Scenario

Your GraphRAG system processes 1,000 documents about Acme Corp. The entity extractor is over-eager and has made two critical errors:

**Error 1 — False Merge:**
- Entity "Apple" (the fruit/tech company mentioned in a supplier agreement)
- Entity "Apple" (the internal codename for Project Eden, a top-secret project)
- Both were merged into one node "apple"

Now every query about Apple supplies ALSO returns Project Eden documents. This is a security breach risk.

**Error 2 — False Split:**
- "Dr. Sarah Chen" (formal)
- "Sarah Chen" (informal)
- "Dr. Chen" (abbreviated)
- These are the SAME person but stored as 3 separate nodes

Traversal from "Dr. Chen" doesn't reach "Dr. Sarah Chen's" relationships (published papers, team membership).

### Your Mission

1. **Detect both errors** using graph statistics and query analysis
2. **Split Error 1:** Separate "Apple (Supplier)" from "Apple (Codename)" — reassign relationships correctly
3. **Merge Error 2:** Merge Dr. Sarah Chen's 3 nodes — deduplicate relationships
4. **Prevent recurrence:** Add context-aware disambiguation rules

### Graph State After Fixes

```python
# Error 1 fixed: two separate nodes
apple_supplier = Entity(name="Apple Inc.", type="ORGANIZATION", description="Technology company, hardware supplier")
apple_codename = Entity(name="Project Eden (Codename: Apple)", type="PROJECT", description="Internal AI research project")

# Error 2 fixed: one canonical node
sarah_chen = Entity(name="Dr. Sarah Chen", type="PERSON", aliases=["Sarah Chen", "Dr. Chen", "S. Chen"])
```

### Success Criteria

1. Query: "Find documents about Apple supplies" — only returns supplier docs, NOT Project Eden docs
2. Query: "What papers did Dr. Chen publish?" — returns ALL of Sarah Chen's papers (from all 3 aliases)
3. Entity resolution accuracy improves from estimated 60% to 90%+
4. You add automatic MERGE DETECTION: if a query retrieves unexpectedly diverse results, flag for review

---

## Challenge 7: The Cost-Quality Crisis

### Scenario

Your Phase 5 RAG system has ALL features enabled:
- HyDE query transformation
- Cross-encoder reranking
- Agentic multi-hop retrieval
- Multimodal vision pipeline
- GraphRAG entity traversal

**The problem:** Your monthly bill is $4,800 for 50,000 queries. Your CTO wants it under $1,500.

**Usage breakdown:**
| Component | Cost/Query | Queries/Day | Monthly Cost |
|-----------|-----------|-------------|--------------|
| HyDE (gpt-4o-mini) | $0.0013 | 50,000 | $1,950 |
| Cross-encoder reranking | $0.0001 | 50,000 | $150 |
| Agentic multi-hop (avg 3 hops) | $0.0030 | 15,000 (30%) | $1,350 |
| Multimodal vision (GPT-4o) | $0.012 | 2,000 (4%) | $720 |
| GraphRAG entity extraction | $0.002 | 50,000 | $300 |
| **Total** | | | **$4,470** |

Your CTO says:
- "80% of queries are simple PDF questions — WHY are they using HyDE?"
- "4% of queries have images — WHY are all queries going through vision?"
- "Multi-hop should only trigger when needed — measure how often it actually helps"

### Your Mission

Design a **tiered routing system** that reduces cost by 70%+ while maintaining 90%+ of the quality.

### Step 1: Measure usage patterns

Run an audit. For the last 10,000 queries, classify:
- How many are simple (1-chunk answers)? → Route to standard RAG
- How many need HyDE? → Route to HyDE
- How many need multi-hop? → Route to agentic
- How many have images? → Route to multimodal
- How many need graph traversal? → Route to GraphRAG

### Step 2: Design the routing table

| Query Type | % of Traffic | Strategy | Cost/Query | Est. Monthly |
|------------|-------------|----------|-----------|-------------|
| Simple fact | 55% | Standard RAG (no extras) | $0.0005 | $412 |
| Vocabulary mismatch | 20% | HyDE only | $0.0013 | $390 |
| Multi-document | 12% | Multi-hop (no HyDE, no graph) | $0.002 | $360 |
| Visual | 4% | Multimodal only (when triggered) | $0.012 | $720 |
| Relationship | 6% | GraphRAG only | $0.002 | $180 |
| Complex (all) | 3% | Full pipeline | $0.015 | $67 |
| **Total** | **100%** | | | **$2,129** |

Still over budget. Optimize further:

| Optimization | Savings | Remaining |
|-------------|---------|-----------|
| Tiered routing | -$2,341 | $2,129 |
| Batch HyDE (generate for 3 queries at once) | -$390 | $1,739 |
| Cache vision results (repeat queries) | -$360 (50% cache hit) | $1,379 |
| Reduce GraphRAG frequency (re-extract weekly) | -$120 | $1,259 |
| Use GPT-4o-mini for simple multi-hop | -$180 | $1,079 |

**Final cost:** ~$1,079/month (77% reduction from $4,800)

### Success Criteria

1. Build the tiered router and measure cost savings
2. Run the SAME 100 test queries through old (all-features) and new (tiered) pipelines
3. Show quality comparison: within 5% of original quality at 77% lower cost
4. Identify WHICH queries lose quality in the tiered system and why

---

## Challenge 8: The Full-Stack Post-Mortem

### Scenario

Your Phase 5 Advanced RAG Engine went to production. After 2 weeks, you have this data:

**Performance metrics:**
- Global average: Faithfulness 0.87, Answer Relevancy 0.84
- But when broken down by query type:

| Query Category | Traffic % | Faithfulness | Issue |
|----------------|-----------|-------------|-------|
| Password/Account | 28% | 0.94 | ✅ |
| Billing/Pricing | 22% | 0.91 | ✅ |
| API/Integration | 18% | 0.88 | ⚠️ Slight drop |
| Troubleshooting | 15% | 0.62 | ❌ Major issue |
| Enterprise/Admin | 12% | 0.79 | ⚠️ Moderate |
| Visual/Screenshots | 5% | 0.45 | ❌ Critical |

**User feedback:**
- "The bot doesn't understand my troubleshooting scenario" — 45 complaints
- "It gave me wrong pricing for enterprise" — 12 complaints
- "It couldn't read the error in my screenshot" — 8 complaints
- "The answer sounded confident but was wrong" — 23 complaints

**System logs show:**
- Retrieval recall is 0.85 overall, but 0.55 for troubleshooting queries
- 34% of troubleshooting queries result in 0 chunks retrieved (empty retrieval)
- Average multi-hop depth: 1.3 hops (most queries stop after 1 hop)
- Vision LLM used for 12% of queries, but only 5% actually have images

### Your Mission

Conduct a full post-mortem:

1. **Diagnose the TROUBLESHOOTING collapse:**
   - Why is recall only 0.55? (Hint: check how troubleshooting docs are written vs. how users query them)
   - Why is empty retrieval at 34%? (Hint: check chunking strategy for troubleshooting docs)
   - Propose 2 specific fixes

2. **Diagnose the ENTERPRISE pricing issue:**
   - Faithfulness is 0.79 but complaints are about WRONG pricing, not vague pricing
   - The system retrieves the right chunks but generates the wrong answer
   - Is the generator ignoring the context? (check component isolation)
   - Is the context window too small? (check how far the relevant sentence is from chunk start)
   - Propose 2 specific fixes

3. **Diagnose the SCREENSHOT failure:**
   - Vision LLM used for 12% of queries but only 5% have images
   - 7% of queries are unnecessarily paying vision cost
   - The 5% that need vision are failing 55% of the time
   - Is the vision pipeline triggering too often AND performing poorly?
   - Propose 2 specific fixes

4. **Diagnose the SHALLOW multi-hop:**
   - Average depth: 1.3 hops. The gap analyzer is stopping too early.
   - How many multi-hop queries end with wrong answers because they only did 1 hop?
   - Is the gap analyzer's "confidence" threshold too high?
   - Propose 2 specific fixes

5. **The overall fix plan:**
   - Rank the 8 proposed fixes by impact
   - Estimate time and cost for each
   - Propose a 2-week sprint plan

### Success Criteria

Your post-mortem report must:
1. Identify ROOT CAUSES (not symptoms) for each issue
2. Propose SPECIFIC, CODABLE fixes
3. Rank fixes by expected impact
4. Include a measurement plan: how will you know the fix worked?
5. Show BEFORE and AFTER metrics for at least one fix

### Post-Mortem Template

```markdown
# Post-Mortem: Phase 5 RAG Engine — Week 2 Review

## Issue 1: Troubleshooting Collapse
- Symptom: Faithfulness 0.62, recall 0.55
- Root cause: [your diagnosis]
- Fix: [specific change]
- Expected impact: [estimated improvement]
- Verification: [how you'll measure]

## Issue 2: Enterprise Pricing Errors
- ...
```

---

## Bonus Challenge: The Zero-to-One Build (120 min)

### Scenario

You're starting a NEW project. No existing RAG system. You have a directory of 500 documents (mix of text, PDFs with diagrams, and screenshots).

### Your Mission

Build Phase 5's Advanced RAG Engine from scratch. The system should:

1. **Ingest all documents** — text chunks, images, tables
2. **Index with multiple strategies** — text embeddings, CLIP embeddings, knowledge graph
3. **Classify queries** — route to the right strategy
4. **Retrieve with the best strategy** — HyDE, multi-hop, graph traversal, or vision
5. **Generate with citations** — every claim attributed to a source document, chunk, or image
6. **Evaluate itself** — built-in evaluation harness for every answer
7. **Log everything** — latency, cost, scores, failures

### Minimum Viable Deliverables

```
advanced_rag_engine/
├── ingest/
│   ├── text_processor.py     # Chunk, embed, index text
│   ├── image_processor.py    # OCR, describe, CLIP-embed images
│   └── graph_builder.py      # Extract entities, build graph
├── retrieve/
│   ├── query_classifier.py   # Route to correct strategy
│   ├── hyde_retriever.py     # HyDE-enhanced retrieval
│   ├── multi_hop.py          # Multi-hop retrieval
│   ├── graph_traversal.py    # Graph-based retrieval
│   └── multimodal.py         # Vision-enhanced retrieval
├── generate/
│   └── generator.py          # Context builder + LLM generation
├── evaluate/
│   ├── evaluator.py          # Component-level + end-to-end
│   └── regression.py         # Regression test suite
└── engine.py                 # Main entry point
```

### Success Criteria

1. The engine answers all 10 Phase 5 scenario queries correctly
2. The engine costs < $0.005/query average
3. The engine's self-evaluation scores match manual evaluation within 10%
4. A README documents architecture decisions, strategy routing, and cost analysis

---

## 🚦 Gate Check

Before moving to File 09 (the Phase 5 project), confirm you have completed:

- [ ] Challenge 1 — HyDE fixes vocabulary mismatch
- [ ] Challenge 2 — Contextual retrieval fixes lost context
- [ ] Challenge 3 — Reranking tuned to improve both precision AND recall
- [ ] Challenge 4 — Agent/CREG bugs fixed (infinite loop, false positives, circular gaps)
- [ ] Challenge 5 — Multimodal retrieval finds screenshots via text and image queries
- [ ] Challenge 6 — Entity resolution errors detected, split, merged, and prevented
- [ ] Challenge 7 — Tiered routing reduces cost by 70%+ while maintaining 90%+ quality
- [ ] Challenge 8 — Full post-mortem with root causes, fixes, and sprint plan
- [ ] [Bonus] Zero-to-One engine built from scratch

---

## 📚 Resources

- Go back to any Phase 5 file for technique-specific help
- RAGAS documentation for evaluation: https://docs.ragas.io/
- NetworkX docs for graph operations: https://networkx.org/
- OpenCLIP for multimodal: https://github.com/mlfoundations/open_clip
- LangGraph for agent loops: https://langchain-ai.github.io/langgraph/
