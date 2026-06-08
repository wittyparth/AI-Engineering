# 01 — HyDE & Query Transformation: Rethinking How We Search

## 🎯 Purpose & Goals

The fundamental assumption behind standard RAG retrieval is: **similar queries have similar embeddings, and similar documents have similar embeddings, so a query embedding near a document embedding means they're about the same thing.**

This assumption breaks all the time in production.

A user asks "How long until I can write off that new server I bought?" The document says "Depreciation follows MACRS with 7-year recovery period." The embeddings are distant because the VOCABULARY is different, even though the MEANING is aligned.

Query transformation techniques fix this by **changing the query before searching** — not changing the documents. The insight: the query is the one thing you can transform cheaply (no re-indexing required), so invest your optimization budget there.

**By the end of this module, you will:**

1. Understand **HyDE (Hypothetical Document Embeddings)** — the technique that generates a "fake document" from the query and uses THAT for retrieval instead of the raw query
2. Master **query expansion** — generating multiple query variants and combining retrieval results
3. Implement **step-back prompting** — asking a broader question before narrowing down
4. Build **query rewriting** for multi-turn conversations and cross-domain retrieval
5. Know when EACH technique applies, how they compose, and when they HURT performance
6. Build a production query transformation pipeline that routes to the right strategy

---

### ⏹ STOP. Answer these before proceeding.

**Question 1 — Why Hypothetical Documents?**

*A user asks "How do I hook up my payment processing?" Your docs say "Stripe integration requires API key configuration in Settings > Payments > API Keys. Use the publishable key for frontend and secret key for backend."*

*Your embedding model maps the query to a region of embedding space near other "how do I" questions. Your document maps to a region near other "Stripe integration" documents. These regions might not overlap much — they're different kinds of text.*

*If you could write a "hypothetical document" that looks like your real documents but answers the user's question — something like "Stripe API key configuration: publishable keys are used for frontend tokenization, secret keys are used for backend charges" — and embed THAT instead of the query... would the embedding be closer to your real documents?*

*Why does embedding length and style matter for search? What assumptions does HyDE make about the relationship between query-embedding-space and document-embedding-space?*

**Question 2 — The Multi-Query Problem**

*You ask a search system: "What's the impact of sleep on learning?" A single embedding captures one angle. Your documents cover sleep's impact on: memory consolidation during REM, attention during lectures, problem-solving after rest, and long-term retention.*

*A single query embedding might be closest to only ONE of these angles, missing the other three. But if you generated 5 different phrasings — "How does REM sleep affect memory consolidation?", "Does sleep improve attention during learning?", etc. — and merged the results, you'd cover all angles.*

*What's the tradeoff between number of generated queries and retrieval quality? At what point do you get diminishing returns? How do you merge results from different queries without introducing bias?*

**Question 3 — The Step-Back Trap**

*A user asks: "Why does my app crash when I try to upload a PDF larger than 10MB?" The direct query retrieves chunks about "upload limits" and "file size restrictions." The answer lists size limits. But the real problem is a memory issue in the PDF parser, not a file size restriction.*

*If you had asked a STEP-BACK question first — "What causes application crashes during file upload?" — you would have retrieved entirely different chunks about memory management, buffer overflows, and PDF parsing.*

*When does narrowing the query hurt? When is the step-back (broader question) more useful than the direct question? How do you know which level of specificity to search at?*

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 15 min |
| The Query Transformation Landscape | 15 min |
| HyDE Deep Dive | 45 min |
| Query Expansion | 30 min |
| Step-Back Prompting | 25 min |
| Query Rewriting | 25 min |
| Multi-Turn Query Handling | 20 min |
| Code: HyDE Pipeline | 45 min |
| Code: Query Router | 40 min |
| Code: Expansion + Merge | 30 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~6.5 hours** |

---

## 📖 Concept Deep-Dive

### 1. The Query Transformation Landscape

Before we dive into techniques, understand the strategic landscape. Query transformations exist on two axes:

```
HOW the query is changed:
                    
                    ADD INFORMATION
                    (expand, rewrite, 
                     generate hypothetical)
                           │
     KEEP ORIGINAL ────────┼──────── REPLACE ORIGINAL
     (just use              │         (step-back,
      the query)             │          generate new)
                           │
                    REMOVE INFORMATION
                    (extract key concepts,
                     simplify)
```

And WHY:

| Problem | Solution | Technique |
|---------|----------|-----------|
| Query vocabulary ≠ document vocabulary | Convert query into document language | HyDE |
| Query is ambiguous, documents cover multiple interpretations | Generate multiple query variants | Query Expansion |
| Query is too specific, documents are at a broader level | Broaden the query | Step-Back |
| Query is vague, documents are specific | Extract concrete search terms | Query Decomposition |
| Multi-turn — query references earlier context | Add context from conversation | Query Rewriting |

---

### 2. HyDE: Hypothetical Document Embeddings

The core insight of HyDE (Gao et al., 2022) is deceptively simple:

**Don't embed the query. Generate a hypothetical document that answers the query, then embed THAT.**

Why does this work? Embedding models are trained on document-document similarity. Two documents about the same topic have similar embeddings. But a query and a document about the same topic might NOT have similar embeddings because they're different TEXT TYPES (question vs. statement, short vs. long, casual vs. formal). HyDE bridges this gap by converting the query into document-style text before embedding.

#### How HyDE Works

```
Standard RAG:
  "How do I reset my password?" ──→ embed ──→ [0.2, 0.5, -0.1, ...] ──→ search
     (short, question format, casual vocabulary)

HyDE:
  "How do I reset my password?" ──→ LLM ──→ "Password reset can be performed
  from the account settings page. Navigate to Settings > Security > Password
  and enter your current password followed by your new password..."

  This hypothetical document ──→ embed ──→ [0.7, 0.3, 0.4, ...] ──→ search
     (longer, statement format, document vocabulary)
```

#### Implementation

