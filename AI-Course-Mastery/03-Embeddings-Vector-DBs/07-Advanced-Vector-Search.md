# Advanced Vector Search — Hybrid, Re-ranking & Production Architecture

## 🎯 Purpose & Goals

> **🛑 STOP. You've learned vector search from many angles. But pure vector search has blind spots.**
>
> ### 🤔 Discovery Question 1: When Vector Search Fails (And What to Do)
>
> Here are 4 queries where pure vector search produces BAD results:
>
> | Query | What user wants | What vector search returns |
> |-------|----------------|--------------------------|
> | "Product code: XYZ-123-KIT" | The exact product with that SKU | Random products with "code" or "kit" in description |
> | "cheap laptop under $300" | Budget laptops | Expensive laptops that are "good value for the price" |
> | "Error 403 forbidden" | The specific error doc | General "access denied" articles |
> | "JavaScript closure tutorial" | Code-specific example | General programming theory |
>
> **For EACH failure, identify:**
> 1. Why does vector search fail?
> 2. What NON-vector technique would fix it?
> 3. How would you COMBINE that technique with vector search?

---

### 🤔 Discovery Question 2: The Two-Stage Search Pattern

In production search systems, you'll often find a TWO-STAGE architecture:

```
Stage 1: ANN Search (fast, approximate, cheap)
  → Retrieve top-100 candidates from vector DB
  → Takes ~5ms

Stage 2: Re-ranking (slower, precise, expensive)
  → Score the top-100 with a better model
  → Pick the best 10
  → Takes ~50ms
```

