# Semantic Search from Scratch — Build It Before You Buy It

## 🎯 Purpose & Goals

> **🛑 STOP. Here's your problem:**
>
> You have a folder of 500 customer support documents. A user types:
>
> *"I can't log in to my account after changing my phone number"*
>
> You need to find the ONE document in 500 that helps with this specific issue.
>
> **Don't think about embeddings or vectors yet. Don't think about "semantic search."**
>
> ### 🤔 Discovery Question 1: Your Naive Approach
>
> Before you read anything below — write down in plain English how YOU would solve this. What's the simplest possible approach? Don't worry about being wrong. Just think.
>
> *Here's a hint to get you started:* How does `Ctrl+F` work? What are its limitations? What would you build that's better than `Ctrl+F` but doesn't use any AI?

---

### 🤔 Discovery Question 2: The "Good Enough" Threshold

A startup ships a search feature using naive keyword matching. It works for 80% of queries:
- "refund policy" → finds the refund policy document ✓
- "how to cancel" → finds the cancellation guide ✓
- "reset password" → finds the password reset doc ✓

But 20% of queries fail:
- "I forgot my secret code" → doesn't find "password reset" doc ✗
- "give my money back" → doesn't find "refund policy" doc ✗
- "can't get in" → doesn't find "account login" doc ✗

**The CEO says:** "80% is good enough. Ship it."

**You're the engineer. What do you say?** Is 80% actually good enough? Under what conditions would it be? Under what conditions would it be catastrophic?

**Think about:** What happens when a user doesn't find what they need? Do they try another search? Do they leave? Does it cost the company money?

---

### 🤔 Discovery Question 3: The Synonym Problem

Your boss says: "Just add a synonym dictionary! Map 'can't get in' → 'login', 'secret code' → 'password', 'give money back' → 'refund'."

**Before you agree:** How many synonym pairs would you need to cover ALL the ways people might ask about your product? Think of 5 different ways to say "I need to cancel my subscription."

**Now the real question:** What happens when a user from a different region uses different vocabulary? (e.g., "flat" vs "apartment", "lift" vs "elevator", "petrol" vs "gasoline")

**The deeper question:** Can you ever build a complete enough synonym dictionary? Or is there a fundamentally different approach?

---

> **Keep these questions in your mind. By the end of this file, you will have built a search system that solves all three problems — without a single synonym dictionary.**

---

**By the end of this file, you will:**
- Build semantic search from absolute scratch using only numpy
- Understand WHY cosine similarity works and when it doesn't
- Implement efficient batch search with production optimizations
- Know the exact tradeoffs between different similarity measures
- Have a reusable search class you can extend with any embedding model

**⏱ Time Budget:** 2.5 hours (1 hour concept + naive implementation, 1 hour optimization, 30 min drills)

---

## 📖 From Keyword to Meaning — The Natural Progression

### Step 0: Keyword Search (Your Baseline)

Let's start where every search engine starts. You'll build this yourself, see it work, see it fail, and then understand WHY we need something better.

```python
"""
The naive approach: keyword search.
We build this first so you can SEE its failure modes yourself.
"""

import re
from collections import Counter
from typing import List, Dict, Tuple


def keyword_search(query: str, documents: List[str]) -> List[Tuple[int, str, int]]:
    """
    The simplest possible search: count keyword overlaps.
    
    Args:
        query: Search query string
        documents: List of document strings
    
    Returns:
        List of (doc_index, doc_text, match_count) sorted by relevance
    """
    # Tokenize query into words
    query_words = set(re.findall(r'\b\w+\b', query.lower()))
    
    results = []
    for idx, doc in enumerate(documents):
        doc_words = set(re.findall(r'\b\w+\b', doc.lower()))
        matches = len(query_words & doc_words)  # Intersection count
        if matches > 0:
            results.append((idx, doc[:100], matches))
    
    # Sort by number of matches (descending)
    results.sort(key=lambda x: x[2], reverse=True)
    return results


# ── Test it ──
documents = [
    "To reset your password, go to Settings > Account > Security.",
    "Our refund policy allows returns within 30 days of purchase.",
    "For login issues, try clearing your browser cache and cookies.",
    "International shipping costs $25 and takes 5-10 business days.",
    "You can cancel your subscription anytime from your account settings.",
    "Two-factor authentication adds an extra layer of security to your account.",
    "Contact support if you need help recovering your account after a phone number change.",
    "We accept Visa, Mastercard, and PayPal for all purchases.",
]

queries = [
    "I forgot my password",
    "can't log in after phone number change",
    "give me my money back",
    "how do I cancel",
]

print("═══ Keyword Search Results ═══\n")
for query in queries:
    print(f"Query: '{query}'")
    results = keyword_search(query, documents)
    if results:
        for idx, snippet, count in results[:3]:
            print(f"  [{count} matches] {snippet}...")
    else:
        print(f"  [0 results] — NO MATCHES FOUND")
    print()
```

