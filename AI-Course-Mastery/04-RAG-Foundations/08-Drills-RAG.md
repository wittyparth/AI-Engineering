# 08 — RAG Drills & Challenges

## 🎯 Purpose & Goals

This module is different. No new concepts. No new theory. Every other file in Phase 4 taught you *what* to build and *why*. This module makes you *actually do it*.

These drills are designed to be **hard**. Not because they use complex algorithms, but because they force you to debug, diagnose, and make decisions with incomplete information — exactly like production RAG engineering.

**By completing this module, you will have:**

1. Debugged 3 broken RAG pipelines (each broken in a different way)
2. Built a complete RAG system from scratch (no frameworks, just your own code)
3. Found and fixed performance bottlenecks in retrieval, context integration, and generation
4. Run a comprehensive evaluation and made data-driven improvements
5. Deployed a RAG system with monitoring, caching, and graceful degradation

**Time budget:** This is not a module to rush. Each drill is designed for 30-60 minutes of focused work. Some will take you longer. That's intentional.

**Rules of engagement:**
- **No AI assistance for the drills.** You can use the Phase 4 files as reference, but don't ask an LLM to write the code for you. The point is to struggle and figure it out.
- **Type every line.** No copy-paste. Typing builds muscle memory and forces you to read what you're writing.
- **Break things.** If a drill says "run this test" and it fails, that's the point. Debug it. Fix it. Learn from the failure mode.
- **Write down what you learn.** Keep a log of each drill: what worked, what didn't, what surprised you.

---

## ⏱ Time Budget

| Drill | Estimated Time | Difficulty |
|-------|---------------|------------|
| Drill 1: Debug the Broken RAG Pipeline | 45 min | Medium |
| Drill 2: Build RAG from Scratch | 90 min | Hard |
| Drill 3: The Chunking Optimization Challenge | 45 min | Medium |
| Drill 4: The Contradiction Autopsy | 35 min | Medium |
| Drill 5: Build an Evaluation Harness | 60 min | Hard |
| Drill 6: The Cache Design Problem | 40 min | Medium |
| Drill 7: The Production Incident | 45 min | Hard |
| Drill 8: Cross-Domain RAG Design | 35 min | Medium |
| Drill 9: The Multi-Turn Disaster | 45 min | Hard |
| Drill 10: The Full Stack Review | 60 min | Hard |
| **Total** | **~8 hours** | |

---

## Drill 1: Debug the Broken RAG Pipeline

**Type:** Debugging  
**Difficulty:** Medium  
**Time:** 45 minutes  

### The Setup

You've inherited a RAG pipeline from a colleague who left the company. It looks functional at first glance, but it produces poor answers. Here's the code:

```python
"""
broken_rag.py — A RAG pipeline with 7 hidden bugs.

Your mission: Find and fix all 7 bugs.
Run the test queries and observe the answers.
Each answer has a specific problem caused by one or more bugs.
"""

import numpy as np
from typing import List, Dict
import tiktoken


# === BROKEN CHUNKING ===

def chunk_document(text: str, chunk_size: int = 500) -> List[str]:
    """Split text into chunks."""
    # BUG 1: This doesn't handle chunk_size correctly
    # BUG 2: Chunks can overlap content from different paragraphs
    chunks = []
    words = text.split()
    for i in range(0, len(words), chunk_size):
        chunk = " ".join(words[i:i + chunk_size])
        chunks.append(chunk)
    return chunks


# === BROKEN EMBEDDING (SIMULATED) ===

def embed_text(text: str) -> List[float]:
    """Generate embedding for text."""
    # Simulated embedding — in production, use an actual model
    np.random.seed(hash(text) % (2**31))
    return list(np.random.randn(384).astype(np.float32))


# === BROKEN RETRIEVAL ===

def cosine_similarity(a: List[float], b: List[float]) -> float:
    """Compute cosine similarity."""
    a, b = np.array(a), np.array(b)
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))


def retrieve(
    query: str,
    chunks: List[str],
    embeddings: List[List[float]],
    top_k: int = 3
) -> List[str]:
    """Retrieve top-k chunks for a query."""
    query_emb = embed_text(query)

    # BUG 3: Similarity calculation is inverted (lower = more similar?)
    similarities = []
    for chunk_emb in embeddings:
        sim = cosine_similarity(query_emb, chunk_emb)
        similarities.append(sim)

    # BUG 4: Sorting direction is wrong
    top_indices = np.argsort(similarities)[:top_k]

    return [chunks[i] for i in top_indices]


# === BROKEN CONTEXT INTEGRATION ===

def build_context(chunks: List[str], query: str) -> str:
    """Build the context prompt."""
    # BUG 5: No instructions about how to use the context
    # BUG 6: No citation markers
    # BUG 7: Chunks are in retrieval order (no position optimization)
    context = "\n\n".join(chunks)
    prompt = f"""Answer the question based on this information:

{context}

Question: {query}
Answer:"""
    return prompt


# === TEST DOCUMENTS ===

documents = [
    """
    The API rate limit for the free tier is 100 requests per minute.
    For the pro tier, the rate limit is 1000 requests per minute.
    Enterprise customers have a rate limit of 10000 requests per minute.
    Rate limits are reset every hour at the top of the hour.
    """,

    """
    API authentication requires an API key. You can generate an API key
    from the Settings > API Keys page in your dashboard. API keys have
    full access to your account. Keep your API keys secure and never
    share them. Rotate keys every 90 days for security.
    """,

    """
    Webhooks allow you to receive real-time notifications when events
    occur in your account. To set up a webhook, go to Settings >
    Webhooks and enter your endpoint URL. We'll send a POST request
    with a JSON payload for each event.
    """,

    """
    The free tier includes 1000 API calls per month, 1 webhook,
    7-day data retention, and community support. The pro tier includes
    50000 API calls per month, 10 webhooks, 30-day data retention,
    and email support. Enterprise plans have custom limits.
    """,
]


# === TEST QUERIES ===

test_queries = [
    "What's the rate limit for the free tier?",
    "How do I authenticate with the API?",
    "What's included in the pro tier?",
    "Can I receive real-time notifications?",
]


# === RUN THE BROKEN SYSTEM ===

def run_broken_rag():
    """Run the broken RAG pipeline."""

    # Index documents
    all_chunks = []
    all_embeddings = []
    for doc in documents:
        chunks = chunk_document(doc)
        all_chunks.extend(chunks)
        for chunk in chunks:
            all_embeddings.append(embed_text(chunk))

    print(f"Indexed {len(all_chunks)} chunks\n")

    for query in test_queries:
        print(f"Query: {query}")
        print("-" * 50)

        retrieved = retrieve(query, all_chunks, all_embeddings, top_k=3)

        print("Retrieved chunks:")
        for i, chunk in enumerate(retrieved, 1):
            print(f"  [{i}] {chunk[:100]}...")

        prompt = build_context(retrieved, query)
        print(f"\nPrompt ({len(prompt)} chars):")
        print(prompt[:200])
        print(f"\n{'='*60}\n")


if __name__ == "__main__":
    run_broken_rag()
```

