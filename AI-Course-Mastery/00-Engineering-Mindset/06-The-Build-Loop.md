# The Build Loop

## 🎯 Purpose & Goals

Every phase of this course follows the same engineering cycle: **PROBLEM → BUILD RAW → ADD FRAMEWORKS → PRODUCTIONIZE → EVALUATE → DEBUG**. This isn't arbitrary — it's the pattern used by every senior AI engineer building production systems.

**By the end of this file, you will:**
- Understand the 6-step build loop that structures this entire course
- Know why building "from scratch first" is essential before using frameworks
- Have a systematic approach to debugging AI failures
- Be ready to apply this loop starting in Phase 1

**⏱ Time Budget:** 1 hour

---

## 📖 The 6-Step Build Loop

```
┌─────────────────────────────────────────────────────────────┐
│                     THE BUILD LOOP                           │
│                                                              │
│  PROBLEM → BUILD RAW → ADD FRAMEWORKS → PRODUCTIONIZE →     │
│                                     ↓                        │
│                              EVALUATE ← → DEBUG             │
│                                                              │
│  (Loop back to BUILD RAW or ADD FRAMEWORKS as needed)       │
└─────────────────────────────────────────────────────────────┘
```

### Step 1: PROBLEM — "What are we actually solving?"

Before writing ANY code, understand the problem deeply:

```python
# Starter questions for EVERY module:
problem_checklist = """
☐ What's the concrete goal? (e.g., "Answer customer questions from docs")
☐ What's the success metric? (e.g., faithfulness > 0.9)
☐ What's the simplest possible solution? (e.g., search + LLM)
☐ What are the known failure modes? (e.g., model hallucinates, wrong docs retrieved)
☐ What constraints exist? (latency, cost, privacy, scale)
"""
```

### Step 2: BUILD RAW — "Implement without frameworks"

Build the simplest possible version with NO frameworks. Just APIs and standard libraries:

```python
# Example: Building RAG "from scratch" before using LangChain

class RawRAG:
    """
    No LangChain. No LlamaIndex. Just OpenAI API + standard Python.
    
    Why build raw first?
    - You understand EVERY line of code
    - You know what the framework is abstracting away
    - You can debug when things go wrong
    - You appreciate why frameworks exist
    """
    
    def __init__(self, openai_api_key: str):
        self.client = OpenAI(api_key=openai_api_key)
        self.documents = []
        self.embeddings = []
    
    def add_document(self, text: str):
        """Raw embedding — no vector DB yet"""
        response = self.client.embeddings.create(
            model="text-embedding-3-small",
            input=text
        )
        self.documents.append(text)
        self.embeddings.append(response.data[0].embedding)
    
    def retrieve(self, query: str, top_k: int = 3) -> list[str]:
        """Raw similarity search — no vector DB"""
        # Embed query
        q_emb = self.client.embeddings.create(
            model="text-embedding-3-small",
            input=query
        ).data[0].embedding
        
        # Linear scan (naive, but you understand it)
        import numpy as np
        scores = []
        for doc_emb in self.embeddings:
            sim = np.dot(q_emb, doc_emb) / (np.linalg.norm(q_emb) * np.linalg.norm(doc_emb))
            scores.append(sim)
        
        # Get top_k
        top_indices = np.argsort(scores)[-top_k:][::-1]
        return [self.documents[i] for i in top_indices]
    
    def generate(self, query: str, context: list[str]) -> str:
        """Raw completion — no LangChain chains"""
        context_text = "\n\n".join(context)
        response = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "Answer using the context. Say 'I don't know' if unsure."},
                {"role": "user", "content": f"Context:\n{context_text}\n\nQuestion:\n{query}"}
            ]
        )
        return response.choices[0].message.content
    
    def query(self, question: str) -> str:
        """End-to-end — you understand every layer"""
        relevant_docs = self.retrieve(question)
        answer = self.generate(question, relevant_docs)
        return answer
```

### Step 3: ADD FRAMEWORKS — "Now introduce abstractions"

Once you understand the raw implementation, introduce frameworks to handle complexity:

```python
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.chains import RetrievalQA

class FrameworkRAG:
    """
    Same RAG system, now using LangChain.
    You understand what LangChain is doing because you built it raw first.
    - ChromaDB handles vector storage + HNSW indexing
    - LangChain handles prompt templating + chain orchestration
    - You can now focus on higher-level concerns
    
    Senior note: The framework FIXES the pain you felt in the raw version.
    You no longer write linear search. You no longer manage embeddings manually.
    """
    
    def __init__(self):
        self.embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
        self.llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
        self.vectorstore = Chroma(
            embedding_function=self.embeddings,
            persist_directory="./chroma_db"
        )
    
    def add_documents(self, documents: list[str]):
        text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=1000, chunk_overlap=200
        )
        chunks = text_splitter.create_documents(documents)
        self.vectorstore.add_documents(chunks)
    
    def query(self, question: str) -> str:
        qa_chain = RetrievalQA.from_chain_type(
            llm=self.llm,
            chain_type="stuff",
            retriever=self.vectorstore.as_retriever(search_kwargs={"k": 3})
        )
        return qa_chain.invoke(question)
```

### Step 4: PRODUCTIONIZE — "Make it reliable"

Add everything from the Production vs Academic Thinking module:

```python
class ProductionRAG:
    """
    The production version adds:
    - Error handling & retries
    - Observability & tracing
    - Cost tracking & budgets
    - Guardrails & validation
    - Caching & performance
    - Graceful degradation
    """
    
    # ... (see Production vs Academic Thinking for full implementation)
```

### Step 5: EVALUATE — "Measure quality systematically"

This is NOT an afterthought — it's how you know your system works:

```python
class BuildLoopEval:
    """
    After every iteration of the build loop, run the eval suite.
    Compare results against the previous version.
    """
    
    def check_progression(self, current_scores: dict, baseline_scores: dict, threshold: float = 0.05):
        """Did we actually improve?"""
        regressions = []
        improvements = []
        
        for metric in current_scores:
            diff = current_scores[metric] - baseline_scores.get(metric, 0)
            if diff < -threshold:
                regressions.append(f"{metric}: {diff:.2f}")
            elif diff > threshold:
                improvements.append(f"{metric}: +{diff:.2f}")
        
        if regressions:
            return {"status": "REGRESSION", "details": regressions}
        elif improvements:
            return {"status": "IMPROVED", "details": improvements}
        else:
            return {"status": "NO_CHANGE", "details": []}
```

### Step 6: DEBUG — "Fix what's broken"

When evaluation shows problems, use systematic debugging:

```python
class AIDebugger:
    """
    Systematic approach to debugging AI system failures.
    Don't guess — follow the evidence.
    """
    
    def debug_rag_failure(self, question: str, answer: str, context: list[str]):
        """Find out WHY a RAG answer was wrong"""
        
        checks = []
        
        # Check 1: Was the right context retrieved?
        context_quality = self.check_retrieval_quality(question, context)
        checks.append(("retrieval", context_quality))
        
        # Check 2: Did the model use the context?
        faithfulness = self.check_faithfulness(context, answer)
        checks.append(("faithfulness", faithfulness))
        
        # Check 3: Was the prompt correct?
        instruction_adherence = self.check_instructions(answer)
        checks.append(("instructions", instruction_adherence))
        
        # Check 4: Is the model capable of this task?
        model_capability = self.check_model_capability(question)
        checks.append(("model_capability", model_capability))
        
        return {
            "failure_analysis": checks,
            "root_cause": self.identify_root_cause(checks),
            "fix_suggestion": self.suggest_fix(checks),
        }
    
    def check_retrieval_quality(self, question: str, context: list[str]) -> dict:
        """Use LLM to judge if retrieved docs are relevant"""
        # (Implementation in Phase 4 RAG module)
        pass
    
    def check_faithfulness(self, context: list[str], answer: str) -> float:
        """Does the answer stick to the context?"""
        # (Implementation in Phase 7 Evals module)
        pass
    
    def identify_root_cause(self, checks: list) -> str:
        """Find the FIRST thing that went wrong in the pipeline"""
        for check_type, result in checks:
            if not result["passed"]:
                return f"Pipeline failure at: {check_type} — {result['reason']}"
        return "Unknown — need deeper investigation"
```

---

## 📖 The Loop in Practice — Phase 1 Example

Here's how the build loop applies to Phase 1 (Foundations & LLM Gateway):

```
PROBLEM: "I need an API that can use any LLM provider, handle streaming,
          manage errors, track costs, and be easy to switch providers."

BUILD RAW: 
  - Write direct OpenAI API calls (synchronous)
  - Add async support
  - Add streaming
  - ✓ Understand the underlying API patterns

ADD FRAMEWORKS:
  - Introduce LiteLLM for provider abstraction
  - Introduce FastAPI for the API layer
  - ✓ Let the framework handle cross-cutting concerns

PRODUCTIONIZE:
  - Add error handling with retries
  - Add rate limiting
  - Add cost tracking
  - Add request validation

EVALUATE:
  - Test with all 3 providers
  - Measure latency (p50, p95, p99)
  - Calculate cost per query
  - Test error scenarios (bad key, timeout, rate limit)

DEBUG:
  - Fix any failures found in evaluation
  - Optimize slow paths
  - Handle edge cases discovered
```

---

## 🧪 Drill: Apply the Build Loop

**Time: 15 minutes**

**Task:** Pick ONE feature you'd build (e.g., "AI-powered search for my notes"). Write down what each step of the build loop would look like:

1. What's the PROBLEM statement?
2. What's the RAW implementation (no frameworks)?
3. What FRAMEWORK would you add and why?
4. What PRODUCTIONIZATION steps are needed?
5. What EVALS would you run?
6. What's a likely DEBUG scenario?

<details>
<summary>Sample: AI Note Search</summary>

1. **PROBLEM**: Search 500 personal notes by meaning, not just keywords. Need to find "ideas about startup fundraising" even when the word "fundraising" isn't in the note.

2. **BUILD RAW**: 
   - Embed all notes with OpenAI API
   - Store in list + numpy array
   - Linear scan cosine similarity
   - Return top 5 results
   
3. **ADD FRAMEWORKS**:
   - ChromaDB for persistent vector storage + HNSW indexing (faster search)
   - Why: Raw approach won't scale past 10K notes. Chroma handles indexing.
   
4. **PRODUCTIONIZE**:
   - Async embedding for batch operations
   - Watch folder for new notes (auto-embed)
   - Redis cache for frequent queries
   - Error handling for corrupt files
   
5. **EVALS**:
   - 20 test queries with known expected results
   - Recall@k (is the right note in top 5?)
   - Query latency (target: < 500ms)
   
6. **DEBUG**:
   - Scenario: Search "Python async code" returns notes about "Python async" but misses notes about "coroutines"
   - Root cause: Embedding doesn't capture synonym relationships well
   - Fix: Use HyDE — generate hypothetical note first, then search
</details>

---

## 🚦 Gate Check

- [ ] Can you name all 6 steps of the build loop from memory?
- [ ] Can you explain why "build raw first" matters?
- [ ] Can you apply the loop to a new problem statement?
- [ ] Do you know where evaluation fits in the loop?

**Proceed to Phase 1 when YES.**