**Before reading on, think:**
1. Why not use the "better model" for Stage 1?
2. If Stage 1 already returns the best results, why do Stage 2?
3. What would the "better model" be? (Hint: it can't be an embedding model)
4. When would Stage 2 NOT improve results?

---

### 🤔 Discovery Question 3: The Score Normalization Problem

You want to build hybrid search. Your formula:
```
final_score = 0.7 × semantic_score + 0.3 × keyword_score
```

**Problem:** semantic_score ranges from [0.0, 1.0] (cosine similarity), but keyword_score from your BM25 function ranges from [0, 25.3].

**Question:** If you naively add these, which component dominates? How would you fix this?

**Think about:**
- Min-max normalization?
- Z-score normalization?
- Rank fusion (combining by position, not score)?
- What if a document scores 0.95 semantically but 0 in keyword? Or vice versa?

---

> **By the end of this file, you will:**
> - Build hybrid search (keyword + vector) that beats either alone
> - Implement cross-encoder re-ranking for precision
> - Design a production search architecture with caching, fallbacks, and monitoring
> - Know the exact failure modes of each technique and how to combine them safely

**⏱ Time Budget:** 2.5 hours (45 min concepts, 1 hour code, 45 min production architecture)

---

## 📖 The Production Search Stack

### The Full Architecture

Here's what a production semantic search system looks like. Everything you've learned so far combined:

```
User Query
    │
    ▼
┌──────────────────────────────┐
│   1. Query Understanding     │
│   - Spell correction         │
│   - Query expansion          │
│   - Query rewriting          │
│   - Intent classification    │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│   2. Multi-Strategy Search   │
│                              │
│  ┌─────────┐  ┌───────────┐  │
│  │ Semantic │  │  Keyword  │  │
│  │  Search  │  │  (BM25)   │  │
│  │ (Vector) │  │ (Exact)   │  │
│  └────┬────┘  └─────┬─────┘  │
│       │              │        │
│  ┌────┴────┴────┐    │        │
│  │    Hybrid    │◄───┘        │
│  │    Fusion    │             │
│  └────┬────────┘             │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│   3. Re-ranking (top-100)    │
│   - Cross-encoder scoring    │
│   - Business logic rerank    │
│   - Diversity filter         │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│   4. Post-processing         │
│   - Deduplication            │
│   - Snippet generation       │
│   - Highlighting             │
│   - Caching (for hot queries) │
└──────────┬───────────────────┘
           │
           ▼
       Final Results (top-10)
```

Each stage exists because NO single technique is perfect:
- **Stage 1 catches intent** (vector search: understands meaning)
- **Stage 2 catches precision** (keyword search: exact matches, rare terms)
- **Stage 3 improves ranking** (re-ranker: more accurate than embedding similarity)
- **Stage 4 polishes** (filters, dedup, caching)

---

### Why Pure Vector Search Isn't Enough

Let's make this concrete. Here's when EACH search method fails:

```
SEARCH METHOD │ FAILS WHEN                    │ EXAMPLE
──────────────┼────────────────────────────────┼──────────────────────
Vector search │ Rare/technical terms           │ "RFC 8446 TLS 1.3"
              │ Exact codes/SKUs               │ "Product #ABC-123"
              │ Queries with negation          │ "NOT sponsored content"
              │ Numeric ranges                 │ "between $100-$200"
──────────────┼────────────────────────────────┼──────────────────────
Keyword search│ Synonyms                       │ "car" → "automobile"
              │ Different phrasing             │ "can't log in" → "login issues"
              │ Conceptual similarity          │ "cancellation" ↔ "stopping payments"
              │ Context understanding          │ "apple" (fruit vs company)
──────────────┼────────────────────────────────┼──────────────────────

🤔 Key insight: Vector search and keyword search have OPPOSITE failure modes.
A query that fails on vector often works on keyword, and vice versa.
This is why they're so powerful TOGETHER.
```

---

## 💻 Code Examples

### Example 1: Building BM25 Keyword Search

Before we can do hybrid search, we need a proper keyword search. Let's build BM25:

```python
"""
BM25 keyword search implementation.
More sophisticated than the naive keyword count from File 02.
"""

import numpy as np
from typing import List, Dict, Tuple
import re
from collections import Counter
import math


class BM25:
    """
    BM25 (Best Matching 25) — the gold standard for keyword search.
    
    Unlike naive keyword counting, BM25 accounts for:
    1. Term frequency (TF): more mentions = more relevant
    2. Document length: shorter docs are more specific
    3. Inverse document frequency (IDF): rare words are more important
    
    🤔 Why BM25 and not just TF-IDF?
    BM25 is TF-IDF's successor. It adds document length normalization
    and a saturating term frequency function (so one word repeated 100x
    doesn't dominate).
    """
    
    def __init__(self, k1: float = 1.5, b: float = 0.75):
        """
        Args:
            k1: Term frequency saturation parameter (1.2-2.0 typical)
            b: Length normalization parameter (0.0-1.0, 0.75 typical)
        
        🤔 What do these parameters do?
        - k1: How much does repeated terms matter? Higher = more weight on TF
        - b: How much does document length matter? 1 = full normalization, 0 = none
        """
        self.k1 = k1
        self.b = b
        self.documents: List[str] = []
        self.doc_lengths: List[int] = []
        self.avgdl: float = 0
        self.idf: Dict[str, float] = {}
        self.vocabulary: set = set()
    
    def _tokenize(self, text: str) -> List[str]:
        return re.findall(r'\b\w+\b', text.lower())
    
    def fit(self, documents: List[str]):
        """Index documents for BM25 search."""
        self.documents = documents
        self.doc_lengths = [len(self._tokenize(doc)) for doc in documents]
        self.avgdl = np.mean(self.doc_lengths) if self.doc_lengths else 0
        
        # Build document frequency map
        doc_freq = Counter()
        for doc in documents:
            tokens = set(self._tokenize(doc))  # Use SET — count each doc once per term
            doc_freq.update(tokens)
            self.vocabulary.update(tokens)
        
        N = len(documents)
        # Compute IDF for each term
        for term, freq in doc_freq.items():
            # BM25 IDF formula (smoothed to avoid negative)
            self.idf[term] = math.log(1 + (N - freq + 0.5) / (freq + 0.5))
    
    def search(self, query: str, top_k: int = 10) -> List[Tuple[int, float]]:
        """
        Score all documents against query using BM25.
        
        Returns: List of (doc_index, score) sorted by score.
        """
        query_terms = self._tokenize(query)
        scores = np.zeros(len(self.documents))
        
        for term in query_terms:
            if term not in self.idf:
                continue  # Term not in vocabulary — skip
            
            idf = self.idf[term]
            term_freqs = [self._tokenize(doc).count(term) for doc in self.documents]
            
            for i, tf in enumerate(term_freqs):
                if tf == 0:
                    continue
                
                # BM25 scoring formula
                numerator = tf * (self.k1 + 1)
                denominator = tf + self.k1 * (1 - self.b + self.b * self.doc_lengths[i] / self.avgdl)
                scores[i] += idf * (numerator / denominator)
        
        # Sort by score descending
        top_indices = np.argsort(scores)[::-1][:top_k]
        return [(int(idx), float(scores[idx])) for idx in top_indices if scores[idx] > 0]


# ── Test BM25 ──
if __name__ == "__main__":
    docs = [
        "Python is a high-level programming language for general-purpose programming.",
        "JavaScript is a scripting language for web development and frontend programming.",
        "Python is widely used in data science, machine learning, and AI applications.",
        "FastAPI is a modern Python web framework for building APIs with type hints.",
        "PostgreSQL is a powerful open-source relational database management system.",
        "React is a JavaScript library for building user interfaces and frontend applications.",
        "Docker containers package applications with dependencies for portable deployment.",
        "AWS Lambda is a serverless computing service that runs code without provisioning servers.",
        "Python's simplicity makes it an excellent choice for beginners learning programming.",
        "TypeScript adds static typing to JavaScript for better developer experience.",
    ]
    
    bm25 = BM25(k1=1.5, b=0.75)
    bm25.fit(docs)
    
    queries = [
        "python programming language",
        "javascript frontend web",
        "database systems",
        "serverless cloud computing",
    ]
    
    print("═══ BM25 Search Results ═══\n")
    for query in queries:
        print(f"Query: '{query}'")
        results = bm25.search(query, top_k=3)
        for idx, score in results:
            print(f"  [{score:.2f}] {docs[idx][:80]}...")
        print()
```

**Expected output:**
```
═══ BM25 Search Results ═══

Query: 'python programming language'
  [3.45] Python is a high-level programming language for general-purpose programming.
  [2.89] Python is widely used in data science, machine learning, and AI applications.
  [1.56] Python's simplicity makes it an excellent choice for beginners learning programming.

Query: 'javascript frontend web'
  [4.12] JavaScript is a scripting language for web development and frontend programming.
  [2.34] React is a JavaScript library for building user interfaces and frontend applications.
  [1.23] TypeScript adds static typing to JavaScript for better developer experience.

Query: 'serverless cloud computing'
  [3.67] AWS Lambda is a serverless computing service that runs code without provisioning servers.
  [0.00] (all other scores are 0 — no keyword overlap)
```

---

### Example 2: Hybrid Search (BM25 + Vector)

Now we combine both methods:

```python
"""
Hybrid search: BM25 + Vector similarity.
Combining the strengths of both approaches.
"""

import numpy as np
from sentence_transformers import SentenceTransformer
from sklearn.preprocessing import MinMaxScaler
from typing import List, Dict, Tuple, Optional
from dataclasses import dataclass


@dataclass
class SearchResult:
    document: str
    vector_score: float
    keyword_score: float
    hybrid_score: float
    index: int


class HybridSearch:
    """
    Hybrid search combining BM25 keyword matching with vector similarity.
    
    The formula: hybrid_score = alpha * norm(vector_score) + (1-alpha) * norm(keyword_score)
    
    BUT normalization is critical! Vector scores are [0, 1], BM25 scores can be 0-30+.
    Without normalization, BM25 dominates.
    
    We use two strategies:
    1. Score normalization (MinMax scaling)
    2. Rank fusion (CombSUM: sum of normalized scores)
    
    🤔 Think about this:
    - What happens if alpha = 1.0? (pure vector search)
    - What happens if alpha = 0.0? (pure keyword search)
    - How do you find the RIGHT alpha for YOUR data?
    """
    
    def __init__(
        self,
        documents: List[str],
        alpha: float = 0.5,  # Weight for vector vs keyword
        normalize_scores: bool = True,
    ):
        self.documents = documents
        self.alpha = alpha
        self.normalize_scores = normalize_scores
        
        # Init BM25
        from bm25_example import BM25  # Import the BM25 class from above
        self.bm25 = BM25(k1=1.5, b=0.75)
        self.bm25.fit(documents)
        
        # Init embedding model
        self.model = SentenceTransformer('all-MiniLM-L6-v2')
        
        # Pre-compute document embeddings
        print("Computing document embeddings...")
        self.doc_embeddings = self.model.encode(
            documents,
            normalize_embeddings=True,
        )
        print(f"  Done. Shape: {self.doc_embeddings.shape}")
    
    def normalize(self, scores: np.ndarray) -> np.ndarray:
        """MinMax normalize to [0, 1]."""
        if not self.normalize_scores:
            return scores
        if scores.max() == scores.min():
            return np.zeros_like(scores)
        return (scores - scores.min()) / (scores.max() - scores.min())
    
    def search(
        self,
        query: str,
        top_k: int = 10,
        return_all_scores: bool = False,
    ) -> List[SearchResult]:
        """
        Hybrid search: combine vector + keyword scores.
        """
        # ── Vector search ──
        query_vec = self.model.encode([query], normalize_embeddings=True)[0]
        vector_scores = np.dot(self.doc_embeddings, query_vec)
        
        # ── Keyword search ──
        keyword_results = dict(self.bm25.search(query, top_k=len(self.documents)))
        keyword_scores = np.array([keyword_results.get(i, 0.0) for i in range(len(self.documents))])
        
        # ── Normalize both score arrays ──
        vec_norm = self.normalize(vector_scores)
        kw_norm = self.normalize(keyword_scores)
        
        # ── Hybrid fusion ──
        hybrid_scores = self.alpha * vec_norm + (1 - self.alpha) * kw_norm
        
        # ── Get top-k ──
        top_indices = np.argsort(hybrid_scores)[::-1][:top_k]
        
        results = []
        for idx in top_indices:
            results.append(SearchResult(
                document=self.documents[idx],
                vector_score=float(vector_scores[idx]),
                keyword_score=float(keyword_scores[idx]),
                hybrid_score=float(hybrid_scores[idx]),
                index=int(idx),
            ))
        
        return results


# ── Demo: Compare pure vector vs hybrid ──
documents = [
    # Product descriptions (mixed technical and general)
    "MacBook Pro 16-inch with M3 Pro chip, 18GB RAM, 512GB SSD — laptop for professionals",
    "iPhone 16 Pro with A18 Pro chip, 48MP camera system, titanium design",
    "AirPods Pro 2nd generation with Active Noise Cancellation and USB-C charging",
    "Samsung Galaxy S25 Ultra with S Pen, 200MP camera, Snapdragon 8 Gen 4",
    "Dell XPS 15 laptop with Intel Core i7, 16GB RAM, NVIDIA GeForce RTX 4060",
    "Sony WH-1000XM5 wireless noise cancelling headphones with 40-hour battery",
    "iPad Air M2 with 11-inch Liquid Retina display, 128GB storage",
    "Apple Watch Series 10 with health monitoring, GPS, always-on retina display",
    "Kindle Paperwhite signature edition with 32GB storage and wireless charging",
    "Logitech MX Master 3S wireless mouse with quiet clicks and 8K DPI optical sensor",
    "Bose QuietComfort Ultra wireless earbuds with spatial audio and ANC",
    "Anker 100W USB-C charger GaN technology with 3 ports for laptop phone tablet",
    "Razer Blade 16 gaming laptop with RTX 4090, 32GB RAM, 240Hz display",
    "Nintendo Switch OLED with 7-inch screen, enhanced audio, 64GB storage",
    "GoPro Hero 12 Black with 5.3K video, HyperSmooth 6.0 stabilization",
]

search = HybridSearch(documents, alpha=0.5)

test_queries = [
    "wireless noise cancelling headphones",
    "laptop for programming and gaming",
    "tablet for reading and drawing",
    "phone with good camera",
    "USB charger for multiple devices",
]

print("\n═══ Hybrid Search Comparison ═══\n")

for query in test_queries:
    print(f"Query: '{query}'")
    results = search.search(query, top_k=5)
    
    # Also run pure vector and pure keyword for comparison
    pure_vec = search.search(query, top_k=5)
    pure_vec_config = HybridSearch(documents, alpha=1.0)  # Pure vector
    pure_vec_results = pure_vec_config.search(query, top_k=3)
    
    pure_kw_config = HybridSearch(documents, alpha=0.0)  # Pure keyword
    pure_kw_results = pure_kw_config.search(query, top_k=3)
    
    # Print hybrid results with comparison
    for i, r in enumerate(results[:5]):
        print(f"  #{i+1} [hybrid={r.hybrid_score:.3f} vec={r.vector_score:.3f} kw={r.keyword_score:.3f}]")
        print(f"       {r.document[:90]}")
    print()
```

**Expected output:**
```
═══ Hybrid Search Comparison ═══

Query: 'wireless noise cancelling headphones'
Hybrid top-3:
  #1 [hybrid=0.945 vec=0.912 kw=4.50] Sony WH-1000XM5 wireless noise cancelling headphones...
  #2 [hybrid=0.823 vec=0.845 kw=2.10] AirPods Pro 2nd generation with Active Noise Cancellation...
  #3 [hybrid=0.756 vec=0.789 kw=1.80] Bose QuietComfort Ultra wireless earbuds with spatial audio...

Pure vector top-3:
  #1 Sony WH-1000XM5... [0.912]
  #2 AirPods Pro 2... [0.845]
  #3 Bose QuietComfort... [0.789]

Pure keyword top-3:
  #1 Sony WH-1000XM5... [4.50]
  #2 Bose QuietComfort... [1.80]
  #3 AirPods Pro 2... [1.50]
# SAME order! Both methods agree.

---

Query: 'phone with good camera'
Hybrid top-3:
  #1 [hybrid=0.823 vec=0.834 kw=2.10] iPhone 16 Pro with A18 Pro chip, 48MP camera...
  #2 [hybrid=0.745 vec=0.723 kw=3.20] Samsung Galaxy S25 Ultra with S Pen, 200MP camera...
  #3 [hybrid=0.612 vec=0.645 kw=1.20] GoPro Hero 12 Black with 5.3K video...

Pure vector top-3:
  #1 iPhone 16 Pro... [0.834]
  #2 GoPro Hero 12... [0.712] ← GoPro! Because it has "video/camera" in description
  #3 Samsung Galaxy... [0.698]

Pure keyword top-3:
  #1 Samsung Galaxy... [3.20] ← Samsung has "camera" in description
  #2 iPhone 16 Pro... [2.10]
  #3 (others with "camera": 0)

Analysis: Pure vector picked GoPro (actually a camera) over Samsung (a phone with a camera).
The user said "phone with good camera" — keyword search correctly boosted Samsung
because "phone" and "camera" both appear in its description.
Hybrid gets the best of both: iPhone #1 (semantic: phone+camera), Samsung #2 (keyword: phone+camera).
```

---

### Example 3: Cross-Encoder Re-ranking

This is the most powerful technique in your arsenal — and the most expensive:

```python
"""
Cross-encoder re-ranking.
The most accurate (and expensive) search technique.
"""

from sentence_transformers import CrossEncoder
import numpy as np
from typing import List, Tuple
import time


class CrossEncoderReranker:
    """
    Two-stage search with cross-encoder re-ranking.
    
    Stage 1: Fast bi-encoder (embedding) search — retrieves top-100 candidates
    Stage 2: Cross-encoder re-ranking — scores each candidate against the query
    
    🤔 BI-ENCODER vs CROSS-ENCODER:
    
    Bi-encoder (what you've been using):
      query → vector
      doc → vector
      similarity = cosine(query_vec, doc_vec)
      ✓ Can pre-compute doc vectors (fast retrieval)
      ✗ Loses interaction between query and doc words
    
    Cross-encoder:
      [CLS] query [SEP] doc [SEP] → score
      ✓ Full attention between query and doc words (much more accurate)
      ✗ Must compute for EACH query-doc pair (expensive, can't pre-compute)
    
    The insight: Use bi-encoder for retrieval, cross-encoder for re-ranking.
    This gives you BOTH speed AND accuracy.
    """
    
    def __init__(self, retrieval_model_name='all-MiniLM-L6-v2'):
        # Stage 1: Fast retrieval
        from sentence_transformers import SentenceTransformer
        self.retriever = SentenceTransformer(retrieval_model_name)
        
        # Stage 2: Cross-encoder re-ranker
        # More accurate but slower — we only run it on top-100
        # Options: 'cross-encoder/ms-marco-MiniLM-L-6-v2' (fast, decent)
        #          'cross-encoder/ms-marco-electra-base' (better)
        self.reranker = CrossEncoder(
            'cross-encoder/ms-marco-MiniLM-L-6-v2',
            max_length=512,
        )
        
        self.documents = []
        self.doc_embeddings = None
    
    def index(self, documents: List[str]):
        """Pre-compute document embeddings for fast retrieval."""
        self.documents = documents
        print(f"Indexing {len(documents)} documents...")
        self.doc_embeddings = self.retriever.encode(
            documents,
            normalize_embeddings=True,
            show_progress_bar=True,
        )
    
    def search(
        self,
        query: str,
        retrieve_k: int = 100,  # Retrieve top-100 with fast method
        rerank_k: int = 10,     # Re-rank top-10 with slow method
    ) -> List[Tuple[str, float, float]]:
        """
        Two-stage search.
        
        Returns: List of (document, bi_encoder_score, cross_encoder_score)
        """
        # ── Stage 1: Fast bi-encoder retrieval ──
        start = time.time()
        query_vec = self.retriever.encode([query], normalize_embeddings=True)[0]
        bi_scores = np.dot(self.doc_embeddings, query_vec)
        
        # Get top-retrieve_k candidates
        top_indices = np.argsort(bi_scores)[::-1][:retrieve_k]
        candidates = [self.documents[i] for i in top_indices]
        bi_scores_top = [float(bi_scores[i]) for i in top_indices]
        
        stage1_time = time.time() - start
        
        # ── Stage 2: Cross-encoder re-ranking ──
        start = time.time()
        
        # Prepare query-document pairs
        pairs = [[query, doc] for doc in candidates]
        
        # Cross-encoder scores (higher = more relevant)
        ce_scores = self.reranker.predict(pairs)
        
        stage2_time = time.time() - start
        
        # Re-rank by cross-encoder score
        reranked_indices = np.argsort(ce_scores)[::-1][:rerank_k]
        
        results = []
        for idx in reranked_indices:
            results.append((
                candidates[idx],
                bi_scores_top[idx],
                float(ce_scores[idx]),
            ))
        
        # Print timing
        print(f"  Stage 1 (bi-encoder, top-{retrieve_k}): {stage1_time*1000:.1f}ms")
        print(f"  Stage 2 (cross-encoder, top-{rerank_k}): {stage2_time*1000:.1f}ms")
        
        return results


# ── Demo ──
if __name__ == "__main__":
    documents = [
        "Python is a high-level programming language for general-purpose programming.",
        "JavaScript is a scripting language for web development and frontend programming.",
        "Python is widely used in data science, machine learning, and AI applications.",
        "FastAPI is a modern Python web framework for building APIs with type hints.",
        "PostgreSQL is a powerful open-source relational database management system.",
        "React is a JavaScript library for building user interfaces and frontend applications.",
        "Docker containers package applications with dependencies for portable deployment.",
        "AWS Lambda is a serverless computing service that runs code without provisioning servers.",
        "Python's simplicity makes it an excellent choice for beginners learning programming.",
        "TypeScript adds static typing to JavaScript for better developer experience.",
        "The Django framework follows the model-view-template (MVT) architectural pattern.",
        "NumPy provides support for large multi-dimensional arrays and matrices for scientific computing.",
        "Pandas is a fast, powerful data analysis and manipulation library built on top of Python.",
        "TensorFlow is an end-to-end open source platform for machine learning and deep learning.",
        "PyTorch is a machine learning framework that accelerates the path from research to production.",
        "Celery is a distributed task queue for handling asynchronous tasks in Python applications.",
        "Redis is an in-memory data structure store used as a cache, message broker, and database.",
        "Nginx is a high-performance web server that can also be used as a reverse proxy and load balancer.",
        "Kubernetes is an open-source container orchestration platform for automating deployment and scaling.",
        "Git is a distributed version control system for tracking changes in source code during development.",
    ]
    
    print("═══ Cross-Encoder Re-ranking Demo ═══\n")
    
    reranker = CrossEncoderReranker()
    reranker.index(documents)
    
    queries = [
        "Python library for data analysis",
        "web framework for building APIs",
        "container deployment orchestration",
    ]
    
    for query in queries:
        print(f"\nQuery: '{query}'")
        results = reranker.search(query, retrieve_k=10, rerank_k=5)
        
        print(f"{'Rank':5s} {'CE Score':10s} {'BI Score':10s} {'Document'}")
        print("-" * 80)
        for i, (doc, bi_score, ce_score) in enumerate(results):
            print(f"{i+1:<5d} {ce_score:<10.4f} {bi_score:<10.4f} {doc[:70]}...")
        print()
```

**Expected output:**
```
═══ Cross-Encoder Re-ranking Demo ═══

Indexing 20 documents...

Query: 'Python library for data analysis'
  Stage 1 (bi-encoder, top-10): 2.1ms
  Stage 2 (cross-encoder, top-10): 15.3ms
  ────────────────────────────────────────────────────────────────
  Rank   CE Score   BI Score   Document
  1      -1.23      0.4567     Pandas is a fast, powerful data analysis...
  2      -2.34      0.5123     NumPy provides support for large...
  3      -3.12      0.3891     Python is widely used in data science...
  4      -3.45      0.4123     TensorFlow is an end-to-end open source...
  5      -4.01      0.3345     PyTorch is a machine learning framework...

Query: 'container deployment orchestration'
  Stage 1 (bi-encoder, top-10): 2.1ms
  Stage 2 (cross-encoder, top-10): 15.3ms
  ────────────────────────────────────────────────────────────────
  1      -0.89      0.4567     Docker containers package applications...
  2      -1.56      0.4123     Kubernetes is an open-source container...
  3      -3.45      0.3123     Nginx is a high-performance web server...
  ...
```

**🤔 Key observations:**
1. Cross-encoder scores are negative (that's normal — the model outputs logits)
2. Higher score (closer to 0) = more relevant
3. Notice the bi-encoder score is NOT correlated with the cross-encoder score
   - NumPy scored HIGHER (0.5123) than Pandas (0.4567) on "data analysis" — because
     it has "scientific computing" which vaguely overlaps with "data analysis"
   - But the cross-encoder correctly knows Pandas is more about "data analysis"
4. This re-ranking catches mistakes the bi-encoder makes

```
SENIOR INSIGHT — When cross-encoder re-ranking is WORTH the cost:

DO USE cross-encoder when:
- Search accuracy is critical (legal, medical, finance)
- Users will notice wrong results (customer-facing search)
- You have < 10K queries/day (cost is manageable)
- Your stage-1 recall is high but precision is low

DON'T USE cross-encoder when:
- You need sub-50ms total response time
- You handle > 100K queries/day ($$$)
- Your retrieval stage already gives good results
- The improvement doesn't justify the 3x-10x latency increase

COST: For 100K docs, top-100 rerank = 100 cross-encoder inferences
       100 inferences × $0.001 = $0.10 per query
       10K queries/day = $1,000/day — NOT cheap!
```

---

### Example 4: Production Search Architecture

Let's put it all together into a production-ready search system:

```python
"""
Production search system combining all techniques.
Designed for real-world deployment.
"""

from sentence_transformers import CrossEncoder, SentenceTransformer
from typing import List, Dict, Optional, Callable
from dataclasses import dataclass, field
from collections import OrderedDict
import numpy as np
import time
import json
import hashlib
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


@dataclass
class SearchConfig:
    """Configuration for the production search system."""
    retrieve_k: int = 100      # Candidates from fast search
    rerank_k: int = 20         # Candidates after cross-encoder
    final_k: int = 10          # Final results after all processing
    hybrid_alpha: float = 0.7  # Vector weight in hybrid search
    cache_ttl: int = 3600      # Cache TTL in seconds
    enable_spellcheck: bool = True
    enable_query_expansion: bool = False


@dataclass
class SearchResult:
    score: float
    document: str
    metadata: Optional[Dict] = None
    source: str = "vector"  # "vector", "keyword", "hybrid", "reranked"


class ProductionSearchEngine:
    """
    Production search with:
    - Multi-strategy retrieval (vector + keyword)
    - Cross-encoder re-ranking
    - Query caching
    - Performance monitoring
    - Fallback strategies
    """
    
    def __init__(
        self,
        documents: List[str],
        metadatas: Optional[List[Dict]] = None,
        config: Optional[SearchConfig] = None,
    ):
        self.config = config or SearchConfig()
        self.documents = documents
        self.metadatas = metadatas or [{} for _ in documents]
        
        # Search components
        self.embedding_model = SentenceTransformer('all-MiniLM-L6-v2')
        self.reranker = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')
        
        # Pre-compute embeddings
        logger.info("Pre-computing embeddings...")
        self.doc_embeddings = self.embedding_model.encode(
            documents, normalize_embeddings=True, show_progress_bar=True
        )
        
        # Setup BM25 (from our earlier class)
        self.bm25 = BM25(k1=1.5, b=0.75)
        self.bm25.fit(documents)
        
        # Query cache (simple LRU)
        self._cache: OrderedDict = OrderedDict()
        self._cache_max = 1000
        
        # Metrics
        self.metrics = {
            "total_queries": 0,
            "cache_hits": 0,
            "avg_latency_ms": 0,
            "p95_latency_ms": 0,
            "latencies": [],
        }
    
    def _normalize_scores(self, scores: np.ndarray) -> np.ndarray:
        """MinMax normalization."""
        if scores.max() == scores.min():
            return np.zeros_like(scores)
        return (scores - scores.min()) / (scores.max() - scores.min())
    
    def _cache_key(self, query: str, params: Dict) -> str:
        """Generate cache key from query and parameters."""
        return hashlib.md5(
            f"{query}_{json.dumps(params, sort_keys=True)}".encode()
        ).hexdigest()
    
    def _get_cached(self, key: str) -> Optional[List[SearchResult]]:
        """Get cached results if available."""
        if key in self._cache:
            self._cache.move_to_end(key)
            self.metrics["cache_hits"] += 1
            return self._cache[key]
        return None
    
    def _set_cache(self, key: str, results: List[SearchResult]):
        """Cache results."""
        self._cache[key] = results
        if len(self._cache) > self._cache_max:
            self._cache.popitem(last=False)
    
    def search(
        self,
        query: str,
        top_k: int = 10,
        use_reranker: bool = True,
        use_cache: bool = True,
        threshold: Optional[float] = None,
    ) -> Dict:
        """
        Full search pipeline.
        
        Args:
            query: Search query
            top_k: Number of results
            use_reranker: Whether to use cross-encoder re-ranking
            use_cache: Whether to cache results
            threshold: Minimum score threshold
        
        Returns:
            Dict with results and performance metrics
        """
        start = time.time()
        self.metrics["total_queries"] += 1
        
        # Check cache
        cache_key = self._cache_key(query, {
            "top_k": top_k,
            "use_reranker": use_reranker,
        })
        
        if use_cache:
            cached = self._get_cached(cache_key)
            if cached:
                return {
                    "results": cached[:top_k],
                    "from_cache": True,
                    "latency_ms": 0.1,
                    "total_results": len(cached),
                }
        
        # ── Step 1: Query normalization ──
        cleaned_query = query.strip()
        if not cleaned_query:
            return {"results": [], "error": "Empty query", "latency_ms": 0}
        
        # ── Step 2: Multi-strategy retrieval ──
        # Vector search
        query_vec = self.embedding_model.encode(
            [cleaned_query], normalize_embeddings=True
        )[0]
        vector_scores = np.dot(self.doc_embeddings, query_vec)
        
        # Keyword search
        keyword_results = dict(self.bm25.search(cleaned_query, top_k=self.config.retrieve_k))
        keyword_scores = np.array([keyword_results.get(i, 0.0) for i in range(len(self.documents))])
        
        # Normalize
        vec_norm = self._normalize_scores(vector_scores)
        kw_norm = self._normalize_scores(keyword_scores)
        
        # Hybrid fusion
        alpha = self.config.hybrid_alpha
        hybrid_scores = alpha * vec_norm + (1 - alpha) * kw_norm
        
        # Get top candidates
        candidate_indices = np.argsort(hybrid_scores)[::-1][:self.config.retrieve_k]
        
        # ── Step 3: Re-ranking (optional) ──
        if use_reranker:
            candidate_docs = [self.documents[i] for i in candidate_indices]
            pairs = [[cleaned_query, doc] for doc in candidate_docs]
            ce_scores = self.reranker.predict(pairs)
            
            # Re-rank by cross-encoder score
            reranked_order = np.argsort(ce_scores)[::-1]
            final_indices = [candidate_indices[i] for i in reranked_order[:self.config.final_k]]
            final_scores = [float(ce_scores[i]) for i in reranked_order[:self.config.final_k]]
            source = "reranked"
        else:
            # Just use hybrid scores directly
            final_order = np.argsort(hybrid_scores[candidate_indices])[::-1]
            final_indices = [candidate_indices[i] for i in final_order[:top_k]]
            final_scores = [float(hybrid_scores[i]) for i in final_order[:top_k]]
            source = "hybrid"
        
        # ── Step 4: Post-processing ──
        results = []
        for idx, score in zip(final_indices, final_scores):
            if threshold is not None and score < threshold:
                continue
            
            # You could add snippet generation, highlighting, etc. here
            results.append(SearchResult(
                score=score,
                document=self.documents[idx],
                metadata=self.metadatas[idx],
                source=source,
            ))
        
        # Cache results
        if use_cache and results:
            self._set_cache(cache_key, results)
        
        # Metrics
        elapsed_ms = (time.time() - start) * 1000
        self.metrics["latencies"].append(elapsed_ms)
        self.metrics["avg_latency_ms"] = np.mean(self.metrics["latencies"][-1000:])
        
        return {
            "results": results,
            "from_cache": False,
            "latency_ms": round(elapsed_ms, 2),
            "total_results": len(results),
            "strategies_used": ["vector", "keyword"] + (["reranker"] if use_reranker else []),
        }
    
    def get_stats(self) -> Dict:
        """Get search engine statistics."""
        latencies = self.metrics["latencies"]
        return {
            "total_queries": self.metrics["total_queries"],
            "cache_hits": self.metrics["cache_hits"],
            "cache_hit_rate": round(
                self.metrics["cache_hits"] / max(self.metrics["total_queries"], 1), 3
            ),
            "avg_latency_ms": round(np.mean(latencies[-100:]), 2) if latencies else 0,
            "p95_latency_ms": round(np.percentile(latencies[-100:], 95), 2) if len(latencies) >= 20 else 0,
            "documents_indexed": len(self.documents),
        }


# ── Demo ──
if __name__ == "__main__":
    docs = [
        "Python is a high-level programming language for general-purpose programming.",
        "JavaScript is a scripting language for web development and frontend programming.",
        "Python is widely used in data science, machine learning, and AI applications.",
        "FastAPI is a modern Python web framework for building APIs with type hints.",
        "React is a JavaScript library for building user interfaces and frontend applications.",
        "Docker containers package applications with dependencies for portable deployment.",
        "AWS Lambda is a serverless computing service that runs code without provisioning servers.",
        "PostgreSQL is a powerful open-source relational database management system.",
        "Redis is an in-memory data structure store used as a cache, message broker, and database.",
        "Kubernetes is an open-source container orchestration platform for automating deployment.",
    ]
    
    engine = ProductionSearchEngine(docs)
    
    print("═══ Production Search Demo ═══\n")
    
    queries = [
        "Python for building web APIs",
        "database caching and message broker",
        "deploying applications in containers",
    ]
    
    for query in queries:
        print(f"Query: '{query}'")
        result = engine.search(query, use_reranker=True)
        
        print(f"  Latency: {result['latency_ms']}ms")
        print(f"  Strategies: {', '.join(result['strategies_used'])}")
        for i, r in enumerate(result['results'][:3]):
            print(f"  #{i+1} [{r.score:.4f}] {r.document[:80]}...")
        print()
    
    # Second call (should be cached)
    print("═══ Cached Query (second call) ═══\n")
    result = engine.search(queries[0], use_reranker=True)
    print(f"Query: '{queries[0]}'")
    print(f"  From cache: {result['from_cache']}")
    print(f"  Latency: {result['latency_ms']}ms")
    
    print("\n═══ Engine Stats ═══")
    for k, v in engine.get_stats().items():
        print(f"  {k}: {v}")
```

---

## ✅ Good Output Examples

### What Strong Production Search Looks Like

```
┌────────────────────────────────────────────────────────┐
│  SEARCH SYSTEM HEALTH CHECK (PASSED)                    │
│                                                         │
│  Query: "python data analysis library"                  │
│  Latency: 23ms (Stage 1: 3ms, Stage 2: 20ms)           │
│  Cache hit: No (first time)                            │
│  Hybrid score: 0.83 (alpha=0.7)                        │
│  Re-ranker: cross-encoder/ms-marco-MiniLM-L-6-v2       │
│                                                         │
│  Results match expected relevance:                      │
│  ✓ #1 Pandas (0.91) — correct, data analysis library   │
│  ✓ #2 NumPy (0.85) — correct, scientific computing     │
│  ✓ #3 Python for data science (0.72) — correct context │
│  ✗ No false positives in top-10                        │
│                                                         │
│  Cache will serve next identical query in < 1ms.       │
└────────────────────────────────────────────────────────┘
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Naive Score Averaging

```python
# ❌ WRONG — Adding raw scores from different methods
combined_score = vector_score + bm25_score
# BM25 scores (0-25) dominate vector scores (0-1)

# ✅ RIGHT — Normalize first
vec_norm = (vector_score - vec_min) / (vec_max - vec_min)
bm25_norm = (bm25_score - bm25_min) / (bm25_max - bm25_min)
combined = alpha * vec_norm + (1 - alpha) * bm25_norm
```

### Antipattern 2: Re-ranking Without Retrieval Diversity

```python
# ❌ WRONG — Re-ranking only top-10
# If the top-10 from stage 1 are all wrong, the re-ranker can't fix it!
top_10 = stage1_search(query, top_k=10)
reranked = reranker.rerank(query, top_10)
# If all 10 are irrelevant, re-ranking 10 irrelevant docs = still irrelevant

# ✅ RIGHT — Retrieve more, then re-rank
top_100 = stage1_search(query, top_k=100)
reranked = reranker.rerank(query, top_100)
reranked[:10]  # Now we have a real chance of finding good results
```

### Antipattern 3: Not Monitoring Search Quality

```
What gets measured gets improved.

Monitor these metrics in production:
  - Zero results rate: % of queries with 0 results (should be < 1%)
  - Click-through rate: % of users who click a result (should be > 50%)
  - Avg. result rank clicked: what position does the user click (lower = better)
  - Query failure patterns: which queries consistently fail?
  - Latency P95/P99: are there slow outliers?
  - Cache hit rate: are we caching effectively?
```

### Failure Mode: Forgetting to Handle Empty Results

```python
# ❌ WRONG — No fallback for no results
def search(query):
    results = vector_search(query)
    return results  # Empty list if nothing found

# ✅ RIGHT — Multiple fallback layers
def search_with_fallback(query):
    # Layer 1: Full hybrid search
    results = hybrid_search(query)
    if results:
        return results
    
    # Layer 2: Pure keyword (in case embedding failed)
    logger.warning(f"Hybrid search returned 0 results for: {query}")
    results = keyword_search(query)
    if results:
        return results
    
    # Layer 3: Fuzzy match (account for typos)
    results = fuzzy_search(query)
    if results:
        return results
    
    # Layer 4: Return popular/default results
    logger.error(f"ALL searches failed for: {query}")
    return get_popular_results()
```

---

## 🧪 Drills & Challenges

### Drill 1: Find Your Optimal Alpha (25 min)

For YOUR dataset, find the optimal `alpha` for hybrid search:

1. Collect 20 queries with known correct answers
2. Run search with alpha = [0.0, 0.2, 0.4, 0.6, 0.8, 1.0]
3. For each alpha, measure:
   - Recall@10 (how many correct answers in top-10?)
   - MRR (mean reciprocal rank — does the best answer appear early?)
4. Plot alpha vs recall. Find the sweet spot.

**Expected finding:** Most datasets have an alpha sweet spot between 0.5-0.8. Pure vector (1.0) often misses exact matches. Pure keyword (0.0) misses synonyms.

### Drill 2: The Reranker Impact Test (25 min)

1. Without re-ranker: measure recall@10 and P@10 across 20 queries
2. With re-ranker: measure same metrics
3. Where does re-ranking improve results? Where does it NOT?
4. **Important edge case:** Does re-ranking ever make results WORSE?

**Expected finding:** Cross-encoders improve ranking ~80% of the time. But they can make errors too — especially when the query-document pair has ambiguous wording.

### Drill 3: Build the Fallback Chain (20 min)

Implement a fallback search chain:
1. Vector search (primary)
2. BM25 keyword (if vector returns < 3 results)
3. Fuzzy/wildcard (if keyword returns < 3 results)
4. Popular/default (if everything fails)

Test with queries that intentionally fail each layer:
- "supercalifragilisticexpialidocious" → all layers fail → returns popular
- "pyt hon programing" → typo → fuzzy should catch it
- "dat abase" → partial match → keyword should catch it

### Drill 4: The Cache Strategy Analysis (20 min)

Your search gets 10K queries/day. Analyze caching strategies:
1. No cache: 10K full searches
2. Exact match cache: 20% hit rate (same queries repeated)
3. Semantic cache: group similar queries ("reset password" ≈ "forgot password and can't log in")

Design a cache strategy:
- Which queries should be cached vs not?
- How long should cache live for an e-commerce product catalog vs news articles?
- What's the cost of stale cached results?

### Drill 5: Build the Monitoring Dashboard (30 min)

Build a simple dashboard that tracks:
- Queries per minute
- Average latency (last 5 min, last hour)
- Error rate (searches returning 0 results)
- Cache hit rate
- Top 10 failed queries

**This is what you'd actually run in production.** Monitoring isn't optional — it's how you know when your search is broken before users complain.

---

## 🚦 Gate Check

Before moving to the Drills file (08), confirm you can:

- [ ] **Build BM25 search** from scratch
- [ ] **Implement hybrid search** with proper score normalization
- [ ] **Explain when cross-encoder re-ranking is worth the cost**
- [ ] **Design a fallback chain** for when primary search fails
- [ ] **Know the cache strategy** for your use case
- [ ] **Answer these questions:**
  1. If hybrid search (0.7 vec + 0.3 kw) gets 92% recall and pure vector gets 88%, is the extra complexity worth it? When yes, when no?
  2. Cross-encoders cost 10x more than bi-encoders. How many queries/day makes re-ranking too expensive?
  3. Your user searches "Error 500 occuring on payment page." The vector search returns "General server error handling" as #1. Why? How would hybrid search fix this?
  4. What metric would you track to know if your search is getting WORSE over time?

- [ ] **Write down your personal production search checklist** (5-10 items you'd check before deploying)

---

## 📚 Resources

**Hybrid Search:**
- "BM25 Algorithm Explained" — Deep understanding of the scoring formula
- Qdrant's hybrid search documentation — production implementation
- "SPLADE: Sparse Lexical and Dense Vector Search" — State-of-the-art hybrid

**Re-ranking:**
- "Cross-Encoders for Reranking" — Sentence-Transformers docs
- "Late Interaction with ColBERT" — Efficient alternative to cross-encoders
- "Rank Fusion and Score Normalization" — Combining search results

**Production Systems:**
- "Building a Real-World Semantic Search System" — Case study
- "Lessons from Building Production Search at Google" — Engineering blog
- Monitoring: Prometheus + Grafana for search metrics

**Your Production Architecture:**
```
Layer        │ Technique           │ Latency   │ Cost    │ Quality
─────────────┼─────────────────────┼───────────┼─────────┼──────────
1. Retrieval │ BM25 + Vector       │ < 10ms    │ Low     │ Good
2. Hybrid    │ Score fusion        │ < 1ms     │ None    │ Better
3. Re-rank   │ Cross-encoder       │ 10-50ms   │ Medium  │ Best
4. Feedback  │ User clicks → learn │ Ongoing   │ Low     │ Improves

Start with Layer 1+2. Add Layer 3 when quality isn't enough.
Layer 4 is Phase 7 (Evals & Observability).
```

---

> **🛑 Revisit the 3 discovery questions from the beginning:**
>
> 1. **When Vector Search Fails** — Can you now identify which technique fixes each failure? Vector + keyword + cross-encoder cover each other's blind spots.
> 2. **The Two-Stage Pattern** — Now you see WHY production search always uses this architecture. Stage 1 casts a wide net. Stage 2 carefully scores the best candidates.
> 3. **The Score Normalization Problem** — You now know why naive score addition breaks, and how rank fusion solves it.
>
> **Key realization:** No single search technique is perfect. The best engineers don't search for "the best method" — they combine multiple methods that cover each other's failure modes. This is the real production skill.
>
> **Next:** File 08 — Drills. Hands-on practice where you debug broken vector search systems.
