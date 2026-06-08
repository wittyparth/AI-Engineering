# 04 — Agentic RAG: Multi-Hop, Self-Query & Iterative Retrieval

## 🎯 Purpose & Goals

Standard RAG does one retrieval, gets one set of chunks, generates one answer. This works for simple questions where all the information lives in a single document or closely-related chunks.

Production questions rarely cooperate.

A user asks: *"What's the parental leave policy for employees in our Tokyo office, and how does it compare to our Berlin office policy?"*

Standard RAG retrieves top-5 chunks. It might get the Tokyo policy. It might get the Berlin policy. It might get both. It might get neither. It has no concept of "I need chunk A AND chunk B, and I need to compare them." It just retrieves by similarity and hopes.

**Agentic RAG** solves this by giving the retrieval system **agency** — the ability to:
1. **Decide WHAT to retrieve** — not just semantically similar text, but the specific information it needs
2. **Decide WHEN to retrieve** — retrieve once, retrieve multiple times, or not at all if it already knows
3. **Decide HOW to retrieve** — use different strategies for different information needs (keyword, semantic, metadata filter, SQL)
4. **Reason about gaps** — identify what's missing and go get it

**By the end of this module, you will:**

1. Build a **self-querying retriever** that extracts structured filters from natural language queries
2. Implement **query decomposition** — breaking complex questions into sub-questions that get answered independently
3. Design **multi-hop retrieval** — where the answer from one retrieval determines what to retrieve next
4. Build **Corrective RAG (CRAG)** — an agent that evaluates retrieved chunks and decides whether to use them, refine the query, or give up
5. Implement **Active RAG** — an agent that decides dynamically whether to retrieve based on its existing knowledge
6. Build a **LangGraph-based retrieval agent** with state management
7. Know the **cost-latency-quality tradeoffs** of agentic vs. standard RAG
8. Debug the **failure modes that are unique** to agentic retrieval loops

---

### ⏹ STOP. Answer these before proceeding.

**Question 1 — The Two-Document Question**

*A user asks: "I'm a senior manager based in the Singapore office. Am I eligible for the cross-regional assignment program?"*

*After retrieval, your system finds:*
- *Doc A: "Senior managers and above are eligible for cross-regional assignments, subject to business unit approval."*
- *Doc B: "The Singapore office participates in the cross-regional program, but assignments are limited to APAC-region posts."*

