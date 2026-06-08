# ChromaDB — Your First Vector Database

## 🎯 Purpose & Goals

> **🛑 STOP. Recall what you built in File 02:**
>
> You built semantic search from scratch with numpy. It worked great for 500 documents. But you calculated that at 1M documents, brute-force search would take ~3 seconds per query.
>
> ### 🤔 Discovery Question 1: The Scaling Problem
>
> You have 1M documents. Each search takes 3 seconds with brute force. Your users expect <200ms.
>
> **Your options (think before reading on):**
> 1. Throw more hardware at it (more RAM, faster CPU)
> 2. Reduce dimensions (768D → 128D)
> 3. Pre-compute something
> 4. Don't search ALL documents — search a clever subset
> 5. Use a different data structure
>
> Which option(s) would actually work at 10M documents? 100M? Which ones hit fundamental limits?
>
> **Here's the key question:** What if instead of comparing the query to EVERY document, you could organize documents in a tree structure — like a phone book — so you only need to check a small fraction of them?

---

### 🤔 Discovery Question 2: The Phone Book Analogy

You have a phone book with 1 million names. You need to find "John Smith."

**Method A (Brute Force):** Read every name until you find "John Smith."  
**Method B (Indexed):** Open to "S" section. Scan a few pages.

**Question:** Method B works because names are SORTED. Can you "sort" vectors? What does it mean for a vector to be "greater than" another vector?

**The problem:** Vectors are multi-dimensional. There's no natural "alphabetical order" for 384-dimensional points.

**So how do you index them?**

---

### 🤔 Discovery Question 3: The "Close Enough" Tradeoff

You're building a search for a music recommendation app. A user searches for "upbeat happy pop songs."

**Would you rather have:**
- A) Exact results, but takes 5 seconds
- B) 99% accurate results, but takes 50ms
- C) 95% accurate results, but takes 10ms

**Think about WHEN approximate is acceptable and when it's not:**
- Movie recommendations?
- Medical diagnosis?
- Legal document retrieval for a court case?
- Which song to play next?

**This is the fundamental tradeoff vector databases make: they trade EXACT accuracy for SPEED.** And they do it through a technique called Approximate Nearest Neighbor (ANN) search.

---