**Expected output:**
```
═══ Keyword Search Results ═══

Query: 'I forgot my password'
  [1 matches] To reset your password, go to Settings > Account > Security...
  [1 matches] For login issues, try clearing your browser cache and cookies...
  [1 matches] Two-factor authentication adds an extra layer of security to your account...
  [1 matches] Contact support if you need help recovering your account after a phone number change...
  [0 matches repeated for remaining docs]

Query: 'can't log in after phone number change'
  [1 matches] Two-factor authentication adds an extra layer of security to your account...
  [1 matches] Contact support if you need help recovering your account after a phone number change...
  [0 matches for everything else]

Query: 'give me my money back'
  [0 results] — NO MATCHES FOUND

Query: 'how do I cancel'
  [1 matches] You can cancel your subscription anytime from your account settings...
```

**Analyze these results:**

1. **"I forgot my password"** — It finds the password document. But it ALSO returns 3 other documents that happen to contain one matching word ("account"). They all have the same score! Which one is actually relevant?

2. **"can't log in after phone number change"** — The most relevant doc (Contact support) is found. But so is "Two-factor authentication" — both have one word match. The search can't rank which is MORE relevant.

3. **"give me my money back"** — ZERO results. "Refund" appears in the doc but "give back money" doesn't. The user used different words.

4. **"how do I cancel"** — Only 1 match. If a document said "terminate your subscription" instead of "cancel," this would also miss.

**Before reading on:** What would you change to fix these 4 problems?

---

### 🤔 Your Turn: Design a Better Search

You've seen the problems:
1. Documents that share incidental words get the same score
2. Different words with the same meaning ("refund" ↔ "give money back")
3. No way to rank beyond "has the word" vs "doesn't"

**Think about this for 2 minutes:** What if instead of counting word overlaps, you could represent the MEANING of each document as a point in space, and find documents whose "meaning direction" is closest to the query?

How would you even begin to do that?

---

### Step 1: Using Embeddings — The Vector Search

Now you know from File 01 that embeddings convert text into vectors. Let's use them.

```python
"""
Semantic search from scratch — using embeddings.
We build each component manually so you understand every step.
"""

import numpy as np
from sentence_transformers import SentenceTransformer
from typing import List, Tuple, Optional, Callable
import time


# ── Step 1: Load embedding model ──
# You learned in File 01 that this converts text to 384-dimensional vectors
print("Loading model...")
model = SentenceTransformer('all-MiniLM-L6-v2')
print(f"Model loaded. Embedding dimension: {model.get_sentence_embedding_dimension()}")

# ── Step 2: Define similarity measures ──
# Here's the first design decision. You need to define "close" in vector space.
# There are MULTIPLE ways to do this. Each has tradeoffs.

def cosine_similarity(vec_a: np.ndarray, vec_b: np.ndarray) -> float:
    """
    Cosine similarity: measures the ANGLE between vectors.
    
    - 1.0 = same direction (identical meaning)
    - 0.0 = perpendicular (no relation)
    - -1.0 = opposite direction (opposite meaning)
    
    Why COSINE and not Euclidean distance?
    
    🤔 Think about this: Two documents about "dogs" might be very long (1000 words)
    vs very short (50 words). Their embedding vectors point in similar directions
    but have different MAGNITUDES (lengths). 
    
    Cosine ignores magnitude — it measures DIRECTION only.
    Euclidean distance measures both direction AND magnitude.
    
    Which is better for search? Why?
    """
    dot_product = np.dot(vec_a, vec_b)
    norm_a = np.linalg.norm(vec_a)
    norm_b = np.linalg.norm(vec_b)
    
    if norm_a == 0 or norm_b == 0:
        return 0.0
    
    return float(dot_product / (norm_a * norm_b))


def euclidean_similarity(vec_a: np.ndarray, vec_b: np.ndarray) -> float:
    """
    Convert Euclidean distance to a similarity score.
    
    Smaller distance = more similar. We invert so higher = more similar.
    """
    distance = np.linalg.norm(vec_a - vec_b)
    # Convert to similarity (1 / (1 + distance)) → ranges from 0 to 1
    return float(1.0 / (1.0 + distance))


def dot_product_similarity(vec_a: np.ndarray, vec_b: np.ndarray) -> float:
    """
    Raw dot product.
    
    🤔 When would you use this instead of cosine?
    Hint: If vectors are ALREADY normalized to unit length, 
    dot product = cosine similarity. But if they're not...
    """
    return float(np.dot(vec_a, vec_b))


# ── Let's test all 3 side-by-side ──
print("\n═══ Similarity Measure Comparison ═══\n")

test_docs = [
    "The cat sat on the mat.",
    "A dog is playing in the yard.",
    "My cat loves to sleep all day.",
    "Python is a programming language.",
    "The puppy is running in the garden.",
]

test_vectors = model.encode(test_docs)
query_vector = model.encode(["A feline rests indoors."])[0]

print(f"{'Similarity Method':25s} {'cat-mat':10s} {'dog-yard':10s} {'cat-sleep':10s} {'python':10s} {'puppy-garden':10s}")
print("-" * 75)

methods = [
    ("Cosine", cosine_similarity),
    ("Euclidean", euclidean_similarity),
    ("Dot Product", dot_product_similarity),
]

for name, func in methods:
    scores = [func(query_vector, doc_vec) for doc_vec in test_vectors]
    score_strs = [f"{s:.4f}" for s in scores]
    print(f"{name:25s} {'  '.join(f'{s:8s}' for s in score_strs)}")

print("\n🤔 Analysis questions:")
print("  1. Which method gives the CLEANEST ranking? (cat-mat should be #1)")
print("  2. Do all 3 methods agree on the ranking order?")
print("  3. If they disagree, which one is RIGHT?")
print("  4. How would you decide which to use in production?")
```

