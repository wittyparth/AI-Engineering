# What Is RAG? — Building It From Scratch

## 🎯 Purpose & Goals

> **🛑 STOP. Before I define RAG — think.**
>
> You have TWO things you built in previous phases:
>
> 1. **Phase 1:** An LLM that can answer questions (your Multi-Provider LLM Gateway)
> 2. **Phase 3:** A search engine that finds relevant documents (your Knowledge Search Engine)
>
> ### 🤔 Discovery Question 1: Before You Read the Definition
>
> You work for a company. They have 5,000 internal documents. An employee asks:
>
> *"What's our policy on remote work for international contractors?"*
>
> You have:
> - A search engine that can find the top-3 most relevant documents
> - An LLM that can answer questions based on provided text
>
> **Your challenge: Design the simplest possible system that combines these two to answer the employee's question. Don't worry about frameworks, libraries, or best practices. Just describe the DATA FLOW — step by step — in plain English.**
>
> *What are the steps from question → answer?*

---

### 🤔 Discovery Question 2: The "Just Stuff It" Approach

Your search engine returns 3 chunks. Each chunk is ~500 characters. Total: 1,500 characters.

You need to give this to the LLM. But the LLM expects a prompt. 

**Before reading on, write a prompt template that:**
1. Takes the user's question
2. Takes the retrieved documents
3. Instructs the LLM to answer using ONLY those documents

**Then ask yourself:** What happens if the LLM ignores the instruction and answers from its training data instead? How would you word the prompt to prevent this?

---

### 🤔 Discovery Question 3: The Attribution Problem

Your RAG system returns this answer:

> "Our remote work policy allows international contractors to work from any location, but they must ensure compliance with local tax laws. You can find the full policy in section 4.2 of the employee handbook."

**Sounds great. BUT:**
- Is this answer IN the retrieved documents, or did the LLM make it up?
- How would you verify?
- How would you SHOW the user which document this came from?

**Think about attribution BEFORE building the system.** This is what separates demos from production.

---

> **Keep these 3 questions in mind. By the end of this file, you'll have a working answer for all of them.**

---

**By the end of this file, you will:**
- Build RAG from scratch: connect your Phase 3 search engine to an LLM
- Understand the RAG pipeline: Retrieve → Augment → Generate
- See exactly where naive RAG works and where it fails
- Know what each subsequent file in this phase fixes

**⏱ Time Budget:** 2.5 hours (30 min concept, 1.5 hours code, 30 min analysis)

---

## 📖 The RAG Pipeline — Retrieve, Augment, Generate

### What RAG Actually Is

RAG = **R**etrieval-**A**ugmented **G**eneration.

It's not a framework. It's not a library. It's a **pattern**:

```
1. RETRIEVE:  Find relevant documents for the user's query
2. AUGMENT:   Stuff those documents + the query into a prompt
3. GENERATE:  LLM produces an answer based on the documents

That's it. Everything else is optimization.
```

### The Naive RAG Flow

```
User: "What's your return policy for electronics?"

    │
    ▼
┌──────────────────┐
│ 1. EMBED QUERY   │  (same embedding model as your index)
│    → vector      │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│ 2. SEARCH INDEX  │  (ChromaDB/Qdrant — from Phase 3)
│    → top-3 chunks│
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│ 3. BUILD PROMPT  │  (template: question + documents)
│    → prompt text │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│ 4. LLM GENERATE  │  (GPT-4, Claude, etc.)
│    → answer text │
└──────┬───────────┘
       │
       ▼
User: "Our electronics return policy allows returns within 30 days..."
```

**This is naive RAG.** It works ~60-70% of the time in production. The remaining 30-40% fails in interesting ways that the rest of this phase fixes.

---

## 💻 Code Examples — Naive RAG from Scratch

### Example 1: Building RAG on Top of Your Phase 3 Search

Let's connect the search engine you built in Phase 3 to an LLM:

```python
"""
Naive RAG from scratch.
No frameworks. Just Phase 3 search + Phase 1 LLM.
"""

import os
from typing import List, Dict, Optional
from dataclasses import dataclass
import json
import time

# ── Phase 1 Components (LLM) ──
# Using OpenAI for this example — replace with your Phase 1 Gateway
from openai import OpenAI

# ── Phase 3 Components (Search) ──
from sentence_transformers import SentenceTransformer
import chromadb
from chromadb.config import Settings


@dataclass
class RAGConfig:
    """Configuration for your RAG system."""
    # LLM settings
    llm_model: str = "gpt-4o-mini"  # Fast, cheap, good for RAG
    llm_temperature: float = 0.0     # RAG should be deterministic!
    llm_max_tokens: int = 500
    
    # Retrieval settings
    embedding_model: str = "all-MiniLM-L6-v2"
    top_k: int = 3                   # Number of chunks to retrieve
    chunk_size: int = 500            # Must match your ingestion
    
    # Prompt template
    system_prompt: str = """You are a helpful customer support assistant.
Answer the user's question based ONLY on the provided context.
If the context doesn't contain the answer, say "I don't have information about that."
Always cite the source document name when you use information from it."""


class NaiveRAG:
    """
    The SIMPLEST possible RAG system.
    
    This is RAG in its most basic form:
    1. Search for relevant documents
    2. Stuff them into a prompt
    3. Ask the LLM to answer
    
    Everything else (chunking strategies, re-ranking, query transformation,
    hybrid search) is an optimization ON TOP of this.
    
    🤔 Why start with "naive" RAG instead of a better version?
    Because you need to SEE the problems before you can fix them.
    Building naive RAG first and watching it fail teaches you more
    than building an "optimized" system you don't understand.
    """
    
    def __init__(self, config: Optional[RAGConfig] = None):
        self.config = config or RAGConfig()
        
        # ── Init LLM ──
        self.llm = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
        
        # ── Init embedding model ──
        self.embedder = SentenceTransformer(self.config.embedding_model)
        
        # ── Init vector DB (ChromaDB, in-memory for this demo) ──
        self.client = chromadb.Client(Settings(anonymized_telemetry=False))
        self.collection = self.client.create_collection("rag_docs")
        
        # Track documents for citation
        self._documents: List[str] = []
        self._doc_sources: List[str] = []
    
    def index_documents(self, documents: List[str], sources: Optional[List[str]] = None):
        """
        Index documents for RAG.
        
        This is the same as Phase 3 ingestion — embed + store.
        """
        self._documents = documents
        self._doc_sources = sources or [f"doc_{i}" for i in range(len(documents))]
        
        # Embed
        embeddings = self.embedder.encode(documents, normalize_embeddings=True)
        
        # Store in ChromaDB
        ids = [f"chunk_{i}" for i in range(len(documents))]
        metadatas = [{"source": src} for src in self._doc_sources]
        
        self.collection.add(
            embeddings=embeddings.tolist(),
            documents=documents,
            metadatas=metadatas,
            ids=ids,
        )
        
        print(f"Indexed {len(documents)} documents")
    
    def retrieve(self, query: str, top_k: Optional[int] = None) -> List[Dict]:
        """
        RETRIEVE phase: Find relevant documents.
        
        This is your Phase 3 search engine in action.
        """
        k = top_k or self.config.top_k
        
        # Embed query
        query_vec = self.embedder.encode([query], normalize_embeddings=True)[0]
        
        # Search
        results = self.collection.query(
            query_embeddings=[query_vec.tolist()],
            n_results=k,
        )
        
        retrieved = []
        for i in range(len(results['documents'][0])):
            retrieved.append({
                "content": results['documents'][0][i],
                "source": results['metadatas'][0][i].get("source", "unknown"),
                "score": 1 - results['distances'][0][i],  # Convert distance to similarity
            })
        
        return retrieved
    
    def augment(self, query: str, retrieved: List[Dict]) -> str:
        """
        AUGMENT phase: Build the prompt with query + context.
        
        This is where the "Augmented" in RAG happens — you take the
        user's question and ADD the retrieved documents as context.
        """
        # Build context string from retrieved documents
        context_parts = []
        for i, doc in enumerate(retrieved):
            context_parts.append(f"Document {i+1} (source: {doc['source']}):\n{doc['content']}")
        
        context = "\n\n".join(context_parts)
        
        # Build prompt
        prompt = f"""Use the following context to answer the question at the end.

Context:
{context}

Question: {query}

Instructions:
1. Answer based ONLY on the provided context
2. If the context doesn't contain the answer, say "I don't have information about that"
3. Cite the source document name when you use information from it
4. Do not use any outside knowledge

Answer:"""
        
        return prompt
    
    def generate(self, prompt: str) -> str:
        """
        GENERATE phase: Call the LLM.
        
        This is your Phase 1 LLM Gateway in action.
        """
        response = self.llm.chat.completions.create(
            model=self.config.llm_model,
            messages=[
                {"role": "system", "content": self.config.system_prompt},
                {"role": "user", "content": prompt},
            ],
            temperature=self.config.llm_temperature,
            max_tokens=self.config.llm_max_tokens,
        )
        
        return response.choices[0].message.content
    
    def answer(self, query: str) -> Dict:
        """
        End-to-end RAG: Retrieve → Augment → Generate.
        
        This is the complete pipeline.
        """
        timings = {}
        
        # Step 1: Retrieve
        t0 = time.time()
        retrieved = self.retrieve(query)
        timings["retrieve_ms"] = round((time.time() - t0) * 1000, 2)
        
        # Step 2: Augment
        t0 = time.time()
        prompt = self.augment(query, retrieved)
        timings["augment_ms"] = round((time.time() - t0) * 1000, 2)
        
        # Step 3: Generate
        t0 = time.time()
        answer_text = self.generate(prompt)
        timings["generate_ms"] = round((time.time() - t0) * 1000, 2)
        
        # Count tokens
        prompt_tokens = len(prompt.split()) * 1.3  # Rough estimate
        answer_tokens = len(answer_text.split()) * 1.3
        
        return {
            "query": query,
            "answer": answer_text,
            "retrieved_documents": retrieved,
            "timing_ms": timings,
            "total_ms": sum(timings.values()),
            "estimated_prompt_tokens": int(prompt_tokens),
            "estimated_answer_tokens": int(answer_tokens),
        }


# ── Demo: RAG on Customer Support Documents ──
if __name__ == "__main__":
    print("═══ Building Naive RAG System ═══\n")
    
    # Our "knowledge base" — customer support documents
    documents = [
        "Return Policy: Electronics can be returned within 30 days of purchase. "
        "Items must be in original packaging. Refunds are processed within 5-7 business days.",
        
        "Return Policy: Clothing and accessories can be returned within 60 days. "
        "Items must be unworn with tags attached. Swimwear and undergarments are final sale.",
        
        "Shipping Policy: Free standard shipping on orders over $50. "
        "Express shipping costs $12.99. International shipping starts at $25.",
        
        "Warranty: All electronics come with a 1-year manufacturer warranty. "
        "Extended warranty plans are available for purchase up to 30 days after the original purchase.",
        
        "Account Recovery: If you've forgotten your password, use the 'Forgot Password' link "
        "on the login page. A reset link will be sent to your registered email within 5 minutes.",
        
        "Payment Methods: We accept Visa, Mastercard, American Express, PayPal, "
        "and Apple Pay. All payments are processed securely through Stripe.",
        
        "Subscription: Monthly plans are $29/month. Annual plans are $290/year "
        "(save 2 months). Cancel anytime from your account settings.",
        
        "Gift Cards: Digital gift cards are delivered via email within 1 hour. "
        "Physical gift cards are shipped within 3-5 business days via USPS.",
    ]
    
    sources = [
        "returns_policy.md",
        "returns_clothing.md",
        "shipping_policy.md",
        "warranty_info.md",
        "account_help.md",
        "payment_methods.md",
        "subscription_plans.md",
        "gift_cards.md",
    ]
    
    rag = NaiveRAG()
    rag.index_documents(documents, sources)
    
    # ── Test Queries ──
    test_queries = [
        "What's your return policy for electronics?",
        "How long does shipping take internationally?",
        "Can I cancel my subscription anytime?",
        "Do you offer a student discount?",  # NOT in the documents!
        "Can I return a laptop after 45 days?",
        "How do I get a refund for my order?",
    ]
    
    print("═══ RAG Query Results ═══\n")
    
    for query in test_queries:
        print(f"🔍 Query: '{query}'")
        print("─" * 60)
        
        result = rag.answer(query)
        
        print(f"\n📝 Answer: {result['answer']}")
        print(f"\n📄 Retrieved Documents:")
        for i, doc in enumerate(result['retrieved_documents']):
            print(f"   [{doc['score']:.3f}] {doc['source']}")
            print(f"       {doc['content'][:80]}...")
        
        print(f"\n⏱  Timing: {result['timing_ms']}")
        print(f"   Total: {result['total_ms']:.0f}ms | "
              f"Prompt tokens: ~{result['estimated_prompt_tokens']} | "
              f"Answer tokens: ~{result['estimated_answer_tokens']}")
        print()
```

