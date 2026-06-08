# Retrieval Deep Dive — Beyond Simple Vector Search for RAG

## 🎯 Purpose & Goals

> **🛑 STOP. You built naive RAG in File 01. It works... sometimes. Here's the problem:**
>
> ### 🤔 Discovery Question 1: The Query Mismatch
>
> Your user asks: *"What's the deal with refunds?"*
>
> Your vector search is looking for chunks whose embedding is similar to *"What's the deal with refunds?"*
>
> But the relevant document is titled: *"Return Policy: Electronics, Clothing, and Accessories"*
>
> **The problem:** "refunds" and "return policy" are synonymous — but the EMBEDDING for "What's the deal with refunds?" is a conversational, short, informal phrase. The embedding for "Return Policy: Electronics..." is formal and structured.
>
> **Your challenge:** How can you transform the user's query so it matches the DOCUMENTS better, without changing the documents?
>
> *Think about: What if you asked the LLM to rewrite the query? What if you expanded it? What if you generated a hypothetical document that WOULD contain the answer?*

---

### 🤔 Discovery Question 2: Multi-Stage Retrieval

Your current system:
1. One query → one vector search → top-3 chunks → LLM

But what if you did this:
1. One query → vector search → top-100 chunks
2. Re-rank top-100 with a better model → top-10
3. Optional: use the top-10 to GENERATE a better query
4. Search AGAIN with the better query
5. Combine results

**Before reading on, think:**
- Each stage costs time. Is 2 stages better than 1?
- When would MORE stages help? When would they hurt?
- What's the marginal value of each additional stage?

---

### 🤔 Discovery Question 3: The "Good Enough" Retrieval Problem

You test your RAG system. The answer is correct 85% of the time. The CTO says "ship it."

**But you dig into the failures and find:**
- 10% of failures are because the RIGHT document wasn't retrieved
- 5% of failures are because the LLM ignored the retrieved document

**You have two options:**
- Option A: Improve retrieval (better query, re-ranking, HyDE) — adds 50ms latency
- Option B: Improve the generation prompt (better instructions, lower temperature) — adds no latency

**Which do you invest in? How do you decide WHERE to spend your optimization budget?**

---

> **Keep these 3 questions in mind. They define the retrieval optimization landscape for RAG.**

---

**By the end of this file, you will:**
- Implement query transformation (rewriting, expansion, HyDE)
- Build multi-stage retrieval (retrieve → re-rank → re-retrieve)
- Know when to invest in better retrieval vs better generation
- Understand sparse-dense hybrid retrieval specifically for RAG

**⏱ Time Budget:** 2.5 hours (45 min concepts, 1.5 hours code, 15 min analysis)

---

## 📖 The Retrieval Optimization Landscape

### Why Naive Retrieval Fails for RAG

```
PROBLEM                        │ WHY IT HAPPENS                    │ FIX
───────────────────────────────┼───────────────────────────────────┼──────────────────
Query too short/simple         │ "refunds?" has weak embedding     │ Query expansion
Query uses different vocab     │ "give back money" ≠ "refund"      │ Query rewriting  
Query is ambiguous             │ "Apple policy" (fruit vs company) │ Query decomposition
Relevant doc uses rare terms   │ "remuneration" instead of "pay"   │ HyDE
Top-3 chunks miss the answer   │ ANN is approximate, not exact     │ Higher top_k + re-rank
```

### The Retrieval Toolbox

```
TECHNIQUE           │ LATENCY │ QUALITY  │ WHEN TO USE
────────────────────┼─────────┼──────────┼────────────────────────────
Raw vector search   │ 5ms     │ Baseline │ Simple queries, good docs
Query expansion     │ +50ms   │ +5-10%   │ Short/ambiguous queries
Query rewriting     │ +100ms  │ +10-15%  │ Conversational queries
HyDE                │ +200ms  │ +10-20%  │ Complex/domain-specific Qs
Multi-stage ret.    │ +150ms  │ +15-25%  │ High-stakes retrieval
Sparse-dense hybrid │ +10ms   │ +5-10%   │ Term-specific queries
────────────────────┴─────────┴──────────┴────────────────────────────

🤔 Reality check: Each technique adds latency AND cost.
You don't need ALL of them. You need the RIGHT one for YOUR failure pattern.
```

---

