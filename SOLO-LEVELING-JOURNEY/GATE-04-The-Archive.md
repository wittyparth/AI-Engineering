# ⛩️ GATE 04 — The Archive (RAG Foundations)

**Rank Requirement:** C-
**Theme:** Retrieval-Augmented Generation — connecting LLMs to external knowledge
**Total Days:** 7 Dungeons + 1 Boss Battle
**Core Skills:** Retrieval pipelines, context injection, source citation, RAG evaluation

---

## Dungeon 1 — The RAG Triad

**Day 1 | Concept:** RAG has three components: Ingestion → Retrieval → Generation. Each leg must be strong.

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│ Ingestion │ → │ Retrieval │ → │Generation │
│ (Chunk +  │    │ (Embed +  │    │ (Context + │
│  Store)   │    │  Search)  │    │  Prompt)   │
└──────────┘    └──────────┘    └──────────┘
```

**Ingestion Failure:** Garbage chunks → garbage retrieval.
**Retrieval Failure:** Wrong context → hallucinated answer.
**Generation Failure:** Bad prompt → bad answer even with right context.

**🗡️ Dungeon Mastery:** Diagram your own RAG pipeline. For each leg, write down 2 things that could go wrong.

**💻 Code — Minimal RAG Pipeline:**
```python
from dataclasses import dataclass, field
from typing import List

@dataclass
class RAGPipeline:
    """The simplest possible RAG system."""
    vector_db: object
    llm: object
    top_k: int = 3

    def retrieve(self, query: str) -> List[str]:
        results = self.vector_db.query(query_texts=[query], n_results=self.top_k)
        return results['documents'][0]

    def generate(self, query: str, contexts: List[str]) -> str:
        context = "\n\n".join(contexts)
        prompt = f"""Use the following context to answer the question.

Context:
{context}

Question: {query}

Answer (cite sources):"""
        return self.llm.generate(prompt)

    def query(self, query: str) -> dict:
        contexts = self.retrieve(query)
        answer = self.generate(query, contexts)
        return {"answer": answer, "sources": contexts}

# Usage
# rag = RAGPipeline(vector_db=chroma_collection, llm=my_llm)
# result = rag.query("What is chunking?")
# print(result["answer"])
```

**👹 Boss Lesson:** A RAG pipeline is only as strong as its weakest leg. Most people over-optimize generation and neglect ingestion.

---

## Dungeon 2 — Document Ingestion Pipeline

**Day 2 | Concept:** Production ingestion is not "load file → chunk → embed." It's a structured pipeline: load → split → clean → enrich → chunk → embed → store.

**Ingestion Steps:**
1. **Loader:** Read PDF, HTML, markdown, code, database
2. **Splitter:** Split by document structure (pages, sections, tables)
3. **Cleaner:** Remove noise (boilerplate, headers, footers)
4. **Enricher:** Add metadata (source URL, date, section hierarchy)
5. **Chunker:** Apply chunking strategy
6. **Embedder:** Convert chunks to vectors
7. **Indexer:** Store in vector DB with metadata

**🗡️ Dungeon Mastery:** Build an ingestion pipeline that loads a markdown file, splits by headings, adds filename as metadata, chunks by paragraph, embeds with MiniLM, and stores in ChromaDB.

**💻 Code — Ingestion Pipeline:**
```python
import hashlib
from pathlib import Path
from typing import Iterator

class IngestionPipeline:
    def __init__(self, chunker, embedder, vector_db):
        self.chunker = chunker
        self.embedder = embedder
        self.vector_db = vector_db

    def process_file(self, filepath: Path):
        """Process a single file through the pipeline."""
        # 1. Load
        text = filepath.read_text(encoding="utf-8")
        source = filepath.name

        # 2. Clean (remove excessive blank lines)
        import re
        text = re.sub(r'\n{3,}', '\n\n', text)

        # 3. Chunk
        chunks = self.chunker(text)

        # 4. Embed + Store
        ids = []
        documents = []
        metadatas = []
        for i, chunk in enumerate(chunks):
            chunk_id = hashlib.md5(f"{source}:{i}:{chunk[:50]}".encode()).hexdigest()
            ids.append(chunk_id)
            documents.append(chunk)
            metadatas.append({"source": source, "chunk_index": i, "char_count": len(chunk)})

        self.vector_db.add(
            documents=documents,
            ids=ids,
            metadatas=metadatas,
        )
        return len(chunks)

    def process_directory(self, dirpath: Path, pattern: str = "*.md"):
        files = list(dirpath.glob(pattern))
        total = 0
        for f in files:
            count = self.process_file(f)
            print(f"  {f.name}: {count} chunks")
            total += count
        return total