```python
"""
hyde.py — Hypothetical Document Embeddings for improved RAG retrieval.

HyDE generates a "hypothetical document" that answers the query,
then uses THAT document's embedding for retrieval instead of the raw query.
"""

from typing import List, Optional, Dict, Any
from dataclasses import dataclass
from openai import AsyncOpenAI
import numpy as np


@dataclass
class HyDEResult:
    """Result of a HyDE-enhanced retrieval."""
    original_query: str
    hypothetical_document: str
    retrieved_chunks: List[Dict[str, Any]]
    query_embedding: Optional[List[float]] = None
    hyde_embedding: Optional[List[float]] = None


class HyDETransformer:
    """
    Generate hypothetical documents for improved retrieval.

    The key hypothesis: a "document answering the question" embeds
    closer to real documents than the question itself does, because
    the embedding model was trained on document-document similarity.
    """

    def __init__(
        self,
        llm_client: AsyncOpenAI,
        embedding_client,
        model: str = "gpt-4o-mini",
        embedding_model: str = "text-embedding-3-small",
        num_hypotheticals: int = 1,
        hyde_instruction: Optional[str] = None
    ):
        self.llm_client = llm_client
        self.embedding_client = embedding_client
        self.model = model
        self.embedding_model = embedding_model
        self.num_hypotheticals = num_hypotheticals

        # Default instruction: produce a document-style passage
        self.hyde_instruction = hyde_instruction or (
            "Write a factual document that answers the given question. "
            "Use formal, technical language. Write in complete sentences "
            "as if documenting the answer in a knowledge base. "
            "Include specific details, numbers, and procedures where applicable."
        )

    async def generate_hypothetical_document(self, query: str) -> str:
        """
        Generate a hypothetical document that would answer the query.

        The prompt is designed to produce text that LOOKS like
        the documents in your knowledge base (statement format,
        technical vocabulary, complete sentences).
        """
        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {
                    "role": "system",
                    "content": self.hyde_instruction
                },
                {
                    "role": "user",
                    "content": f"Question: {query}"
                }
            ],
            max_tokens=300,
            temperature=0.7  # Slight randomness for diversity
        )

        return response.choices[0].message.content.strip()

    async def generate_multiple_hypotheticals(
        self, query: str, n: int = 3
    ) -> List[str]:
        """
        Generate multiple hypothetical documents for the same query.

        Each one covers a different angle or uses different vocabulary.
        Results are merged for more comprehensive retrieval.
        """
        tasks = [self.generate_hypothetical_document(query) for _ in range(n)]
        results = await asyncio.gather(*tasks)
        return results

    async def embed(self, text: str) -> List[float]:
        """Embed text using the embedding model."""
        response = await self.embedding_client.embeddings.create(
            model=self.embedding_model,
            input=text
        )
        return response.data[0].embedding

    async def retrieve(
        self,
        query: str,
        retriever,
        top_k: int = 5,
        use_hyde: bool = True
    ) -> HyDEResult:
        """
        Retrieve chunks using HyDE.

        Args:
            query: The user's original query
            retriever: A retriever object with a `search(embedding, top_k)` method
            top_k: Number of chunks to retrieve
            use_hyde: If True, use HyDE. If False, use standard query embedding.

        Returns:
            HyDEResult with retrieved chunks and metadata
        """
        if not use_hyde:
            # Standard retrieval (baseline)
            query_emb = await self.embed(query)
            chunks = await retriever.search(query_emb, top_k)
            return HyDEResult(
                original_query=query,
                hypothetical_document="",
                retrieved_chunks=chunks,
                query_embedding=query_emb
            )

        # HyDE: generate hypothetical document
        hypo_doc = await self.generate_hypothetical_document(query)

        # Embed the hypothetical document instead of the query
        hyde_emb = await self.embed(hypo_doc)

        # Optionally also embed the original query for hybrid
        query_emb = await self.embed(query)

        # Retrieve using HyDE embedding
        chunks = await retriever.search(hyde_emb, top_k)

        return HyDEResult(
            original_query=query,
            hypothetical_document=hypo_doc,
            retrieved_chunks=chunks,
            query_embedding=query_emb,
            hyde_embedding=hyde_emb
        )

    async def retrieve_hybrid(
        self,
        query: str,
        retriever,
        top_k: int = 5,
        hyde_weight: float = 0.7
    ) -> HyDEResult:
        """
        Retrieve using BOTH the query embedding and the HyDE embedding,
        then fuse results with weighted scoring.

        This combines the precision of direct query search with the
        recall of HyDE's document-style search.
        """
        # Get both embeddings
        query_emb = await self.embed(query)
        hypo_doc = await self.generate_hypothetical_document(query)
        hyde_emb = await self.embed(hypo_doc)

        # Search with both
        query_chunks = await retriever.search(query_emb, top_k * 2)
        hyde_chunks = await retriever.search(hyde_emb, top_k * 2)

        # Fuse with weighted Reciprocal Rank Fusion
        chunk_scores = {}

        for i, chunk in enumerate(query_chunks):
            chunk_scores[chunk["id"]] = chunk_scores.get(chunk["id"], 0) + (
                (1 - hyde_weight) * (1.0 / (i + 1))
            )

        for i, chunk in enumerate(hyde_chunks):
            chunk_scores[chunk["id"]] = chunk_scores.get(chunk["id"], 0) + (
                hyde_weight * (1.0 / (i + 1))
            )

        # Sort by combined score
        ranked = sorted(
            chunk_scores.items(),
            key=lambda x: x[1],
            reverse=True
        )

        # Map back to chunk data
        chunk_map = {c["id"]: c for c in query_chunks + hyde_chunks}
        merged_chunks = [
            chunk_map[cid] for cid, _ in ranked[:top_k]
            if cid in chunk_map
        ]

        return HyDEResult(
            original_query=query,
            hypothetical_document=hypo_doc,
            retrieved_chunks=merged_chunks,
            query_embedding=query_emb,
            hyde_embedding=hyde_emb
        )


# ============================================================
# A/B Test: HyDE vs. Standard Retrieval
# ============================================================

class HyDEBenchmark:
    """Compare HyDE retrieval vs standard retrieval on test queries."""

    async def compare(
        self,
        test_queries: List[str],
        hyde_system: HyDETransformer,
        retriever,
        relevant_chunks: Dict[str, List[str]]  # query -> [relevant chunk IDs]
    ) -> Dict:
        """
        Compare HyDE vs standard retrieval.

        Returns metrics for both approaches.
        """
        results = {"standard": {}, "hyde": {}, "hybrid": {}}

        for approach, use_hyde, use_hybrid in [
            ("standard", False, False),
            ("hyde", True, False),
            ("hybrid", True, True),
        ]:
            precisions = []
            recalls = []

            for query in test_queries:
                if use_hybrid:
                    result = await hyde_system.retrieve_hybrid(query, retriever)
                else:
                    result = await hyde_system.retrieve(query, retriever, use_hyde=use_hyde)

                retrieved_ids = [c["id"] for c in result.retrieved_chunks]
                relevant = set(relevant_chunks.get(query, []))

                if retrieved_ids:
                    p = len(set(retrieved_ids) & relevant) / len(retrieved_ids)
                else:
                    p = 0.0

                if relevant:
                    r = len(set(retrieved_ids) & relevant) / len(relevant)
                else:
                    r = 0.0

                precisions.append(p)
                recalls.append(r)

            results[approach] = {
                "avg_precision": sum(precisions) / len(precisions),
                "avg_recall": sum(recalls) / len(recalls),
                "f1": 2 * (
                    (sum(precisions) / len(precisions) * sum(recalls) / len(recalls))
                    / (sum(precisions) / len(precisions) + sum(recalls) / len(recalls) + 1e-10)
                )
            }

        return results
```