*The answer is YES (senior manager qualifies, Singapore participates), but with a constraint (APAC only). The correct answer requires BOTH documents. Retrieval that returns only Doc A gives an incomplete answer. Retrieval that returns only Doc B gives a wrong answer (it doesn't know who qualifies).*

*How would you design a system that KNOWS it needs both documents? What signal tells you "I have partial information, I need more"?*

*What if there were 3 documents needed? 5? How does the system know when it has ENOUGH to answer, versus when it needs to keep retrieving?*

**Question 2 — The Iterative Discovery Problem**

*Your company has a complex product with documentation spread across 12 different systems: API docs, SDK guides, architecture decisions records (ADRs), changelogs, internal wikis, and Slack archives.*

*A user asks: "I'm migrating from our legacy payments API v2 to v3. Do I need to change the webhook signature verification, and what's the migration window?"*

*The answer lives in MULTIPLE places:*
- *The v3 migration guide says webhooks now use ED25519 instead of HMAC-SHA256*
- *The changelog mentions the migration window is Q3 2025 to Q1 2026*
- *An ADR from 2024 explains WHY the change was made (security audit finding)*

*No single retrieval call gets all three. You need to:*
1. *Find the migration guide → learn about webhook changes*
2. *Notice there's a migration window mentioned → search for "migration window v3"*
3. *Find conflicting info about webhooks → search for ADR about the decision*

*Each step depends on what the previous step found. You don't know in advance what you'll need to search for next — the search strategy EMERGES from what you discover.*

*How would you design a retrieval system that can do this kind of exploratory, iterative discovery? What tells the system "stop searching, I have enough to answer"?*

**Question 3 — The Conflicting Evidence Problem**

*A user asks: "What's the maximum file size for upload?"*

*Your system retrieves two chunks:*
- *Chunk A (from the public docs): "The maximum file size for upload is 25MB."*
- *Chunk B (from an internal FAQ): "Enterprise customers can upload files up to 100MB."*

*Both chunks are relevant. Both are from authoritative sources. But they CONTRADICT each other.*
*Standard RAG dumps both into the context and hopes the LLM resolves the contradiction. Sometimes the LLM notices and explains the discrepancy. Sometimes it picks one and ignores the other. Sometimes it hallucinates a middle ground ("uploads are limited to 62.5MB").*

*How would you build a system that:*
1. *Detects contradictions in retrieved chunks*
2. *Resolves them (or escalates them) intelligently*
3. *Provides a clear, truthful answer that accounts for the discrepancy*

*What if the contradiction is SUBTLE — not a direct disagreement, but a difference in framing that leads to different answers?*

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 15 min |
| The Agentic RAG Landscape | 15 min |
| Self-Query Retrieval | 45 min |
| Query Decomposition | 35 min |
| Multi-Hop Retrieval | 45 min |
| Corrective RAG (CRAG) | 30 min |
| Active RAG | 25 min |
| LangGraph Agent Implementation | 60 min |
| Production Considerations | 15 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~6.5 hours** |

---

## 📖 Concept Deep-Dive

### 1. The Agentic RAG Spectrum

Agentic RAG isn't one technique — it's a SPECTRUM of systems that give the retrieval process more agency. Understanding this spectrum helps you choose the RIGHT level of complexity for your use case:

```
LOW AGENCY ──────────────────────────────────────────────► HIGH AGENCY

[Standard RAG] ─→ [Self-Query] ─→ [Multi-Hop] ─→ [CRAG] ─→ [Active RAG] ─→ [Full Agent]

Cost/Latency increase ─────────────────────────────────────────────────────►
Complexity increase ──────────────────────────────────────────────────────►
Failure mode variety increase ────────────────────────────────────────────►
Capability increase ──────────────────────────────────────────────────────►
```

| Level | What It Does | When to Use | Cost Add |
|-------|-------------|-------------|----------|
| **Standard RAG** | One retrieval, one generation | Simple Q&A from single sources | Baseline |
| **Self-Query** | Extracts filters from query | Multi-tenant, versioned, or filtered corpora | +1 LLM call |
| **Multi-Hop** | Sequential retrieval, each informed by previous | Questions spanning multiple documents | +2-5 LLM calls |
| **Corrective RAG** | Evaluates retrieved chunks, decides next action | When retrieval quality is unreliable | +2-4 LLM calls |
| **Active RAG** | Decides WHEN to retrieve vs. use knowledge | When the model has relevant training knowledge | +1-3 LLM calls |
| **Full Agent** | Full tool-use loop with reasoning | Complex exploratory research | 5-20+ LLM calls |

**The critical insight:** Most production RAG systems should sit at the **Self-Query** or **Multi-Hop** level. Only truly complex research scenarios need the full agent loop. Start simple, measure failures, add complexity only where it helps.

---

### 2. Self-Query Retrieval

**The problem:** A user asks "What's the rate limit for the Enterprise tier?" Your vector database has chunks with metadata `{"tier": "free"}, {"tier": "pro"}, {"tier": "enterprise"}` and `{"tier": "enterprise"}`. Semantic search might retrieve the free tier chunk if it uses similar language — and the LLM might answer based on the WRONG chunk.

**Self-querying** solves this by having the LLM extract STRUCTURED FILTERS from the natural language query BEFORE retrieval:

```
User query: "What's the rate limit for the Enterprise tier?"

Self-query step:
  LLM extracts: {
    "query": "API rate limit requests per minute or second",
    "filter": {"tier": "enterprise"}
  }

Retrieval: Search with query embed + apply {"tier": "enterprise"} filter
→ Only returns enterprise-tier chunks
→ Free tier and pro tier chunks are EXCLUDED even if semantically similar
```

**Why this matters in production:**

- **Multi-tenant systems:** Every query implicitly has a tenant filter. If user from Acme Corp asks about "employee count," you must only search Acme's docs, not all docs.
- **Versioned documentation:** "What changed in v3.2?" should only retrieve v3.2 changelog entries, not v3.1 or v4.0.
- **Tiered products:** "Can I use this on the free plan?" must only consider free plan constraints.
- **Date-ranged data:** "What were the Q4 2024 sales numbers?" must filter to Q4 2024.
- **Permission boundaries:** "How do I configure the admin panel?" must respect user role filters.

**The self-query pipeline:**

```
[User Query]
     │
     ▼
[LLM Extraction]
     │  query: str — the search text (rewritten for retrieval)
     │  filter: Dict — metadata filters (exact, range, fuzzy)
     │  top_k: int — how many results to retrieve
     ▼
[Execute Search]
     │  vector_search(embed(query), filter=filter, top_k=top_k)
     ▼
[Return Filtered Results]
```

---

### 3. Query Decomposition

**The problem:** A question like "Compare the onboarding flows for mobile and web users" is actually TWO questions: "What's the mobile onboarding flow?" and "What's the web onboarding flow?" combined with a comparison instruction.

Query decomposition splits the complex question into simpler sub-questions, retrieves for each, and combines results:

```
Original query: "Compare the onboarding flows for mobile and web users."

Decomposition:
  1. "What is the mobile app onboarding flow?"
  2. "What is the web application onboarding flow?"

Retrieval (independent, can be parallel):
  1. Chunks about mobile onboarding
  2. Chunks about web onboarding

Merge → Generate comparison answer
```

**Types of decomposition:**

| Type | Pattern | Example |
|------|---------|---------|
| **Parallel** | Sub-questions are independent, can be retrieved simultaneously | "Compare X and Y" → [X info, Y info] |
| **Sequential** | Sub-question depends on previous answer | "Who wrote the paper about X, and what else did they write?" |
| **Conditional** | Sub-question depends on the answer to a yes/no gate | "Do we support OAuth? If so, how do I configure it?" |
| **Hierarchical** | Sub-questions form a tree, results get merged up | "What's the org chart and who handles what responsibilities?" |

**The key insight:** Decomposition turns ONE hard retrieval problem into MULTIPLE easy ones. Each sub-query is simpler and more focused than the original compound query.

---

### 4. Multi-Hop Retrieval

Multi-hop retrieval is the SEQUENTIAL form of decomposition. Each retrieval step is INFORMED by the previous step's results:

```
Hop 1: "Who is the CTO of Acme Corp?"
  → Retrieves: "The CTO of Acme Corp is Dr. Jane Smith"

Hop 2: "What are Dr. Jane Smith's research interests?"
  → Retrieves: "Dr. Jane Smith's research focuses on NLP and reinforcement learning"

Hop 3: "What NLP papers did Dr. Jane Smith publish in 2024?"
  → Retrieves: "In 2024, Dr. Smith published 3 papers on transformer efficiency..."
```

Each hop uses information from the previous hop to construct a better query. The system doesn't know in advance how many hops it needs — it discovers the path as it goes.

**When multi-hop is necessary:**

- **Referential questions** — "What's his salary?" (Who is "his"? You need the antecedent first)
- **Bridging questions** — "Do I qualify for the program?" (Need to determine qualification THEN check program details)
- **Exploratory questions** — "What should I know before migrating?" (The answer determines what's relevant)
- **Cumulative questions** — "Summarize the changes across the last 3 versions" (Need each version's changelog)

**The critical challenge:** How does the system know when to STOP hopping? Three strategies:

1. **Fixed depth** — Always do N hops. Simple but wasteful.
2. **Sufficiency check** — After each hop, ask "Can I answer the question with what I have?" Stop when yes.
3. **Gap analysis** — Ask "What's missing to answer this question?" If nothing, stop. If something, hop for it.

---

### 5. Corrective RAG (CRAG)

CRAG (Yan et al., 2024) adds a CRITIQUE step between retrieval and generation. The system evaluates whether retrieved chunks are good enough, and decides what to do next:

```
[User Query]
     │
     ▼
[Retrieve] ──→ [Relevance Evaluator]
                    │
              ┌─────┼─────┐
              │     │     │
              ▼     ▼     ▼
          [Good] [Bad] [Uncertain]
              │     │     │
              │     ▼     │
              │ [Refine  ]│
              │  Query +  │
              │  Retry    │
              │     │     │
              └──┬──┘     │
                 │        │
                 ▼        ▼
            [Generate]   [Fallback]
```

**The three decisions:**

| Evaluation | Action | Rationale |
|------------|--------|-----------|
| **Relevant** (confidence > threshold) | Generate from retrieved chunks | Standard RAG path |
| **Irrelevant** (confidence < low threshold) | Refine query and retry, or fallback | The model had hope but failed |
| **Ambiguous** (confidence in middle) | Refine query AND keep original chunks for second pass | The chunks might be useful with a better query |

**The relevance evaluator** is critical. Two approaches:

1. **LLM-as-judge:** Ask the LLM "Is this chunk relevant to answering the user's question?" Expensive but flexible.
2. **Trained classifier:** Fine-tune a small model (e.g., DeBERTa) to classify relevance. Cheaper but requires training data.

CRAG's key advantage: it catches BAD RETRIEVAL before it reaches the generator. In production, 10-30% of retrievals return irrelevant chunks. CRAG prevents those from contaminating the answer.

---

### 6. Active RAG

Active RAG (or "Self-RAG") takes agency one step further. Instead of always retrieving, the system decides WHETHER TO RETRIEVE at all:

```
Query: "What is the capital of France?"

Active RAG:
  1. Reflection: "Do I know this from training data? Yes, Paris."
  2. Decision: SKIP retrieval, generate from knowledge.
  3. Output: "Paris"

Query: "What's the current API rate limit for our production cluster?"

Active RAG:
  1. Reflection: "This is specific to their deployment. I don't know this."
  2. Decision: RETRIEVE from their documentation.
  3. Retrieve → Generate from chunks.
```

**When to skip retrieval:**

- **Factual knowledge** the model was trained on (common knowledge, well-known facts)
- **Instructions** about how to do common tasks (model knows general patterns)
- **Creative tasks** where retrieval would constrain output

**When to always retrieve:**

- **Specific to a private corpus** (company docs, personal data)
- **Time-sensitive information** (prices, availability, recent changes)
- **Numbers and statistics** (the model guesses wrong too often)
- **Multi-hop questions** where you need document-specific evidence

The self-reflection step adds overhead, so it's only worthwhile when a significant fraction of queries can skip retrieval. In most production RAG systems, 90%+ of queries DO need retrieval (otherwise, why have the RAG system?), so Active RAG is usually overkill.

---

### 7. Full Agentic RAG (with LangGraph)

The full agent loop combines ALL the above capabilities:

```
State: {
  query: str,
  chat_history: List[Message],
  retrieved_chunks: List[Chunk],
  current_plan: List[Step],
  completed_steps: List[Step],
  gaps: List[str],
  final_answer: Optional[str]
}

Loop:
  1. REASON: "What do I need to answer this question?"
  2. PLAN: "I need chunk A and chunk B. Let me get chunk A first."
  3. ACT: Retrieve chunk A
  4. OBSERVE: "Chunk A says X. But I also need chunk B."
  5. REASON: "Now I need chunk B. The search for chunk B should use info from chunk A."
  6. ACT: Retrieve chunk B with refined query
  7. OBSERVE: "I have both A and B. Do I have enough?"
  8. EVALUATE: "Yes, I can answer now."
  9. GENERATE: Final answer with citations to A and B
```

This is implemented as a **state machine** — the agent has a state, takes actions, observes results, and updates its state. LangGraph is purpose-built for this pattern.

---

## 💻 Code Examples

### 1. Self-Query Retriever

```python
"""
self_query.py — Self-querying retriever that extracts structured filters
from natural language queries before performing retrieval.

This is the simplest form of agentic RAG and should be your default
before adding more complex agent behavior.
"""

from typing import List, Optional, Dict, Any, Literal
from dataclasses import dataclass, field
from datetime import datetime
import json
import re

from openai import AsyncOpenAI
import numpy as np


# ─── Data Models ────────────────────────────────────────────────────────────

@dataclass
class FilterCondition:
    """A structured filter condition for metadata filtering."""
    field: str
    operator: Literal["eq", "neq", "gt", "gte", "lt", "lte", "in", "contains"]
    value: Any

    def to_mongo_filter(self) -> Dict[str, Any]:
        """Convert to a MongoDB-style filter dict."""
        operator_map = {
            "eq": "$eq",
            "neq": "$ne",
            "gt": "$gt",
            "gte": "$gte",
            "lt": "$lt",
            "lte": "$lte",
            "in": "$in",
            "contains": "$regex"
        }
        op = operator_map[self.operator]

        # Special handling for contains → regex
        if self.operator == "contains":
            return {self.field: {op: f".*{re.escape(str(self.value))}.*", "$options": "i"}}

        return {self.field: {op: self.value}}

    @classmethod
    def from_expression(cls, expression: Dict) -> "FilterCondition":
        """Create from a parsed filter expression dict."""
        return cls(
            field=expression["field"],
            operator=expression["operator"],
            value=expression["value"]
        )


@dataclass
class SelfQueryResult:
    """The result of a self-query extraction + retrieval."""
    natural_query: str
    extracted_query: str
    filters: List[FilterCondition]
    top_k: int
    retrieved_chunks: List[Dict[str, Any]]
    extraction_latency_ms: float
    retrieval_latency_ms: float


# ─── Self-Query Extractor ───────────────────────────────────────────────────

class SelfQueryExtractor:
    """
    Extract structured search parameters from natural language queries.

    Given "What's the rate limit for enterprise tier users in the US region?"
    extracts: {query: "API rate limit", filters: [tier=enterprise, region=US]}
    """

    SYSTEM_PROMPT = """You are a search query analyzer. Extract structured search parameters 
from user questions for a RAG system.

Your job is to determine:
1. The CORE SEARCH QUERY — the text that should be embedded and searched
2. METADATA FILTERS — structured filters to apply to the search
3. HOW MANY RESULTS to retrieve

Available filter fields and their types:
{available_filters}

Rules:
- The core query should capture the INFORMATION NEED, not the question format
- Filters should ONLY use the available fields listed above
- If you're unsure about a filter value, set confidence to "low"
- If no filters apply, return an empty filters list
- Extract a clean query — remove conversational filler, keep key terms

Return your analysis as JSON with exactly this structure:
{{
    "query": str,              // The cleaned, embeddable search query
    "filters": [               // List of filter conditions
        {{
            "field": str,       // Field name from available_filters
            "operator": "eq" | "neq" | "gt" | "gte" | "lt" | "lte" | "in" | "contains",
            "value": str | int | float | List,
            "confidence": "high" | "medium" | "low"
        }}
    ],
    "top_k": int,              // Number of results to return (5-20)
    "reasoning": str           // Brief explanation of extraction decisions
}}
"""

    def __init__(
        self,
        llm_client: AsyncOpenAI,
        model: str = "gpt-4o-mini",
        available_filters: Optional[Dict[str, str]] = None
    ):
        self.llm_client = llm_client
        self.model = model

        # Default available filters — customize for your document corpus
        self.available_filters = available_filters or {
            "tier": "Product tier (free, pro, enterprise)",
            "product": "Product or service name",
            "doc_type": "Type of document (guide, changelog, api-ref, faq, tutorial)",
            "version": "Software version (semver format)",
            "date": "Date (ISO format YYYY-MM-DD)",
            "date_range_start": "Start of a date range",
            "date_range_end": "End of a date range",
            "author": "Document author username",
            "department": "Department name (engineering, product, design, sales)",
            "region": "Geographic region (US, EU, APAC, etc.)",
            "language": "Document language code (en, ja, de, fr, es, zh)",
        }

    def _build_system_prompt(self) -> str:
        """Build the system prompt with available filter context."""
        avail = "\n".join(
            f"  - {field}: {desc}"
            for field, desc in self.available_filters.items()
        )
        return self.SYSTEM_PROMPT.replace("{available_filters}", avail)

    async def extract(
        self,
        query: str,
        chat_history: Optional[List[Dict]] = None
    ) -> Dict[str, Any]:
        """
        Extract structured search parameters from a natural language query.

        Args:
            query: The user's natural language question
            chat_history: Optional chat history for context

        Returns:
            Dict with query, filters, top_k, and reasoning
        """
        messages = [{"role": "system", "content": self._build_system_prompt()}]

        # Add chat history context if available
        if chat_history:
            for msg in chat_history[-4:]:  # Last 4 messages for context
                messages.append(msg)

        messages.append({"role": "user", "content": query})

        import time
        start = time.time()

        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=messages,
            response_format={"type": "json_object"},
            temperature=0.0,  # Deterministic extraction
            max_tokens=500,
        )

        extraction_ms = (time.time() - start) * 1000

        try:
            result = json.loads(response.choices[0].message.content)

            # Validate required fields
            if "query" not in result:
                result["query"] = query  # Fallback to original query
            if "filters" not in result:
                result["filters"] = []
            if "top_k" not in result:
                result["top_k"] = 10
            if "reasoning" not in result:
                result["reasoning"] = ""

            # Remove low-confidence filters
            result["filters"] = [
                f for f in result["filters"]
                if f.get("confidence", "low") != "low"
            ]

            result["extraction_ms"] = extraction_ms
            return result

        except (json.JSONDecodeError, KeyError) as e:
            # Fallback: use the original query as-is
            return {
                "query": query,
                "filters": [],
                "top_k": 10,
                "reasoning": f"Extraction failed: {e}. Using raw query.",
                "extraction_ms": extraction_ms
            }


# ─── Self-Query Retriever ───────────────────────────────────────────────────

class SelfQueryRetriever:
    """
    Combined retriever that extracts structured parameters and searches.

    This wraps a standard vector retriever with the self-query layer,
    applying metadata filters before returning results.
    """

    def __init__(
        self,
        extractor: SelfQueryExtractor,
        vector_search_fn: callable,  # async fn(query_embedding, top_k, filter) -> list[dict]
    ):
        self.extractor = extractor
        self.vector_search = vector_search_fn

    async def retrieve(
        self,
        query: str,
        chat_history: Optional[List[Dict]] = None,
        embed_fn: Optional[callable] = None,
        top_k_default: int = 10,
    ) -> SelfQueryResult:
        """
        Perform self-query retrieval.

        1. Extract structured parameters from natural language
        2. Build metadata filter from extracted conditions
        3. Embed the extracted query
        4. Search with embedding + filter
        5. Return combined result
        """
        import time

        # Step 1: Extract
        extraction_start = time.time()
        extracted = await self.extractor.extract(query, chat_history)
        extraction_ms = (time.time() - extraction_start) * 1000

        # Step 2: Build filter
        filter_conditions = [
            FilterCondition.from_expression(f)
            for f in extracted["filters"]
        ]
        combined_filter = self._build_combined_filter(filter_conditions)

        # Step 3: Embed and search
        retrieval_start = time.time()
        if embed_fn:
            query_embedding = await embed_fn(extracted["query"])
        else:
            query_embedding = extracted["query"]  # Assume string if no embedder

        chunks = await self.vector_search(
            query_embedding,
            top_k=extracted.get("top_k", top_k_default),
            filter=combined_filter
        )
        retrieval_ms = (time.time() - retrieval_start) * 1000

        return SelfQueryResult(
            natural_query=query,
            extracted_query=extracted["query"],
            filters=filter_conditions,
            top_k=extracted.get("top_k", top_k_default),
            retrieved_chunks=chunks,
            extraction_latency_ms=extraction_ms,
            retrieval_latency_ms=retrieval_ms,
        )

    def _build_combined_filter(
        self, conditions: List[FilterCondition]
    ) -> Optional[Dict[str, Any]]:
        """Combine multiple filter conditions into a single filter dict."""
        if not conditions:
            return None

        # All conditions are AND-ed together
        if len(conditions) == 1:
            return conditions[0].to_mongo_filter()

        return {
            "$and": [c.to_mongo_filter() for c in conditions]
        }

    async def retrieve_no_filters(
        self,
        query: str,
        embed_fn: Optional[callable] = None,
        top_k: int = 10,
    ) -> SelfQueryResult:
        """
        Baseline retrieval WITHOUT self-query filtering.
        For A/B testing.
        """
        import time

        extraction_ms = 0.0

        retrieval_start = time.time()
        if embed_fn:
            query_embedding = await embed_fn(query)
        else:
            query_embedding = query

        chunks = await self.vector_search(query_embedding, top_k=top_k, filter=None)
        retrieval_ms = (time.time() - retrieval_start) * 1000

        return SelfQueryResult(
            natural_query=query,
            extracted_query=query,
            filters=[],
            top_k=top_k,
            retrieved_chunks=chunks,
            extraction_latency_ms=extraction_ms,
            retrieval_latency_ms=retrieval_ms,
        )


# ─── A/B Test Harness ───────────────────────────────────────────────────────

class SelfQueryBenchmark:
    """
    Compare self-query retrieval vs. standard retrieval.

    Run this on a set of test queries with known relevant results
    to measure whether self-query improves precision/recall.
    """

    @staticmethod
    async def compare(
        retriever: SelfQueryRetriever,
        test_cases: List[Dict],  # [{"query": str, "relevant_ids": List[str], "filters_expected": List}]
        embed_fn: callable,
    ) -> Dict:
        """
        Run A/B comparison.

        Args:
            retriever: The SelfQueryRetriever instance
            test_cases: List of test queries with ground truth
            embed_fn: Function to embed text

        Returns:
            Dict with comparison metrics
        """
        results = {"standard": [], "self_query": [], "summary": {}}

        for case in test_cases:
            query = case["query"]
            relevant = set(case["relevant_ids"])

            # Standard retrieval (no filters)
            std = await retriever.retrieve_no_filters(query, embed_fn)
            std_ids = set(c["id"] for c in std.retrieved_chunks)
            std_precision = len(std_ids & relevant) / max(len(std_ids), 1)
            std_recall = len(std_ids & relevant) / max(len(relevant), 1)
            std_f1 = 2 * std_precision * std_recall / max(std_precision + std_recall, 1e-10)

            # Self-query retrieval
            sq = await retriever.retrieve(query, embed_fn=embed_fn)
            sq_ids = set(c["id"] for c in sq.retrieved_chunks)
            sq_precision = len(sq_ids & relevant) / max(len(sq_ids), 1)
            sq_recall = len(sq_ids & relevant) / max(len(relevant), 1)
            sq_f1 = 2 * sq_precision * sq_recall / max(sq_precision + sq_recall, 1e-10)

            # Check if filters were correctly extracted
            expected_filters = set(case.get("filters_expected", []))
            actual_filters = set(
                (f.field, f.operator, str(f.value))
                for f in sq.filters
            )
            filter_accuracy = len(expected_filters & actual_filters) / max(len(expected_filters), 1)

            results["standard"].append({
                "query": query,
                "precision": std_precision,
                "recall": std_recall,
                "f1": std_f1,
            })

            results["self_query"].append({
                "query": query,
                "precision": sq_precision,
                "recall": sq_recall,
                "f1": sq_f1,
                "filter_accuracy": filter_accuracy,
                "extracted_filters": [(f.field, f.operator, f.value) for f in sq.filters],
            })

        # Compute aggregate metrics
        for approach in ["standard", "self_query"]:
            precisions = [r["precision"] for r in results[approach]]
            recalls = [r["recall"] for r in results[approach]]
            f1s = [r["f1"] for r in results[approach]]

            results["summary"][approach] = {
                "avg_precision": sum(precisions) / len(precisions),
                "avg_recall": sum(recalls) / len(recalls),
                "avg_f1": sum(f1s) / len(f1s),
            }

        # Compute filter accuracy separately
        filter_accs = [
            r.get("filter_accuracy", 0)
            for r in results["self_query"]
        ]
        results["summary"]["self_query"]["avg_filter_accuracy"] = (
            sum(filter_accs) / len(filter_accs) if filter_accs else 0
        )

        return results


# ─── Usage Example ──────────────────────────────────────────────────────────

async def example_usage():
    """Demonstrate self-query retrieval with a mock vector store."""
    from openai import AsyncOpenAI

    client = AsyncOpenAI()

    # Mock vector store for demonstration
    mock_docs = [
        {"id": "doc_1", "text": "Free tier: 1000 requests/hour", "metadata": {"tier": "free", "product": "api"}},
        {"id": "doc_2", "text": "Pro tier: 10000 requests/hour", "metadata": {"tier": "pro", "product": "api"}},
        {"id": "doc_3", "text": "Enterprise tier: 100000 requests/hour", "metadata": {"tier": "enterprise", "product": "api"}},
        {"id": "doc_4", "text": "Free tier: 1 concurrent connection", "metadata": {"tier": "free", "product": "websocket"}},
        {"id": "doc_5", "text": "Enterprise tier: unlimited concurrent connections", "metadata": {"tier": "enterprise", "product": "websocket"}},
    ]

    async def mock_vector_search(embedding, top_k=10, filter=None):
        """Mock vector search that respects filters."""
        results = mock_docs.copy()

        # Apply metadata filter
        if filter:
            filtered = []
            for doc in results:
                meta = doc["metadata"]
                match = True
                for field, cond in filter.items():
                    if field == "$and":
                        for sub in cond:
                            for f, op in sub.items():
                                op_name = list(op.keys())[0]
                                val = list(op.values())[0]
                                if op_name == "$eq" and meta.get(f) != val:
                                    match = False
                    elif field in meta:
                        if "$eq" in cond and meta[field] != cond["$eq"]:
                            match = False
                    else:
                        match = False
                if match:
                    filtered.append(doc)
            results = filtered

        # Simulate embedding similarity — in production this would be real vector search
        # For this example, just return top_k
        return results[:top_k]

    async def mock_embed(text: str):
        """Mock embedding — in production, use real embedder."""
        return text  # Just pass through for this example

    # Build the system
    extractor = SelfQueryExtractor(
        llm_client=client,
        model="gpt-4o-mini",
    )

    retriever = SelfQueryRetriever(
        extractor=extractor,
        vector_search_fn=mock_vector_search,
    )

    # Test queries
    test_queries = [
        "What's the rate limit for the enterprise tier?",
        "How many concurrent connections on free?",
        "What's the difference in limits between tiers?",
        "Show me websocket limits for enterprise customers",
    ]

    for query in test_queries:
        print(f"\n{'='*60}")
        print(f"QUERY: {query}")
        print(f"{'='*60}")

        # Self-query retrieval
        result = await retriever.retrieve(query, embed_fn=mock_embed)

        print(f"\nExtracted query: '{result.extracted_query}'")
        print(f"Filters: {[f'{f.field} {f.operator} {f.value}' for f in result.filters]}")
        print(f"Top-K: {result.top_k}")
        print(f"Latency: {result.extraction_latency_ms:.0f}ms (extraction) + {result.retrieval_latency_ms:.0f}ms (retrieval)")

        print(f"\nRetrieved chunks:")
        for chunk in result.retrieved_chunks:
            print(f"  • [{chunk['id']}] (tier={chunk['metadata']['tier']}) {chunk['text'][:80]}")
```

**What to notice:**
- The extractor uses `response_format: json_object` for reliable structured output
- Low-confidence filters are dropped to prevent false positives
- The A/B test harness lets you measure whether self-query actually improves recall/precision for YOUR data
- The `retrieve_no_filters` method gives you a direct baseline comparison

---

### 2. Query Decomposition & Multi-Hop Retrieval

```python
"""
multi_hop.py — Query decomposition and multi-hop retrieval.

Transforms a complex question into a series of simpler retrievals,
where each step can depend on information from previous steps.
"""

from typing import List, Optional, Dict, Any, Callable, Awaitable
from dataclasses import dataclass, field
from enum import Enum
import json
import asyncio
import time

from openai import AsyncOpenAI


# ─── Data Models ────────────────────────────────────────────────────────────

class HopType(Enum):
    """Type of retrieval hop."""
    SEMANTIC = "semantic"           # Standard vector search
    KEYWORD = "keyword"             # Keyword/exact match search
    METADATA = "metadata_filter"    # Filter-based search
    SQ = "self_query"               # Self-query search (extract + embed)


@dataclass
class HopResult:
    """Result of a single retrieval hop."""
    step_number: int
    query_used: str
    hop_type: HopType
    retrieved_chunks: List[Dict[str, Any]]
    answer_so_far: Optional[str]  # What the system understood from this hop
    gaps_identified: List[str]    # What's still missing
    latency_ms: float


@dataclass
class DecompositionPlan:
    """Plan for decomposing a complex query into sub-queries."""
    original_query: str
    sub_queries: List[Dict]  # [{"query": str, "depends_on": List[int]}]
    merge_instructions: str
    full_plan: str


@dataclass
class MultiHopResult:
    """Complete result of multi-hop retrieval."""
    original_query: str
    hops: List[HopResult]
    merged_chunks: List[Dict[str, Any]]
    total_latency_ms: float
    total_tokens: int


# ─── Query Decomposer ───────────────────────────────────────────────────────

class QueryDecomposer:
    """
    Decompose complex questions into sequences of simpler sub-questions.

    For a question like "Compare the onboarding flows for mobile and web users":
    Decomposes into: [
        {"query": "What is the mobile app onboarding flow?", "depends_on": []},
        {"query": "What is the web application onboarding flow?", "depends_on": []},
    ]
    Merge instruction: "Compare the two flows, highlighting key differences"
    """

    DECOMPOSITION_PROMPT = """You are a query decomposition specialist for a RAG system.

Given a complex user question, break it down into SIMPLER sub-questions
that can each be answered by retrieving a small set of relevant documents.

RULES:
1. Each sub-query should be INDEPENDENTLY searchable (good embedding)
2. If a sub-query depends on the answer of another, mark it with depends_on
3. Parallel sub-queries (no dependencies) can be executed simultaneously
4. Maximum 6 sub-queries total
5. Keep sub-queries focused — each should retrieve ONE piece of information

Return JSON:
{{
    "sub_queries": [
        {{
            "id": 0,
            "query": "Specific, searchable sub-query text",
            "depends_on": []  // IDs of sub-queries that must complete first
        }}
    ],
    "merge_instructions": "Instructions for combining the answers",
    "reasoning": "Why this decomposition makes sense"
}}
"""

    def __init__(self, llm_client: AsyncOpenAI, model: str = "gpt-4o-mini"):
        self.llm_client = llm_client
        self.model = model

    async def decompose(self, query: str) -> DecompositionPlan:
        """Decompose a complex query into a plan."""
        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": self.DECOMPOSITION_PROMPT},
                {"role": "user", "content": query}
            ],
            response_format={"type": "json_object"},
            temperature=0.1,
            max_tokens=1000,
        )

        try:
            data = json.loads(response.choices[0].message.content)

            return DecompositionPlan(
                original_query=query,
                sub_queries=data["sub_queries"],
                merge_instructions=data.get("merge_instructions", "Combine the answers naturally"),
                full_plan=data.get("reasoning", "")
            )
        except (json.JSONDecodeError, KeyError) as e:
            # Fallback: single query
            return DecompositionPlan(
                original_query=query,
                sub_queries=[{"id": 0, "query": query, "depends_on": []}],
                merge_instructions="Answer the question directly.",
                full_plan=f"Decomposition failed ({e}). Using original query."
            )

    async def recover_sub_queries(
        self,
        query: str,
        failed_plan: DecompositionPlan,
        error_info: str
    ) -> DecompositionPlan:
        """
        Recover when a decomposition plan fails during execution.

        This is important for production resilience — decomposition
        can produce plans that don't work in practice.
        """
        recovery_prompt = f"""
The original query was: {query}

The decomposition plan was:
{json.dumps([s["query"] for s in failed_plan.sub_queries], indent=2)}

Execution failed with error: {error_info}

Please create a SIMPLER decomposition with fewer sub-queries, or
suggest using the original query as a single search.
"""
        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": self.DECOMPOSITION_PROMPT},
                {"role": "user", "content": recovery_prompt}
            ],
            response_format={"type": "json_object"},
            temperature=0.0,
            max_tokens=1000,
        )

        try:
            data = json.loads(response.choices[0].message.content)
            return DecompositionPlan(
                original_query=query,
                sub_queries=data["sub_queries"],
                merge_instructions=data.get("merge_instructions", "Combine the answers naturally"),
                full_plan=data.get("reasoning", f"Recovered from: {error_info}")
            )
        except (json.JSONDecodeError, KeyError):
            # Final fallback: single query
            return DecompositionPlan(
                original_query=query,
                sub_queries=[{"id": 0, "query": query, "depends_on": []}],
                merge_instructions="Answer the question directly.",
                full_plan="Emergency fallback: using original query as single search."
            )


# ─── Gap Analyzer ───────────────────────────────────────────────────────────

class GapAnalyzer:
    """
    After each retrieval hop, analyze what's still missing.

    This tells the multi-hop system whether to:
    - Continue to the next planned hop (sufficient info)
    - Plan a NEW hop for unexpected information
    - Stop and generate the answer (all info gathered)
    """

    GAP_ANALYSIS_PROMPT = """You are analyzing whether retrieved information is sufficient
to answer a user's question.

Given:
- The original question
- The information retrieved so far (from all hops)
- The planned next step (if any)

Determine:
1. Can the question be answered NOW with the information available?
2. If NOT, what EXACTLY is missing?
3. What specific query would find the missing information?
4. Should we continue with the planned search or adjust?

Return JSON:
{{
    "can_answer": true/false,
    "confidence": "high" | "medium" | "low",
    "missing_information": ["Specific piece 1", "Specific piece 2"],
    "suggested_query": "A new query to find missing info" or null,
    "reasoning": "Analysis of what we have and what's missing"
}}
"""

    def __init__(self, llm_client: AsyncOpenAI, model: str = "gpt-4o-mini"):
        self.llm_client = llm_client
        self.model = model

    async def analyze(
        self,
        query: str,
        retrieved_chunks: List[Dict[str, Any]],
        planned_sub_queries: List[Dict],
        completed_hop_ids: List[int],
    ) -> Dict:
        """Analyze gaps in retrieved information."""
        context = "\n\n".join(
            f"[Chunk from hop {c.get('hop', '?')}] {c.get('text', str(c))[:500]}"
            for c in retrieved_chunks[-5:]  # Last 5 chunks for context
        )

        planned = "\n".join(
            f"  Hop {sq['id']}: {sq['query']} {'(COMPLETED)' if sq['id'] in completed_hop_ids else '(PENDING)'}"
            for sq in planned_sub_queries
        )

        prompt = f"""Original question: {query}

Information retrieved so far:
{context}

Planned retrieval steps:
{planned}

Analyze whether we can answer the question with current information."""
        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": self.GAP_ANALYSIS_PROMPT},
                {"role": "user", "content": prompt}
            ],
            response_format={"type": "json_object"},
            temperature=0.0,
            max_tokens=500,
        )

        try:
            return json.loads(response.choices[0].message.content)
        except json.JSONDecodeError:
            return {
                "can_answer": True,
                "confidence": "low",
                "missing_information": [],
                "suggested_query": None,
                "reasoning": "Gap analysis failed. Defaulting to answer with available info."
            }


# ─── Multi-Hop Retrieval Engine ─────────────────────────────────────────────

class MultiHopRetriever:
    """
    Execute multi-hop retrieval: decompose → execute sub-queries → merge.

    Supports parallel execution of independent sub-queries and sequential
    execution for dependent ones. After each hop, analyzes gaps to decide
    whether to continue.
    """

    def __init__(
        self,
        decomposer: QueryDecomposer,
        gap_analyzer: GapAnalyzer,
        search_fn: Callable[[str, int], Awaitable[List[Dict]]],  # (query, top_k) → chunks
        embed_search_fn: Optional[Callable] = None,  # (embedding, top_k, filter) → chunks
        max_hops: int = 6,
    ):
        self.decomposer = decomposer
        self.gap_analyzer = gap_analyzer
        self.search_fn = search_fn
        self.embed_search_fn = embed_search_fn
        self.max_hops = max_hops

    async def retrieve(
        self,
        query: str,
        top_k_per_hop: int = 5,
        verbose: bool = False,
    ) -> MultiHopResult:
        """
        Execute full multi-hop retrieval.

        1. Decompose query into sub-queries
        2. Execute sub-queries respecting dependencies
        3. After each hop, analyze gaps
        4. Stop when all info gathered or max hops reached
        5. Merge all retrieved chunks
        """
        start_time = time.time()
        all_hops: List[HopResult] = []
        all_chunks: List[Dict] = []
        completed_ids: List[int] = []
        total_tokens = 0

        # Step 1: Decompose
        plan = await self.decomposer.decompose(query)
        if verbose:
            print(f"\nDecomposition plan ({len(plan.sub_queries)} sub-queries):")
            for sq in plan.sub_queries:
                deps = f" (depends on: {sq['depends_on']})" if sq["depends_on"] else ""
                print(f"  [{sq['id']}] {sq['query']}{deps}")

        # Step 2: Execute sub-queries respecting dependency graph
        # Build dependency graph
        dependency_map = {sq["id"]: sq["depends_on"] for sq in plan.sub_queries}
        ready_ids = [sq["id"] for sq in plan.sub_queries if not sq["depends_on"]]
        remaining_ids = [sq["id"] for sq in plan.sub_queries if sq["depends_on"]]

        hop_count = 0

        while ready_ids and hop_count < self.max_hops:
            # Execute all ready (non-blocked) sub-queries in parallel
            tasks = []
            for sid in ready_ids:
                sq = next(s for s in plan.sub_queries if s["id"] == sid)
                tasks.append(self._execute_hop(
                    sq["query"], hop_count + 1, top_k_per_hop
                ))
                hop_count += 1

            hop_results = await asyncio.gather(*tasks)
            all_hops.extend(hop_results)

            for hr in hop_results:
                for chunk in hr.retrieved_chunks:
                    chunk["hop"] = hr.step_number
                    chunk["sub_query"] = hr.query_used
                    all_chunks.append(chunk)

            completed_ids.extend(ready_ids)

            # Check if remaining sub-queries are now unblocked
            new_ready = []
            still_remaining = []
            for sid in remaining_ids:
                deps = dependency_map[sid]
                if all(d in completed_ids for d in deps):
                    new_ready.append(sid)
                else:
                    still_remaining.append(sid)

            ready_ids = new_ready
            remaining_ids = still_remaining

            # Gap analysis after each batch of hops
            if remaining_ids or ready_ids:
                gap = await self.gap_analyzer.analyze(
                    query, all_chunks, plan.sub_queries, completed_ids
                )
                if gap.get("can_answer", False) and gap.get("confidence") == "high":
                    if verbose:
                        print(f"\nGap analysis: can answer with high confidence. Stopping.")
                    break

                # If gap analysis suggests a NEW query not in plan
                if gap.get("suggested_query") and not ready_ids and not remaining_ids:
                    if verbose:
                        print(f"\nGap analysis suggests new query: {gap['suggested_query']}")
                    # Add recovered sub-query
                    plan = await self.decomposer.recover_sub_queries(
                        query, plan,
                        f"Missing: {gap['missing_information']}"
                    )
                    # Rebuild execution state
                    ready_ids = [sq["id"] for sq in plan.sub_queries
                                 if not sq["depends_on"]
                                 and sq["id"] not in completed_ids]
                    remaining_ids = [sq["id"] for sq in plan.sub_queries
                                     if sq["depends_on"]
                                     and sq["id"] not in completed_ids]

        total_latency = (time.time() - start_time) * 1000

        return MultiHopResult(
            original_query=query,
            hops=all_hops,
            merged_chunks=all_chunks,
            total_latency_ms=total_latency,
            total_tokens=total_tokens,
        )

    async def _execute_hop(
        self,
        query: str,
        step_number: int,
        top_k: int,
    ) -> HopResult:
        """Execute a single retrieval hop."""
        hop_start = time.time()

        # Perform search
        chunks = await self.search_fn(query, top_k)

        latency = (time.time() - hop_start) * 1000

        return HopResult(
            step_number=step_number,
            query_used=query,
            hop_type=HopType.SEMANTIC,
            retrieved_chunks=chunks,
            answer_so_far=None,
            gaps_identified=[],
            latency_ms=latency,
        )

    async def retrieve_standard_baseline(
        self, query: str, top_k: int = 10
    ) -> MultiHopResult:
        """Standard single-query retrieval for baseline comparison."""
        start = time.time()
        chunks = await self.search_fn(query, top_k)
        latency = (time.time() - start) * 1000

        return MultiHopResult(
            original_query=query,
            hops=[HopResult(
                step_number=0,
                query_used=query,
                hop_type=HopType.SEMANTIC,
                retrieved_chunks=chunks,
                answer_so_far=None,
                gaps_identified=[],
                latency_ms=latency,
            )],
            merged_chunks=chunks,
            total_latency_ms=latency,
            total_tokens=0,
        )


# ─── Usage Example ──────────────────────────────────────────────────────────

async def multi_hop_demo():
    """Demonstrate multi-hop retrieval on a real scenario."""
    client = AsyncOpenAI()

    # Mock document store
    docs = {
        "alpha": {"id": "doc_1", "text": "Project Alpha was led by Dr. Sarah Chen, who joined Acme Corp in 2018.", "source": "org_chart"},
        "sarah": {"id": "doc_2", "text": "Dr. Sarah Chen is the Director of AI Research, reporting to the CTO.", "source": "org_chart"},
        "ai_research": {"id": "doc_3", "text": "The AI Research team published 15 papers in 2024 focusing on NLP and computer vision.", "source": "annual_report"},
        "papers": {"id": "doc_4", "text": "2024 AI Research papers include: 'Efficient Transformer Architectures', 'Multimodal Learning for Robotics', and 'Low-Resource NLP Techniques'.", "source": "publications"},
        "alpha_status": {"id": "doc_5", "text": "Project Alpha is currently in phase 2, focused on productionizing the NLP research findings.", "source": "project_status"},
    }

    async def mock_search(query: str, top_k: int = 5) -> List[Dict]:
        """Simulate semantic search."""
        # In production: real embedding search
        # For demo: match on keyword overlap
        query_lower = query.lower()
        scored = []
        for key, doc in docs.items():
            text_lower = doc["text"].lower()
            # Count overlapping words
            query_words = set(query_lower.split())
            doc_words = set(text_lower.split())
            overlap = len(query_words & doc_words)
            scored.append((overlap, doc))

        scored.sort(key=lambda x: x[0], reverse=True)
        return [doc for score, doc in scored[:top_k] if score > 0]

    # Build system
    decomposer = QueryDecomposer(llm_client=client)
    gap_analyzer = GapAnalyzer(llm_client=client)
    retriever = MultiHopRetriever(decomposer, gap_analyzer, mock_search)

    # Test query
    query = "What research areas does Project Alpha's leader focus on?"
    print(f"\n{'='*60}")
    print(f"QUERY: {query}")
    print(f"{'='*60}\n")

    # Standard baseline
    print(">>> STANDARD RETRIEVAL (baseline) <<<")
    baseline = await retriever.retrieve_standard_baseline(query, top_k=5)
    print(f"Retrieved {len(baseline.merged_chunks)} chunks in {baseline.total_latency_ms:.0f}ms:")
    for c in baseline.merged_chunks:
        print(f"  [{c['id']}] {c['text'][:100]}...")

    print(f"\n>>> MULTI-HOP RETRIEVAL <<<")
    result = await retriever.retrieve(query, verbose=True)
    print(f"\nTotal: {len(result.hops)} hops, {len(result.merged_chunks)} chunks, {result.total_latency_ms:.0f}ms")

    for hop in result.hops:
        print(f"\n  Hop {hop.step_number}: '{hop.query_used}' ({hop.latency_ms:.0f}ms)")
        for c in hop.retrieved_chunks:
            print(f"    • [{c['id']}] {c['text'][:100]}")

    assert len(result.hops) >= 2, "Multi-hop should require at least 2 hops for this question"
    print("\n✅ Multi-hop succeeded — found information across multiple documents!")
```

**Key design decisions:**
- Parallel execution of independent sub-queries reduces total latency
- Gap analysis provides a dynamic stopping condition
- Recovery planning handles decomposition that fails in practice
- Baseline comparison lets you measure the value of multi-hop

---

### 3. Corrective RAG (CRAG)

```python
"""
crag.py — Corrective RAG with relevance evaluation and dynamic routing.

Instead of blindly sending retrieved chunks to the LLM, CRAG evaluates
their relevance first and decides whether to:
1. Generate from them (if relevant)
2. Refine the query and retry (if uncertain)
3. Use a fallback (if consistently irrelevant)
"""

from typing import List, Optional, Dict, Any, Tuple, Literal
from dataclasses import dataclass, field
from enum import Enum
import json
import asyncio
import time

from openai import AsyncOpenAI


# ─── Data Models ────────────────────────────────────────────────────────────

class RelevanceLevel(Enum):
    """How relevant a chunk is to answering a query."""
    RELEVANT = "relevant"
    PARTIALLY_RELEVANT = "partially_relevant"
    IRRELEVANT = "irrelevant"


@dataclass
class ChunkEvaluation:
    """Evaluation of a single chunk's relevance."""
    chunk_id: str
    chunk_text: str
    relevance: RelevanceLevel
    confidence: float  # 0.0 to 1.0
    reasoning: str


@dataclass
class RetrievalEvaluation:
    """Evaluation of the entire retrieval result."""
    query: str
    chunk_evaluations: List[ChunkEvaluation]
    overall_assessment: Literal["good", "mixed", "poor"]
    action_needed: Literal["generate", "refine", "fallback"]
    refinement_suggestion: Optional[str] = None


@dataclass
class CRAGResult:
    """Complete CRAG pipeline result."""
    query: str
    retrieval_results: List[Dict[str, Any]]
    evaluations: RetrievalEvaluation
    rewritten_query: Optional[str]
    second_retrieval_results: Optional[List[Dict[str, Any]]]
    final_chunks: List[Dict[str, Any]]
    final_answer: Optional[str]
    total_latency_ms: float
    actions_taken: List[str]


# ─── Relevance Evaluator ────────────────────────────────────────────────────

class RelevanceEvaluator:
    """
    Evaluate whether retrieved chunks are relevant to answering a query.

    This is the core of CRAG — if retrieval quality is poor, CRAG catches
    it BEFORE the generation step and takes corrective action.
    """

    EVALUATION_PROMPT = """You are evaluating whether a retrieved document chunk is RELEVANT
to answering a user's question.

A chunk is RELEVANT if it contains information that DIRECTLY helps answer the question.
A chunk is PARTIALLY RELEVANT if it contains RELATED information but doesn't directly answer.
A chunk is IRRELEVANT if it has nothing to do with the question.

Be STRICT. A chunk about "login timeout settings" is NOT relevant to "how do I reset my password."
A chunk about "general pricing" is NOT relevant to "enterprise tier rate limits."

For each chunk, determine:
1. Is it relevant, partially relevant, or irrelevant?
2. How confident are you? (0.0-1.0)
3. WHY did you make this judgment?

Return JSON:
{{
    "chunks": [
        {{
            "chunk_id": "id",
            "relevance": "relevant" | "partially_relevant" | "irrelevant",
            "confidence": 0.95,
            "reasoning": "This chunk contains direct information about X..."
        }}
    ],
    "overall_assessment": "good" | "mixed" | "poor",
    "action_needed": "generate" | "refine" | "fallback",
    "refinement_suggestion": "Suggestion for how to refine the query if needed"
}}

Assessment guidelines:
- GOOD: Most chunks are relevant. Generate the answer.
- MIXED: Mix of relevant and irrelevant chunks. Consider refining the query.
- POOR: Most chunks are irrelevant. Definitely refine the query or use fallback.
"""

    def __init__(self, llm_client: AsyncOpenAI, model: str = "gpt-4o-mini"):
        self.llm_client = llm_client
        self.model = model

    async def evaluate(
        self,
        query: str,
        chunks: List[Dict[str, Any]],
    ) -> RetrievalEvaluation:
        """
        Evaluate all retrieved chunks for relevance.
        """
        # Truncate chunk text to avoid token overflow
        truncated_chunks = []
        for c in chunks:
            text = c.get("text", c.get("content", str(c)))[:1000]
            truncated_chunks.append({"id": c.get("id", "unknown"), "text": text})

        prompt = f"""Question: {query}

Retrieved chunks:
{json.dumps(truncated_chunks, indent=2)}

Evaluate each chunk's relevance to answering the question."""

        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": self.EVALUATION_PROMPT},
                {"role": "user", "content": prompt}
            ],
            response_format={"type": "json_object"},
            temperature=0.0,
            max_tokens=2000,
        )

        try:
            data = json.loads(response.choices[0].message.content)
        except json.JSONDecodeError:
            # Fallback: assume all relevant
            data = {
                "chunks": [
                    {"chunk_id": c.get("id", "unknown"), "relevance": "relevant",
                     "confidence": 0.5, "reasoning": "Evaluation failed, assuming relevant"}
                    for c in chunks
                ],
                "overall_assessment": "mixed",
                "action_needed": "generate",
                "refinement_suggestion": None
            }

        chunk_evals = []
        for ce in data.get("chunks", []):
            try:
                level = RelevanceLevel(ce["relevance"])
            except ValueError:
                level = RelevanceLevel.PARTIALLY_RELEVANT
            chunk_evals.append(ChunkEvaluation(
                chunk_id=ce.get("chunk_id", "unknown"),
                chunk_text="",
                relevance=level,
                confidence=ce.get("confidence", 0.5),
                reasoning=ce.get("reasoning", ""),
            ))

        return RetrievalEvaluation(
            query=query,
            chunk_evaluations=chunk_evals,
            overall_assessment=data.get("overall_assessment", "mixed"),
            action_needed=data.get("action_needed", "generate"),
            refinement_suggestion=data.get("refinement_suggestion"),
        )

    async def evaluate_fast(
        self,
        query: str,
        chunks: List[Dict[str, Any]],
    ) -> RetrievalEvaluation:
        """
        Fast evaluation using keyword heuristics (no LLM call).

        This is an alternative to the LLM-based evaluator that trades
        accuracy for speed. Use when latency is critical.
        """
        chunk_evals = []
        query_terms = set(query.lower().split())

        for c in chunks:
            text = c.get("text", c.get("content", str(c))).lower()
            text_terms = set(text.split())

            # Jaccard similarity as a relevance proxy
            overlap = len(query_terms & text_terms)
            union = len(query_terms | text_terms)
            jaccard = overlap / union if union > 0 else 0

            if jaccard > 0.3:
                level = RelevanceLevel.RELEVANT
            elif jaccard > 0.1:
                level = RelevanceLevel.PARTIALLY_RELEVANT
            else:
                level = RelevanceLevel.IRRELEVANT

            chunk_evals.append(ChunkEvaluation(
                chunk_id=c.get("id", "unknown"),
                chunk_text="",
                relevance=level,
                confidence=jaccard,
                reasoning=f"Keyword overlap: {jaccard:.2f}",
            ))

        # Overall assessment
        relevant_count = sum(1 for e in chunk_evals if e.relevance == RelevanceLevel.RELEVANT)
        total = len(chunk_evals)

        if relevant_count / total >= 0.6:
            assessment = "good"
            action = "generate"
        elif relevant_count / total >= 0.3:
            assessment = "mixed"
            action = "refine"
        else:
            assessment = "poor"
            action = "fallback"

        return RetrievalEvaluation(
            query=query,
            chunk_evaluations=chunk_evals,
            overall_assessment=assessment,
            action_needed=action,
        )


# ─── Query Refiner ──────────────────────────────────────────────────────────

class QueryRefiner:
    """
    Refine a search query to get better retrieval results.

    Used by CRAG when the initial retrieval is poor.
    """

    REFINEMENT_PROMPT = """The following search query returned poor results.
Please rewrite it to be more specific and searchable.

Original query: {original_query}

Evaluation feedback: {feedback}

Rules for refinement:
- Use more precise, domain-specific vocabulary
- Add distinguishing keywords to narrow the search
- Remove ambiguous or vague terms
- Keep it concise (10-20 words)
- Format as a search query, not a question

Return ONLY the new query text, nothing else.
"""

    def __init__(self, llm_client: AsyncOpenAI, model: str = "gpt-4o-mini"):
        self.llm_client = llm_client
        self.model = model

    async def refine(
        self,
        original_query: str,
        evaluation: RetrievalEvaluation,
        max_retries: int = 2,
    ) -> Tuple[str, int]:
        """
        Refine a query based on evaluation feedback.

        Returns: (refined_query, number_of_attempts)
        """
        feedback = "\n".join(
            f"  Chunk {e.chunk_id}: {e.relevance.value} (confidence: {e.confidence:.2f})"
            for e in evaluation.chunk_evaluations[:5]
        )

        if evaluation.refinement_suggestion:
            feedback += f"\nSuggestion: {evaluation.refinement_suggestion}"

        for attempt in range(max_retries):
            response = await self.llm_client.chat.completions.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": self.REFINEMENT_PROMPT.format(
                        original_query=original_query, feedback=feedback
                    )}
                ],
                temperature=0.3,
                max_tokens=100,
            )

            refined = response.choices[0].message.content.strip().strip('"\'')
            if len(refined) > 5 and refined != original_query:
                return refined, attempt + 1

        # Fallback: append "detailed specification" to original
        return f"{original_query} detailed specification", max_retries


# ─── CRAG Pipeline ──────────────────────────────────────────────────────────

class CRAGPipeline:
    """
    Complete Corrective RAG pipeline.

    Retrieves → Evaluates → (Generates | Refines & Retries | Falls Back)

    This is the production-ready implementation with:
    - Multi-level evaluation (fast heuristic + deep LLM)
    - Configurable thresholds for decision-making
    - Refinement with limited retries
    - Graceful fallback when all retrieval attempts fail
    """

    def __init__(
        self,
        llm_client: AsyncOpenAI,
        relevance_evaluator: RelevanceEvaluator,
        query_refiner: QueryRefiner,
        search_fn: callable,
        generate_fn: callable,
        model: str = "gpt-4o-mini",
        good_threshold: float = 0.6,   # Fraction of relevant chunks to call it "good"
        poor_threshold: float = 0.3,   # Below this → "poor"
        max_retries: int = 2,
    ):
        self.llm_client = llm_client
        self.evaluator = relevance_evaluator
        self.refiner = query_refiner
        self.search_fn = search_fn
        self.generate_fn = generate_fn
        self.model = model
        self.good_threshold = good_threshold
        self.poor_threshold = poor_threshold
        self.max_retries = max_retries

    async def run(
        self,
        query: str,
        top_k: int = 10,
        fast_eval_first: bool = True,
    ) -> CRAGResult:
        """
        Run the full CRAG pipeline.

        Args:
            query: User's question
            top_k: Number of chunks to retrieve initially
            fast_eval_first: Use fast heuristic eval before LLM eval

        Returns:
            CRAGResult with full pipeline trace
        """
        start_time = time.time()
        actions: List[str] = []
        rewritten_query: Optional[str] = None
        second_results: Optional[List[Dict]] = None

        # Step 1: Initial retrieval
        chunks = await self.search_fn(query, top_k)
        actions.append(f"retrieved {len(chunks)} chunks")

        # Step 2: Evaluate relevance
        if fast_eval_first:
            # Fast heuristic eval first
            fast_eval = await self.evaluator.evaluate_fast(query, chunks)

            if fast_eval.action_needed == "generate":
                # Fast eval says it's good — use as-is
                evaluation = fast_eval
                actions.append("fast_eval: good, proceeding to generate")
            else:
                # Fast eval flagged issues — do deep LLM eval
                evaluation = await self.evaluator.evaluate(query, chunks)
                actions.append(f"deep_eval: {evaluation.overall_assessment}")
        else:
            evaluation = await self.evaluator.evaluate(query, chunks)
            actions.append(f"deep_eval: {evaluation.overall_assessment}")

        # Step 3: Decide action
        if evaluation.action_needed == "generate":
            # Good retrieval — generate directly
            final_chunks = await self._filter_relevant_chunks(chunks, evaluation)
            final_answer = await self.generate_fn(query, final_chunks)
            actions.append(f"generated from {len(final_chunks)} relevant chunks")

        elif evaluation.action_needed == "refine":
            # Refine the query and retry
            rewritten_query, attempts = await self.refiner.refine(query, evaluation)

            second_results = await self.search_fn(rewritten_query, top_k)
            actions.append(f"refined query, retrieved {len(second_results)} chunks (attempt {attempts})")

            # Evaluate second retrieval
            second_eval = await self.evaluator.evaluate(query, second_results)

            if second_eval.action_needed == "generate":
                final_chunks = await self._filter_relevant_chunks(second_results, second_eval)
            else:
                # Second attempt also failed — merge first and second, keep what's relevant
                all_chunks = chunks + second_results
                merged_eval = await self.evaluator.evaluate(query, all_chunks)
                final_chunks = await self._filter_relevant_chunks(all_chunks, merged_eval)
                actions.append("merged first+second retrieval, kept relevant chunks")

            final_answer = await self.generate_fn(query, final_chunks)
            actions.append(f"generated from {len(final_chunks)} chunks after refinement")

        else:  # fallback
            # Both evaluators agree it's poor — use fallback
            final_chunks = []
            final_answer = await self._fallback_generation(query)
            actions.append("fallback: generated from model knowledge (no retrieval)")

        total_latency = (time.time() - start_time) * 1000

        return CRAGResult(
            query=query,
            retrieval_results=chunks,
            evaluations=evaluation,
            rewritten_query=rewritten_query,
            second_retrieval_results=second_results,
            final_chunks=final_chunks,
            final_answer=final_answer,
            total_latency_ms=total_latency,
            actions_taken=actions,
        )

    async def _filter_relevant_chunks(
        self,
        chunks: List[Dict],
        evaluation: RetrievalEvaluation,
    ) -> List[Dict]:
        """Filter chunks to only those deemed relevant."""
        eval_map = {e.chunk_id: e for e in evaluation.chunk_evaluations}

        # Build chunk->eval lookup
        relevant = []
        for c in chunks:
            cid = c.get("id", "unknown")
            eval_result = eval_map.get(cid)

            if eval_result and eval_result.relevance in (
                RelevanceLevel.RELEVANT,
                RelevanceLevel.PARTIALLY_RELEVANT
            ):
                relevant.append(c)

        # If nothing is relevant, keep top-2 partially relevant
        if not relevant:
            for c in chunks[:2]:
                relevant.append(c)

        return relevant

    async def _fallback_generation(self, query: str) -> str:
        """Generate an answer from model knowledge alone."""
        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": (
                    "You are answering a question WITHOUT access to relevant documents. "
                    "The retrieval system was unable to find relevant information. "
                    "Answer based on your training data, but clearly note that you "
                    "could not find specific documentation to support this answer."
                )},
                {"role": "user", "content": query}
            ],
            temperature=0.3,
        )
        return response.choices[0].message.content


# ─── Usage Example ──────────────────────────────────────────────────────────

async def crag_demo():
    """Demonstrate CRAG pipeline with good and bad retrieval scenarios."""
    client = AsyncOpenAI()

    # Mock documents
    docs_db = [
        {"id": "d1", "text": "Enterprise API rate limit: 100,000 requests per hour. Contact support for custom limits."},
        {"id": "d2", "text": "Enterprise tier SLA guarantees 99.99% uptime with 15-minute response time for critical issues."},
        {"id": "d3", "text": "Enterprise pricing starts at $999/month for up to 5 users. Additional users are $199/user/month."},
        {"id": "d4", "text": "Free tier limits: 1,000 requests/hour, 1 concurrent connection, community support only."},
        {"id": "d5", "text": "Pro tier: 10,000 requests/hour, 10 concurrent connections, email support."},
    ]

    async def mock_search(query: str, top_k: int = 5) -> List[Dict]:
        """Mock search that sometimes returns wrong results."""
        query_lower = query.lower()
        scored = []
        for doc in docs_db:
            text = doc["text"].lower()
            query_words = set(query_lower.split())
            doc_words = set(text.split())
            overlap = len(query_words & doc_words)
            scored.append((overlap, doc))

        # Randomize some results to simulate noisy retrieval
        import random
        random.shuffle(scored)

        scored.sort(key=lambda x: x[0], reverse=True)

        # Take top_k, ensuring at least some results
        results = [doc for _, doc in scored[:top_k]]
        return results if results else docs_db[:2]

    async def mock_generate(query: str, chunks: List[Dict]) -> str:
        """Mock generation."""
        if not chunks:
            return "I couldn't find relevant documents for this question."

        context = "\n\n".join(c["text"] for c in chunks)
        return f"Based on {len(chunks)} relevant documents:\n\n{context[:200]}"

    # Build CRAG pipeline
    evaluator = RelevanceEvaluator(llm_client=client)
    refiner = QueryRefiner(llm_client=client)
    crag = CRAGPipeline(
        llm_client=client,
        relevance_evaluator=evaluator,
        query_refiner=refiner,
        search_fn=mock_search,
        generate_fn=mock_generate,
    )

    # Test with a query that has good document coverage
    query = "What is the enterprise API rate limit?"
    print(f"\n{'='*60}")
    print(f"QUERY: {query}")
    print(f"{'='*60}")

    result = await crag.run(query)

    print(f"Actions: {' → '.join(result.actions_taken)}")
    print(f"Rewritten query: {result.rewritten_query}")
    print(f"Final chunks: {len(result.final_chunks)}")
    print(f"Total latency: {result.total_latency_ms:.0f}ms")

    # Test with a query that has NO good document coverage
    bad_query = "What's the weather forecast for Tokyo next week?"
    print(f"\n{'='*60}")
    print(f"QUERY: {bad_query} (no matching docs)")
    print(f"{'='*60}")

    bad_result = await crag.run(bad_query)

    print(f"Actions: {' → '.join(bad_result.actions_taken)}")
    print(f"Rewritten query: {bad_result.rewritten_query}")
    print(f"Final chunks: {len(bad_result.final_chunks)}")
    print(f"Is fallback: {bad_result.rewritten_query is not None}")
```

---

### 4. LangGraph Agentic RAG

For the full agent loop, use LangGraph to build a state machine that reasons, retrieves, evaluates, and generates:

```python
"""
langgraph_rag.py — Full Agentic RAG with LangGraph state machine.

When standard retrieval, self-query, multi-hop, and CRAG aren't enough,
this agent loop gives the system full agency to decide WHAT to retrieve,
WHEN to retrieve it, and whether the answer is complete.

Requires: pip install langgraph langchain-openai
"""

from typing import List, Optional, Dict, Any, Annotated, Literal, TypedDict
from dataclasses import dataclass, field
import json
import operator

# LangGraph imports
from langgraph.graph import StateGraph, END
from langgraph.checkpoint import MemorySaver
from langchain_openai import ChatOpenAI

# ─── State Definition ───────────────────────────────────────────────────────

@dataclass
class RAGAgentState:
    """
    State of the agentic RAG loop.

    This is mutated as the agent reasons, retrieves, and generates.
    """
    query: str
    messages: List[Dict] = field(default_factory=list)  # Full conversation
    retrieved_chunks: List[Dict] = field(default_factory=list)  # All chunks found
    current_plan: Optional[Dict] = None  # Current retrieval plan
    gaps: List[str] = field(default_factory=list)  # Identified gaps
    search_history: List[Dict] = field(default_factory=list)  # What was searched and found
    final_answer: Optional[str] = None
    max_retrievals: int = 5
    retrieval_count: int = 0
    iteration_count: int = 0
    max_iterations: int = 10


# ─── Agent Nodes ────────────────────────────────────────────────────────────

class RAGAgentNodes:
    """
    Each node in the LangGraph state machine.

    The graph structure:
    [PLAN] → [RETRIEVE] → [EVALUATE] → [GENERATE or BACK TO PLAN]
    """

    def __init__(
        self,
        llm: ChatOpenAI,
        search_fn: callable,
        embed_fn: Optional[callable] = None,
    ):
        self.llm = llm
        self.search_fn = search_fn
        self.embed_fn = embed_fn

    async def plan_node(self, state: RAGAgentState) -> RAGAgentState:
        """
        PLAN: Given the query and what's been retrieved so far,
        decide what to retrieve next.
        """
        context = ""
        if state.retrieved_chunks:
            context = "Information retrieved so far:\n" + "\n".join(
                f"[{c.get('id', i)}] {c.get('text', str(c))[:300]}"
                for i, c in enumerate(state.retrieved_chunks[-5:])
            )

        gaps = ""
        if state.gaps:
            gaps = "Identified gaps:\n" + "\n".join(f"  • {g}" for g in state.gaps)

        plan_prompt = f"""Original question: {state.query}

{context}

{gaps}

Previous searches conducted: {len(state.search_history)}
Retrievals remaining: {state.max_retrievals - state.retrieval_count}

Decide what to do next:
1. RETRIEVE: Search for more information with a specific query
2. GENERATE: You have enough information to answer

If RETRIEVE, provide:
- search_query: The specific query to search for
- reasoning: Why this search is needed

If GENERATE, provide:
- answer: Your complete answer to the user's question
- citations: List of chunk IDs used

Return JSON:
{{
    "action": "retrieve" | "generate",
    "search_query": "query text (if retrieve)",
    "reasoning": "why this action"
}}
"""
        response = await self.llm.ainvoke([
            {"role": "system", "content": "You are a RAG agent deciding what to do next."},
            {"role": "user", "content": plan_prompt}
        ])

        try:
            decision = json.loads(response.content)
        except (json.JSONDecodeError, AttributeError):
            decision = {"action": "generate", "reasoning": "Plan parsing failed, generating answer."}

        state.current_plan = decision
        state.iteration_count += 1

        return state

    async def retrieve_node(self, state: RAGAgentState) -> RAGAgentState:
        """
        RETRIEVE: Execute a search based on the current plan.
        """
        query = state.current_plan.get("search_query", state.query)

        chunks = await self.search_fn(query, top_k=5)

        # Tag chunks with metadata
        for c in chunks:
            c["retrieved_by"] = query
            c["retrieval_number"] = state.retrieval_count + 1

        state.retrieved_chunks.extend(chunks)
        state.search_history.append({
            "query": query,
            "chunk_count": len(chunks),
            "chunk_ids": [c.get("id", "?") for c in chunks],
        })
        state.retrieval_count += 1

        return state

    async def evaluate_node(self, state: RAGAgentState) -> RAGAgentState:
        """
        EVALUATE: Check if retrieved information fills the gaps.
        Identify what's still missing.
        """
        context = "\n\n".join(
            f"[{c.get('id', i)}] {c.get('text', str(c))[:500]}"
            for i, c in enumerate(state.retrieved_chunks[-10:])
        )

        eval_prompt = f"""Original question: {state.query}

All information retrieved:
{context}

Evaluate:
1. Can you answer the question with this information?
2. If NOT, what SPECIFICALLY is missing?
3. What query would find the missing information?

Be specific about gaps. "More information" is not specific.
"Details about the migration window" is specific.

Return JSON:
{{
    "can_answer": true/false,
    "gaps": ["Specific gap 1", "Specific gap 2"],
    "suggested_search": "Specific search to fill gaps"
}}
"""
        response = await self.llm.ainvoke([
            {"role": "system", "content": "You are evaluating whether retrieved documents are sufficient."},
            {"role": "user", "content": eval_prompt}
        ])

        try:
            eval_result = json.loads(response.content)
        except (json.JSONDecodeError, AttributeError):
            eval_result = {
                "can_answer": True,
                "gaps": [],
                "suggested_search": None,
            }

        state.gaps = eval_result.get("gaps", [])

        return state

    def should_continue(self, state: RAGAgentState) -> Literal["retrieve", "generate", "stop"]:
        """
        CONDITIONAL EDGE: Decide whether to retrieve more or generate the answer.

        This is called after EVALUATE to route to the next node.
        """
        # Hard stop conditions
        if state.iteration_count >= state.max_iterations:
            return "generate"  # Force generate to prevent infinite loop

        if state.retrieval_count >= state.max_retrievals:
            return "generate"  # Used up all retrievals

        plan_action = state.current_plan.get("action") if state.current_plan else None

        if plan_action == "retrieve" and state.gaps:
            return "retrieve"

        if plan_action == "generate" or (plan_action == "retrieve" and not state.gaps):
            return "generate"

        # If gaps identified and we have capacity, retrieve more
        if state.gaps and state.retrieval_count < state.max_retrievals:
            return "retrieve"

        return "generate"

    async def generate_node(self, state: RAGAgentState) -> RAGAgentState:
        """
        GENERATE: Produce the final answer with citations.
        """
        context = "\n\n".join(
            f"[Source: {c.get('id', i)}] {c.get('text', str(c))}"
            for i, c in enumerate(state.retrieved_chunks)
        )

        generate_prompt = f"""Answer the user's question based on the retrieved documents.

Question: {state.query}

Retrieved documents:
{context}

Instructions:
- Only use information from the retrieved documents
- If the documents don't contain enough information, say so
- Cite specific documents using [Source: id]
- If documents contradict each other, explain the contradiction

Answer:"""

        response = await self.llm.ainvoke([
            {"role": "system", "content": "You are a RAG system answering based on retrieved documents."},
            {"role": "user", "content": generate_prompt}
        ])

        state.final_answer = response.content

        return state


# ─── Graph Construction ────────────────────────────────────────────────────

def build_rag_agent(llm, search_fn, embed_fn=None):
    """
    Build the LangGraph state machine for agentic RAG.

    Graph:
        START → PLAN → (retrieve? → RETRIEVE → EVALUATE → back to PLAN)
                    → (generate? → GENERATE → END)
    """
    nodes = RAGAgentNodes(llm, search_fn, embed_fn)

    # Build graph
    builder = StateGraph(RAGAgentState)

    # Add nodes
    builder.add_node("plan", nodes.plan_node)
    builder.add_node("retrieve", nodes.retrieve_node)
    builder.add_node("evaluate", nodes.evaluate_node)
    builder.add_node("generate", nodes.generate_node)

    # Set entry point
    builder.set_entry_point("plan")

    # Add conditional edges
    builder.add_conditional_edges(
        "plan",
        lambda s: s.current_plan.get("action", "generate") if s.current_plan else "generate",
        {
            "retrieve": "retrieve",
            "generate": "generate",
        }
    )

    builder.add_edge("retrieve", "evaluate")
    builder.add_conditional_edges(
        "evaluate",
        nodes.should_continue,
        {
            "retrieve": "plan",
            "generate": "generate",
            "stop": END,
        }
    )

    builder.add_edge("generate", END)

    # Compile with checkpointing for state persistence
    memory = MemorySaver()
    graph = builder.compile(checkpointer=memory)

    return graph


# ─── Usage Example ──────────────────────────────────────────────────────────

async def agent_demo():
    """Demonstrate the full agentic RAG loop."""
    from langchain_openai import ChatOpenAI

    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

    # Mock documents
    docs = {
        "doc_v3_migration": {"id": "doc_v3", "text": "API v3 migration guide: Webhook signatures now use ED25519 instead of HMAC-SHA256. Migration window: Q3 2025 to Q1 2026."},
        "doc_changelog": {"id": "doc_changelog", "text": "Changelog v3.0.0 (2025-07-15): Breaking change — webhook signature algorithm changed to ED25519. Old HMAC-SHA256 signatures will be rejected after Q1 2026."},
        "doc_adr": {"id": "doc_adr", "text": "ADR-2024-013: Webhook signature algorithm change. Decision: Migrate from HMAC-SHA256 to ED25519. Rationale: Security audit finding #2024-089 identified HMAC-SHA256 as susceptible to hash length extension attacks."},
    }

    async def mock_search_fn(query: str, top_k: int = 5) -> List[Dict]:
        """Mock search that finds relevant docs by keyword overlap."""
        query_lower = query.lower()
        scored = []
        for key, doc in docs.items():
            text = doc["text"].lower()
            words = set(query_lower.split())
            doc_words = set(text.split())
            overlap = len(words & doc_words)
            scored.append((overlap, doc))

        scored.sort(key=lambda x: x[0], reverse=True)
        results = [doc for _, doc in scored[:top_k] if _ > 0]
        return results if results else [{"id": "fallback", "text": f"Generic info about: {query}"}]

    graph = build_rag_agent(llm, mock_search_fn)

    query = "I'm migrating from API v2 to v3. Do I need to change webhook signature verification? What's the migration window?"

    # Initialize state
    initial_state = RAGAgentState(
        query=query,
        max_retrievals=3,
        max_iterations=6,
    )

    # Run the graph
    result = await graph.ainvoke(initial_state)

    print(f"\n{'='*60}")
    print(f"QUERY: {query}")
    print(f"{'='*60}")
    print(f"\nRetrievals: {result.retrieval_count}")
    print(f"Iterations: {result.iteration_count}")
    print(f"Chunks found: {len(result.retrieved_chunks)}")
    print(f"\nSearch history:")
    for i, s in enumerate(result.search_history):
        print(f"  {i+1}. '{s['query']}' → {s['chunk_count']} chunks: {s['chunk_ids']}")

    print(f"\nFINAL ANSWER:\n{result.final_answer}")
```

---

### 5. Agentic RAG Decision Router

A critical production pattern: route queries to the RIGHT level of RAG based on query complexity:

```python
"""
rag_router.py — Intelligent routing across RAG strategies.

Not every query needs multi-hop. Not every query needs agentic RAG.
This router classifies query complexity and routes to the optimal strategy.
"""

from typing import Literal, Dict, Any, Optional
from enum import Enum
import json
import time


class ComplexityLevel(Enum):
    """Query complexity levels for strategy routing."""
    SIMPLE = "simple"             # One chunk can answer it
    COMPOUND = "compound"         # Multiple parallel sub-questions
    SEQUENTIAL = "sequential"     # Multi-hop: answer depends on previous
    COMPLEX = "complex"           # Needs full agent loop


class RAGStrategy(Enum):
    """Available RAG strategies, ordered by complexity."""
    STANDARD = "standard"         # Single retrieval → generate
    SELF_QUERY = "self_query"     # Extract filters → retrieve → generate
    DECOMPOSE = "decompose"       # Decompose → parallel retrieval → merge → generate
    MULTI_HOP = "multi_hop"       # Sequential retrieval with gap analysis
    CRAG = "crag"                 # Corrective RAG with evaluation
    AGENTIC = "agentic"           # Full agent loop


class RAGRouter:
    """
    Classify query complexity and route to the right strategy.

    This is the KEY production pattern — don't use a sledgehammer
    for every query. Route intelligently based on query properties.
    """

    COMPLEXITY_PROMPT = """Classify the complexity of this search/query for a RAG system:

Query: {query}

Determine the complexity level:

1. SIMPLE — One retrieval answers it. "What's the refund policy?"
   - Single fact or piece of information
   - Can be answered by one good chunk
   - No multi-document reasoning needed

2. COMPOUND — Multiple independent sub-questions.
   "Compare the mobile and web onboarding flows."
   - Multiple parts, but each part is independently searchable
   - Results can be merged without sequential dependencies
   - Parallel retrieval works

3. SEQUENTIAL — Multi-hop required.
   "Who wrote the paper about X, and what else did they write?"
   - Answer to one part determines what to search for next
   - Sequential dependencies between sub-questions
   - Cannot answer in a single retrieval

4. COMPLEX — Needs full reasoning + retrieval loop.
   "I'm migrating from v2 to v3, do I need to change my webhook setup?"
   - Requires exploration: don't know what you'll find
   - Multiple possible paths depending on intermediate results
   - Needs evaluation of retrieved information

Return JSON:
{{
    "complexity": "simple" | "compound" | "sequential" | "complex",
    "reasoning": "Why this classification",
    "recommended_strategy": "standard" | "self_query" | "decompose" | "multi_hop" | "crag" | "agentic",
    "sub_queries": ["query 1", "query 2"] or null
}}
"""

    def __init__(self, llm_client, model: str = "gpt-4o-mini"):
        self.llm_client = llm_client
        self.model = model

    async def classify(self, query: str) -> Dict[str, Any]:
        """Classify query and recommend strategy."""
        prompt = self.COMPLEXITY_PROMPT.replace("{query}", query)

        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "You are a RAG query complexity classifier."},
                {"role": "user", "content": prompt}
            ],
            response_format={"type": "json_object"},
            temperature=0.0,
        )

        try:
            return json.loads(response.choices[0].message.content)
        except json.JSONDecodeError:
            return {
                "complexity": "simple",
                "reasoning": "Classification failed, defaulting to simple.",
                "recommended_strategy": "standard",
                "sub_queries": None,
            }

    def strategy_to_use(self, classification: Dict) -> RAGStrategy:
        """Map classification to actual strategy."""
        strategy_map = {
            "standard": RAGStrategy.STANDARD,
            "self_query": RAGStrategy.SELF_QUERY,
            "decompose": RAGStrategy.DECOMPOSE,
            "multi_hop": RAGStrategy.MULTI_HOP,
            "crag": RAGStrategy.CRAG,
            "agentic": RAGStrategy.AGENTIC,
        }
        recommended = classification.get("recommended_strategy", "standard")
        return strategy_map.get(recommended, RAGStrategy.STANDARD)


# ─── Tie It All Together ────────────────────────────────────────────────────

class TieredRAGEngine:
    """
    Complete RAG engine that routes queries to the optimal strategy.

    This is the PRODUCTION PATTERN you should use:
    - 80% of queries → standard or self-query (fast, cheap)
    - 15% of queries → multi-hop or CRAG (moderate, more capable)
    - 5% of queries → agentic loop (slow, expensive, comprehensive)
    """

    def __init__(self, router: RAGRouter, strategies: Dict[RAGStrategy, Any]):
        self.router = router
        self.strategies = strategies

    async def answer(self, query: str) -> Dict:
        """
        Route a query to the right strategy and return the answer.

        Includes cost/latency tracking for each strategy.
        """
        start = time.time()

        # Step 1: Classify
        classification = await self.router.classify(query)
        strategy = self.router.strategy_to_use(classification)

        # Step 2: Route to strategy
        strategy_handler = self.strategies.get(strategy)
        if not strategy_handler:
            # Fallback to standard
            strategy = RAGStrategy.STANDARD
            strategy_handler = self.strategies.get(strategy)

        result = await strategy_handler(query)

        latency = (time.time() - start) * 1000

        return {
            "query": query,
            "strategy_used": strategy.value,
            "complexity": classification.get("complexity"),
            "reasoning": classification.get("reasoning"),
            "latency_ms": latency,
            "result": result,
        }
```

---

## ✅ Good Output Examples

### Self-Query: Before and After

```
BEFORE — Self-Query Extraction:
Query: "What's the rate limit on the free plan?"
Without self-query:
  Retrieved: ["Pro tier: 10,000 req/hr" (0.92 sim), "Free tier: 1,000 req/hr" (0.91 sim)]
  → LLM sees conflicting info, might pick the wrong one
  
With self-query:
  Extracted: {query: "API rate limit", filter: {tier: "free"}}
  Retrieved: ["Free tier: 1,000 req/hr"] (only free tier)
  → LLM sees exactly the right info
```

### Multi-Hop: Full Trace

```
Query: "What research areas does Project Alpha's leader focus on?"

Hop 1: "Project Alpha leadership and team structure"
  → "Project Alpha was led by Dr. Sarah Chen"
  
Hop 2: "Dr. Sarah Chen research focus"
  → "Dr. Sarah Chen is Director of AI Research"
  
Hop 3: "AI Research team focus areas 2024"
  → "NLP and computer vision"

Gap analysis: "can_answer: true, confidence: high"
→ Generate answer with citations to all 3 hops
```

### CRAG: Catching Bad Retrieval

```
Query: "How do I reset my password?"
Initial retrieval: ["Login timeout settings", "Account lockout policy", "Session management"]
  → Relevance evaluator: ALL 3 are IRRELEVANT
  → Refine query: "Password reset procedure steps"
  → Second retrieval: ["Password reset from settings page", "Forgot password flow"]
  → Relevance evaluator: 2/2 RELEVANT
  → Generate with refined results
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Infinite Retrieval Loops

```
Query: "Tell me everything about the company"

Agent loop:
  Retrieve → "Company founded in 2010"
  → "Find more" → "Has 500 employees"
  → "Find more" → "Located in San Francisco"
  → "Find more" → ... (infinite)
```

**The fix:** Always have a **hard limit** on iterations AND a **sufficiency check**. The agent should stop when it can answer the question, whether or not there's more information available.

### Antipattern 2: Self-Query Filter Hallucination

```
Query: "What's the pricing?"
Self-query extraction: {query: "pricing", filter: {tier: "enterprise"}}

But the user didn't ask about enterprise pricing! The extractor
hallucinated a filter. Now they only see enterprise pricing
when they wanted all pricing.
```

**The fix:** Only apply filters when confidence is HIGH. Always have a no-filter fallback. The `SelfQueryExtractor` in this module drops low-confidence filters for exactly this reason.

### Antipattern 3: The CRAG Overhead Trap

```
Every query → LLM evaluation → 2x latency
Even for simple queries that always retrieve well
```

**The fix:** Use the **tiered routing** approach. Only run CRAG evaluation for queries where retrieval quality is uncertain. For known-good query types, skip straight to generation.

### Antipattern 4: Multi-Hop with Overlapping Sub-Queries

```
Query: "What's the weather in Tokyo and how does it compare to London?"

Sub-queries:
  1. "Weather in Tokyo today"
  2. "Weather in London today"
  3. "Tokyo vs London weather comparison"
  4. "Climate data for Tokyo"
  5. "Climate data for London"

Sub-queries 3, 4, 5 are REDUNDANT — they overlap with 1 and 2.
```

**The fix:** Limit decomposition to the MINIMUM number of sub-queries needed. Two sub-queries (Tokyo weather, London weather) plus a comparison instruction is sufficient.

### Antipattern 5: Agent Over-Escalation

```
Query: "What time does the cafeteria close?"

Agent decides it needs: 3 retrievals, a relevance check,
gap analysis, and a multi-hop plan for a SIMPLE question.
```

**The fix:** The classifier should route SIMPLE queries to STANDARD RAG. Save the agent loop for queries that genuinely need it.

### Failure Mode: Self-Query Removing Necessary Context

```
Query: "Show me docs from last week about the deployment"
Extracted: {query: "deployment", filter: {date: "2025-01-10"}}

But "last week" is relative! The extractor might hardcode
a date that's already stale by the time the query runs.
```

**Production solution:** Extract RELATIVE dates as relative filters, not absolute dates:

```python
# Instead of:
{"date": "2025-01-10"}  # Absolute — stale tomorrow

# Use:
{"date_relative": "last_7_days"}  # Relative — computed at query time
```

### Failure Mode: Gap Analysis Circularity

The gap analyzer says: "I need more information about X, so search for X."
You retrieve info about X. Now the gap analyzer says: "I also need more context about Y."

This chain can go on indefinitely if the gap analyzer doesn't have a clear stopping criterion.

**Production solution:** Three-stop mechanism:
1. **Hard cap:** Max 5 retrievals total
2. **Diminishing returns:** If the last retrieval added < 20% new information, stop
3. **Confidence threshold:** If answering confidence is "high," stop regardless of remaining gaps

---

## 🧪 Drills & Challenges

### Drill 1: Self-Query for a Multi-Tenant Doc Store (45 min)

You have a document store with documents from 3 companies (Acme, BetaCorp, GammaInc), each with their own policies, APIs, and documentation.

**Task:** Build a self-query system that:
1. Extracts the company name from the query
2. Filters to only that company's documents
3. Falls back to cross-company search if no company is detected
4. Handles queries like "What's Acme's refund policy?" and "What's the refund policy?" differently

**Success criteria:** 5 test queries, all correctly filtered. Benchmark against standard retrieval.

### Drill 2: Build a Multi-Hop System for HR Policies (45 min)

You have 3 documents:
```
Doc 1: "Manager-level employees and above qualify for executive benefits."
Doc 2: "Executive benefits include supplemental life insurance, deferred comp, health screening."
Doc 3: "Senior Manager is classified as manager-level."
```

**Task:** Build a multi-hop retrieval system that answers "As a senior manager, do I get health screening?"

**Expected path:**
1. Find Doc 3 → "senior manager = manager-level"
2. Find Doc 1 → "manager-level qualifies for exec benefits"
3. Find Doc 2 → "exec benefits includes health screening"
4. Answer: YES, with full reasoning chain

**Success criteria:** System outputs the full reasoning chain with citations.

### Drill 3: CRAG with Deliberately Broken Retrieval (45 min)

**Task:** Build a CRAG pipeline where the retriever is intentionally broken:
- First retrieval always returns 3 irrelevant chunks (pricing info for ANY query)
- The relevance evaluator must catch this
- The query refiner must fix the query
- Second retrieval must return the correct info

**Success criteria:** CRAG pipeline achieves 100% correct answers despite the broken initial retriever.

### Drill 4: Implement Diminishing Returns Stopping (30 min)

**Task:** Modify the multi-hop system to track how much NEW information each hop adds. Stop retrieving when:
- A hop adds less than 20% new information (by word overlap with previous chunks)
- Or 5 hops have been executed
- Or gap analyzer says confidence is "high"

**Success criteria:** System stops at the right time — not too early (missing info), not too late (wasted retrievals).

### Drill 5: The Contradiction Detector (45 min)

**Task:** Build a component that takes a list of retrieved chunks and detects contradictions. For each contradiction found:
1. Identify the conflicting claims
2. Identify the source documents
3. Decide which source to trust (based on recency, authority, specificity)
4. Surface the contradiction in the final answer

**Test with:**
- Chunk A: "Max upload: 25MB" (from public docs, dated 2024-01)
- Chunk B: "Enterprise max upload: 100MB" (from enterprise FAQ, dated 2024-06)

**Success criteria:** Output clearly explains the contradiction AND resolves it.

### Drill 6: Build a Tiered RAG Router (60 min)

**Task:** Implement the `TieredRAGEngine` pattern that routes queries:

| Query Type | Route To |
|------------|----------|
| "What's the refund policy?" | Standard RAG |
| "What's the rate limit for enterprise?" | Self-Query RAG |
| "Compare mobile and web onboarding" | Decompose + Parallel |
| "What papers did the author of X write?" | Multi-Hop RAG |
| "I'm migrating v2→v3, any issues?" | Agentic RAG |

**Success criteria:** 10 test queries, each routed to the correct strategy. Measure total latency and cost savings vs. running everything through the agent loop.

### Drill 7: Debug the Infinite Loop (30 min)

**Given this buggy multi-hop system:**

```python
# BUGGY — identifies gaps but never fills them
class BuggyGapAnalyzer:
    async def analyze(self, query, chunks, sub_queries, completed):
        # Always finds new gaps, never says "can_answer"
        return {
            "can_answer": False,
            "gaps": ["Need more context"],
            "suggested_query": query,  # Same query every time!
        }
```

**Task:** Fix the gap analyzer so it:
1. Detects when the same gap is being re-identified
2. Escalates after 3 identical gap reports
3. Falls back to "answer with what you have"

---

## 🚦 Gate Check

Before moving to File 05 (Multimodal RAG), confirm you can:

- [ ] Build a self-query retriever that extracts structured filters from natural language queries
- [ ] Implement query decomposition that breaks complex questions into sub-questions
- [ ] Build a multi-hop retrieval system and trace its execution
- [ ] Implement CRAG with relevance evaluation and query refinement
- [ ] Know when to use STANDARD vs SELF-QUERY vs MULTI-HOP vs AGENTIC RAG
- [ ] Have a strategy for preventing infinite retrieval loops
- [ ] Can detect when retrieved chunks contradict each other
- [ ] Built a tiered routing system that allocates queries to the right strategy
- [ ] Measured the cost-latency-quality tradeoff between standard and agentic RAG

**Sample gate questions:**

1. **You have a multi-tenant RAG system serving 50 companies. A user asks "What's our data retention policy?" Design the self-query extraction and routing.**

2. **A user asks: "Who manages the product team and what's their background?" Walk through the multi-hop retrieval path.**

3. **Your CRAG system is 3x slower than standard RAG. How do you determine whether the quality improvement is worth the latency cost? What metrics matter?**

4. **When would you use the FULL agent loop vs. multi-hop retrieval? Give a concrete example where multi-hop is sufficient, and another where only the full agent loop works.**

---

## 📚 Resources

- **Self-RAG (Asai et al., 2023):** Learning to Retrieve, Generate, and Critique — the foundational paper on agentic retrieval decisions
- **CRAG (Yan et al., 2024):** Corrective Retrieval Augmented Generation — the CRAG paper with relevance evaluation methodology
- **ReAct (Yao et al., 2022):** Synergizing Reasoning and Acting in Language Models — the reasoning+acting pattern that agentic RAG builds on
- **LangGraph Documentation:** State machine-based agent orchestration — https://langchain-ai.github.io/langgraph/
- **Query Decomposition Survey (2024):** Comprehensive survey of query decomposition techniques — arXiv:2401.07752
- **Self-Querying Retriever (LangChain docs):** https://python.langchain.com/docs/modules/data_connection/retrievers/self_query/
- **Production RAG Patterns:** "Building Production RAG" by Chip Huyen — https://huyenchip.com/2024/07/08/rag.html
- **Multi-Hop QA Survey:** "A Survey on Multi-hop Question Answering" — arXiv:2301.02216
- **Gorilla (Patil et al., 2023):** API retrieval with self-query — relevant for tool-using RAG agents
