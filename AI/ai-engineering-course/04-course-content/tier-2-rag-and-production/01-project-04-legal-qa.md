# Project 4: Legal Document Q&A System

- **Tier:** 2 — First Real Clients
- **Project #:** 4 of 7 (first of Tier 2)
- **Tech Stack:** Python, PyMuPDF (pdfplumber), sentence-transformers + BM25 (or Qdrant hybrid), FastAPI, basic RAGAS/DeepEval evaluation
- **Concepts:** RAG from scratch, PDF parsing, chunking strategies, hybrid search (dense + sparse), citation rendering, retrieval evaluation
- **Quality Gate:** ✅ APPROVED when every answer includes the source document, page number, and excerpt — AND you can prove retrieval precision > 80% on a held-out test set of 50 queries

---

## Phase 1: Brief (Priya)

> *Priya walks in with actual printed documents. Like, paper. She drops a 200-page contract on your desk with a thud.*

"Meet our new client: **Reyes & Associates**, a mid-sized law firm with 50 lawyers. They handle corporate law — contracts, mergers, regulatory compliance. Lots of paper. Lots of PDFs.

**Their problem:** Junior lawyers spend **4 hours a day** searching for precedents. A senior partner asks: 'What were the indemnification terms in the Acme Corp acquisition?' And a junior lawyer spends half a day digging through 500+ PDFs to find the answer.

**What we're building:** A system where a lawyer can type a question in plain English — *'what are the indemnification clauses in our 2023 vendor contracts'* — and get an answer with the exact source document, page number, and relevant excerpt.