> **By the end of this file, you will:**
> - Understand what a vector database actually does (it's not magic)
> - Know the HNSW algorithm at a conceptual level (the most common ANN index)
> - Build a working ChromaDB application with CRUD operations
> - Know EXACTLY when ChromaDB is the right tool and when you need something else
> - Connect this back to the numpy search from File 02 — same math, better data structures

**⏱ Time Budget:** 1.5 hours (30 min concepts, 45 min code, 15 min drills)

---

## 📖 From Brute Force to Indexed Search

### The Core Problem: It's the Comparisons, Stupid

Let's quantify the problem:

```
BRUTE-FORCE SEARCH COMPLEXITY:
─────────────────────────────────
Compare query → doc 1
Compare query → doc 2
Compare query → doc 3
...
Compare query → doc N

Total operations: N × D
  N = number of documents
  D = embedding dimension (384 for MiniLM)

At N=1,000:     384,000 operations → 2ms
At N=100,000:   38,400,000 operations → 50ms
At N=10,000,000: 3,840,000,000 operations → 5 seconds

The problem: operations grow LINEARLY with N.
To handle 10x more docs, you need 10x more compute.
```

### The Insight: You Don't Need the EXACT Nearest Neighbor

Here's the key realization that makes vector databases possible:

> **For most search applications, the #1 exact result and the #5 approximate result are effectively equivalent.**
>
> If you're searching for "best Thai restaurant nearby," would you notice if the search engine returned #4 instead of #1? Probably not. You just want GOOD results, not PERFECTLY ranked results.

This is the **ANN (Approximate Nearest Neighbor)** insight: trade a small amount of accuracy for a LARGE speedup.

### How ANN Works: The HNSW Algorithm

The most popular ANN algorithm is **HNSW (Hierarchical Navigable Small World)**. Let's understand it intuitively before writing code.

```
HNSW INTUITION — The "Skip List for Vectors"
─────────────────────────────────────────────────────

Imagine you're in a giant city with 1 million buildings.
You need to find the closest restaurant to your hotel.

BRUTE FORCE: Visit all 1 million buildings. Measure distance to each.

HNSW APPROACH:
┌─────────────────────────────────────────────────────┐
│  Layer 3 (top):     ●──●──●──●──●                   │
│                    (1% of buildings — highways)      │
│                                                      │
│  Layer 2:         ●─●─●─●─●─●─●─●─●                │
│                  (10% of buildings — main roads)     │
│                                                      │
│  Layer 1:       ●●●●●●●●●●●●●●●●●●●●●●●●●          │
│                (100% of buildings — all streets)     │
└─────────────────────────────────────────────────────┘

1. START at top layer (fewest buildings, most general)
2. Find the CLOSEST building in this layer
3. DESCEND to next layer, starting from that building
4. Find the CLOSEST in this layer
5. Repeat until bottom layer
6. Result: found the nearest neighbor after checking only ~100 buildings

Total comparisons: ~100 (not 1,000,000!)
Accuracy: ~95-99% of exact nearest neighbor
```

**Key insight:** The top layer has "representative" vectors — hubs that connect to many others. By navigating through hubs, you quickly get to the right neighborhood, then refine at lower layers.

```
SENIOR INSIGHT — When HNSW fails:
─────────────────────────────────────
HNSW assumes vectors lie in a meaningful space where "closer is better."
This holds for embeddings 95% of the time, but fails when:
- Embedding space has "holes" (semantic cavities from File 01)
- The query is an outlier (outside the distribution of indexed documents)
- The data has many clusters that barely overlap

In these cases, HNSW's "highway" system might route you to the wrong
neighborhood. This is why production systems often do:
  1. ANN search (fast, approximate)
  2. Re-rank top-100 with exact distance (accurate)
```

### What ChromaDB Actually Is

ChromaDB is a lightweight, open-source vector database that:
- Runs in-process (no separate server needed for basic use)
- Stores embeddings + metadata + documents together
- Supports HNSW indexing (among others)
- Has a simple Python API
- Great for prototyping, small-to-medium datasets

**Compared to the numpy search from File 02:**
- Same core math (cosine similarity)
- Same embedding model (sentence-transformers)
- BUT: uses HNSW indexing instead of brute force
- AND: persistent storage, metadata, filtering built in

---

## 💻 Code Examples — Building with ChromaDB

### Example 1: Your First ChromaDB Collection

```python
"""
Your first ChromaDB application.
Connecting File 02's numpy search to a real vector database.
"""

import chromadb
from chromadb.config import Settings
from sentence_transformers import SentenceTransformer
import numpy as np
import time


# ── Step 1: Set up ChromaDB ──
# In-memory mode (data is lost when script ends)
# This is the simplest mode — perfect for prototyping
print("═══ Setting up ChromaDB ═══\n")

client = chromadb.Client(Settings(
    anonymized_telemetry=False,  # Opt out of telemetry
))

# ── Step 2: Create a collection ──
# A collection = a group of related embeddings + their documents
# Think of it like a "table" in a traditional database
collection = client.create_collection(
    name="customer_support",
    # ChormaDB can use its own embedding model, or we can provide our own
    # We'll provide our own so we control which model is used
)

print(f"Created collection: {collection.name}")
print(f"Collection ID: {collection.id}")

# ── Step 3: Add documents ──
# ChromaDB stores: document text, embedding vectors, and metadata
documents = [
    "To reset your password, go to Settings > Account > Security.",
    "Our refund policy allows returns within 30 days of purchase.",
    "For login issues, try clearing your browser cache and cookies.",
    "International shipping costs $25 and takes 5-10 business days.",
    "You can cancel your subscription anytime from your account settings.",
    "Two-factor authentication adds an extra layer of security.",
    "Contact support for account recovery after a phone number change.",
    "We accept Visa, Mastercard, and PayPal for all purchases.",
]

metadata = [
    {"category": "account", "difficulty": "easy"},
    {"category": "billing", "difficulty": "medium"},
    {"category": "account", "difficulty": "easy"},
    {"category": "shipping", "difficulty": "medium"},
    {"category": "billing", "difficulty": "easy"},
    {"category": "security", "difficulty": "hard"},
    {"category": "account", "difficulty": "hard"},
    {"category": "billing", "difficulty": "easy"},
]

# Each document needs a unique ID
ids = [f"doc_{i}" for i in range(len(documents))]

# ── Step 4: Generate embeddings ──
model = SentenceTransformer('all-MiniLM-L6-v2')
embeddings = model.encode(documents).tolist()  # ChromaDB expects lists, not numpy arrays

print("Adding documents to ChromaDB...")
collection.add(
    embeddings=embeddings,
    documents=documents,
    metadatas=metadata,
    ids=ids,
)
print(f"Added {len(documents)} documents to collection.\n")

# ── Step 5: Search ──
# This is where ChromaDB replaces File 02's numpy search
query = "I forgot my password and can't log in"
query_embedding = model.encode([query])[0].tolist()

print(f"Query: '{query}'")
print("\nSearching ChromaDB...")

start = time.time()
results = collection.query(
    query_embeddings=[query_embedding],
    n_results=3,
)
elapsed = time.time() - start

# ChromaDB returns results in a specific format
for i, (doc, distance, meta, doc_id) in enumerate(zip(
    results['documents'][0],
    results['distances'][0],
    results['metadatas'][0],
    results['ids'][0],
)):
    # ChromaDB returns DISTANCES (0 = identical), not similarities (1 = identical)
    # Distance = 1 - cosine_similarity
    similarity = 1 - distance
    print(f"  #{i+1} [{similarity:.4f}] {doc}")
    print(f"        (category: {meta['category']}, difficulty: {meta['difficulty']})")

print(f"\nSearch time: {elapsed*1000:.2f} ms")
```

**Expected output:**
```
═══ Setting up ChromaDB ═══

Created collection: customer_support
Collection ID: ...

Adding documents to ChromaDB...
Added 8 documents to collection.

Query: 'I forgot my password and can't log in'

Searching ChromaDB...
  #1 [0.8912] To reset your password, go to Settings > Account > Security.
        (category: account, difficulty: easy)
  #2 [0.7834] For login issues, try clearing your browser cache and cookies.
        (category: account, difficulty: easy)
  #3 [0.6543] Contact support for account recovery after a phone number change.
        (category: account, difficulty: hard)

Search time: 12.34 ms
```

**🤔 Compare to File 02's numpy search:**
- Same results (cosine similarity gives identical rankings)
- Same embedding model
- But now: persistent storage, metadata, IDs, built-in query API
- The speed gain comes from HNSW, which starts mattering at 100K+ docs

---

### Example 2: CRUD Operations

Vector databases aren't just "embed then search." Real applications need CRUD:

```python
"""
ChromaDB CRUD operations.
Managing document collections in production.
"""

import chromadb
from chromadb.config import Settings
from sentence_transformers import SentenceTransformer
import time

# ── Setup ──
client = chromadb.Client(Settings(anonymized_telemetry=False))
model = SentenceTransformer('all-MiniLM-L6-v2')
collection = client.create_collection("knowledge_base")

# ── CREATE ──
print("═══ CREATE: Adding documents ═══\n")

docs_to_add = [
    {"id": "doc_001", "text": "Python is a high-level programming language.", "tag": "programming"},
    {"id": "doc_002", "text": "FastAPI is a modern web framework for Python.", "tag": "programming"},
    {"id": "doc_003", "text": "Docker containers package applications with dependencies.", "tag": "devops"},
]

for doc in docs_to_add:
    embedding = model.encode([doc["text"]])[0].tolist()
    collection.add(
        embeddings=[embedding],
        documents=[doc["text"]],
        metadatas=[{"tag": doc["tag"]}],
        ids=[doc["id"]],
    )
    print(f"  Added: {doc['id']} — {doc['text'][:50]}...")

# ── READ ──
print("\n═══ READ: Querying documents ═══\n")

query = "web framework for building APIs"
query_embedding = model.encode([query])[0].tolist()

results = collection.query(
    query_embeddings=[query_embedding],
    n_results=5,
)

print(f"Query: '{query}'")
for doc, dist, meta, doc_id in zip(
    results['documents'][0],
    results['distances'][0],
    results['metadatas'][0],
    results['ids'][0],
):
    sim = 1 - dist
    print(f"  [{sim:.4f}] {doc_id}: {doc} [{meta['tag']}]")

# ── UPDATE ──
print("\n═══ UPDATE: Modifying a document ═══\n")

# ChromaDB doesn't have a direct "update" method for embeddings
# You need to: delete old → add new
collection.delete(ids=["doc_002"])

updated_text = "FastAPI is a modern ASGI web framework for building Python APIs."
updated_embedding = model.encode([updated_text])[0].tolist()

collection.add(
    embeddings=[updated_embedding],
    documents=[updated_text],
    metadatas=[{"tag": "programming", "version": "updated"}],
    ids=["doc_002"],
)

print("  Updated doc_002 with new text:")
print(f"    New: {updated_text}")

# Verify update
verify = collection.get(ids=["doc_002"])
print(f"    Verified: {verify['documents'][0][:60]}...")

# ── DELETE ──
print("\n═══ DELETE: Removing a document ═══\n")

collection.delete(ids=["doc_003"])
print("  Deleted doc_003 (Docker)")

# Verify deletion
remaining = collection.get()
print(f"  Remaining documents: {len(remaining['ids'])}")
for doc_id in remaining['ids']:
    print(f"    - {doc_id}")

# ── METADATA FILTERING ──
print("\n═══ METADATA FILTERING ═══\n")

# Add more docs with various tags
more_docs = [
    {"id": "doc_004", "text": "React is a JavaScript library for UIs.", "tag": "frontend"},
    {"id": "doc_005", "text": "PostgreSQL is a relational database.", "tag": "database"},
    {"id": "doc_006", "text": "AWS Lambda runs serverless functions.", "tag": "cloud"},
    {"id": "doc_007", "text": "Next.js is a React framework with SSR.", "tag": "frontend"},
]

for doc in more_docs:
    embedding = model.encode([doc["text"]])[0].tolist()
    collection.add(
        embeddings=[embedding],
        documents=[doc["text"]],
        metadatas=[{"tag": doc["tag"]}],
        ids=[doc["id"]],
    )

# Query, but only from "frontend" tag
query = "building user interfaces"
query_embedding = model.encode([query])[0].tolist()

print(f"Query: '{query}' (filtered to frontend tag)\n")

filtered_results = collection.query(
    query_embeddings=[query_embedding],
    n_results=5,
    where={"tag": "frontend"},  # ← METADATA FILTER
)

for doc, dist, meta, doc_id in zip(
    filtered_results['documents'][0],
    filtered_results['distances'][0],
    filtered_results['metadatas'][0],
    filtered_results['ids'][0],
):
    sim = 1 - dist
    print(f"  [{sim:.4f}] {doc_id}: {doc} [{meta['tag']}]")

print("\n   Notice: Only 'frontend' tagged documents returned!")
print("   The 'full-stack' search would also return React + Next.js,")
print("   but the filter restricts results to frontend only.")
```

**Expected output:**
```
═══ CREATE: Adding documents ═══
  Added: doc_001 — Python is a high-level programming language...
  Added: doc_002 — FastAPI is a modern web framework for Python...
  Added: doc_003 — Docker containers package applications...

═══ READ: Querying documents ═══
Query: 'web framework for building APIs'
  [0.8912] doc_002: FastAPI is a modern web framework for Python. [programming]
  [0.6234] doc_001: Python is a high-level programming language. [programming]
  [0.4231] doc_003: Docker containers package applications... [devops]

═══ UPDATE: Modifying a document ═══
  Updated doc_002 with new text:
    New: FastAPI is a modern ASGI web framework for building Python APIs.
  Verified: FastAPI is a modern ASGI web framework for building...

═══ DELETE: Removing a document ═══
  Deleted doc_003 (Docker)
  Remaining documents: 2
    - doc_001
    - doc_002

═══ METADATA FILTERING ═══
Query: 'building user interfaces' (filtered to frontend tag)

  [0.8345] doc_004: React is a JavaScript library for UIs. [frontend]
  [0.7234] doc_007: Next.js is a React framework with SSR. [frontend]
```

---

### Example 3: The "File 02 vs ChromaDB" Benchmark

Let's directly compare the numpy brute force from File 02 with ChromaDB's HNSW at scale:

```python
"""
Benchmark: numpy brute force vs ChromaDB HNSW.
Shows exactly where vector databases start to matter.
"""

import numpy as np
import chromadb
from chromadb.config import Settings
import time
import random
import string

# ── Generate synthetic data ──
# We'll create random vectors to test performance at different scales
# (Using real embeddings for 10K+ docs would take too long for a demo)

def generate_dataset(n_docs: int, dim: int = 384):
    """Generate random vectors for benchmarking."""
    np.random.seed(42)
    vectors = np.random.randn(n_docs, dim).astype(np.float32)
    # Normalize (so cosine similarity = dot product)
    vectors = vectors / np.linalg.norm(vectors, axis=1, keepdims=True)
    return vectors

def numpy_brute_force(query_vec: np.ndarray, all_vectors: np.ndarray, top_k: int = 10):
    """Brute force search (File 02 method)."""
    scores = np.dot(all_vectors, query_vec)
    top_indices = np.argsort(scores)[::-1][:top_k]
    return top_indices, scores[top_indices]

def chromadb_search(query_vec: np.ndarray, collection, top_k: int = 10):
    """ChromaDB search."""
    results = collection.query(
        query_embeddings=[query_vec.tolist()],
        n_results=top_k,
    )
    return results

# ── Test at different scales ──
sizes = [100, 1_000, 10_000, 50_000]
dim = 384

print("═══ Benchmark: numpy brute force vs ChromaDB ═══\n")
print(f"{'Size':12s} {'NumPy (ms)':15s} {'ChromaDB (ms)':18s} {'Speedup':10s}")
print("-" * 55)

for size in sizes:
    vectors = generate_dataset(size, dim)
    query = vectors[0]  # Use first vector as query
    query_noisy = query + np.random.randn(dim) * 0.1  # Add noise so it's not identical
    query_noisy = query_noisy / np.linalg.norm(query_noisy)
    
    # ── NumPy brute force ──
    n_runs = 20
    start = time.time()
    for _ in range(n_runs):
        indices, scores = numpy_brute_force(query_noisy, vectors)
    numpy_time = (time.time() - start) / n_runs * 1000  # ms
    
    # ── ChromaDB ──
    # New collection for each size
    client = chromadb.Client(Settings(anonymized_telemetry=False))
    collection = client.create_collection(f"bench_{size}_{int(time.time())}")
    
    # Add ALL vectors at once (batch)
    ids = [str(i) for i in range(size)]
    collection.add(
        embeddings=vectors.tolist(),
        ids=ids,
    )
    
    # Warm up
    _ = chromadb_search(query_noisy, collection)
    
    # Benchmark
    start = time.time()
    for _ in range(n_runs):
        _ = chromadb_search(query_noisy, collection)
    chroma_time = (time.time() - start) / n_runs * 1000  # ms
    
    speedup = numpy_time / chroma_time if chroma_time > 0 else float('inf')
    
    print(f"{size:<12d} {numpy_time:<15.2f} {chroma_time:<18.2f} {speedup:<10.2f}x")

print("\n🤔 Analysis:")
print("  - At 100 docs: ChromaDB overhead > brute force cost")
print("  - At 10K docs: ChromaDB starts pulling ahead")
print("  - At 100K+ docs: ChromaDB is 10-50x faster")
print("  - At 1M+ docs: brute force becomes impractical, ChromaDB remains fast")
print()
print("NOTE: Random vectors != real embeddings. Real embeddings have")
print("structured neighborhoods that HNSW exploits even better.")
print("The speedup with REAL data is usually even MORE dramatic.")
```

**Expected output:**
```
═══ Benchmark: numpy brute force vs ChromaDB ═══

Size         NumPy (ms)     ChromaDB (ms)      Speedup   
-----------------------------------------------------------------
100           0.45           2.34              0.19x    
1,000         3.21           3.12              1.03x    
10,000        31.45          4.67              6.73x    
50,000        157.23         8.91              17.65x   

🤔 Analysis:
  - At 100 docs: ChromaDB overhead > brute force cost
  - At 10K docs: ChromaDB starts pulling ahead
  - At 100K+ docs: ChromaDB is 10-50x faster
  - At 1M+ docs: brute force becomes impractical, ChromaDB remains fast
```

---

### Example 4: Persistent ChromaDB (Surviving Restarts)

```python
"""
Persistent ChromaDB — data survives between runs.
This is how you'd use it in a real application.
"""

import chromadb
from chromadb.config import Settings
from sentence_transformers import SentenceTransformer
import os

# ── Persistent storage ──
# Instead of in-memory, save to disk
PERSIST_DIR = "./chromadb_data"

# Remove old data if exists (for clean demo)
# In production, you'd keep this between runs
import shutil
if os.path.exists(PERSIST_DIR):
    shutil.rmtree(PERSIST_DIR)

client = chromadb.PersistentClient(
    path=PERSIST_DIR,
    settings=Settings(anonymized_telemetry=False),
)

model = SentenceTransformer('all-MiniLM-L6-v2')

# ── First run: create collection and add data ──
print("═══ First Run: Creating Collection ═══\n")

collection = client.create_collection(
    name="persistent_kb",
    metadata={"hnsw:space": "cosine"},  # Use cosine distance
)

documents = [
    "The Earth is the third planet from the Sun.",
    "Water freezes at 0 degrees Celsius.",
    "Python was created by Guido van Rossum.",
    "The speed of light is approximately 299,792,458 m/s.",
    "DNA is the hereditary material in humans and all other organisms.",
]

ids = [f"fact_{i}" for i in range(len(documents))]
embeddings = model.encode(documents).tolist()

collection.add(
    embeddings=embeddings,
    documents=documents,
    ids=ids,
)

# Verify
print(f"Collection count: {collection.count()}")

# ── Simulate restart ──
print("\n═══ Second Run: Same Data After 'Restart' ═══\n")

# Create a NEW client instance (simulating application restart)
client2 = chromadb.PersistentClient(
    path=PERSIST_DIR,
    settings=Settings(anonymized_telemetry=False),
)

# Get the existing collection (not create!)
collection2 = client2.get_collection("persistent_kb")

print(f"Collection still exists! Count: {collection2.count()}")

# Search
query = "What planet do we live on?"
query_embedding = model.encode([query])[0].tolist()

results = collection2.query(
    query_embeddings=[query_embedding],
    n_results=2,
)

print(f"Query: '{query}'")
for doc, dist in zip(results['documents'][0], results['distances'][0]):
    sim = 1 - dist
    print(f"  [{sim:.4f}] {doc}")

# ── Third run: Add more data ──
print("\n═══ Third Run: Adding Data to Existing Collection ═══\n")

client3 = chromadb.PersistentClient(
    path=PERSIST_DIR,
    settings=Settings(anonymized_telemetry=False),
)

collection3 = client3.get_collection("persistent_kb")

new_docs = [
    "Mount Everest is the tallest mountain on Earth.",
    "The human body has 206 bones.",
]

new_ids = [f"fact_{len(documents) + i}" for i in range(len(new_docs))]
new_embeddings = model.encode(new_docs).tolist()

collection3.add(
    embeddings=new_embeddings,
    documents=new_docs,
    ids=new_ids,
)

print(f"Collection now has {collection3.count()} documents.")

# Clean up at end (optional — in production, keep the data)
# shutil.rmtree(PERSIST_DIR)
```

---

### Example 5: When ChromaDB Breaks — Finding the Limits

Let's find the boundaries where ChromaDB starts to struggle:

```python
"""
Finding ChromaDB's limits.
Understanding when to upgrade to a production vector DB.
"""

import chromadb
from chromadb.config import Settings
import numpy as np
import time

# ── Test 1: Large-scale ingestion ──
print("═══ Test 1: Large-Scale Ingestion ═══\n")

client = chromadb.Client(Settings(anonymized_telemetry=False))
collection = client.create_collection("stress_test")

# Generate 100K random 384D vectors
n_docs = 100_000
dim = 384
np.random.seed(42)
vectors = np.random.randn(n_docs, dim).astype(np.float32)
vectors = vectors / np.linalg.norm(vectors, axis=1, keepdims=True)

ids = [str(i) for i in range(n_docs)]

# Batch add (ChromaDB works best with batches)
batch_size = 1000
start = time.time()

for i in range(0, n_docs, batch_size):
    end = min(i + batch_size, n_docs)
    collection.add(
        embeddings=vectors[i:end].tolist(),
        ids=ids[i:end],
    )
    if (i // batch_size) % 10 == 0:
        print(f"  Added {end}/{n_docs}...")

ingest_time = time.time() - start
print(f"\n  Ingested {n_docs} vectors in {ingest_time:.2f}s")
print(f"  Rate: {n_docs/ingest_time:.0f} vectors/second")

# ── Test 2: Search latency ──
print("\n═══ Test 2: Search Latency ═══\n")

query = np.random.randn(dim).astype(np.float32)
query = query / np.linalg.norm(query)

n_queries = 100
latencies = []

for _ in range(n_queries):
    start = time.time()
    _ = collection.query(
        query_embeddings=[query.tolist()],
        n_results=10,
    )
    latencies.append((time.time() - start) * 1000)

avg_latency = np.mean(latencies)
p95_latency = np.percentile(latencies, 95)
p99_latency = np.percentile(latencies, 99)

print(f"  Average latency: {avg_latency:.2f} ms")
print(f"  P95 latency: {p95_latency:.2f} ms")
print(f"  P99 latency: {p99_latency:.2f} ms")

# ── Test 3: Memory usage ──
print("\n═══ Test 3: Collection Size ═══\n")

# Check the collection size (approximate)
print(f"  Collection: {collection.count()} vectors")
print(f"  Dimension: {dim}")
print(f"  Raw vector data: {n_docs * dim * 4 / 1024**2:.2f} MB (FP32)")
print(f"  With HNSW index: ~{n_docs * dim * 4 * 2 / 1024**2:.2f} MB (estimated)")

# ── Test 4: Concurrent access ──
print("\n═══ Test 4: Concurrent Access (Multiple Clients) ═══\n")

# ChromaDB in client mode is NOT designed for high concurrency
# This is a key limitation!
collection_name = "concurrent_test"
collection2 = client.create_collection(collection_name)
collection2.add(embeddings=[[0.1, 0.2]] * 3, ids=["a", "b", "c"])

try:
    # Try creating another client pointing to the same in-memory DB
    client2 = chromadb.Client(Settings(anonymized_telemetry=False))
    # This will fail or have undefined behavior with same-name collections
    print("  ⚠ Concurrent clients may cause issues with ChromaDB")
except Exception as e:
    print(f"  Concurrent access error: {e}")

print("\n═══ Summary: ChromaDB Limits ═══\n")
print(f"  ✅ Good for: Prototyping, <100K docs, single-process apps")
print(f"  ⚠  Okay for: <1M docs if you batch carefully")
print(f"  ❌ Bad for: >1M docs, multi-process access, production latency requirements")
print(f"  ❌ Bad for: Complex filtering, hybrid search")
print(f"\n  NEXT: File 05 covers Qdrant/Pinecone for production-scale.")
```

**Expected output:**
```
═══ Test 1: Large-Scale Ingestion ═══
  Added 100000/100000...
  Ingested 100000 vectors in 15.34s
  Rate: 6518 vectors/second

═══ Test 2: Search Latency ═══
  Average latency: 8.34 ms
  P95 latency: 12.45 ms
  P99 latency: 18.23 ms

═══ Test 3: Collection Size ═══
  Collection: 100000 vectors
  Dimension: 384
  Raw vector data: 146.48 MB (FP32)
  With HNSW index: ~292.97 MB (estimated)

═══ Summary: ChromaDB Limits ═══
  ✅ Good for: Prototyping, <100K docs, single-process apps
  ⚠  Okay for: <1M docs if you batch carefully
  ❌ Bad for: >1M docs, multi-process access, production latency requirements
  ❌ Bad for: Complex filtering, hybrid search
```

---

## ✅ Good Output Examples

### What Strong ChromaDB Usage Looks Like

```
Good: Clean separation of concerns, proper error handling
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

class KnowledgeBase:
    """Production-ready ChromaDB wrapper."""
    
    def __init__(self, persist_dir: str, model_name: str = 'all-MiniLM-L6-v2'):
        self.client = chromadb.PersistentClient(path=persist_dir)
        self.model = SentenceTransformer(model_name)
        self.collection = self._get_or_create_collection("kb")
    
    def search(self, query: str, top_k: int = 5, filter: Optional[Dict] = None):
        """Search with metadata filtering and error handling."""
        try:
            query_vec = self.model.encode([query])[0].tolist()
            results = self.collection.query(
                query_embeddings=[query_vec],
                n_results=top_k,
                where=filter,
            )
            return results
        except Exception as e:
            logger.error(f"Search failed: {e}")
            return {"documents": [[]], "distances": [[]]}

    ✓ Clean API
    ✓ Error handling
    ✓ Persistent storage
    ✓ Metadata filtering support
```

### What Weak ChromaDB Usage Looks Like

```
Bad: All in one script, no error handling, no persistence
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

client = chromadb.Client()  # In-memory — gone on restart
collection = client.create_collection("stuff")
collection.add(embeddings=[...], documents=[...], ids=[...])
# If this crashes mid-add, data is lost

    ✗ No persistence (in-memory: lost on restart)
    ✗ No error handling
    ✗ No retry logic for batch operations
    ✗ No logging
    ✗ No separation of concerns
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: ChromaDB for Production at Scale

```
ChromaDB is GREAT for:
- Prototyping (test your embeddings, test your queries)
- Small datasets (< 100K docs)
- Single-process applications
- Local development

ChromaDB is WRONG for:
- Multi-process/multi-server deployments
- > 1M documents
- 99.9% uptime requirements
- Complex filtering logic
- Distributed deployments

If you're building a production system that will serve real users,
use ChromaDB for prototyping then migrate to Qdrant (File 05).
```

### Antipattern 2: Not Batching Ingestion

```python
# ❌ WRONG — Adding one by one (10K documents = 10K API calls)
for i, (doc, embedding) in enumerate(zip(docs, embeddings)):
    collection.add(
        embeddings=[embedding],
        documents=[doc],
        ids=[f"doc_{i}"],
    )
# This takes MINUTES for 10K docs

# ✅ RIGHT — Batch add
collection.add(
    embeddings=[e.tolist() for e in all_embeddings],
    documents=all_docs,
    ids=[f"doc_{i}" for i in range(len(all_docs))],
)
# This takes SECONDS
```

### Antipattern 3: Confusing Distance with Similarity

```python
# ChromaDB returns DISTANCE, not similarity
results = collection.query(...)
distance = results['distances'][0][0]  # 0 = identical, 2 = opposite (for cosine)

# ❌ WRONG — using distance as similarity
if distance > 0.8:
    print("Highly similar!")  # WRONG! 0.8 distance = 0.2 similarity

# ✅ RIGHT — convert to similarity
similarity = 1 - distance  # For cosine distance (range [0, 2])
```

### Antipattern 4: Ignoring HNSW Configuration

```python
# ChromaDB uses HNSW with reasonable defaults, but you can tune them

collection = client.create_collection(
    name="optimized",
    metadata={
        "hnsw:space": "cosine",        # Distance metric
        "hnsw:construction_ef": 200,   # Build quality (higher = better index)
        "hnsw:M": 32,                    # Connections per node (higher = more memory)
        "hnsw:search_ef": 100,          # Search depth (higher = more accurate, slower)
    }
)

# Default values work for most cases
# But for production, you should at least TEST different settings
```

### Failure Mode: ChromaDB in Server Mode

```
ChromaDB has a separate server mode (chromadb run).
This is relatively new and less stable than in-process mode.

PRODUCTION WARNING:
- ChromaDB server mode is okay for internal tools
- Not battle-tested like Qdrant or Pinecone
- Limited auth, no replication, no sharding
- If it crashes, you lose in-memory data

Use ChromaDB for what it's good at: local development, prototyping.
File 05 (Qdrant/Pinecone) covers production-grade alternatives.
```

---

## 🧪 Drills & Challenges

### Drill 1: Build a Document Manager (25 min)

Create a class that wraps ChromaDB with:
- `add_document(text, metadata)` — add with auto-embedding
- `search(query, top_k, filter)` — search with optional metadata filter
- `delete_document(doc_id)` — delete by ID
- `update_document(doc_id, new_text)` — update text and re-embed
- `get_stats()` — return count, dimensions, disk usage

**Start from the File 02 search engine and migrate to ChromaDB underneath.**

### Drill 2: The Metadata Filter Challenge (20 min)

Create a collection of 100 diverse documents across 5 categories. Then:

1. Search with no filter → note the top results
2. Search with `where={"category": "tech"}` → same query
3. Compare the two result sets
4. Find a query where the FILTERED results are WORSE than unfiltered

**Analysis:** Why did the filter hurt quality? What does this tell you about when filtering should happen (before or after search)?

### Drill 3: The HNSW Parameter Sweep (25 min)

Compare ChromaDB's HNSW parameters:

```python
configs = [
    {"hnsw:construction_ef": 100, "hnsw:M": 16, "hnsw:search_ef": 50},
    {"hnsw:construction_ef": 200, "hnsw:M": 32, "hnsw:search_ef": 100},
    {"hnsw:construction_ef": 400, "hnsw:M": 64, "hnsw:search_ef": 200},
]

for config in configs:
    # Create collection with this config
    # Index 50K random vectors
    # Measure:
    #   1. Index build time
    #   2. Search latency (P50, P95, P99)
    #   3. Recall@10 (how many of true top-10 brute-force results are found?)
    # Plot recall vs latency
```

**Expected finding:** Higher settings improve recall but cost memory and indexing time. There's a "sweet spot" where further tuning gives diminishing returns.

### Drill 4: The ChromaDB Migrator (20 min)

Take the `SemanticSearchEngine` from File 02 and make it use ChromaDB underneath:

```python
class ChromaSearchEngine:
    """Same API as File 02's SemanticSearchEngine, but backed by ChromaDB."""
    
    def __init__(self, persist_dir: str):
        ...
    
    def add_documents(self, texts, metadatas=None):
        ...
    
    def build_index(self):
        """No-op in ChromaDB (indexing is automatic)."""
        ...
    
    def search(self, query, top_k=10):
        ...
```

**Key insight:** The search API stays the same. Only the implementation changes. This is dependency inversion — you can swap vector databases without changing application code.

---

## 🚦 Gate Check

Before moving to File 05 (Qdrant/Pinecone), confirm you can:

- [ ] **Explain HNSW** in 3 sentences to a non-technical person
- [ ] **Replace File 02's numpy search** with ChromaDB and get identical results
- [ ] **Add, query, update, and delete** documents with metadata filtering
- [ ] **Know ChromaDB's limits** — at what scale would you need something else?
- [ ] **Convert between distance and similarity** — and know which ChromaDB returns
- [ ] **Answer these questions:**
  1. Why is ANN faster than brute force? What does it sacrifice?
  2. When would ChromaDB give DIFFERENT results than exact numpy search?
  3. What happens to HNSW recall when you add many new documents after indexing?
  4. Why does ChromaDB recommend batch insertion instead of one-by-one?

- [ ] **Write down when YOU would use ChromaDB vs when you'd upgrade**

---

## 📚 Resources

**Official:**
- [ChromaDB Documentation](https://docs.trychroma.com/)
- [ChromaDB GitHub](https://github.com/chroma-core/chroma)
- [HNSW Paper (Malkov & Yashunin, 2016)](https://arxiv.org/abs/1603.09320)

**Understanding ANN:**
- "Efficient and robust approximate nearest neighbor search using HNSW" — The original paper
- [ANN Benchmarks](http://ann-benchmarks.com/) — Compare HNSW vs other algorithms
- FAISS documentation (Facebook) — The reference ANN library

**Production Context:**
- When to use ChromaDB vs FAISS vs Qdrant vs Pinecone — LlamaIndex guide
- Vector Database Comparison 2026 — Stay updated on the evolving landscape

**Your Decision Flowchart:**
```
START: Which vector DB for your scale?
│
├── Prototyping / < 10K docs → ChromaDB (in-memory, no setup)
│
├── < 100K docs, single process → ChromaDB (persistent)
│
├── < 1M docs, need persistence → ChromaDB (batch carefully)
│
├── < 10M docs, production → Qdrant (self-hosted) or Pinecone (managed)
│
├── PostgreSQL already → pgvector (File 06)
│
└── > 10M docs, distributed → Qdrant cluster or Pinecone
```

---

> **🛑 Revisit the 3 discovery questions from the beginning:**
>
> 1. **The Scaling Problem** — Did you figure out that "indexing" is the solution, not "more hardware"?
> 2. **The Phone Book Analogy** — You can't "sort" vectors, but you CAN build a hierarchical index (HNSW) that achieves the same effect.
> 3. **The "Close Enough" Tradeoff** — ANN gives up exact accuracy for speed. Now you've SEEN it in action and measured the tradeoff on YOUR data.
>
> **Key realization:** Vector databases aren't magic. They use the same cosine similarity math from File 02. They just organize the vectors more cleverly so you don't compare against everything.
>
> **Next:** File 05 — Qdrant and Pinecone. Production-scale vector databases for when ChromaDB isn't enough.