#### When HyDE Works Best

```
✅ High vocabulary mismatch: 
   Query: "How do I get my money back?"
   Doc says: "Refund processing requires submission within 30-day window"
   → HyDE generates: "Refund policy requires submissions within 30 days"
   → Embedding now close to the document

✅ When queries are short and documents are long:
   Short queries lack context for good embeddings.
   HyDE adds context.

✅ When the domain has specialized vocabulary:
   User says "write off" but doc says "depreciation."
   HyDE translates to document vocabulary.

❌ When the query is already well-formed:
   "What is the MACRS depreciation recovery period for 7-year property?"
   → Already matches document vocabulary. HyDE might introduce noise.

❌ When queries are VERY domain-specific:
   HyDE might generate plausible-sounding but wrong hypotheticals.
   "What's the torque spec for the 2024 Model X engine mount bolts?"
   → LLM might guess "45 ft-lbs" but real doc says "52 ft-lbs"
   → Wrong hypothetical embedding leads to wrong retrieval.

❌ When latency matters:
   HyDE adds 300-800ms for LLM generation.
   Standard retrieval is just the embedding call.
```

---

### 3. Query Expansion

Instead of generating ONE hypothetical document, generate MULTIPLE queries and merge results.

```python
class QueryExpander:
    """
    Generate multiple query variants and merge retrieval results.

    Each variant captures a different angle of the query.
    Merging them provides more comprehensive coverage.
    """

    def __init__(self, llm_client, model: str = "gpt-4o-mini"):
        self.llm_client = llm_client
        self.model = model

    async def expand_query(
        self, query: str, n_variants: int = 5
    ) -> List[str]:
        """
        Generate multiple search queries from the original.

        Each variant: different phrasing, different angle,
        different level of specificity.

        Returns: [original_query, variant_1, variant_2, ...]
        """
        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {
                    "role": "system",
                    "content": (
                        "You are a search query expansion system. "
                        "Given a user's question, generate {n_variants} "
                        "different search queries that would retrieve "
                        "relevant documents.\n\n"
                        "Guidelines:\n"
                        "- Vary the phrasing (different words for same concept)\n"
                        "- Vary the specificity (some general, some specific)\n"
                        "- Use different terms a document might use\n"
                        "- Include synonyms and related concepts\n"
                        "- Cover different aspects of multi-part questions\n\n"
                        "Return each query on a new line starting with 'Q:'"
                    ).format(n_variants=n_variants)
                },
                {"role": "user", "content": f"Question: {query}"}
            ],
            max_tokens=500,
            temperature=0.8
        )

        content = response.choices[0].message.content
        variants = [
            line[2:].strip() for line in content.split("\n")
            if line.strip().startswith("Q:")
        ]

        # Always include the original query
        return [query] + variants[:n_variants]

    async def retrieve_expanded(
        self,
        query: str,
        retriever,
        embedder,
        top_k: int = 5,
        n_variants: int = 5
    ) -> List[Dict]:
        """
        Retrieve using expanded queries and merge results.

        Strategy: Reciprocal Rank Fusion across all query variants.
        """
        variants = await self.expand_query(query, n_variants)

        # Retrieve for each variant
        all_results = []
        for variant in variants:
            emb = await embedder(variant)
            chunks = await retriever.search(emb, top_k)
            all_results.append(chunks)

        # Reciprocal Rank Fusion
        rrf_scores = {}
        for k in range(60):  # RRF constant
            for rank_list in all_results:
                for rank, chunk in enumerate(rank_list, 1):
                    cid = chunk["id"]
                    rrf_scores[cid] = rrf_scores.get(cid, 0) + 1 / (k + rank)

        # Sort by RRF score
        ranked = sorted(
            rrf_scores.items(),
            key=lambda x: x[1],
            reverse=True
        )

        # Map back to chunk data
        chunk_map = {}
        for rank_list in all_results:
            for chunk in rank_list:
                chunk_map[chunk["id"]] = chunk

        return [
            chunk_map[cid] for cid, _ in ranked[:top_k]
            if cid in chunk_map
        ]
```

**The tradeoff:** More variants → better coverage → higher recall → but slower retrieval (O(n) embedding calls) and more noise (some variants might retrieve irrelevant chunks).

---

### 4. Step-Back Prompting

Sometimes the DIRECT query retrieves poorly because it's too specific. Step-back prompting asks a broader question first, then narrows down.

