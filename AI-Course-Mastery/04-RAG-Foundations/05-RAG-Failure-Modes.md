# 05 — RAG Failure Modes: The 7 Ways RAG Breaks in Production

## 🎯 Purpose & Goals

RAG systems look great in demos. You show a clean notebook: retrieve three chunks, feed them to an LLM, get a perfect answer. The user is impressed. You deploy to production.

Two weeks later, users are complaining the bot is "dumb," it's contradicting itself, giving wrong answers, or — worst of all — confidently giving *almost* right answers with subtle errors that slip past human reviewers.

This module is about what happens when RAG hits reality. Not the happy path — the **failure modes**. Every way RAG can break, how to detect it in production, and how to design systems that fail gracefully instead of silently.

**By the end of this module, you will:**

1. Know the 7 fundamental failure modes of RAG systems — not as a memorized list, but as failure *patterns* you can recognize from symptoms
2. Build diagnostic tools to detect each failure mode in production (not just in evaluation)
3. Design mitigation strategies — some in the retrieval layer, some in the context integration layer, some in the generation layer
4. Understand that **not all failures are fixable** — and some require architectural changes, not prompt tweaks
5. Internalize that debugging a RAG system means answering "Is the problem in R, A, or G?" — and each requires different tools

---

### ⏹ STOP. Answer these questions before proceeding.

Spend real time with each. If a question makes you uncomfortable because you realize you don't know the answer — **good**. That's exactly where learning happens.

**Question 1 — The Silent Contradiction**

*Your customer support RAG bot answers: "Our standard shipping takes 5-7 business days within the US." A user replies with a screenshot showing your website says "2-3 business days." You check the knowledge base — both are in there. One page describes standard shipping, another page from a later update changed the timeline. Your bot retrieved both, cited one, ignored the other. The answer was wrong not because it couldn't find information, but because it didn't notice it found conflicting information.*

*How would you design a RAG system that detects contradictions between retrieved chunks and handles them explicitly? What signal would tell you "there are two incompatible statements in the context"? How should the system respond — pick one, ask the user, or report the conflict? What if the conflict is subtle (e.g., "unlimited storage" vs. "10TB storage cap")?*

**Question 2 — The Retrieval That Looks Right But Is Wrong**

*You have a documentation RAG system. A user asks "How do I configure OAuth for our production environment?" The retrieval pipeline returns chunks about "OAuth configuration" with a relevance score of 0.91 — excellent. The chunks describe configuring OAuth in a development environment. The user follows the instructions and their production setup breaks because dev and prod configurations differ.*

*The retrieval engine did its job correctly — it found OAuth config documents. But it failed at understanding the user's specific intent (production vs. development). The user's query had the word "production," but the embedding model didn't semantically discriminate between dev and prod OAuth docs because they're structurally similar.*

*Where does this failure live? In the retriever? In the chunking? In the context integration? In the LLM's use of the context? How would you detect this in production where you can't manually inspect every query? What's the cheapest mitigation that would catch most of these?*

**Question 3 — The Confidence Trap**

*Your RAG system answers 100 test questions. On 90 of them, it gives excellent answers. You're ready to ship. In production, users rate 85% of answers as "helpful" — great numbers.*

*But the 10 questions it got wrong in testing were obvious errors — wrong numbers, hallucinated facts. The 15% unhelpful ratings in production also seem to be obvious errors. The scary part: there's an unknown number of answers where the system was subtly wrong — not obviously wrong, but wrong in ways users wouldn't notice unless they already knew the correct answer.*

*A user asks "What's the maximum file size for uploads?" The system answers "100MB." The actual answer (in the retrieved document) is "100MB for free tier, 500MB for pro tier." The system dropped the qualification. The user is on the pro tier. They see "100MB," don't know they're being capped unnecessarily.*

*How do you detect subtle errors that users won't report? What's your evaluation strategy for catching answers that are "almost right"? How do you measure the gap between "the answer is correct" and "the answer is complete"?*

**After answering, proceed to the deep-dive. These failure modes are what keep RAG engineers up at night.**

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 15 min |
| The 7 Failure Modes — Deep Dive | 60 min |
| Failure Mode 1: Missing Context | 15 min |
| Failure Mode 2: Irrelevant Context | 15 min |
| Failure Mode 3: Contradictory Context | 15 min |
| Failure Mode 4: Stale Context | 15 min |
| Failure Mode 5: The Distraction Effect | 15 min |
| Failure Mode 6: The Missing Signal | 15 min |
| Failure Mode 7: Format/Length Mismatch | 15 min |
| Code: Diagnostic Pipeline | 45 min |
| Code: Mitigation Strategies | 45 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes (meta) | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~5.5 hours** |

---

## 📖 Concept Deep-Dive

### The RAG Failure Taxonomy

Before we look at individual failure modes, understand this: **every RAG failure traces back to one of three layers:**

```
┌─────────────────────────────────────────────────────────┐
│                    RETRIEVAL FAILURES                     │
│  "The right information wasn't in the context at all"    │
│  Root cause: embedding quality, chunking, query wording  │
├─────────────────────────────────────────────────────────┤
│                 CONTEXT INTEGRATION FAILURES              │
│  "The right information was there but used poorly"       │
│  Root cause: ordering, formatting, instruction design   │
├─────────────────────────────────────────────────────────┤
│                 GENERATION FAILURES                       │
│  "The LLM ignored, misused, or overwrote the context"    │
│  Root cause: model capability, prompt design, alignment  │
└─────────────────────────────────────────────────────────┘
```

When you debug a RAG failure, your first question is always: **R, A, or G?**

Let's build the mental model by examining each failure mode in depth.

---

### Failure Mode 1: Missing Context

**Symptoms:**
- The LLM says "I don't have enough information" or hallucinates
- The answer contains information that seems vaguely related but misses specifics
- The user is frustrated because they *know* the information exists

**Root Causes:**

```
1. Dense retrieval failure: The query embedding was far from relevant chunk embeddings
   Why: Query-document vocabulary mismatch, embedding model limitations
   
2. Sparse retrieval failure: Keyword overlap was insufficient
   Why: Different terminology, synonyms, abbreviations
   
3. Chunking failure: The information exists but was split across chunks poorly
   Why: The answer requires combining info from chunks that weren't both retrieved
   
4. Index coverage gap: The document exists but wasn't indexed
   Why: Ingestion pipeline failure, format not supported, permission restrictions
   
5. Query ambiguity: The user's question maps to many possible intents
   Why: "How do I reset it?" — reset what? Too vague for precise retrieval
```

**Detection in Production:**

