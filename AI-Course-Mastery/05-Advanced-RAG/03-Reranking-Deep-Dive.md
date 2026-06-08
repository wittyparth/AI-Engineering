# 03 — Reranking Deep Dive: Precision Through Second-Pass Ranking

## 🎯 Purpose & Goals

Your first-pass retrieval (dense vector search, BM25, or hybrid) returns a SET of relevant chunks. But it returns them ranked by APPROXIMATE similarity — embeddings that capture "vibe" more than exact relevance. The top-3 results might be good, but positions 4-8 might be valuable chunks that don't rank highly because their embedding is slightly off.

**Reranking is the fix.** A reranker is a second-pass model that takes the query and a chunk TOGETHER (not separately like embeddings) and computes a precise relevance score. The difference is dramatic: a good reranker can improve retrieval precision by 15-30%.

By the end of this module, you'll understand WHY rerankers work so much better than embedding similarity, WHEN to use them, and HOW to build a production reranking pipeline that balances precision, latency, and cost.

**By the end of this module, you will:**

1. Understand the architectural difference between **bi-encoders** (dense retrieval) and **cross-encoders** (rerankers) — and why cross-encoders are strictly more powerful
2. Implement a complete reranking pipeline using open-source cross-encoder models (sentence-transformers)
3. Integrate with the **Cohere Rerank API** for managed reranking
4. Build a **two-stage retrieval architecture**: retrieve N with bi-encoder, rerank top-K with cross-encoder
5. Implement **listwise reranking** using LLMs as rankers
6. Know the latency-quality tradeoff curve and how to find the optimal operating point
7. Understand when NOT to rerank — and what to do instead

---

### ⏹ STOP. Answer these before proceeding.

**Question 1 — Why Cross-Encoders Win**

*A user asks "What's the return policy for opened electronics?"*

*Your dense retrieval returns these chunks with similarity scores:*
1. *"Return policy: 30-day return window for all items" (0.89)*
2. *"Electronics returns require original packaging" (0.87)*
3. *"Opened items may be subject to restocking fee" (0.72)*
4. *"Defective items can be returned within 1 year" (0.58)*

*Chunk 4 is about DEFECTIVE items, not opened electronics. The embedding similarity is low because the words are different ("defective" ≠ "opened"). But it's actually RELEVANT — the user's opened electronics might be defective. The embedding model can't see this relationship because it computes query-chunk similarity independently.*

*A cross-encoder processes query and chunk TOGETHER. It sees:*
- *Query words: "return", "opened", "electronics"*
- *Chunk words: "defective", "return", "1 year"*
- *The cross-attention can connect "opened electronics → could be defective → falls under defective policy"*

*Why can't a bi-encoder (dense retrieval) make this connection? What fundamental architectural limitation prevents it? What would need to change in the embedding model to enable this?*

**Question 2 — The N vs. K Tradeoff**

*Your reranking pipeline:*
- *Retrieve N=100 chunks (fast embedding search)*
- *Rerank top-10 with cross-encoder (slow but precise)*
- *Return top-5 to the LLM*

*If you increase N to 200, you capture more potentially relevant chunks, but the top-10 might still be the same because the bottom 100 are increasingly irrelevant. If you decrease N to 20, you're faster but might miss chunks that would have been ranked #11-20 by the retriever but would be #1-3 after reranking.*

*What determines the optimal N? Is it document-dependent, query-dependent, or model-dependent? How would you find the optimal N for YOUR system experimentally? What's the cost of setting N too high (in latency) vs. too low (in missed results)?*

**Question 3 — The LLM as Reranker**

*Instead of a cross-encoder model, you use an LLM to rerank: "Given the query, rank these 10 chunks by relevance." The LLM returns an ordered list.*

*This sounds powerful — the LLM understands semantics deeply. But:*
- *GPT-4o-mini reranking: ~$0.002 per 10 chunks, ~500ms latency*
- *Cross-encoder reranking: ~$0.00001 per 10 chunks, ~50ms latency*
- *The cross-encoder is 200x cheaper and 10x faster*

*When would the LLM's deeper understanding be worth 200x the cost? What kinds of queries benefit from LLM reranking that cross-encoders can't handle? Is there a HYBRID approach where some queries get cross-encoder and some get LLM reranking?*

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 15 min |
| Bi-Encoder vs. Cross-Encoder | 25 min |
| Cross-Encoder Architecture Deep Dive | 30 min |
| Two-Stage Retrieval Architecture | 25 min |
| Open-Source Cross-Encoders | 20 min |
| Cohere Rerank API | 15 min |
| ColBERT Late Interaction | 20 min |
| LLM Listwise Reranking | 25 min |
| Code: Full Reranking Pipeline | 60 min |
| Code: Latency Benchmarks | 30 min |
| Code: Adaptive Reranking | 30 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~6 hours** |

---

## 📖 Concept Deep-Dive

### 1. Bi-Encoder vs. Cross-Encoder

The fundamental architectural difference:

```
BI-ENCODER (Dense Retrieval):
           
Query: "What's the refund policy?"
    ↓
[Transformer] ──→ Query embedding [0.2, 0.5, ...]
                     │
                     ├── Similarity (cosine) ──→ Score
                     │
Chunk: "Returns within 30 days..."
    ↓
[Transformer] ──→ Chunk embedding [0.7, 0.3, ...]

→ Query and chunk are encoded SEPARATELY
→ No interaction between query and chunk during encoding
→ Fast: all chunk embeddings are pre-computed
→ Approximate: each piece is summarized independently


CROSS-ENCODER (Reranker):

Query: "What's the refund policy?"  │  Chunk: "Returns within 30 days..."
                  │                          │
                  └──────────┬───────────────┘
                             ↓
                    [Transformer with Cross-Attention]
                             ↓
                    [CLS] token representation
                             ↓
                    Relevance Score: 0.94

→ Query and chunk are processed TOGETHER
→ Full cross-attention between query and chunk tokens
→ Slow: must run for EACH (query, chunk) pair
→ Precise: the model can directly compare query and chunk content
```