**Expected output:**
```
═══ RAG Query Results ═══

🔍 Query: 'What's your return policy for electronics?'
────────────────────────────────────────────────────────────
📝 Answer: Our return policy allows electronics to be returned within 30 days of purchase. Items must be in original packaging, and refunds are processed within 5-7 business days. (Source: returns_policy.md)

📄 Retrieved Documents:
   [0.912] returns_policy.md
       Return Policy: Electronics can be returned within 30 days of purchase...
   [0.845] warranty_info.md
       Warranty: All electronics come with a 1-year manufacturer warranty...
   [0.623] returns_clothing.md
       Return Policy: Clothing and accessories can be returned within 60 days...

✅ CORRECT: Answer matches the retrieved document perfectly.

🔍 Query: 'Do you offer a student discount?'
────────────────────────────────────────────────────────────
📝 Answer: I don't have information about that. The provided context does not mention any student discount policy.

📄 Retrieved Documents:
   [0.312] subscription_plans.md
       Subscription: Monthly plans are $29/month...
   [0.289] payment_methods.md
       Payment Methods: We accept Visa, Mastercard...
   [0.245] gift_cards.md
       Gift Cards: Digital gift cards are delivered via email within 1 hour...

✅ CORRECT: The LLM correctly refused to answer when context didn't contain the info.
(Note: This requires a GOOD system prompt. With a weaker prompt, the LLM might hallucinate.)

🔍 Query: 'Can I return a laptop after 45 days?'
────────────────────────────────────────────────────────────
📝 Answer: Unfortunately, electronics like laptops can only be returned within 30 days of purchase. Since 45 days has passed, this item would not be eligible for return under our standard return policy. You may want to check if your purchase has an extended warranty or was purchased with a credit card that offers purchase protection.

📄 Retrieved Documents:
   [0.891] returns_policy.md
       Return Policy: Electronics can be returned within 30 days...
   [0.756] warranty_info.md
       Warranty: All electronics come with a 1-year manufacturer warranty...

⚠️ INTERESTING: The correct answer should say "No, 45 days exceeds our 30-day policy."
The LLM added "check your credit card" advice — which is NOT in the context!
THIS IS A HALLUCINATION — a small one, but still not grounded in the retrieved docs.
```