```

**👹 Boss Lesson:** Ingestion is 80% of RAG quality. A well-ingested document is already half-retrieved.

---

## Dungeon 3 — Context Injection Strategies

**Day 3 | Concept:** How you inject retrieved context into the prompt dramatically affects answer quality.

**Strategies (ranked by quality):**
| Strategy | Format | Quality |
|----------|--------|---------|
| Concatenation | `Context: {docs}\n\nQ: {q}` | Baseline |
| Numbered | `[1] {doc1}\n[2] {doc2}\n\nQ: {q}` | Better — model can cite |
| Reranked | Best docs first, worst last | Good — priority ordering |
| Windowed | Chunk + neighbors for more context | Better — full picture |
| Summary | LLM-summarized context | Best — but expensive |

**🗡️ Dungeon Mastery:** Take the same 5 retrieved docs. Inject them using all 5 strategies. Query "What is a vector database?" Compare which strategy gives the best answer.

**💻 Code — Context Injection:**
```python
def inject_concatenation(docs: list[str], query: str) -> str:
    context = "\n\n".join(docs)
    return f"""Answer the question using the provided context.

Context:
{context}

Question: {query}

Answer:"""

def inject_numbered(docs: list[str], query: str) -> str:
    numbered = "\n\n".join(f"[{i+1}] {doc}" for i, doc in enumerate(docs))
    return f"""Answer the question. Cite sources using [1], [2], etc.

Sources:
{numbered}

Question: {query}

Answer (with citations):"""

def inject_windowed(docs: list[dict], query: str) -> str:
    """docs is list of {text, prev_chunk, next_chunk}"""
    contexts = []
    for d in docs:
        parts = []
        if d.get("prev_chunk"):
            parts.append(f"[Previous section]: {d['prev_chunk']}")
        parts.append(f"[Main section]: {d['text']}")
        if d.get("next_chunk"):
            parts.append(f"[Next section]: {d['next_chunk']}")
        contexts.append("\n".join(parts))

    return inject_numbered(contexts, query)
```

**👹 Boss Lesson:** Numbered citation format is the sweet spot — it's simple, the model understands it, and it enables source tracing.

---

## Dungeon 4 — Source Citation & Attribution

**Day 4 | Concept:** RAG without citations is just fancy hallucination. Every claim must trace back to a source.

**Citation Formats:**
```
Simple: [1], [2, 3]
Rich: [Source: report.pdf, Page 5, Section 3.2]
Inline: According to [1], vector databases...
```

**🗡️ Dungeon Mastery:** Build a citation tracker. Given a response with [1], [2] markers, extract which sources were used and validate they exist.

**💻 Code — Citation System:**
```python
import re
from dataclasses import dataclass

@dataclass
class Citation:
    source: str
    chunk_index: int
    relevance_score: float

class CitationTracker:
    def __init__(self):
        self.sources: dict[str, Citation] = {}

    def register_source(self, doc_id: str, citation: Citation):
        self.sources[doc_id] = citation

    def extract_citations(self, response: str) -> list[str]:
        """Extract [N] citation markers from response."""
        return re.findall(r'\[(\d+)\]', response)

    def validate_response(self, response: str) -> dict:
        """Check if all citations in response reference real sources."""
        cited = self.extract_citations(response)
        valid = [c for c in cited if c in self.sources]
        invalid = [c for c in cited if c not in self.sources]

        return {
            "citation_count": len(cited),
            "valid_citations": valid,
            "invalid_citations": invalid,
            "all_valid": len(invalid) == 0,
            "source_coverage": f"{len(valid)}/{len(self.sources)} sources used",
        }

