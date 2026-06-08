# 02 — Contextual Retrieval: Giving Chunks Their Context Back

## 🎯 Purpose & Goals

When you chunk a document, each chunk loses its surrounding context. A chunk that reads "The rate limit is 1000 requests per minute" loses the context that this is about the FREE TIER, not all tiers. A chunk that reads "Press Cancel to exit" loses the context that this is part of the PASSWORD RESET flow, not the account deletion flow.

This information loss is permanent and invisible. When your RAG system retrieves a chunk, it has no way of knowing what surrounded it — unless you deliberately preserve that context.

**Contextual retrieval** is a family of techniques that enrich chunks with their surrounding context BEFORE embedding and retrieval. The goal: each chunk carries enough information to be understood in isolation.

**By the end of this module, you will:**

1. Understand the **chunk context problem** — why standard chunking destroys meaning
2. Implement **contextual chunk enrichment** — prepending chunk summaries/context before embedding
3. Build **sentence-window retrieval** — retrieving at sentence granularity but returning paragraph context
4. Implement **auto-merging retrieval** — retrieving small chunks but merging them into larger contexts for the LLM
5. Master **Anthropic's contextual retrieval** approach — the most sophisticated public technique
6. Know the storage, latency, and quality tradeoffs of each approach
7. Know when contextual retrieval is worth the complexity and when it's overkill

---

### ⏹ STOP. Answer these before proceeding.

**Question 1 — The Cutoff Problem**

*You chunk a troubleshooting document at 500-token boundaries. One chunk ends with "If the error persists, the issue may be caused by —" and the next chunk begins "— a corrupted configuration file. To fix this, delete the config and restart."*

*A user asks "What causes the persistent error during startup?" The retrieval finds BOTH chunks (they both contain relevant keywords). But the first chunk's embedding is pulled toward "persistent error" language. The second chunk's embedding is pulled toward "corrupted configuration" language. They might not BOTH be in the top-5 results.*

*If only the first chunk is retrieved, the LLM sees "If the error persists, the issue may be caused by —" — a sentence that's literally cut off. The answer will be speculation. If only the second chunk is retrieved, the LLM sees "— a corrupted configuration file" — a sentence that starts mid-thought.*

*How could you structure your chunks so that EITHER chunk, when retrieved alone, carries enough context to be useful? What metadata would help?*

**Question 2 — The Referential Ambiguity Problem**

*A product comparison document says:*

> *"The Pro tier includes unlimited API access, priority support, and advanced analytics."*
> *..."These features make it ideal for growing businesses."*

*If a user asks "What features make it ideal for growing businesses?" and the retrieval returns the SECOND sentence only — "These features make it ideal for growing businesses" — the chunk is useless. "These features" has no antecedent within the chunk.*

*The embedding for this chunk is about "growing businesses," not about "unlimited API access" or "priority support." If the user's query is about API access, this chunk won't rank highly even though it's directly relevant.*

*How would you design a chunking + retrieval system where the meaning of "these features" is preserved? What if the antecedent is 3 paragraphs earlier?*

**Question 3 — The Granularity Tradeoff**

*Small chunks (100-200 tokens) give you precise retrieval — you find exactly the sentence containing the answer. But the LLM lacks context to interpret it. Large chunks (1000+ tokens) give the LLM full context, but retrieval precision suffers — the embedding is diluted across the larger text.*

*Sentence-window retrieval tries to have both: retrieve at a small granularity (sentence), but return a WINDOW of surrounding text (paragraph or section).*

*What's the optimal window size? If you retrieve a sentence but return 5 sentences before and after, does the extra context help or add noise? How do you determine the right window for YOUR documents? What if different documents need different window sizes?*

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 15 min |
| The Chunk Context Problem — Deep Dive | 20 min |
| Contextual Chunk Enrichment | 30 min |
| Sentence-Window Retrieval | 30 min |
| Auto-Merging Retrieval | 25 min |
| Anthropic's Contextual Retrieval | 25 min |
| Comparison & Tradeoffs | 15 min |
| Code: Enrichment Pipeline | 45 min |
| Code: Window/Merge Systems | 45 min |
| Code: Production Implementation | 30 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~6 hours** |

---

## 📖 Concept Deep-Dive

### 1. The Chunk Context Problem

When you split a document into chunks, you're making a permanent decision about what information lives together. Every chunk boundary is a point where context is lost.

**What you lose at chunk boundaries:**

| Lost Context | Example |
|-------------|---------|
| **Section identity** | "The rate limit is 1000/min" — which tier? |
| **Referential scope** | "These features" — which features? |
| **Narrative flow** | "If it fails, try option B" — what is "it"? |
| **Conditional framing** | "For enterprise customers..." — missing from chunk |
| **Document source** | Which document is this from? How authoritative? |

**The standard approach amplifies this problem:**

```python
# Standard chunking: destroy context
chunks = []
for i in range(0, len(words), chunk_size):
    chunk = " ".join(words[i:i+chunk_size])
    chunks.append(chunk)
    # Every chunk boundary loses:
    # - document title
    # - section heading
    # - preceding context
    # - following context
    # - position in document
```

**The result:** Each chunk's embedding represents not just "what this text says" but also "what this text DOESN'T say (the missing context)." This makes embeddings less discriminative — two chunks from different parts of different documents might have similar embeddings because they're both "missing their context" in similar ways.

---

### 2. Contextual Chunk Enrichment

The simplest solution: **prepend context to each chunk before embedding.**