---

### 🤔 Analysis: Three Immediate Problems

Your naive RAG is working... partially. Let's see what's wrong:

```
QUERY                          │ RESULT                               │ PROBLEM
───────────────────────────────┼──────────────────────────────────────┼────────────────────────
"return electronics"           │ Correct, cites source                │ ✅ Works
"student discount"             │ Correctly says "no info"            │ ✅ Works (good prompt)
"return laptop after 45 days"  │ Mostly right, added UNGROUNDED advice│ ⚠️ Small hallucination
"International shipping time"  │ ?                                    │ ❓ Let's test...
```

**Problem 1: Chunking is naive**

Right now, each document is ONE chunk. What if a document is 5,000 words?
- The embedding loses specificity (a 5,000-word blob represents too many concepts)
- The LLM gets irrelevant context (a chunk about "shipping" contains "return policy" info too)
- **Solution:** File 02 — Chunking Strategies

**Problem 2: Retrieval is fragile**

If your top-3 retrieved chunks don't contain the answer, the LLM will either:
- Hallucinate (bad)
- Say "I don't know" when it COULD have answered from a slightly different chunk
- **Solution:** File 03 — Retrieval Deep Dive (query transformation, hyde, etc.)

**Problem 3: The LLM ignores the context**

Sometimes the LLM uses its TRAINING knowledge instead of the retrieved documents, especially when:
- The training data is more detailed than your documents
- Your prompt doesn't strongly enforce grounding
- The LLM "thinks" it knows better
- **Solution:** File 04 — Context Integration (prompt engineering for RAG)

---

### Example 2: RAG with Attribution (Source Citations)

Let's fix the most glaring issue — attribution:

```python
"""
RAG with source attribution.
Showing users WHERE each part of the answer came from.
"""

import re
from typing import List, Tuple


class RAGWithCitations(NaiveRAG):
    """
    RAG that cites sources in its answers.
    
    Instead of just returning text, it:
    1. Asks the LLM to include [1], [2], etc. referencing source documents
    2. Maps those citations back to actual document names
    3. Returns structured results with source links
    
    🤔 Think about this: How would you verify that a citation is REAL?
    The LLM might say "[1]" even if doc 1 doesn't contain that information.
    This is called "citation hallucination" — and it's a known problem.
    """
    
    def augment(self, query: str, retrieved: List[Dict]) -> str:
        """Build prompt with numbered sources for citation."""
        context_parts = []
        for i, doc in enumerate(retrieved):
            context_parts.append(
                f"[{i+1}] (source: {doc['source']}):\n{doc['content']}"
            )
        
        context = "\n\n".join(context_parts)
        
        prompt = f"""Use the following context to answer the question.

Context:
{context}

Question: {query}

Instructions:
1. Answer based ONLY on the provided context
2. If the context doesn't contain the answer, say "I don't have information about that"
3. CRITICAL: Cite your sources using [1], [2], etc. after each statement
4. Example: "Our return policy allows 30 days for electronics [1]."
5. Only cite a source if that source actually contains the information

Answer:"""
        
        return prompt
    
    def answer(self, query: str) -> Dict:
        """Get answer with parsed citations."""
        result = super().answer(query)
        
        # Parse citations from the answer
        citations = self._parse_citations(
            result["answer"],
            result["retrieved_documents"],
        )
        
        result["citations"] = citations
        return result
    
    def _parse_citations(self, answer: str, documents: List[Dict]) -> List[Dict]:
        """
        Extract and verify citations from the answer.
        
        This is a SIMPLE parser. Production systems use more sophisticated
        approaches (or ask the LLM to output structured citations as JSON).
        """
        # Find all [N] patterns
        citation_refs = re.findall(r'\[(\d+)\]', answer)
        
        verified_citations = []
        for ref in citation_refs:
            idx = int(ref) - 1  # Convert to 0-indexed
            if 0 <= idx < len(documents):
                verified_citations.append({
                    "reference": f"[{ref}]",
                    "source": documents[idx]["source"],
                    "content_preview": documents[idx]["content"][:100],
                })
            else:
                # Citation to a non-existent document — hallucination!
                verified_citations.append({
                    "reference": f"[{ref}]",
                    "source": "UNKNOWN — citation hallucination!",
                    "content_preview": "This source doesn't exist in the retrieved documents.",
                })
        
        return verified_citations


# ── Demo ──
if __name__ == "__main__":
    rag = RAGWithCitations()
    rag.index_documents(documents, sources)
    
    query = "What's the return policy for electronics and what payment methods do you accept?"
    result = rag.answer(query)
    
    print(f"Query: {query}\n")
    print(f"Answer: {result['answer']}\n")
    
    print("📋 Verified Citations:")
    for cit in result.get("citations", []):
        status = "✅" if "UNKNOWN" not in cit["source"] else "❌"
        print(f"  {status} {cit['reference']} → {cit['source']}")
        print(f"     {cit['content_preview']}...")
```

**Expected output:**
```
📋 Verified Citations:
  ✅ [1] → returns_policy.md
     Return Policy: Electronics can be returned within 30 days of purchase...
  ✅ [2] → payment_methods.md
     Payment Methods: We accept Visa, Mastercard, American Express...

Now you can SHOW citations next to the answer — like a search engine shows snippets.
This is the minimum bar for a production RAG system.
```

---

## ✅ Good Output Examples

### What Good Naive RAG Output Looks Like

```
Query: "What's your return policy for electronics?"

Answer: Our return policy allows electronics to be returned within 30 days of 
purchase [1]. Items must be in original packaging [1]. Refunds are processed 
within 5-7 business days after we receive the item [1].

Citations:
  [1] returns_policy.md

Retrieved: 3 docs (scores: 0.91, 0.78, 0.45)
LLM: gpt-4o-mini (temp=0.0)
Latency: 1.2s total

✓ Correct answer
✓ Cites sources
✓ Refuses to answer if context doesn't have info
✓ Low temperature (deterministic)
```

### What Bad Naive RAG Output Looks Like