```python
class MissingContextDetector:
    """
    Detects when the LLM likely lacked sufficient context.
    Uses multiple signals for robustness.
    """

    def __init__(self, llm_client):
        self.llm_client = llm_client

    def check_retrieval_quality(self, chunks, query):
        """Heuristic: low max score or too few chunks."""
        if not chunks:
            return {
                "issue": "zero_chunks",
                "severity": "critical",
                "signal": "No documents retrieved at all"
            }

        max_score = max(c.score for c in chunks)
        if max_score < 0.6:  # Threshold depends on your embedding model
            return {
                "issue": "low_relevance",
                "severity": "warning",
                "signal": f"Max relevance score only {max_score:.3f}",
                "detail": "Best chunk may not be relevant enough"
            }

        return None

    async def check_answer_uses_context(self, answer, chunks):
        """LLM-based check: does the answer reference the provided context?"""
        # This is expensive — only run on a sample
        sources_text = "\n".join(
            f"[S{i+1}]: {c.text[:300]}"
            for i, c in enumerate(chunks)
        )

        check_prompt = f"""
Context provided to the model:
{sources_text}

Model's answer:
{answer}

Can this answer be supported by the provided context?
Reply ONLY with one of:
- YES (all claims in answer are supported by context)
- PARTIAL (some claims supported, some not)
- NO (answer cannot be supported by provided context)
- UNKNOWN (cannot determine)
"""

        response = await self.llm_client.complete(check_prompt)
        return response.strip()

    async def detect_hallucination_risk(self, answer, chunks):
        """Checks if answer introduces info not in any chunk."""
        if not chunks:
            return {"risk": "high", "reason": "No context to ground the answer"}

        check_prompt = f"""
Fact-check this answer against only the provided documents.
For EACH factual claim in the answer, determine if it appears
in ANY of the provided documents.

Documents:
{chr(10).join(f'--- Document {i+1} ---\n{c.text[:500]}' for i, c in enumerate(chunks))}

Answer:
{answer}

Return:
- claims_found_in_docs: [list of claims that ARE in the docs]
- claims_not_in_docs: [list of claims that are NOT in any doc]
- verdict: "all_grounded" | "partially_hallucinated" | "fully_hallucinated"
"""
        result = await self.llm_client.complete(check_prompt)
        return result
```

**Mitigations:**

| Severity | Mitigation |
|----------|-----------|
| Zero chunks returned | Fall back to keyword search (BM25) — it might catch what dense embeddings missed |
| All low relevance scores | Broaden query: rewrite using synonyms, expand acronyms |
| Partial coverage | Retrieve MORE chunks and use MMR diversity; the answer might need multiple chunks combined |
| Persistent missing context | Log the query for manual review → add document or improve chunking |

**The fallback chain:**

```python
async def retrieve_with_fallbacks(query, retriever, bm25_retriever=None):
    """Try dense retrieval first, fall back to sparse if needed."""
    # Try 1: Dense retrieval
    chunks = await retriever.retrieve(query, top_k=5)

    # Check: did we get anything useful?
    if not chunks or max(c.score for c in chunks) < 0.5:
        # Try 2: Sparse retrieval (BM25)
        if bm25_retriever:
            sparse_chunks = await bm25_retriever.retrieve(query, top_k=5)
            if sparse_chunks:
                return sparse_chunks, {"fallback_used": "bm25"}

        # Try 3: Query expansion
        expanded = await expand_query(query)
        chunks = await retriever.retrieve(expanded, top_k=5)

    return chunks, {}
```

---

### Failure Mode 2: Irrelevant Context

**Symptoms:**
- The LLM produces an answer that seems biased or focused on the wrong thing
- The answer includes information that's technically correct but irrelevant
- The answer misses the user's actual intent

**Root Causes:**

```
1. Embedding surface-level similarity: "How do I delete my account?" retrieves
   "How do I delete my project?" — similar phrasing, different intent
   
2. Metadata mismatch: User asks about "enterprise" pricing, but the
   enterprise pricing doc has low overlap with the query embedding
   
3. Query lacks specificity: "Tell me about pricing" retrieves ALL pricing
   docs instead of the specific tier the user cares about
   
4. Retrieval swallowed noise: The top-5 includes 2 relevant + 3 irrelevant
   chunks, and the LLM gets distracted by the noise
```

**The "Drowning in Noise" Problem:**

This is more dangerous than missing context. With missing context, the LLM usually knows it doesn't have enough information (especially with good prompting). With irrelevant context, the LLM tries to use the information it was given — even if it's wrong. **Irrelevant context actively degrades answers.**

Research has shown that adding irrelevant context can drop accuracy by **20-30%** compared to providing no context at all. The model gets confused by the noise.

**Detection:**

```python
class IrrelevantContextDetector:
    """Detects when retrieved chunks are irrelevant to the actual question."""

    def check_chunk_diversity(self, chunks, threshold=0.3):
        """
        If chunks have very different content but query is specific,
        some chunks are likely noise.
        """
        if len(chunks) < 2:
            return None

        # Compute pairwise similarities
        similarities = []
        for i in range(len(chunks)):
            for j in range(i+1, len(chunks)):
                sim = self._cosine_sim(chunks[i].embedding, chunks[j].embedding)
                similarities.append(sim)

        avg_similarity = sum(similarities) / len(similarities)

        # High diversity among chunks for a specific query is suspicious
        if avg_similarity < 0.3:
            return {
                "issue": "high_diversity",
                "severity": "info",
                "signal": f"Average chunk similarity: {avg_similarity:.3f}",
                "detail": "Chunks may cover different topics — some may be irrelevant"
            }

        return None

    async def check_intent_alignment(self, query, chunks):
        """LLM-based check: do the chunks address the user's actual intent?"""
        check_prompt = f"""
User query: {query}

Retrieved documents:
{chr(10).join(f'{i+1}. {c.text[:300]}' for i, c in enumerate(chunks))}

For each document, determine if it actually helps answer
the user's SPECIFIC intent (not just the general topic).
Reply with:
- number_of_relevant_docs: X
- number_of_irrelevant_docs: Y
- primary_intent: what the user actually wants
- intent_drift: whether the docs address a different intent than the query
"""
        return await self.llm_client.complete(check_prompt)

    def _cosine_sim(self, a, b):
        import numpy as np
        a, b = np.array(a), np.array(b)
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
```

**Mitigations:**

1. **Query rewriting on the front end**: Before retrieval, classify the user's intent and rewrite the query to match the document corpus vocabulary.

2. **Hard negative rejection**: Keep a threshold — any chunk below score X gets dropped, even if that means the LLM gets fewer chunks.

3. **Metadata filtering**: Use structured metadata (document type, tier, audience) to pre-filter. If the user asks about pricing and you know their tier, filter to that tier's docs *before* semantic search.

4. **Adaptive top-K**: For specific queries, use a lower K (3-5). For exploratory queries, use a higher K (7-10). Specific queries are more harmed by noise.

```python
def adaptive_top_k(query: str, base_k: int = 5) -> int:
    """Reduce K for specific queries to avoid noise."""
    specific_markers = [
        "how do i", "how to", "steps to", "what is the",
        "what's the", "exact", "specific", "precise"
    ]
    query_lower = query.lower()

    if any(query_lower.startswith(m) for m in specific_markers):
        return max(3, base_k - 2)  # Fewer, more precise chunks
    elif any(kw in query_lower for kw in ["compare", "difference", "vs"]):
        return min(10, base_k + 3)  # More chunks for comparison
    else:
        return base_k
```

---

### Failure Mode 3: Contradictory Context

