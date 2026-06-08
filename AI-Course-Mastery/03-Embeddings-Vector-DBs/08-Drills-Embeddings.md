# Phase 3 Drills — Embeddings, Search & Vector Databases

## 🎯 Purpose & Goals

This file is different from the previous ones. **There's no teaching here — only doing.** Each drill is a self-contained challenge that tests what you've learned across Files 01-07.

**⏱ Time Budget:** 1.5 hours total (pick 3-4 drills, or do all of them)

---

## 🔧 Drill 1: Debug the Broken Search Engine

The code below has 7 bugs. Your job: find and fix all of them.

```python
"""
BROKEN SEARCH ENGINE — Find and fix 7 bugs.
This code looks correct but produces terrible results.
"""

import numpy as np
from sentence_transformers import SentenceTransformer
from sklearn.metrics.pairwise import cosine_similarity

documents = [
    "Python is a programming language",
    "JavaScript is for web development",
    "Docker containers for deployment",
]

# Bug 1: Missing method call?
model = SentenceTransformer("all-MiniLM-L6-v2")

def search(query, docs):
    """
    Bug 2: What's wrong with this embedding?
    Bug 3: What's wrong with this similarity calculation?
    Bug 4: Why are the results always in the same order?
    
    Hint: There are MULTIPLE bugs in this function.
    """
    # Embed query
    query_vec = model.encode(query)
    
    # Bug: document embeddings computed EVERY search (slow!)
    doc_vecs = model.encode(docs)
    
    # Bug: what's wrong with this similarity?
    scores = []
    for doc_vec in doc_vecs:
        sim = np.dot(query_vec, doc_vec)  # Bug 3?
        scores.append(sim)
    
    # Sort
    ranked = sorted(zip(scores, docs), key=lambda x: x[0])
    return ranked

# Test
query = "web frameworks for building applications"
results = search(query, documents)

for score, doc in results:
    print(f"{score:.4f}: {doc}")

# Bug 5: The output shows...
# Bug 6: Performance issue with re-embedding
# Bug 7: Sorting order (ascending vs descending)
```

**Expected output (after fixing all bugs):**
```
✅ JavaScript is for web development  (score should be HIGH)
✅ Python is a programming language  (medium score)
✅ Docker containers for deployment  (lowest score)
```

**Fix all 7 bugs, then add:**
- Normalize embeddings
- Pre-compute document embeddings in the constructor
- Cosine similarity instead of raw dot product
- Sort descending (highest similarity first)

---

## 🔧 Drill 2: The Metadata Filter Mystery

This ChromaDB code should work but doesn't. Find the bugs:

```python
"""
BROKEN CHROMA DB CODE — Metadata filtering doesn't work.
"""

import chromadb
from chromadb.config import Settings

client = chromadb.Client(Settings(anonymized_telemetry=False))

# Bug 1: What's wrong with the collection creation?
collection = client.create_collection("test")

# Add documents
documents = [
    "Python is great for data science",
    "JavaScript powers the web",
    "SQL databases store data",
]

metadatas = [
    {"category": "programming", "difficulty": "beginner"},
    {"category": "programming", "difficulty": "intermediate"},
    {"category": "database", "difficulty": "advanced"},
]

# Bug 2: Why aren't embeddings provided?
# Bug 3: IDs can't be integers?
collection.add(
    documents=documents,
    metadatas=metadatas,
    ids=[0, 1, 2],  # Bug?
)

# Bug 4: Is this filtering syntax correct?
results = collection.query(
    query_texts=["coding languages"],
    n_results=5,
    where={"category": {"$eq": "programming"}},  # Bug 4?
)

print(f"Results: {len(results['documents'][0])}")
for doc, meta in zip(results['documents'][0], results['metadatas'][0]):
    print(f"  {doc} — {meta}")
```

**Find and fix all bugs.** Expected: only "programming" category documents returned.

---

## 🔧 Drill 3: The Performance Showdown

Benchmark 3 indexing strategies and find which is fastest:

```python
"""
Performance drill: Find the fastest way to index 10K vectors.
"""

import numpy as np
import time

n_docs = 10_000
dim = 384

# Generate test data
np.random.seed(42)
vectors = np.random.randn(n_docs, dim).astype(np.float32)
ids = [f"doc_{i}" for i in range(n_docs)]

# Strategy 1: One-by-one
def index_one_by_one():
    start = time.time()
    # Your code: add one document at a time to ChromaDB/Qdrant
    ...
    return time.time() - start

# Strategy 2: Batch of 100
def index_batch_100():
    start = time.time()
    # Your code: batch 100 at a time
    ...
    return time.time() - start

# Strategy 3: Single batch
def index_single_batch():
    start = time.time()
    # Your code: all 10K at once
    ...
    return time.time() - start

# Run and compare
for name, fn in [("One-by-one", index_one_by_one), 
                  ("Batch 100", index_batch_100),
                  ("Single batch", index_single_batch)]:
    elapsed = fn()
    print(f"{name:15s}: {elapsed:.2f}s")
```

**Expected finding:** One-by-one is 100x slower than batch. Write down the actual numbers.

---

## 🔧 Drill 4: The Embedding Space Explorer

Your task: Find the "weird" vectors in an embedding space.

```python
"""
Find outlier vectors in an embedding space.
"""

import numpy as np
from sentence_transformers import SentenceTransformer

# These documents are mostly about technology...
documents = [
    "Python programming for data science",
    "JavaScript frameworks like React and Vue",
    "Docker container deployment strategies",
    "SQL database optimization techniques",
    "AWS cloud infrastructure services",
    "Machine learning model training pipelines",
    # ... but someone mixed in these:
    "The cat sat on the mat and purred",
    "Chocolate chip cookie recipe with nuts",
    "How to train your dragon book review",
    "World Cup 2026 qualifying matches",
]

model = SentenceTransformer('all-MiniLM-L6-v2')
embeddings = model.encode(documents)

# Task 1: Find the 3 documents that don't belong (outliers)
# Hint: Find documents whose nearest neighbor is FARTHER than average

# Task 2: For each outlier, identify which category it belongs to
# (animals, food, books, sports)

# Task 3: If you were building a search system, how would you
# automatically detect and handle these outliers?
```

**Challenge:** Don't look at the documents list — find the outliers purely from the embedding geometry.

---

## 🔧 Drill 5: The Cross-Database Migration

You have a search engine running on ChromaDB. Migrate it to Qdrant (or pgvector).

```python
"""
Migrate from ChromaDB to Qdrant.
Same API, different backend.
"""

# EXISTING ChromaDB implementation
class ChromaSearch:
    def __init__(self):
        self.client = chromadb.Client()
        self.collection = self.client.create_collection("docs")
    
    def add(self, texts, ids):
        embeddings = model.encode(texts).tolist()
        self.collection.add(embeddings=embeddings, documents=texts, ids=ids)
    
    def query(self, text, top_k=5):
        embedding = model.encode([text])[0].tolist()
        results = self.collection.query(
            query_embeddings=[embedding], n_results=top_k
        )
        return results

# YOUR TASK: Implement the SAME API but backed by Qdrant
class QdrantSearch:
    def __init__(self):
        # Your code here
        ...
    
    def add(self, texts, ids):
        # Your code here
        ...
    
    def query(self, text, top_k=5):
        # Your code here
        ...
```

**Success criteria:** `QdrantSearch` returns the SAME results as `ChromaSearch` for any query.

---

## 🔧 Drill 6: The Hybrid Search Tuner

Find the optimal alpha for YOUR data:

```python
"""
Find the best hybrid search weight for your specific use case.
"""

import numpy as np

# You'll need:
# - BM25 implementation (from File 07)
# - Embedding model
# - A set of documents with known relevance judgments

documents = [...]  # Your documents
queries = [...]    # Your test queries  
relevant = [[0, 1], [2, 3], ...]  # Relevant doc indices for each query

def evaluate_alpha(alpha, documents, queries, relevant):
    """
    Evaluate hybrid search with a given alpha.
    
    Returns: average recall@10 across all queries.
    """
    # 1. Build BM25 index
    # 2. Pre-compute embeddings
    # 3. For each query:
    #    a. Get vector scores
    #    b. Get BM25 scores
    #    c. Normalize and combine: score = alpha * vec + (1-alpha) * bm25
    #    d. Check if relevant docs are in top-10
    # 4. Return average recall
    ...

# Test alphas from 0.0 to 1.0
alphas = np.linspace(0, 1, 11)
results = {}

for alpha in alphas:
    recall = evaluate_alpha(alpha, documents, queries, relevant)
    results[alpha] = recall
    print(f"alpha={alpha:.1f}: recall={recall:.3f}")

best_alpha = max(results, key=results.get)
print(f"\nBest alpha: {best_alpha:.1f} (recall: {results[best_alpha]:.3f})")
```