```python
class StepBackPrompting:
    """
    Two-step retrieval:

    Step 1: "Step back" — ask a broader/general question
    Step 2: Retrieve from the broader question's context
    Step 3: (Optional) Narrow down with the original query
    """

    def __init__(self, llm_client, model: str = "gpt-4o-mini"):
        self.llm_client = llm_client
        self.model = model

    async def step_back(self, query: str) -> str:
        """
        Generate a step-back question.

        The step-back question is MORE GENERAL than the original.
        It asks about the CONCEPT or PRINCIPLE rather than the specific case.

        Examples:
          Original: "Why does my app crash when uploading PDFs over 10MB?"
          Step-back: "What causes application crashes during file upload?"

          Original: "How do I configure OAuth for production environment?"
          Step-back: "What are the OAuth configuration steps and environments?"

          Original: "What's the depreciation schedule for the new server?"
          Step-back: "How does asset depreciation work?"
        """
        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {
                    "role": "system",
                    "content": (
                        "You are a search query reasoning system. "
                        "Given a specific user question, generate a "
                        "STEP-BACK question — a broader, more general "
                        "question that asks about the underlying concept "
                        "or principle.\n\n"
                        "The step-back question should:\n"
                        "- Ask about general principles, not specific cases\n"
                        "- Use broader, more foundational terminology\n"
                        "- Cover the CONCEPT the user is really asking about\n"
                        "- Be answerable from general documentation\n\n"
                        "Return ONLY the step-back question, nothing else."
                    )
                },
                {"role": "user", "content": f"Question: {query}"}
            ],
            max_tokens=150,
            temperature=0.3
        )

        return response.choices[0].message.content.strip()

    async def retrieve_with_step_back(
        self,
        query: str,
        retriever,
        embedder,
        top_k: int = 5,
        step_back_weight: float = 0.4
    ) -> List[Dict]:
        """
        Retrieve using BOTH the original query and the step-back query.

        The step-back brings in broader context.
        The original query keeps the results specific.

        Fusion: weighted combination of both result sets.
        """
        # Step 1: Generate step-back question
        step_back_q = await self.step_back(query)

        # Step 2: Retrieve with both
        query_emb = await embedder(query)
        step_back_emb = await embedder(step_back_q)

        direct_chunks = await retriever.search(query_emb, top_k)
        broad_chunks = await retriever.search(step_back_emb, top_k)

        # Step 3: Fuse results
        fused_scores = {}

        # Direct chunks get higher weight
        for rank, chunk in enumerate(direct_chunks, 1):
            fused_scores[chunk["id"]] = fused_scores.get(chunk["id"], 0) + (
                (1 - step_back_weight) * (1.0 / rank)
            )

        # Step-back chunks supplement
        for rank, chunk in enumerate(broad_chunks, 1):
            fused_scores[chunk["id"]] = fused_scores.get(chunk["id"], 0) + (
                step_back_weight * (1.0 / rank)
            )

        ranked = sorted(
            fused_scores.items(), key=lambda x: x[1], reverse=True
        )

        chunk_map = {c["id"]: c for c in direct_chunks + broad_chunks}
        return [
            chunk_map[cid] for cid, _ in ranked[:top_k]
            if cid in chunk_map
        ]
```

**The key insight:** Step-back isn't always useful. For factual lookup queries ("What's the refund policy?"), the direct query is best. For diagnostic/analytical queries ("Why does X happen when I do Y?"), step-back often finds the root cause better.

---

### 5. Query Rewriting

For multi-turn conversations, the user's query often references earlier context:

- "How do I reset my password?" → system answers
- "What about for admin accounts?" → This query MEANS "How do I reset my password for admin accounts?"

Query rewriting makes the implicit context explicit:

```python
class QueryRewriter:
    """
    Rewrite queries in context of conversation history.

    Handles:
    - Pronoun resolution ("it", "that", "they")
    - Ellipsis ("What about...")
    - Context-dependent terms ("the API key" → which API key)
    - Follow-up questions that depend on previous answer
    """

    def __init__(self, llm_client, model: str = "gpt-4o-mini"):
        self.llm_client = llm_client
        self.model = model

    async def rewrite(
        self,
        current_query: str,
        conversation_history: List[Dict]
    ) -> str:
        """
        Rewrite the current query to be self-contained.

        A self-contained query can be understood WITHOUT
        the conversation history — it includes all context
        needed for retrieval.

        If the query is already self-contained, return it as-is.
        """
        if not conversation_history:
            return current_query

        # Check if this likely needs rewriting
        if self._is_self_contained(current_query):
            return current_query

        # Format history
        history_text = self._format_history(conversation_history)

        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {
                    "role": "system",
                    "content": (
                        "Given a conversation history and a current query, "
                        "rewrite the current query to be SELF-CONTAINED. "
                        "The rewritten query should include ALL context "
                        "needed to search a knowledge base effectively.\n\n"
                        "Rules:\n"
                        "- Resolve pronouns to their antecedents\n"
                        "- Include relevant context from the conversation\n"
                        "- Keep the query focused on what's being asked NOW\n"
                        "- Don't add information that wasn't implied\n"
                        "- If no rewrite is needed, return the original query\n\n"
                        "Return ONLY the rewritten query."
                    )
                },
                {
                    "role": "user",
                    "content": (
                        f"Conversation history:\n{history_text}\n\n"
                        f"Current query: {current_query}\n\n"
                        f"Rewritten query:"
                    )
                }
            ],
            max_tokens=200,
            temperature=0.2
        )

        return response.choices[0].message.content.strip()

    def _is_self_contained(self, query: str) -> bool:
        """Check if a query is probably self-contained."""
        query_lower = query.lower()
        # References to "it", "that", "this" suggest dependency
        dependency_markers = [
            "what about", "how about", "and", "also",
            "what if", "what's the", "is there",
            "does it", "can i", "will it"
        ]
        query_start = query_lower.split()[0] if query_lower.split() else ""
        return not any(query_lower.startswith(m) for m in dependency_markers)

    def _format_history(self, history: List[Dict]) -> str:
        """Format conversation history for the LLM."""
        # Keep last 2 exchanges for context
        recent = history[-4:] if len(history) >= 4 else history
        lines = []
        for msg in recent:
            role = "User" if msg["role"] == "user" else "Assistant"
            # Truncate long messages
            content = msg["content"][:500]
            lines.append(f"{role}: {content}")
        return "\n".join(lines)
```

---

### 6. The Combined Pipeline: Query Router

The most sophisticated approach: route each query to the best transformation strategy (or combine multiple).