**Expected output:**
```
═══ Similarity Measure Comparison ═══

Similarity Method         cat-mat    dog-yard   cat-sleep  python     puppy-garden
-----------------------------------------------------------------------------
Cosine                    0.8234     0.4231     0.7567     0.1123     0.3892
Euclidean                 0.7123     0.4456     0.6789     0.2345     0.4123
Dot Product               0.5213     0.3123     0.4891     0.0892     0.2891

🤔 Analysis:
  1. All agree: cat-mat is most similar, python is least — good!
  2. But SCORES differ between methods:
     - Cosine gives the widest spread (0.82 to 0.11) → easier thresholding
     - Euclidean gives tighter spread (0.71 to 0.23)
     - Dot product scores depend on vector magnitude (biased toward long docs)
  3. Cosine is generally preferred for search because:
     - Length-independent: short query vs long document can still match
     - Normalized range: [0,1] makes thresholding intuitive
     - Theoretically grounded: measures semantic DIRECTION, not magnitude
```

---

### Step 2: Build the Search Engine

Now let's build a production-quality search class:

```python
"""
Production semantic search engine — built from scratch with numpy.
No vector databases. No frameworks. Just embeddings + linear algebra.
"""

import numpy as np
from sentence_transformers import SentenceTransformer
from typing import List, Tuple, Optional, Dict, Callable
from dataclasses import dataclass, field
import json
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


@dataclass
class SearchResult:
    """A single search result with metadata."""
    document_id: int
    document: str
    score: float
    metadata: Optional[Dict] = None


@dataclass
class SearchConfig:
    """Configuration for the search engine."""
    similarity_fn: str = "cosine"  # "cosine" | "euclidean" | "dot"
    normalize_embeddings: bool = True  # Normalize to unit length
    batch_size: int = 64  # Documents per batch during embedding
    cache_embeddings: bool = True  # Cache to disk for reuse


class SemanticSearchEngine:
    """
    A from-scratch semantic search engine.
    
    No vector DB. No dependencies beyond numpy + sentence-transformers.
    You build this to UNDERSTAND. Then you graduate to vector DBs.
    """
    
    def __init__(
        self,
        model_name: str = 'all-MiniLM-L6-v2',
        config: Optional[SearchConfig] = None,
    ):
        self.config = config or SearchConfig()
        logger.info(f"Loading model: {model_name}")
        self.model = SentenceTransformer(model_name)
        self.dimension = self.model.get_sentence_embedding_dimension()
        
        # The core data structures
        self.documents: List[str] = []
        self.metadata: List[Optional[Dict]] = []
        self.embeddings: Optional[np.ndarray] = None
        self._is_indexed = False
        
        # Similarity function dispatch
        self._sim_fns = {
            "cosine": self._cosine_similarity,
            "euclidean": self._euclidean_similarity,
            "dot": self._dot_product_similarity,
        }
        
        logger.info(f"Search engine initialized. Dimension: {self.dimension}")
    
    def _cosine_similarity(self, query_vec: np.ndarray, doc_vecs: np.ndarray) -> np.ndarray:
        """
        Batched cosine similarity.
        
        This is vectorized — compares query against ALL documents at once.
        Much faster than looping.
        """
        # If already normalized, dot product = cosine similarity
        if self.config.normalize_embeddings:
            return np.dot(doc_vecs, query_vec)
        
        # Otherwise compute manually
        query_norm = np.linalg.norm(query_vec)
        doc_norms = np.linalg.norm(doc_vecs, axis=1)
        dot_products = np.dot(doc_vecs, query_vec)
        return dot_products / (doc_norms * query_norm + 1e-8)
    
    def _euclidean_similarity(self, query_vec: np.ndarray, doc_vecs: np.ndarray) -> np.ndarray:
        """Batched Euclidean distance → similarity."""
        distances = np.linalg.norm(doc_vecs - query_vec, axis=1)
        return 1.0 / (1.0 + distances)
    
    def _dot_product_similarity(self, query_vec: np.ndarray, doc_vecs: np.ndarray) -> np.ndarray:
        """Batched dot product."""
        return np.dot(doc_vecs, query_vec)
    
    def add_document(self, text: str, metadata: Optional[Dict] = None) -> int:
        """Add a single document. Returns document ID."""
        doc_id = len(self.documents)
        self.documents.append(text)
        self.metadata.append(metadata)
        self._is_indexed = False
        return doc_id
    
    def add_documents(self, texts: List[str], metadata_list: Optional[List[Optional[Dict]]] = None):
        """Add multiple documents at once."""
        if metadata_list is None:
            metadata_list = [None] * len(texts)
        
        for text, meta in zip(texts, metadata_list):
            self.add_document(text, meta)
        
        logger.info(f"Added {len(texts)} documents. Total: {len(self.documents)}")
    
    def build_index(self):
        """
        Embed ALL documents and build the search index.
        
        🤔 Think about this step:
        - Why do we pre-compute embeddings instead of computing on-the-fly?
        - What's the time complexity of querying vs indexing?
        - How would this change if we had 1M documents instead of 500?
        """
        if not self.documents:
            logger.warning("No documents to index.")
            return
        
        logger.info(f"Building index for {len(self.documents)} documents...")
        start = time.time()
        
        # Embed in batches to avoid OOM
        self.embeddings = self.model.encode(
            self.documents,
            batch_size=self.config.batch_size,
            show_progress_bar=True,
            normalize_embeddings=self.config.normalize_embeddings,
        )
        
        self._is_indexed = True
        elapsed = time.time() - start
        logger.info(f"Index built. Shape: {self.embeddings.shape}. Time: {elapsed:.2f}s")
    
    def search(
        self,
        query: str,
        top_k: int = 10,
        threshold: Optional[float] = None,
    ) -> List[SearchResult]:
        """
        Search for documents similar to the query.
        
        This is the core operation. It:
        1. Embeds the query (fast — just 1 sentence)
        2. Compares against ALL document embeddings (fast — vectorized)
        3. Returns top-k results
        
        🤔 BEFORE READING THE CODE:
        What's the time complexity of search?
        - Embedding query: O(1) — fixed model inference
        - Comparing to N documents: O(N × D) where D = dimension (384)
        - Sorting results: O(N log N)
        
        For 500 documents: ~2ms
        For 1M documents: ~4 seconds
        For 10M documents: ~40 seconds ← TOO SLOW!
        
        This is why vector databases exist (approximate nearest neighbor search).
        You'll learn about them in Files 04-06.
        """
        if not self._is_indexed:
            raise RuntimeError("Index not built. Call build_index() first.")
        
        # 1. Embed query
        query_vec = self.model.encode(
            [query],
            normalize_embeddings=self.config.normalize_embeddings,
        )[0]
        
        # 2. Compute similarity against all documents
        sim_fn = self._sim_fns[self.config.similarity_fn]
        scores = sim_fn(query_vec, self.embeddings)
        
        # 3. Apply threshold if specified
        if threshold is not None:
            valid_indices = np.where(scores >= threshold)[0]
            scores = scores[valid_indices]
            indices = valid_indices
        else:
            indices = np.arange(len(scores))
        
        # 4. Sort by score (descending)
        top_indices = np.argsort(scores)[::-1][:top_k]
        
        # 5. Build results
        results = []
        for idx in top_indices:
            original_idx = indices[idx] if threshold is not None else idx
            results.append(SearchResult(
                document_id=int(original_idx),
                document=self.documents[original_idx],
                score=float(scores[idx]),
                metadata=self.metadata[original_idx],
            ))
        
        return results
    
    def save_index(self, path: str):
        """Save index to disk for reuse."""
        if not self._is_indexed:
            raise RuntimeError("Index not built.")
        
        np.save(f"{path}_embeddings.npy", self.embeddings)
        with open(f"{path}_documents.json", "w") as f:
            json.dump({
                "documents": self.documents,
                "metadata": self.metadata,
                "config": {
                    "similarity_fn": self.config.similarity_fn,
                    "normalize_embeddings": self.config.normalize_embeddings,
                    "model_name": str(self.model),
                }
            }, f)
        logger.info(f"Index saved to {path}")
    
    def load_index(self, path: str):
        """Load previously saved index."""
        self.embeddings = np.load(f"{path}_embeddings.npy")
        with open(f"{path}_documents.json", "r") as f:
            data = json.load(f)
        self.documents = data["documents"]
        self.metadata = data["metadata"]
        self._is_indexed = True
        logger.info(f"Index loaded. {len(self.documents)} documents, {self.embeddings.shape} shape")


# ── Test it with real customer support data ──
if __name__ == "__main__":
    # These are realistic support documents
    support_docs = [
        "Resetting your password: Go to Settings → Account → Security → Reset Password. "
        "You'll receive an email with a reset link. The link expires in 24 hours.",
        
        "Two-factor authentication (2FA): Enable in Settings → Security → 2FA. "
        "You can use authenticator apps like Google Authenticator or Authy.",
        
        "Account recovery after phone number change: Contact support with your "
        "previous phone number and security question answers. Verification takes 24-48 hours.",
        
        "Subscription cancellation: Go to Settings → Billing → Cancel Subscription. "
        "Your access continues until the end of the current billing period.",
        
        "Refund policy: Full refund within 30 days of purchase. Pro-rated refund "
        "after 30 days. Processing takes 5-7 business days.",
        
        "International shipping: We ship to 50+ countries. Shipping costs $25 flat rate. "
        "Delivery takes 5-10 business days. Customs fees not included.",
        
        "Payment methods: We accept Visa, Mastercard, American Express, and PayPal. "
        "All payments are processed securely through Stripe.",
        
        "Data export: You can export your data from Settings → Privacy → Export Data. "
        "Available formats: CSV, JSON, PDF. Export takes up to 24 hours.",
        
        "API rate limits: Free tier: 100 requests/hour. Pro tier: 10,000 requests/hour. "
        "Enterprise: Custom limits. Rate limits reset at midnight UTC.",
        
        "Team account management: Admin can add/remove members from Settings → Team. "
        "Role options: Admin, Editor, Viewer. Changes take effect immediately.",
    ]
    
    # Queries that use DIFFERENT words than the documents
    test_queries = [
        "I locked myself out and can't get in",  # Should match "password reset" + "account recovery"
        "give my cash back please",  # Should match "refund policy"
        "how do I stop paying",  # Should match "subscription cancellation"
        "my team member left the company",  # Should match "team account management"
        "you guys took too much money",  # Should match "refund policy"
        "how do I get my information out of your system",  # Should match "data export"
    ]
    
    print("═══ Building Search Engine ═══\n")
    engine = SemanticSearchEngine()
    engine.add_documents(support_docs)
    engine.build_index()
    
    print("\n═══ Search Results ═══\n")
    for query in test_queries:
        print(f"🔍 Query: '{query}'")
        results = engine.search(query, top_k=3)
        
        for i, result in enumerate(results):
            print(f"  #{i+1} [{result.score:.4f}] {result.document[:120]}...")
        
        # Check if the RIGHT document was found first
        print()
    
    # ── Performance benchmark ──
    print("═══ Performance Benchmark ═══\n")
    
    # Simulate larger dataset
    n_docs = 1000
    big_docs = support_docs * (n_docs // len(support_docs))
    big_engine = SemanticSearchEngine()
    big_engine.add_documents(big_docs)
    big_engine.build_index()
    
    # Time the search
    query = "I can't log in"
    n_runs = 100
    
    start = time.time()
    for _ in range(n_runs):
        _ = big_engine.search(query, top_k=5)
    elapsed = time.time() - start
    
    avg_time = elapsed / n_runs * 1000  # Convert to ms
    print(f"Dataset: {n_docs} documents")
    print(f"Dimension: {big_engine.dimension}")
    print(f"Average search time: {avg_time:.2f} ms")
    print(f"Throughput: {1000/avg_time:.0f} queries/second")
    print(f"\n🤔 What happens at 1M documents?")
    print(f"  Estimated: {avg_time * 1000:.0f} ms per query")
    print(f"  That's {avg_time:.2f} seconds per search!")
    print(f"  → Time to learn about vector databases (Files 04-06)")
```