```
Query: "Do you offer a student discount?"

Answer: Yes! We offer a 15% student discount with a valid .edu email address. 
Simply use code STUDENT15 at checkout to save on your first purchase!

Citations: (none — hallucinated from training data)

Retrieved: 3 docs (scores: 0.31, 0.28, 0.24 — all LOW)
LLM: gpt-4o-mini (temp=0.7 — too high!)

✗ Hallucination — no document mentions student discount
✗ Low retrieval scores should have triggered a "don't know" response
✗ High temperature made the LLM "creative"
✗ No citations to verify

ROOT CAUSE: The system prompt didn't enforce "only use context" strongly enough.
The LLM fell back to its training data knowledge.
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: High Temperature for RAG

```python
# ❌ WRONG — RAG should be deterministic
llm = OpenAI(temperature=0.7)
# The LLM will be "creative" — rewriting, adding, inventing
# Every query returns DIFFERENT answers for the same context

# ✅ RIGHT — RAG should be factual
llm = OpenAI(temperature=0.0)  # or very close to 0
# The LLM sticks to the facts in the context
# Same query + same context = same answer every time
```

### Antipattern 2: No "Don't Know" Training

```python
# ❌ WRONG — System prompt doesn't say what to do when info is missing
system_prompt = "You are a helpful assistant. Answer the user's question."
# LLM will ALWAYS try to answer, even without context → hallucination

# ✅ RIGHT — Explicit instruction for missing information
system_prompt = """You are a helpful assistant. Answer based ONLY on the provided context.
If the context doesn't contain the answer, say 'I don't have information about that.'
Do not use any outside knowledge."""
# LLM has permission to say "I don't know"
```

### Antipattern 3: Retrieving Too Many Chunks

```python
# ❌ WRONG — Retrieving 10 chunks for every query
retrieved = search(query, top_k=10)
# Problem: 8 irrelevant chunks dilute the 2 good ones
# The LLM gets confused by conflicting/unrelated information
# Cost: 10 chunks = more tokens = higher latency + higher cost

# ✅ RIGHT — Start with 3-5, increase only if needed
retrieved = search(query, top_k=3)
# Fewer, more relevant chunks = clearer signal
# Lower cost, lower latency
```

### Antipattern 4: Not Logging Retrieval Scores

```python
# ❌ WRONG — No visibility into retrieval quality
response = rag.answer(query)
return response["answer"]
# You have NO IDEA if the retrieval was good or bad

# ✅ RIGHT — Log retrieval scores
response = rag.answer(query)
log_retrieval_quality(query, response["retrieved_documents"])
# Track: average score, minimum score, score distribution
# Alert if scores drop below threshold — means your index is stale
```

### Failure Mode: The "Lost in the Middle" Problem

```
RAG-specific failure: When you retrieve 10 chunks, the LLM pays
MOST attention to the FIRST and LAST chunks, and IGNORES the middle ones.

From "Lost in the Middle" (Liu et al., 2023):
- Chunk at position 1: 85% accuracy (used correctly)
- Chunk at position 5: 45% accuracy (often missed)
- Chunk at position 10: 80% accuracy (recency bias)

FIX: Put the MOST relevant chunk at the START or END of the context,
not in the middle. Re-rank chunks by relevance before building the prompt.
```

---

## 🧪 Drills & Challenges

### Drill 1: Break Your RAG (20 min)

Run the naive RAG code above with these queries. For each, document:
1. Is the answer correct?
2. Does it cite sources properly?
3. Is there any hallucination?

Then find 3 NEW queries where naive RAG FAILS. The failure must be:
- A hallucination (LLM makes something up)
- A contradiction (LLM says something that conflicts with the retrieved docs)
- A miss (LLM says "don't know" when the answer IS in the docs)

**Write down each failure and WHY it happened.**

### Drill 2: The Prompt Tuning Experiment (15 min)

Test 5 different system prompts for RAG with the SAME query and documents:

```python
prompts = [
    "Answer the question.",
    "Answer based on the context provided.",
    "Answer based ONLY on the following context. If the context doesn't have the answer, say you don't know.",
    "You are a helpful assistant. Use the context to answer. Cite sources with [1], [2]. If unsure, say 'I don't know.'",
    "CONTEXT: {context}\n\nQUESTION: {query}\n\nINSTRUCTIONS: ...",
]
```

Which prompt gives the BEST results? The most factual? The most helpful?

### Drill 3: The Attribution Challenge (20 min)

Build a RAG system that returns structured attribution:

```json
{
    "answer": "Our policy allows 30-day returns for electronics.",
    "claims": [
        {
            "text": "30-day returns for electronics",
            "source": "returns_policy.md",
            "exact_match": true
        }
    ]
}
```

**Challenge:** The LLM might rephrase. "30-day returns" might be written as "returns within 30 days" in the source. How do you verify attribution when the wording differs?

### Drill 4: The Cost Calculator (15 min)

Calculate the cost of running naive RAG at scale:

```
Documents: 10,000 chunks
Queries/day: 1,000
LLM: gpt-4o-mini
Embedding model: all-MiniLM-L6-v2 (free, local)