```python
"""
query_router.py — Route queries to the optimal transformation strategy.

Classification dimensions:
- Is the query self-contained or context-dependent?
- Is the query specific or general?
- Is there vocabulary mismatch likely?
- Is this a diagnostic/analytical query?
- How many sub-questions does it contain?
"""

from enum import Enum
from dataclasses import dataclass
from typing import List, Optional


class QueryComplexity(Enum):
    SIMPLE = "simple"           # Factual lookup, single concept
    CONTEXT_DEPENDENT = "context"  # References conversation history
    VOCAB_MISMATCH = "vocab"    # User uses different words than docs
    ANALYTICAL = "analytical"   # "Why does X happen?"
    MULTI_PART = "multi_part"   # Multiple sub-questions
    AMBIGUOUS = "ambiguous"     # Could mean multiple things


@dataclass
class QueryPlan:
    """The transformation plan for a query."""
    original: str
    rewritten: str  # After context incorporation
    complexity: QueryComplexity
    strategies: List[str]  # ["hyde", "expansion", "step_back", ...]
    n_variants: int
    use_hyde: bool
    use_step_back: bool
    use_expansion: bool


class QueryRouter:
    """
    Classifies queries and routes them to appropriate strategies.

    The goal: use expensive transformations ONLY when they help.
    Simple queries get direct retrieval. Complex queries get the toolkit.
    """

    def __init__(self, llm_client, model: str = "gpt-4o-mini"):
        self.llm_client = llm_client
        self.model = model
        self.rewriter = QueryRewriter(llm_client, model)
        self.expander = QueryExpander(llm_client, model)
        self.step_back = StepBackPrompting(llm_client, model)
        self.hyde = None  # Initialized with embedding client

    async def classify(self, query: str, history: List[Dict] = None) -> QueryPlan:
        """
        Classify the query and create a transformation plan.

        Uses lightweight heuristics first, falls back to LLM for ambiguity.
        """
        # Step 1: Check if context-dependent
        needs_rewrite = history and not self.rewriter._is_self_contained(query)
        rewritten = query
        if needs_rewrite:
            rewritten = await self.rewriter.rewrite(query, history)

        # Step 2: Classify complexity
        complexity = await self._classify_complexity(rewritten)

        # Step 3: Build strategy plan
        plan = QueryPlan(
            original=query,
            rewritten=rewritten,
            complexity=complexity,
            strategies=[],
            n_variants=1,
            use_hyde=False,
            use_step_back=False,
            use_expansion=False,
        )

        if complexity == QueryComplexity.SIMPLE:
            # Direct retrieval, no transformation needed
            plan.strategies = ["direct"]

        elif complexity == QueryComplexity.VOCAB_MISMATCH:
            # HyDE addresses vocabulary mismatch
            plan.use_hyde = True
            plan.strategies = ["hyde"]

        elif complexity == QueryComplexity.ANALYTICAL:
            # Step-back for "why" questions
            plan.use_step_back = True
            plan.strategies = ["step_back"]

        elif complexity == QueryComplexity.MULTI_PART:
            # Expansion for multi-part questions
            plan.use_expansion = True
            plan.n_variants = 5
            plan.strategies = ["expansion"]

        elif complexity == QueryComplexity.AMBIGUOUS:
            # Expansion covers multiple interpretations
            plan.use_expansion = True
            plan.n_variants = 5
            plan.strategies = ["expansion"]

        elif complexity == QueryComplexity.CONTEXT_DEPENDENT:
            # Already handled via rewriting, add HyDE for safety
            plan.use_hyde = True
            plan.strategies = ["rewrite", "hyde"]

        return plan

    async def _classify_complexity(self, query: str) -> QueryComplexity:
        """Classify query complexity using heuristics + LLM."""

        # Quick heuristics
        q_lower = query.lower()

        # Multi-part questions
        if any(m in q_lower for m in [" and ", " vs ", " versus "]):
            if len(query.split()) > 8:
                return QueryComplexity.MULTI_PART

        # Analytical questions
        if q_lower.startswith(("why", "how does", "what causes", "what's the reason")):
            return QueryComplexity.ANALYTICAL

        # Vocabulary mismatch suspicion: short queries with common words
        if len(query.split()) <= 5:
            common_words = {"get", "make", "do", "have", "need", "want", "how", "what"}
            query_words = set(w.lower() for w in query.split())
            if query_words & common_words:
                return QueryComplexity.VOCAB_MISMATCH

        # Long, specific queries are usually fine
        if len(query.split()) > 15:
            return QueryComplexity.SIMPLE

        # Default: use LLM for borderline cases
        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {
                    "role": "system",
                    "content": (
                        "Classify this search query into one category:\n"
                        "- simple: factual lookup, direct, single concept\n"
                        "- vocab_mismatch: user likely uses different "
                        "vocabulary than documents\n"
                        "- analytical: 'why' or causal question\n"
                        "- multi_part: multiple distinct sub-questions\n"
                        "- ambiguous: could mean different things\n\n"
                        "Reply with ONE word."
                    )
                },
                {"role": "user", "content": query}
            ],
            max_tokens=10,
            temperature=0.1
        )

        label = response.choices[0].message.content.strip().lower()

        mapping = {
            "simple": QueryComplexity.SIMPLE,
            "vocab_mismatch": QueryComplexity.VOCAB_MISMATCH,
            "analytical": QueryComplexity.ANALYTICAL,
            "multi_part": QueryComplexity.MULTI_PART,
            "ambiguous": QueryComplexity.AMBIGUOUS,
        }
        return mapping.get(label, QueryComplexity.SIMPLE)

    async def retrieve(
        self,
        query: str,
        retriever,
        embedder,
        history: List[Dict] = None,
        top_k: int = 5
    ) -> Dict:
        """Execute the query plan and retrieve results."""
        plan = await self.classify(query, history)
        chunks = []

        if "direct" in plan.strategies:
            emb = await embedder(plan.rewritten)
            chunks = await retriever.search(emb, top_k)

        if "hyde" in plan.strategies:
            chunks = await self.hyde.retrieve_hybrid(
                plan.rewritten, retriever, top_k
            )
            chunks = chunks.retrieved_chunks

        if "step_back" in plan.strategies:
            chunks = await self.step_back.retrieve_with_step_back(
                plan.rewritten, retriever, embedder, top_k
            )

        if "expansion" in plan.strategies:
            chunks = await self.expander.retrieve_expanded(
                plan.rewritten, retriever, embedder, top_k,
                n_variants=plan.n_variants
            )

        return {
            "chunks": chunks,
            "plan": plan,
            "query_used": plan.rewritten
        }
```

---

### 7. The HyDE Failure Modes

Knowing when HyDE FAILS is as important as knowing when it works:

**Failure 1: The Hallucinated Document**

The LLM generates a hypothetical document that sounds plausible but is factually wrong. The embedding is close to WRONG documents, leading retrieval astray.

```text
Query: "What's the torque spec for 2024 Model X engine mount bolts?"
HyDE generates: "Engine mount bolts for 2024 Model X require 45 ft-lbs of torque."
Real document: "Engine mount bolts for 2024 Model X require 52 ft-lbs of torque."

The HyDE embedding is close to the "45 ft-lbs" document space,
but the real answer is in the "52 ft-lbs" document space.
Result: Retrieval misses the correct document.
```

**Mitigation:** Multiple hypothetical documents (take the best, or use consensus). Or use HyDE only for recall expansion, not as the primary signal.