```python
"""
chunk_enrichment.py — Enrich chunks with surrounding context.
"""

from dataclasses import dataclass, field
from typing import List, Optional


@dataclass
class EnrichedChunk:
    """A chunk with preserved context."""
    chunk_id: str
    content: str  # The original chunk text
    enriched_text: str  # The text that gets embedded (content + context)
    metadata: dict = field(default_factory=dict)

    # Context fields (stored separately for LLM assembly)
    document_title: str = ""
    section_heading: str = ""
    preceding_text: str = ""  # ~200 chars before this chunk
    following_text: str = ""  # ~200 chars after this chunk
    document_summary: str = ""  # One-line summary of the document


class ContextualChunker:
    """
    Enhanced chunking that preserves context.

    Strategy: embed enriched text, but return original chunk text to the LLM.
    This way: retrieval uses the enriched embedding, but generation gets clean text.
    """

    def __init__(
        self,
        chunk_size: int = 500,
        overlap: int = 50,
        context_window: int = 200  # chars of preceding/following context
    ):
        self.chunk_size = chunk_size
        self.overlap = overlap
        self.context_window = context_window

    def chunk_with_context(
        self,
        document: str,
        doc_title: str,
        section_headings: List[str] = None
    ) -> List[EnrichedChunk]:
        """
        Split document into chunks, preserving context for each.

        Each chunk gets:
        1. The chunk text itself
        2. Document title (prepended)
        3. Section heading hierarchy (prepended)
        4. Preceding text (last N chars)
        5. Following text (next N chars)

        The ENRICHED text is what gets embedded.
        The ORIGINAL text is what gets sent to the LLM.
        """
        chunks = self._split_into_chunks(document)
        enriched_chunks = []

        for i, (chunk_text, start_pos, end_pos) in enumerate(chunks):
            # Preceding context (from previous chunk)
            preceding = document[
                max(0, start_pos - self.context_window):start_pos
            ].strip()

            # Following context (from next chunk)
            following = document[
                end_pos:min(len(document), end_pos + self.context_window)
            ].strip()

            # Build enriched text for embedding
            enriched_parts = []

            # Document title provides top-level context
            enriched_parts.append(f"Document: {doc_title}")

            # Section heading provides mid-level context
            if section_headings and i < len(section_headings):
                enriched_parts.append(f"Section: {section_headings[i]}")

            # Preceding context provides flow
            if preceding:
                enriched_parts.append(f"[Previous context: {preceding}]")

            # The actual chunk content
            enriched_parts.append(chunk_text)

            # Following context
            if following:
                enriched_parts.append(f"[Following context: {following}]")

            enriched_text = "\n".join(enriched_parts)

            enriched_chunks.append(EnrichedChunk(
                chunk_id=f"{doc_title}-{i}",
                content=chunk_text,
                enriched_text=enriched_text,
                metadata={
                    "document_title": doc_title,
                    "chunk_index": i,
                    "char_start": start_pos,
                    "char_end": end_pos,
                },
                document_title=doc_title,
                section_heading=section_headings[i] if section_headings else "",
                preceding_text=preceding,
                following_text=following,
            ))

        return enriched_chunks

    def _split_into_chunks(self, text: str) -> List[tuple]:
        """
        Split text into chunks with character positions.

        Returns: [(chunk_text, start_pos, end_pos)]
        """
        words = text.split()
        chunks = []
        start_pos = 0

        for i in range(0, len(words), self.chunk_size - self.overlap):
            chunk_words = words[i:i + self.chunk_size]
            chunk_text = " ".join(chunk_words)

            # Find actual character positions
            # (This is simplified — use proper word→char mapping in production)
            chunk_len = len(chunk_text)
            end_pos = start_pos + chunk_len
            chunks.append((chunk_text, start_pos, end_pos))
            start_pos += len(" ".join(words[i:i+1])) if i > 0 else chunk_len

        return chunks


# ============================================================
# Embedding with Enrichment
# ============================================================

class EnrichedEmbedder:
    """
    Embed chunks using enriched text for retrieval,
    but store original text for generation.
    """

    def __init__(self, embedding_client):
        self.embedding_client = embedding_client

    async def embed_chunks(
        self, enriched_chunks: List[EnrichedChunk]
    ) -> List[dict]:
        """
        Embed each chunk using its enriched text.

        Returns records ready for vector DB insertion.
        """
        texts_to_embed = [c.enriched_text for c in enriched_chunks]

        response = await self.embedding_client.embeddings.create(
            model="text-embedding-3-small",
            input=texts_to_embed
        )

        records = []
        for i, chunk in enumerate(enriched_chunks):
            records.append({
                "id": chunk.chunk_id,
                "embedding": response.data[i].embedding,
                "text": chunk.content,  # Original text for LLM
                "metadata": {
                    "document_title": chunk.document_title,
                    "section_heading": chunk.section_heading,
                    "chunk_index": chunk.metadata.get("chunk_index", 0),
                    "preceding_text": chunk.preceding_text,
                    "following_text": chunk.following_text,
                }
            })

        return records
```

**The key insight:** You embed MORE than you retrieve. The enrichment is ONLY for the embedding step. When the chunk is retrieved and sent to the LLM, you can send the original (clean) text or the enriched text — whichever produces better answers.

---

### 3. Sentence-Window Retrieval

This technique retrieves at a FINE granularity (individual sentences) but returns a WIDER window of context.

```
Retrieval space:                LLM context:

Sentence ───┐                  ┌──────────────────┐
Sentence    │                  │  5 sentences     │
Sentence    ├─► Retrieved ──►  │  BEFORE the      │
Sentence    │   sentence      │  retrieved one    │
Sentence ───┘                  │                   │
                               │  RETRIEVED        │
Sentence                       │  SENTENCE         │
Sentence                       │                   │
Sentence                       │  5 sentences      │
Sentence                       │  AFTER the        │
Sentence                       │  retrieved one     │
                               └──────────────────┘
```