# In the generation prompt:
CITATION_SYSTEM_PROMPT = """You are a helpful assistant that answers using provided sources.
IMPORTANT RULES:
- Every factual claim MUST be followed by a citation like [1], [2]
- If no source supports a claim, say "I don't have information about that"
- Never make up citations
- Format: Claim [SourceNumber]"""
```

**👹 Boss Lesson:** A RAG system without citations is a hallucination engine with extra steps. Always, always cite.

---

## Dungeon 5 — Query Transformation

**Day 5 | Concept:** Users don't write good retrieval queries. Transform their question into something the vector DB can actually match.

**Query Transformations:**
| Technique | Input | Output |
|-----------|-------|--------|
| Query Rewrite | "Tell me about it" | "Explain how vector databases perform similarity search" |
| HyDE | "What's RAG?" | "RAG is a technique that combines retrieval..." (hypothetical doc) |
| Multi-query | "Tell me about vectors" | ["What are vector embeddings?", "How does vector search work?", ...] |
| Step-back | "How do I fix this error?" | "What are common FastAPI error handling patterns?" |

**🗡️ Dungeon Mastery:** Take a vague query. Apply all 4 transformations. Retrieve with each. Which transformation got the best results?

**💻 Code — Query Transformer:**
```python
class QueryTransformer:
    def __init__(self, llm):
        self.llm = llm

    def rewrite(self, query: str) -> str:
        prompt = f"""Rewrite this vague question into a detailed, specific question suitable for searching documentation.

Original: {query}

Rewritten:"""
        return self.llm.generate(prompt)

    def hyde(self, query: str) -> str:
        """Hypothetical Document Embeddings — generate a pseudo-document."""
        prompt = f"""Write a short paragraph that would be the perfect answer to this question.
Make it factual and detailed.

Question: {query}

Hypothetical answer:"""
        return self.llm.generate(prompt)

    def multi_query(self, query: str, n: int = 3) -> list[str]:
        prompt = f"""Generate {n} different specific questions that would help answer this broad question.

Original: {query}

Questions (one per line):"""
        result = self.llm.generate(prompt)
        return [q.strip() for q in result.split("\n") if q.strip() and q.strip().endswith("?")]

    def step_back(self, query: str) -> str:
        prompt = f"""Step back from this specific question to ask a more general one.

Specific: {query}
General principle:"""
        return self.llm.generate(prompt)
```

**👹 Boss Lesson:** Users are lazy query-writers. Transform their vague inputs into precise retrievable queries. Your vector DB will thank you.

---

## Dungeon 6 — RAG Evaluation

**Day 6 | Concept:** How do you know if your RAG system is good? Measure retrieval precision, context relevance, and answer faithfulness.

**RAG Evaluation Metrics:**
| Metric | What it Measures | How |
|--------|-----------------|-----|
| **Hit Rate** | Did we retrieve the right doc? | % of queries where gold doc is in top-k |
| **MRR** | How high was the gold doc ranked? | Mean Reciprocal Rank |
| **Faithfulness** | Does answer stay true to context? | NLI model checks if answer is entailed |
| **Answer Relevance** | Does answer address the query? | Embedding similarity between query and answer |
| **Context Precision** | How much retrieved context is useful? | % of retrieved docs that were relevant |

**🗡️ Dungeon Mastery:** Create a test set of 5 queries with known correct answers. Run through your RAG pipeline. Compute Hit Rate@3, MRR, and Faithfulness.

**💻 Code — RAG Evaluation:**
```python
import numpy as np
from datasets import Dataset
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

class RAGEvaluator:
    """Evaluate RAG pipeline quality."""

    @staticmethod
    def hit_rate(retrieved_ids: list[str], relevant_ids: list[str], k: int = 3) -> float:
        """Did any relevant doc appear in top-k?"""
        top_k = set(retrieved_ids[:k])
        return 1.0 if any(rid in top_k for rid in relevant_ids) else 0.0

    @staticmethod
    def mrr(retrieved_ids: list[str], relevant_ids: list[str]) -> float:
        """Mean Reciprocal Rank of first relevant doc."""
        for rank, doc_id in enumerate(retrieved_ids, 1):
            if doc_id in relevant_ids:
                return 1.0 / rank
        return 0.0

    @staticmethod
    def precision_at_k(retrieved_ids: list[str], relevant_ids: list[str], k: int = 3) -> float:
        top_k = set(retrieved_ids[:k])
        if not top_k:
            return 0.0
        return len(top_k & set(relevant_ids)) / len(top_k)

    def evaluate_retrieval(self, queries: list[dict], retriever, k: int = 3) -> dict:
        """queries: [{query, relevant_ids}]"""
        hit_rates = []
        reciprocal_ranks = []
        precisions = []

        for q in queries:
            results = retriever.query(q["query"], top_k=k)
            retrieved_ids = [r["id"] for r in results]

            hit_rates.append(self.hit_rate(retrieved_ids, q["relevant_ids"], k))
            reciprocal_ranks.append(self.mrr(retrieved_ids, q["relevant_ids"]))
            precisions.append(self.precision_at_k(retrieved_ids, q["relevant_ids"], k))

        return {
            "hit_rate": np.mean(hit_rates),
            "mrr": np.mean(reciprocal_ranks),
            "precision": np.mean(precisions),
        }