**Symptoms:**
- The answer contains internal contradictions
- The LLM says "some sources say X, others say Y" when it shouldn't
- Answer quality degrades as you add more chunks (a key diagnostic sign)

**Root Causes:**

```
1. Version conflicts: Old docs vs. updated docs both in the index
2. Perspective conflicts: One doc says "use X for feature A," another
   says "for large deployments, use Y instead"
3. Generalization vs. exception: "All plans include unlimited storage"
   vs. "Free tier has a 10GB storage cap"
4. Temporal conflicts: "The API endpoint is at /v2/users" vs.
   "/v3/users (deprecated v2)"
```

**The Contradiction Detection Problem:**

Embedding models are bad at detecting contradictions. Two chunks:
- "The sky is blue"
- "The sky is green"

These have high similarity because they're about the same topic. The embedding model can't tell they're contradictory. **Contradiction detection is an LLM task, not an embedding task.**

**Production Detection with Cynical Prompting:**

```python
class ContradictionDetector:
    """Detects contradictions between retrieved chunks FOR evaluation logging,
    NOT in the main generation path (too expensive per-query)."""

    async def scan_for_contradictions(self, chunks):
        """Scan retrieved chunks for factual contradictions."""
        if len(chunks) < 2:
            return {"has_contradictions": False}

        # Group by document title — contradictions within same doc
        # are more problematic than across different docs
        from collections import defaultdict
        by_title = defaultdict(list)
        for c in chunks:
            by_title[c.document_title].append(c)

        contradictions = []

        for title, doc_chunks in by_title.items():
            if len(doc_chunks) < 2:
                continue

            # Check pairs of chunks from the same document
            for i in range(len(doc_chunks)):
                for j in range(i+1, len(doc_chunks)):
                    check = await self._check_pair(
                        doc_chunks[i].text, doc_chunks[j].text
                    )
                    if check:
                        contradictions.append({
                            "document": title,
                            "chunk_a": doc_chunks[i].id,
                            "chunk_b": doc_chunks[j].id,
                            "contradiction": check
                        })

        # Also check cross-document contradictions
        all_titles = list(by_title.keys())
        for i in range(len(all_titles)):
            for j in range(i+1, len(all_titles)):
                # Check most relevant chunk from each document
                c1 = max(by_title[all_titles[i]], key=lambda c: c.score)
                c2 = max(by_title[all_titles[j]], key=lambda c: c.score)
                check = await self._check_pair(c1.text, c2.text)
                if check:
                    contradictions.append({
                        "document_a": all_titles[i],
                        "document_b": all_titles[j],
                        "contradiction": check
                    })

        return {
            "has_contradictions": len(contradictions) > 0,
            "count": len(contradictions),
            "contradictions": contradictions
        }

    async def _check_pair(self, text_a, text_b):
        """Check if two texts contain contradictory factual claims."""
        check_prompt = f"""
Text A: {text_a[:500]}

Text B: {text_b[:500]}

Do these two texts contain ANY contradictory factual claims?
Focus on specific facts (numbers, dates, limits, capabilities),
not different perspectives or levels of detail.

Reply with:
- CONTRADICTION_FOUND: true | false
- contradictory_claims: [list of specific contradictions found]
- severity: "high" | "medium" | "low"
  (high = directly opposite facts, medium = different specific values,
   low = different emphasis but reconcilable)
"""
        response = await self.llm_client.complete(check_prompt)
        return response
```

**Mitigations:**

1. **Contradiction-aware prompting** — The BEST mitigation because it's cheap and always active:

```
You have been provided with reference documents. Some documents may
contain outdated information or conflicting statements.

INSTRUCTIONS:
1. When you find conflicting information in the provided sources,
   explicitly note the conflict and explain which source is more
   authoritative or recent.
2. If a specific number, date, or policy differs between sources,
   present both versions and note the discrepancy.
3. Do NOT silently pick one source over another — the user needs
   to know there's conflicting information.
```

2. **Version-aware retrieval**: Tag documents with version metadata. When retrieving, prefer newer versions unless explicitly asked about historical info.

3. **Document-level deduplication**: If contradictory information is detected (via eval logging), flag the older document for re-indexing or removal.

4. **Ask the user**: For critical contradictions (e.g., legal or medical), the system should defer rather than guess:

> "I found conflicting information in your documentation. Your pricing guide states 'unlimited storage for all plans' (2024-01), while your feature matrix shows 'Free tier: 10GB' (2024-06). Could you clarify which is current?"

---

### Failure Mode 4: Stale Context

**Symptoms:**
- Answers contradict the production service behavior
- Users report "it told me X, but when I looked, it's actually Y"
- The answer references features, APIs, or policies that no longer exist

**Root Causes:**

```
1. The index is not synced with document updates
2. Documents were updated but re-ingestion didn't trigger
3. The embedding index has both old and new versions (version clash)
4. Documents are auto-generated and the source changed without re-indexing
```

**The Staleness Blindness:**

The LLM has no way to know a document is outdated unless you tell it. To the LLM, every chunk in the context is equally authoritative. A 2022 document and a 2024 document look the same.

**Detection:**

```python
class StalenessDetector:
    """
    Detects potential staleness in retrieved chunks.

    Requires metadata with timestamps or version numbers.
    """

    def __init__(self, max_doc_age_days=180):
        self.max_doc_age_days = max_doc_age_days

    def check_chunk_freshness(self, chunks):
        """Check if any chunk exceeds max age."""
        from datetime import datetime, timezone

        warnings = []
        for chunk in chunks:
            doc_date = chunk.metadata.get("last_updated")
            if doc_date:
                age = (datetime.now(timezone.utc) - doc_date).days
                if age > self.max_doc_age_days:
                    warnings.append({
                        "chunk_id": chunk.id,
                        "doc_title": chunk.document_title,
                        "age_days": age,
                        "severity": "high" if age > 365 else "medium"
                    })

        return warnings

    def detect_version_clash(self, chunks):
        """
        Detect if we're retrieving from multiple versions of the same doc.
        """
        from collections import Counter
        titles = [c.document_title for c in chunks]
        versioned = [t for t in titles if "v" in t.lower() or "version" in t.lower()]

        if len(versioned) > 1:
            # Same base document, different versions?
            bases = set()
            for t in versioned:
                base = t.split("v")[0].strip() if "v" in t.lower() else t
                bases.add(base)

            if len(bases) < len(versioned):
                return {
                    "issue": "version_clash",
                    "severity": "high",
                    "detail": "Multiple versions of the same document retrieved"
                }

        return None

    async def detect_possible_staleness(self, answer, context_has_dates):
        """Check if answer references possibly outdated information."""
        if not context_has_dates:
            return None  # Can't detect without date metadata

        check_prompt = f"""
The following answer was generated from documents with dates ranging
from {context_has_dates['min_date']} to {context_has_dates['max_date']}.

Answer: {answer}

Does this answer contain information that is likely outdated given
the date range? Look for:
1. References to "current" features that might have changed
2. Specific numbers (prices, limits) from older documents
3. API endpoints or commands that may have been deprecated

Reply: "current", "possibly_outdated", or "unknown"
"""
        return await self.llm_client.complete(check_prompt)
```