**Failure 2: The Overly Specific Document**

HyDE generates a document that's TOO specific, matching only a narrow subset of relevant documents.

```text
Query: "Tell me about the benefits."
HyDE generates: "Health insurance benefits include dental, vision, and medical."
Real documents also cover: retirement benefits, vacation policy, stock options.

HyDE retrieves only insurance docs, missing the broader benefits landscape.
```

**Mitigation:** Combine HyDE with step-back or query expansion to maintain breadth.

**Failure 3: The Vocabulary Already Matches**

When the query is already well-phrased for the document corpus, HyDE adds noise:

```text
Query: "What's the API rate limit for GPT-4o-mini?"
Documents: "GPT-4o-mini rate limits: 500 RPM for Tier 1, 5000 RPM for Tier 2..."

HyDE generates: "The API rate limit for GPT-4o-mini varies by tier..."
This is already similar to the document. HyDE adds no value (and might hallucinate tiers).
```

**Mitigation:** Only use HyDE when vocabulary mismatch is detected. The Query Router's `VOCAB_MISMATCH` classification should be conservative.

---

## 💻 Code Examples

### Example 1: Production HyDE Pipeline

```python
"""
production_hyde.py — Production-ready HyDE pipeline with monitoring.
"""

import asyncio
import time
import logging
from typing import List, Optional
from dataclasses import dataclass, field


@dataclass
class HyDEMetrics:
    query: str
    hyde_generation_ms: float
    standard_top5_overlap: int  # How many chunks overlap between standard and HyDE
    hyde_improvement: float  # Measured improvement in retrieval quality
    hyde_document: str


class MonitoredHyDE:
    """
    HyDE with production monitoring.

    Tracks: when HyDE helps, when it hurts, latency, overlap with standard.
    """

    def __init__(self, hyde_transformer, retriever, embedder, logger=None):
        self.hyde = hyde_transformer
        self.retriever = retriever
        self.embedder = embedder
        self.logger = logger or logging.getLogger("hyde")
        self.metrics_log = []

    async def retrieve_with_monitoring(
        self, query: str, top_k: int = 5
    ) -> tuple:
        """
        Retrieve with HyDE, logging metrics.

        Also runs STANDARD retrieval for comparison.
        Only returns HyDE results, but logs the comparison.
        """
        t0 = time.perf_counter()

        # Standard retrieval
        query_emb = await self.embedder(query)
        standard_chunks = await self.retriever.search(query_emb, top_k)
        standard_ids = {c["id"] for c in standard_chunks}

        # HyDE retrieval
        hyde_result = await self.hyde.retrieve_hybrid(query, self.retriever, top_k)
        hyde_ids = {c["id"] for c in hyde_result.retrieved_chunks}

        hyde_latency = (time.perf_counter() - t0) * 1000

        # Metrics
        overlap = len(standard_ids & hyde_ids)
        new_chunks = len(hyde_ids - standard_ids)

        metrics = HyDEMetrics(
            query=query,
            hyde_generation_ms=hyde_latency,
            standard_top5_overlap=overlap,
            hyde_improvement=new_chunks / top_k,  # Fraction of new chunks found
            hyde_document=hyde_result.hypothetical_document[:200]
        )

        self.metrics_log.append(metrics)
        self.logger.info(
            f"HyDE | query={query[:50]} | "
            f"latency={hyde_latency:.0f}ms | "
            f"overlap={overlap}/{top_k} | "
            f"new={new_chunks}/{top_k}"
        )

        return hyde_result.retrieved_chunks, metrics

    def get_hyde_help_rate(self) -> float:
        """Fraction of queries where HyDE found NEW relevant chunks."""
        if not self.metrics_log:
            return 0.0
        helped = sum(
            1 for m in self.metrics_log
            if m.standard_top5_overlap < 5  # Didn't already find everything
        )
        return helped / len(self.metrics_log)

    def get_avg_latency_overhead(self) -> float:
        """Average HyDE latency overhead in ms."""
        if not self.metrics_log:
            return 0.0
        return sum(m.hyde_generation_ms for m in self.metrics_log) / len(self.metrics_log)
```

### Example 2: The Complete Query Transformation Pipeline

```python
"""
full_query_pipeline.py — All query transformations, production-ready.
"""

from transformers import (
    HyDETransformer,
    QueryExpander,
    StepBackPrompting,
    QueryRewriter,
    QueryRouter,
)


class QueryTransformationPipeline:
    """
    Complete query transformation pipeline.

    Flow:
    1. Rewrite (if multi-turn)
    2. Classify
    3. Select strategy(ies)
    4. Execute
    5. Return results + transformation metadata
    """

    def __init__(self, llm_client, embedding_client, config: dict = None):
        self.config = config or {}
        self.llm_client = llm_client
        self.embedding_client = embedding_client

        # Initialize transformers
        self.rewriter = QueryRewriter(
            llm_client,
            model=self.config.get("model", "gpt-4o-mini")
        )
        self.router = QueryRouter(
            llm_client,
            model=self.config.get("model", "gpt-4o-mini")
        )
        self.hyde = HyDETransformer(
            llm_client,
            embedding_client,
            model=self.config.get("model", "gpt-4o-mini"),
            embedding_model=self.config.get("embedding_model", "text-embedding-3-small")
        )
        self.expander = QueryExpander(
            llm_client,
            model=self.config.get("model", "gpt-4o-mini")
        )
        self.step_back = StepBackPrompting(
            llm_client,
            model=self.config.get("model", "gpt-4o-mini")
        )

    async def transform(
        self,
        query: str,
        retriever,
        conversation_history: List[Dict] = None
    ) -> Dict:
        """Transform query and retrieve."""

        t0 = time.perf_counter()

        # Step 1: Classify and plan
        plan = await self.router.classify(query, conversation_history)

        # Step 2: Execute transformations
        chunks = []
        transformations_applied = []

        async def embed_and_search(text: str, k: int = 5) -> List:
            emb = await self.embedding_client.embeddings.create(
                model="text-embedding-3-small",
                input=text
            )
            return await retriever.search(emb.data[0].embedding, k)

        if plan.strategies == ["direct"]:
            chunks = await embed_and_search(plan.rewritten)
            transformations_applied.append("direct")

        if "hyde" in plan.strategies:
            hyde_result = await self.hyde.retrieve_hybrid(
                plan.rewritten, retriever, top_k=5
            )
            chunks = hyde_result.retrieved_chunks
            transformations_applied.append(f"hyde (hybrid)")

        if "step_back" in plan.strategies:
            chunks = await self.step_back.retrieve_with_step_back(
                plan.rewritten, retriever,
                lambda t: embed_and_search(t),
                top_k=5
            )
            transformations_applied.append("step_back")

        if "expansion" in plan.strategies:
            chunks = await self.expander.retrieve_expanded(
                plan.rewritten, retriever,
                lambda t: embed_and_search(t),
                top_k=5,
                n_variants=plan.n_variants
            )
            transformations_applied.append(f"expansion (n={plan.n_variants})")

        total_latency = (time.perf_counter() - t0) * 1000

        return {
            "chunks": chunks,
            "plan": {
                "original": plan.original,
                "rewritten": plan.rewritten,
                "complexity": plan.complexity.value,
                "strategies": plan.strategies,
                "transformations_applied": transformations_applied,
            },
            "performance": {
                "total_ms": total_latency,
                "n_chunks": len(chunks),
            }
        }


# Usage
pipeline = QueryTransformationPipeline(
    llm_client=AsyncOpenAI(),
    embedding_client=AsyncOpenAI(),
    config={"model": "gpt-4o-mini"}
)

# Simple query
result = await pipeline.transform(
    "What's the return policy?",
    retriever=my_retriever
)

# Context-dependent query
result = await pipeline.transform(
    "What about international orders?",
    retriever=my_retriever,
    conversation_history=[
        {"role": "user", "content": "What's your return policy?"},
        {"role": "assistant", "content": "Returns within 30 days..."}
    ]
)

# Analytical query
result = await pipeline.transform(
    "Why does my app crash when I upload large PDFs?",
    retriever=my_retriever
)
```