```python
"""
sentence_window.py — Sentence-window retrieval for precise + contextual RAG.
"""

from typing import List, Tuple, Dict
import re


class SentenceWindowRetriever:
    """
    Retrieve at sentence level, return with surrounding window.

    Advantages:
    - Precise retrieval (sentences are specific)
    - Contextual generation (window provides surrounding meaning)
    - No wasted embedding tokens (each sentence has a focused embedding)
    """

    def __init__(
        self,
        embedding_client,
        vector_db,
        window_size: int = 5,  # Sentences before and after
        embedder_model: str = "text-embedding-3-small"
    ):
        self.embedding_client = embedding_client
        self.vector_db = vector_db
        self.window_size = window_size
        self.embedder_model = embedder_model

    def _split_sentences(self, text: str) -> List[Tuple[str, int]]:
        """
        Split text into sentences with indices.

        Returns: [(sentence_text, sentence_index)]
        """
        # Simple sentence splitting. In production, use nltk or spaCy.
        sentence_endings = re.compile(r'(?<=[.!?])\s+(?=[A-Z])')
        sentences = sentence_endings.split(text)

        return [(s.strip(), i) for i, s in enumerate(sentences) if s.strip()]

    async def index_document(self, doc_id: str, text: str, metadata: dict):
        """
        Index a document at sentence level.

        Each sentence gets its own embedding and is stored with
        window information.
        """
        sentences = self._split_sentences(text)

        # Embed all sentences
        texts = [s[0] for s in sentences]
        response = await self.embedding_client.embeddings.create(
            model=self.embedder_model,
            input=texts
        )

        # Store each sentence with window metadata
        points = []
        for i, ((sentence, _), embedding) in enumerate(
            zip(sentences, response.data)
        ):
            # Calculate window boundaries
            window_start = max(0, i - self.window_size)
            window_end = min(len(sentences), i + self.window_size + 1)

            # Assemble the window text (for retrieval display)
            window_text = " ".join(
                sentences[j][0] for j in range(window_start, window_end)
            )

            points.append({
                "id": f"{doc_id}_sent_{i}",
                "embedding": embedding.embedding,
                "text": sentence,
                "window_text": window_text,
                "window_start": window_start,
                "window_end": window_end,
                "metadata": {
                    **metadata,
                    "doc_id": doc_id,
                    "sentence_index": i,
                }
            })

        await self.vector_db.upsert(points)

    async def retrieve(
        self,
        query: str,
        top_k: int = 5
    ) -> List[Dict]:
        """
        Retrieve sentences and return their windows.

        The returned text includes the window context so the LLM
        has enough information to interpret the sentence.
        """
        # Embed query
        response = await self.embedding_client.embeddings.create(
            model=self.embedder_model,
            input=query
        )
        query_emb = response.data[0].embedding

        # Search for sentences
        results = await self.vector_db.search(query_emb, top_k=top_k)

        # Expand each result to include window
        expanded = []
        for result in results:
            expanded.append({
                "id": result["id"],
                "sentence": result["text"],
                "window_text": result["window_text"],
                "metadata": result["metadata"],
                "score": result["score"]
            })

        return expanded

    async def retrieve_and_assemble_context(
        self,
        query: str,
        top_k: int = 5,
        max_tokens: int = 3000
    ) -> str:
        """
        Retrieve sentences and assemble into a context block
        with window expansions.

        De-duplicates overlapping windows.
        """
        results = await self.retrieve(query, top_k)

        # Collect all unique sentences from windows
        all_sentences = set()
        ordered_sentences = []

        for r in results:
            for s in r["window_text"].split(". "):
                s_clean = s.strip() + "." if not s.endswith(".") else s.strip()
                if s_clean not in all_sentences and len(s_clean) > 10:
                    all_sentences.add(s_clean)
                    ordered_sentences.append(s_clean)

        # Build context respecting token budget
        context = ""
        for s in ordered_sentences:
            candidate = context + "\n" + s if context else s
            if len(candidate.split()) <= max_tokens:
                context = candidate
            else:
                break

        return context
```

---

### 4. Auto-Merging Retrieval

Instead of retrieving at the sentence level, this approach indexes SMALL chunks but MERGES them into LARGER chunks before sending to the LLM.

```
Indexing:              Retrieval:            Generation:

┌──────────────────┐   Query ──►             ┌──────────────────┐
│ Small chunk  1   │            │            │  MERGED CONTEXT  │
│ (200 tokens)     │            │            │                  │
├──────────────────┤            ▼            │  Chunks 1-3      │
│ Small chunk  2   │   Returns ──►          │  merged into     │
│ (200 tokens)     │   chunks 2, 3          │  a single block  │
├──────────────────┤                        │                  │
│ Small chunk  3   │                        │  More context    │
│ (200 tokens)     │                        │  than any single │
├──────────────────┤                        │  chunk alone     │
│ Small chunk  4   │                        └──────────────────┘
│ (200 tokens)     │
└──────────────────┘
```