**Mitigations:**

1. **Date metadata in every chunk**: Every chunk should include a `last_updated` field. Display it in the context:

```
[SOURCE 3: Pricing Guide — Updated 2024-01-15]
```

2. **Instruct the LLM about recency**:

```
The reference documents are tagged with update dates.
When information conflicts, prefer the most recently updated source.
If a document is older than 1 year, note that it may be outdated.
```

3. **Index expiration**: Set TTL (time-to-live) on documents. After 6 months (or whatever your update cycle is), documents are automatically de-prioritized or excluded from retrieval until re-verified.

4. **The "Audit the index" drill**: Periodically sample queries, check if the retrieved documents are still current. This is a continuous process, not a one-time fix.

---

### Failure Mode 5: The Distraction Effect

This is the most psychologically interesting failure mode.

**What it is:** When you add context that's topically related but doesn't directly answer the question, the LLM can be *distracted* by it — producing a worse answer than if you'd given it no context at all.

**Research evidence:** Multiple studies have shown that adding irrelevant documents to a RAG context reduces accuracy by 10-30% compared to giving the model no context and relying on its parametric knowledge. The mechanism is hypothesized to be the model's instruction-following bias — it was told to "use the context," so it tries to use the context even when the context is wrong or irrelevant.

**Symptoms:**
- The answer is worse when you add more chunks
- Adding top-5 chunks gives worse results than top-3
- The model ignores its own training knowledge in favor of context that's less accurate

```python
class DistractionDetector:
    """
    Detects the distraction effect by comparing model behavior
    with and without context.
    """

    async def distraction_test(self, query, chunks, llm_client):
        """
        Run the same query with and without context and compare answers.
        This is a diagnostic tool, not production middleware.
        """
        from_time = "without_context"

        # Answer without context
        no_context_answer = await llm_client.complete(
            f"Answer: {query}"
        )

        # Answer with context
        context = "\n\n".join(
            f"[S{i+1}] {c.text[:300]}"
            for i, c in enumerate(chunks)
        )
        with_context_answer = await llm_client.complete(
            f"Context:\n{context}\n\nQuestion: {query}\n\nAnswer based on context:"
        )

        # Compare
        comparison = await llm_client.complete(
            f"""
Query: {query}

Answer without context:
{no_context_answer}

Answer with context:
{with_context_answer}

Rate 1-10:
- quality_without_context: (how good is the no-context answer?)
- quality_with_context: (how good is the context-augmented answer?)
- distraction_score: (is the context answer WORSE? 1=no, 10=much worse)
- verdict: "context_helped" | "context_hurt" | "similar"
"""
        )

        return comparison
```

**Mitigations:**

1. **Aggressive relevance filtering**: Only include chunks with high relevance scores. A chunk with 0.65 relevance might be more harmful than helpful.

2. **Explicit "ignore if irrelevant" instructions**:

```
The reference documents below MAY or MAY NOT contain information
relevant to the user's question. If a document is not relevant,
ignore it completely. Do not try to use information from irrelevant
documents. If no document contains the answer, say so.
```

3. **The "trust parametric knowledge" fallback**: If retrieval quality is low (max score < 0.7), consider running without context and relying on the model's training data — but add a disclaimer that the answer wasn't verified against your documents.

4. **Distraction-aware chunk selection**: This is where MMR (from the previous module) helps. By ensuring chunk diversity, you reduce the chance of the model latching onto one theme and ignoring others.

---

### Failure Mode 6: The Missing Signal

**Symptoms:**
- The answer seems factually correct but doesn't address what the user really wants
- The user follows up with clarifying questions they shouldn't need to ask
- High "unhelpful" ratings despite technically correct answers

**Root Causes:**

```
1. Intent mismatch: User asks "Is this covered?" (about insurance), 
   system answers "what's covered" but doesn't address the specific claim
   
2. Level mismatch: Expert user asks a question and gets "beginner" answer
   
3. Format mismatch: User wants a comparison table, gets a paragraph
   
4. Scope mismatch: User asks a yes/no question, gets a page of explanation
```

This failure mode is subtle because **the answer is technically correct** — it just doesn't help.

**Detection:**

This is hard to detect automatically because it requires understanding the user's unstated intent. The best detection is:

1. **User follow-up patterns**: If users consistently ask a second question after the first answer, the first answer probably missed the signal.
2. **Session-level analysis**: Look for sequences where the first answer didn't resolve the query.
3. **Explicit feedback**: "Was this helpful?" with a free-text field.

**Mitigations:**

1. **Query rewriting with intent extraction**:

```python
async def extract_user_intent(query):
    """Extract what the user REALLY wants."""
    intent_prompt = f"""
User query: {query}

Determine what the user actually needs:
1. What's their goal? (fix something, learn something, decide something)
2. What's their expertise level? (beginner, intermediate, expert)
3. What format would help most? (step-by-step, comparison, explanation, code)
4. What's the scope? (yes/no, detailed, comprehensive)

Return as concise analysis.
"""
    return await llm_client.complete(intent_prompt)
```

2. **Level-aware context selection**: Tag documents by audience (beginner, admin, developer) and prefer the appropriate level when detectable.

3. **Format negotiation** in the system prompt:

```
If the user asks a yes/no question, answer yes/no FIRST, then elaborate.
If the user asks "how to X," provide step-by-step instructions.
If the user asks "compare X and Y," provide a structured comparison.
Match the user's communication style and level.
```

---

### Failure Mode 7: Format/Length Mismatch

**Symptoms:**
- The answer is cut off mid-sentence
- The answer is too verbose for a simple question
- The answer format doesn't match what the user asked for (asked for JSON, got prose)
- The answer exceeds a downstream limit (SMS, chat field, API response size)

**Root Causes:**

```
1. Output token estimation failure: The model writes more than you reserved
2. Instruction ambiguity: "Keep it short" means different things to different models
3. Output format instructions lost in the context: The instruction is 3000 tokens before the output
4. Chunk content overwhelms format: The chunks contain tables, and the model reproduces tables even when asked for text
```

**Detection:**

```python
class FormatMismatchDetector:
    """Detects format and length issues in answers."""

    def check_truncation(self, answer_text):
        """Check if the answer appears truncated."""
        truncation_signals = [
            answer_text.endswith((".", "!", "?")),
            answer_text.endswith(('"', "'")),
            "..." in answer_text[-20:],
            # Check for unmatched brackets
            answer_text.count("```") % 2 != 0,
        ]

        if not any(truncation_signals):
            return {
                "issue": "possible_truncation",
                "severity": "warning",
                "signal": "Answer doesn't end cleanly"
            }

        return None

    def check_format_compliance(self, expected_format, actual_answer):
        """Check if answer follows requested format."""
        if expected_format == "json":
            import json
            try:
                json.loads(actual_answer)
                return None
            except json.JSONDecodeError:
                return {
                    "issue": "format_mismatch",
                    "severity": "high",
                    "expected": "valid JSON",
                    "actual": "invalid JSON"
                }

        if expected_format == "numbered_list":
            lines = actual_answer.strip().split("\n")
            numbered = [l for l in lines if l.strip().startswith(("1.", "2.", "3.", "- ", "* "))]
            if len(numbered) < 2:
                return {
                    "issue": "format_mismatch",
                    "severity": "medium",
                    "expected": "numbered list",
                    "actual": f"only {len(numbered)} list items found"
                }

        return None