**Why cross-encoders are strictly more powerful:**

The cross-encoder's attention mechanism lets every query token attend to every chunk token. This enables the model to:

1. **Match synonyms**: Query says "buy" but chunk says "purchase" → cross-attention connects them
2. **Resolve references**: Query asks about "it" → chunk's preceding sentence says "the refund"
3. **Detect negation**: Query asks "what is NOT covered" → chunk says "electronics are covered" but "damage is NOT covered" — the model sees both
4. **Infer relationships**: Query asks about "student discount" → chunk says "educational pricing available" → model sees the connection

A bi-encoder CANNOT do any of this because each piece is summarized independently — the interaction is reduced to a single cosine similarity score.

---

### 2. Cross-Encoder Architecture Deep Dive

```
Input: [CLS] query tokens [SEP] chunk tokens [SEP]
         ↓
    [Embedding Layer] ──→ Token embeddings + position embeddings
         ↓
    [Transformer Layers] ──→ N layers of self-attention + cross-attention
         │                    (query tokens attend to chunk tokens and vice versa)
         ↓
    [CLS] Token Representation
         ↓
    [Classification Head] ──→ Single logit → sigmoid → relevance score (0-1)
```

```python
"""
cross_encoder_arch.py — Understanding cross-encoder internals.
"""

import torch
import torch.nn as nn
from transformers import AutoModel, AutoTokenizer


class CrossEncoder(nn.Module):
    """
    Cross-encoder for query-document relevance scoring.

    Architectural overview:
    - Takes (query, document) as a single concatenated input
    - Processes through full transformer with cross-attention
    - Uses [CLS] token representation for classification
    - Outputs a single relevance score
    """

    def __init__(self, model_name: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"):
        super().__init__()
        self.model = AutoModel.from_pretrained(model_name)
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)

        # Classification head on top of [CLS] token
        hidden_size = self.model.config.hidden_size
        self.classifier = nn.Linear(hidden_size, 1)

    def forward(self, query: str, document: str) -> float:
        """
        Compute relevance score between query and document.

        The key difference from bi-encoders:
        - Single forward pass with (query + document) as input
        - Full attention between all tokens
        - One score output
        """
        # Tokenize as a single sequence
        inputs = self.tokenizer(
            query,
            document,
            return_tensors="pt",
            truncation=True,
            max_length=512,
            padding=True
        )

        # Forward pass — single transformer processes BOTH
        outputs = self.model(**inputs)

        # [CLS] token captures query-document interaction
        cls_token = outputs.last_hidden_state[:, 0, :]

        # Score
        score = torch.sigmoid(self.classifier(cls_token))
        return score.item()

    def score_batch(self, query: str, documents: List[str]) -> List[float]:
        """
        Score a query against multiple documents.

        This requires SEPARATE forward passes for each document.
        Unlike bi-encoders where embeddings are pre-computed,
        cross-encoders must process each pair individually.
        """
        scores = []
        for doc in documents:
            score = self.forward(query, doc)
            scores.append(score)
        return scores
```

**Why cross-encoders are SLOW:**

For N chunks and 1 query:
- **Bi-encoder**: 1 query encode + N saved chunk embeddings → 1 forward pass (query) + N cosine similarities
- **Cross-encoder**: N forward passes (query + chunk_i for each i)

The cross-encoder's advantage is precision. The cost is that you can't pre-compute anything — every reranking call requires a full forward pass per candidate.

---

### 3. Two-Stage Retrieval Architecture

The standard architecture for production RAG with reranking:

```
QUERY
  │
  ▼
┌─────────────────────────────────────────────┐
│ STAGE 1: FIRST-PASS RETRIEVAL              │
│                                             │
│  Bi-encoder (dense) + BM25 (sparse)         │
│  Hybrid fusion                              │
│                                             │
│  Returns: N candidates (N = 20-200)         │
│  Latency: ~50-200ms                         │
│  Cost: ~$0.00001 per query                  │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│ STAGE 2: RERANKING                          │
│                                             │
│  Cross-encoder scores each (query, chunk)    │
│  Sorts by relevance score                    │
│                                             │
│  Returns: K candidates (K = 3-10)           │
│  Latency: ~100-1000ms (depends on N and model)│
│  Cost: ~$0.0001-0.001 per query             │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│ STAGE 3: LLM GENERATION                     │
│                                             │
│  Top-K chunks → context → LLM               │
│                                             │
│  Latency: ~500-2000ms                       │
│  Cost: ~$0.001-0.01 per query               │
└─────────────────────────────────────────────┘
```