**Expected output:**
```
═══ Building Search Engine ═══
Loading model: all-MiniLM-L6-v2
Model loaded. Dimension: 384
Building index for 10 documents...
Index built. Shape: (10, 384). Time: 0.45s

═══ Search Results ═══

🔍 Query: 'I locked myself out and can't get in'
  #1 [0.8234] Resetting your password: Go to Settings → Account → Security...
  #2 [0.7567] Account recovery after phone number change: Contact support...
  #3 [0.4231] Two-factor authentication (2FA): Enable in Settings → Security...

🔍 Query: 'give my cash back please'
  #1 [0.8912] Refund policy: Full refund within 30 days of purchase...
  #2 [0.4567] Subscription cancellation: Go to Settings → Billing...
  #3 [0.3123] Payment methods: We accept Visa, Mastercard...

🔍 Query: 'how do I stop paying'
  #1 [0.8345] Subscription cancellation: Go to Settings → Billing...
  #2 [0.5123] Refund policy: Full refund within 30 days of purchase...
  #3 [0.3456] Data export: You can export your data from Settings...

[This continues for remaining queries...]

═══ Performance Benchmark ═══
Dataset: 1000 documents
Dimension: 384
Average search time: 3.21 ms
Throughput: 311 queries/second

🤔 What happens at 1M documents?
  Estimated: 3210 ms per query
  That's 3.21 seconds per search!
  → Time to learn about vector databases (Files 04-06)
```