```python
"""
auto_merge.py — Auto-merging retrieval for context-rich RAG.
"""

from typing import List, Dict, Optional
from dataclasses import dataclass


@dataclass
class ChunkNode:
    """A node in the chunk hierarchy."""
    id: str
    text: str
    parent_id: Optional[str] = None
    child_ids: List[str] = None
    start_char: int = 0
    end_char: int = 0

    def __post_init__(self):
        if self.child_ids is None:
            self.child_ids = []


class HierarchicalChunker:
    """
    Creates a hierarchy of chunks at different granularities.

    Level 0: Very fine (100 tokens) — used for retrieval
    Level 1: Fine (500 tokens) — parent of level 0
    Level 2: Medium (2000 tokens) — parent of level 1
    Level 3: Coarse (8000 tokens) — parent of level 2

    Retrieval happens at level 0 (precision).
    Context assembly merges up to level 2 (context).
    """

    def __init__(self, token_counter):
        self.token_counter = token_counter

    def build_hierarchy(self, document: str, doc_id: str) -> Dict:
        """
        Build a hierarchical chunk structure.

        Returns: {level: [ChunkNode, ...]}
        """
        hierarchy = {}

        # Level 0: fine chunks
        level0 = self._chunk_at_level(document, 100, f"{doc_id}_l0")
        hierarchy[0] = level0

        # Level 1: chunks that group level 0 chunks
        level1 = self._merge_chunks(level0, 500, f"{doc_id}_l1")
        hierarchy[1] = level1

        # Level 2: broader grouping
        level2 = self._merge_chunks(level1, 2000, f"{doc_id}_l2")
        hierarchy[2] = level2

        # Build parent/child links
        self._link_hierarchy(hierarchy)

        return hierarchy

    def _chunk_at_level(
        self, text: str, target_tokens: int, prefix: str
    ) -> List[ChunkNode]:
        """Split text into chunks of approximately target_tokens."""
        words = text.split()
        chunks = []
        pos = 0

        for i in range(0, len(words), target_tokens):
            chunk_words = words[i:i + target_tokens]
            chunk_text = " ".join(chunk_words)
            start_char = pos
            end_char = pos + len(chunk_text)
            chunks.append(ChunkNode(
                id=f"{prefix}_{i // target_tokens}",
                text=chunk_text,
                start_char=start_char,
                end_char=end_char
            ))
            pos = end_char + 1  # +1 for space

        return chunks

    def _merge_chunks(
        self, chunks: List[ChunkNode], target_tokens: int, prefix: str
    ) -> List[ChunkNode]:
        """Merge fine chunks into coarser chunks."""
        merged = []
        current_text = ""
        current_start = 0
        merge_idx = 0

        for i, chunk in enumerate(chunks):
            candidate = current_text + " " + chunk.text if current_text else chunk.text
            if self.token_counter(candidate) <= target_tokens:
                current_text = candidate
                if not current_start:
                    current_start = chunk.start_char
            else:
                if current_text:
                    merged.append(ChunkNode(
                        id=f"{prefix}_{merge_idx}",
                        text=current_text,
                        start_char=current_start,
                        end_char=chunk.start_char
                    ))
                    merge_idx += 1
                current_text = chunk.text
                current_start = chunk.start_char

        if current_text:
            merged.append(ChunkNode(
                id=f"{prefix}_{merge_idx}",
                text=current_text,
                start_char=current_start,
                end_char=chunks[-1].end_char
            ))

        return merged

    def _link_hierarchy(self, hierarchy: Dict[int, List[ChunkNode]]):
        """Link parent-child relationships across levels."""
        for level in sorted(hierarchy.keys()):
            if level == 0:
                continue
            children = hierarchy[level - 1]
            parents = hierarchy[level]

            for parent in parents:
                parent.child_ids = []
                for child in children:
                    if (child.start_char >= parent.start_char and
                        child.end_char <= parent.end_char):
                        child.parent_id = parent.id
                        parent.child_ids.append(child.id)


class AutoMergeRetriever:
    """
    Retrieve at fine granularity, merge to coarse for LLM context.

    How it works:
    1. Retrieve top-k fine chunks (level 0)
    2. For each retrieved chunk, find its parent at level 2
    3. Merge unique parent chunks into the final context
    4. The LLM gets broader context than any single fine chunk
    """

    def __init__(
        self,
        embedding_client,
        vector_db,
        hierarchy_store: Dict,
        merge_level: int = 2
    ):
        self.embedding_client = embedding_client
        self.vector_db = vector_db
        self.hierarchy_store = hierarchy_store
        self.merge_level = merge_level

    async def retrieve_and_merge(
        self,
        query: str,
        top_k: int = 5
    ) -> List[Dict]:
        """
        Retrieve fine chunks and merge into coarser context blocks.
        """
        # 1. Embed query
        response = await self.embedding_client.embeddings.create(
            model="text-embedding-3-small",
            input=query
        )
        query_emb = response.data[0].embedding

        # 2. Retrieve fine chunks (level 0)
        fine_results = await self.vector_db.search(
            query_emb, top_k=top_k
        )

        # 3. Find parent contexts
        parent_ids = set()
        for result in fine_results:
            chunk_id = result["id"]
            # Find this chunk's hierarchy
            hierarchy_node = self._find_node(chunk_id)
            if hierarchy_node and hierarchy_node.parent_id:
                parent_ids.add(hierarchy_node.parent_id)

        # 4. Retrieve parent chunks
        merged_contexts = []
        for parent_id in parent_ids:
            parent_node = self._find_node(parent_id)
            if parent_node:
                merged_contexts.append({
                    "id": parent_id,
                    "text": parent_node.text,
                    "source_chunks": [
                        r["id"] for r in fine_results
                        if self._is_child_of(r["id"], parent_id)
                    ]
                })

        return merged_contexts

    def _find_node(self, node_id: str) -> Optional[ChunkNode]:
        """Find a chunk node across all hierarchy levels."""
        for level, nodes in self.hierarchy_store.items():
            for node in nodes:
                if node.id == node_id:
                    return node
        return None

    def _is_child_of(self, child_id: str, parent_id: str) -> bool:
        """Check if a child ID belongs to a parent."""
        child = self._find_node(child_id)
        if child:
            return child.parent_id == parent_id
        return False
```

---

### 5. Anthropic's Contextual Retrieval

