# 04 — Context Integration: The Art of the Augmented Prompt

## 🎯 Purpose & Goals

You've built a retrieval pipeline that fetches relevant chunks. You have vector search, you have hybrid search, you have reranking. Now comes the most deceptively difficult part: **putting those chunks into a prompt so the LLM actually uses them well.**

This module is about the "Augment" step in RAG — the bridge between retrieval and generation. It's the part most tutorials gloss over with "just prepend the context to the user's question." That naive approach is why most RAG systems in production underperform their demos.

**By the end of this module, you will:**

1. Understand why chunk ordering, instruction placement, and context window management dramatically affect output quality
2. Know the "Lost in the Middle" problem and how to mitigate it with structural prompt engineering
3. Build reusable context templates that handle dynamic token budgets, multi-turn conversations, and citation formatting
4. Master dynamic context compression — deciding what to keep when you have 50 chunks and 8K tokens of space
5. Implement citation injection that actually works (models cite sources correctly and verifiably)

---

### ⏹ STOP. Answer these questions before proceeding.

These are not review questions. They are **discovery probes**. Sit with each one for at least 3 minutes. Write down your thinking before reading further.

**Question 1 — The Memo Problem**

*You're building a RAG system for a legal document review tool. Your retrieval pipeline returns the top 10 chunks, each ~500 tokens. You have room for only 4,000 tokens of context after accounting for the system prompt and user question. You sort chunks by relevance score and feed the top 8 to the LLM. The LLM consistently misses key information that exists in the middle-ranked chunks, even though you know those chunks contain the answer. The model also occasionally contradicts itself — citing chunk [1] for one claim and chunk [7] for a contradictory claim without noticing.*

*What's happening inside the model's attention mechanism? Why does ordering matter to a transformer? If you could only change ONE thing about how you structure the context (not the retrieval, not the model, not the chunking), what would it be and why?*

**Question 2 — The Citation Problem**

*Your RAG system returns answers with citations like "According to our documentation, the API rate limit is 1000 requests per minute [3]." But when you manually check chunk [3], it actually says "The rate limit was raised from 1000 to 5000 requests per minute in the June 2024 update." The answer is outdated because your embedding index has both old and new versions of the document, and sometimes the old version ranks higher for certain queries.*

*How do you design your context integration so the LLM can detect and resolve contradictions between chunks? What prompt structure would make the model notice "this chunk says 1000, that chunk says 5000" rather than just picking one and citing it? Is this a retrieval problem, a context integration problem, or both?*

**Question 3 — The Token Budget Problem**

*You have a RAG pipeline that needs to handle questions ranging from "What's the refund policy?" (answerable from one chunk) to "Compare all our pricing tiers across enterprise, business, and startup plans" (needs 8+ chunks). You have a fixed 8K token context window. For simple questions, you waste tokens on irrelevant chunks. For complex questions, you can't fit all the needed context.*

*Design a context integration strategy that dynamically adapts to question complexity. How do you decide the chunk budget per query? What happens when the budget is exceeded? How do you communicate truncation to the LLM so it knows it's working with partial information?*

**After answering, proceed to the deep-dive. These questions will make much more sense as we build.**

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 15 min (already doing it) |
| Context Window Anatomy | 25 min |
| The Lost in the Middle Problem | 30 min |
| Instruction Placement & Prompt Architecture | 25 min |
| Dynamic Context Compression | 30 min |
| Citation Injection Systems | 25 min |
| Multi-Turn Context Management | 20 min |
| Code: Building a Context Integration Pipeline | 60 min |
| Code: Dynamic Token Budget Manager | 45 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~6 hours** |

---

## 📖 Concept Deep-Dive

### 1. Context Window Anatomy

Before we talk about prompt structure, we need to understand what happens inside a transformer when you present context.

A transformer's attention mechanism processes all tokens in its context window in parallel. Every token can attend to every other token. But here's the key insight: **attention is not uniform across the context window.**

Research has consistently shown that transformers display positional biases:

- **Primacy bias**: Tokens near the beginning of the context get higher attention weight
- **Recency bias**: Tokens near the end of the context also get higher attention weight
- **Middle neglect**: Tokens in the middle of long contexts receive disproportionately lower attention

This isn't a bug — it's a consequence of how positional encodings interact with the softmax attention mechanism over long sequences. The middle of a long context has tokens competing with both left-side and right-side neighbors for attention mass, and they systematically lose.

**What this means for RAG:**