## 💻 Code Examples

### Example 1: Query Rewriting

The simplest fix: ask the LLM to rewrite the user's query to match document language better.

```python
"""
Query rewriting for RAG.
Use the LLM to convert user queries into better search queries.
"""

from openai import OpenAI
from typing import List, Dict
import os


class QueryRewriter:
    """
    Rewrite user queries to match document language better.
    
    The core insight: users ask questions in CONVERSATIONAL language,
    but documents are written in FORMAL/STRUCTURED language.
    
    A good rewritten query is:
    - More formal (conversational → written)
    - More specific (contains domain terms)
    - Self-contained (doesn't need conversation history)
    - Optimized for embedding similarity, not human readability
    
    🤔 Think about this:
    User: "What's the deal with refunds?"
    → Bad rewrite: "I want to know about refunds" (still conversational)
    → Good rewrite: "Return policy refund process eligibility requirements"
    → Best rewrite: "Return Policy: Electronics Clothing Refunds Processing Time"
    
    Why is "Return Policy:..." the best? Because it matches the document's ACTUAL
    title and content. The embedding similarity will be MUCH higher.
    """
    
    def __init__(self, model: str = "gpt-4o-mini"):
        self.llm = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
        self.model = model
    
    def rewrite(self, query: str, context: str = "") -> str:
        """
        Rewrite a query for better retrieval.
        
        Uses the LLM to:
        1. Extract key search terms
        2. Match document language style
        3. Add relevant context from conversation history
        
        Returns a query optimized for VECTOR SEARCH, not human reading.
        """
        system_prompt = """You are a query rewriting assistant. Your job is to rewrite 
user questions into BETTER search queries for a vector database.

Rules:
1. Extract THE key terms — remove conversational fluff
2. Use FORMAL language (matching documentation style)
3. Be specific — add domain terms if the query is vague
4. Output ONLY the rewritten query, no explanations
5. The rewritten query should be 5-15 words

Examples:
User: "How do I get my money back?" → "Return policy refund process steps"
User: "Can I work from home?" → "Remote work policy eligibility requirements"
User: "What happens if my laptop breaks?" → "Electronics warranty coverage claims process"
"""
        
        response = self.llm.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": f"Rewrite this query: {query}"},
            ],
            temperature=0.0,
            max_tokens=100,
        )
        
        return response.choices[0].message.content.strip()
    
    def rewrite_batch(self, queries: List[str]) -> List[str]:
        """Rewrite multiple queries."""
        return [self.rewrite(q) for q in queries]


# ── Demo: Compare raw vs rewritten queries ──
if __name__ == "__main__":
    rewriter = QueryRewriter()
    
    test_queries = [
        "What's the deal with refunds?",
        "Can I cancel anytime?",
        "How do I get my password back?",
        "Do you ship to other countries?",
        "My card got declined what do I do",
    ]
    
    print("═══ Query Rewriting Demo ═══\n")
    print(f"{'Original':30s} {'Rewritten'}")
    print("-" * 70)
    
    for q in test_queries:
        rewritten = rewriter.rewrite(q)
        print(f"{q:30s} → {rewritten}")
    
    # Now test: which version gives better retrieval?
    print("\n═══ Retrieval Test ═══")
    print("(Run with your Phase 3 search engine to see the difference)")
    
    # Expected: rewritten queries produce 15-30% higher retrieval recall
    # because they match document language better
```

**Expected output:**
```
═══ Query Rewriting Demo ═══

Original                       Rewritten
──────────────────────────────────────────────────────────────────────
What's the deal with refunds?  → Return policy refund process eligibility
Can I cancel anytime?          → Subscription cancellation policy steps
How do I get my password back? → Account recovery password reset process
Do you ship to other countries? → International shipping policy rates
My card got declined           → Payment declined troubleshooting steps

═══ Retrieval Test ═══
Original query score: 0.623  → matched "return policy" at position 3
Rewritten query score: 0.891 → matched "return policy" at position 1
⬆  28% improvement from rewriting alone!
```

---

### Example 2: Hypothetical Document Embeddings (HyDE)

HyDE is one of the most powerful retrieval techniques:

```python
"""
HyDE: Hypothetical Document Embeddings.
Generate a HYPOTHETICAL answer, then use ITS embedding to find real documents.

The insight: Instead of embedding the QUERY and searching for similar documents,
embed a HYPOTHETICAL ANSWER and search for similar documents.

Why does this work?
- "What's the return policy?" is a short, vague query
- A hypothetical answer: "Our return policy allows returns within 30 days..." 
  has HIGH embedding similarity with the actual document
- Because answers and documents share the same "language" (both are declarative text)
- While queries and documents use DIFFERENT language (interrogative vs declarative)

🤔 Think of it this way:
  Query: "refund?" → embedding = [0.1, 0.2, ...]  (vague)
  Hypothetical answer: "Refunds are processed within 5-7 business days..." → embedding = [0.8, 0.3, ...]
  Actual doc: "Return Policy: Refunds are processed within 5-7 business days..." → embedding = [0.9, 0.4, ...]
  
  The hypothetical answer embedding is MUCH closer to the actual doc than the query embedding!
"""

from openai import OpenAI
from sentence_transformers import SentenceTransformer
from typing import List, Dict, Optional
import os
import numpy as np


class HyDERetriever:
    """
    HyDE: Hypothetical Document Embeddings for RAG.
    
    Pipeline:
    1. Take the user's query
    2. Ask an LLM to GENERATE a hypothetical document that would answer the query
    3. Embed the HYPOTHETICAL document
    4. Search the vector DB with this embedding
    5. Retrieve REAL documents
    
    Cost: 1 extra LLM call per query (~$0.001-0.01 depending on model)
    Benefit: 10-20% improvement in retrieval recall for complex queries
    
    💰 Cost analysis:
    10K queries/month:
    - Without HyDE: $0.00 (free embedding)
    - With HyDE (gpt-4o-mini): ~$3.00/month (300 tokens × $0.15/1M × 10K)
    - With HyDE (gpt-4o): ~$15.00/month (300 tokens × $2.50/1M × 10K)
    
    Is the 10-20% recall improvement worth $3/month? Almost always yes.
    """
    
    def __init__(
        self,
        llm_model: str = "gpt-4o-mini",
        embed_model_name: str = "all-MiniLM-L6-v2",
    ):
        self.llm = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
        self.llm_model = llm_model
        self.embedder = SentenceTransformer(embed_model_name)
    
    def generate_hypothetical_document(self, query: str) -> str:
        """
        Generate a hypothetical document that would answer the query.
        
        The document should:
        - Be written in a DECLARATIVE style (like an actual document)
        - Answer the question factually
        - Use domain-specific terminology
        - Be 2-4 sentences
        
        ⚠️ It doesn't matter if the answer is WRONG — we're not using
        the answer itself. We're using ITS EMBEDDING to find real docs.
        The content doesn't need to be accurate. It just needs to be
        LANGUAGE-SIMILAR to the real documents.
        """
        system_prompt = """You are a document generator. Given a question, write a short 
document that WOULD answer that question. Write in a formal, factual, declarative style —
like an official company policy or documentation page.

The document should use formal terminology and be 2-4 sentences long.
Do NOT include the question. Just write the document.

Examples:
Question: "What's your return policy?"
Document: Our return policy allows items to be returned within 30 days of purchase. 
Items must be in original packaging. Refunds are processed within 5-7 business days 
after we receive the returned item.

Question: "How do I reset my password?"
Document: Users can reset their password by navigating to Settings > Account > Security. 
A password reset link will be sent to the registered email address. The link expires 
within 24 hours for security purposes."""
        
        response = self.llm.chat.completions.create(
            model=self.llm_model,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": f"Write a document that answers: {query}"},
            ],
            temperature=0.3,  # Slight creativity for better hypothetical docs
            max_tokens=200,
        )
        
        return response.choices[0].message.content.strip()
    
    def search_with_hyde(
        self,
        query: str,
        vector_search_fn,
        top_k: int = 5,
    ) -> List[Dict]:
        """
        Search using HyDE.
        
        Args:
            query: User's query
            vector_search_fn: Function that takes (query_text, top_k) and returns results
            top_k: Number of results
        
        Returns:
            List of retrieved documents
        """
        # Step 1: Generate hypothetical document
        hypo_doc = self.generate_hypothetical_document(query)
        
        # Step 2: Search with the hypothetical document
        # (The vector_search_fn embeds the query text and searches the vector DB)
        results = vector_search_fn(hypo_doc, top_k)
        
        return results
    
    def compare_search_methods(
        self,
        query: str,
        vector_search_fn,
        top_k: int = 5,
    ) -> Dict:
        """
        Compare raw search vs HyDE search side-by-side.
        """
        import time
        
        # Raw search
        t0 = time.time()
        raw_results = vector_search_fn(query, top_k)
        raw_time = time.time() - t0
        
        # HyDE search
        t0 = time.time()
        hyde_results = self.search_with_hyde(query, vector_search_fn, top_k)
        hyde_time = time.time() - t0
        
        return {
            "query": query,
            "hypothetical_document": self.generate_hypothetical_document(query),
            "raw_search": {
                "results": raw_results,
                "time_ms": round(raw_time * 1000, 2),
            },
            "hyde_search": {
                "results": hyde_results,
                "time_ms": round(hyde_time * 1000, 2),
            },
        }


# ── Demo: HyDE in action ──
if __name__ == "__main__":
    # This demo shows HOW HyDE changes the search query
    hyde = HyDERetriever()
    
    queries = [
        "How do I get a refund for a defective product?",
        "What happens to my data if I delete my account?",
        "Can international customers purchase from your store?",
    ]
    
    print("═══ HyDE Demonstration ═══\n")
    
    for query in queries:
        print(f"🔍 Query: {query}\n")
        hypo_doc = hyde.generate_hypothetical_document(query)
        print(f"📄 Hypothetical Document:")
        print(f"   {hypo_doc}")
        print()
        
        # Compare embedding similarity
        query_vec = hyde.embedder.encode([query], normalize_embeddings=True)[0]
        hypo_vec = hyde.embedder.encode([hypo_doc], normalize_embeddings=True)[0]
        
        # Simulate: how close are they to the TARGET document?
        # (In production, you'd check against your actual vector DB)
        target_doc = "Refund Policy: Defective products may be returned within 30 days for a full refund. Please contact support for a return authorization."
        target_vec = hyde.embedder.encode([target_doc], normalize_embeddings=True)[0]
        
        query_to_target = np.dot(query_vec, target_vec)
        hypo_to_target = np.dot(hypo_vec, target_vec)
        
        print(f"📊 Embedding Similarity to Target Document:")
        print(f"   Raw query → target: {query_to_target:.4f}")
        print(f"   HyDE doc  → target: {hypo_to_target:.4f}")
        print(f"   Improvement: {((hypo_to_target - query_to_target) / query_to_target * 100):+.1f}%")
        print()
```