---

## ✅ Good Output Examples

### HyDE in Action

**Without HyDE:**
```
Query: "How do I get my money back?"
Query embedding → [0.3, 0.1, -0.2, ...]
Retrieved chunks: ["Payment methods: credit card, PayPal...",
                   "Contact our support team for billing issues...",
                   "Account cancellation requires 24hr notice..."]
→ Missed the refund policy entirely
```

**With HyDE:**
```
Query: "How do I get my money back?"
HyDE generates: "Customers may request refunds within 30 days of purchase.
Refunds are processed to the original payment method within 5-7 business days.
Certain categories (digital goods, personalized items) are non-refundable."

HyDE embedding → [0.7, 0.4, 0.1, ...]
Retrieved chunks: ["Refund policy: 30-day return window...",
                   "Refund processing times by payment method...",
                   "Non-refundable items list..."]
→ Hit the refund policy directly
```

### Query Expansion in Action

```
Original: "What's the impact of sleep on learning?"

Expanded queries:
  Q1: "How does REM sleep affect memory consolidation?"
  Q2: "Sleep deprivation effects on cognitive performance"
  Q3: "Relationship between sleep quality and academic performance"  
  Q4: "Neural mechanisms of sleep-dependent memory processing"
  Q5: "Optimal sleep duration for learning retention"

→ Each variant retrieves DIFFERENT chunks
→ Fused results cover sleep's impact on memory, attention, neural mechanisms, duration
→ Far more comprehensive than any single query
```

### Step-Back in Action

```
Query: "My app crashes on startup after the latest update on Ubuntu 22.04"

Step-back: "What are common causes of application startup crashes on Linux?"

Direct retrieval finds: Ubuntu 22.04-specific notes
Step-back retrieval finds: general Linux crash causes (permissions, dependencies, config)

Combined: The crash was from a missing system dependency (found by step-back),
not an Ubuntu-specific issue (what direct retrieval found).
```

---

## ❌ Antipatterns & Failure Modes

### 1. HyDE on Every Query

**What:** Applying HyDE to ALL queries regardless of need.

**Why it fails:** Adds 300-800ms latency + ~$0.001/query cost. For simple, well-formed queries, HyDE adds NO value and may introduce noise. You're paying for no benefit.

**Fix:** Always classify first. Only use HyDE when vocabulary mismatch is likely.

### 2. Expansion Without Merging Strategy

**What:** Generating 10 query variants but naively concatenating results.

**Why it fails:** The most common result across all variants dominates the merged list, drowning out rare-but-important results from unique variants.

**Fix:** Use Reciprocal Rank Fusion (RRF) or a learned merging function. Don't just union results.

### 3. Step-Back on Factual Queries

**What:** Applying step-back prompting to simple factual lookups.

```text
Query: "What's the CEO's email address?"
Step-back: "How does corporate communication work?"
→ Retrieves general communication policy, not the CEO's email
```

**Fix:** Step-back is for analytical/diagnostic questions only. Use query classification.

### 4. The Rewrite Hallucination

**What:** The query rewriter adds information that wasn't in the original query or conversation.

```text
User: "How do I reset it?"
Previous context: "What's the admin password?"
Rewritten: "How do I reset the admin password?"
→ Correct — context was about admin password

User: "How do I reset it?"
Previous context: "What's the printer IP?"
Rewritten: "How do I reset the printer IP?"
→ Wrong! User might be asking about resetting the printer ITSELF, not the IP
```

**Fix:** The rewriter should ADD context but never CHANGE the intent. Use conservative rewriting that only resolves pronouns and references, not reinterprets.

### 5. HyDE on Non-Factual Queries

**What:** Using HyDE for subjective or opinion-based queries.

```text
Query: "Which is better, AWS or Azure?"
HyDE generates a hypothetical document that makes comparative claims.
→ The embedding reflects the LLM's opinion, not the document base
→ Retrieval finds documents matching the LLM's bias, not the user's question
```

**Fix:** Don't use HyDE for subjective/comparative queries. Use expansion instead to cover multiple perspectives.

---

## 🧪 Drills & Challenges

### Drill 1: Build HyDE from Scratch (40 min)

Implement HyDE without using an LLM-based approach. Instead:

1. Take a query
2. Use a thesaurus/replacement strategy to convert question-form to statement-form
3. Expand abbreviations and acronyms
4. Embed the expanded statement and compare retrieval to raw query embedding

```python
def rule_based_hyde(query: str) -> str:
    """
    Convert query to document-like statement using rules.

    Rules:
    - "How do I X?" → "X can be done by..."
    - "What is X?" → "X is defined as..."
    - Abbreviations → expanded forms
    - Synonyms → domain vocabulary
    """
    # Your implementation
    pass
```