If you have 8 retrieved chunks and you place them all in the middle of your prompt (between a long system prompt and the user's question), the chunks in positions 4-6 will get less attention than chunks in positions 1-3 and 7-8 — regardless of their relevance scores.

```text
Bad layout:
[System prompt: 1500 tokens] [Chunks 1-8: 4000 tokens] [User question: 100 tokens]
                                                  ^
                                                  |
                                         Middle chunks get neglected
```

Better layout:
```text
[Relevant chunks: 2000 tokens] [System prompt: 500 tokens] [User question + remaining chunks: 2000 tokens]
                               ^                           ^
                               |                           |
                        Primacy zone                Recency zone
```

But we can do even better by understanding *what* we place in each zone.

---

### 2. The "Lost in the Middle" Problem — Deep Dive

The paper *"Lost in the Middle: How Language Models Use Long Contexts"* (Liu et al., 2023) conducted a systematic study of how LLMs use information at different positions in long input contexts. The findings were stark:

**When relevant information is at the beginning or end of the context, models answer correctly ~75% of the time. When it's in the middle, accuracy drops to ~50%.**

Let's think about what this means for RAG:

1. You retrieve 10 chunks, rank them by relevance
2. Chunk #3 (relevance score 0.89) and chunk #7 (relevance score 0.82) both contain the answer
3. You put them in order: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
4. Chunk #3 is in the "early middle" — gets decent attention
5. Chunk #7 is in the "late middle" — gets less attention
6. The model misses the information in chunk #7

**But here's the twist**: What if chunk #7 actually has *better* information but lower retrieval score because of embedding quirks? The model might miss the better answer because of positional neglect on top of retrieval ranking imperfectness.

#### Mitigation Strategies

**Strategy 1: Sandwich the most relevant**

Place the single most relevant chunk at the VERY beginning and the second most relevant at the VERY end. Then fill the middle with everything else sorted by relevance.

```
[Best chunk] [4th best] [5th best] [6th best] [7th best] [3rd best] [2nd best]
     ^                                                                    ^
     |                                                                    |
  Primacy hook                                                    Recency hook
```

**Strategy 2: Repeat key information**

If a chunk is essential (above a relevance threshold), include its key information twice — once early and once late. This is the RAG equivalent of the "tell them what you're going to tell them, tell them, tell them what you told them" technique.

```python
# Pseudo-code: strategic context repetition
def build_context(chunks, essential_threshold=0.9):
    essential = [c for c in chunks if c.score >= essential_threshold]
    supporting = [c for c in chunks if c.score < essential_threshold]

    # Place essential chunks at both start and end
    context_parts = []

    # Opening: brief summary of essential info
    for chunk in essential[:2]:
        context_parts.append(f"[KEY DOCUMENT: {chunk.title}]\n{chunk.summarize(100)}")

    # Middle: all chunks sorted by relevance
    context_parts.append("--- DETAILED REFERENCES ---")
    for chunk in sorted(chunks, key=lambda c: c.score, reverse=True):
        context_parts.append(f"[{chunk.id}] {chunk.text}")

    # Closing: re-state the most essential
    if essential:
        context_parts.append("--- CRITICAL INFORMATION ---")
        context_parts.append(essential[0].text)

    return "\n\n".join(context_parts)
```

**Strategy 3: Structural markers that guide attention**

Use clear, distinct markers that segment the context. Models attend better to clearly delineated sections than to undifferentiated blocks of text.

```text
╔══════════════════════════════════════════════╗
║           PRIMARY REFERENCE                  ║
╚══════════════════════════════════════════════╝
[Full text of most relevant chunk]

╔══════════════════════════════════════════════╗
║           SUPPORTING REFERENCES              ║
╚══════════════════════════════════════════════╝
[Chunks 2-5 in relevance order]

╔══════════════════════════════════════════════╗
║           ADDITIONAL CONTEXT                 ║
╚══════════════════════════════════════════════╝
[Chunks 6-10 in relevance order]
```

The visual demarcation helps the model segment the context and allocate attention accordingly.

---

### 3. Instruction Placement & Prompt Architecture

Where you put your instructions relative to the context profoundly affects how well the model follows them.

#### The Instruction Gradient

Research on instruction-tuned models shows a gradient of compliance:

```
Beginning of context  ──────────────────►  End of context
   HIGH compliance                         LOW compliance
   (system prompt)                         (user message)
```

But RAG complicates this because you have multiple components:

1. **System prompt** — persona, rules, output format
2. **Context** — retrieved chunks
3. **User question** — what they're asking
4. **Conversation history** — for multi-turn

#### Optimal Layout for RAG

After considerable experimentation (both in research and production), the most effective layout is:

```text
[SYSTEM PROMPT]              — Rules, persona, output format (concise)
[BRIEF CONTEXT HEADER]       — "Here are reference documents..."
[MOST RELEVANT CHUNK]        — Primacy zone: the most critical info
[SUPPORTING CHUNKS]          — Middle zone: supporting evidence
[REINFORCED INSTRUCTION]     — Reminder of what to do with the context
[REST OF CHUNKS]             — Recency zone: additional context
[USER QUESTION]              — The actual query (gets high attention naturally)
```

Why this works:
- The system prompt at the start sets the model's behavior frame
- The most relevant chunk in position 2 gets primacy attention
- The reinforced instruction before the user question reminds the model how to use the material that was just in the middle
- The user question at the end gets natural recency attention

**Critical nuance**: The reinforced instruction should reference the context specifically:

```text
Bad reinforced instruction:
"Answer the user's question based on the provided context."

Good reinforced instruction:
"IMPORTANT: The documents provided above between [PRIMARY REFERENCE] and [ADDITIONAL CONTEXT] contain the information you need. If the information in multiple documents conflicts, note the conflict and explain which source is more authoritative. If no document answers the question, say so explicitly — do not make up information."
```

---

### 4. Dynamic Context Compression

You can't always fit everything. Real-world RAG systems must handle variable-length contexts dynamically.

#### The Token Budget Problem

Given:
- Model context window: 8,192 tokens
- System prompt: 500 tokens
- Conversation history: 1,000 tokens (multi-turn)
- User question: 100 tokens
- Output tokens: 1,000 tokens (reserved)
- **Available for context: ~5,600 tokens**

With each chunk averaging 500 tokens, you can fit ~11 chunks. But the retriever might return 20+ chunks, or individual chunks might be 1,000+ tokens.

#### Compression Strategies

**Strategy A: Token-Limited Truncation**

The simplest approach — keep the highest-ranked chunks up to your token budget.

```python
def truncate_to_budget(chunks, budget_tokens, token_counter):
    """Keep highest-scored chunks until budget exhausted."""
    selected = []
    total_tokens = 0

    for chunk in sorted(chunks, key=lambda c: c.score, reverse=True):
        chunk_tokens = token_counter(chunk.text)
        if total_tokens + chunk_tokens <= budget_tokens:
            selected.append(chunk)
            total_tokens += chunk_tokens
        else:
            # Try to fit a truncated version
            remaining = budget_tokens - total_tokens
            if remaining > 100:  # Only include if meaningful
                chunk.text = truncate_text(chunk.text, remaining, token_counter)
                selected.append(chunk)
            break

    return selected
```

**Problem**: You might lose diversity. The top 8 chunks might all be from the same document, while chunks 9-12 from a different document contain complementary information.

**Strategy B: Maximal Marginal Relevance (MMR) Selection**

Select chunks that are both relevant AND diverse:

```python
def mmr_select(chunks, query_embedding, budget_tokens,
               token_counter, lambda_param=0.7):
    """Select chunks maximizing relevance + diversity."""
    selected = []
    candidates = list(chunks)
    total_tokens = 0

    while candidates and total_tokens < budget_tokens:
        best_idx = -1
        best_score = -float('inf')

        for i, candidate in enumerate(candidates):
            # Relevance to query
            relevance = cosine_similarity(
                candidate.embedding, query_embedding
            )

            # Diversity penalty: similarity to already selected
            if selected:
                max_sim_to_selected = max(
                    cosine_similarity(candidate.embedding, s.embedding)
                    for s in selected
                )
            else:
                max_sim_to_selected = 0

            # MMR score
            mmr = lambda_param * relevance - (1 - lambda_param) * max_sim_to_selected

            if mmr > best_score:
                best_score = mmr
                best_idx = i

        best_chunk = candidates.pop(best_idx)
        chunk_tokens = token_counter(best_chunk.text)

        if total_tokens + chunk_tokens <= budget_tokens:
            selected.append(best_chunk)
            total_tokens += chunk_tokens
        else:
            # Try truncated version
            remaining = budget_tokens - total_tokens
            if remaining > 100:
                best_chunk.text = truncate_text(
                    best_chunk.text, remaining, token_counter
                )
                selected.append(best_chunk)
            break

    return selected
```

**Strategy C: Summary-Based Compression**

When you can't fit a chunk, summarize it:

```python
async def compress_chunk(chunk, target_tokens, llm_client):
    """Summarize a chunk to fit within target token count."""
    current_tokens = count_tokens(chunk.text)

    if current_tokens <= target_tokens:
        return chunk  # No compression needed

    # Use LLM to create a lossy summary
    response = await llm_client.complete(
        f"Summarize the following text in {target_tokens} tokens "
        f"or less, preserving all specific facts, numbers, and names:\n\n"
        f"{chunk.text}"
    )

    chunk.text = response
    chunk.compressed = True
    chunk.original_length = current_tokens
    return chunk
```

**This is expensive** — you're calling the LLM for every chunk. Only use this for the most critical chunks or as a fallback.

**Strategy D: Structured Truncation**

Not all parts of a chunk are equally important. Use structure-aware truncation:

```python
def structured_truncation(chunk_text, target_tokens, token_counter):
    """Truncate preserving the most informative parts."""
    # Strategy: keep first 20% (intro/context) + last 60% (details/conclusions)
    # Drop middle 20% (elaboration, examples)
    total_tokens = token_counter(chunk_text)

    if total_tokens <= target_tokens:
        return chunk_text

    # Calculate token proportions
    keep_head_ratio = 0.2
    keep_tail_ratio = 0.6  # Bias toward keeping more of the tail
    head_tokens = int(target_tokens * keep_head_ratio)
    tail_tokens = int(target_tokens * keep_tail_ratio)

    words = chunk_text.split()
    # Approximate: assume 1.3 tokens per word
    head_words = int(head_tokens / 1.3)
    tail_words = int(tail_tokens / 1.3)

    head = " ".join(words[:head_words])
    tail = " ".join(words[-tail_words:])

    return f"{head}\n[...content truncated...]\n{tail}"
```

**In production, you'll likely use a hybrid**: try MMR first, fall back to truncation, and for high-value queries, use summary compression on the most important chunks.

---

### 5. Citation Injection Systems

Citations are the difference between a "helpful" RAG system and a "trustworthy" one. But getting citations right is surprisingly hard.

#### The Two Citation Problems

**Problem 1: Model invents citations**
The model cites "[3]" for a fact, but chunk [3] doesn't contain that fact. The model learned from training data that citations look like "[N]" and generates them even when the content doesn't match.

**Problem 2: Model omits citations**
The model uses information from the context but doesn't cite it, making the answer impossible to verify.

#### Citation Injection Architecture

The key insight: **number the chunks explicitly in the prompt, and instruct the model to reference those numbers.**

```python
def build_context_with_citations(chunks):
    """Build context with explicit, unambiguous citations."""
    context_parts = []
    for i, chunk in enumerate(chunks, 1):
        context_parts.append(
            f"[SOURCE {i}: {chunk.document_title} | "
            f"Relevance: {chunk.score:.2f}]\n"
            f"{chunk.text}"
        )
    return "\n\n---\n\n".join(context_parts)
```

Then the system prompt should instruct:

```
When answering, cite your sources using the [SOURCE N] identifiers.
Always cite the specific source for each factual claim.

Example:
"The API rate limit is 1000 requests per minute [SOURCE 3]."

If multiple sources support the same claim, cite all of them:
"The API rate limit is 1000 requests per minute [SOURCE 3][SOURCE 7]."

If sources contradict each other, explain the contradiction:
"Source [SOURCE 3] states the limit is 1000 requests/minute, while
source [SOURCE 7] states it was raised to 5000 in June 2024."
```

#### Enforcing Citation Accuracy

The most reliable technique is **structured output with citation validation**:

```python
from pydantic import BaseModel, Field
from typing import List, Optional

class Citation(BaseModel):
    source_id: int
    relevant_text: str = Field(
        description="The exact text from the source that supports this claim"
    )

class CitedClaim(BaseModel):
    claim: str = Field(description="A single factual claim")
    citations: List[Citation] = Field(
        description="Sources that support this claim, min 1"
    )

class AnswerWithCitations(BaseModel):
    answer: str = Field(description="The complete answer with citations inline")
    claims: List[CitedClaim] = Field(
        description="Breakdown of claims and evidence"
    )
```

With structured output, you can post-process to verify that cited sources actually contain the claimed text:

```python
def verify_citations(answer: AnswerWithCitations, chunks: List[Chunk]):
    """Check each citation claims exists in the cited source."""
    for claim in answer.claims:
        for citation in claim.citations:
            if citation.source_id > len(chunks):
                print(f"WARNING: Citation source {citation.source_id} out of range")
                continue

            source_text = chunks[citation.source_id - 1].text
            # Check if the cited text appears in the source
            if citation.relevant_text.lower() not in source_text.lower():
                print(
                    f"WARNING: Claim '{claim.claim[:50]}...' "
                    f"cites source {citation.source_id} but "
                    f"'{citation.relevant_text[:50]}...' not found in source"
                )
    return answer
```

#### The Document Title Strategy

A more natural approach: use document titles instead of numbers.

```
╔══════════════════════════════════════════════╗
║  REFERENCE DOCUMENTS                        ║
╚══════════════════════════════════════════════╝

[From: API Documentation v3.2 — Rate Limiting]
...

[From: API Changelog — June 2024 Changes]
...
```

Then instruct the model: "Cite sources by their document title." The model tends to generate more accurate title citations because the titles carry semantic meaning that the model can verify against.

---

### 6. Multi-Turn Context Management

Real RAG systems handle conversations, not single questions. Each turn adds conversation history, consuming precious context window space.

#### The Rolling Window Problem

```
Turn 1: Question + chunks → Answer (1K tokens)
Turn 2: History + Question + chunks → Answer (2K tokens)
Turn 3: History + Question + chunks → Answer (3K tokens)
...
Turn 8: Context window exhausted → System must evict
```

**What do you evict?**

**Strategy: Hierarchical Context Management**

```
┌──────────────────────────────────────────────────────┐
│                    CONTEXT WINDOW                     │
├──────────────────────────────────────────────────────┤
│  SYSTEM PROMPT (500 tokens) — Always kept            │
├──────────────────────────────────────────────────────┤
│  CONVERSATION SUMMARY (500 tokens) — Rolling summary │
├──────────────────────────────────────────────────────┤
│  RECENT HISTORY (1000 tokens) — Last 2-3 exchanges   │
├──────────────────────────────────────────────────────┤
│  RETRIEVED CHUNKS (variable) — Current query context │
├──────────────────────────────────────────────────────┤
│  USER QUESTION (100 tokens) — Current query          │
└──────────────────────────────────────────────────────┘
```

The conversation summary is a **running summary** that gets compressed each turn:

```python
class ConversationManager:
    def __init__(self, max_history_tokens=2000, summary_model=None):
        self.max_history_tokens = max_history_tokens
        self.summary_model = summary_model
        self.exchanges = []
        self.running_summary = ""

    async def add_exchange(self, question, context, answer):
        """Add a turn and compress if needed."""
        exchange = {
            "question": question,
            "context_preview": self._preview_context(context),
            "answer": answer
        }
        self.exchanges.append(exchange)
        await self._compress_if_needed()

    async def _compress_if_needed(self):
        """Compress history if exceeding budget."""
        history_text = self._format_history()
        if count_tokens(history_text) <= self.max_history_tokens:
            return

        # Keep last 2 exchanges verbatim, summarize the rest
        recent = self.exchanges[-2:]
        old = self.exchanges[:-2]

        if old:
            old_text = self._format_exchanges(old)
            response = await self.summary_model.complete(
                f"Summarize this conversation history concisely, "
                f"preserving key facts, user preferences, "
                f"and unresolved questions:\n\n{old_text}"
            )
            self.running_summary = response
            self.exchanges = recent

    def get_history_context(self):
        """Build the history section of the prompt."""
        parts = []
        if self.running_summary:
            parts.append(
                f"[CONVERSATION SUMMARY]\n{self.running_summary}"
            )
        if self.exchanges:
            parts.append(
                f"[RECENT EXCHANGES]\n{self._format_history()}"
            )
        return "\n\n".join(parts)

    def _format_history(self):
        lines = []
        for i, ex in enumerate(self.exchanges):
            lines.append(f"User: {ex['question']}")
            lines.append(f"Assistant: {ex['answer'][:200]}...")
        return "\n".join(lines)

    def _preview_context(self, chunks):
        """Summarize what information was retrieved."""
        sources = list(set(c.document_title for c in chunks))
        return f"[Retrieved from: {', '.join(sources[:3])}]"
```

**Critical design decision**: The conversation summary should be written by the **same model** that answers queries, using an explicit compression prompt. This ensures the summary format matches what the model naturally uses.

---

### 7. Context Window Architecture Patterns

Let me show you a comprehensive context integration pipeline that ties everything together:

```python
"""
context_integrator.py — Production RAG Context Assembly

This module handles the entire "Augment" step:
1. Dynamic chunk selection (MMR-based)
2. Token budget management
3. Multi-turn history compression
4. Citation formatting
5. Context window assembly
"""

import tiktoken
from dataclasses import dataclass, field
from typing import List, Optional, Dict, Any, Callable
import numpy as np


@dataclass
class Chunk:
    id: str
    text: str
    score: float
    embedding: List[float]
    document_title: str
    metadata: Dict[str, Any] = field(default_factory=dict)


@dataclass
class ConversationExchange:
    question: str
    answer: str
    retrieved_titles: List[str]


class ContextIntegrator:
    """
    Assembles the optimal context for a RAG query.

    Handles:
    - Dynamic chunk selection within token budgets
    - MMR-based diversity selection vs relevance-only
    - Conversation history compression
    - Citation formatting
    - Position optimization (Lost in the Middle mitigation)
    """

    def __init__(
        self,
        model_name: str = "gpt-4",
        max_context_tokens: int = 8192,
        reserve_output_tokens: int = 1024,
        system_prompt_tokens: int = 500,
        use_mmr: bool = True,
        mmr_lambda: float = 0.7,
        token_counter: Optional[Callable] = None
    ):
        self.model_name = model_name
        self.max_context_tokens = max_context_tokens
        self.reserve_output_tokens = reserve_output_tokens
        self.system_prompt_tokens = system_prompt_tokens
        self.use_mmr = use_mmr
        self.mmr_lambda = mmr_lambda

        # Token counter
        if token_counter:
            self.count_tokens = token_counter
        else:
            encoding = tiktoken.encoding_for_model(model_name)
            self.count_tokens = lambda text: len(encoding.encode(text))

    def get_available_budget(self, question: str) -> int:
        """Calculate token budget available for context."""
        question_tokens = self.count_tokens(question)
        budget = (
            self.max_context_tokens
            - self.reserve_output_tokens
            - self.system_prompt_tokens
            - question_tokens
        )
        return max(budget, 500)  # Always reserve at least 500 for minimal context

    def select_chunks(
        self,
        chunks: List[Chunk],
        query_embedding: List[float],
        budget_tokens: int
    ) -> List[Chunk]:
        """Select chunks respecting token budget."""
        if not chunks:
            return []

        if self.use_mmr and len(chunks) > 3:
            return self._mmr_select(chunks, query_embedding, budget_tokens)
        else:
            return self._relevance_truncate(chunks, budget_tokens)

    def _relevance_truncate(
        self, chunks: List[Chunk], budget_tokens: int
    ) -> List[Chunk]:
        """Simple relevance-based truncation."""
        sorted_chunks = sorted(chunks, key=lambda c: c.score, reverse=True)
        selected = []
        total = 0

        for chunk in sorted_chunks:
            tokens = self.count_tokens(chunk.text)
            if total + tokens <= budget_tokens:
                selected.append(chunk)
                total += tokens
            else:
                remaining = budget_tokens - total
                if remaining > 200:
                    chunk.text = self._truncate_text(
                        chunk.text, remaining
                    )
                    selected.append(chunk)
                break

        return selected

    def _mmr_select(
        self,
        chunks: List[Chunk],
        query_embedding: List[float],
        budget_tokens: int
    ) -> List[Chunk]:
        """MMR-based selection for relevance + diversity."""
        selected = []
        candidates = list(chunks)
        total_tokens = 0

        while candidates and total_tokens < budget_tokens:
            best_idx = -1
            best_score = -float('inf')

            for i, c in enumerate(candidates):
                relevance = self._cosine_sim(c.embedding, query_embedding)

                if selected:
                    diversity_penalty = max(
                        self._cosine_sim(c.embedding, s.embedding)
                        for s in selected
                    )
                else:
                    diversity_penalty = 0

                mmr = self.mmr_lambda * relevance - (
                    1 - self.mmr_lambda
                ) * diversity_penalty

                if mmr > best_score:
                    best_score = mmr
                    best_idx = i

            best = candidates.pop(best_idx)
            tokens = self.count_tokens(best.text)

            if total_tokens + tokens <= budget_tokens:
                selected.append(best)
                total_tokens += tokens
            else:
                remaining = budget_tokens - total_tokens
                if remaining > 200:
                    best.text = self._truncate_text(best.text, remaining)
                    selected.append(best)
                break

        return selected

    def position_chunks(self, chunks: List[Chunk]) -> List[Chunk]:
        """
        Position chunks to mitigate Lost in the Middle.

        Places the most relevant chunk at the start (primacy),
        the second most relevant at the end (recency),
        and fills the middle with the rest sorted by relevance.
        """
        if len(chunks) <= 2:
            return chunks

        sorted_chunks = sorted(chunks, key=lambda c: c.score, reverse=True)

        # Best chunk goes first, second best goes last
        best = [sorted_chunks[0]]
        middle = sorted_chunks[1:-1]  # Middle sorted by relevance descending
        second_best = [sorted_chunks[-1]]

        return best + middle + second_best

    def format_citations(self, chunks: List[Chunk]) -> str:
        """Format chunks with citation markers."""
        parts = []
        for i, chunk in enumerate(chunks, 1):
            # Include document title in citation for semantic grounding
            source_line = (
                f"[SOURCE {i}: {chunk.document_title} "
                f"(relevance: {chunk.score:.3f})]"
            )
            parts.append(f"{source_line}\n{chunk.text}")

        return "\n\n---\n\n".join(parts)

    def build_system_prompt(self) -> str:
        """Build the system prompt with citation instructions."""
        return (
            "You are a helpful assistant that answers questions based "
            "on the provided reference documents.\n\n"

            "RULES:\n"
            "1. Answer based ONLY on the provided [SOURCE N] documents.\n"
            "2. If the documents don't contain the answer, say so explicitly.\n"
            "3. Always cite sources using [SOURCE N] notation.\n"
            "4. When sources contradict, explain the contradiction.\n"
            "5. Never make up information or use your own knowledge.\n"
            "6. If asked about something outside the documents, "
            "say 'I cannot answer this from the provided documents.'\n\n"

            "CITATION FORMAT:\n"
            "Factual claim here [SOURCE 1].\n"
            "Claim from multiple sources [SOURCE 1][SOURCE 3].\n"
            "Contradiction: Source A says X [SOURCE 2], "
            "while Source B says Y [SOURCE 5]."
        )

    async def build_prompt(
        self,
        question: str,
        chunks: List[Chunk],
        query_embedding: List[float],
        conversation_history: Optional[List[ConversationExchange]] = None,
        conversation_summary: Optional[str] = None
    ) -> str:
        """
        Build the complete RAG prompt.

        This is the main entry point — it orchestrates the entire
        context assembly pipeline.
        """
        # 1. Calculate budget
        budget_tokens = self.get_available_budget(question)

        # 2. Reserve tokens for conversation history
        history_tokens = 0
        history_section = ""

        if conversation_summary:
            history_tokens += self.count_tokens(conversation_summary)
            history_section = (
                f"[CONVERSATION SUMMARY]\n{conversation_summary}\n\n"
            )

        if conversation_history:
            for ex in conversation_history[-2:]:  # Keep last 2
                exchange_text = (
                    f"User: {ex.question}\n"
                    f"Assistant: {ex.answer}\n"
                )
                history_tokens += self.count_tokens(exchange_text)
                history_section += exchange_text

        context_budget = budget_tokens - history_tokens

        # 3. Select chunks within budget
        selected = self.select_chunks(chunks, query_embedding, context_budget)

        # 4. Position them optimally
        positioned = self.position_chunks(selected)

        # 5. Format context
        context = self.format_citations(positioned)

        # 6. Assemble the full prompt
        system = self.build_system_prompt()

        # 7. Check if we need to add a "context exhausted" note
        context_note = ""
        if len(chunks) > len(selected):
            context_note = (
                "\n\n[NOTE: Some retrieved documents were excluded "
                "due to context length limits. The most relevant "
                f"documents ({len(selected)} of {len(chunks)}) "
                "are provided above.]"
            )

        prompt = (
            f"{system}\n\n"
            f"{history_section}"
            f"[REFERENCE DOCUMENTS]\n"
            f"{context}"
            f"{context_note}\n\n"
            f"[USER QUESTION]\n{question}\n\n"
            f"[INSTRUCTION]\n"
            f"Answer the user's question based on the reference documents. "
            f"Cite sources. If uncertain, say so."
        )

        return prompt

    def _cosine_sim(self, a: List[float], b: List[float]) -> float:
        a = np.array(a)
        b = np.array(b)
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

    def _truncate_text(self, text: str, target_tokens: int) -> str:
        """Truncate text to target token count, preserving structure."""
        tokens = self.count_tokens(text)
        if tokens <= target_tokens:
            return text

        # Keep head and tail, drop middle
        # Bias heavily toward the tail (where conclusions usually are)
        head_ratio = 0.3
        tail_ratio = 0.7

        head_tokens = int(target_tokens * head_ratio)
        tail_tokens = int(target_tokens * tail_ratio)

        # Approximate: ~0.75 words per token for English
        words = text.split()
        head_words = int(head_tokens * 0.75)
        tail_words = int(tail_tokens * 0.75)

        if head_words + tail_words >= len(words):
            return text

        head = " ".join(words[:head_words])
        tail = " ".join(words[-tail_words:])

        return f"{head}\n[...{tokens - target_tokens} tokens truncated...]\n{tail}"


# ============================================================
# Usage Example
# ============================================================

async def main():
    # Sample data
    chunks = [
        Chunk(
            id="c1",
            text="The API rate limit is 1000 requests per minute for the free tier.",
            score=0.92,
            embedding=[0.1] * 384,
            document_title="API Documentation v3.2"
        ),
        Chunk(
            id="c2",
            text="Enterprise tier customers can make up to 10000 requests per minute.",
            score=0.88,
            embedding=[0.15] * 384,
            document_title="API Documentation v3.2"
        ),
        Chunk(
            id="c3",
            text="The rate limit was raised from 1000 to 5000 in June 2024.",
            score=0.76,
            embedding=[0.12] * 384,
            document_title="API Changelog June 2024"
        ),
        Chunk(
            id="c4",
            text="Rate limits reset at midnight UTC regardless of tier.",
            score=0.65,
            embedding=[0.08] * 384,
            document_title="FAQ"
        ),
    ]

    query_embedding = [0.11] * 384

    integrator = ContextIntegrator(
        model_name="gpt-4",
        max_context_tokens=4096,
        use_mmr=True,
        mmr_lambda=0.7
    )

    prompt = await integrator.build_prompt(
        question="What's the current API rate limit?",
        chunks=chunks,
        query_embedding=query_embedding,
        conversation_summary="User has been asking about API limits."
    )

    print(prompt)
    """
    This will output a well-structured prompt with:
    - System prompt with citation rules
    - Conversation summary
    - Chunks in optimal positions (most relevant first, second most relevant last)
    - Explicit citation markers
    - Reinforced instruction before the question
    """


# ============================================================
# Self-Correction: Where This Architecture Could Go Wrong
# ============================================================

# 1. Token counting drift: tiktoken counts change between model versions.
#    Always verify with actual API responses.

# 2. MMR diversity works against specificity: If the query is very specific,
#    diversity selection might exclude the exact match. Set lambda_param
#    based on query type — use lower diversity (higher lambda) for specific
#    factual queries, higher diversity for exploration queries.

# 3. Position optimization assumes the model uses retrieved chunks at all.
#    Some models (especially smaller ones) struggle with long contexts
#    regardless of positioning. For these, use fewer, more carefully
#    selected chunks.

# 4. Citation verification is best-effort: Even with verification, the model
#    can cite the right source for the wrong reason. Citation accuracy
#    requires end-to-end eval, not just source-text matching.
```

---

### 8. The Context Assembly Checklist

When you build your context integration pipeline, here's the decision tree:

```
START: Got retrieved chunks + user question
│
├─ Step 1: Calculate token budget
│  ├─ Model max context
│  ├─ Minus: system prompt, output reservation, conversation history
│  └─ = Available for context chunks
│
├─ Step 2: Select chunks within budget
│  ├─ Simple question requiring specific fact?
│  │  └─ Use relevance-only truncation (highest scores first)
│  ├─ Complex question needing multiple perspectives?
│  │  └─ Use MMR selection (relevance + diversity)
│  └─ Few chunks returned?
│     └─ Use all of them, no selection needed
│
├─ Step 3: Position selected chunks
│  ├─ 1-2 chunks: natural order
│  ├─ 3-5 chunks: best-first, rest middle, second-best last
│  └─ 6+ chunks: best-first with structural markers
│     ├─ [PRIMARY REFERENCE]
│     ├─ [SUPPORTING REFERENCES]
│     └─ [ADDITIONAL CONTEXT]
│
├─ Step 4: Format citations
│  ├─ Use [SOURCE N: document_title] format
│  └─ Ensure citation IDs are visible and unambiguous
│
├─ Step 5: Build conversation context
│  ├─ Has running summary? Include
│  ├─ Has recent exchanges? Include last 2
│  └─ Budget exceeded? Compress oldest exchanges
│
├─ Step 6: Construct final prompt
│  ├─ System prompt (rules, format, citation guidelines)
│  ├─ Conversation context (summary + recent)
│  ├─ Reference documents (positioned and formatted)
│  ├─ Context exhaustion note (if chunks were excluded)
│  ├─ User question
│  └─ Reinforced instruction (brief reminder)
│
└─ Step 7: Verify token count before sending
   └─ Over budget? Reduce chunk count, truncate oldest history
```

---

## 💻 Code Examples

### Example 1: Naive vs. Optimized Context — Side by Side

```python
"""Compare naive context integration vs. optimized integration."""

import tiktoken

# Sample chunks
chunks_data = [
    ("The human attention span has declined from 12 seconds in 2000 to 8 seconds today.", 0.95),
    ("Studies show goldfish have an attention span of 9 seconds.", 0.92),
    ("The average office worker checks email 74 times per day.", 0.45),
    ("Multitasking can reduce productivity by up to 40%.", 0.40),
    ("Regular breaks improve cognitive function and focus.", 0.38),
    ("Sleep deprivation impairs decision-making capabilities.", 0.35),
    ("Meditation has been shown to increase attention span.", 0.32),
    ("The Pomodoro Technique uses 25-minute work intervals.", 0.30),
]

chunks = [(text, score) for text, score in chunks_data]

encoding = tiktoken.encoding_for_model("gpt-4")

# === NAIVE APPROACH ===
def naive_context(chunks, question):
    """The naive approach: concatenate everything in retrieval order."""
    context = "\n\n".join([text for text, _ in chunks])
    prompt = f"""Answer the question based on this context:

{context}

Question: {question}

Answer:"""
    return prompt

# === OPTIMIZED APPROACH ===
def optimized_context(chunks, question):
    """The optimized approach: position, filter, instruct."""

    # 1. Filter: keep only relevant chunks (score > 0.5)
    relevant = [(text, score) for text, score in chunks if score > 0.5]

    if not relevant:
        return "I cannot answer this from the provided documents."

    # 2. Sort by relevance, descending
    relevant.sort(key=lambda x: x[1], reverse=True)

    # 3. Position: best first, rest sorted, second-best last
    if len(relevant) >= 3:
        best = [relevant[0]]
        middle = relevant[1:-1]
        second_best = [relevant[-1]]
        positioned = best + middle + second_best
    else:
        positioned = relevant

    # 4. Format with structural markers and citations
    context_parts = []
    for i, (text, score) in enumerate(positioned, 1):
        context_parts.append(f"[SOURCE {i} (relevance: {score:.2f})]\n{text}")

    context = "\n\n---\n\n".join(context_parts)

    # 5. Build complete prompt with instructions
    prompt = f"""[SYSTEM]
You are a research assistant. Answer based ONLY on the provided sources.
Cite sources using [SOURCE N]. If no source supports an answer, say so.

[REFERENCE DOCUMENTS]
{context}

[QUESTION]
{question}

[INSTRUCTION]
Answer the question. Cite your sources. If the sources contradict,
explain the contradiction.
"""
    return prompt


question = "How does the human attention span compare to a goldfish's?"

print("=== NAIVE PROMPT ===")
naive = naive_context(chunks, question)
print(naive)
print(f"\nToken count: {len(encoding.encode(naive))}")
print("\n" + "="*50 + "\n")

print("=== OPTIMIZED PROMPT ===")
optimized = optimized_context(chunks, question)
print(optimized)
print(f"\nToken count: {len(encoding.encode(optimized))}")

# Notice:
# 1. Optimized version excluded irrelevant chunks (saves ~100 tokens)
# 2. Optimized version puts the contradiction front-and-center
# 3. Optimized version instructs the model to notice contradictions
# 4. Optimized version uses structural markers and citation formatting
#
# The model will produce DIFFERENT quality answers from these two prompts
# even though they contain the same relevant information.
```

### Example 2: Structured Citation Extraction

```python
"""Extract and verify citations from LLM responses."""

from pydantic import BaseModel, Field
from typing import List
import re
from openai import AsyncOpenAI

client = AsyncOpenAI()

class CitationCheck(BaseModel):
    """Model for citation extraction."""
    sources_cited: List[int] = Field(
        description="Source numbers cited in the response"
    )
    claims_per_source: dict = Field(
        description="Map of source_id -> list of claims attributed to it"
    )
    has_unsupported_claims: bool = Field(
        description="Whether the response makes claims without citations"
    )

async def extract_citations(response: str, context_chunks: List[str]):
    """Extract and verify citations from an LLM response."""

    # 1. Extract all [SOURCE N] references
    citation_pattern = r'\[SOURCE (\d+)\]'
    matches = re.findall(citation_pattern, response)
    cited_sources = [int(m) for m in matches]

    print(f"Cited sources: {cited_sources}")
    print(f"Available sources: 1-{len(context_chunks)}")

    # 2. Check for hallucinated sources
    valid_sources = set(range(1, len(context_chunks) + 1))
    hallucinated = [s for s in cited_sources if s not in valid_sources]

    if hallucinated:
        print(f"⚠ WARNING: Hallucinated sources: {hallucinated}")

    # 3. Check that the model actually used the best sources
    # This is a heuristic — not definitive
    if cited_sources:
        # The most relevant sources should ideally be cited
        least_cited = min(cited_sources)
        if least_cited > 1:
            print(
                f"⚠ NOTE: Best source [SOURCE 1] not cited. "
                f"Model cited sources starting from [{least_cited}]."
            )

    # 4. Use LLM to do deeper citation verification
    check_prompt = f"""
Analyze this response for citation accuracy:

Chunks provided to the model:
{chr(10) + chr(10).join(f'[SOURCE {i+1}]: {c[:200]}' for i, c in enumerate(context_chunks))}

Model response:
{response}

For each [SOURCE N] citation in the response, determine:
1. Does the cited source actually contain the information?
2. Is the information accurately represented (not distorted)?
3. Is any factual claim in the response NOT supported by any source?

Return as structured data.
"""

    result = await client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Analyze citation accuracy."},
            {"role": "user", "content": check_prompt}
        ],
        response_model=CitationCheck
    )

    return result


# Usage
chunks = [
    "The human attention span has declined from 12 seconds in 2000 to 8 seconds today.",
    "Studies show goldfish have an attention span of 9 seconds.",
    "The average human attention span is now shorter than that of a goldfish.",
]

response = """
Based on the provided sources:

The human attention span has declined to approximately 8 seconds [SOURCE 1],
which is indeed shorter than a goldfish's attention span of 9 seconds [SOURCE 2].
This means a goldfish can maintain focus longer than the average human [SOURCE 3].
"""

result = await extract_citations(response, chunks)
```

### Example 3: Dynamic Token Budget with Query Complexity Detection

```python
"""Detect query complexity and allocate context accordingly."""

from dataclasses import dataclass
from typing import List
import tiktoken


@dataclass
class QueryComplexity:
    is_complex: bool
    estimated_chunks_needed: int
    reason: str


class QueryComplexityAnalyzer:
    """Analyzes query complexity to determine context budget."""

    def __init__(self, llm_client=None):
        self.llm_client = llm_client
        self.encoding = tiktoken.encoding_for_model("gpt-4")

        # Simple keyword-based complexity detection (fast path)
        self.comparison_keywords = {
            "compare", "contrast", "difference", "versus", "vs",
            "similarities", "differences", "pros and cons"
        }
        self.list_keywords = {
            "list", "all", "every", "each", "enum", "various",
            "types of", "kinds of", "categories"
        }
        self.analytical_keywords = {
            "why", "how does", "explain", "analyze",
            "relationship", "correlation", "impact", "effect"
        }

    def analyze_fast(self, question: str) -> QueryComplexity:
        """Quick keyword-based complexity estimate."""
        question_lower = question.lower()

        # Comparison queries need multiple chunks for different entities
        if any(kw in question_lower for kw in self.comparison_keywords):
            return QueryComplexity(
                is_complex=True,
                estimated_chunks_needed=8,
                reason="Comparison query — needs coverage of multiple subjects"
            )

        # Listing queries need comprehensive coverage
        if any(kw in question_lower for kw in self.list_keywords):
            return QueryComplexity(
                is_complex=True,
                estimated_chunks_needed=10,
                reason="Enumeration query — needs comprehensive coverage"
            )

        # Analytical queries benefit from broader context
        if any(kw in question_lower for kw in self.analytical_keywords):
            return QueryComplexity(
                is_complex=True,
                estimated_chunks_needed=6,
                reason="Analytical query — benefits from multiple perspectives"
            )

        # Simple factual queries
        return QueryComplexity(
            is_complex=False,
            estimated_chunks_needed=3,
            reason="Factual query — likely answerable from few sources"
        )

    async def analyze_deep(self, question: str) -> QueryComplexity:
        """LLM-based complexity analysis (slower, more accurate)."""

        if not self.llm_client:
            return self.analyze_fast(question)

        response = await self.llm_client.complete(
            f"Analyze this user query for a RAG system. "
            f"Determine how many retrieved chunks it likely needs "
            f"(3-10 range, where 3 = simple fact, 10 = broad comparison):\n\n"
            f"Query: {question}\n\n"
            f"Return only a number between 3 and 10."
        )

        try:
            num = int(response.strip())
            num = max(3, min(10, num))
            return QueryComplexity(
                is_complex=num > 5,
                estimated_chunks_needed=num,
                reason="LLM-based complexity analysis"
            )
        except ValueError:
            return self.analyze_fast(question)


class AdaptiveContextAllocator:
    """
    Dynamically allocates context based on query complexity.
    Simple questions get high-quality, focused context.
    Complex questions get broader, more diverse context.
    """

    def __init__(self, max_context_tokens: int = 8192):
        self.max_context_tokens = max_context_tokens
        self.complexity_analyzer = QueryComplexityAnalyzer()
        self.encoding = tiktoken.encoding_for_model("gpt-4")

    def allocate(
        self,
        chunks: List,
        query_complexity: QueryComplexity,
        system_prompt: str,
        question: str
    ) -> dict:
        """
        Allocate context budget based on query complexity.

        Returns allocation plan.
        """
        total_budget = self.max_context_tokens

        # Fixed allocations
        system_tokens = len(self.encoding.encode(system_prompt))
        question_tokens = len(self.encoding.encode(question))
        output_reserve = 1024

        # Remaining for chunks + conversation history
        available = total_budget - system_tokens - question_tokens - output_reserve

        if query_complexity.is_complex:
            # Complex query: spread budget across more chunks
            # Each chunk gets fewer tokens (possibly truncated)
            min_tokens_per_chunk = 300
            max_chunks = min(
                query_complexity.estimated_chunks_needed,
                available // min_tokens_per_chunk
            )
            tokens_per_chunk = available // max_chunks

            allocation_type = "broad"
        else:
            # Simple query: fewer chunks, higher quality per chunk
            max_chunks = min(
                query_complexity.estimated_chunks_needed,
                available // 500  # More tokens per chunk
            )
            tokens_per_chunk = available // max_chunks

            allocation_type = "deep"

        return {
            "total_budget": total_budget,
            "available_for_context": available,
            "max_chunks": max_chunks,
            "tokens_per_chunk": tokens_per_chunk,
            "allocation_type": allocation_type,
            "chunks_to_use": chunks[:max_chunks]
        }


# Usage
analyzer = QueryComplexityAnalyzer()

simple_query = "What is the refund policy for annual plans?"
complex_query = "Compare the enterprise, business, and startup pricing tiers including all features, limitations, and usage caps."

for q in [simple_query, complex_query]:
    complexity = analyzer.analyze_fast(q)
    print(f"Query: {q[:50]}...")
    print(f"  Complexity: {'COMPLEX' if complexity.is_complex else 'SIMPLE'}")
    print(f"  Estimated chunks needed: {complexity.estimated_chunks_needed}")
    print(f"  Reason: {complexity.reason}")
    print()
```

---

## ✅ Good Output Examples

### What Proper Context Integration Looks Like

**Good — Structured context with clear sections:**

```text
[SYSTEM]
You are a support agent answering based on product documentation.
Cite sources using [SOURCE N]. If sources contradict, explain.

[CONVERSATION SUMMARY]
User has been asking about billing. Established they are on the
Enterprise plan. Previous question was about team member limits.

[REFERENCE DOCUMENTS]

[SOURCE 1: Enterprise Pricing Guide (relevance: 0.94)]
Enterprise plans include unlimited team members, 99.99% SLA,
and dedicated support with 15-minute response time.

[SOURCE 2: Enterprise vs Business Comparison (relevance: 0.87)]
The key difference: Business plans cap team members at 50,
while Enterprise has no upper limit.

[SOURCE 3: Pricing FAQ (relevance: 0.62)]
All plans are billed monthly or annually. Annual plans get 20% discount.

[NOTE: 2 of 7 retrieved documents shown due to context limits.
The most relevant documents are provided above.]

[USER QUESTION]
How many team members can I add on my current plan?

[INSTRUCTION]
Answer based only on the provided sources. Cite your sources.
```

This works because:
- The system prompt sets clear expectations
- Conversation summary gives context without eating the budget
- Sources are ordered by relevance with clear formatting
- The "NOTICE" prevents the model from thinking it has complete information
- The reinforced instruction stops the model from ignoring the context

**Bad — Undifferentiated text block:**

```text
System: answer based on context

Enterprise plans include unlimited team members 99.99% SLA and dedicated support with 15-minute response time

The key difference Business plans cap team members at 50 while Enterprise has no upper limit

All plans are billed monthly or annually Annual plans get 20% discount

how many team members can i add on my current plan

answer:
```

This fails because:
- No clear separation between instructions, context, and question
- No citation markers — model can't cite sources
- No indication of which sources are most important
- Missing metadata (plan relevance to user's current plan)
- The model doesn't know what to do with the context

---

## ❌ Antipatterns & Failure Modes

### 1. The Firehose Pattern

**What**: Dumping everything retrieved into the context without selection, ordering, or structure.

```python
# ANTIPATTERN
prompt = f"Context:\n{''.join(c.text for c in all_chunks)}\n\nQuestion: {q}"
```

**Why it fails**: The model gets overwhelmed and either hallucinates from irrelevant info or misses the relevant info buried in the middle. Wastes tokens on low-value chunks.

**Fix**: Always select, position, and structure your context. Even 3 well-positioned chunks beat 10 random ones.

### 2. The Citation Hallucination Loop

**What**: Using citation formatting but not verifying that the model actually cites correctly.

**Why it fails**: Models learn that "[SOURCE N]" looks authoritative and generate citations even when the source doesn't support the claim. The citation format becomes a *license to hallucinate convincingly*.

**Fix**: Always couple citation formatting with:
1. Explicit instruction: "Only cite SOURCE N if N actually contains this information"
2. Structured output that forces citation-to-text mapping
3. Post-processing verification (check cited text exists in source)

### 3. The Token Budget Ignorance

**What**: Not calculating tokens before sending the prompt, assuming it fits.

```python
# ANTIPATTERN
prompt = f"""
System: {system_prompt}
Context: {all_context}
History: {full_history}
Question: {question}
"""
await llm.complete(prompt)  # Might get truncated silently!
```

**Why it fails**: LLM APIs either silently truncate the input (losing the user's question or the reinforced instruction) or throw an error. Either way, the RAG system breaks.

**Fix**: Always pre-calculate token counts and have a truncation strategy before sending.

### 4. The Static Structure Fallacy

**What**: Using the same context structure for every query type.

**Why it fails**: A simple "what's the refund policy?" query needs 2-3 focused chunks, not a full MMR-selected context. A "compare all tiers" query needs 8-10 diverse chunks. Using one template for both wastes tokens on the simple query and provides insufficient coverage for the complex one.

**Fix**: Implement query complexity detection and dynamic context allocation.

### 5. The Conversation Overwrite

**What**: Including full conversation history without compression, crowding out retrieved context.

**Why it fails**: After 5 turns, the history consumes most of the context window. The retrieved chunks for the current question get squeezed into a tiny token budget, making the RAG system effectively a non-RAG chatbot.

**Fix**: Implement conversation summarization. Compress history to 20% of its original size after 2-3 turns. Keep only the last exchange verbatim.

### 6. The Silent Drop

**What**: Excluding chunks due to token budget without telling the LLM.

**Why it fails**: The model assumes it has complete information. When the excluded chunk contained critical information, the model confidently answers incorrectly.

**Fix**: Always include a context exhaustion notice: "NOTE: X of Y retrieved documents excluded due to context limits."

### 7. The Citation Overload

**What**: Including 15+ citations in a single sentence.

**Why it fails**: The citation string becomes noisy and the LLM starts generating them incorrectly. Models have been observed to "exhaust" their citation behavior after 5-6 citations in a single response.

**Fix**: Limit citations to 3-4 per claim. Use the most authoritative sources only. Post-process to remove redundant citations.

---

## 🧪 Drills & Challenges

### Drill 1: Fix the Broken Context (30 min)

You're given this RAG prompt that produces poor answers:

```text
System: You are a helpful assistant.

Context:
The capital of France is Paris. The capital of Germany is Berlin.
The population of Paris is 2.1 million.
The population of Berlin is 3.6 million.
The Eiffel Tower is in Paris.
The Brandenburg Gate is in Berlin.
Paris has a Mediterranean climate.
Berlin has a continental climate.
Both cities have extensive public transportation.
France has a unitary semi-presidential system.
Germany has a federal parliamentary system.

Question: Compare Paris and Berlin as potential relocation destinations.
```

**Tasks:**
1. Identify at least 5 problems with how this context is structured
2. Restructure the context for optimal LLM comprehension and citation accuracy
3. Add the system prompt, structural markers, and citation format
4. Show the improved prompt

### Drill 2: Build a Token Budget Manager (45 min)

Write a class that:
- Takes a list of chunks with token counts and relevance scores
- Takes a target token budget
- Implements MMR selection to choose diverse chunks within budget
- Falls back to relevance-only truncation if MMR is too slow
- Returns the selected chunks and a note about what was excluded
- Handles edge case: budget is too small for even one chunk (return a truncated version of the best chunk)

**Requirements:**
- Handle 100+ chunks efficiently (don't O(n²) the MMR similarity matrix)
- Properly chunk the truncated text (don't cut in the middle of a word)
- The MMR diversity tradeoff should be configurable per-query

### Drill 3: The Citation Accuracy Audit (40 min)

You have a RAG system that returns answers like this:

```
The API supports both REST and GraphQL endpoints [SOURCE 1].
Rate limiting is applied per API key [SOURCE 2].
Webhook retries follow an exponential backoff pattern [SOURCE 3].
The maximum webhook payload size is 1MB [SOURCE 4].
```

You need to verify these citations. Write a function that:

1. Extracts all [SOURCE N] references from the response
2. For each cited source, checks if the referenced text actually appears in the source document
3. Flags any claim without a citation as potentially hallucinated
4. Returns a structured citation report with: valid citations, invalid citations, uncited claims

**Test your function against this response and these sources:**

```python
sources = {
    1: "Our API provides REST endpoints for all CRUD operations. GraphQL is available as a preview feature.",
    2: "API keys are used for authentication. Rate limits are applied per API key.",
    3: "Webhooks use exponential backoff retry with a maximum of 5 attempts.",
    4: "Webhook payloads are limited to 256KB. Larger payloads will be rejected.",  # Contradicts the answer!
}
```

### Drill 4: Multi-Turn Context Crisis (35 min)

You're building a customer support RAG bot. After 12 exchanges with a user, the context window is full. The current question is about international shipping rates.

The conversation history is:
```python
conversation = [
    {"q": "What's your return policy?", "a": "Returns within 30 days..."},
    {"q": "Do you ship to Canada?", "a": "Yes, we ship to Canada..."},
    {"q": "What are the shipping costs to Toronto?", "a": "$15 standard, $25 express..."},
    {"q": "How long does standard shipping take?", "a": "7-10 business days..."},
    {"q": "Can I return if I don't like the product?", "a": "Yes, within 30 days..."},
    {"q": "What about returns from Canada?", "a": "You pay return shipping..."},
    {"q": "Is there a warranty?", "a": "1-year limited warranty..."},
    {"q": "Does the warranty cover accidental damage?", "a": "No, only manufacturing defects..."},
    {"q": "What currency are prices in?", "a": "USD for all international orders..."},
    {"q": "Do you have a presence in Europe?", "a": "We ship to EU but no warehouse yet..."},
    {"q": "What are customs fees?", "a": "Buyer is responsible for customs..."},
    {"q": "How do I track my order?", "a": "Tracking link emailed within 24 hours..."},
]
```

**Tasks:**
1. Design a compression strategy that preserves context needed for the shipping question
2. Write a function that takes the full history and current query, and produces a compressed history (max 500 tokens)
3. Show the compressed output
4. Explain what information you kept, what you summarized, and what you dropped — and why

### Drill 5: Build a Complete Context Integrator (60 min)

Combine everything you've learned into a single `ContextIntegrator` class. It should:

1. Accept a question, list of chunks, query embedding, and optional conversation history
2. Detect query complexity (simple vs. complex)
3. Select chunks using either relevance-only or MMR based on complexity
4. Position chunks to mitigate Lost in the Middle
5. Format with structural markers and citations
6. Handle conversation history compression (if provided)
7. Include a context exhaustion note (if chunks were excluded)
8. Build and return the final prompt
9. Validate the final token count against the model's limit

**Edge cases to handle:**
- Zero chunks returned (fallback: tell the LLM no documents found)
- Single chunk (skip MMR, skip positioning complexity)
- Very long single chunk (truncate with structured truncation)
- Conversation history plus chunks exceed budget (compress history first, then chunks)
- Query that's clearly outside domain (instruct LLM to decline gracefully)

### Drill 6: The Citation Format Experiment (30 min)

Create an experiment that tests three different citation formats:

1. Numeric: `[SOURCE 1]`
2. Semantic: `[From: API Documentation]`
3. Embedded: `According to the API Documentation (v3.2, page 14):`

For each format, generate a test prompt with the same content and ask the same question. Run each through an LLM (or simulate with a test script).

**What to measure:**
- Citation accuracy: Does the cited source match the claim?
- Citation frequency: Does the model cite more or less with each format?
- Answer quality: Does the format affect the actual answer content?

**Write a test harness that:**
1. Generates prompts for all three formats
2. Collects responses
3. Analyzes citation patterns
4. Reports which format produces the most reliable citations

---

## 🚦 Gate Check

Before proceeding to Phase 4.05 (RAG Failure Modes), you must demonstrate:

1. **Build your own context integrator**: Create a `ContextIntegrator` class that handles the full pipeline — budget calculation, chunk selection, positioning, formatting, and validation. It must handle dynamic token budgets based on query complexity and conversation history.

2. **Measure the difference**: Run a simple A/B test with an LLM. For the same question and same chunks, compare:
   - Naive context (concatenate everything, no structure)
   - Your optimized context (with positioning, structure, citation format)
   
   Document at least 3 differences in the quality of the responses.

3. **Fix a broken integration**: Take a real or simulated poor context integration and identify at least 5 specific problems. Fix each one and show the before/after.

4. **Explain your design decisions**: In a short write-up, answer:
   - When would you use MMR vs. relevance-only selection?
   - How do you decide token budget allocation between chunks and conversation history?
   - What's your strategy for handling context exhaustion (not all chunks fit)?
   - How do you verify citation accuracy?

**Passing criteria:** Your context integrator produces prompts where the LLM:
- Correctly cites sources for factual claims
- Does NOT hallucinate citations to non-existent sources
- Handles multi-turn conversation gracefully (doesn't lose context or waste tokens)
- Explicitly notes when sources contradict
- Admits when it can't answer from the provided context

---

## 📚 Resources

### Papers
- **"Lost in the Middle: How Language Models Use Long Contexts"** (Liu et al., 2023) — The definitive study on positional bias in LLMs. Read this before designing your context structure.
- **"Structured Prompting for Reliable Source Attribution"** — Techniques for getting models to cite correctly.
- **"Adaptive Retrieval-Augmented Generation"** — Dynamic context allocation based on query complexity.

### Production RAG Systems
- **LangChain's Context Compression** — https://python.langchain.com/docs/how_docs/document_transformers
  Study how LangChain implements context compression, but don't copy blindly. You now know the principles to evaluate their approach critically.
- **LlamaIndex's Prompt Helpers** — https://docs.llamaindex.ai/en/stable/module_guides/.../prompt_helper/
  Their `PromptHelper` class handles token budget management. Examine their approach, identify its limitations, and think about how you'd improve it.

### Tools
- **tiktoken** — OpenAI's tokenizer, essential for token counting
- **tokenizers** — Hugging Face's tokenizer library for open-source models
- **Instructor** — Python library for structured outputs (citation extraction)
- **Guardrails AI** — Output validation including citation checking

### Your Own Code
- Revisit your Phase 3 **Knowledge Search Engine** project. The `03-Retrieval-Deep-Dive.md` file covered chunking but assumed a simple concatenation context. Now you know better. Consider:
  - How would you add context integration to the search engine?
  - Where would the token budget decisions be made?
  - How would conversation history work in the API?

### Expert-Level Deep Dives
- **For production engineers**: Study how Bing Chat (now Copilot) structures its context to handle web search results. Their approach to citation formatting and context management is one of the most battle-tested in the industry.
- **For system designers**: The "RAG triad" — retrieval quality, context integration quality, and model capability — are interconnected. Improving context integration can compensate for mediocre retrieval, and vice versa. Understanding these tradeoffs is what separates senior from junior RAG engineers.
- **For critical thinkers**: The best context integration strategy depends on your specific model. GPT-4 handles long contexts differently than Claude, which handles them differently than Llama. Your context pipeline should be *tested and tuned for your specific model*, not copied from a blog post.

---

*"The difference between a good RAG system and a great one isn't the retrieval — it's what you do with what you retrieved. Context is the interface between search and intelligence. Design it carefully."*