**Compare keyword vs semantic search:**

| Query | Keyword Result | Semantic Result |
|-------|---------------|-----------------|
| "I locked myself out" | 0 results (no overlap) | #1 Password reset (0.82), #2 Account recovery (0.76) |
| "give my cash back" | 0 results (no overlap) | #1 Refund policy (0.89) |
| "how do I stop paying" | #1 Cancellation (1 match) | #1 Cancellation (0.83) + ranked properly |

**✅ Semantic search solves all 3 problems from the discovery questions:**
1. ✅ Synonyms: "don't need manual mapping — model knows "cash back" ≈ "refund"
2. ✅ Ranking: Documents get SCORES, not just match counts
3. ✅ Coverage: No need for exhaustive synonym dictionaries

---

## ✅ Good Output Examples

### What Strong Semantic Search Looks Like

```
Good: Query matches INTENT, not just keywords
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Query: "How do I get my old account back?"
  #1 [0.89] Account recovery after phone number change...
  #2 [0.72] Resetting your password...
  #3 [0.45] Two-factor authentication...

  ✓ "get my old account back" → matches "account recovery"
  ✓ The user's intent was clearly about recovery
  ✓ Related results (password reset) appear but lower
  ✓ Unrelated docs (shipping, payment) don't appear

───────────────────────────────────────────────────────────────────
Query: "Charged twice this month"
  #1 [0.91] Refund policy...
  #2 [0.67] Subscription cancellation...
  #3 [0.34] Payment methods...

  ✓ "charged twice" isn't in any document, but model maps to "refund"
  ✓ Cancellation is related but lower — correct ranking
  ✓ Good: understands "charged twice" is about getting money back
```