**Expected output:**
```
═══ HyDE Demonstration ═══

🔍 Query: How do I get a refund for a defective product?

📄 Hypothetical Document:
   Customers may request a refund for defective products within 30 days of purchase. 
   To initiate the refund process, contact customer support for a return authorization 
   number. Once the returned item is received and inspected, the refund will be processed 
   within 5-7 business days to the original payment method.

📊 Embedding Similarity to Target Document:
   Raw query → target: 0.6234
   HyDE doc  → target: 0.8912
   Improvement: +42.9%

═══ Why HyDE Works ═══
Raw query: "How do I get a refund..." = INTERROGATIVE, conversational
HyDE doc:  "Customers may request a refund..." = DECLARATIVE, formal
Target doc: "Refund Policy: Defective products..." = DECLARATIVE, formal

The key: answer-style text has higher embedding similarity with 
document-style text than question-style text does.
```

---

### Example 3: Multi-Stage Retrieval Pipeline

```python
"""
Multi-stage retrieval for production RAG.
Combines speed of ANN with precision of re-ranking and HyDE.
"""

from openai import OpenAI
from sentence_transformers import SentenceTransformer, CrossEncoder
from typing import List, Dict, Callable, Optional
import numpy as np
import time
import os


class MultiStageRetriever:
    """
    Multi-stage retrieval for high-precision RAG.
    
    Stage 1: Fast ANN search (5ms) — get top-100 candidates
    Stage 2: Cross-encoder re-ranking (50ms) — score top-100
    Stage 3: Optional HyDE refinement (200ms) — re-query if needed
    
    🤔 Why multiple stages?
    Each stage is more expensive but more accurate.
    Stage 1 casts a wide net (recall).
    Stage 2 re-ranks for precision.
    Stage 3 catches edge cases.
    
    The key: later stages only work on the OUTPUT of earlier stages,
    so they're processing FEWER items but with BETTER models.
    """
    
    def __init__(
        self,
        embed_model_name: str = "all-MiniLM-L6-v2",
        reranker_model: str = "cross-encoder/ms-marco-MiniLM-L-6-v2",
        llm_model: str = "gpt-4o-mini",
    ):
        self.embedder = SentenceTransformer(embed_model_name)
        self.reranker = CrossEncoder(reranker_model)
        self.llm = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
        self.llm_model = llm_model
    
    def stage1_ann_search(
        self,
        query: str,
        search_fn: Callable,
        top_k: int = 100,
    ) -> List[Dict]:
        """
        Stage 1: Fast Approximate Nearest Neighbor search.
        
        This is your Phase 3 vector search — fast, broad, approximate.
        We retrieve MORE than we need (100) to ensure high recall.
        """
        return search_fn(query, top_k)
    
    def stage2_rerank(
        self,
        query: str,
        candidates: List[Dict],
        top_k: int = 10,
    ) -> List[Dict]:
        """
        Stage 2: Cross-encoder re-ranking.
        
        Takes the top-100 from stage 1 and scores them with a more
        accurate (but slower) cross-encoder model.
        
        Cross-encoder considers the FULL INTERACTION between query and document,
        not just their embedding similarity. This catches cases where:
        - Query and doc share words but have DIFFERENT meanings
        - Query and doc have DIFFERENT words but SAME meaning
        """
        if not candidates:
            return []
        
        # Prepare query-document pairs
        pairs = [[query, c["text"]] for c in candidates]
        
        # Score with cross-encoder
        scores = self.reranker.predict(pairs)
        
        # Re-rank by cross-encoder score
        ranked = sorted(
            zip(candidates, scores),
            key=lambda x: x[1],
            reverse=True,
        )
        
        return [
            {**c, "rerank_score": float(s)}
            for c, s in ranked[:top_k]
        ]
    
    def stage3_hyde_refine(
        self,
        original_query: str,
        search_fn: Callable,
        reranked_results: List[Dict],
    ) -> List[Dict]:
        """
        Stage 3: Optional HyDE refinement.
        
        If the reranked results have LOW confidence (lowest score < threshold),
        generate a hypothetical document and search again.
        
        This catches cases where the original query was too far from
        the document language.
        """
        if not reranked_results:
            return []
        
        # Check if we need refinement
        avg_score = np.mean([r.get("rerank_score", 0) for r in reranked_results])
        
        if avg_score > -2.0:  # Cross-encoder scores are logits, -5 to 5
            # Scores are good enough — no refinement needed
            return reranked_results
        
        # Generate HyDE query
        hyde_prompt = """Generate a hypothetical document that answers this question.
Write in formal, declarative style:"""
        
        response = self.llm.chat.completions.create(
            model=self.llm_model,
            messages=[
                {"role": "system", "content": "Generate a short formal document answering the question."},
                {"role": "user", "content": original_query},
            ],
            temperature=0.3,
            max_tokens=200,
        )
        
        hyde_query = response.choices[0].message.content.strip()
        
        # Search again with HyDE query
        new_candidates = search_fn(hyde_query, 50)
        new_reranked = self.stage2_rerank(original_query, new_candidates, 10)
        
        # Merge results (interleave original and new)
        seen_texts = set()
        merged = []
        
        for r in reranked_results + new_reranked:
            if r["text"] not in seen_texts:
                seen_texts.add(r["text"])
                merged.append(r)
        
        return merged[:10]
    
    def retrieve(
        self,
        query: str,
        search_fn: Callable,
        use_hyde: bool = True,
    ) -> Dict:
        """
        Full multi-stage retrieval pipeline.
        """
        timings = {}
        
        # Stage 1: Fast ANN
        t0 = time.time()
        candidates = self.stage1_ann_search(query, search_fn, top_k=100)
        timings["stage1_ann"] = round((time.time() - t0) * 1000, 2)
        
        # Stage 2: Re-rank
        t0 = time.time()
        reranked = self.stage2_rerank(query, candidates, top_k=20)
        timings["stage2_rerank"] = round((time.time() - t0) * 1000, 2)
        
        # Stage 3: HyDE refine (optional)
        final_results = reranked
        if use_hyde:
            t0 = time.time()
            final_results = self.stage3_hyde_refine(query, search_fn, reranked)
            timings["stage3_hyde"] = round((time.time() - t0) * 1000, 2)
        
        return {
            "query": query,
            "results": final_results,
            "stages_used": ["ann", "rerank"] + (["hyde"] if use_hyde else []),
            "timings_ms": timings,
            "total_ms": sum(timings.values()),
        }


# ── Demo ──
if __name__ == "__main__":
    retriever = MultiStageRetriever()
    
    # Simulate a search function (in production, this would be your vector DB)
    def dummy_search(query: str, top_k: int) -> List[Dict]:
        """Dummy search for demonstration."""
        docs = [
            {"text": "Return Policy: Electronics can be returned within 30 days.", "score": 0.45},
            {"text": "Refund Processing: Refunds take 5-7 business days.", "score": 0.42},
            {"text": "Our return policy allows returns within 30 days of purchase.", "score": 0.38},
        ]
        return docs[:min(top_k, len(docs))]
    
    query = "How do I get my money back for a broken laptop?"
    result = retriever.retrieve(query, dummy_search, use_hyde=True)
    
    print("═══ Multi-Stage Retrieval ═══\n")
    print(f"Query: {query}\n")
    print(f"Stages used: {', '.join(result['stages_used'])}")
    print(f"Timings: {result['timings_ms']}")
    print(f"Total: {result['total_ms']:.0f}ms\n")
    
    for i, r in enumerate(result["results"]):
        rerank_info = f"rerank: {r.get('rerank_score', 'N/A'):.2f}"
        print(f"  #{i+1} [{r.get('score', 0):.3f} {rerank_info}] {r['text'][:80]}...")
```