```

**👹 Boss Lesson:** What gets measured gets improved. If you're not evaluating your RAG pipeline, you're guessing.

---

## Dungeon 7 — Advanced Retrieval Techniques

**Day 7 | Concept:** Basic RAG is just the start. Advanced techniques dramatically improve retrieval quality.

**Techniques:**
1. **Reranking:** Retrieve 20 docs, rerank with cross-encoder, keep top-3
2. **FlashRank:** Ultra-fast reranking for production
3. **Contextual Retrieval:** Include chunk's surrounding context automatically
4. **Late Interaction** (ColBERT): Token-level matching for higher precision

**🗡️ Dungeon Mastery:** Implement a reranker. Retrieve 10 docs with embedding search, then rerank with a cross-encoder. Compare results before and after reranking.

**💻 Code — Reranking:**
```python
from sentence_transformers import CrossEncoder

class Reranker:
    def __init__(self, model_name: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"):
        self.model = CrossEncoder(model_name)

    def rerank(self, query: str, documents: list[str], top_k: int = 3) -> list[dict]:
        """Rerank documents by relevance to query."""
        pairs = [(query, doc) for doc in documents]
        scores = self.model.predict(pairs)

        # Sort by score descending
        scored = [(doc, float(score)) for doc, score in zip(documents, scores)]
        scored.sort(key=lambda x: x[1], reverse=True)

        return [
            {"document": doc, "score": score}
            for doc, score in scored[:top_k]
        ]

# Full retrieval pipeline with reranking:
class AdvancedRetriever:
    def __init__(self, vector_db, embedder, reranker, top_k_retrieve=20, top_k_rerank=3):
        self.vector_db = vector_db
        self.embedder = embedder
        self.reranker = reranker
        self.top_k_retrieve = top_k_retrieve
        self.top_k_rerank = top_k_rerank

    def retrieve(self, query: str) -> list[dict]:
        # Step 1: Fast embedding search
        results = self.vector_db.query(
            query_texts=[query],
            n_results=self.top_k_retrieve,
        )
        docs = results['documents'][0]

        # Step 2: Expensive but accurate reranking
        reranked = self.reranker.rerank(query, docs, self.top_k_rerank)
        return reranked
```

**👹 Boss Lesson:** Retrieval is a funnel. Cast wide with cheap embedding search, filter precise with expensive reranking. This is the production pattern.

---

## ⚔️ BOSS BATTLE: Customer Support Bot

**Objective:** Build a complete RAG system for a fictional product's documentation:
1. Create 10+ FAQ/support documents
2. Ingest them through a proper pipeline (chunk → embed → index)
3. Build a retrieval pipeline with query rewriting + reranking
4. Generate answers with numbered citations
5. Evaluate with Hit Rate, MRR, and Faithfulness metrics

**🗡️ Victory Conditions:**
- Ingestion pipeline handles all 10+ documents
- Retrieval finds correct docs for 5 test queries (Hit Rate@3 ≥ 0.8)
- Answers include citations
- Evaluation metrics are computed and logged
- System handles vague queries via transformation

**🏆 Rewards:**
- +500 XP
- C-Rank → C+ Rank
- Skill Unlock: **RAG Architect** — All RAG system designs get a structural bonus
- Title: **The Archivist**

---

*"Knowledge without retrieval is just memorization. Retrieval without generation is just search. Together, they are wisdom."*