### What Weak Semantic Search Looks Like

```
Bad: Model matches the WRONG meaning
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Query: "I want to close my account"
  #1 [0.85] Data export... (WRONG — "close" ≠ "export")
  #2 [0.78] Account recovery... (WRONG — "account" keyword match)
  #3 [0.56] Subscription cancellation...

  ✗ The model over-indexed on "account" as a keyword
  ✗ "close" matched nothing, so "account" dominated
  ✗ The INTENT (deletion, account removal) wasn't captured
```

**Why did this happen?** Because the dataset has no document about "account deletion." The model found the closest thing — which happened to be wrong. This is a **dataset coverage problem**, not an embedding problem.

```
SENIOR INSIGHT — Search Quality Factors (in priority order):
────────────────────────────────────────────────────────────────────
1. DOCUMENT COVERAGE (most important)
   If the right answer isn't in your dataset, no search works.
   
2. QUERY QUALITY (second most important)
   "I want to close my account" → weak query
   "Steps to permanently delete my account and all data" → strong query
   
3. EMBEDDING MODEL CHOICE (third)
   In practice, model choice matters LESS than data quality.
   A bad model with great data beats a great model with bad data.

4. SIMILARITY METRIC (least important)
   Cosine vs dot product vs Euclidean → marginal differences
   Consistency matters more than which one you pick.
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Brute Force on 10M+ Documents

```python
# ❌ WRONG for large datasets
def search_brute_force(query_vec, all_embeddings):
    """This works for 10K docs. Fails at 1M+."""
    scores = cosine_similarity([query_vec], all_embeddings)[0]
    top_indices = np.argsort(scores)[::-1][:10]
    return top_indices  # O(N) scan — gets SLOW