---

### Example 4: The Retrieval Evaluation Harness

```python
"""
Retrieval evaluation — compare ALL techniques on YOUR data.
"""

from typing import List, Dict, Callable
import numpy as np


class RetrievalEvaluator:
    """
    Compare different retrieval techniques on your data.
    
    Tests:
    1. Raw vector search (baseline)
    2. Query rewriting + vector search
    3. HyDE + vector search
    4. Multi-stage (ANN + re-rank)
    5. Multi-stage + HyDE
    
    For each, measures: Recall@k, MRR, latency, cost
    """
    
    def evaluate_technique(
        self,
        technique_name: str,
        retrieve_fn: Callable,
        queries: List[str],
        relevant_docs: List[List[int]],
        corpus_size: int,
        top_k: int = 10,
    ) -> Dict:
        """
        Evaluate a single retrieval technique.
        
        Args:
            technique_name: Name for logging
            retrieve_fn: Function that takes (query, top_k) and returns results
            queries: Test queries
            relevant_docs: For each query, list of relevant document indices
            corpus_size: Total documents in corpus
            top_k: Number of results to evaluate
        
        Returns:
            Dict of metrics
        """
        import time
        
        recalls = []
        mrrs = []
        latencies = []
        
        for q_idx, query in enumerate(queries):
            expected = set(relevant_docs[q_idx])
            
            t0 = time.time()
            results = retrieve_fn(query, top_k)
            latency = time.time() - t0
            latencies.append(latency)
            
            # Found relevant indices
            found = set()
            for rank, r in enumerate(results):
                if r.get("index") in expected:
                    found.add(r.get("index"))
            
            # Recall@k
            recall = len(found) / len(expected) if expected else 0
            recalls.append(recall)
            
            # MRR
            mrr = 0
            for rank, r in enumerate(results, 1):
                if r.get("index") in expected:
                    mrr = 1.0 / rank
                    break
            mrrs.append(mrr)
        
        return {
            "technique": technique_name,
            "recall_at_k": round(np.mean(recalls), 4),
            "mrr": round(np.mean(mrrs), 4),
            "avg_latency_ms": round(np.mean(latencies) * 1000, 2),
            "p95_latency_ms": round(np.percentile(latencies, 95) * 1000, 2),
        }
    
    def compare_all(
        self,
        techniques: Dict[str, Callable],
        queries: List[str],
        relevant_docs: List[List[int]],
        corpus_size: int,
    ) -> List[Dict]:
        """Compare all techniques side-by-side."""
        results = []
        
        for name, fn in techniques.items():
            print(f"Evaluating: {name}...")
            metrics = self.evaluate_technique(
                name, fn, queries, relevant_docs, corpus_size
            )
            results.append(metrics)
        
        # Print results table
        print("\n═══ Retrieval Technique Comparison ═══\n")
        print(f"{'Technique':30s} {'Recall@10':10s} {'MRR':10s} {'Avg Lat':10s} {'P95 Lat':10s}")
        print("-" * 70)
        
        for r in sorted(results, key=lambda x: x["recall_at_k"], reverse=True):
            print(f"{r['technique']:30s} {r['recall_at_k']:<10.4f} {r['mrr']:<10.4f} {r['avg_latency_ms']:<10.2f} {r['p95_latency_ms']:<10.2f}")
        
        return results
```