**Compare** the rule-based HyDE vs. LLM-based HyDE vs. standard retrieval on 10 queries. Report the precision/recall for each.

### Drill 2: Build the Expansion Benchmark (35 min)

Create an experiment that tests how many query variants are optimal.

```python
async def benchmark_expansion(
    queries: List[str],
    expander: QueryExpander,
    retriever,
    embedder,
    max_variants: int = 10
) -> Dict:
    """
    For each query, run expansion with n=1,2,3,...,10 variants.
    Measure recall@5 for each n.
    Find the optimal n.

    Expected result: recall improves up to a point, then plateaus or degrades.
    Your job: find the inflection point for YOUR data.
    """
    pass
```

### Drill 3: The Step-Back Classifier (30 min)

Build a classifier that determines whether a query would benefit from step-back prompting.

```python
def should_step_back(query: str) -> tuple:
    """
    Return (bool, reason) indicating whether step-back would help.

    Heuristics:
    - Starts with "why": YES
    - Contains "causes", "reason", "impact": YES
    - Is a simple "what is" question: NO
    - Is a lookup/reference question: NO
    - Contains a specific product name + symptom: YES
    """
    pass
```

Test on 20 queries. Measure accuracy against your judgment (or an LLM's judgment).

### Drill 4: The Multi-Turn Rewriter (45 min)

Build a query rewriter that handles these edge cases:

```python
test_cases = [
    {
        "history": [
            ("user", "How do I reset my password?"),
            ("assistant", "Go to Settings > Security > Password Reset"),
        ],
        "current": "What if I forgot my current password?",
        "expected": "How to reset password when current password is forgotten"
    },
    {
        "history": [
            ("user", "What's the refund policy?"),
            ("assistant", "Returns within 30 days in original condition"),
            ("user", "Does that apply to electronics?"),
            ("assistant", "Electronics have a 15-day window"),
        ],
        "current": "What about opened software?",
        "expected": "Refund policy for opened software"
    },
    {
        "history": [],
        "current": "What's your pricing?",
        "expected": "What's your pricing?"  # No rewrite needed
    },
]
```

For each case, implement the rewrite and compare to the expected output. Handle the case where no rewrite is needed.

### Drill 5: The Query Router Experiment (50 min)

Build the full query router and run this experiment:

1. Create 20 test queries across 5 complexity levels (4 each)
2. Run each query with: (a) direct retrieval, (b) correct strategy (per router), (c) ALL strategies
3. Measure: precision, recall, latency, cost
4. Report: Which strategies compose well? Which conflict?

**Key question:** Does the router's classification accuracy matter more than the individual strategy performance? I.e., is it better to route perfectly with a mediocre HyDE, or misroute with a perfect HyDE?

### Drill 6: The HyDE Poisoning Experiment (30 min)

Demonstrate a case where HyDE HURTS retrieval:

1. Find or create a query where HyDE generates a plausible but wrong hypothetical
2. Show that standard retrieval finds the correct document, but HyDE retrieves the wrong one
3. Design a guard that detects this situation (e.g., overlap between standard and HyDE results is very low → flag for review)

```python
async def hyde_poisoning_test(hyde_system, retriever, query: str):
    """
    Find a query where HyDE retrieves WORSE results than standard.

    Return: query, standard_result, hyde_result, and what went wrong
    """
    pass
```

---

## 🚦 Gate Check

Before proceeding to Phase 5.02 (Contextual Retrieval), you must demonstrate:

1. **Implement HyDE**: Build a working HyDE pipeline and compare it to standard retrieval on at least 10 test queries. Show a case where HyDE helps and a case where it doesn't.

2. **Build query expansion**: Generate multiple query variants and implement RRF merging. Show that expansion improves recall over single-query retrieval.

3. **Implement step-back prompting**: Demonstrate that step-back retrieval finds broader context that direct retrieval misses. Show one case where it helps and one where it doesn't.

4. **Build the query router**: Create a classifier that routes queries to the appropriate strategy. Measure its accuracy on 20+ test queries.

5. **Answer the reflection questions**:
   - When would you use HyDE vs. expansion vs. step-back?
   - How do you detect vocabulary mismatch without an LLM?
   - What's the HyDE latency budget — when is it too expensive?
   - How do you verify that your query transformation actually improved retrieval?

**Passing criteria:** You can take ANY query and choose the right transformation strategy with justification. You know when EACH technique helps, when it hurts, and when it's not worth the cost.

---

## 📚 Resources

### Papers
- **"HyDE: Precise Zero-Shot Dense Retrieval without Relevance Labels"** (Gao et al., 2022) — The original HyDE paper. Short and readable.
- **"Step-Back Prompting Enables Reasoning in LLMs"** (Zhou et al., 2023) — The step-back technique, originally for reasoning, adapted here for retrieval.
- **"Query Expansion by Prompting LLMs"** — Modern approaches to using LLMs for search query expansion.

### Production Patterns
- **Anthropic's Contextual Retrieval** — Their approach combines HyDE-like techniques with context enrichment (next module).
- **Glean's retrieval augmentation** — How a production enterprise search handles query transformation.

### Tools
- **Cohere Rerank** — Not directly query transformation, but their approach to ranking complements transformation.
- **OpenAI Embeddings** — text-embedding-3-small/large are the workhorses for embedding transformed queries.

### Your Own Code
- The **QueryRouter** class you built in this module will be a central component in your Advanced RAG Engine project (File 09).
- HyDE composes naturally with **Reranking** (File 03) — use HyDE for recall, reranking for precision.

### Expert-Level Deep Dives
- **For production engineers**: Monitor your query transformation success rate. For every transformed query, log whether the transformation improved or hurt retrieval compared to the raw query. Over time, you'll learn which query patterns benefit from which transformations — and you can hardcode the most reliable patterns.
- **For system designers**: Query transformation is a "pre-retrieval" optimization. It's cheaper than changing your index or embedding model. Before investing in re-indexing everything, try transforming the query — you might get 80% of the improvement for 10% of the cost.
- **For critical thinkers**: The best query transformation might be NONE. A significant fraction of queries in production are already well-formed for your document corpus. Adding transformation latency + cost for these queries is pure waste. The real skill isn't building HyDE — it's knowing when NOT to use it.

---

*"The query is the cheapest thing in your RAG system to change. Before you rebuild your index, retrain your embedding model, or reorganize your documents — try transforming the query first. You might be surprised how much mileage you get."*