**Expected finding:** The optimal alpha depends ENTIRELY on your data. Technical/specialized domains get lower optimal alpha (keyword helps more). General/conceptual domains get higher optimal alpha (vector helps more).

---

## 🔧 Drill 7: Failure Mode Collection (Phase Summary)

Collect ONE real example of each failure mode. If you can't find one from your experiments, write a synthetic example:

### 1. Embedding Failure
A pair of sentences that are similar according to embeddings but SHOULDN'T be:
```
Query: "..."
Document that ranks highly: "..."
Why it's wrong: "..."
```

### 2. Keyword Failure
A query where pure keyword search completely fails:
```
Query: "..."
What it should find: "..."
What keyword returns: "..."
```

### 3. Cross-encoder Failure
A query where re-ranking makes things WORSE:
```
Query: "..."
Top-3 before re-ranking: "..."
Top-3 after re-ranking: "..."
Why re-ranking failed: "..."
```

### 4. Scale Failure
A scenario where your chosen vector DB (ChromaDB) can't handle the load:
```
Document count: "..."
Query concurrency: "..."
What breaks: "..."
```

### 5. Normalization Failure
A query where naive score fusion (no normalization) produces wrong rankings:
```
Query: "..."
Scores without normalization: "..."
Scores with normalization: "..."
```

**Why this matters:** Collecting failure modes is how you build intuition. After you've seen each failure once, you'll recognize it before it happens in production.

---

## 🔧 Drill 8: The "No-Frameworks" Challenge

Build semantic search WITHOUT using any of these:
- `sentence-transformers`
- `chromadb`
- `qdrant-client`
- `pinecone-client`
- `psycopg2` / `pgvector`

Allowed:
- `numpy` (for basic math)
- Your own embedding model (from scratch or a tiny file)
- Basic Python

```python
"""
NO-FRAMEWORKS CHALLENGE.
Build search with only numpy and Python stdlib.
"""

import numpy as np

# Step 1: Build a tiny embedding model
# Use word co-occurrence (like File 01's mini embedding)
# or: load pre-computed GloVe word vectors from a text file

# Step 2: Sentence embedding by averaging word vectors
def sentence_to_vec(sentence, word_vectors):
    words = sentence.lower().split()
    vectors = [word_vectors[w] for w in words if w in word_vectors]
    if not vectors:
        return np.zeros(50)  # Return zero vector if no words found
    return np.mean(vectors, axis=0)

# Step 3: Search
def search(query, documents, word_vectors):
    # YOUR CODE HERE
    # No numpy allowed for distance (just kidding, numpy is allowed)
    pass

# Test with a tiny word vector file
# (Download glove.6B.50d.txt or create your own)
```

**This is the ultimate test of understanding.** If you can build search from scratch with only numpy, you truly understand embeddings.

---

## 📊 Drill Progress Tracker

| # | Drill | Time | Completed? | Key Insight |
|---|-------|------|------------|------------|
| 1 | Debug Search | 15min | ☐ | |
| 2 | ChromaDB Mystery | 10min | ☐ | |
| 3 | Performance Showdown | 20min | ☐ | |
| 4 | Embedding Explorer | 15min | ☐ | |
| 5 | Cross-DB Migration | 20min | ☐ | |
| 6 | Hybrid Tuner | 20min | ☐ | |
| 7 | Failure Collection | 15min | ☐ | |
| 8 | No-Frameworks | 30min | ☐ | |

**Complete at least 4 drills** before moving to the project.

---

> **🛑 Before moving to the project:**
>
> - Which drill was hardest? Why?
> - Which failure mode surprised you most?
> - Is there anything from Files 01-07 that you need to re-read?
>
> **Next:** File 09 — Project: Personal Knowledge Search Engine. Build the portfolio project that ties everything together.