```

**Mitigations:**

1. **Explicit output format in system prompt** — not just "be concise" but "Answer in 2-3 sentences maximum. Use bullet points if listing 3+ items."

2. **Output token reservation**: When using OpenAI-style APIs, set `max_tokens` to a reasonable value. Don't leave it unlimited.

3. **Post-generation format validation**: For critical formats (JSON, specific schema), validate after generation and retry with a stricter prompt if it fails.

4. **Length-tiered prompting**:

```python
def length_instruction(complexity: str) -> str:
    instructions = {
        "simple": "Answer in 1-2 sentences. Be direct.",
        "moderate": "Answer in 2-3 paragraphs. Include relevant details.",
        "complex": "Provide a comprehensive answer with examples."
    }
    return instructions.get(complexity, instructions["moderate"])
```

---

## 💻 Code Examples

### Example 1: The Full Diagnostic Pipeline

Here's how you'd wire up all the detectors in production:

```python
"""
rag_diagnostics.py — On-demand RAG diagnostic system.

This runs asynchronously for each query, logging warnings without
blocking the user's answer. The warnings feed into dashboards,
alerts, and periodic analysis.
"""

import logging
import json
from dataclasses import dataclass, field
from typing import List, Optional, Dict, Any
from datetime import datetime
import tiktoken


@dataclass
class DiagnosticReport:
    """Full diagnostic report for a single RAG query."""
    query: str
    chunks: List[Any]
    answer: str
    timestamp: datetime = field(default_factory=datetime.now)

    # Detection results
    missing_context: Optional[Dict] = None
    irrelevant_context: Optional[Dict] = None
    contradictions: Optional[Dict] = None
    stale_context: Optional[List] = None
    distraction_risk: Optional[Dict] = None
    format_issues: Optional[List] = None

    # Aggregate
    has_issues: bool = False
    issue_count: int = 0
    severity: str = "ok"  # "ok", "warning", "critical"

    # Performance
    retrieval_latency_ms: float = 0.0
    generation_latency_ms: float = 0.0
    context_token_count: int = 0


class RAGDiagnosticPipeline:
    """
    Non-blocking diagnostic pipeline for RAG queries.

    Runs in parallel with the main generation path. Diagnostics
    are logged but never block the answer.
    """

    def __init__(self, llm_client, enable_llm_checks=False):
        self.llm_client = llm_client
        self.enable_llm_checks = enable_llm_checks
        self.logger = logging.getLogger("rag_diagnostics")
        self.encoding = tiktoken.encoding_for_model("gpt-4")

    async def diagnose(
        self,
        query: str,
        chunks: List[Any],
        answer: str,
        retrieval_time_ms: float,
        generation_time_ms: float
    ) -> DiagnosticReport:
        """Run all diagnostic checks. This is fire-and-forget from the main path."""
        report = DiagnosticReport(
            query=query,
            chunks=chunks,
            answer=answer,
            retrieval_latency_ms=retrieval_time_ms,
            generation_latency_ms=generation_time_ms,
            context_token_count=sum(
                len(self.encoding.encode(c.text)) for c in chunks
            )
        )

        # Run cheap checks first (no LLM calls)
        report.missing_context = self._check_missing_context(chunks)
        report.stale_context = self._check_staleness(chunks)
        report.format_issues = self._check_format(answer)
        report.irrelevant_context = self._check_diversity(chunks)

        # Run expensive checks only if enabled (sampling recommended)
        if self.enable_llm_checks:
            try:
                report.contradictions = await self._check_contradictions(chunks)
            except Exception as e:
                self.logger.warning(f"Contradiction check failed: {e}")

            try:
                report.distraction_risk = await self._check_distraction(
                    query, chunks
                )
            except Exception as e:
                self.logger.warning(f"Distraction check failed: {e}")

        # Aggregate
        report.issue_count = sum(
            1 for field in [
                report.missing_context,
                report.irrelevant_context,
                report.contradictions,
                report.distraction_risk,
            ] if field and field.get("severity") in ("high", "critical")
        )

        if report.stale_context:
            report.issue_count += len([
                s for s in report.stale_context
                if s.get("severity") == "high"
            ])

        if report.format_issues:
            report.issue_count += len(report.format_issues)

        report.has_issues = report.issue_count > 0
        report.severity = (
            "critical" if report.issue_count > 3
            else "warning" if report.issue_count > 0
            else "ok"
        )

        # Log structured diagnostics
        self.logger.info(
            f"RAG diagnostic | query={query[:50]} | "
            f"issues={report.issue_count} | "
            f"severity={report.severity} | "
            f"retrieval_ms={retrieval_time_ms:.0f} | "
            f"generation_ms={generation_time_ms:.0f}"
        )

        return report

    def _check_missing_context(self, chunks):
        """Cheap: check if chunks are missing or low-scoring."""
        if not chunks:
            return {"issue": "no_chunks", "severity": "critical"}
        max_score = max(c.score for c in chunks)
        if max_score < 0.5:
            return {"issue": "low_relevance", "severity": "high",
                    "max_score": max_score}
        return None

    def _check_staleness(self, chunks):
        """Check for stale documents."""
        from datetime import datetime, timezone
        warnings = []
        for chunk in chunks:
            date = chunk.metadata.get("last_updated")
            if date:
                age = (datetime.now(timezone.utc) - date).days
                if age > 365:
                    warnings.append({
                        "doc": chunk.document_title,
                        "age_days": age,
                        "severity": "high"
                    })
                elif age > 180:
                    warnings.append({
                        "doc": chunk.document_title,
                        "age_days": age,
                        "severity": "warning"
                    })
        return warnings if warnings else None

    def _check_format(self, answer):
        """Check for format issues."""
        issues = []
        if len(answer) < 10:
            issues.append({"issue": "too_short", "severity": "warning"})
        if "```" in answer and answer.count("```") % 2 != 0:
            issues.append({"issue": "unclosed_code_block", "severity": "high"})
        return issues if issues else None

    def _check_diversity(self, chunks):
        """Check if chunks are suspiciously diverse."""
        if len(chunks) < 2:
            return None
        import numpy as np
        sims = []
        for i in range(len(chunks)):
            for j in range(i+1, len(chunks)):
                e1, e2 = np.array(chunks[i].embedding), np.array(chunks[j].embedding)
                sim = float(np.dot(e1, e2) / (np.linalg.norm(e1) * np.linalg.norm(e2)))
                sims.append(sim)
        avg = sum(sims) / len(sims) if sims else 1.0
        if avg < 0.25:
            return {"issue": "high_diversity", "severity": "info",
                    "avg_similarity": avg}
        return None

    async def _check_contradictions(self, chunks):
        """LLM-based contradiction check (expensive, sample only)."""
        # Only check top 3 most relevant chunks
        top = sorted(chunks, key=lambda c: c.score, reverse=True)[:3]
        if len(top) < 2:
            return None

        texts = [c.text[:400] for c in top]
        check = await self.llm_client.complete(
            f"Do these texts contain contradictions?\n\n"
            f"1: {texts[0]}\n\n2: {texts[1]}\n" +
            (f"\n\n3: {texts[2]}" if len(texts) > 2 else "") +
            "\n\n Reply: CONTRADICTION or NO_CONTRADICTION or UNCLEAR"
        )
        if "CONTRADICTION" in check:
            return {"issue": "contradiction_found", "severity": "high"}
        return None

    async def _check_distraction(self, query, chunks):
        """Check if irrelevant context might distract."""
        if not chunks:
            return None
        max_score = max(c.score for c in chunks)
        if max_score < 0.6 and len(chunks) > 3:
            return {
                "issue": "distraction_risk",
                "severity": "warning",
                "detail": "Low relevance scores with many chunks — may distract"
            }
        return None