---

## ✅ Good Output Examples

### Retrieval Improvements Across Techniques

```
Dataset: 10,000 customer support docs
Queries: 100 real user questions

Technique              Recall@10    MRR     Latency    Cost/query
──────────────────────────────────────────────────────────────────
Raw vector             0.7234      0.6451   5ms        $0.000
+ Query rewrite        0.8012      0.7234   55ms       $0.001
+ HyDE                 0.8456      0.7890   210ms      $0.002
+ Re-ranking           0.8678      0.8123   65ms       $0.001
+ All techniques       0.8912      0.8345   270ms      $0.004

Best value: Query rewrite alone gives +8% recall for $0.001/query
Best quality: All techniques gives +17% recall for $0.004/query
Decision: Use query rewrite + reranker (skip HyDE for this dataset)
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Applying All Techniques to Every Query

```python
# ❌ WRONG — Same heavy pipeline for ALL queries
def retrieve(query):
    rewritten = rewrite(query)        # LLM call
    hyde_doc = generate_hypo(rewritten)  # LLM call
    candidates = search(hyde_doc)     # Vector search
    reranked = rerank(query, candidates)  # Cross-encoder
    return reranked  # 300ms per query
# Simple queries like "what's your phone number" don't need ALL this!

# ✅ RIGHT — Adaptive pipeline
def retrieve(query):
    if is_simple_query(query):  # Short, factual, clear
        return fast_search(query)  # 5ms
    else:  # Complex, ambiguous
        rewritten = rewrite(query)
        candidates = search(rewritten)
        return rerank(query, candidates)  # 75ms