In 2024, Anthropic published their approach to contextual retrieval. The key innovation: **use an LLM to generate context for each chunk before embedding.**

```python
"""
anthropic_contextual.py — Anthropic-style contextual chunk enrichment.

Each chunk gets a natural language context snippet generated by an LLM.
This context is prepended to the chunk BEFORE embedding.
"""

from typing import List
from openai import AsyncOpenAI


class AnthropicContextualRetrieval:
    """
    Contextual retrieval using LLM-generated chunk context.

    For each chunk, the LLM generates a ~50-100 word explanation of:
    - What document this chunk is from
    - What section this chunk belongs to
    - How this chunk relates to the surrounding content
    - Any important context needed to understand the chunk in isolation

    This context is prepended to the chunk text before embedding.
    """

    def __init__(
        self,
        llm_client: AsyncOpenAI,
        embedding_client,
        model: str = "gpt-4o-mini",
        embedding_model: str = "text-embedding-3-small",
        context_chars: int = 500  # Max chars for generated context
    ):
        self.llm_client = llm_client
        self.embedding_client = embedding_client
        self.model = model
        self.embedding_model = embedding_model
        self.context_chars = context_chars

    async def generate_chunk_context(
        self,
        chunk_text: str,
        document_text: str,
        document_title: str,
        chunk_start_char: int,
        chunk_end_char: int
    ) -> str:
        """
        Generate context for a chunk using an LLM.

        The LLM sees: the full document, the chunk boundaries,
        and generates a concise context snippet.
        """
        # Extract surrounding context
        preceding = document_text[
            max(0, chunk_start_char - 1000):chunk_start_char
        ]
        following = document_text[
            chunk_end_char:min(len(document_text), chunk_end_char + 1000)
        ]

        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {
                    "role": "system",
                    "content": (
                        "You are a document context generator. "
                        "Given a document and a specific chunk within it, "
                        "generate a concise context snippet that explains:\n"
                        "1. What broader topic or section this chunk belongs to\n"
                        "2. Any important framing needed to understand it\n"
                        "3. How it relates to the content around it\n\n"
                        f"Keep the context under {self.context_chars} characters. "
                        "Write in a neutral, factual style. "
                        "Do NOT repeat the chunk content itself."
                    )
                },
                {
                    "role": "user",
                    "content": (
                        f"Document title: {document_title}\n\n"
                        f"Full document (excerpts):\n"
                        f"[...{preceding}...]\n"
                        f"[CHUNK START]\n{chunk_text}\n[CHUNK END]\n"
                        f"[...{following}...]\n\n"
                        f"Generate context for the chunk between [CHUNK START] "
                        f"and [CHUNK END]."
                    )
                }
            ],
            max_tokens=self.context_chars,
            temperature=0.3
        )

        return response.choices[0].message.content.strip()

    async def process_document(
        self,
        full_text: str,
        doc_title: str,
        chunk_size: int = 500,
        overlap: int = 50
    ) -> List[dict]:
        """
        Full pipeline: chunk → generate context → embed enriched text.

        Returns records ready for vector DB.
        """
        from chunk_enrichment import ContextualChunker

        # Step 1: Create base chunks
        chunker = ContextualChunker(
            chunk_size=chunk_size,
            overlap=overlap
        )
        chunks = chunker.chunk_with_context(full_text, doc_title)

        # Step 2: Generate LLM context for each chunk
        enriched_chunks = []
        for i, chunk in enumerate(chunks):
            print(f"Generating context for chunk {i+1}/{len(chunks)}...")

            context = await self.generate_chunk_context(
                chunk_text=chunk.content,
                document_text=full_text,
                document_title=doc_title,
                chunk_start_char=chunk.metadata.get("char_start", 0),
                chunk_end_char=chunk.metadata.get("char_end", 0),
            )

            # The enriched text = LLM context + original chunk
            enriched_text = f"{context}\n\n{chunk.content}"

            enriched_chunks.append({
                "id": chunk.chunk_id,
                "original_text": chunk.content,
                "enriched_text": enriched_text,
                "context": context,
                "metadata": chunk.metadata,
            })

        # Step 3: Embed enriched text
        enriched_texts = [c["enriched_text"] for c in enriched_chunks]
        response = await self.embedding_client.embeddings.create(
            model=self.embedding_model,
            input=enriched_texts
        )

        # Step 4: Prepare DB records
        records = []
        for i, chunk in enumerate(enriched_chunks):
            records.append({
                "id": chunk["id"],
                "embedding": response.data[i].embedding,
                "text": chunk["original_text"],  # LLM gets original, not enriched
                "context": chunk["context"],
                "metadata": chunk["metadata"],
            })

        return records
```

**Cost consideration:** Generating LLM context for every chunk adds ONE LLM call per chunk during ingestion. For 10,000 chunks with gpt-4o-mini, that's ~$3-5 in one-time cost. The ongoing retrieval cost is unchanged.

**Tradeoff:** The LLM-generated context is higher quality than rule-based context enrichment, but adds ingestion time and cost. For small to medium document sets (< 10,000 chunks), it's usually worth it. For massive document sets, the rule-based approaches scale better.

---

### 6. Which Approach Should You Use?

| Approach | Best For | Cost | Complexity | Context Quality |
|----------|----------|------|------------|-----------------|
| **No enrichment** | Trivial RAG, POC | None | None | Poor — chunks are isolated |
| **Rule-based enrichment** | Production RAG with known structure | Free | Low | Good — document title + section + neighbors |
| **Sentence-window** | Precision-critical RAG (needle-in-haystack) | Free (more storage) | Medium | Very good — fine retrieval + wide context |
| **Auto-merging** | Variable-difficulty queries | Free (more storage) | High | Good — adapts to query needs |
| **LLM contextual** | Quality-critical RAG (best possible) | One-time LLM cost | Medium | Excellent — LLM-generated, most natural |