# ============================================================
# Integration into your RAG pipeline
# ============================================================

async def rag_answer(query, retriever, llm_client, diagnostics=None):
    """Your main RAG pipeline with optional diagnostics."""
    t0 = datetime.now()

    # 1. Retrieve
    chunks = await retriever.retrieve(query)
    t1 = datetime.now()

    # 2. Build context (from previous module)
    context = build_optimized_context(chunks, query)

    # 3. Generate
    answer = await llm_client.complete(
        f"{context}\n\nQuestion: {query}"
    )
    t2 = datetime.now()

    # 4. Diagnostics (non-blocking)
    if diagnostics:
        # Don't await — fire and forget
        asyncio.create_task(
            diagnostics.diagnose(
                query=query,
                chunks=chunks,
                answer=answer,
                retrieval_time_ms=(t1 - t0).total_seconds() * 1000,
                generation_time_ms=(t2 - t1).total_seconds() * 1000,
            )
        )

    return answer


# ============================================================
# Log-based Analysis (for periodic review)
# ============================================================

class RAGHealthReport:
    """
    Aggregates diagnostics over time to identify systemic issues.

    Run daily/weekly to find patterns:
    - Queries with persistent missing context → need better chunking
    - Documents causing contradictions → need de-duplication
    - Users asking follow-ups → first answer missed the mark
    """

    def __init__(self, log_path: str = "rag_diagnostics.jsonl"):
        self.log_path = log_path

    def analyze_period(self, start_date, end_date):
        """Analyze diagnostics over a time period."""
        import json
        from collections import Counter

        issues = Counter()
        total = 0
        critical_queries = []

        with open(self.log_path) as f:
            for line in f:
                report = json.loads(line)
                ts = datetime.fromisoformat(report["timestamp"])
                if start_date <= ts <= end_date:
                    total += 1
                    for issue_type in [
                        "missing_context", "irrelevant_context",
                        "contradictions", "stale_context",
                        "distraction_risk", "format_issues"
                    ]:
                        if report.get(issue_type):
                            issues[issue_type] += 1
                            if report.get("severity") in ("high", "critical"):
                                critical_queries.append(report)

        print(f"Total queries: {total}")
        print(f"Queries with issues: {sum(issues.values())}")
        print(f"Issue breakdown:")
        for issue, count in issues.most_common():
            pct = (count / total) * 100
            print(f"  {issue}: {count} ({pct:.1f}%)")

        if critical_queries:
            print(f"\nCritical queries requiring investigation:")
            for q in critical_queries[:5]:
                print(f"  - {q['query'][:80]}")

        return {
            "total": total,
            "issue_counts": dict(issues),
            "critical_queries": critical_queries
        }
```

### Example 2: The Answer Quality Checklist

When you deploy a RAG system, run this checklist on a sample of answers each week:

```python
"""
answer_quality_checklist.py — Manual QA checklist for RAG answers.

This is NOT automated — it's a structured review process a human does
on a sample of answers to catch failure modes the automated pipeline missed.
"""

class AnswerQAReport:
    """Structured quality report for human reviewers."""

    def __init__(self, query, chunks, answer):
        self.query = query
        self.chunks = chunks
        self.answer = answer
        self.checks = {}

    def check_completeness(self):
        """Did the answer address ALL parts of the question?"""
        self.checks["completeness"] = {
            "question_parts": [],  # Break query into sub-questions
            "addressed_parts": [],
            "missed_parts": [],
            "score": None  # 0-10
        }

    def check_grounding(self):
        """Can every claim be traced to a source?"""
        self.checks["grounding"] = {
            "total_claims": 0,
            "supported_claims": 0,
            "unsupported_claims": [],
            "hallucinated_claims": []
        }

    def check_comparison(self):
        """Does the answer match what you'd write?"""
        self.checks["comparison"] = {
            "model_version_of_truth": "",  # You write the ideal answer
            "semantic_similarity": None,
            "factual_agreement": 0.0
        }

    def check_harmfulness(self):
        """Are there any harmful or problematic statements?"""
        self.checks["harmfulness"] = {
            "has_issues": False,
            "issues": []
        }

    def summary(self):
        """Generate review summary."""
        return f"""
Query: {self.query[:100]}...
Review Date: {datetime.now().date()}

Checks:
{chr(10).join(f'  {k}: {v}' for k, v in self.checks.items())}
"""
```

### Example 3: Failure-Aware Prompt Template

Here's a system prompt designed to handle common failures gracefully:

```python
FAILURE_AWARE_SYSTEM_PROMPT = """You are an AI assistant that answers questions
based on the provided reference documents.

## CORE RULES

1. **USE THE DOCUMENTS**: Base your answer on the reference documents below.
   Do not rely on your own knowledge unless the documents don't contain
   the answer.

2. **HANDLE MISSING INFORMATION**: If the documents don't contain enough
   information to answer fully, say so explicitly. Do not make up information.

3. **HANDLE CONTRADICTIONS**: If different documents contradict each other:
   - Identify the specific contradiction
   - Explain which source is more recent/authoritative (if determinable)
   - Present both perspectives when the correct one can't be determined

4. **HANDLE AMBIGUITY**: If the question is ambiguous or could be interpreted
   multiple ways:
   - State your interpretation
   - Answer based on that interpretation
   - Note what other interpretations exist

5. **CITE SOURCES**: Use [SOURCE N] for every factual claim.

6. **ACKNOWLEDGE UNCERTAINTY**: If you're unsure, say so. "I think" or
   "Based on the available information" is better than false certainty.

7. **IGNORE IRRELEVANT DOCUMENTS**: Not all provided documents may be
   relevant to the question. Ignore irrelevant ones completely.

## RESPONSE STRUCTURE

For simple factual questions: Direct answer with citations.
For complex questions: Brief summary first, then detailed breakdown.
For comparison questions: Structured comparison with pros/cons.
For "how to" questions: Step-by-step instructions.
For ambiguous questions: Clarify assumptions first, then answer. If insufficient info found, state what additional information would help.

Remember: It's better to say "I don't know" than to give a wrong answer with confidence.
"""
```

---

## ✅ Good Output Examples

### What Good Failure Handling Looks Like

**Handling missing context correctly:**

```
User: What's the refresh rate of the Dell U2723QE monitor?

