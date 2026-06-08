# Project 3: Internal Semantic Search Tool

- **Tier:** 1 — Internal Studio Tools
- **Project #:** 3 of 3
- **Tech Stack:** Python, sentence-transformers (or OpenAI embeddings), NumPy, FAISS (optional), FastAPI
- **Concepts:** Embeddings, vector similarity search (in-memory with NumPy), semantic vs keyword search, simple API serving
- **Quality Gate:** ✅ APPROVED when you can explain why cosine similarity was the right choice (or when it wasn't), and demonstrate your search returns **provably better results** than Ctrl+F on the same documents.

---

## Phase 1: Brief (Priya)

> *Priya appears at your desk with a stack of printed pages. She drops them with a thud.*

"We have 500+ pages of internal documentation. Notion docs. Google Docs. Slack threads. Old decisions that nobody remembers. Every time someone asks 'didn't we decide this already?' — nobody knows. Because finding anything in our documentation is impossible.

Here's the specific problem: **Ctrl+F doesn't work when you don't know the exact words.** You remember the concept but not the phrasing. 'That thing we discussed about rate limiting for the legal client' — try Ctrl+F on that. You'll be searching for hours.

**What I need:** A search tool where you can type what you're looking for in natural language — *"how do we handle API rate limiting for large clients"* — and it finds the right document even if the document uses completely different words: *"throttling strategy for enterprise customers."*

**Non-negotiable:**
- It has to work on our actual docs (I'll give you a folder of markdown files — the exported versions)
- It has to be FAST. I'm not waiting 30 seconds for a search
- It has to be BETTER than Ctrl+F. We can measure this
- The results must show me WHY a document matched — give me the relevant passage, not just the filename

**Deadline:** 5 days. The team keeps asking me the same questions and I'm tired of answering them.

**Definition of done:** I can type a vague description of what I need → get ranked results with the actual relevant passage highlighted → prove this is better than keyword search."

---

## Phase 2: Learning Path (Maya)

> *Maya grins when she sees Priya's ticket. "Oh, this is the good one. This project is your gateway drug to RAG."*

"This is the most important project in Tier 1. Because **every RAG pipeline** — every single one — is this project plus one extra step (generation). When you understand embeddings and vector search, you understand the foundation of everything that comes next.

### Learning Order (Scratch-First)

**Step 1: Naive keyword search (baseline).**
Build a simple keyword search first. Split documents into chunks, split queries into words, find chunks with the most word overlap.

> *"This will be terrible. That's the point. You need to feel the pain of keyword search so you can APPRECIATE what embeddings solve."*

**Step 2: Generate embeddings.**
Take each document chunk and convert it into a vector using an embedding model. You have two choices:

- **sentence-transformers** (open-source, runs locally, free): `all-MiniLM-L6-v2` gives you 384-dimension vectors, runs on CPU, ~50ms per chunk
- **OpenAI text-embedding-3-small** (API, costs money, higher quality): 1536-dimension vectors, but costs $0.02/1M tokens

> *"Research reality (2026): The gap between open-source and commercial embeddings has narrowed dramatically. `BGE-M3` and `E5` models are now within a few points of OpenAI on most benchmarks. For this project, use `sentence-transformers/all-MiniLM-L6-v2` — it's free, fast, and good enough."*

**Step 3: Implement vector search with NumPy.**
Store embeddings in a NumPy array. Implement cosine similarity manually. Write a search function that embeds the query and finds the closest chunks.

> *"FAISS is faster. Qdrant is more scalable. But if you can't implement cosine similarity in NumPy, you don't understand what they're optimizing. Build from scratch first."*

**Step 4: Compare against baseline.**
Run 10 real search queries. Compare your semantic search results against the keyword search baseline. Show which one finds better results. Document the cases where keyword search still wins.

**Step 5: Serve it with FastAPI.**
Wrap the search in a simple API endpoint. `/search?q=how+to+handle+rate+limiting` returns ranked results with passages and similarity scores.

> *"This is your first RAG pipeline. No generation yet. Just retrieval. You're building the engine that powers every RAG system in production."*

### Memory Triage

**Memorize cold:**
- Embedding API call pattern — `model.encode(text)` for sentence-transformers, `client.embeddings.create(input=text)` for OpenAI
- Cosine similarity formula — `dot(a, b) / (norm(a) * norm(b))`
- Top-k retrieval pattern — sort by similarity, take top K
- FastAPI endpoint structure — `@app.get("/search")` with query param

**Look up when needed:**
- Specific sentence-transformers model names and their dimensions
- NumPy dot product and norm functions (`np.dot`, `np.linalg.norm`)
- FastAPI `Query` parameter details
- FAISS index types if you add it as an optimization

**Understand deeply:**
- Why cosine similarity works for semantic search — *"it measures direction, not magnitude. Two documents about rate limiting point in similar directions in embedding space even if one says 'throttle' and the other says 'rate limit.'"*
- Why chunk size matters — *"too small = no context. too large = too many concepts in one vector. you're about to discover this empirically."*
- The difference between sparse (keyword) and dense (embedding) retrieval — *"keyword finds exact matches. Embeddings find conceptual matches. The best systems use both (hybrid search). You'll build that in Tier 2."*

### First Concrete Step

> "First: `pip install sentence-transformers numpy fastapi uvicorn`. Then: take 5 documents. Write a script that embeds them with `SentenceTransformer('all-MiniLM-L6-v2')`. Manually inspect the embeddings — print one. What does a 384-dimensional vector look like?"

> *"Most people never look at an embedding. Look at one. It's just a list of numbers. But when you arrange them right, they encode meaning. That's the magic."*

### Resources (Just-in-Time)

- **sentence-transformers docs** — quickstart, model list, usage examples (use when setting up embeddings)
- **NumPy Linear Algebra** — `np.dot`, `np.linalg.norm`, `np.argsort` (reference for similarity implementation)
- **FastAPI Official Tutorial** — path operations, query params, Pydantic response models (use when serving the API)
- **The MTEB Leaderboard** — see how embedding models compare (browse, don't overthink)
- **Note:** Don't touch FAISS yet. You'll add it when the NumPy approach gets too slow. That's the point.

---

## Phase 3: The Build

> *Time to build the search engine that will crush Ctrl+F.*

### Milestone 1: Keyword Search Baseline

Build a simple search:

```python
import re
from collections import Counter

def keyword_search(query: str, documents: list[dict], top_k: int = 5):
    """Split query into words, find documents with most matching words."""
    query_words = set(re.findall(r'\w+', query.lower()))
    
    scores = []
    for doc in documents:
        doc_words = set(re.findall(r'\w+', doc["text"].lower()))
        overlap = len(query_words & doc_words)
        scores.append((doc["id"], overlap, doc["text"]))
    
    scores.sort(key=lambda x: x[1], reverse=True)
    return scores[:top_k]
```

**Expected stuck point:** You search for "rate limiting strategies" and the document titled "Throttling Guide" gets 0 matches even though it's the most relevant document.

**Maya's Socratic question:**
> *"Your search for 'rate limiting' didn't find 'throttling guide.' What information is missing from your search that embeddings can add?"*

> They should arrive at: keyword search matches exact words, not concepts. Embeddings capture semantic relationships — "throttling" and "rate limiting" are close in embedding space even though they share no words.

### Milestone 2: Embeddings + Vector Search

Implement the embedding pipeline:

```python
from sentence_transformers import SentenceTransformer
import numpy as np

# Load model (downloads on first run, cached after that)
model = SentenceTransformer('all-MiniLM-L6-v2')

# Chunk documents
def chunk_text(text: str, chunk_size: int = 512, overlap: int = 50):
    """Split text into overlapping chunks of ~chunk_size characters."""
    # Start with simple character-based chunking first
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start = end - overlap
    return chunks

# Embed all chunks
def embed_chunks(chunks: list[str]) -> np.ndarray:
    embeddings = model.encode(chunks, show_progress_bar=True)
    return np.array(embeddings)  # shape: (n_chunks, 384)

# Search function
def semantic_search(query: str, chunk_embeddings: np.ndarray, chunks: list[str], top_k: int = 5):
    query_embedding = model.encode([query])
    
    # Cosine similarity
    similarities = np.dot(chunk_embeddings, query_embedding.T).flatten()
    # Normalize
    query_norm = np.linalg.norm(query_embedding)
    chunk_norms = np.linalg.norm(chunk_embeddings, axis=1)
    similarities = similarities / (chunk_norms * query_norm)
    
    top_indices = np.argsort(similarities)[::-1][:top_k]
    
    results = []
    for idx in top_indices:
        results.append({
            "chunk": chunks[idx],
            "score": float(similarities[idx]),
            "index": int(idx)
        })
    return results
```

**Expected stuck point:** Your cosine similarity produces NaN for some chunks because their norm is 0 (empty or very short chunks).

**Maya's Socratic question:**
> *"Some of your similarity scores are NaN. What does a norm of 0 mean? And how should you handle it?"*

> They should: identify the root cause (empty/short chunks), add a minimum length filter, and handle edge cases in normalization.

### Milestone 3: The A/B Test — Semantic vs Keyword

This is the most important part. Prove your search is better:

1. Write 10 test queries that represent real searches people would make
2. For each query, manually determine which document is the correct answer
3. Run both keyword and semantic search
4. Compare: which one found the right document? At what rank?
5. Document cases where keyword search won (yes, there will be some)

**Expected finding:** Semantic search wins for ~7/10 queries. Keyword wins for ~3/10 — typically queries with unique proper nouns like "Acme Corp contract clause 14.3" where exact word matching is better.

**Maya's Socratic question:**
> *"Keyword search won on 3 of your 10 test queries. Why? And more importantly — how would you design a search that gets the best of both?"*

> This plants the seed for **hybrid search** (dense + sparse) — the production standard you'll implement in Tier 2.

### Milestone 4: FastAPI + Response

Serve the search:

```python
from fastapi import FastAPI, Query
from pydantic import BaseModel

app = FastAPI(title="NexaAI Semantic Search")

class SearchResult(BaseModel):
    passage: str
    score: float
    document_title: str

class SearchResponse(BaseModel):
    query: str
    results: list[SearchResult]
    total_results: int

@app.get("/search", response_model=SearchResponse)
def search(q: str = Query(..., description="Natural language search query"), top_k: int = 5):
    results = semantic_search(q, chunk_embeddings, chunks, top_k)
    return SearchResponse(
        query=q,
        results=results,
        total_results=len(results)
    )
```

**Expected stuck point:** The first load is slow because you're embedding 500+ documents on startup. Users have to wait.

**Maya's Socratic question:**
> *"Your API takes 15 seconds to start because it's embedding everything on boot. How would you fix this?"*

> They should think about: pre-compute and cache embeddings, save them to disk with `np.save()`, load on startup instead of recomputing. This is the exact pattern vector databases solve (persistent storage + fast retrieval).

### Rohan's Mid-Build Interruption

> *Rohan appears during your A/B testing.*

"Your semantic search results. Let me see them. ... Okay, I have a question.

Why cosine similarity and not dot product? Your embeddings are normalized anyway, so they'd give the same ranking. Why did you write `dot / (norm * norm)` instead of just `dot`?"

> If the learner normalized their embeddings, cosine similarity and dot product ARE identical — Rohan is checking whether they understand the math or just copied the formula.

> *"If you normalized your vectors, `cosine_sim(a, b)` == `dot(a, b)`. Did you notice? If not, what does that tell you about when to use each?"*

### Priya's Requirement Change

> *Priya: "Great news — Rohan loves the search tool. Now he wants it to ALSO show which documents DON'T match anything. Like, for a given query, tell me which of our 500 documents you think are irrelevant. He wants to audit the coverage of our documentation."*

> *"Can you add a relevance filtering feature? Show all documents ranked, with a similarity threshold below which things are marked as 'likely irrelevant to this query.'"*

This forces the learner to implement **confidence thresholds** — a production pattern they'll use constantly for "I don't know" handling.

---

## Phase 4: Review (Rohan)

> *You submit: FastAPI app with pre-computed embeddings, working search endpoint, A/B test results against keyword search, relevance filtering.*

### Decision Documentation Required

1. Why you chose sentence-transformers vs OpenAI embeddings (cost, quality, latency comparison on YOUR data)
2. Your chunking strategy and why (chunk size, overlap, what edge cases you discovered)
3. Why cosine similarity was the right metric (and when it wouldn't be)
4. What the A/B test showed — where semantic search won, where keyword search won, and why

### Rohan's Review

| # | Check | Status | Notes |
|---|---|---|---|
| 1 | **Does it work?** | ✅ PASS | Search works, returns relevant results, fast response |
| 2 | **Edge cases handled?** | 🔄 REVISE | Handles empty queries, short queries, long queries? What about special characters? Emoji? What about queries for things that DON'T exist in the corpus? |
| 3 | **Cost-aware?** | ✅ PASS | Pre-computed embeddings. Free model. One-time cost, zero per-query cost. Good. |
| 4 | **Observable?** | 🔄 REVISE | I can't see what chunks were retrieved and their scores. Add a `debug=true` query parameter that returns intermediate data (similarity distribution, top 20 scores not just top 5) |
| 5 | **Right approach?** | ✅ PASS | Chunking → embedding → NumPy similarity → FastAPI. Clean, minimal, correct. |
| 6 | **Decisions justified?** | ✅ PASS | Clear reasoning on sentence-transformers vs OpenAI. You actually tested both. |
| 7 | **Measurable quality?** | ✅ PASS | The A/B test against keyword search is exactly what I wanted to see. You proved improvement with data. |

### Verdict: 🔄 REVISE

*Rohan closes his laptop.*

"Two things:

1. **Edge case: out-of-domain queries** — If I search for 'how to cook pasta' on our technical docs, your system will still return the closest match even if it's completely unrelated. You need a similarity threshold — below 0.3 (or whatever threshold your data shows), return 'no relevant results found.' This is the same pattern you'll use for 'I don't know' handling in every RAG system from now on.

2. **Observability** — Add a `debug=true` mode. When enabled, return the top 20 similarity scores (not just top 5), the query embedding, the distribution of all scores. When your search fails to find something relevant, this is how you debug why.

Fix both and you're done with Tier 1."

---

## Phase 5: Debrief (Maya)

> *After APPROVED. Rohan nods at you in the hallway. It means nothing and everything.*

**Maya:** "You just built the foundation of every RAG system in production. Let's name what you learned:

- **Embeddings are the new index.** Instead of keyword-to-document mapping, you built concept-to-vector mapping. This is how all modern search works.
- **Vector similarity is a ranking mechanism.** Cosine similarity, dot product, euclidean distance — they all measure 'closeness' in different ways. You now know when each matters.
- **Chunking is the first hard problem.** You discovered empirically that chunk size affects search quality. This is the #1 lever in RAG quality — and you've already started developing intuition for it.
- **Evaluation by A/B test.** You didn't assume semantic search was better. You proved it against a baseline. This mindset is what makes you an engineer, not a hobbyist.

**What you built is literally the 'R' in RAG.** Project 4 (Legal Document Q&A) is exactly this, plus one step: after retrieval, pass the chunks to an LLM with a prompt like 'answer based only on this context and cite your sources.'

**What you'll see again:**
- **Hybrid search** — In Tier 2 you'll add BM25 (keyword) alongside your vector search so exact matches don't get lost
- **Vector databases** — NumPy arrays work for 500 docs. For 50,000 docs, you need Qdrant. Project 4.
- **Reranking** — Your top-5 results are good, but some chunk might be irrelevant. Cross-encoder reranking fixes this. Project 5.
- **Semantic caching** — When the same query comes twice, return cached results. You'll do this in Project 7.

> *Maya holds up three fingers. "Three projects. You've gone from 'can call an API' to 'built semantic search from scratch.' Take a breath. Tier 2 starts with a real client — and they have 500 PDFs that need answers."*

---

## Bonus: What You Could Do Next (Optional Enhancement)

If you want to push this project further (before Rohan moves you on):

1. **FAISS acceleration** — Replace NumPy brute-force with FAISS `IndexFlatIP` (inner product for normalized vectors). See the speed difference.
2. **Async FastAPI** — Add `async def search()` so multiple users can search simultaneously without blocking
3. **Dockerize it** — Package the search app in a Docker container. This is your first deployment-ready artifact.
4. **OpenAI embeddings comparison** — Swap sentence-transformers for `text-embedding-3-small` and compare quality/cost on the same 10 test queries. Write up the tradeoffs.

These aren't required for approval, but they'll make your Project 3 significantly easier when you see the same patterns in Tier 2.