Calculate per-query:
1. Tokens in retrieved chunks (3 chunks × 500 chars ≈ how many tokens?)
2. Cost of LLM call (gpt-4o-mini: $0.15/1M input tokens, $0.60/1M output tokens)
3. Cost per day
4. Cost per month
5. Cost per year

Now calculate with gpt-4o instead ($2.50/1M input, $10/1M output)
How much more expensive? Is the quality worth it?
```

### Drill 5: The "RAG or No RAG" Test (15 min)

Compare LLM answers WITH and WITHOUT retrieval for the SAME question:

```python
# WITHOUT RAG (pure LLM knowledge)
no_context_answer = llm.answer("What's your return policy?")

# WITH RAG (retrieved documents)
rag_answer = rag.answer("What's your return policy?")

# Compare:
# 1. Which is more accurate?
# 2. Which is more detailed?
# 3. Which has fewer hallucinations?
# 4. Which would you trust more for customer support?
```

**Expected finding:** For GENERAL knowledge, the LLM might do fine without RAG. For COMPANY-SPECIFIC knowledge, RAG is essential. This tells you WHEN to use RAG vs pure LLM.

---

## 🚦 Gate Check

Before moving to File 02 (Chunking Strategies), confirm you can:

- [ ] **Explain the RAG pipeline** in 2 sentences: Retrieve → Augment → Generate
- [ ] **Build naive RAG from scratch** connecting a vector search to an LLM
- [ ] **Identify 3 failure modes** of naive RAG (hallucination, missed retrieval, poor citation)
- [ ] **Know why temperature=0** is critical for RAG
- [ ] **Implement source citations** in your RAG output
- [ ] **Answer these questions:**
  1. If your RAG system always says "I don't know," even for questions that ARE in the docs — what's broken? (Retrieval? Prompt? Both?)
  2. If your RAG system hallucinates despite having correct documents in context — what's wrong? (Prompt? Temperature? Model?)
  3. You retrieve 10 chunks. The answer is in chunk #7 (middle). The LLM misses it. Why?
  4. Should you use the SAME embedding model for retrieval that you use for indexing? Why or why not?
  5. Your RAG costs $0.02/query. You have 100K queries/month. How do you reduce cost?

- [ ] **Write down the #1 thing** you want to fix about your naive RAG before going to production

---

## 📚 Resources

**Foundational:**
- "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" (Lewis et al., 2020) — The original RAG paper
- "Lost in the Middle: How Language Models Use Long Contexts" (Liu et al., 2023) — Why chunk position matters
- "RAG vs Fine-Tuning" — When to use each (spoilertools: RAG wins for most use cases)

**Code References:**
- LangChain RAG documentation
- LlamaIndex RAG documentation
- OpenAI Cookbook: RAG patterns

**Production Reading:**
- "Building RAG at Scale: Lessons from Production" — Various engineering blogs
- "The 7 Failure Modes of RAG" — Barns' blog (we'll cover in File 05)

**Your RAG Progress Checklist:**
```
☐ Naive RAG works (this file)
☐ Chunking strategy chosen (File 02)
☐ Retrieval optimized (File 03)
☐ Context integration done (File 04)
☐ Failure modes documented (File 05)
☐ Evaluation pipeline running (File 06)
☐ Production-ready (File 07)
☐ Drills completed (File 08)
☐ Project shipped (File 09)
```

---

> **🛑 Revisit the 3 discovery questions from the beginning:**
>
> 1. **Before You Read the Definition** — How close was your mental model to the actual RAG pipeline?
> 2. **The "Just Stuff It" Approach** — Did you see why prompt engineering (Phase 2) is critical for RAG?
> 3. **The Attribution Problem** — You've now seen that citations can be hallucinated too. How will you verify them?
>
> **Key realization:** RAG is conceptually simple — retrieve docs, stuff them in a prompt, generate answer. But EVERY step has failure modes. The rest of this phase is about fixing those failures.
>
> **Next:** File 02 — Chunking Strategies. How you split documents determines EVERYTHING about retrieval quality.