**The pragmatic choice for most production systems:**

1. **Rule-based enrichment** as the default (cheap, good, easy)
2. +**Sentence-window** for queries that need high precision (identify these from eval)
3. +**LLM contextual** for the most important document sets (pricing, legal, critical docs)

---

## 💻 Code Examples

### Example 1: Production Contextual Retrieval Pipeline

```python
"""
production_contextual.py — Full production pipeline with all techniques.
"""

from typing import List, Optional, Dict
from enum import Enum


class RetrievalStrategy(Enum):
    STANDARD = "standard"
    ENRICHED = "enriched"
    SENTENCE_WINDOW = "sentence_window"
    AUTO_MERGE = "auto_merge"


class ContextualRetrievalPipeline:
    """
    Production pipeline that selects the right contextual retrieval strategy.
    """

    def __init__(self, config: dict):
        self.config = config
        self.strategies = {}

    async def retrieve(
        self,
        query: str,
        strategy: RetrievalStrategy = RetrievalStrategy.ENRICHED,
        top_k: int = 5
    ) -> List[Dict]:
        """
        Retrieve using the specified strategy.

        Returns chunks with enhanced context for the LLM.
        """
        if strategy == RetrievalStrategy.STANDARD:
            chunks = await self._standard_retrieve(query, top_k)
        elif strategy == RetrievalStrategy.ENRICHED:
            chunks = await self._enriched_retrieve(query, top_k)
        elif strategy == RetrievalStrategy.SENTENCE_WINDOW:
            chunks = await self._window_retrieve(query, top_k)
        elif strategy == RetrievalStrategy.AUTO_MERGE:
            chunks = await self._merge_retrieve(query, top_k)
        else:
            chunks = await self._enriched_retrieve(query, top_k)

        # Post-process: assemble context for LLM
        for chunk in chunks:
            chunk["llm_context"] = self._assemble_llm_context(chunk)

        return chunks

    def _assemble_llm_context(self, chunk: Dict) -> str:
        """
        Assemble the final context that goes to the LLM.

        For enriched retrieval: include document context.
        For window retrieval: include the window.
        For standard: just the chunk text.
        """
        text = chunk.get("text", "")
        metadata = chunk.get("metadata", {})

        context_parts = []
        if metadata.get("document_title"):
            context_parts.append(
                f"[Document: {metadata['document_title']}]"
            )
        if metadata.get("section_heading"):
            context_parts.append(
                f"[Section: {metadata['section_heading']}]"
            )
        if chunk.get("context"):  # LLM-generated context
            context_parts.append(
                f"[Context: {chunk['context']}]"
            )

        context_parts.append(text)

        # Add window text if available
        if chunk.get("window_text"):
            return chunk["window_text"]  # Window includes everything

        return "\n".join(context_parts)
```

### Example 2: A/B Test — Does Contextual Retrieval Actually Help?

```python
async def benchmark_contextual_retrieval(
    test_queries: List[str],
    gold_answers: Dict[str, str],
    pipeline: ContextualRetrievalPipeline
) -> Dict:
    """
    Compare standard vs. enriched vs. window retrieval on the SAME queries.

    Measures: Does enrichment improve retrieval quality?
    """
    strategies = [
        RetrievalStrategy.STANDARD,
        RetrievalStrategy.ENRICHED,
        RetrievalStrategy.SENTENCE_WINDOW,
    ]

    results = {}
    for strategy in strategies:
        correct = 0
        total = len(test_queries)

        for query in test_queries:
            chunks = await pipeline.retrieve(query, strategy=strategy, top_k=3)
            context = "\n".join(c["llm_context"] for c in chunks)

            # Check if the gold answer's information is in the retrieved context
            gold_info = gold_answers.get(query, "")
            if gold_info.lower() in context.lower():
                correct += 1

        accuracy = correct / total
        results[strategy.value] = {
            "accuracy": accuracy,
            "correct": correct,
            "total": total
        }

    return results


# Typical result:
# {
#   "standard": {"accuracy": 0.65, ...},
#   "enriched": {"accuracy": 0.78, ...},  # +13% from document context
#   "sentence_window": {"accuracy": 0.82, ...}  # +17% from window context
# }
```

---

### Example 3: Context-Aware Chunk Assembly for the LLM

Once chunks are retrieved with their context, how do you present them to the LLM?

```python
def assemble_context_with_enrichment(
    chunks: List[Dict],
    max_tokens: int = 3000,
    include_document_context: bool = True
) -> str:
    """
    Assemble retrieved chunks into an LLM context block.

    Each chunk includes its document/section context so the LLM
    knows WHERE the information came from.
    """
    context_parts = []
    total_tokens = 0
    approx_tokens = lambda t: len(t.split()) * 1.3  # Rough estimate

    for chunk in chunks:
        # Build source header
        source = chunk.get("metadata", {})
        header_parts = []

        if include_document_context and source.get("document_title"):
            header_parts.append(f"[{source['document_title']}]")

        if include_document_context and source.get("section_heading"):
            header_parts.append(f"[{source['section_heading']}]")

        header = " ".join(header_parts) if header_parts else f"[Chunk {chunk.get('id', '?')}]"

        # Content = header + chunk text (with optional LLM context)
        content = chunk.get("text", "")
        if chunk.get("context"):  # LLM-generated context
            content = f"{chunk['context']}\n{content}"

        entry = f"{header}\n{content}"

        entry_tokens = approx_tokens(entry)
        if total_tokens + entry_tokens <= max_tokens:
            context_parts.append(entry)
            total_tokens += entry_tokens
        else:
            break

    return "\n\n---\n\n".join(context_parts)
```