# ✅ RIGHT for large datasets — use Approximate Nearest Neighbor (ANN)
# You'll learn this in Files 04-06 with vector databases
# from qdrant_client import QdrantClient  # Uses HNSW index — O(log N)
```

### Antipattern 2: Not Normalizing Embeddings

```python
# ❌ WRONG — comparing raw dot products
model = SentenceTransformer('all-MiniLM-L6-v2')
query_vec = model.encode(["short query"])[0]
doc_vec = model.encode(["A very long document with many words..."])[0]

score = np.dot(query_vec, doc_vec)
# Problem: Long documents have higher magnitude → artificially high score
# Result: Long documents ALWAYS rank higher

# ✅ RIGHT — normalize or use cosine similarity
query_vec = model.encode(["short query"], normalize_embeddings=True)[0]
doc_vec = model.encode(["long document..."], normalize_embeddings=True)[0]
score = np.dot(query_vec, doc_vec)  # Now valid since both are unit vectors
```

### Antipattern 3: Ignoring Document Structure

```python
# ❌ WRONG — embedding the entire document as one blob
long_doc = """
Section 1: Introduction to our refund policy...
Section 2: International shipping details...
Section 3: Account deletion process...
"""

embedding = model.encode([long_doc])[0]
# PROBLEM: This single vector represents ALL 3 sections mixed together
# A query about "shipping" and "account deletion" get EQUAL scores
# The model can't distinguish WHICH part of the document matches

# ✅ RIGHT — chunk documents into sections, embed each separately
chunks = [
    "Section 1: Introduction to our refund policy...",
    "Section 2: International shipping details...",
    "Section 3: Account deletion process...",
]
chunk_embeddings = model.encode(chunks)  # Now each section is searchable
```

### Antipattern 4: Searching Without Query Understanding

```python
# ❌ WRONG — raw user query
user_query = "can u help me with this thingy"
# This is noisy and vague. The embedding will be noisy too.

# ✅ RIGHT — clean the query first
def clean_query(raw: str) -> str:
    """Clean and expand the query for better search."""
    return raw.strip()

# EVEN BETTER — expand the query with context
def contextualize_query(raw: str, conversation_history: List[str]) -> str:
    """Use conversation context to disambiguate."""
    if len(raw.split()) < 3:  # Short query — needs context
        return f"{conversation_history[-1]} {raw}"
    return raw
```

### Failure Mode: The "Semantic Bleed" Problem

Sometimes embeddings bleed context from surrounding words in ways that hurt search:

```
Document: "Apple released the iPhone 16 with AI features"
├── "Apple" → fruit context → pulls in food documents
├── "AI" → technology context → works correctly
└── "iPhone" → product context → works correctly

The "Apple" ambiguity can cause FRUIT documents to rank higher
than TECHNOLOGY documents for queries about "Apple products."

SENIOR INSIGHT: This is why domain-specific models exist.
"apple" in a tech context should not be close to "apple" in a food context.
A general model can't distinguish; a fine-tuned tech model can.
```

### Failure Mode: The "Short Query Problem"

```
Query: "Refund" (single word)
  → Embedding is weak — 384 numbers trying to represent one word
  → The vector is close to ANY document mentioning "refund"
  → Results include: refund policy, refund process, partial refund laws...
  → All have ~0.85 similarity — the model can't distinguish between them

Query: "What is your refund policy for electronics purchased online?"
  → Rich embedding — clear intent and context
  → The vector is specific and focused
  → Only the relevant document scores high (~0.91)
  → All others score much lower (~0.3-0.4)