```python
"""
two_stage_retrieval.py — Complete two-stage retrieval with reranking.
"""

from typing import List, Dict, Optional, Callable
from dataclasses import dataclass
import time
import asyncio


@dataclass
class RetrievalResult:
    query: str
    first_pass_candidates: int
    reranked_candidates: int
    chunks: List[Dict]
    first_pass_latency_ms: float
    rerank_latency_ms: float
    scores_before: List[float]
    scores_after: List[float]


class TwoStageRetriever:
    """
    Two-stage retrieval: first-pass (fast, high recall) → reranker (precise).

    Configurable:
    - N: number of candidates from first-pass
    - K: number of results after reranking
    - Reranker type: cross-encoder, API, or LLM
    """

    def __init__(
        self,
        first_pass_retriever,  # Dense, sparse, or hybrid retriever
        reranker,              # Cross-encoder or API-based reranker
        n_candidates: int = 50,
        k_final: int = 5,
        min_score_threshold: float = 0.0
    ):
        self.first_pass = first_pass_retriever
        self.reranker = reranker
        self.n_candidates = n_candidates
        self.k_final = k_final
        self.min_score_threshold = min_score_threshold

    async def retrieve(self, query: str) -> RetrievalResult:
        """
        Two-stage retrieval.

        Stage 1: Get N candidates (fast, high recall)
        Stage 2: Rerank top-N to find top-K (precise)
        """
        # Stage 1: First-pass retrieval
        t1 = time.perf_counter()
        candidates = await self.first_pass.retrieve(query, top_k=self.n_candidates)
        stage1_latency = (time.perf_counter() - t1) * 1000

        if not candidates:
            return RetrievalResult(
                query=query,
                first_pass_candidates=0,
                reranked_candidates=0,
                chunks=[],
                first_pass_latency_ms=stage1_latency,
                rerank_latency_ms=0,
                scores_before=[],
                scores_after=[],
            )

        # Store first-pass scores
        scores_before = [c.get("score", 0) for c in candidates]

        # Stage 2: Rerank
        t2 = time.perf_counter()
        reranked = await self.reranker.rerank(
            query=query,
            candidates=candidates,
            top_k=self.k_final
        )
        stage2_latency = (time.perf_counter() - t2) * 1000

        # Apply score threshold
        reranked = [
            c for c in reranked
            if c.get("rerank_score", 1.0) >= self.min_score_threshold
        ]

        scores_after = [c.get("rerank_score", 0) for c in reranked]

        return RetrievalResult(
            query=query,
            first_pass_candidates=len(candidates),
            reranked_candidates=len(reranked),
            chunks=reranked,
            first_pass_latency_ms=stage1_latency,
            rerank_latency_ms=stage2_latency,
            scores_before=scores_before,
            scores_after=scores_after,
        )
```

**The critical design parameter: N (candidates from first-pass)**

```
N=10 → rerank 10 chunks: fast (100ms), but might miss good chunks ranked #11-20
N=50 → rerank 50 chunks: moderate (500ms), good coverage
N=100 → rerank 100 chunks: slow (1000ms), diminishing returns
N=200 → rerank 200 chunks: very slow (2000ms), rare improvements

Sweet spot: N=20-50 for most production systems
```

**How to find YOUR optimal N:**

```python
async def find_optimal_n(
    test_queries: List[str],
    retriever_builder,
    n_values: List[int] = [10, 20, 50, 100, 200]
) -> Dict:
    """
    For each N, measure:
    - Recall@K (does the final top-K contain the answer?)
    - Total latency (first-pass + rerank)

    Find the N that maximizes recall while staying under latency SLO.
    """
    results = {}

    for n in n_values:
        retriever = retriever_builder(n_candidates=n)
        latencies = []
        correct = 0

        for query, gold_chunk_id in test_queries:
            t0 = time.perf_counter()
            result = await retriever.retrieve(query)
            latency = (time.perf_counter() - t0) * 1000
            latencies.append(latency)

            retrieved_ids = [c["id"] for c in result.chunks]
            if gold_chunk_id in retrieved_ids:
                correct += 1

        results[n] = {
            "recall": correct / len(test_queries),
            "p95_latency_ms": sorted(latencies)[int(len(latencies) * 0.95)],
        }

    return results
```

---

### 4. Open-Source Cross-Encoders

The `sentence-transformers` library provides pre-trained cross-encoder models. The most popular:

| Model | Size | Speed | Quality | Best For |
|-------|------|-------|---------|----------|
| `ms-marco-MiniLM-L-2-v2` | 22MB | Fastest | Adequate | High-throughput, low-latency |
| `ms-marco-MiniLM-L-4-v2` | 44MB | Fast | Good | Balanced production use |
| `ms-marco-MiniLM-L-6-v2` | 80MB | Moderate | Better | Quality-sensitive production |
| `ms-marco-MiniLM-L-12-v2` | 180MB | Slow | Best | Highest quality (batch) |
| `ce-ms-marco-electra-base` | 200MB | Slow | Best (different arch) | Research/benchmarking |

```python
"""
cross_encoder_reranker.py — Cross-encoder reranking using sentence-transformers.
"""

from sentence_transformers import CrossEncoder
from typing import List, Dict, Optional
import torch


class CrossEncoderReranker:
    """
    Rerank retrieved chunks using a cross-encoder model.

    Usage:
        reranker = CrossEncoderReranker(model_name="cross-encoder/ms-marco-MiniLM-L-6-v2")
        reranked = await reranker.rerank(query, candidates, top_k=5)
    """

    def __init__(
        self,
        model_name: str = "cross-encoder/ms-marco-MiniLM-L-6-v2",
        device: Optional[str] = None,
        batch_size: int = 32,
        max_length: int = 512
    ):
        if device is None:
            device = "cuda" if torch.cuda.is_available() else "cpu"

        self.model = CrossEncoder(model_name, max_length=max_length, device=device)
        self.batch_size = batch_size
        self.max_length = max_length

    async def rerank(
        self,
        query: str,
        candidates: List[Dict],
        top_k: int = 5,
        return_scores: bool = True
    ) -> List[Dict]:
        """
        Rerank candidates using cross-encoder.

        Args:
            query: The search query
            candidates: List of candidate chunks with 'text' field
            top_k: Number of results to return

        Returns:
            Candidates sorted by cross-encoder relevance score
        """
        if not candidates:
            return []

        # Prepare (query, document) pairs
        pairs = [
            (query, c.get("text", c.get("content", "")))
            for c in candidates
        ]

        # Score all pairs (batched for efficiency)
        scores = self.model.predict(pairs, batch_size=self.batch_size)

        # Handle different return types
        if isinstance(scores, list):
            scores = scores
        elif hasattr(scores, 'tolist'):
            scores = scores.tolist()
        else:
            scores = list(scores)

        # Attach scores and sort
        for i, candidate in enumerate(candidates):
            candidate["rerank_score"] = float(scores[i])

        reranked = sorted(
            candidates,
            key=lambda x: x["rerank_score"],
            reverse=True
        )

        return reranked[:top_k]

    async def score(self, query: str, document: str) -> float:
        """Score a single query-document pair."""
        return self.model.predict([(query, document)])[0]


# ============================================================
# Pipeline Integration
# ============================================================

class RerankingPipeline:
    """
    Full reranking pipeline with first-pass + cross-encoder.

    Usage:
        pipeline = RerankingPipeline(first_pass_retriever, reranker)
        result = await pipeline.search("What's the refund policy?")
    """

    def __init__(self, first_pass, reranker, n_candidates=50, k_final=5):
        self.first_pass = first_pass
        self.reranker = reranker
        self.n_candidates = n_candidates
        self.k_final = k_final

    async def search(self, query: str) -> Dict:
        """Full search pipeline with reranking."""
        # Stage 1: First-pass retrieval
        candidates = await self.first_pass.retrieve(query, top_k=self.n_candidates)

        # Stage 2: Rerank
        if candidates:
            reranked = await self.reranker.rerank(
                query, candidates, top_k=self.k_final
            )
        else:
            reranked = []

        return {
            "query": query,
            "results": reranked,
            "total_candidates": len(candidates),
            "metadata": {
                "first_pass": "hybrid",
                "reranker": self.reranker.model.model_name
                if hasattr(self.reranker, 'model') else "api",
            }
        }
```