---

## ✅ Good Output Examples

### Without Contextual Enrichment

```
Chunk retrieved:
  "The rate limit is 1000 requests per minute."

LLM context:
  [Chunk 42]
  The rate limit is 1000 requests per minute.

LLM sees: A rate limit number. No context about WHICH tier, WHICH API, or UNDER WHAT CONDITIONS.
```

### With Contextual Enrichment

```
Chunk retrieved:
  "Document: API Reference v3.2
   Section: Rate Limits - Free Tier
   The rate limit is 1000 requests per minute."

Or with LLM-generated context:
  "This chunk describes the API rate limit for the free tier of the API Reference v3.2.
   It is part of the Rate Limits section and specifies the maximum requests per minute
   for free-tier users.
   The rate limit is 1000 requests per minute."

LLM sees: The rate limit applies to the FREE TIER of the API Reference v3.2.
          This is a limit, not a recommendation.
```

### Sentence-Window in Action

```
User query: "What features make it ideal for businesses?"

Retrieved sentence: "These features make it ideal for growing businesses."

Window context (5 sentences before):
  "The Pro tier includes unlimited API access, priority support,
   and advanced analytics. The Pro tier is designed for teams that
   need more throughput. It includes 99.9% uptime SLA.
   Setup takes under 5 minutes with our guided wizard.
   These features make it ideal for growing businesses."

LLM sees: The full context — "these features" = unlimited API, priority support,
          advanced analytics, 99.9% SLA, quick setup.
```

---

## ❌ Antipatterns & Failure Modes

### 1. Embedding Enriched Text but Returning Raw Text

**What:** You embed the enriched text (with context prepended) but store and return the raw chunk text to the LLM.

**Why it might fail:** The retrieval finds the chunk because the enriched embedding was close to the query. But the LLM receives the RAW text without the context — and can't understand it.

**Fix:** Return the enriched text (or at least the context snippet) to the LLM, not the raw chunk.

### 2. Context Overload

**What:** Prepending so much context that the chunk's embedding is dominated by the context, not the content.

```python
# BAD: 500 chars of context, 100 chars of actual chunk
enriched = "Document: X, Section: Y, Preceding: ..., Following: ... [THE ACTUAL CHUNK]"
# The embedding is about the CONTEXT, not the CHUNK
```

**Fix:** Keep context proportional. Rule: context ≤ content. If your chunk is 200 tokens, context shouldn't exceed 200 tokens.

### 3. Window Blindness

**What:** In sentence-window retrieval, the window is fixed size regardless of document structure.

**Problem:** A 5-sentence window might cross section boundaries, pulling in irrelevant context from a different section.

**Fix:** Make windows structure-aware. Don't cross section headings. If a section break is within the window, stop at the break.

### 4. LLM Context Generation Hallucination

**What:** The LLM generates context that's wrong or misleading.

```text
Chunk: "The rate was increased to 5000."
LLM context: "This chunk describes the rate limit for the enterprise tier..."

But the ACTUAL document section is about the PRO tier.
The LLM hallucinated "enterprise" based on plausible guessing.
```

**Fix:** Show the LLM the full document and chunk boundaries. Use low temperature (0.3). For critical documents, use rule-based enrichment instead of LLM-generated context.

### 5. Ingestion-Time Only Optimization

**What:** Treating contextual retrieval as a one-time ingestion optimization.

**Problem:** When documents are updated, the chunk context needs to be regenerated. If you add a new document or change a section heading, the context chunks referencing that section become stale.

**Fix:** Make contextual enrichment part of your ingestion pipeline, not a one-time script. Re-generate context whenever the source document changes.

---

## 🧪 Drills & Challenges

### Drill 1: Build the Context Enricher (35 min)

Take a document and implement three enrichment strategies:

1. **Rule-based**: Prepend document title, section heading, and 100 chars of preceding/following text.
2. **LLM-based**: Use an LLM to generate a 50-word context snippet.
3. **Hybrid**: Rule-based enrichment, but let the LLM refine it.

```python
def test_enrichment_strategies(document, queries):
    """
    For each strategy:
    1. Enrich and embed all chunks
    2. Retrieve for each query
    3. Measure: did retrieval find the right chunks?

    Report: which strategy works best for WHICH types of queries?
    """
    pass
```

**Test on these edge cases:**
- A chunk that starts mid-sentence
- A chunk containing only a table or code block
- A chunk with heavy pronoun usage ("it", "they", "this")
- A chunk from deep inside a long document (no nearby section headings)

### Drill 2: Sentence-Window Size Experiment (40 min)

Build an experiment to find the optimal window size for YOUR documents.

```python
async def find_optimal_window_size(
    queries_with_targets: List[tuple],
    retriever_builder,
    window_sizes: List[int] = [0, 1, 3, 5, 10, 20]
) -> Dict:
    """
    For each window size:
    1. Build a sentence-window retriever with that window
    2. Run all test queries
    3. Measure: precision (is the answer in the window?) and noise
       (how much irrelevant text is included?)

    Report: the window size that maximizes precision while minimizing noise.
    """
    pass
```

**Key question:** At what window size does adding more sentences hurt precision more than it helps recall?

### Drill 3: The Parent-Child Merge (30 min)

Implement auto-merging retrieval manually:

1. Create chunks at 3 levels: small (200t), medium (800t), large (3000t)
2. Embed and index only the small chunks
3. When a small chunk is retrieved, find its medium parent
4. Return the medium parent to the LLM

```python
def build_hierarchy(text: str):
    """Create 3-level chunk hierarchy."""
    pass

def retrieve_and_merge(query: str, hierarchy, top_k: int):
    """Retrieve small, merge to medium, return."""
    pass
```