Assistant: I cannot find information about the refresh rate of the Dell
U2723QE monitor in the provided documentation. The documents cover:
- Dell monitor lineup overview (no specific model specs)
- Monitor calibration guide
- Warranty information

The specific specification you're asking about is not in any of the
documents I have access to. Could you provide the product specifications
sheet, or check the model number?
```

**Handling contradiction correctly:**

```
User: What's the maximum team size for the Business plan?

Assistant: I found conflicting information in your documentation:

SOURCE 1 (Pricing Guide v3.2 - 2024-01): "Business plan supports
up to 50 team members."

SOURCE 3 (Pricing FAQ - 2024-06): "Business plan team size was
increased to 100 members in the latest update."

The FAQ (SOURCE 3) is more recent (June 2024 vs January 2024),
so the current limit appears to be 100 members. However, you may
want to verify this with the sales team as the Pricing Guide
hasn't been updated yet.
```

**Handling missing signal correctly:**

```
User: Can I use Python with your API?

Assistant: Yes, your API supports Python. 

However, I should clarify — do you need:
1. Authentication setup in Python (API keys, OAuth)?
2. Example code for specific endpoints?
3. SDK installation (if available)?
4. General best practices?

The documents I have cover authentication and endpoint examples.
Please specify what you're trying to do, and I'll give you
step-by-step instructions.
```

---

## ❌ Antipatterns & Failure Modes (Meta)

### 1. The Single-Symptom Fallacy

**What**: Assuming a RAG failure has one cause.

**Example**: "The answer was wrong — the retrieval is broken." You spend a week tuning embeddings, but the problem was actually contradictory context.

**Fix**: Always check all three layers (R, A, G) before deciding where the failure lives. The diagnostic pipeline helps here.

### 2. The "More Chunks = Better Answers" Fallacy

**What**: Increasing top-K to give the model more context.

**Reality**: More chunks → more noise → more contradictions → more distraction → WORSE answers. Above a certain threshold (typically 5-8 chunks), answer quality plateaus or declines.

**Fix**: Always A/B test your top-K. Don't assume more is better. Start with 3 and add only if evaluation shows improvement.

### 3. The Perfect Retrieval Trap

**What**: Trying to fix retrieval noise so aggressively that you miss relevant documents.

**Example**: Setting relevance threshold to 0.85 to avoid noise, but many valid answers have scores of 0.7-0.8.

**Fix**: Noise handling belongs in the generation layer, not just the retrieval layer. A good system prompt ("ignore irrelevant documents") combined with moderate retrieval threshold usually beats aggressive filtering.

### 4. The Ignorance of Staleness

**What**: Building a RAG system and assuming the index is correct forever.

**Reality**: Documents change. APIs change. Pricing changes. Your index is stale the moment it's created.

**Fix**: Treat your index like a database — it needs migrations, updates, and audits. Build staleness detection from day one.

### 5. The Feedback Loop Blindness

**What**: Using user feedback ("was this helpful?") as the only quality signal.

**Problem**: Users don't know what they don't know. If the answer is subtly wrong but sounds authoritative, users will mark it "helpful." The feedback measures satisfaction, not accuracy.

**Fix**: Use LLM-based evaluation on a sample of answers to measure actual accuracy, not just user satisfaction. These two metrics often diverge.

---

## 🧪 Drills & Challenges

### Drill 1: Diagnose the Failure (30 min)

For each scenario below, determine if the failure is in the **Retrieval**, **Augmentation** (context integration), or **Generation** layer. Explain your reasoning.

**Scenario A:**
A user asks "What's the battery life of the MacBook Pro M3?" The RAG system retrieves chunks about "MacBook Pro M3 thermal management" and "MacBook Pro M3 performance benchmarks." The answer says "The thermal management system uses a vapor chamber design" — which is true, but doesn't answer the question.

**Scenario B:**
A user asks "How do I reset my password?" The system retrieves "password reset flow for admin users" and "password policy requirements." The answer correctly describes the admin password reset flow, but the user is a regular user and the admin flow doesn't work for them.

**Scenario C:**
A user asks "What's the refund policy?" The system retrieves chunk A: "30-day refund policy" and chunk B: "90-day refund for annual plans." The answer says "30-day refund policy" — correctly citing chunk A, but missing the annual plan exception in chunk B.

**Scenario D:**
A user asks a question in Spanish: "¿Cómo configuro la autenticación de dos factores?" The retrieval returns chunks in English about "two-factor authentication setup." The LLM answers in Spanish with correct information, but the Spanish terminology doesn't match the user's expectations.

### Drill 2: Build a Contradiction-Aware System (45 min)

Extend your context integrator from the previous module to detect and handle contradictions.

**Requirements:**
1. Before passing chunks to the LLM, check for contradictions between chunks from the same document
2. If contradictions are found, add a "CONTRADICTION WARNING" section to the prompt
3. The warning should list the specific contradictory claims and their source IDs
4. Modify the system prompt to instruct the LLM to explicitly address contradictions
5. Handle the case where contradictions are benign (different contexts) vs. harmful (opposite facts)

**Test with:**
```python
chunks_with_conflict = [
    Chunk(text="The API rate limit is 100 requests per minute.", score=0.95, doc_title="API Docs v2"),
    Chunk(text="Rate limits were increased to 500 requests per minute in v3.", score=0.88, doc_title="API Docs v3"),
    Chunk(text="Rate limits apply per API key.", score=0.70, doc_title="API Docs v2"),
]
query = "What's the current rate limit?"
```

**Question to answer:** How does your system handle this? Does it default to the newer version? Does it present both? What's the user experience?

### Drill 3: The Staleness Experiment (30 min)

Create an experiment that demonstrates the impact of stale context on answer quality.

**Setup:**
1. Take a document about "Product Pricing" from 2023 (with old prices)
2. Take a document about "Product Pricing" from 2024 (with new prices)
3. Create 5 test queries about pricing
4. Run each query against: (a) only 2023 doc, (b) only 2024 doc, (c) both docs

**Measure:**
- Does the model prefer the newer document when both are provided?
- If not, does adding date metadata help?
- How does the model handle the discrepancy without explicit contradiction instructions?

**Write-up:** Document your findings. If your model handles staleness well without extra instructions, great — you've validated the model. If not, what did you need to add to the prompt?

### Drill 4: Build the Distraction Detector (35 min)

Create a function that measures the distraction effect for a given query.

```python
async def measure_distraction(
    query: str,
    relevant_chunk: str,
    noise_chunks: List[str],
    llm_client
) -> Dict:
    """
    Measure how much noise chunks distract from the answer.

    Args:
        query: The user's question
        relevant_chunk: The chunk that actually answers the question
        noise_chunks: Irrelevant but topically related chunks
        llm_client: The LLM to test

    Returns:
        accuracy_without_noise, accuracy_with_noise, distraction_delta
    """
    # Your implementation here
    pass