**Non-negotiable:**
- Every answer MUST cite its source — document name, page number, exact passage
- Zero tolerance for hallucinated citations. If the answer isn't in the documents, the system says 'I don't know'
- Must handle PDFs (they're all PDFs — contracts, case law, regulatory filings)
- Answers in under 10 seconds

**Deadline:** 5 days. The partner has a board demo on Friday.

**Definition of done (from the client's perspective):** A lawyer can get a correct, cited answer to any question about their documents in under 10 seconds. No hallucinated citations."

---

## Phase 2: Learning Path (Maya)

> *Maya meets you at the whiteboard. She draws a diagram: `[PDF] → [Chunks] → [Vectors + Keywords] → [Search] → [LLM + Context] → [Answer + Citation]`.*

"This is the big one. **RAG — Retrieval-Augmented Generation.** 70% of production AI systems use this pattern. If you learn nothing else from this course, learn this.

But here's the deal: **I don't want you touching LangChain for this project.** I want you to feel every piece of the pipeline. Every chunk boundary. Every similarity score. Every citation format decision. Because when you use LangChain later, I need you to know what it's doing under the hood.

### Learning Order (Scratch-First)

**Step 1: PDF extraction.**
You need to get text out of PDFs. PyMuPDF (`fitz`) or `pdfplumber`. Extract page by page. See the mess — PDFs aren't clean text. There are headers, footers, page numbers, tables, weird encodings.

> *"You'll discover that extracting clean text from PDFs is 30% of the work. This is not AI. This is data engineering. And it's what makes RAG hard."*

**Step 2: Chunking with edge cases.**
Split extracted text into chunks. This time it matters — lawyers need exact citations. A chunk strategy that cuts a sentence in half means the citation is wrong.

Try three strategies and compare:
1. Fixed-size with overlap (what you did in Project 3)
2. Recursive splitting (sentences → paragraphs → token limits)
3. Semantic chunking (split at topic boundaries)

> *"Research reality (2026): Chunking strategy is the #1 quality lever in RAG. The most consistent performer is 512-token semantic chunks with 10-15% overlap. But strategy varies BY DOCUMENT TYPE — contracts need different chunking than case law."*

**Step 3: Build the retriever.**
For each chunk, generate:
- A **dense vector** (embedding — same as Project 3)
- A **sparse vector** (BM25 — keyword importance scores)

Store them together. Search by combining both. This is **hybrid search** — the production standard.

> *"Hybrid search outperforms vector-only in EVERY measured deployment. Semantic embeddings understand meaning. BM25 catches exact phrases — like 'Section 14.3 Indemnification.' You need both."*

**Step 4: Generation with citations.**
The RAG part: pass retrieved chunks to the LLM with a prompt that says "answer based ONLY on this context. Cite the source document, page, and exact passage for every claim you make."

> *"This is where you learn that prompt engineering for RAG is different from normal prompt engineering. You're not asking the model to be creative. You're asking it to be faithful — to stick strictly to the provided context."*

**Step 5: Evaluation.**
Build a test set of 50 queries with known correct answers. Measure:
- **Retrieval precision:** % of retrieved chunks that are actually relevant to the query
- **Recall@k:** % of all relevant chunks that were retrieved in the top K
- **Faithfulness:** Does the answer stick to the retrieved context? (Use LLM-as-judge)
- **Citation accuracy:** Does every citation actually say what the answer claims?

> *"Rohan won't APPROVE this project without eval numbers. RAGAS and DeepEval can compute these automatically. But you should also understand what each metric means — because they're all flawed and you need to know when they lie."*

### Memory Triage

**Memorize cold:**
- The RAG pipeline structure: chunk → embed → store → retrieve → generate
- Qdrant upsert and query pattern (or your vector DB of choice)
- Hybrid query API: `prefetch` (dense + sparse) → `fusion` (RRF)
- The RAG generation prompt pattern: *"Answer based only on this context. If you cannot answer from the context, say you don't know."*

**Look up when needed:**
- PyMuPDF / pdfplumber API specifics (documentation is good, no need to memorize)
- Qdrant client SDK methods (well-documented)
- RAGAS metric implementations (the math is documented, you just call the functions)

**Understand deeply:**
- Why chunking strategy is the #1 quality lever — *"a chunk that splits a key clause makes it unrecoverable by any amount of prompt engineering"*
- Why hybrid search > vector-only — *"embeddings miss exact matches. BM25 misses semantic matches. Together they cover both failure modes."*
- Why evaluation is hard for RAG — *"a 0.95 faithfulness score doesn't mean the answer is correct. It means the answer faithfully reflects the retrieved chunks. If the chunks are wrong, the answer is wrong. Trust no single metric."*

### First Concrete Step

> "Download 5 sample PDFs. Use PyMuPDF to extract text from one of them page by page. Print the first 3 pages. Look at the output. Is there header/footer noise? Page numbers in the middle of sentences? Tables that came out as garbage? Document what you find."

> *"This will take 20 minutes and tell you more about the challenge than reading about it."*

### Resources (Just-in-Time)

- **PyMuPDF (`fitz`) Quickstart** — PDF text extraction (use at Step 1)
- **LangChain Text Splitters documentation** — *read but don't use LangChain*. Learn the chunking strategies they implement so you can build them yourself (Step 2)
- **Qdrant Hybrid Search Tutorial** — their Universal Query API with prefetch + fusion (Step 3)
- **RAGAS Documentation** — faithfulness, answer relevancy, context precision/recall (Step 5)
- *Don't touch LangChain or LlamaIndex. You build this manually. THEN you get to use them.*

---

## Phase 3: The Build

> *This is the longest build phase in the course so far. Take it in milestones. Each one builds on the last.*

### Milestone 1: PDF Pipeline + Chunking

Extract text from PDFs and build a chunking module:

```python
import fitz  # PyMuPDF

def extract_text_from_pdf(pdf_path: str) -> dict[int, str]:
    """Extract text page by page. Returns {page_num: text}."""
    doc = fitz.open(pdf_path)
    pages = {}
    for page_num in range(len(doc)):
        page = doc[page_num]
        text = page.get_text()
        pages[page_num + 1] = text
    return pages
```

Then implement chunking strategies:

```python
def chunk_fixed_size(text: str, chunk_size: int = 1500, overlap: int = 150) -> list[dict]:
    """Simple character-based chunking with overlap."""
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunk_text = text[start:end]
        chunks.append({"text": chunk_text, "start": start, "end": end})
        start = end - overlap
    return chunks

def chunk_recursive(text: str, max_chars: int = 1500) -> list[dict]:
    """
    Recursive chunking: split on paragraph breaks first,
    then sentence breaks, then character count.
    This is what LangChain's RecursiveCharacterTextSplitter does.
    """
    # Split on double newlines (paragraphs)
    paragraphs = text.split("\n\n")
    
    chunks = []
    current_chunk = ""
    for para in paragraphs:
        if len(current_chunk) + len(para) < max_chars:
            current_chunk += para + "\n\n"
        else:
            if current_chunk:
                chunks.append({"text": current_chunk.strip()})
            current_chunk = para + "\n\n"
    if current_chunk:
        chunks.append({"text": current_chunk.strip()})
    return chunks
```

**Expected stuck point:** Page numbers and headers get mixed into the middle of sentences. "--- Page 4 --- Acme Corp v. Smith, 2023" becomes noise in your chunks.

**Maya's Socratic question:**
> *"Your first chunk includes '--- Page 3 ---' in the middle of a sentence about indemnification. How would a lawyer feel if their search for 'indemnification' returned a chunk that's mostly a page number? And how would you clean this?"*

> They should: implement text cleaning (regex to strip headers/footers/page numbers), add metadata (source document, page range) to each chunk so citations are possible.

### Milestone 2: Hybrid Search with Qdrant

Set up Qdrant (local Docker or in-memory) and implement hybrid search:

```python
from qdrant_client import QdrantClient
from qdrant_client.models import (
    VectorParams, SparseVectorParams, SparseIndexParams,
    PointStruct, Filter, FusionQuery, Fusion
)

client = QdrantClient(":memory:")  # or "localhost:6333" for Docker

# Create collection with both dense and sparse vectors
client.create_collection(
    collection_name="legal_docs",
    vectors_config={
        "dense": VectorParams(size=384, distance="Cosine"),
    },
    sparse_vectors_config={
        "sparse": SparseVectorParams(
            modifier="idf"  # BM25-style weighting
        )
    }
)

# Upload chunks with both embedding types
points = []
for chunk in chunks:
    dense_vec = embedding_model.encode(chunk["text"])
    sparse_vec = bm25_model.encode(chunk["text"])
    
    points.append(PointStruct(
        id=chunk["id"],
        vector={
            "dense": dense_vec.tolist(),
            "sparse": sparse_vec
        },
        payload={
            "text": chunk["text"],
            "source": chunk["source"],
            "page": chunk["page"]
        }
    ))

client.upsert(collection_name="legal_docs", points=points)

# Hybrid search with Reciprocal Rank Fusion
def hybrid_search(query: str, top_k: int = 5):
    dense_vec = embedding_model.encode(query)
    sparse_vec = bm25_model.encode(query)
    
    results = client.query_points(
        collection_name="legal_docs",
        prefetch=[
            Prefetch(query=dense_vec.tolist(), using="dense", limit=20),
            Prefetch(query=sparse_vec, using="sparse", limit=20),
        ],
        query=FusionQuery(fusion=Fusion.RRF),
        limit=top_k
    )
    return results.points
```

**Expected stuck point:** Running Qdrant needs Docker. If Docker isn't installed, set it up. The in-memory mode works for testing but won't persist.

**Maya's Socratic question:**
> *"You're using Reciprocal Rank Fusion (RRF) for hybrid search. Why RRF instead of averaging scores? What problem does RRF solve that score averaging doesn't?"*

> They should arrive at: dense and sparse have different score ranges (cosine similarity 0-1, BM25 unbounded). RRF uses rank positions instead of raw scores, so the ranges don't matter. A document ranked #1 by one method still gets high fusion score regardless of raw score magnitude.

### Milestone 3: RAG Generation with Citations

The critical part — turning retrieved chunks into cited answers:

```python
RAG_PROMPT = """You are a legal document assistant. Answer the user's question based ONLY on the provided context.

For every claim you make, you MUST cite:
- The source document name
- The page number
- The exact passage that supports your claim

Format citations as: [Source: DocumentName, Page X]

If the context does not contain enough information to answer the question, say:
"I cannot answer this question based on the provided documents."

Context:
{context}

Question: {query}
Answer:"""

def answer_question(query: str, retrieved_chunks: list[dict]) -> str:
    context = "\n\n---\n\n".join([
        f"[Source: {c['source']}, Page {c['page']}]\n{c['text']}"
        for c in retrieved_chunks
    ])
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": RAG_PROMPT.format(context=context, query=query)}
        ],
        temperature=0.0  # ZERO temperature. We want deterministic, faithful answers
    )
    return response.choices[0].message.content
```

**Expected stuck point:** The LLM sometimes cites a source that sounds plausible but doesn't actually exist in the context. The model "hallucinates" citations — especially when chunks are similar.

**Maya's Socratic question:**
> *"Your LLM cited 'Acme Corp Contract, Page 15' for an indemnification clause. But Page 15 of that contract is actually the signature page, not the indemnification clause. How do you catch and prevent citation hallucination?"*

> They should: (1) post-process the response to verify every citation exists in the provided context, (2) use structured output (Pydantic model with `citations: list[Citation]`) so citations are parseable, (3) implement a citation checker that confirms each claim has a supporting source.

### Milestone 4: Evaluation with RAGAS

Build an evaluation pipeline:

```python
from deepeval.metrics import FaithfulnessMetric, AnswerRelevancyMetric
from deepeval.test_case import LLMTestCase

# Build 50 test queries with expected answers
test_queries = [
    {
        "query": "What are the indemnification terms in the Acme Corp contract?",
        "expected_answer": "The Acme Corp contract indemnifies both parties...",
        "relevant_docs": ["acme_corp_contract.pdf"],
    },
    # ... 49 more
]

# Evaluate
faithfulness = FaithfulnessMetric(threshold=0.8)
answer_relevancy = AnswerRelevancyMetric(threshold=0.8)

for test in test_queries:
    chunks = hybrid_search(test["query"])
    answer = answer_question(test["query"], chunks)
    
    test_case = LLMTestCase(
        input=test["query"],
        actual_output=answer,
        retrieval_context=[c["text"] for c in chunks]
    )
    
    faithfulness.measure(test_case)
    answer_relevancy.measure(test_case)
    
    print(f"Query: {test['query']}")
    print(f"Faithfulness: {faithfulness.score:.2f}")
    print(f"Answer Relevancy: {answer_relevancy.score:.2f}")
    print(f"Citation Verified: {verify_citations(answer, chunks)}")
    print("---")
```

**Expected stuck point:** The evaluation itself costs money (each metric call is an LLM-as-judge API call). Evaluating 50 queries × 4 metrics = 200 LLM calls.

**Maya's Socratic question:**
> *"Your evaluation just cost you $3 to run 50 queries. If you run this eval after every change, your eval budget will exceed your build budget. How do you balance evaluation thoroughness with cost?"*

> They should consider: eval on a smaller sample (10 queries) during iteration, full 50-query eval only for final submission, or using a cheaper model for the LLM-as-judge.

### Rohan's Mid-Build Interruption

> *Rohan appears mid-chunking. He's holding a printout of a chunk that clearly cut a sentence in half.*

"Your chunking strategy split a key clause — 'The Company shall indemnify the Client against...' on one chunk and '...any losses arising from breach of contract' on the next. A lawyer searching for 'indemnification' gets the first chunk, the LLM reads it, and the answer misses HALF the clause.

Your citation is technically correct but substantively WRONG. The answer only tells half the story.

**Question:** How do you prevent chunk boundaries from destroying semantic meaning? And how do you measure whether your chunking is good enough?"

### Priya's Requirement Change

> *Priya: "Client just called. Half their PDFs are scanned images, not text. They were scanned from physical contracts. Your text extraction will return nothing for those. You need OCR. I got you two extra days."*

Bonus hardship: integrate OCR (Tesseract via `pytesseract` or an API like Azure Document Intelligence) as a fallback when PyMuPDF returns empty or near-empty text for a page.

---

## Phase 4: Review (Rohan)

> *You submit: RAG pipeline with hybrid search, citation rendering, 50-query evaluation report, OCR fallback.*

### Decision Documentation Required

1. Chunking strategy comparison — which you chose and why (with evidence from your own testing)
2. Why hybrid search (RRF) vs vector-only or keyword-only — quantified improvement on your test set
3. How you prevent citation hallucination — your verification mechanism and its effectiveness
4. Evaluation results — faithfulness, answer relevancy, citation accuracy with specific numbers

### Rohan's Review

| # | Check | Status | Notes |
|---|---|---|---|
| 1 | **Does it work?** | ✅ PASS | End-to-end pipeline. Ask questions, get cited answers. |
| 2 | **Edge cases?** | ✅ PASS | Handles scanned PDFs, mixed-format docs, out-of-domain queries correctly abstains. Good. |
| 3 | **Cost-aware?** | 🔄 REVISE | You're using GPT-4o-mini for generation which is smart. But you embed EVERY query with no caching — identical questions re-embed and re-search. Add a query cache. |
| 4 | **Observable?** | 🔄 REVISE | When a query fails (returns wrong answer), I can't debug why. Log: which chunks were retrieved, their similarity scores, what the LLM received as context, the raw LLM response. |
| 5 | **Right approach?** | ✅ PASS | Scratch-first RAG. Hybrid search. Citation enforcement. This is production architecture. |
| 6 | **Decisions justified?** | ✅ PASS | Chunking comparison data. Hybrid search vs vector-only numbers. Solid. |
| 7 | **Measurable quality?** | ✅ PASS | 50-query eval with faithfulness (0.91), answer relevancy (0.88), citation accuracy (0.94). These numbers give me confidence. |

### Verdict: 🔄 REVISE

*Rohan closes your eval report.*

"Two things:

1. **Query caching.** When Priya asks the same question twice (she will), your system should return instantly from cache. Implement a simple in-memory cache keyed on query text + top-k. TT L of 5 minutes is fine for this use case.

2. **Observability.** Add a trace endpoint. `/debug/query?q=...` returns: which chunks were retrieved, their similarity scores (dense + sparse separately, and RRF fused), the full context sent to the LLM, the raw response, and the citation verification results. This is how you debug failures.

Fix both and resubmit. The law firm demo is in 2 days."

---

## Phase 5: Debrief (Maya)

> *After APPROVED. Rohan actually said "this passes." You're not sure how to feel.*

**Maya:** "You just built the most common AI system in production. Let me tell you what that means:

- **RAG isn't a product feature — it's THE pattern.** 70% of enterprise AI systems use RAG. You now know how the stack works from bedrock to surface.
- **Hybrid search is the production standard.** You built dense + sparse with RRF. This is what every serious RAG deployment uses. No one in production runs vector-only after they've tried hybrid.
- **Evaluation is what makes it real.** Those RAGAS numbers? They're what Rohan (and real CTOs) use to decide whether a system goes to production. Without eval, you're guessing.
- **Citations are a UX guarantee.** The lawyers don't trust the AI. They trust citations. By enforcing citation verification, you built trust into the system architecturally.

**What you'll see again:**
- **Agentic RAG** — Project 9 adds LLM-driven query rewriting and multi-step retrieval. Your pipeline becomes an agent that decides HOW to search.
- **Reranking** — Project 5 adds cross-encoder reranking between retrieval and generation. Your top-5 will get even better.
- **Evaluation in production** — Tier 3 adds continuous evaluation. Your RAGAS script becomes a CI job that runs on every change.
- **LangChain** — Now that you've built RAG by hand, you have PERMISSION to try LangChain. But here's the secret: by building it manually, you've learned more than most people learn in 3 months of LangChain tutorials.

> *Maya smiles. "You're now dangerous. Priya's already got the next client brief ready — an e-commerce company whose support team is drowning in tickets."*