**Compare:** Single-level (200t) vs. auto-merge (200t retrieve, 800t return). Does the extra context improve answer quality? For which queries?

### Drill 4: The Ambiguity Detector (25 min)

Build a function that detects when a chunk is ambiguous without context:

```python
def detect_ambiguous_chunks(chunks: List[str]) -> List[tuple]:
    """
    Identify chunks that would be ambiguous without context.

    Signals:
    - Starts with pronoun ("It", "This", "They")
    - Starts with conjunction ("However", "Additionally")
    - Contains unresolved references ("these features", "the above")
    - Is very short (< 50 chars)
    - Contains cutoff text (starts/ends mid-sentence)

    Returns: [(chunk_index, chunk_text, ambiguity_reason)]
    """
    pass
```

This is a diagnostic tool — it tells you WHICH chunks need contextual enrichment most urgently.

### Drill 5: Build the Context-Aware Indexer (45 min)

Build a complete indexer that enriches chunks and stores metadata for contextual retrieval.

```python
class ContextAwareIndexer:
    """
    Full ingestion pipeline with contextual enrichment.

    1. Load document
    2. Extract structure (headings, sections)
    3. Chunk with context
    4. Generate LLM context (optional, per config)
    5. Embed enriched text
    6. Store: original text + context + embedding + metadata
    """

    async def index_document(self, doc_path: str):
        """Index a single document with full context."""
        pass

    async def index_directory(self, dir_path: str):
        """Index all documents in a directory."""
        pass

    def get_statistics(self) -> Dict:
        """Return indexing stats: chunks, avg context length, etc."""
        pass
```

### Drill 6: The Context Verification Audit (30 min)

After implementing contextual retrieval, audit whether it actually helps:

```python
async def audit_context_quality(
    pipeline,
    test_queries: List[str],
    n_samples: int = 50
) -> Dict:
    """
    For each query, compare:
    - Standard retrieval context quality
    - Enriched retrieval context quality

    Measure:
    - Does enrichment include the RIGHT context (helpful)?
    - Does enrichment include WRONG context (misleading)?
    - Does enrichment add USEFUL context that wasn't in the chunk?

    Report: enrichment helpfulness rate, hallucination rate, coverage improvement.
    """
    pass
```

---

## 🚦 Gate Check

Before proceeding to Phase 5.03 (Reranking), you must demonstrate:

1. **Implement contextual chunk enrichment**: Build at least one enrichment strategy (rule-based or LLM-generated) and show that it improves retrieval for 5 test queries where standard chunking fails.

2. **Build sentence-window retrieval**: Implement sentence-level retrieval with configurable window size. Find the optimal window size for your documents.

3. **Show a case where enrichment hurts**: Find a query where adding context makes retrieval WORSE (e.g., the context embedding dominates and pulls the chunk away from relevant queries).

4. **Measure the impact**: On 20 test queries, compare: standard retrieval accuracy vs. enriched retrieval accuracy. Show the numbers.

5. **Answer the reflection questions**:
   - When would you use rule-based enrichment vs. LLM-generated context?
   - How do you choose the window size for sentence-window retrieval?
   - What proportion of your chunks contain ambiguous references that need context?
   - How do you verify that the enrichment is accurate?

**Passing criteria:** You know that a chunk's embedding is only as good as the context it carries. You can diagnose "embedding dilution" — when a chunk's embedding is pulled away from relevant queries because it lacks context — and fix it with enrichment.

---

## 📚 Resources

### Papers & Techniques
- **Anthropic's Contextual Retrieval** (2024) — Their blog post introduced the LLM-generated context approach. The most practical treatment of this topic.
- **"Sentence-Window Retrieval for RAG"** — Community techniques from LlamaIndex and LangChain implementations.
- **"Lost in the Middle" (Liu et al., 2023)** — Revisited: context enrichment is one mitigation for this problem.

### Framework Implementations
- **LlamaIndex's SentenceWindowNodeParser** — Reference implementation of sentence-window retrieval.
- **LangChain's ParentDocumentRetriever** — Auto-merging retrieval implementation.
- **Anthropic's cookbook** — Code examples for contextual retrieval.

### Production Patterns
- **Glean's approach to context** — How a production search engine handles chunk context at scale.
- **Notion AI's chunk enrichment** — Their approach to preserving document structure in retrieval.

### Your Own Code
- Your Phase 4 **Customer Support Bot** can be immediately improved with contextual chunk enrichment. The next time you re-index, use enriched embeddings.
- The **sentence-window approach** composes naturally with **reranking** (next module) — retrieve sentences, rerank, then expand to windows.

### Expert-Level Deep Dives
- **For production engineers**: The highest-leverage contextual retrieval improvement is DOCUMENT TITLE. Many vector DB indexes don't include the document title in the chunk embedding. Simply prepending "[Document: X]" to every chunk before embedding can improve retrieval by 5-15% — and it costs nothing.
- **For system designers**: Contextual enrichment and query transformation (HyDE) solve the same problem from opposite sides. HyDE transforms the query to match the documents. Contextual enrichment transforms the documents to match the query. Use BOTH for maximum coverage — they address different failure modes and compose well.
- **For critical thinkers**: The best context for a chunk is not "more context" — it's the RIGHT context. A chunk from a pricing page needs to know WHICH TIER. A chunk from a troubleshooting guide needs to know WHICH ERROR. Generic enrichment (document title + section + neighbors) helps, but LLM-generated context that identifies the SPECIFIC relevance is significantly better — at a cost.

---

*"Every chunk boundary is a point where understanding is lost. Contextual retrieval doesn't eliminate the boundary — it builds a bridge across it."*