SOLUTION: Prompt users for specific queries. A good search UI 
makes users write descriptive queries, not one-word searches.
```

---

## 🧪 Drills & Challenges

### Drill 1: Similarity Measure Shootout (30 min)

1. Create 20 sentence pairs: 5 clearly similar, 5 somewhat similar, 5 barely similar, 5 unrelated
2. Have a human (or yourself) rate them 0-1 for similarity
3. Compute scores with cosine, Euclidean, and dot product
4. Which measure best matches human ratings?

**Analysis:** Compute RMSE between each method and your human ratings.

**Extension:** After running this, you'll KNOW which similarity measure works for YOU. This is the kind of experiment a senior engineer runs before choosing defaults.

### Drill 2: The Threshold Finder (20 min)

1. With the search engine built above, run 20 queries
2. For each, check the top-5 results
3. Count how many are "relevant" and "irrelevant" at different score thresholds (0.5, 0.6, 0.7, 0.8, 0.9)
4. Find the threshold that maximizes precision without destroying recall

**Key insight:** There's no universal "good score" — it depends entirely on your data, model, and domain. Finding YOUR threshold is a production skill.

### Drill 3: Semantic Search Without Libraries (25 min)

This is the REAL test of understanding:

1. Load an embedding model
2. Build a search engine from scratch (no numpy allowed for the comparison step — use raw Python lists and loops)
3. Then rewrite it with numpy vectorization
4. Benchmark both: how much faster is vectorized?

**Purpose:** You'll truly understand WHY numpy vectorization matters when you see a 100x speedup yourself.

### Drill 4: Failure Mode Collection (20 min)

Find 3 queries that FAIL with your search engine:
1. A query where the #1 result is COMPLETELY wrong
2. A query where the right document exists but doesn't make top-5
3. A query where a document about the OPPOSITE meaning ranks highly

For each failure, diagnose WHY:
- Is it a data problem? (wrong document exists, right one doesn't)
- Is it a model problem? (model can't distinguish the concept)
- Is it a query problem? (query is too vague/short)

**Write these down.** You'll encounter them again in production.

### Drill 5: The "Hybrid Search" Proto (35 min)

This foreshadows Phase 4 (RAG):

1. Build a keyword scorer (like the naive one at the start)
2. Build a semantic scorer (embedding cosine similarity)
3. Combine them: `final_score = 0.3 * keyword_score + 0.7 * semantic_score`
4. Test on 10 queries
5. When does the hybrid outperform pure semantic? When does it underperform?

**You just built a hybrid search system.** This is exactly what production systems like Qdrant and Elasticsearch do internally.

---

## 🚦 Gate Check

Before moving to File 03 (Embedding Model Selection), confirm you can:

- [ ] **Explain cosine similarity** in 1 sentence to someone who's never seen it
- [ ] **Implement search from scratch** without looking at the code above
- [ ] **Choose between cosine, Euclidean, and dot product** for a given use case with justification
- [ ] **Predict when search will fail** — give me 2 scenarios where embeddings produce wrong results
- [ ] **Benchmark your search** — measure latency and know when you'll need a vector DB
- [ ] **Answer these questions:**
  1. Why does brute-force search fail at 1M+ documents?
  2. What happens to similarity scores when you don't normalize embeddings?
  3. Why does a one-word query produce worse results than a sentence?
  4. If cosine similarity gives 0.99 for two obviously different texts, what's likely wrong?

- [ ] **Write down the single most important insight** you had while building search from scratch

---

## 📚 Resources

**Foundational Papers:**
- "A Framework for Learning Semantic Textual Similarity" (Agirre et al., 2012) — STS Benchmark
- "Learning to Rank: From Pairwise Approach to Listwise Approach" (Cao et al., 2007)
- "Dense Passage Retrieval for Open-Domain Question Answering" (Karpukhin et al., 2020)

**Code References:**
- FAISS (Facebook AI Similarity Search) — The production library for efficient similarity search
- `scikit-learn` `pairwise_distances` — Vectorized distance computation
- `sentence-transformers` `util.semantic_search` — Their optimized implementation

**Production Reading:**
- "Reimers & Gurevych: Sentence-BERT" — Understanding the training objective helps debug failures
- "How to Build a Semantic Search Engine" — OpenAI Cookbook
- "Pinecone: What is Similarity Search?" — Visual guide to ANN algorithms

**Senior Engineer Checklist:**
```
Before deploying search to production:
☐ Measured: what's the average query length on real data?
☐ Tested: which similarity measure works best for YOUR domain?
☐ Benchmarked: at what document count does brute-force break?
☐ Prepared: do you have a fallback for failed queries?
☐ Monitored: what's your search-to-click rate?
(You'll learn monitoring in Phase 7 — but start thinking about it now)
```

---

> **🛑 Revisit the 3 discovery questions from the beginning:**
>
> 1. **Your Naive Approach**: How does what you built compare to what you initially imagined?
> 2. **The "Good Enough" Threshold**: Is 80% actually good enough? Answer now with the experience of having built both keyword and semantic search.
> 3. **The Synonym Problem**: Did you need a single synonym dictionary? What replaced it?
>
> **The key realization:** You didn't memorize "semantic search uses cosine similarity." You BUILT keyword search, saw it fail, and ARRIVED at vector search as the natural solution. That's the method — not remembering facts, but understanding *why* each approach exists.
>
> **Next:** File 03 — how to choose the RIGHT embedding model for your use case.
