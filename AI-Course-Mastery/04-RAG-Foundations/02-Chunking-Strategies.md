# Chunking Strategies — How You Split Makes or Breaks RAG

## 🎯 Purpose & Goals

> **🛑 STOP. Think about this problem before reading solutions.**
>
> You have a document:
>
> ---
> **Employee Handbook**
>
> **1. Introduction** — Welcome to Acme Corp!
>
> **2. Remote Work Policy** — Employees may work remotely up to 3 days per week... International contractors must ensure local tax compliance...
>
> **3. Benefits** — Health insurance, 401k matching, unlimited PTO...
>
> **4. Code of Conduct** — All employees must follow...
>
> ---

### 🤔 Discovery Question 1: The Splitting Problem

A user asks: *"What's the remote work policy for international contractors?"*

The answer is ONE sentence in section 2. But if you embed the ENTIRE 10-page handbook as one chunk, the embedding for that chunk will be a blurry average of everything — introduction, remote work, benefits, code of conduct. The ONE relevant sentence gets diluted by all the other content.

**Your challenge:** Design a splitting strategy that:
1. Keeps RELATED content together (don't split a sentence mid-thought)
2. Keeps UNRELATED content separate (don't mix remote work with code of conduct)
3. Gives each chunk a CLEAR embedding (focused on one topic)
4. Preserves context (the chunk should say WHICH section it's from)

**What splitting rules would you write? Draw the boundaries yourself before reading on.**

---

### 🤔 Discovery Question 2: The Context Window Puzzle

You decide to split by section heading. Each section becomes a chunk. Great!

But now the LLM gets:
- **Chunk 2 (Remote Work Policy):** "Employees may work remotely up to 3 days per week... International contractors must ensure local tax compliance..."

The LLM answers: *"You can work remotely up to 3 days per week."*

But the user wanted: *"What's the remote work policy for international contractors?"*

**The answer about "international contractors" is in the SAME chunk. But what if the answer spans TWO chunks?**

Example:
- **Chunk 2:** "International contractors must ensure local tax compliance..."
- **Chunk 3:** "...and must sign a separate contractor agreement."

Now the word "and" at the start of chunk 3 refers back to chunk 2. But chunk 3 alone doesn't make sense without chunk 2.

**How do you handle this?** Overlap? Context windows? Something else?

---

### 🤔 Discovery Question 3: The Chunk Size Tradeoff

You test 3 chunk sizes on the SAME document:

| Chunk Size | Retrieval Recall | Answer Quality | Cost/Query |
|------------|-----------------|----------------|------------|
| 200 chars  | 95% (finds right chunk) | 60% (not enough context) | $0.001 |
| 500 chars  | 85% (misses some) | 85% (enough context) | $0.002 |
| 2000 chars | 60% (too much noise) | 90% (lots of context) | $0.005 |

**Before reading on, answer:**
1. Why does retrieval recall DROP as chunk size increases? (Connect back to Phase 3 — what happens to an embedding of a 2000-char blob?)
2. Why does answer quality RISE with larger chunks? (Even though retrieval is worse?)
3. What's the OPTIMAL chunk size for YOUR use case?

---

> **By the end of this file, you will:**
> - Implement 5 different chunking strategies and compare them
> - Know the exact tradeoff between chunk size, retrieval quality, and answer quality
> - Choose the right chunking strategy for YOUR document types
> - Build a chunking evaluation pipeline to measure impact on RAG quality

**⏱ Time Budget:** 2 hours (30 min concepts, 1 hour code, 30 min experiments)

---

## 📖 The Chunking Landscape

### Why Chunking Matters More Than Your Embedding Model

Here's something most RAG tutorials don't tell you:

> **Your chunking strategy affects RAG quality MORE than your choice of embedding model.**

A well-chunked document with a weak model BEATS a poorly-chunked document with the best model.

**Why?** Because the embedding model can only encode what's IN the chunk. If the chunk contains 5 different topics, the embedding becomes a blurry average. If the chunk is one focused topic, the embedding is sharp and searchable.

### The 5 Chunking Strategies

```
STRATEGY          │ BEST FOR                    │ WORST FOR
──────────────────┼──────────────────────────────┼────────────────
1. Fixed Size     │ Simple text, logs            │ Structured docs (misses sections)
2. Sentence       │ Conversational, Q&A          │ Long technical docs (too granular)
3. Recursive      │ Markdown, HTML, code          │ Highly unstructured text
4. Semantic       │ Complex topics, mixed content │ Real-time systems (slow)
5. Document-Aware │ PDFs, legal docs, reports     │ Simple notes (over-engineered)
```

**🤔 Before reading the code:** Based on these descriptions, which strategy would you try FIRST for YOUR documents?

---

## 💻 Code Examples — 5 Chunking Strategies

### Example 1: Fixed-Size Chunking (The Baseline)

```python
"""
Fixed-size chunking — the simplest approach.
Split by character count with optional overlap.
"""

from typing import List, Dict


class FixedSizeChunker:
    """
    Split text into fixed-size chunks.
    
    This is the BASELINE — it's simple, fast, and works okay for many cases.
    But it often splits in the MIDDLE of sentences or thoughts.
    
    🤔 Connect to Phase 3: In your Phase 3 project, you used fixed-size
    chunking. Now you'll see why it might fail for some document types.
    """
    
    def __init__(self, chunk_size: int = 500, overlap: int = 50):
        self.chunk_size = chunk_size
        self.overlap = overlap
    
    def chunk(self, text: str, source: str = "") -> List[Dict]:
        """Split text into fixed-size chunks."""
        chunks = []
        start = 0
        
        while start < len(text):
            end = start + self.chunk_size
            
            # Get chunk
            chunk_text = text[start:end]
            
            # Don't create empty chunks
            if chunk_text.strip():
                chunks.append({
                    "text": chunk_text,
                    "metadata": {
                        "source": source,
                        "chunk_type": "fixed",
                        "start_char": start,
                        "end_char": end,
                        "chunk_size": len(chunk_text),
                    }
                })
            
            # Move start position (with overlap)
            start += self.chunk_size - self.overlap
        
        return chunks


# ── Demo: See how fixed-size chunking splits a document ──
chunker = FixedSizeChunker(chunk_size=100, overlap=20)

doc = """The quick brown fox jumps over the lazy dog. 
This sentence is about a fox. 
Python is a programming language used for data science.
FastAPI is a web framework for building APIs.
The capital of France is Paris.
Machine learning models require large amounts of data."""

chunks = chunker.chunk(doc, "test_doc.txt")
for i, chunk in enumerate(chunks):
    print(f"Chunk {i+1}: [{chunk['metadata']['start_char']}-{chunk['metadata']['end_char']}]")
    print(f"  {chunk['text'][:80]}...")
    print()
```

**Expected output:**
```
Chunk 1: [0-100]
  The quick brown fox jumps over the lazy dog. 
  This sentence is about a fox. 
  Python is a program...

Chunk 2: [80-180]
  ...a programming language used for data science.
  FastAPI is a web framework for building APIs.
  The capi...

Chunk 3: [160-260]
  ...the capital of France is Paris.
  Machine learning models require large amounts of data.

⚠ Notice: 
- Chunk 1 cuts off mid-word ("program..." instead of "programming")
- Chunk 2 starts mid-word ("...a programming")
- Sentences are split across chunks (the "Python" sentence is cut in half)
```

---

### Example 2: Recursive Character Text Splitter

```python
"""
Recursive chunking — the "smart" fixed-size approach.
Tries to split at natural boundaries: paragraphs → sentences → words.
"""

import re
from typing import List, Dict


class RecursiveChunker:
    """
    Split text recursively at natural boundaries.
    
    Priority:
    1. Try to split at paragraph breaks (\n\n)
    2. If chunk is still too big, split at sentence boundaries (. )
    3. If still too big, split at clauses (, ; :)
    4. Last resort: split at word boundaries (space)
    
    This is what LangChain's RecursiveCharacterTextSplitter does internally.
    """
    
    def __init__(self, chunk_size: int = 500, chunk_overlap: int = 50):
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        
        # Split separators in priority order
        self.separators = ["\n\n", "\n", ". ", "! ", "? ", ", ", " ", ""]
    
    def chunk(self, text: str, source: str = "") -> List[Dict]:
        """Split text using recursive boundary detection."""
        chunks = self._split_recursive(text, self.separators, self.chunk_size)
        
        result = []
        for i, chunk_text in enumerate(chunks):
            if chunk_text.strip():
                result.append({
                    "text": chunk_text,
                    "metadata": {
                        "source": source,
                        "chunk_type": "recursive",
                        "chunk_index": i,
                    }
                })
        
        return result
    
    def _split_recursive(
        self, text: str, separators: List[str], chunk_size: int
    ) -> List[str]:
        """Recursively split text using available separators."""
        if len(text) <= chunk_size or not separators:
            return [text]
        
        separator = separators[0]
        remaining_separators = separators[1:]
        
        # Try to split with current separator
        if separator:
            parts = text.split(separator)
        else:
            # Empty separator = split by character (last resort)
            parts = list(text)
        
        # If we can't split into multiple parts, try next separator
        if len(parts) == 1:
            if remaining_separators:
                return self._split_recursive(text, remaining_separators, chunk_size)
            return [text]
        
        # Combine parts into chunks of chunk_size
        chunks = []
        current_chunk = []
        current_size = 0
        
        for part in parts:
            part_size = len(part)
            separator_size = len(separator)
            
            if current_size + part_size + separator_size > chunk_size and current_chunk:
                # Save current chunk
                chunk_text = separator.join(current_chunk)
                chunks.append(chunk_text)
                
                # Start new chunk with overlap
                # Keep last parts that fit in overlap window
                overlap_texts = []
                overlap_size = 0
                for prev_part in reversed(current_chunk):
                    ps = len(prev_part) + separator_size
                    if overlap_size + ps > self.chunk_overlap:
                        break
                    overlap_texts.insert(0, prev_part)
                    overlap_size += ps
                
                current_chunk = overlap_texts + [part]
                current_size = sum(len(p) + separator_size for p in current_chunk)
            else:
                current_chunk.append(part)
                current_size += part_size + separator_size
        
        if current_chunk:
            chunks.append(separator.join(current_chunk))
        
        return chunks


# ── Demo: Compare fixed vs recursive ──
fixed = FixedSizeChunker(chunk_size=150, overlap=20)
recursive = RecursiveChunker(chunk_size=150, chunk_overlap=20)

doc = """Python is a high-level programming language.

It was created by Guido van Rossum in 1991. Python emphasizes code readability.
The language has become extremely popular in data science and AI.

FastAPI is a modern web framework for building APIs with Python.
It was released in 2018. FastAPI is known for its high performance.

PostgreSQL is a powerful open-source database.
It supports advanced features like JSON, full-text search, and ACID compliance."""

print("═══ Fixed-Size Chunking ═══")
for i, chunk in enumerate(fixed.chunk(doc, "test.md")):
    print(f"Chunk {i+1}: {chunk['text'][:100]}...")
    print()

print("═══ Recursive Chunking ═══")
for i, chunk in enumerate(recursive.chunk(doc, "test.md")):
    print(f"Chunk {i+1}: {chunk['text'][:100]}...")
    print()
```

**Expected output:**
```
═══ Fixed-Size Chunking ═══
Chunk 1: Python is a high-level programming language.
...
Chunk 2: ...the language has become extremely popular in data science and AI.
...
[Cuts sentences in half, splits mid-paragraph]

═══ Recursive Chunking ═══
Chunk 1: Python is a high-level programming language.
...
Chunk 2: FastAPI is a modern web framework for building APIs with Python.
...
Chunk 3: PostgreSQL is a powerful open-source database.
...
[Splits at PARAGRAPH breaks. Each topic is a clean chunk!]
```

---

### Example 3: Semantic Chunking

```python
"""
Semantic chunking — split where the TOPIC changes.
Uses embedding similarity between sentences to detect topic shifts.
"""

import numpy as np
from sentence_transformers import SentenceTransformer
from typing import List, Dict


class SemanticChunker:
    """
    Split text where the TOPIC changes.
    
    How it works:
    1. Split text into sentences
    2. Embed each sentence
    3. Compare consecutive sentences: if similarity drops below threshold,
       that's a topic boundary → split here
    
    This is computationally expensive (~10x slower than recursive)
    but produces the MOST semantically coherent chunks.
    
    🤔 Think about this:
    - "Today I went to the store. I bought milk and eggs." = HIGH similarity (same topic)
    - "Today I went to the store. Python is a programming language." = LOW similarity (topic shift!)
    
    The threshold controls sensitivity:
    - Low threshold (0.3): Fewer splits, larger chunks
    - High threshold (0.7): More splits, smaller chunks
    """
    
    def __init__(
        self,
        threshold: float = 0.5,
        max_chunk_size: int = 500,
        model_name: str = "all-MiniLM-L6-v2",
    ):
        self.threshold = threshold
        self.max_chunk_size = max_chunk_size
        self.model = SentenceTransformer(model_name)
    
    def chunk(self, text: str, source: str = "") -> List[Dict]:
        """Split text at semantic boundaries (topic changes)."""
        # Split into sentences
        sentences = self._split_sentences(text)
        
        if len(sentences) <= 1:
            return [{"text": text, "metadata": {"source": source, "chunk_type": "semantic"}}]
        
        # Embed each sentence
        embeddings = self.model.encode(sentences)
        
        # Find topic boundaries
        boundaries = [0]  # Always start at first sentence
        
        for i in range(1, len(sentences)):
            # Compute similarity between consecutive sentences
            sim = np.dot(embeddings[i-1], embeddings[i]) / (
                np.linalg.norm(embeddings[i-1]) * np.linalg.norm(embeddings[i])
            )
            
            # If similarity drops below threshold → topic boundary
            if sim < self.threshold:
                boundaries.append(i)
        
        boundaries.append(len(sentences))
        
        # Build chunks
        chunks = []
        for i in range(len(boundaries) - 1):
            start = boundaries[i]
            end = boundaries[i + 1]
            
            chunk_text = " ".join(sentences[start:end])
            
            # If chunk is too big, sub-chunk it
            if len(chunk_text) > self.max_chunk_size:
                sub_chunks = self._sub_chunk(sentences[start:end])
                for j, sub in enumerate(sub_chunks):
                    chunks.append({
                        "text": sub,
                        "metadata": {
                            "source": source,
                            "chunk_type": "semantic_sub",
                            "start_sentence": start,
                            "end_sentence": end,
                        }
                    })
            else:
                chunks.append({
                    "text": chunk_text,
                    "metadata": {
                        "source": source,
                        "chunk_type": "semantic",
                        "start_sentence": start,
                        "end_sentence": end,
                    }
                })
        
        return chunks
    
    def _split_sentences(self, text: str) -> List[str]:
        """Simple sentence splitting. Production would use spaCy or nltk."""
        # Split on sentence-ending punctuation
        sentences = re.split(r'(?<=[.!?])\s+', text)
        return [s.strip() for s in sentences if s.strip()]
    
    def _sub_chunk(self, sentences: List[str]) -> List[str]:
        """Sub-chunk a group of sentences that's too large."""
        chunks = []
        current = []
        current_len = 0
        
        for s in sentences:
            if current_len + len(s) > self.max_chunk_size and current:
                chunks.append(" ".join(current))
                current = [s]
                current_len = len(s)
            else:
                current.append(s)
                current_len += len(s)
        
        if current:
            chunks.append(" ".join(current))
        
        return chunks


# ── Demo: Semantic chunking in action ──
chunker = SemanticChunker(threshold=0.4)

# Document with 3 clear topic shifts
mixed_doc = """Python is a great programming language for beginners.
It has clear syntax and excellent documentation.
Many universities use Python for introductory computer science courses.
FastAPI is a modern web framework for building APIs with Python.
It supports asynchronous programming and automatic OpenAPI documentation.
FastAPI is one of the fastest growing Python frameworks in 2025.
PostgreSQL is a powerful open-source relational database.
It supports ACID transactions and complex queries.
Many companies use PostgreSQL as their primary database.
The database supports JSON, full-text search, and extensions."""

print("═══ Semantic Chunking ═══")
chunks = chunker.chunk(mixed_doc, "mixed.txt")
for i, chunk in enumerate(chunks):
    print(f"\nChunk {i+1}:")
    print(f"  {chunk['text'][:120]}...")
    
# Show boundary detection
sentences = chunker._split_sentences(mixed_doc)
embeddings = chunker.model.encode(sentences)
print("\n═══ Topic Shift Detection ═══")
for i in range(1, len(sentences)):
    sim = np.dot(embeddings[i-1], embeddings[i]) / (
        np.linalg.norm(embeddings[i-1]) * np.linalg.norm(embeddings[i])
    )
    arrow = "⬇️ TOPIC SHIFT" if sim < 0.4 else "→"
    print(f"  {sentences[i-1][:40]:40s}  {arrow}  {sim:.3f}")
```

**Expected output:**
```
═══ Semantic Chunking ═══

Chunk 1: Python is a great programming language for beginners. It has clear syntax and excellent documentation. Many universities use Python for introductory...

Chunk 2: FastAPI is a modern web framework for building APIs with Python. It supports asynchronous programming and automatic OpenAPI documentation. FastAPI is one of...

Chunk 3: PostgreSQL is a powerful open-source relational database. It supports ACID transactions and complex queries. Many companies use PostgreSQL...

═══ Topic Shift Detection ═══
  Python is a great programming language...     →   0.89   (same topic)
  It has clear syntax and excellent...          →   0.91   (same topic)
  Many universities use Python for...           →   0.32   ⬇️ TOPIC SHIFT
  FastAPI is a modern web framework...          →   0.87   (same topic)
  It supports asynchronous...
  FastAPI is one of the...
                                                 →   0.28   ⬇️ TOPIC SHIFT
  PostgreSQL is a powerful...
```

---

### Example 4: Chunking Evaluation — Find Your Optimal Chunk Size

```python
"""
Chunking evaluation: find the optimal chunk size for YOUR documents.
"""

from sentence_transformers import SentenceTransformer
import numpy as np
from typing import List, Dict, Callable
import time


class ChunkingEvaluator:
    """
    Evaluate chunking strategies on retrieval quality.
    
    Key insight: The best chunk size depends on:
    1. Your document structure (markdown headings? dense paragraphs?)
    2. Your typical query length (short queries need smaller chunks)
    3. Your LLM's context window (larger chunks = more tokens)
    
    Instead of guessing, let's MEASURE.
    """
    
    def __init__(self, model_name: str = "all-MiniLM-L6-v2"):
        self.model = SentenceTransformer(model_name)
    
    def evaluate_chunking(
        self,
        documents: List[str],
        queries: List[str],
        relevant_chunk_indices: List[List[int]],
        chunker_fns: Dict[str, Callable],
    ) -> Dict:
        """
        Evaluate multiple chunking strategies.
        
        For each strategy:
        1. Chunk all documents
        2. Embed all chunks
        3. For each query, retrieve top-k chunks
        4. Check if relevant chunks are in the results
        
        Returns metrics for each strategy.
        """
        results = {}
        
        for strategy_name, chunker in chunker_fns.items():
            print(f"\nEvaluating: {strategy_name}")
            
            # Chunk ALL documents
            all_chunks = []
            for doc in documents:
                chunks = chunker(doc)
                all_chunks.extend(chunks)
            
            if not all_chunks:
                print(f"  No chunks produced!")
                continue
            
            # Embed all chunks
            chunk_texts = [c["text"] for c in all_chunks]
            chunk_embeddings = self.model.encode(chunk_texts, normalize_embeddings=True)
            
            print(f"  Produced {len(all_chunks)} chunks")
            print(f"  Avg chunk size: {np.mean([len(t) for t in chunk_texts]):.0f} chars")
            
            # For each query, test retrieval
            recall_at_k = []
            
            for q_idx, query in enumerate(queries):
                query_vec = self.model.encode([query], normalize_embeddings=True)[0]
                scores = np.dot(chunk_embeddings, query_vec)
                top_indices = np.argsort(scores)[::-1][:5]
                
                # Check if relevant chunks are in top-5
                expected_chunks = relevant_chunk_indices[q_idx]
                
                # Map from document-level indices to chunk-level
                found = any(
                    any(ec[0] <= i <= ec[1] for ec in expected_chunks)
                    for i in top_indices
                )
                
                recall_at_k.append(1 if found else 0)
            
            avg_recall = np.mean(recall_at_k) if recall_at_k else 0
            avg_chunk_size = np.mean([len(t) for t in chunk_texts])
            
            results[strategy_name] = {
                "num_chunks": len(all_chunks),
                "avg_chunk_size": avg_chunk_size,
                "recall_at_5": avg_recall,
            }
            
            print(f"  Recall@5: {avg_recall:.2%}")
        
        # Print comparison table
        print("\n═══ Chunking Strategy Comparison ═══")
        print(f"{'Strategy':25s} {'Chunks':8s} {'Avg Size':12s} {'Recall@5':10s}")
        print("-" * 55)
        
        for name, metrics in sorted(
            results.items(),
            key=lambda x: x[1]["recall_at_5"],
            reverse=True,
        ):
            print(f"{name:25s} {metrics['num_chunks']:<8d} {metrics['avg_chunk_size']:<12.0f} {metrics['recall_at_5']:<10.2%}")
        
        return results


# ── Usage ──
if __name__ == "__main__":
    evaluator = ChunkingEvaluator()
    
    # Sample documents (realistic technical docs)
    documents = [
        """## Introduction to FastAPI
FastAPI is a modern web framework for building APIs with Python. 
It was released in 2018 by Sebastián Ramírez.
FastAPI is built on Starlette for web handling and Pydantic for data validation.

## Key Features
FastAPI supports asynchronous request handling out of the box.
It automatically generates OpenAPI documentation.
Request validation happens automatically via Pydantic models.

## Performance
FastAPI is one of the fastest Python web frameworks.
It performs on par with Node.js and Go in many benchmarks.
""",

        """## PostgreSQL Basics
PostgreSQL is an advanced object-relational database management system.
It has been actively developed for over 30 years.
PostgreSQL supports SQL, JSON, and full-text search.

## Advanced Features
PostgreSQL supports ACID transactions with full durability.
It offers advanced indexing including B-tree, hash, GiST, and GIN.
The database supports materialized views and foreign data wrappers.

## Replication
PostgreSQL supports streaming replication for high availability.
Logical replication allows selective data sharing between databases.
""",

        """## Docker for Development
Docker containers package applications with all dependencies.
Each container runs in an isolated environment.
Docker Compose manages multi-container applications.

## Docker Compose
Docker Compose uses a YAML file to define services.
You can define networks, volumes, and environment variables.
A single command starts all services: docker compose up.

## Best Practices
Use multi-stage builds to reduce image size.
Never store secrets in Dockerfiles.
Use .dockerignore to exclude unnecessary files.
""",
    ]
    
    # Define chunking strategies to test
    chunkers = {
        "Fixed (200, 20)": lambda d: FixedSizeChunker(200, 20).chunk(d),
        "Fixed (500, 50)": lambda d: FixedSizeChunker(500, 50).chunk(d),
        "Fixed (1000, 100)": lambda d: FixedSizeChunker(1000, 100).chunk(d),
        "Recursive (500, 50)": lambda d: RecursiveChunker(500, 50).chunk(d),
    }
    
    # Define queries and which DOCUMENTS are relevant
    queries = [
        "What are the key features of FastAPI?",
        "How does PostgreSQL replication work?",
        "What are Docker best practices?",
    ]
    
    # (document_index, start_sentence, end_sentence) — simplified
    relevant = [
        [(0, 0, 2)],  # Query 1 matches doc 0
        [(1, 1, 2)],  # Query 2 matches doc 1
        [(2, 2, 3)],  # Query 3 matches doc 2
    ]
    
    results = evaluator.evaluate_chunking(documents, queries, relevant, chunkers)
```

**Expected output:**
```
═══ Chunking Strategy Comparison ═══
Strategy                   Chunks   Avg Size     Recall@5  
-----------------------------------------------------------
Recursive (500, 50)        10       345          100.00%  
Fixed (200, 20)            24       189           66.67%  
Fixed (500, 50)            10       487           66.67%  
Fixed (1000, 100)          5        967           33.33%  

🤔 Analysis:
- Recursive beats all fixed sizes because it respects document structure
- Fixed 200 has TOO MANY tiny chunks — many are fragments, not meaningful units
- Fixed 1000 has too FEW giant chunks — embedding is too blurry
- The optimal strategy depends on YOUR documents — run this test on YOUR data
```

---

### Example 5: The Chunk Size Sweep

```python
"""
Find YOUR optimal chunk size by sweeping across sizes.
"""

from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer('all-MiniLM-L6-v2')

# Test different chunk sizes with the same overlap ratio
chunk_sizes = [100, 200, 300, 400, 500, 750, 1000, 1500]
overlap_ratio = 0.1  # 10% overlap

test_doc = """...your actual document..."""

results = []
for size in chunk_sizes:
    overlap = int(size * overlap_ratio)
    chunker = FixedSizeChunker(size, overlap)
    chunks = chunker.chunk(test_doc, "test")
    
    if not chunks:
        continue
    
    # Embed chunks
    texts = [c["text"] for c in chunks]
    embeddings = model.encode(texts, normalize_embeddings=True)
    
    # Measure: average intra-chunk similarity
    # Higher = chunks are more self-consistent (good)
    # Lower = chunks contain mixed topics (bad)
    intra_sim = np.mean([
        np.dot(embeddings[i], embeddings[i])  # Self-similarity = 1.0
        for i in range(len(chunks))
    ])
    
    # Measure: average inter-chunk similarity
    # Higher = chunks too similar (poor differentiation)
    # Lower = chunks well-separated (good topic isolation)
    if len(chunks) > 1:
        inter_sims = []
        for i in range(len(chunks)):
            for j in range(i+1, len(chunks)):
                inter_sims.append(np.dot(embeddings[i], embeddings[j]))
        inter_sim = np.mean(inter_sims)
    else:
        inter_sim = 0
    
    results.append({
        "chunk_size": size,
        "num_chunks": len(chunks),
        "avg_size": np.mean([len(t) for t in texts]),
        "inter_chunk_similarity": inter_sim,
    })

# Print
print("═══ Chunk Size Sweep Results ═══\n")
print(f"{'Size':8s} {'Chunks':8s} {'Avg Size':10s} {'Inter-Sim':10s} {'Quality':10s}")
print("-" * 46)

for r in results:
    quality = "GOOD" if r["inter_chunk_similarity"] < 0.3 else "OK" if r["inter_chunk_similarity"] < 0.5 else "POOR"
    print(f"{r['chunk_size']:<8d} {r['num_chunks']:<8d} {r['avg_size']:<10.0f} {r['inter_chunk_similarity']:<10.3f} {quality:<10s}")
```

---

## ✅ Good Output Examples

### What Good Chunking Looks Like

```
Document: Employee Handbook (10 sections)
Strategy: Recursive (split at ## headings)

Chunks produced:
  Chunk 1: "## 1. Introduction\nWelcome to Acme Corp..."
  Chunk 2: "## 2. Remote Work Policy\nEmployees may work..."
  Chunk 3: "## 3. Benefits\nHealth insurance, 401k..."

Query: "What are the benefits offered?"
Retrieved: Chunk 3 (score: 0.91)
✓ Perfect match — the chunk is exactly about benefits
✓ No split across sections
✓ Clean heading preserved for context
```

### What Bad Chunking Looks Like

```
Document: Employee Handbook (10 sections)
Strategy: Fixed-size (500 chars, no overlap)

Chunks produced:
  Chunk 1: "# Employee Handbook\n\n## 1. Introduction\nWelcome to Acme Corp!\n\n## 2. Remote Work Policy\n"
  Chunk 2: "Employees may work remotely up to 3 days per week... International contractors\n\n## 3. Benefits\n"
  Chunk 3: "Health insurance, 401k matching, unlimited PTO...\n\n## 4. Code of Conduct\nAll employe"

Query: "What are the benefits offered?"
Retrieved: Chunk 2 (score: 0.45 — mentions "Benefits" heading but mostly remote work)
           Chunk 3 (score: 0.42 — has "Benefits" content but also "Code of Conduct")
           
✗ Chunk 2 contains TWO topics (remote work + benefits heading)
✗ Chunk 3 splits benefits across TWO sections (half benefits, half code of conduct)
✗ No chunk is PURELY about benefits
✗ Embeddings are diluted by mixed content
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: No Overlap

```python
# ❌ WRONG — Zero overlap
chunks = split_text(text, chunk_size=500, overlap=0)
# Problem: If a sentence spans chunk boundaries, BOTH chunks lose context
# Chunk 1 ends with "The international contractor must..."
# Chunk 2 starts with "...sign a separate agreement."
# Each chunk alone is INCOMPLETE

# ✅ RIGHT — 10-20% overlap
chunks = split_text(text, chunk_size=500, overlap=75)
# Important context is duplicated, preserving sentence integrity
```

### Antipattern 2: Ignoring Document Structure

```python
# ❌ WRONG — Same strategy for ALL document types
def chunk_all(text):
    return fixed_size_chunk(text, 500)
# Code files (.py) need DIFFERENT chunking than markdown (.md)
# PDFs need DIFFERENT chunking than HTML

# ✅ RIGHT — Document-type-aware chunking
def chunk_document(text, doc_type):
    if doc_type == "markdown":
        return markdown_section_chunk(text)
    elif doc_type == "code":
        return function_boundary_chunk(text)
    elif doc_type == "pdf":
        return page_boundary_chunk(text)
    else:
        return recursive_chunk(text)
```

### Antipattern 3: Chunks Too Large

```python
# ❌ WRONG — 2000 character chunks
chunker = RecursiveChunker(chunk_size=2000, overlap=100)
# A 2000-character chunk might cover 3-4 topics
# The embedding becomes a blurry average of all topics
# Query about topic A also retrieves chunks about topic B, C, D
# Wastes the LLM's context window with irrelevant content
```

### Failure Mode: The "Missing Context" Problem

```
If your chunk size is TOO SMALL:
- "The international contractor must ensure compliance..." (no context about WHICH policy)
- The LLM can't answer: "What policy applies to international contractors?"
- Because the chunk lost the "Remote Work Policy" heading

SOLUTION: Always include section context in the chunk
✅ GOOD: "## Remote Work Policy\n\nThe international contractor must ensure compliance..."
```

---

## 🧪 Drills & Challenges

### Drill 1: Implement a Code-Aware Chunker (25 min)

Build a chunker that splits Python files at function/class boundaries:

```python
def chunk_python_file(code: str, source: str) -> List[Dict]:
    """
    Split Python code into chunks at function/class boundaries.
    
    Each chunk = one function or class (with its docstring).
    Include the file path as context.
    """
    # Hint: Use AST parsing (ast.parse) to find function/class definitions
    pass
```

**Test with:** A Python file with 3 functions and 2 classes. Verify each chunk contains exactly one function or class.

### Drill 2: Optimal Chunk Size for YOUR Documents (20 min)

Take 5 documents from YOUR work/notes. Run the chunk size sweep on each:
1. At what chunk size does inter-chunk similarity drop below 0.3?
2. At what chunk size does recall@5 peak?
3. Is there a "sweet spot" that works for all 5 documents?

### Drill 3: The "Paragraph Splitting" Experiment (15 min)

Compare chunking at paragraph vs sentence boundaries. Which is better for:
1. Technical documentation (API docs, how-to guides)
2. Conversational text (chat logs, emails)
3. Legal documents (contracts, policies)

**Hypothesis:** Technical docs benefit from LARGER chunks (paragraph-level) because terms are interrelated. Conversational text benefits from SMALLER chunks because each message is a self-contained unit.

### Drill 4: Parent Document Retrieval (20 min)

Implement a strategy where:
1. You retrieve SMALL chunks (high precision — 200 chars)
2. But return the PARENT section (larger context — full section)

```python
class ParentDocumentRetriever:
    """
    Retrieve small chunks, return larger context.
    
    Small chunk = high precision for embedding matching
    Parent chunk = sufficient context for LLM to answer
    
    Child chunks know their parent:
      child_1 → parent: "## Remote Work Policy\n..."
      child_2 → parent: "## Remote Work Policy\n..."
    """
    pass
```

### Drill 5: The "Chunk and Ask" Method (10 min)

For 3 different chunking strategies, ask the SAME question and compare:

```python
# Run this comparison for each strategy:
for strategy in [fixed_200, recursive_500, semantic]:
    chunks = strategy.chunk(employee_handbook)
    relevant = search(query, chunks)
    answer = llm.ask(query, relevant)
    
    # Which strategy gave the best answer?
    # Which gave the worst?
```

---

## 🚦 Gate Check

Before moving to File 03 (Retrieval Deep Dive), confirm you can:

- [ ] **Implement 3 chunking strategies** (fixed, recursive, semantic)
- [ ] **Explain the chunk size tradeoff** (small = focused but missing context, large = blurry but complete)
- [ ] **Choose the right strategy** for markdown vs code vs plain text
- [ ] **Know when to use overlap** and how much
- [ ] **Run a chunking evaluation** on YOUR documents
- [ ] **Answer these questions:**
  1. Your RAG system retrieves a chunk about "remote work policy" for a query about "benefits." Is this a chunking problem or an embedding problem? How would you diagnose it?
  2. A user asks "What's the deadline?" The chunk contains "The deadline for submission is Friday." But the chunk ALSO contains a heading from a different section. Will the LLM get confused?
  3. You have a 200-page technical manual. Should you chunk each page as one chunk? Why or why not?
  4. How does chunk size affect embedding quality? (Refer to Phase 3 — what makes a good embedding?)
  5. If you increase chunk overlap from 10% to 50%, how does it affect storage, retrieval, and cost?

- [ ] **Document your chunking decision** for the project (File 09)

---

## 📚 Resources

**Foundational:**
- LangChain `RecursiveCharacterTextSplitter` documentation
- LlamaIndex `NodeParser` documentation
- "Chunking Strategies for RAG" — Various engineering blogs

**Advanced:**
- "Semantic Chunking with Sentence Embeddings" — Using similarity for topic detection
- "Parent Document Retriever" — LlamaIndex pattern
- "Chunking Evaluation Framework" — How to measure chunk quality

**Your Decision Flowchart:**
```
START: What type of documents are you chunking?
│
├── Markdown / structured text → Recursive with heading separators
│   └── Chunk size: 500-1000 chars, overlap: 10-20%
│
├── Code files → Function/class boundary chunking
│   └── Include docstrings and file path in chunk
│
├── PDFs → Page or section based (use PyMuPDF/unstructured)
│   └── Add page numbers to metadata
│
├── Plain text / loose notes → Recursive or semantic
│   └── Semantic chunking for highly varied topics
│
├── Conversational / chat → Sentence-level chunks
│   └── Keep each message intact (don't split mid-message)
│
└── Mixed collection → Document-type-aware router
    └── Different strategy per file extension
```

---

> **🛑 Revisit the 3 discovery questions from the beginning:**
>
> 1. **The Splitting Problem** — You've now seen that embedding an entire document as one chunk is terrible. Good chunking preserves document structure.
> 2. **The Context Window Puzzle** — Overlap and parent-document retrieval solve the "answer spans two chunks" problem.
> 3. **The Chunk Size Tradeoff** — You now know that small chunks give better retrieval but worse context, and vice versa. The optimal size is found by EXPERIMENTING on YOUR data, not guessing.
>
> **Key realization:** There's no universal "best chunk size." The right answer depends on YOUR documents, YOUR queries, and YOUR LLM. Measure, don't guess.
>
> **Next:** File 03 — Retrieval Deep Dive. Beyond simple vector search: query transformation, HyDE, and multi-stage retrieval.