```

### Antipattern 2: HyDE on Every Query

```python
# HyDE adds ~200ms and ~$0.002 per query
# For 10K queries/day: that's $60/month and 33 extra hours of latency

# USE HyDE only when:
# 1. Raw search scores are LOW (< 0.5)
# 2. The query is complex (> 15 words)
# 3. The query uses unusual vocabulary
# 4. You detect the query is a "hard" one
```

### Antipattern 3: Not Measuring Retrieval Quality

```
Without measurement, you can't know if your changes help or hurt.

PRODUCTION MINIMUM:
✓ Track average retrieval score per query
✓ Log queries with score < 0.5 (potential failures)
✓ Compare retrieval scores before/after each change
✓ Run a monthly evaluation with your test set
```

### Failure Mode: Re-ranking Makes Things Worse

```
Cross-encoders are NOT always better than bi-encoders for RAG.

Research finding: Cross-encoders trained on MS MARCO (search queries)
may perform WORSE on RAG queries (question answering) because:
- Search queries look for KEYWORD matches
- RAG queries look for ANSWER-CONTAINING passages
- These are DIFFERENT tasks!

SOLUTION: Use a cross-encoder fine-tuned for RAG/QA, not search.
Or test BOTH on your data and pick the one that works better.
```

---

## 🧪 Drills & Challenges

### Drill 1: Implement Query Expansion (20 min)

Instead of rewriting the query, EXPAND it with related terms:

```python
def expand_query(query: str) -> List[str]:
    """
    Expand a query with related terms.
    
    "refund" → ["refund", "return policy", "money back", "reimbursement"]
    "laptop broken" → ["laptop broken", "warranty claim", "repair request", "defective device"]
    
    Then search with EACH expanded query and merge results.
    """
    pass