### The Symptoms

When you run this pipeline, you'll observe:

1. **Retrieval returns wrong chunks** — chunks that don't answer the query appear in top-3
2. **The LLM answer misses key information** — even when the right chunk is retrieved, it's not used properly
3. **Answers lack citations** — the model doesn't indicate which chunk it's using
4. **Chunks are cut off mid-sentence** — the chunking destroys paragraph boundaries
5. **The prompt has no structure** — context and instructions blend together

### Your Tasks

1. **Identify all 7 bugs** (they're marked with `# BUG` comments, but there may be more)
2. **Fix each bug** with a corrected implementation
3. **Add 3 improvements** that aren't bug fixes but would meaningfully improve output quality
4. **Run the fixed pipeline** on the test queries and document the improvements
5. **Add a test case** that would have caught the original bugs if it existed

### Success Criteria

- Retrieval returns relevant chunks for all 4 test queries
- The context prompt has: clear structure, citation markers, explicit instructions
- Chunks don't break mid-sentence
- The pipeline handles edge cases (empty query, single document, very long document)

### Reflection Questions

After fixing the pipeline, answer:
- Which bug was hardest to find? Why?
- Which fix had the biggest impact on answer quality?
- If you had to add one more feature to this pipeline (from Phase 4 modules), what would it be and why?

---

## Drill 2: Build RAG from Scratch

**Type:** Implementation  
**Difficulty:** Hard  
**Time:** 90 minutes  

### The Challenge

No frameworks. No LangChain. No LlamaIndex. No ChromaDB. Build a functional RAG system using ONLY:
- Standard library (`json`, `hashlib`, `math`)
- `numpy` (for vector operations)
- `httpx` or `openai` (for LLM API calls)
- `tiktoken` (for token counting)

### Specifications

Your system must:

1. **Ingest documents** from a directory of .txt files
2. **Chunk** each document intelligently (respect paragraph boundaries, configurable chunk size)
3. **Embed** chunks using an embedding API (any provider)
4. **Store** embeddings in a simple in-memory index (FAISS is allowed, or you can implement brute-force search)
5. **Retrieve** top-k chunks for a query
6. **Build context** with proper structure, citations, and position optimization
7. **Generate** an answer using an LLM API
8. **Return** the answer with source citations

### What We're Testing

- Can you implement retrieval without a vector database library?
- Can you build a context prompt that actually makes the LLM use the context?
- Can you handle edge cases (no relevant chunks, query too long, empty documents)?
- Can you structure the code so someone else could understand it?

### Starter Code

```python
"""
build_your_own_rag.py — Implement RAG from scratch.

No vector DB frameworks allowed. No LangChain. No LlamaIndex.
"""

import numpy as np
from pathlib import Path
from typing import List, Dict, Optional
import json
import hashlib


class SimpleVectorIndex:
    """
    In-memory vector index.

    Implement: add(), search(), save(), load()
    """

    def __init__(self):
        self.vectors = []
        self.metadata = []

    def add(self, vector: List[float], metadata: Dict):
        """Add a vector to the index."""
        # Your implementation here
        pass

    def search(self, query_vector: List[float], top_k: int = 5) -> List[Dict]:
        """Search for similar vectors."""
        # Your implementation here
        # Must return: [{id, score, metadata}]
        pass

    def save(self, path: str):
        """Persist index to disk."""
        # Your implementation here
        pass

    def load(self, path: str):
        """Load index from disk."""
        # Your implementation here
        pass


class Chunker:
    """
    Intelligent text chunker.

    Must handle:
    - Paragraph boundary detection
    - Configurable chunk size (tokens)
    - Overlap between chunks
    - Graceful handling of small documents
    """

    def __init__(self, chunk_size: int = 500, overlap: int = 50):
        self.chunk_size = chunk_size
        self.overlap = overlap

    def chunk(self, text: str, metadata: Dict) -> List[Dict]:
        """Split text into chunks with metadata."""
        # Your implementation here
        pass


class ContextBuilder:
    """
    Build optimized context prompts.

    Must handle:
    - Citation formatting
    - Position optimization (Lost in the Middle)
    - Token budget management
    - Dynamic chunk selection
    """

    def build(self, chunks: List[Dict], query: str, max_tokens: int = 3000) -> str:
        """Build an optimized context prompt."""
        # Your implementation here
        pass


class RAGSystem:
    """
    Complete RAG system from scratch.

    Brings together: ingestion, indexing, retrieval, context, generation
    """

    def __init__(
        self,
        embedding_api_key: str,
        llm_api_key: str,
        embedding_model: str = "text-embedding-3-small",
        llm_model: str = "gpt-4o-mini"
    ):
        self.embedding_api_key = embedding_api_key
        self.llm_api_key = llm_api_key
        self.embedding_model = embedding_model
        self.llm_model = llm_model

        self.index = SimpleVectorIndex()
        self.chunker = Chunker(chunk_size=500, overlap=50)
        self.context_builder = ContextBuilder()

    async def embed(self, text: str) -> List[float]:
        """Embed text using the embedding API."""
        # Your implementation here
        pass

    async def ingest_document(self, file_path: str):
        """Ingest a single document."""
        # Your implementation here
        pass

    async def ingest_directory(self, dir_path: str):
        """Ingest all documents in a directory."""
        # Your implementation here
        pass

    async def retrieve(self, query: str, top_k: int = 5) -> List[Dict]:
        """Retrieve relevant chunks for a query."""
        # Your implementation here
        pass

    async def answer(self, query: str) -> Dict:
        """Answer a query using RAG."""
        # Your implementation here
        # Must return: {answer, sources, latency}
        pass

    def save_index(self, path: str):
        """Persist the vector index."""
        self.index.save(path)

    def load_index(self, path: str):
        """Load a persisted index."""
        self.index.load(path)
```

### Deliverables

1. A working `RAGSystem` class that passes the specification tests
2. At least 3 sample documents ingested and queryable
3. Test output showing: query → retrieved chunks → context → answer
4. A brief write-up (200 words max) on one design decision you made and why

### Evaluation Criteria

| Criterion | Pass | Exceeds |
|-----------|------|---------|
| Retrieval returns relevant results | Relevant in top-5 | Relevant in top-3 for 8/10 test queries |
| Context is well-structured | Has citation markers | Has position optimization + token budget management |
| Answer uses context correctly | Answer cites sources | Answer handles contradictions gracefully |
| Code quality | Functions have docstrings | Tests included for edge cases |
| Error handling | Doesn't crash on empty documents | Graceful degradation for API failures |

---

## Drill 3: The Chunking Optimization Challenge

**Type:** Optimization  
**Difficulty:** Medium  
**Time:** 45 minutes  

### The Setup

You're given this document and 5 test queries:

```python
document = """
Introduction to Machine Learning

Machine learning is a subset of artificial intelligence that enables systems
to learn and improve from experience without being explicitly programmed.
It focuses on the development of computer programs that can access data
and use it to learn for themselves.

Types of Machine Learning

There are three main types of machine learning: supervised learning,
unsupervised learning, and reinforcement learning. Each type has its own
characteristics and use cases.

Supervised Learning

Supervised learning is the most common type of machine learning. In this
approach, the algorithm is trained on a labeled dataset, where each
training example is paired with an output label. The algorithm learns to
map inputs to outputs based on these examples. Common algorithms include
linear regression, decision trees, and neural networks.

Unsupervised Learning

Unsupervised learning involves training on data without labeled responses.
The algorithm must find patterns and relationships in the data on its own.
Common techniques include clustering (K-means, hierarchical) and
dimensionality reduction (PCA, t-SNE).

Reinforcement Learning

Reinforcement learning is about taking suitable actions to maximize reward
in a particular situation. The agent learns from the consequences of its
actions, rather than from being taught explicitly. It has been successfully
applied to game playing, robotics, and navigation.

Deep Learning

Deep learning is a subset of machine learning that uses neural networks
with many layers (hence "deep"). These deep neural networks can learn
complex patterns in large amounts of data. Deep learning has driven
advances in computer vision, natural language processing, and speech
recognition.

Applications

Machine learning has numerous practical applications:
- Image recognition and computer vision
- Natural language processing and text analysis
- Recommendation systems
- Fraud detection
- Predictive maintenance
- Autonomous vehicles

Each application area uses different combinations of the techniques
described above, chosen based on the specific requirements of the task.
"""

test_queries = [
    "What's the difference between supervised and unsupervised learning?",
    "What are the applications of machine learning?",
    "How does reinforcement learning work?",
    "What is deep learning and how is it different from regular machine learning?",
    "What are the three main types of machine learning?",
]
```

### The Challenge

Your task is to find the **optimal chunking strategy** for this document and these queries. You need to determine:

1. **Chunk size**: What's the best chunk size (in tokens)?
2. **Chunking method**: Fixed-size, paragraph-based, or semantic?
3. **Overlap**: How much overlap between chunks?
4. **Number of chunks to retrieve**: What's the optimal top-K?

### Required Tests

For each of 5 chunking configurations, measure:

```python
def evaluate_chunking_strategy(
    document: str,
    queries: List[str],
    chunk_size: int,
    chunk_method: str,
    overlap: int
) -> Dict:
    """
    Evaluate a chunking strategy.

    Measures:
    - Number of chunks produced
    - Average chunk length (tokens)
    - For each query: does the correct information appear in top-3 retrieved chunks?
    - Coverage: what % of the document's key facts are retrievable?

    Returns: metrics for this configuration
    """
    # Your implementation here
    pass
```

### Configurations to Test

| Config | Method | Chunk Size | Overlap |
|--------|--------|------------|---------|
| A | Fixed-size | 200 tokens | 0 |
| B | Fixed-size | 500 tokens | 50 |
| C | Paragraph | N/A | 0 |
| D | Paragraph | N/A | 1 sentence |
| E | Fixed-size | 1000 tokens | 100 |

### Analysis Questions

1. Which configuration performs best for queries that span multiple sections (like "What are the three main types?")?
2. Which configuration performs best for specific factual queries (like "How does reinforcement learning work?")?
3. What's the tradeoff between chunk count (more = better coverage) and average chunk quality (larger = more context)?
4. If you could only use one configuration for ALL queries, which would you choose and why?
5. Bonus: Does the optimal strategy change if you have 10 similar documents vs. 1 long document vs. 100 short documents?

### Deliverables

A short report with:
- The optimal configuration you found
- The data that supports your decision
- One scenario where your optimal configuration performs poorly (so you know its limitations)
- The SQL query or Python code you'd use to find this optimal configuration automatically

---

## Drill 4: The Contradiction Autopsy

**Type:** Debugging + Analysis  
**Difficulty:** Medium  
**Time:** 35 minutes  

### The Setup

Your RAG system retrieved these 4 chunks for the query: "What's the storage limit on the basic tier?"

```python
chunks = [
    {
        "id": "C1",
        "text": "Basic tier includes 10GB of storage, 1000 API calls per month, and community support.",
        "score": 0.94,
        "document_title": "Pricing Guide v2.1",
        "last_updated": "2023-06-15"
    },
    {
        "id": "C2",
        "text": "All tiers include unlimited storage. Basic, Pro, and Enterprise plans all feature unlimited storage with no caps.",
        "score": 0.91,
        "document_title": "Features Overview",
        "last_updated": "2024-01-20"
    },
    {
        "id": "C3",
        "text": "Storage limits: Basic = 10GB, Pro = 100GB, Enterprise = 1TB. Additional storage can be purchased at $0.10/GB/month.",
        "score": 0.87,
        "document_title": "Technical Specifications",
        "last_updated": "2023-09-01"
    },
    {
        "id": "C4",
        "text": "Basic tier now includes 50GB of storage as of the January 2024 update. Existing customers were automatically upgraded.",
        "score": 0.82,
        "document_title": "Changelog 2024",
        "last_updated": "2024-01-25"
    },
]
```

### The Problem

These chunks contain multiple contradictions:

- C1 says 10GB, C2 says unlimited, C3 says 10GB, C4 says 50GB
- C2 directly contradicts all other chunks ("unlimited" vs. specific limits)
- C4 is the most recent but has the LOWEST retrieval score

### Your Tasks

1. **Build a contradiction detector**: Write code that identifies contradictory claims across these chunks
2. **Prioritize by recency**: Implement a version-aware selection that prefers more recent information
3. **Generate a contradiction-aware prompt**: Create a context that explicitly highlights the contradictions and asks the LLM to handle them
4. **Design the ideal output**: What should the RAG system answer given these contradictory sources?

### Contradiction-Aware Prompt Design

Design a prompt that, given these chunks, would produce an answer that:

1. NOTES the conflict explicitly
2. IDENTIFIES which source is most recent and likely authoritative
3. PRESENTS the information clearly without hiding the uncertainty
4. ADVISES the user on next steps (verify with sales team, check their actual account)

### Deliverables

```python
# Your code should:
def detect_contradictions(chunks: List[Dict]) -> List[Dict]:
    """
    Identify contradictory claims across chunks.

    Returns list of contradictions: {claim_a, claim_b, chunk_a_id, chunk_b_id, severity}
    """
    # Your implementation
    pass

def build_contradiction_aware_context(chunks: List[Dict]) -> str:
    """
    Build context that explicitly handles contradictions.
    """
    # Your implementation
    pass

def ideal_answer(chunks: List[Dict]) -> str:
    """
    What the ideal RAG answer looks like given contradictory sources.
    """
    # Your implementation (write the expected answer)
    pass
```

---

## Drill 5: Build an Evaluation Harness

**Type:** Implementation  
**Difficulty:** Hard  
**Time:** 60 minutes  

### The Challenge

Build a complete RAG evaluation harness from scratch. No `ragas` library allowed. You must implement the metrics yourself using LLM-as-judge.

### Specifications

Your evaluation harness must:

1. **Load evaluation dataset** from a JSON file with: `query`, `gold_answer`, `gold_contexts`
2. **Run RAG system** on each query
3. **Compute metrics**:
   - **Faithfulness** (fraction of answer claims supported by context)
   - **Answer Relevance** (how well the answer addresses the query)
   - **Context Precision** (fraction of retrieved chunks that are relevant)
   - **Context Recall** (fraction of needed information that was retrieved)
4. **Cache results** (don't re-evaluate the same (query, answer) pair)
5. **Generate report** with per-query scores and aggregate statistics

### Evaluation Dataset

Create your own evaluation dataset with at least 15 queries covering:

- 5 easy queries (single chunk, direct answer)
- 5 medium queries (need 2-3 chunks, some inference)
- 5 hard queries (need 4+ chunks, cross-document synthesis, or handle contradictions)

### Metrics Implementation

```python
class FaithfulnessMetric:
    """Implement faithfulness evaluation using LLM-as-judge."""

    async def score(self, answer: str, contexts: List[str]) -> float:
        """
        Extract claims from answer, check each against contexts.

        Returns: 0.0 (no claims supported) to 1.0 (all claims supported)
        """
        # Your implementation here
        pass


class AnswerRelevanceMetric:
    """Implement answer relevance evaluation."""

    async def score(self, question: str, answer: str) -> float:
        """
        Generate reverse questions from answer, compare to original.

        Returns: 0.0 to 1.0
        """
        # Your implementation here
        pass


class ContextPrecisionMetric:
    """Implement context precision evaluation."""

    async def score(self, question: str, contexts: List[str]) -> float:
        """
        Check each chunk for relevance, weighted by position.

        Returns: 0.0 to 1.0
        """
        # Your implementation here
        pass


class ContextRecallMetric:
    """Implement context recall evaluation."""

    async def score(self, question: str, answer: str, contexts: List[str]) -> float:
        """
        Check if contexts contain ALL info needed for answer.

        Returns: 0.0 to 1.0
        """
        # Your implementation here
        pass
```

### Bonus: Metric Correlation Analysis

Run your evaluation on 3 different versions of your RAG system (e.g., different chunk sizes, different top-K, with/without reranking). For each pair of metrics, compute the correlation:

```python
def analyze_metric_correlation(results: List[Dict]) -> Dict:
    """
    For each pair of metrics, compute correlation coefficient.

    This tells you: when faithfulness goes up, does relevance also go up?
    Or do they trade off against each other?
    """
    # Your implementation
    pass
```

### Success Criteria

- Your faithfulness metric correctly identifies hallucinated claims
- Your context precision metric penalizes irrelevant chunks
- Your evaluation harness runs 15 queries in under 5 minutes (with caching)
- The report includes: per-query scores, aggregate statistics, and low-scoring query analysis

---

## Drill 6: The Cache Design Problem

**Type:** Design + Implementation  
**Difficulty:** Medium  
**Time:** 40 minutes  

### The Scenario

You have a RAG system serving 50,000 queries/day. Your costs are $300/day. Your CTO wants to reduce costs by 40% through caching.

### The Data

After analyzing query logs for 30 days, you find:

```
Query repetition patterns:
- 25% of queries are exact repeats (same user asking same question again)
- 15% of queries are semantic repeats (same intent, different words)
- 10% are near-duplicates (minor wording changes)
- 50% are truly unique queries

Query timing:
- 60% of repeat queries happen within 1 hour of the original
- 25% happen within 1-24 hours
- 15% happen after 24 hours

Document update frequency:
- Pricing docs: updated monthly
- Product docs: updated quarterly
- API docs: updated bi-weekly
- FAQ: updated weekly
```

### Your Tasks

1. **Design a multi-level cache** with different TTLs for different document types
2. **Estimate the savings** — calculate the expected cache hit rate and cost reduction
3. **Build the cache invalidation strategy** — what happens when each document type is updated?

### Implementation

```python
class ProductionCache:
    """
    Multi-level cache for production RAG.

    Level 1: Exact match (in-memory, fast, limited TTL)
    Level 2: Semantic match (embedding-based, slower, longer TTL)
    Level 3: Conditional (serves cached answer only if confident)

    Each level has different TTLs based on document type.
    """

    def __init__(self):
        # Your configuration here
        pass

    async def get(
        self,
        query: str,
        doc_types: List[str]  # Types of documents used in the context
    ) -> Optional[Dict]:
        """
        Get cached answer if available and fresh.

        Must check: is the cached answer still valid given
        the document types involved?
        """
        # Your implementation
        pass

    async def set(
        self,
        query: str,
        answer: str,
        sources: List[Dict],
        doc_types: List[str]
    ):
        """Cache an answer with document-type awareness."""
        # Your implementation
        pass

    async def invalidate_document_type(self, doc_type: str):
        """Invalidate all cached answers using a specific document type."""
        # Your implementation
        pass
```

### Analysis

After implementing, calculate:

1. Expected cache hit rate: _____%
2. Expected daily cost after caching: $_____
3. Expected p95 latency improvement: _____%
4. Stale answer risk: _____% of queries might get slightly outdated info
5. How would you detect and measure stale answers in production?

---

## Drill 7: The Production Incident

**Type:** Troubleshooting  
**Difficulty:** Hard  
**Time:** 45 minutes  

### The Incident Report

```
INCIDENT: RAG-2024-0042
DATE: 2024-03-15
TIMELINE:

14:00 - User reports "the bot told me my plan has unlimited storage,
         but my dashboard shows 10GB and I can't upload files"

14:05 - Support team confirms: 12 similar reports in the last hour

14:10 - You check the RAG system logs:
        Query: "How much storage do I have on the basic tier?"
        Retrieved chunks:
          - C1: "Basic tier includes 10GB of storage" (score: 0.91, doc: pricing-v2.1)
          - C2: "All tiers feature unlimited storage" (score: 0.88, doc: features-overview)
          - C3: "Storage limits are based on your plan tier" (score: 0.72, doc: faq)
        Generated answer: "Your basic tier plan includes unlimited storage."

14:15 - You discover: the "features-overview" document is a MARKETING document
         that describes the aspirational roadmap, not current features.
         It was indexed by mistake.

14:20 - You remove the document from the index.
         But cached answers still serve the wrong information.

14:30 - You clear the cache. The system starts serving correct answers.
         But now 15 minutes have passed with wrong answers to hundreds of users.
```

### Root Cause Analysis

Identify the MULTIPLE failures that allowed this incident:

1. **Ingestion failure**: How was a marketing document indexed alongside technical docs?
2. **Retrieval failure**: Why did a marketing doc with different intent rank highly?
3. **Context integration failure**: Why didn't the system detect the contradiction?
4. **Generation failure**: Why did the LLM choose the marketing claim over the technical one?
5. **Monitoring failure**: Why didn't any alert fire when this started happening?

### Your Tasks

1. **Write the post-mortem**: A structured incident analysis with:
   - What happened (user-visible impact)
   - Root cause (not the document, but the systemic failure)
   - Why existing defenses didn't catch it
   - Timeline (what happened when)

2. **Design the prevention system**: A multi-layered defense that prevents this specific incident AND the general class of incidents it belongs to:
   - Layer 1: Ingestion guard (prevent wrong documents from being indexed)
   - Layer 2: Retrieval guard (detect out-of-domain chunks)
   - Layer 3: Context guard (detect contradictions)
   - Layer 4: Generation guard (verify output claims against trusted sources)
   - Layer 5: Monitoring (alert when unexpected patterns emerge)

3. **Implement one guard**: Choose ONE layer and implement it in code:

```python
# Example: Layer 3 — Context Contradiction Guard
class ContextContradictionGuard:
    """
    Detect contradictions between retrieved chunks BEFORE
    passing them to the LLM.

    If contradictions are found:
    1. Log warning with contradiction details
    2. Flag the answer for human review
    3. Add contradiction warning to prompt
    4. (Optional) Suppress less authoritative source
    """

    async def check_context(
        self,
        chunks: List[Chunk],
        query: str
    ) -> ContextCheckResult:
        """
        Check context for contradictions.

        Returns: {
            has_contradiction: bool,
            conflicting_chunks: List[Tuple],
            recommended_action: str,
            severity: str
        }
        """
        # Your implementation
        pass
```

### Deliverables

1. Post-mortem document with root cause analysis
2. Design document for the prevention system
3. Working implementation of one prevention layer with tests

---

## Drill 8: Cross-Domain RAG Design

**Type:** Design  
**Difficulty:** Medium  
**Time:** 35 minutes  

### The Scenario

You need to design a RAG system for a company with three completely different knowledge domains:

- **Legal documents**: Contracts, terms of service, privacy policies. MUST be 100% accurate. Citations are MANDATORY. Errors can result in lawsuits.
- **Engineering docs**: API references, SDK guides, architecture docs. Should be accurate but tolerates some ambiguity. Code examples must be runnable.
- **Sales materials**: Product comparisons, pricing, case studies. Can be promotional. Must cite sources but has more flexibility.

### The Design Problem

A single RAG system with one configuration can't serve all three domains well. Design a **domain-aware RAG system** that adapts its behavior based on the query's domain.

### Your Tasks

1. **Design the domain classifier**: How does the system know if a query is about legal, engineering, or sales?

2. **Specify per-domain configurations**:

| Configuration | Legal | Engineering | Sales |
|--------------|-------|-------------|-------|
| **Top-K chunks** | ? | ? | ? |
| **Chunk size** | ? | ? | ? |
| **Model** | ? | ? | ? |
| **Citation requirement** | ? | ? | ? |
| **Contradiction handling** | ? | ? | ? |
| **Fallback behavior** | ? | ? | ? |

3. **Implement the domain dispatcher**:

```python
class DomainAwareRAG:
    """
    RAG system that adapts to query domain.

    Classifies the query into a domain, then uses domain-specific
    configurations for retrieval, context, and generation.
    """

    async def classify_domain(self, query: str) -> str:
        """Classify query into 'legal', 'engineering', or 'sales'."""
        # Your implementation
        pass

    def get_domain_config(self, domain: str) -> Dict:
        """Get configuration for a specific domain."""
        # Your implementation
        pass

    async def answer(self, query: str) -> Dict:
        """Answer with domain-appropriate configuration."""
        domain = await self.classify_domain(query)
        config = self.get_domain_config(domain)
        # Use config for retrieval, context, generation
        pass
```

### Edge Cases to Handle

1. **Ambiguous query**: "What are the limits?" — could be legal (liability limits), engineering (rate limits), or sales (plan limits)
2. **Cross-domain query**: "What does the contract say about API rate limits?" — legal AND engineering
3. **Out-of-domain query**: "What's the weather?" — doesn't match any domain

### Deliverables

- Domain classification strategy with justification
- Completed configuration table with reasoning for each choice
- Edge case handling strategy
- Brief: How would you evaluate whether your domain-aware system is better than a generic one?

---

## Drill 9: The Multi-Turn Disaster

**Type:** Debugging + Design  
**Difficulty:** Hard  
**Time:** 45 minutes  

### The Scenario

You're building a customer support RAG bot. It needs to handle multi-turn conversations. After 8 exchanges with a user, here's what happens:

```python
# Current state after 8 turns

conversation_history = [
    {"role": "user", "content": "What's your return policy?"},
    {"role": "assistant", "content": "Returns within 30 days of purchase. Items must be in original condition."},
    {"role": "user", "content": "Do I need the original packaging?"},
    {"role": "assistant", "content": "Yes, original packaging is required for returns."},
    {"role": "user", "content": "What about electronics?"},
    {"role": "assistant", "content": "Electronics have a 15-day return window and a 15% restocking fee."},
    {"role": "user", "content": "Can I return a laptop after 20 days?"},
    {"role": "assistant", "content": "No, laptops are electronics and have a 15-day return window."},
    {"role": "user", "content": "That's frustrating. What if it's defective?"},
    {"role": "assistant", "content": "Defective items can be returned within 30 days regardless of category."},
    {"role": "user", "content": "Okay, I bought a laptop 18 days ago and it won't charge."},
    {"role": "assistant", "content": "Since it's defective, you can return it within 30 days. Please start a return... wait, but laptops are electronics with 15-day window. I'm confused. Let me check..."},  # The bot contradicted itself!
    {"role": "user", "content": "So can I return it or not? I need a clear answer."},
    # Turn 9: The current query
    {"role": "user", "content": "The laptop is 18 days old and won't charge. Can I return it?"},
]

# The problem: 
# 1. The bot gave conflicting answers (15-day vs. 30-day for defective)
# 2. The history is now 8 turns = ~2000 tokens
# 3. The current query requires info from the return policy doc PLUS
#    the conversation context (they already explained the situation)
# 4. The bot needs to reconcile: "electronics 15 days" vs. "defective 30 days"
```

### Your Tasks

1. **Identify the failures**: What went wrong in the bot's reasoning chain?

2. **Design a conversation-aware context strategy**:
   - How do you compress the 8-turn history without losing critical context?
   - What information MUST be preserved from the conversation for the current turn?
   - What can be safely summarized?

3. **Implement history compression**:

```python
class ConversationCompressor:
    """
    Compress conversation history for multi-turn RAG.

    Must preserve:
    - User's identity/context (who they are, what plan they're on)
    - Already established facts ("user was told electronics have 15-day window")
    - Unresolved questions ("user hasn't confirmed their laptop purchase date")
    - Inconsistencies that need resolution

    Can discard:
    - Confirmed information that doesn't affect current query
    - Greetings and meta-conversation
    - Redundant exchanges
    """

    async def compress(
        self,
        history: List[Dict],
        current_query: str
    ) -> Dict:
        """
        Compress conversation history.

        Returns:
        {
            "summary": str (compressed history),
            "retained_turns": List[Dict] (last 2 turns verbatim),
            "key_facts": List[str] (facts established in earlier turns),
            "pending_issues": List[str] (unresolved items)
        }
        """
        # Your implementation here
        pass
```

4. **Design the multi-turn RAG prompt** that incorporates:
   - Compressed conversation history
   - Relevant retrieved chunks (for the current query about laptop return)
   - Explicit contradiction resolution (15-day electronics vs. 30-day defective)
   - Clear instruction to resolve conflicts based on conversation context

### Expected Output

Write the prompt that your system would generate for Turn 9, given:
- The compressed conversation history
- Retrieved chunks about return policy (with the electronics exception and defective clause)

The prompt should guide the LLM to:
1. Recognize the user has already been told conflicting information
2. Acknowledge the confusion
3. Give a CLEAR answer to the specific situation (18-day-old defective laptop)
4. Explain why this answer overrides the previous one

---

## Drill 10: The Full Stack Review

**Type:** Refactoring + Design  
**Difficulty:** Hard  
**Time:** 60 minutes  

### The Scenario

You're reviewing a junior engineer's RAG system code. They've built a working system, but it has issues you need to identify and fix.

```python
"""
review_me.py — A junior engineer's RAG system for code review.
"""

import openai
import chromadb
from typing import List
import numpy as np


class SimpleRAG:
    def __init__(self, api_key: str):
        openai.api_key = api_key
        self.chroma_client = chromadb.Client()
        self.collection = self.chroma_client.create_collection("docs")

    def add_documents(self, texts: List[str]):
        ids = [str(i) for i in range(len(texts))]
        # Get embeddings
        response = openai.Embedding.create(
            model="text-embedding-3-small",
            input=texts
        )
        embeddings = [d["embedding"] for d in response["data"]]
        self.collection.add(
            embeddings=embeddings,
            documents=texts,
            ids=ids
        )

    def query(self, question: str):
        # Get query embedding
        response = openai.Embedding.create(
            model="text-embedding-3-small",
            input=question
        )
        query_embedding = response["data"][0]["embedding"]

        # Search
        results = self.collection.query(
            query_embeddings=[query_embedding],
            n_results=5
        )

        # Build prompt
        context = "\n\n".join(results["documents"][0])
        prompt = f"""Answer the question based on this context:

{context}

Question: {question}
Answer:"""

        # Generate
        response = openai.ChatCompletion.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "You are a helpful assistant."},
                {"role": "user", "content": prompt}
            ]
        )

        return response.choices[0].message.content
```

### Identify the Issues

This code has at least 10 issues ranging from critical bugs to bad practices. Find them all.

**Categories to check:**
1. **Architecture**: Single-threaded, no separation of concerns
2. **Reliability**: No error handling, no retries, no fallbacks
3. **Performance**: Embedding every time, no caching
4. **Quality**: No context optimization, no citations, no instruction
5. **Security**: API key exposure pattern, no input validation
6. **Monitoring**: No logging, no metrics, no tracing
7. **Scalability**: Creates a new ChromaDB collection every time

### Your Tasks

1. **List every issue** you find, categorized by severity (critical/high/medium/low)
2. **Fix the top 5 critical issues** with corrected code
3. **Add 3 features** that would make this production-ready (from Phase 4 modules)
4. **Write a code review comment** explaining the issues to the junior engineer in a constructive way

### Deliverables

1. `code_review.md` — Structured code review with all issues found
2. `fixed_rag.py` — The refactored, production-ready version
3. `review_comment.md` — A constructive message to the junior engineer explaining the most important fixes

### Review Criteria

Your review should cover:

| Category | Issues Found | Severity |
|----------|-------------|----------|
| Architecture | ? | ? |
| Reliability | ? | ? |
| Performance | ? | ? |
| Quality | ? | ? |
| Security | ? | ? |
| Monitoring | ? | ? |
| Scalability | ? | ? |

---

## 🚦 Gate Check

Completing this module means completing at least 6 of 10 drills:

**Minimum for Phase 4 completion:**
- Drill 1 (Debug Broken RAG) — Must fix all 7 bugs
- Drill 2 (Build RAG from Scratch) — Must produce working RAG system
- Drill 4 (Contradiction Autopsy) — Must build contradiction detector
- Drill 5 (Evaluation Harness) — Must run on 15 queries with scores
- Choose 2 more from Drills 3, 6, 7, 8, 9, 10

**For each completed drill, log:**
- What you built/did
- What went wrong and how you fixed it
- What you learned that wasn't obvious from the theory modules
- One thing you'd do differently next time

---

*"You don't learn RAG by reading about it. You learn RAG by debugging it at 2 AM, wondering why the LLM keeps citing the wrong source. These drills are designed to give you that experience before you're on call."*