```

**Test with:**
- Query: "What's the return policy for electronics?"
- Relevant: "Electronics can be returned within 30 days of purchase with original packaging."
- Noise 1: "Electronics warranty covers manufacturing defects for 1 year."
- Noise 2: "Electronics recycling program accepts old devices for free."
- Noise 3: "Electronics installation service costs $49.99."

**Hypothesis to test**: Does the model's answer quality degrade as you add more noise chunks? At what point (2 chunks? 3 chunks?) does the distraction become measurable?

### Drill 5: The RAG Autopsy (45 min)

You're given a production log with real RAG failures. For each entry, perform a "RAG autopsy" — determine the root cause and recommend a fix.

```python
# RAG autopsy cases
cases = [
    {
        "id": "C001",
        "query": "How do I cancel my subscription?",
        "retrieved_chunks": [
            "Our subscription model offers monthly and annual plans...",
            "To cancel: go to Settings > Billing > Cancel Subscription...",  # Relevant!
            "Cancellation requests are processed within 24 hours.",
        ],
        "retrieval_scores": [0.92, 0.88, 0.82],
        "answer": "Our subscription plans start at $9.99/month for the Basic tier.",
        "user_feedback": "unhelpful"
    },
    {
        "id": "C002",
        "query": "What's the file size limit for uploads?",
        "retrieved_chunks": [
            "Free tier upload limit: 10MB per file",
            "Pro tier upload limit: 500MB per file",
            "Enterprise tier: custom limits negotiated with account manager",
        ],
        "retrieval_scores": [0.95, 0.93, 0.87],
        "answer": "The maximum file size for uploads is 10MB.",
        "user_tier": "pro",  # This is known from user profile
        "user_feedback": "unhelpful"
    },
    {
        "id": "C003",
        "query": "Does your API support webhooks?",
        "retrieved_chunks": [
            "Webhooks allow real-time event notifications...",
            "To configure webhooks: navigate to Settings > Integrations...",
            "Webhook payloads are delivered with a signature header for verification.",
        ],
        "retrieval_scores": [0.87, 0.85, 0.83],
        "answer": "",  # Empty answer! The LLM returned nothing.
        "user_feedback": "unhelpful"
    },
]
```

For each case:
1. Identify the failure mode
2. Determine R, A, or G layer
3. Propose the cheapest fix
4. Propose the most robust fix
5. Describe how you'd detect this in production

### Drill 6: Design a Fallback Chain (40 min)

Design a fallback chain for a RAG system that gracefully degrades when retrieval fails.

```python
class FallbackChain:
    """
    Design a multi-level fallback chain for failed retrievals.

    Level 0: Normal RAG — retrieve, integrate, generate
    Level 1: Query expansion — rewrite query, retry retrieval
    Level 2: Sparse retrieval — fall back to BM25/keyword search
    Level 3: Parametric knowledge — answer from LLM knowledge (with disclaimer)
    Level 4: Defer — "I can't answer this" with suggestions
    """
```

**Requirements:**
- Each level should be a separate, independently testable strategy
- The system should log which fallback level was used
- You need a scoring function that determines when to escalate to the next level
- The answer should include a transparency note: "This answer was generated using [method]"

**Edge cases:**
- What if all levels fail?
- What if the first answer seems good but a fallback answer is better?
- What if the user's query is clearly malicious or out-of-domain?

---

## 🚦 Gate Check

Before proceeding to Phase 4.06 (Evaluating RAG), you must demonstrate:

1. **Diagnose 3 real failures**: Take a RAG system you've built (or set up a mock one with intentional flaws) and use the diagnostic patterns from this module to identify:
   - One retrieval failure
   - One context integration failure
   - One generation failure
   
   Document each with: the query, what went wrong, what layer the failure was in, what evidence supports your diagnosis, and the recommended fix.

2. **Build and run a contradiction detection**: Create a function that checks a set of chunks for factual contradictions and generates a contradiction-aware prompt. Test it on at least 3 cases:
   - Obvious contradiction (different numbers for the same thing)
   - Subtle contradiction (same policy, different qualifiers)
   - No contradiction (chunks discuss different aspects of the same topic)

3. **Deploy a diagnostic logger**: Add logging to your RAG pipeline that captures:
   - Query text
   - Number of chunks retrieved
   - Max/min/avg relevance scores
   - Token count of context
   - Retrieval latency
   - Generation latency
   - Any detected issues (missing context, contradictions, staleness)

4. **Answer the reflection questions**:
   - What's your strategy for determining if a failure is in R, A, or G?
   - When would you accept a certain failure rate vs. investing in more complex mitigation?
   - What's the single most impactful thing you can do to make your RAG system fail gracefully?
   - How do you distinguish between "the model doesn't know" and "the model has conflicting information"?

**Passing criteria:** You can look at a bad RAG answer and systematically determine where in the pipeline the failure originated. You're not guessing — you have evidence and a method.

---

## 📚 Resources

### Papers
- **"When Not to Trust Language Models: Investigating Effectiveness of Parametric and Non-Parametric Memories"** — Directly studies when RAG helps vs. hurts.
- **"The Power of Noise: Redefining Retrieval for RAG Systems"** — On the impact of irrelevant context on answer quality.
- **"CRUD-RAG: A Comprehensive Chinese Benchmark for Retrieval-Augmented Generation"** — Includes systematic failure mode analysis.

### Production Systems to Study
- **Bing Copilot's fallback mechanisms**: Watch how it handles queries it can't answer from web results. It uses a tiered approach — search results, then query refinement, then "I can't find this."
- **Glean's RAG architecture**: Their engineering blog covers how they handle document version conflicts and staleness.
- **Notion AI's handling of contradictory context**: Pay attention to how their AI handles conflicts between different pages.

### Tools
- **RAGAS** — Has built-in metrics for faithfulness and relevance that detect certain failure modes
- **Langfuse** — Tracing that can capture retrieval details for post-hoc analysis
- **Phoenix (Arize)** — LLM observability with RAG-specific traces

### Your Own Code
- Revisit your **Knowledge Search Engine (Phase 3 project)**. It doesn't have an LLM generation step yet, but now you know what happens when you add one. Consider:
  - What failure modes would your search engine introduce if you wrapped it with an LLM?
  - How would you detect them?
  - What's the minimum diagnostic instrumentation you'd add before deploying?

### Expert-Level Deep Dives
- **For production engineers**: The most expensive RAG failures are silent ones — where the answer looks right but is wrong. Invest disproportionately in detecting silent failures over obvious ones. A "I don't know" that gets escalated to a human is infinitely better than a confident wrong answer.
- **For system designers**: Design for observability from day one. Adding diagnostics after a failure has occurred is too late — you'll never have the data you need. Every RAG pipeline should log the full (query, chunks, answer) triple for every production query.
- **For critical thinkers**: The failure modes interact. Contradictions + stale context + distraction can compound. A single failure mode might be benign, but two together can cascade. Build defenses that handle combinations, not just individual failure modes.

---

*"A RAG system that fails silently is a time bomb. A RAG system that fails loudly is a system you can fix. The goal is not zero failures — it's zero undetected failures."*