---

### 5. Cohere Rerank API

For teams that don't want to self-host cross-encoder models, Cohere provides a managed reranking API.

```python
"""
cohere_reranker.py — Managed reranking via Cohere API.
"""

import cohere
from typing import List, Dict


class CohereReranker:
    """
    Rerank using Cohere's managed Rerank API.

    Advantages:
    - No model hosting
    - State-of-the-art quality
    - Handles long documents (up to 5120 tokens)
    - Returns relevance scores

    Disadvantages:
    - Cost per query
    - Latency depends on API
    - Rate limits
    """

    def __init__(self, api_key: str, model: str = "rerank-v3.5"):
        self.client = cohere.Client(api_key)
        self.model = model

    async def rerank(
        self,
        query: str,
        documents: List[Dict],
        top_k: int = 5,
        max_chunks_per_doc: int = 5000
    ) -> List[Dict]:
        """
        Rerank documents using Cohere.

        Args:
            query: Search query
            documents: List of documents with 'text' field
            top_k: Number of results to return
            max_chunks_per_doc: Max characters per document

        Returns:
            Documents sorted by relevance with scores
        """
        texts = [
            d.get("text", "")[:max_chunks_per_doc]
            for d in documents
        ]

        response = self.client.rerank(
            model=self.model,
            query=query,
            documents=texts,
            top_n=top_k,
            return_documents=True
        )

        results = []
        for result in response.results:
            original_idx = result.index
            doc = documents[original_idx].copy()
            doc["rerank_score"] = result.relevance_score
            doc["rerank_index"] = result.index
            results.append(doc)

        return results


# Cost analysis
"""
Cohere Rerank v3.5 pricing (as of 2024):
- $1.00 per 1000 reranking units
- 1 unit = 1 query × 1 document (up to 5120 chars each)
- For 50 candidates reranked per query:
  - 50 units per query
  - 50,000 units = $50 per 1000 queries
  - $0.05 per query

Compare to self-hosted cross-encoder:
- GPU cost: ~$30/month (T4 on GCP)
- 100,000+ queries per month
- $0.0003 per query

When to use each:
- Cohere: < 10K queries/month, no GPU budget, need top quality
- Self-hosted: > 10K queries/month, have GPU infra, latency-sensitive
"""
```

---

### 6. ColBERT: Late Interaction

ColBERT (Contextualized Late Interaction over BERT) is a HYBRID between bi-encoder speed and cross-encoder quality.