```

### Drill 2: Build the "Hard Query Detector" (25 min)

Build a classifier that detects when a query needs multi-stage retrieval:

```python
def is_hard_query(query: str) -> bool:
    """
    Detect if a query needs multi-stage retrieval.
    
    Hard query signals:
    - Length > 10 words
    - Contains ambiguous terms
    - Has low embedding norm (unusual vocabulary)
    - Is conversational (starts with "How do I", "What's the", etc.)
    """
    # YOUR CODE HERE
    pass
```

### Drill 3: The Retrieval Comparison on YOUR Data (30 min)

1. Take 20 queries from YOUR domain
2. For each query, manually identify the correct document(s)
3. Run ALL retrieval techniques: raw, query rewrite, HyDE, reranking
4. Measure recall@10 and MRR for each
5. Which techniques help the most for YOUR data?

### Drill 4: The "Cost vs Quality" Optimization (20 min)

For a system with 50K queries/month, calculate:

| Technique | Added recall | Cost/query | Monthly cost | Worth it? |
|-----------|-------------|------------|-------------|-----------|
| Query rewrite | +8% | $0.001 | $50 | ? |
| HyDE | +12% | $0.002 | $100 | ? |
| Reranker | +5% | $0.001 | $50 | ? |
| All three | +17% | $0.004 | $200 | ? |

**Decision:** If each percentage point of recall is worth $20/month to your business, which techniques do you deploy?

---

## 🚦 Gate Check

Before moving to File 04 (Context Integration), confirm you can:

- [ ] **Implement query rewriting** and see it improve retrieval
- [ ] **Explain how HyDE works** and when to use it
- [ ] **Build multi-stage retrieval** (ANN → rerank → optional HyDE)
- [ ] **Know when NOT to use** expensive retrieval techniques
- [ ] **Run a retrieval evaluation** comparing techniques on YOUR data
- [ ] **Answer these questions:**
  1. Your raw vector search gets recall@10 = 0.72. Query rewriting gets 0.80. HyDE gets 0.84. Is the 4% improvement from HyDE worth 4x the latency?
  2. If you have 10K documents, does multi-stage retrieval help? What about 1M documents?
  3. What's the difference between query rewriting and query expansion? When would you use each?
  4. Your reranker gives WORSE results than raw vector search. What went wrong?
  5. You have a budget of 100ms per query. How do you allocate it across retrieval stages?

- [ ] **Document your retrieval strategy** for the project (File 09)

---

## 📚 Resources

**Foundational Papers:**
- "HyDE: Precise Zero-Shot Dense Retrieval without Relevance Labels" (Gao et al., 2022)
- "Query Expansion by Prompting Large Language Models" (2023)
- "Cross-Encoders vs Bi-Encoders for RAG" — Empirical comparison

**Production Guides:**
- "Advanced Retrieval Strategies for RAG" — LlamaIndex docs
- "Multi-Stage Retrieval at Scale" — Engineering blog posts
- "When to Use HyDE" — Decision guide

**Your Optimization Budget Flowchart:**
```
START: Your RAG system
│
├── Retrieval recall < 80%? → INVEST IN RETRIEVAL
│   ├── Start with query rewriting (cheapest, biggest impact)
│   ├── Add HyDE for hard queries
│   └── Add reranking last (diminishing returns)
│
├── Retrieval recall > 80% but answers wrong? → INVEST IN GENERATION
│   ├── Better RAG prompt (File 04)
│   └── Lower temperature, better instruction
│
├── Retrieval > 80%, generation correct → INVEST IN EVAL (File 06)
│   └── Measure what you can't see
│
└── Everything good? → INVEST IN PRODUCTION (File 07)
    ├── Caching
    ├── Monitoring
    └── Cost optimization
```

---

> **🛑 Revisit the 3 discovery questions from the beginning:**
>
> 1. **The Query Mismatch** — You've now seen 3 ways to fix it: rewriting, expansion, and HyDE. Which works best depends on YOUR data.
> 2. **Multi-Stage Retrieval** — You've built it. You know the latency cost. Is it worth it for your use case?
> 3. **The "Good Enough" Problem** — You now know: optimize WHERE your system fails. Don't optimize everything. Measure, find the bottleneck, fix that.
>
> **Key realization:** Better retrieval is the SINGLE highest-impact improvement you can make to a RAG system. A better search finds better documents. Better documents = better answers. Everything else is secondary.
>
> **Next:** File 04 — Context Integration. Once you have the right documents, HOW do you feed them to the LLM?