**How it works:**
1. Encode query and document SEPARATELY (like bi-encoder)
2. But keep ALL token embeddings (don't pool to a single vector)
3. Compute similarity as: max over query tokens of (max over doc tokens of dot product)

```python
"""
colbert_reranker.py — ColBERT-style late interaction reranking.

This is a simplified implementation for understanding.
For production, use the official ColBERT library.
"""

import torch
import torch.nn.functional as F


class ColBERTReranker:
    """
    Late interaction reranking.

    The key insight:
    - Encode query and document separately (fast, parallelizable)
    - But keep ALL token embeddings (not pooled)
    - Compute MaxSim: max similarity for each query token across all doc tokens
    - Sum the max similarities

    This captures fine-grained token-level interactions
    without requiring a full cross-encoder forward pass.
    """

    def __init__(self, model, tokenizer, device="cpu"):
        self.model = model
        self.tokenizer = tokenizer
        self.device = device

    def encode(self, texts: List[str]) -> torch.Tensor:
        """Encode texts keeping all token embeddings."""
        inputs = self.tokenizer(
            texts,
            return_tensors="pt",
            padding=True,
            truncation=True,
            max_length=512
        ).to(self.device)

        with torch.no_grad():
            outputs = self.model(**inputs)

        # Use all token embeddings (not just [CLS])
        # Shape: (batch_size, seq_len, hidden_dim)
        return outputs.last_hidden_state

    def maxsim(self, query_emb: torch.Tensor, doc_emb: torch.Tensor) -> float:
        """
        MaxSim: For each query token, find max similarity with doc tokens.

        query_emb: (1, q_len, dim)
        doc_emb:   (1, d_len, dim)

        Returns: scalar score
        """
        # Normalize
        query_emb = F.normalize(query_emb, p=2, dim=2)
        doc_emb = F.normalize(doc_emb, p=2, dim=2)

        # Dot product: (1, q_len, d_len)
        sim = torch.matmul(query_emb, doc_emb.transpose(1, 2))

        # Max over document tokens for each query token
        max_sim = sim.max(dim=2).values  # (1, q_len)

        # Sum over query tokens
        score = max_sim.sum(dim=1)  # (1,)

        return score.item()

    def rerank(self, query: str, documents: List[str]) -> List[float]:
        """
        Rerank documents using late interaction.

        This is faster than cross-encoders because:
        - Query encoded once (not once per document)
        - Documents can be pre-encoded (unlike cross-encoders)
        - MaxSim is cheap (O(q_len * d_len) dot product per pair)
        """
        query_emb = self.encode([query])  # (1, q_len, dim)

        scores = []
        for doc in documents:
            doc_emb = self.encode([doc])  # (1, d_len, dim)
            score = self.maxsim(query_emb, doc_emb)
            scores.append(score)

        return scores
```

**ColBERT vs. Cross-Encoder vs. Bi-Encoder:**

| Aspect | Bi-Encoder | ColBERT | Cross-Encoder |
|--------|-----------|---------|---------------|
| Pre-compute embeddings | ✅ Yes | ✅ Yes (docs) | ❌ No |
| Token-level interaction | ❌ Single vector | ✅ Per-token MaxSim | ✅ Full attention |
| Rerank speed (50 docs) | N/A | ~100ms | ~500ms |
| Quality | Baseline | +5-10% over bi-encoder | +15-30% over bi-encoder |
| Index size | Small (1 vec/chunk) | Large (all tokens) | N/A |

---

### 7. LLM Listwise Reranking

Use an LLM to compare and rank chunks. This is the MOST powerful reranking approach — and the MOST expensive.

```python
"""
llm_reranker.py — LLM-based listwise reranking.
"""

from typing import List, Dict
from openai import AsyncOpenAI


class LLMReranker:
    """
    Rerank using an LLM to evaluate chunks.

    Strategies:
    - Pointwise: LLM scores each chunk independently (score 1-10)
    - Pairwise: LLM compares two chunks, picks the better one
    - Listwise: LLM receives all chunks and returns ordered list
    """

    def __init__(self, llm_client: AsyncOpenAI, model: str = "gpt-4o-mini"):
        self.llm_client = llm_client
        self.model = model

    async def listwise_rerank(
        self,
        query: str,
        candidates: List[Dict],
        top_k: int = 5,
        max_chunks_in_prompt: int = 10
    ) -> List[Dict]:
        """
        Listwise reranking: LLM receives all chunks and returns ordered list.

        Best quality, highest cost.
        Only works well for small candidate sets (< 20 chunks).
        """
        # Take top candidates (we can only fit so many in the prompt)
        top_candidates = candidates[:max_chunks_in_prompt]

        # Build prompt with all candidates
        chunks_text = "\n\n".join(
            f"Chunk {i+1}: {c.get('text', '')[:500]}"
            for i, c in enumerate(top_candidates)
        )

        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {
                    "role": "system",
                    "content": (
                        "You are a search result reranker. "
                        "Given a query and a set of text chunks, "
                        "rank the chunks by their relevance to the query.\n\n"
                        "Output format: A comma-separated list of chunk numbers "
                        "in order of relevance (most relevant first).\n"
                        "Example: 3, 1, 5, 2, 4\n\n"
                        "Only output the list, nothing else."
                    )
                },
                {
                    "role": "user",
                    "content": f"Query: {query}\n\nChunks:\n{chunks_text}"
                }
            ],
            max_tokens=100,
            temperature=0.1
        )

        # Parse the ranked list
        content = response.choices[0].message.content.strip()
        try:
            ranked_indices = [int(x.strip()) - 1 for x in content.split(",")]
        except (ValueError, IndexError):
            # Fallback: return original order
            return top_candidates[:top_k]

        # Apply ranking
        reranked = [top_candidates[i] for i in ranked_indices if 0 <= i < len(top_candidates)]

        # Assign scores based on position
        for i, chunk in enumerate(reranked):
            chunk["rerank_score"] = 1.0 - (i / len(reranked))
            chunk["rerank_method"] = "llm_listwise"

        return reranked[:top_k]

    async def pointwise_rerank(
        self,
        query: str,
        candidates: List[Dict],
        top_k: int = 5
    ) -> List[Dict]:
        """
        Pointwise reranking: LLM scores each chunk independently.

        More reliable than listwise for large candidate sets.
        More expensive (N LLM calls for N chunks).
        """
        scored = []

        for candidate in candidates:
            text = candidate.get("text", "")[:500]

            response = await self.llm_client.chat.completions.create(
                model=self.model,
                messages=[
                    {
                        "role": "system",
                        "content": (
                            "Rate the relevance of the following document "
                            "to the user's query on a scale of 0-10.\n"
                            "0 = completely irrelevant, 10 = perfectly relevant.\n"
                            "Respond with ONLY a number."
                        )
                    },
                    {
                        "role": "user",
                        "content": f"Query: {query}\n\nDocument: {text}"
                    }
                ],
                max_tokens=5,
                temperature=0.1
            )

            try:
                score = float(response.choices[0].message.content.strip()) / 10.0
            except (ValueError, TypeError):
                score = 0.5

            candidate["rerank_score"] = score
            candidate["rerank_method"] = "llm_pointwise"
            scored.append(candidate)

        scored.sort(key=lambda x: x["rerank_score"], reverse=True)
        return scored[:top_k]
```

**When to use LLM reranking vs. Cross-encoder:**

| Scenario | Recommended Reranker | Reason |
|----------|---------------------|--------|
| High throughput (> 100 QPS) | Cross-encoder (self-hosted) | Latency, cost |
| Low throughput (< 1 QPS), high quality | LLM listwise | Best quality, cost okay at low volume |
| Need explainability | LLM pointwise (can explain scores) | Cross-encoders are black boxes |
| Long documents (> 512 tokens) | Cohere Rerank or LLM | Cross-encoders have token limits |
| Batch processing | Cross-encoder (batched) | LLM doesn't batch well |
| Research/experimentation | Try all three | Need to find what works for YOUR data |

---

### 8. Reranking Benchmark: Quality vs. Latency vs. Cost

```python
"""
rerank_benchmark.py — Compare reranking strategies on quality, latency, cost.
"""

import time
import asyncio
from typing import List, Dict


class RerankBenchmark:
    """
    Compare different reranking strategies.

    Measures:
    - Quality: Does reranking improve top-K precision?
    - Latency: How much time does reranking add?
    - Cost: How much does reranking cost per query?
    - R@1, R@3, R@5 improvement over no-reranking baseline
    """

    async def compare_strategies(
        self,
        test_queries: List[str],
        candidates_data: Dict[str, List[Dict]],
        ground_truth: Dict[str, List[str]],  # query -> [relevant chunk IDs]
        rerankers: Dict[str, object],  # name -> reranker instance
    ) -> Dict:
        """Compare multiple reranking strategies."""
        results = {}

        for name, reranker in rerankers.items():
            total_latency = 0
            correct_at_1 = 0
            correct_at_3 = 0
            correct_at_5 = 0
            total = len(test_queries)

            for query in test_queries:
                candidates = candidates_data.get(query, [])
                relevant = set(ground_truth.get(query, []))

                t0 = time.perf_counter()
                if hasattr(reranker, 'rerank'):
                    reranked = await reranker.rerank(query, candidates, top_k=5)
                else:
                    # Fallback: use a default method
                    reranked = candidates[:5]

                latency = (time.perf_counter() - t0) * 1000
                total_latency += latency

                retrieved_ids = [c["id"] for c in reranked]

                if len(retrieved_ids) >= 1 and retrieved_ids[0] in relevant:
                    correct_at_1 += 1
                if len(retrieved_ids) >= 3 and any(id in relevant for id in retrieved_ids[:3]):
                    correct_at_3 += 1
                if any(id in relevant for id in retrieved_ids):
                    correct_at_5 += 1

            results[name] = {
                "r@1": correct_at_1 / total,
                "r@3": correct_at_3 / total,
                "r@5": correct_at_5 / total,
                "avg_latency_ms": total_latency / total,
                "throughput_qps": 1000 / (total_latency / total),
            }

        return results


# Typical benchmark results:
"""
Reranking Benchmark Results (on MS MARCO passage ranking):

Strategy              | R@1    | R@5    | Latency | Cost/query
----------------------|--------|--------|---------|-----------
No reranking (hybrid) | 0.52   | 0.78   | 100ms   | $0.00001
MiniLM-L-2 (CE)      | 0.61   | 0.85   | 200ms   | $0.00003
MiniLM-L-6 (CE)      | 0.67   | 0.88   | 500ms   | $0.00003
Cohere Rerank v3.5   | 0.71   | 0.91   | 800ms   | $0.05000
GPT-4o-mini listwise | 0.73   | 0.92   | 1500ms  | $0.00200
GPT-4o listwise      | 0.76   | 0.93   | 3000ms  | $0.05000

Key insight: MiniLM-L-6 gives 90% of the quality improvement
at 1% of the cost of GPT-4o. Most production systems should
use a lightweight cross-encoder as their primary reranker.
"""
```

---

### 9. Adaptive Reranking

The insight: not all queries need reranking. Simple queries benefit less from reranking than complex ones.

```python
"""
adaptive_reranker.py — Only rerank when it's worth it.
"""

from typing import List, Dict, Optional
import time


class AdaptiveReranker:
    """
    Selectively applies reranking based on query difficulty.

    Strategy:
    - Simple queries → skip reranking (use first-pass results directly)
    - Moderate queries → cross-encoder reranking
    - Complex queries → LLM reranking

    The cost savings: 60-80% of queries skip the expensive reranking step.
    """

    def __init__(
        self,
        cross_encoder_reranker,
        llm_reranker: Optional = None,
        rerank_threshold: float = 0.6,  # Simple queries below this threshold
    ):
        self.cross_encoder = cross_encoder_reranker
        self.llm_reranker = llm_reranker
        self.rerank_threshold = rerank_threshold

    def _estimate_query_difficulty(self, query: str, candidates: List[Dict]) -> float:
        """
        Estimate how difficult this query is.

        Returns 0.0 (trivial) to 1.0 (very difficult).

        Signals:
        - First-pass max score (low max score → hard query)
        - Query length (longer → potentially harder)
        - Score gap between top-1 and top-5 (large gap → clear winner)
        """
        if not candidates:
            return 1.0

        max_score = max(c.get("score", 0) for c in candidates)

        if len(candidates) >= 5:
            scores = sorted([c.get("score", 0) for c in candidates], reverse=True)
            gap = scores[0] - scores[4]
        else:
            gap = 0

        # Low max score + small gap = hard query
        difficulty = (1.0 - max_score) * 0.7 + (1.0 - min(gap / 0.3, 1.0)) * 0.3

        return min(max(difficulty, 0.0), 1.0)

    async def retrieve(
        self,
        query: str,
        candidates: List[Dict],
        top_k: int = 5
    ) -> Dict:
        """
        Adaptive retrieval: only rerank when needed.

        Returns results + diagnostic info about the decision.
        """
        difficulty = self._estimate_query_difficulty(query, candidates)

        if difficulty < self.rerank_threshold:
            # Simple query: use first-pass results directly
            return {
                "chunks": candidates[:top_k],
                "reranker_used": None,
                "difficulty": difficulty,
                "reason": "query_direct",
                "latency_ms": 0,
            }

        # Difficult query: apply cross-encoder
        t0 = time.perf_counter()
        reranked = await self.cross_encoder.rerank(query, candidates, top_k=top_k)
        ce_latency = (time.perf_counter() - t0) * 1000

        # Very difficult query: optionally apply LLM reranking
        if difficulty > 0.85 and self.llm_reranker and ce_latency < 500:
            llm_reranked = await self.llm_reranker.listwise_rerank(
                query, reranked, top_k=top_k
            )
            return {
                "chunks": llm_reranked,
                "reranker_used": "llm",
                "difficulty": difficulty,
                "reason": "very_difficult",
                "latency_ms": (time.perf_counter() - t0) * 1000,
            }

        return {
            "chunks": reranked,
            "reranker_used": "cross_encoder",
            "difficulty": difficulty,
            "reason": "difficult",
            "latency_ms": ce_latency,
        }
```

---

## ✅ Good Output Examples

### Before Reranking

```
Query: "What's the process for filing a claim after hours?"

First-pass results (by embedding similarity):
1. "Our claims department is open Monday-Friday 9am-5pm EST" (0.91)
2. "Emergency claims can be filed through our 24/7 hotline" (0.85)
3. "Standard claim forms are available on our website" (0.79)
4. "After-hours support is available for urgent issues" (0.74)
5. "Claims processing takes 5-7 business days" (0.68)

→ Chunk 2 is the most relevant ("emergency" + "24/7" = after hours filing)
  But it's ranked #2 because #1 has higher embedding similarity
```

### After Reranking

```
Query: "What's the process for filing a claim after hours?"

Cross-encoder reranked results:
1. "Emergency claims can be filed through our 24/7 hotline" (0.94)
2. "After-hours support is available for urgent issues" (0.89)
3. "Our claims department is open Monday-Friday 9am-5pm EST" (0.72)
4. "Standard claim forms are available on our website" (0.55)
5. "Claims processing takes 5-7 business days" (0.41)

→ The cross-encoder correctly identified that "emergency claims" +
  "24/7 hotline" directly addresses "filing a claim after hours"
```

### LLM Listwise Reranking

```
Query: "Compare the storage limits between the free and pro tiers"

Chunks to rank:
1. "Free tier includes 10GB of storage"
2. "Pro tier includes 100GB of storage"
3. "Enterprise plans have custom storage limits"
4. "Storage can be increased with add-ons"
5. "All plans include unlimited bandwidth"

LLM ranking:
3, 1, 2, 5, 4
→ The LLM correctly identified that we need FREE and PRO comparisons,
  so it ranked the storage-specific chunks higher than general ones
  (even though chunk 5 had higher embedding similarity)
```

---

## ❌ Antipatterns & Failure Modes

### 1. Reranking the Top-3

**What:** Only retrieving 3-5 chunks from first-pass, then reranking them.

**Why it fails:** If the relevant chunk is at position #6 in first-pass results, it never reaches the reranker. You've added reranking latency but gained zero recall improvement.

**Fix:** Always retrieve MORE candidates than you need. N should be at least 4-5x K.

### 2. Reranking Everything

**What:** Reranking all candidates (e.g., N=200 with cross-encoder).

**Why it fails:** The cross-encoder runs 200 forward passes. At 50ms each, that's 10 seconds. Query latency destroys user experience. And most of those bottom-100 chunks are irrelevant even after reranking.

**Fix:** Find the optimal N experimentally. It's usually 20-50 for most systems.

### 3. The Score Threshold Trap

**What:** Applying a rigid reranker score threshold (e.g., "only include chunks with rerank_score > 0.5").

**Why it fails:** Cross-encoder scores are relative, not absolute. A "0.5" score for one query might mean "highly relevant" while for another query it means "barely relevant." The score distribution depends on the query-document pair, not a fixed scale.

**Fix:** Use reranker scores for RELATIVE ranking, not absolute thresholds. Or normalize scores per query (e.g., min-max normalization across candidates for that query).

### 4. Using the Same Model for Retrieval and Reranking

**What:** Using the same bi-encoder embeddings (with different pooling) for both retrieval and "reranking."

**Why it fails:** If the embedding model misses certain relationships in the first pass, it'll miss the same relationships in the "reranking" pass. You're just re-ranking by the same flawed similarity metric.

**Fix:** The reranker MUST be a fundamentally different model (cross-encoder vs. bi-encoder). Different architecture, different training data, different objective.

### 5. Reranking Without Evaluation

**What:** Adding a reranker without measuring whether it actually improves results.

**Why it fails:** Rerankers can HURT if they're not well-suited to your domain. A cross-encoder trained on MS MARCO (web search) might rank chunks differently than needed for your technical documentation.

**Fix:** Always A/B test reranking vs. no-reranking on YOUR queries before deploying.

---

## 🧪 Drills & Challenges

### Drill 1: Reranking from Scratch (45 min)

Implement a simple cross-encoder inspired reranker using an LLM API:

```python
async def llm_rerank_pointwise(query: str, chunks: List[str]) -> List[float]:
    """
    For each chunk, ask the LLM to rate relevance on a scale of 1-10.
    Return scores.

    This is a POINTWISE approach (each chunk scored independently).
    """
    pass
```

Then:
1. Compare pointwise vs. listwise reranking on 10 queries
2. Which produces better ordering? Which is more consistent?
3. When does the LLM contradict itself (chunk A > B in one call, B > A in another)?

### Drill 2: Build the Two-Stage Pipeline (40 min)

Build a complete two-stage retrieval pipeline:

1. Stage 1: Retrieve 100 candidates using hybrid search (dense + sparse)
2. Stage 2: Rerank with any method (cross-encoder, LLM, or Cohere API)
3. Return top-5

```python
class TwoStagePipeline:
    async def search(self, query: str) -> Dict:
        """Full two-stage search."""
        pass
```

**Benchmark:**
- Measure recall@5 without reranking vs. with reranking
- Measure the latency overhead
- Find: at what N does reranking stop improving recall?

### Drill 3: The N vs. Recall Experiment (35 min)

Design and run an experiment to find the optimal N (number of candidates from first-pass) for YOUR system:

```python
async def find_optimal_n(
    first_pass_retriever,
    reranker,
    test_queries: List[tuple],  # (query, gold_chunk_id)
    k_final: int = 5
) -> Dict:
    """
    For N in [5, 10, 20, 50, 100, 200]:
    - Run pipeline with N candidates, K=5 final
    - Measure: recall@5, latency, cost
    - Find the N where recall plateaus

    Report: the optimal N and the shape of the curve.
    """
    pass
```

### Drill 4: The Cross-Encoder vs. LLM Comparison (45 min)

Run a head-to-head comparison on 20 test queries:

```python
comparison = {
    "no_rerank": results_without_reranking,
    "cross_encoder": results_with_MiniLM_L6,
    "llm_pointwise": results_with_GPT4o_mini_pointwise,
    "llm_listwise": results_with_GPT4o_mini_listwise,
    "cohere": results_with_Cohere_Rerank,
}

# For each, measure:
# - Recall@1, Recall@3, Recall@5
# - Average latency
# - Cost per query
# - Position changes: how many chunks moved up/down vs. baseline
```

**Write the analysis:**
- Which reranker gives the best quality improvement per dollar?
- Which gives the best improvement per millisecond?
- Are there QUERY TYPES where one reranker clearly beats others?

### Drill 5: The Adaptive Reranker (30 min)

Build a classifier that predicts whether a query will benefit from reranking:

```python
def will_rerank_help(
    query: str,
    first_pass_results: List[Dict]
) -> float:
    """
    Return a confidence score (0-1) that reranking will improve results.

    Signals:
    - First-pass max score: low → likely needs reranking
    - Score gap between top-1 and top-N: small → ambiguous → needs reranking
    - Query length: very short → likely needs context enrichment before reranking helps
    - Query contains comparison terms ("vs", "compare", "difference"): likely benefits
    """
    pass
```

Then integrate with your pipeline: only run reranker when confidence > 0.7.

### Drill 6: The Reranking Audit (30 min)

Take a set of 50 queries with their first-pass and reranked results. Analyze:

```python
def audit_reranking(
    before: List[Dict],  # Results before reranking
    after: List[Dict],   # Results after reranking
    gold: List[str]      # Gold chunk IDs
) -> Dict:
    """
    Answer:
    - Did reranking improve or hurt? (for each query)
    - Which queries improved most? (common characteristics)
    - Which queries got worse? (common characteristics)
    - What's the net recall change?
    - What's the "regret rate": queries where reranking made top-5 WORSE?
    """
    pass
```

---

## 🚦 Gate Check

Before proceeding to Phase 5.04 (Agentic RAG), you must demonstrate:

1. **Implement a reranking pipeline**: Build a complete two-stage retrieval system (first-pass → cross-encoder rerank → top-K). It must work end-to-end with your Phase 4 Customer Support Bot.

2. **Benchmark the improvement**: Measure recall@5 without reranking vs. with reranking on 20 test queries. Show the improvement as a percentage. Also measure the latency cost.

3. **Find your optimal N**: Run an N-vs-recall experiment and report the optimal N for your system. Show the curve.

4. **Show one case where reranking helped and one where it didn't**: Demonstrate with actual queries, chunks, and scores. Explain WHY reranking helped in one case and not the other.

5. **Answer the reflection questions**:
   - Why can't a bi-encoder match a cross-encoder's precision?
   - How do you choose between open-source cross-encoder, Cohere Rerank, and LLM reranking?
   - What's the optimal N for your system and how did you find it?
   - When would reranking HURT recall (and how do you detect it)?

**Passing criteria:** You can articulate why reranking is the single highest-impact precision improvement for a RAG system. You know that retrieval finds candidates; reranking chooses winners. You've measured the improvement on YOUR data.

---

## 📚 Resources

### Papers
- **"Cross-Encoder vs. Bi-Encoder for Passage Reranking"** — The foundational comparison.
- **"ColBERT: Efficient and Effective Passage Search via Contextualized Late Interaction"** (Khattab & Zaharia, 2020) — The ColBERT paper. A must-read for understanding the quality-speed tradeoff.
- **"RankGPT: Listwise Reranking with Large Language Models"** — Using LLMs for listwise reranking. State-of-the-art but expensive.

### Models & APIs
- **sentence-transformers Cross-Encoder models**: https://www.sbert.net/docs/pretrained_cross-encoders.html
- **Cohere Rerank**: https://cohere.com/rerank
- **ColBERT (official implementation)**: https://github.com/stanford-futuredata/ColBERT

### Production Patterns
- **Elasticsearch + reranker architecture** — Common pattern for combining keyword search with neural reranking.
- **Vespa.ai's reranking support** — Production-grade two-phase ranking.

### Your Own Code
- Your **Customer Support Bot** (Phase 4) should add a reranking stage. It's one of the highest-ROI improvements you can make.
- Reranking COMPOSES with everything you've built: HyDE (better query → better candidates → reranking works even better), contextual retrieval (enriched chunks → cross-encoder sees more context → better scoring).

### Expert-Level Deep Dives
- **For production engineers**: The #1 mistake in reranking is not retrieving enough candidates. Most teams set N=20 and wonder why reranking doesn't help much. Try N=100. You'll be surprised how many good chunks are hiding at positions 30-80.
- **For system architects**: Reranking is a natural place to introduce a QUALITY GATE. Before reranking, you don't know if you found the answer. After reranking, the top score tells you confidence. Use the reranker score as a signal: low max score → trigger fallback (query expansion, broader retrieval, or "I don't know").
- **For critical thinkers**: Reranking is a POINTWISE operation (each query-chunk pair scored independently). This means you can parallelize it trivially. Run reranking across multiple GPUs or API calls in parallel. The latency problem is an engineering problem, not a fundamental limitation.

---

*"First-pass retrieval finds candidates. Reranking chooses winners. Never confuse the two jobs — and never try to do both with the same model."*
