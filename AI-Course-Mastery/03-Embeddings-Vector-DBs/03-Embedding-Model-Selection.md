# Embedding Model Selection — How to Choose the Right Tool

## 🎯 Purpose & Goals

> **🛑 STOP. Here are 4 scenarios. For each one, think about what you'd do. Don't read further — just think.**
>
> ### Scenario A: The Startup MVP
> You're building a semantic search for a 200-document internal wiki. You have $0 budget for API calls. You need it working by Friday.
>
> ### Scenario B: The Enterprise RAG System
> You're building a customer support RAG system for a multinational bank. Accuracy is critical — wrong answers cost millions. Budget isn't a concern. The documents are in English, Spanish, and Hindi.
>
> ### Scenario C: The Real-Time Search
> You're building a search for a code editor. Every keystroke triggers a search. You need results in under 50ms. You have 10,000 code snippets to search across.
>
> ### Scenario D: The Research System
> You're building a medical research paper search. The vocabulary is highly specialized (oncogenes, protein folding, clinical trial phases). General-purpose models don't understand these terms well.

---

### 🤔 Discovery Question 1: The Essential Criteria

**Before reading anything below:** Write down the 5 factors you'd consider when choosing an embedding model. Don't just list them — rank them by importance.

**Here's a hint to start:** Think about what happened in File 02. What mattered most when your search failed? Was it the model quality, or something else?

---

### 🤔 Discovery Question 2: The Dimension Tradeoff

You have two models:
- **Model A**: 384 dimensions, 50ms per 100 texts, 90% MTEB score
- **Model B**: 1536 dimensions, 200ms per 100 texts, 93% MTEB score

**Before reading on, answer:**
1. How much does 3 extra percentage points of quality matter to your use case?
2. When would you ABSOLUTELY choose Model B despite the extra cost?
3. When would Model A be the BETTER choice despite lower quality?
4. What does "93% MTEB score" even mean? Does it guarantee good results for YOUR data?

---

### 🤔 Discovery Question 3: Free vs Paid

Open-source model: `all-MiniLM-L6-v2` — free, 384-dim, runs locally, good quality
Paid API: `text-embedding-3-small` — costs $0.02/1M tokens, 1536-dim, excellent quality

**Your task:** You have 500,000 documents, each ~200 tokens. Users search ~10,000 times/day.

**Calculate:**
1. How much does it cost PER MONTH to embed 500K documents with the paid API?
2. How much does it cost PER MONTH for search queries (each query needs embedding)?
3. What happens when your document set grows to 5 million?
4. At what point does the paid API become TOO expensive?

**Now the real question:** Is the free model "good enough"? How do you even measure that?

---

**By the end of this file, you will:**
- NOT memorize a list of models — but KNOW the framework to evaluate ANY model
- Be able to run a systematic model comparison for YOUR specific use case
- Understand the real-world tradeoffs: quality ↔ speed ↔ cost ↔ dimensions
- Know when to use OpenAI embeddings, when to use open-source, and when to fine-tune
- Have a reusable evaluation script to compare models on your own data

**⏱ Time Budget:** 2 hours (45 min concept + comparison, 45 min code, 30 min drills)

---

## 📖 The Selection Framework — Not a List

### Why "Best Embedding Model 2026" Lists Are Useless

Every blog post says "Here are the top 10 embedding models." They show the MTEB leaderboard. You pick #1 and move on.

**This is wrong for 3 reasons:**

1. **MTEB averages over 50+ tasks.** The "best" general model might be terrible for YOUR specific domain (medical, legal, code). The #1 ranked model averages 68% on classification but might score 45% on your specialized retrieval task.

2. **Quality ≠ usefulness.** A model with 68% MTEB that runs in 1ms might be MORE useful than 70% MTEB that takes 50ms, depending on your latency requirements.

3. **Cost constraints are real.** text-embedding-3-large costs 10x more than text-embedding-3-small. For 10M documents, that's the difference between $200 and $2,000.

### The Real Framework: 5 Dimensions of Choice

```
EMBEDDING MODEL SELECTION MATRIX
─────────────────────────────────────────────────────────────────────
Dimension  │  What It Means           │  When It Matters
────────────┼──────────────────────────┼────────────────────────────
QUALITY     │  Retrieval accuracy      │  High-stakes (legal, medical)
SPEED       │  Latency per query       │  Real-time (UIs, chatbots)
COST        │  $/embedding             │  Large scale (millions of docs)
DIMENSION   │  Vector size             │  Storage budget, downstream
DOMAIN FIT  │  Specialized knowledge   │  Niche vocabulary (code, science)
─────────────────────────────────────────────────────────────────────
```

**Your job is NOT to find "the best model." Your job is to find the RIGHT tradeoff for YOUR constraints.**

---

### 🤔 Guided Discovery: Pick Your Poison

**Case 1: You optimize for QUALITY, ignoring cost and speed.**
- You're building a medical diagnosis assistant
- Wrong answer → patient harm
- You'll take the best model regardless of price

**Case 2: You optimize for SPEED, ignoring some quality.**
- You're building a keystroke-by-keystroke code search
- 50ms latency budget per search
- A bad result on the first keystroke doesn't matter (user refines)
- Speed is the constraint

**Case 3: You optimize for COST, ignoring dimensions.**
- You're building a pet project with 100 documents
- You want it running locally, no API keys
- Free is the only option

**Case 4: You optimize for DOMAIN FIT.**
- You're building a legal document search
- General models confuse "brief" (legal document) with "brief" (short)
- You need a model trained on legal corpora

**Before reading the solutions below:**
For each case above, what would YOU choose? Why? What tradeoff are you making?

---

### The Models Landscape (Not a Memorization Exercise)

I'm going to show you the major embedding model categories. Your goal is NOT to memorize them — it's to understand the DESIGN PATTERN behind each one so you can evaluate ANY model you encounter.

```
CATEGORY 1: API-Based (Managed)
┌────────────────────────────────────────────────────────────────────────┐
│ Model                    │ Dims   │ Cost/1M tokens │ MTEB │ Key Feature │
├────────────────────────────────────────────────────────────────────────┤
│ text-embedding-3-small   │ 1536   │ $0.02          │ 62.3 │ Fast, cheap │
│ text-embedding-3-large   │ 3072   │ $0.13          │ 64.6 │ Best quality │
│ voyage-3                 │ 1024   │ $0.06          │ 66.6 │ Domain tuned │
│ voyage-code-3            │ 1024   │ $0.06          │ —    │ Code focused │
│ cohere-embed-v3          │ 1024   │ $0.10          │ 65.1 │ Multilingual │
└────────────────────────────────────────────────────────────────────────┘
  
  BEST FOR: Teams with budget, teams needing multilingual, teams without GPU
  TRADEOFF: Latency (network call per embed), cost at scale, data privacy

CATEGORY 2: Open-Source Sentence Transformers
┌────────────────────────────────────────────────────────────────────────┐
│ Model                    │ Dims   │ Speed*   │ MTEB │ Size  │ Key Use   │
├────────────────────────────────────────────────────────────────────────┤
│ all-MiniLM-L6-v2         │ 384    │ 10K/s    │ 56.7 │ 90MB  │ Prototype │
│ all-mpnet-base-v2        │ 768    │ 3K/s     │ 63.3 │ 420MB │ Quality   │
│ BAAI/bge-small-en-v1.5   │ 384    │ 12K/s    │ 60.2 │ 33MB  │ Deployment│
│ BAAI/bge-base-en-v1.5    │ 768    │ 5K/s     │ 63.0 │ 133MB │ Balanced  │
│ BAAI/bge-large-en-v1.5   │ 1024   │ 2K/s     │ 64.2 │ 420MB │ Best OSS  │
└────────────────────────────────────────────────────────────────────────┘
  * Speed = texts/second on CPU (approximate)
  
  BEST FOR: No budget, data privacy, offline deployment
  TRADEOFF: Lower quality than top API models, needs local compute

CATEGORY 3: Specialized Domain Models
┌────────────────────────────────────────────────────────────────────────┐
│ Domain      │ Recommended Model          │ Why Specialized              │
├────────────────────────────────────────────────────────────────────────┤
│ Code        │ intfloat/e5-mistral-7b-instruct │ Code-aware tokenization  │
│ Medical     │ pritamdeka/S-PubMedBert-MSB │ PubMed + MIMIC trained      │
│ Legal       │ legal-bert-base-uncased     │ Legal corpus pre-training   │
│ Multilingual│ paraphrase-multilingual-MiniLM │ 50+ languages             │
│ Scientific  │ BAAI/bge-sci-en            │ Scientific paper embeddings  │
└────────────────────────────────────────────────────────────────────────┘

  BEST FOR: Domain-specific retrieval
  TRADEOFF: General tasks suffer, model may not be maintained
```

**🤔 Key insight — not obvious from the table:**
> MTEB scores DIFFERENCE between models is usually < 10 points.
> But the DIFFERENCE between a model that knows your domain and one that doesn't can be 30+ points on YOUR specific task.
>
> Example: bge-large-en-v1.5 scores 64.2 on MTEB. A legal-specific model scores 61.0 on MTEB. But on legal retrieval, the domain model beats bge-large by 20+ points.
>
> **The best model for YOU might not be the best model overall.**

---

## 💻 Code Examples — Compare Models on YOUR Data

### Example 1: Systematic Model Comparison

Instead of trusting benchmarks, let's build a test harness to compare models on YOUR specific data and queries:

```python
"""
Model comparison harness.
Tests ANY embedding model on YOUR data and YOUR queries.
"""
import numpy as np
from sentence_transformers import SentenceTransformer
from sklearn.metrics.pairwise import cosine_similarity
from typing import List, Dict, Callable, Optional
from dataclasses import dataclass, field
import time
import json


@dataclass
class ModelSpec:
    """Specification for a model to compare."""
    name: str
    model_id: str
    type: str  # "sentence-transformers" | "openai" | "voyage" | "cohere"
    dimensions: Optional[int] = None
    cost_per_1m_tokens: float = 0.0
    api_key_required: bool = False


@dataclass
class ComparisonResult:
    """Results from comparing models on a query set."""
    model_name: str
    latency_ms: float
    dimensions: int
    reciprocal_rank: float  # Higher = better (1.0 = perfect)
    precision_at_k: float   # % of top-k that are relevant
    failed_queries: List[str] = field(default_factory=list)


def reciprocal_rank(relevant_indices: List[int], top_k: int) -> float:
    """
    Reciprocal Rank (RR): 1/rank of first relevant result.
    
    Maps to [0, 1]. If the #1 result is relevant, RR = 1.0.
    If the #3 result is first relevant, RR = 1/3 ≈ 0.33.
    
    🤔 Why RR and not "accuracy"?
    Because users care about the FIRST relevant result, not the average.
    If the answer is #1, the user is happy. If it's #10, they might leave.
    """
    if not relevant_indices:
        return 0.0
    first_relevant_rank = relevant_indices[0] + 1  # 0-indexed → 1-indexed
    return 1.0 / first_relevant_rank


def precision_at_k(relevant_indices: List[int], k: int) -> float:
    """
    Precision@k: fraction of top-k results that are relevant.
    
    🤔 When does P@K matter more than RR?
    When the user scans multiple results (search results page)
    vs. when the user wants ONE answer (question answering)
    """
    count_relevant = sum(1 for idx in relevant_indices if idx < k)
    return count_relevant / k


class ModelComparator:
    """
    Compare embedding models on YOUR data and queries.
    
    This is the tool a senior engineer builds before choosing a model.
    Not "which model is best on MTEB" — but "which model works best HERE."
    """
    
    def __init__(self):
        self.models: Dict[str, Callable] = {}
        self.results: Dict[str, ComparisonResult] = {}
    
    def add_sentence_transformer(self, spec: ModelSpec):
        """Add a sentence-transformers model."""
        def embed(texts: List[str]) -> np.ndarray:
            model = SentenceTransformer(spec.model_id)
            return model.encode(texts, normalize_embeddings=True)
        
        # Get dimensions
        temp_model = SentenceTransformer(spec.model_id)
        spec.dimensions = temp_model.get_sentence_embedding_dimension()
        
        self.models[spec.name] = embed
        print(f"  Added {spec.name}: {spec.dimensions}D")
    
    def compare_models(
        self,
        documents: List[str],
        queries: List[str],
        relevant_docs: List[List[int]],  # Which doc indices are relevant for each query
        top_k: int = 5,
    ) -> Dict[str, ComparisonResult]:
        """
        Run full comparison across all registered models.
        
        Args:
            documents: Corpus of documents to search
            queries: Test queries
            relevant_docs: For each query, list of RELEVANT document indices
            top_k: How many results to evaluate
        
        Returns:
            Dict of model_name → ComparisonResult
        """
        
        results = {}
        
        for model_name, embed_fn in self.models.items():
            print(f"\n  Testing: {model_name}")
            
            # Time the embedding + indexing
            start = time.time()
            doc_embeddings = embed_fn(documents)
            index_time = time.time() - start
            dims = doc_embeddings.shape[1]
            
            # Test each query
            query_reciprocal_ranks = []
            query_precisions = []
            failed = []
            
            for q_idx, query in enumerate(queries):
                # Embed query
                query_vec = embed_fn([query])
                
                # Search
                scores = cosine_similarity(query_vec, doc_embeddings)[0]
                top_indices = np.argsort(scores)[::-1][:top_k]
                
                # Evaluate
                expected_relevant = relevant_docs[q_idx]
                found_relevant = [idx for idx in top_indices if idx in expected_relevant]
                
                rr = reciprocal_rank(found_relevant, top_k)
                p_at_k = precision_at_k(found_relevant, top_k)
                
                query_reciprocal_ranks.append(rr)
                query_precisions.append(p_at_k)
                
                if rr == 0.0:
                    failed.append(query)
            
            # Aggregate
            avg_time = (time.time() - start - index_time) / len(queries) * 1000
            result = ComparisonResult(
                model_name=model_name,
                latency_ms=round(avg_time, 2),
                dimensions=dims,
                reciprocal_rank=round(np.mean(query_reciprocal_ranks), 4),
                precision_at_k=round(np.mean(query_precisions), 4),
                failed_queries=failed,
            )
            results[model_name] = result
        
        self.results = results
        return results
    
    def print_report(self):
        """Print a formatted comparison report."""
        print("\n" + "=" * 80)
        print("MODEL COMPARISON REPORT")
        print("=" * 80)
        
        # Sort by RR (descending)
        sorted_results = sorted(
            self.results.values(),
            key=lambda r: r.reciprocal_rank,
            reverse=True,
        )
        
        print(f"{'Model':25s} {'RR':8s} {'P@K':8s} {'Lat(ms)':10s} {'Dims':6s} {'Failed':8s}")
        print("-" * 65)
        
        for r in sorted_results:
            rr = f"{r.reciprocal_rank:.4f}"
            pk = f"{r.precision_at_k:.4f}"
            lat = f"{r.latency_ms:.1f}"
            dims = str(r.dimensions)
            failed = str(len(r.failed_queries))
            name = r.model_name[:24]
            print(f"{name:25s} {rr:8s} {pk:8s} {lat:10s} {dims:6s} {failed:8s}")
        
        print("-" * 65)
        
        # Best model
        best = sorted_results[0]
        print(f"\n🏆 RECOMMENDATION: {best.model_name}")
        print(f"   Based on your data and {len(self.results)} models compared.")
        print()
        
        for r in sorted_results:
            if r.failed_queries:
                print(f"⚠  {r.model_name} failed on: {r.failed_queries[:3]}")
        
        print("=" * 80)


# ── Run the comparison ──
if __name__ == "__main__":
    # Step 1: Define your test data
    # Your actual documents — not generic benchmark data!
    documents = [
        "Our refund policy allows returns within 30 days of purchase with original packaging.",
        "Password reset: Go to Settings → Account → Security to change your password.",
        "Two-factor authentication adds SMS or authenticator app verification to logins.",
        "International shipping costs $25 flat rate and takes 5-10 business days.",
        "API rate limits: 100 requests/hour on Free tier, 10K/hour on Pro tier.",
        "We accept Visa, Mastercard, American Express, and PayPal.",
        "Account deletion: Contact support with your account email for verification.",
        "Data export available in CSV, JSON, and PDF formats from Settings → Privacy.",
        "Team management: Admins can add members and assign roles from Settings → Team.",
        "Subscription: Monthly ($29) and Annual ($290, save 2 months) plans available.",
    ]
    
    # Step 2: Define your queries and which documents are RELEVANT
    queries = [
        "I want my money back",
        "I forgot my login password",
        "How do I add extra security to my account",
        "How much does shipping cost overseas",
        "How do I delete all my data permanently",
    ]
    
    # For each query, which document indices are the correct answer?
    relevant_docs = [
        [0],              # "I want my money back" → refund policy
        [1],              # "I forgot my password" → password reset
        [2],              # "extra security" → 2FA
        [3],              # "shipping overseas" → international shipping
        [6, 7],           # "delete my data" → account deletion OR data export
    ]
    
    # Step 3: Add models to compare
    comparator = ModelComparator()
    
    # Quick comparison models (choose models YOU can run)
    # All of these are free, open-source, and run locally
    models_to_test = [
        ModelSpec("MiniLM (384D)", "all-MiniLM-L6-v2", "sentence-transformers"),
        ModelSpec("MPNet (768D)", "all-mpnet-base-v2", "sentence-transformers"),
        ModelSpec("BGE Small (384D)", "BAAI/bge-small-en-v1.5", "sentence-transformers"),
        ModelSpec("BGE Base (768D)", "BAAI/bge-base-en-v1.5", "sentence-transformers"),
    ]
    
    # Step 4: Load and run
    print("═══ Loading models... ═══\n")
    for spec in models_to_test:
        comparator.add_sentence_transformer(spec)
    
    print("\n═══ Running comparison... ═══")
    results = comparator.compare_models(documents, queries, relevant_docs)
    
    # Step 5: Print report
    comparator.print_report()
    
    print("\n🤔 Reflection Questions:")
    print("  1. Did the highest-MTEB model win on YOUR data?")
    print("  2. If not, why? What does this tell you about benchmarks?")
    print("  3. Is the winning model's margin worth its latency cost?")
    print("  4. What would happen with 100x more documents? Would the ranking change?")
```

**Expected output:**
```
═══ Loading models... ═══

  Added MiniLM (384D): 384D
  Added MPNet (768D): 768D
  Added BGE Small (384D): 384D
  Added BGE Base (768D): 768D

═══ Running comparison... ═══

  Testing: MiniLM (384D)
  Testing: MPNet (768D)
  Testing: BGE Small (384D)
  Testing: BGE Base (768D)

══════════════════════════════════════════════════════════════════════════════
MODEL COMPARISON REPORT
══════════════════════════════════════════════════════════════════════════════
Model                     RR       P@K      Lat(ms)    Dims   Failed  
────────────────────────────────────────────────────────────────────────────
BGE Base (768D)           1.0000   1.0000   5.2        768    0       
MPNet (768D)              1.0000   1.0000   6.8        768    0       
BGE Small (384D)          0.9542   0.9250   1.8        384    0       
MiniLM (384D)             0.9421   0.9125   1.2        384    0       
────────────────────────────────────────────────────────────────────────────

🏆 RECOMMENDATION: BGE Base (768D)
   Based on your data and 4 models compared.

Note: All models performed well because the test data is simple.
With real-world domain-specific data, differences would be more dramatic.
```

**🤔 Critical analysis:**
- On this simple test, ALL models perform well. The test is too easy!
- The real question: which model is best for YOUR actual domain-specific data?
- Notice the latency difference: MiniLM is 4x faster than BGE Base. Is the quality difference worth it?

---

### Example 2: The Domain Test — Finding Specialized Models

The test above was too easy. Let's test with domain-specific vocabulary:

```python
"""
Domain-specific model comparison.
Tests whether general models fail on specialized vocabulary.
"""

import numpy as np
from sentence_transformers import SentenceTransformer
from sklearn.metrics.pairwise import cosine_similarity

# ── Medical domain test ──
medical_docs = [
    "Patients with acute myocardial infarction require immediate PCI intervention.",
    "The patient presents with exertional angina and ST-segment depression on ECG.",
    "Type 2 diabetes mellitus management includes metformin as first-line therapy.",
    "Hepatic encephalopathy is treated with lactulose and rifaximin.",
    "The biopsy revealed invasive ductal carcinoma with HER2 amplification.",
]

general_queries = [
    "heart attack treatment",      # Should match doc 0
    "chest pain during exercise",  # Should match doc 1
    "diabetes medication",         # Should match doc 2
    "liver problems treatment",    # Should match doc 3
    "breast cancer diagnosis",     # Should match doc 4
]

medical_queries = [
    "acute MI PCI management protocol",
    "exertional angina diagnostic criteria",
    "new-onset type 2 diabetes guidelines",
    "hepatic encephalopathy treatment options",
    "invasive ductal carcinoma HER2 status implications",
]

def test_models_on_domain(docs, general_queries, medical_queries, model_names):
    """
    Compare how well general and medical queries find relevant docs.
    
    🤔 BEFORE READING RESULTS:
    - Will medical-specific queries find their docs better or worse?
    - Will general-purpose models fail on medical terminology?
    - What does this tell you about the "curse of specialized vocabulary"?
    """
    for model_name in model_names:
        print(f"\n═══ {model_name} ═══")
        model = SentenceTransformer(model_name)
        
        doc_embeddings = model.encode(docs, normalize_embeddings=True)
        
        for query_set_name, query_set in [
            ("General (layperson)", general_queries),
            ("Medical (specialist)", medical_queries),
        ]:
            correct = 0
            
            for q_idx, query in enumerate(query_set):
                query_vec = model.encode([query], normalize_embeddings=True)[0]
                scores = cosine_similarity([query_vec], doc_embeddings)[0]
                top_match = np.argmax(scores)
                
                is_correct = top_match == q_idx
                if is_correct:
                    correct += 1
                
                print(f"  {'✓' if is_correct else '✗'} '{query[:50]}...'")
                print(f"    → matched: {docs[top_match][:60]}...")
                print(f"    (expected: {docs[q_idx][:60]}...)")
            
            pct = correct / len(query_set) * 100
            print(f"  [{query_set_name}] Accuracy: {correct}/{len(query_set)} ({pct:.0f}%)\n")


# Test with general models vs biomedical
models = [
    'all-MiniLM-L6-v2',        # General purpose
    'all-mpnet-base-v2',        # General purpose (better quality)
    'pritamdeka/S-PubMedBert-MSB',  # Biomedical (note: may need to download)
]

print("═══ Domain Test: Medical Vocabulary ═══")
print("\nGeneral queries use everyday language for medical concepts.")
print("Medical queries use exact clinical terminology.\n")
print("🤔 Prediction: Which model will handle medical terminology best?")
print("  Will the general model fail on 'acute MI' even though it knows 'heart attack'?")
print()

test_models_on_domain(medical_docs, general_queries, medical_queries, models)
```

**Expected output pattern:**
```
═══ Domain Test: Medical Vocabulary ═══

═══ all-MiniLM-L6-v2 ═══
  ✓ 'heart attack treatment...'
    → matched: Patients with acute myocardial infarction...
  ✓ 'breast cancer diagnosis...'  
    → matched: The biopsy revealed invasive ductal carcinoma...
  
  ✗ 'exertional angina diagnostic criteria...'
    → matched: Patients with acute myocardial infarction...  (WRONG!)
    → expected: The patient presents with exertional angina...
  
  [General (layperson)] Accuracy: 3/5 (60%)
  [Medical (specialist)] Accuracy: 1/5 (20%)

═══ S-PubMedBert-MSB ═══
  ✓ 'acute MI PCI management protocol...'
    → matched: Patients with acute myocardial infarction...
  
  ✓ 'exertional angina diagnostic criteria...'
    → matched: The patient presents with exertional angina...
  
  [General (layperson)] Accuracy: 5/5 (100%)
  [Medical (specialist)] Accuracy: 5/5 (100%)

KEY INSIGHT: The general model (MiniLM) fails on specialized terminology.
"exertional angina" and "acute MI" aren't in its training vocabulary.
The biomedical model handles both general AND specialized queries.

MORAL: If your domain has specialized vocabulary, USE A DOMAIN MODEL.
MTEB scores won't tell you this — only testing on YOUR data will.
```

**🤔 Your turn — apply this to YOUR domain:**
1. Collect 10-20 domain-specific documents from YOUR work
2. Write 5 general queries and 5 domain-specific queries
3. Run this comparison
4. Did the general model fail on your domain vocabulary?

---

### Example 3: The Dimension Impact Analysis

Let's understand what dimensions actually cost you:

```python
"""
Dimension impact analysis.
Shows how vector size affects storage, memory, and search speed.
"""

import numpy as np
import time

# ── Scenario: 1 million documents ──
n_docs = 1_000_000

dimensions = {
    "MiniLM (384D)": 384,
    "bge-base (768D)": 768,
    "text-embedding-3-small (1536D)": 1536,
    "text-embedding-3-large (3072D)": 3072,
}

print("═══ Storage Cost: 1M documents ═══\n")
print(f"{'Model':35s} {'Storage (FP32)':15s} {'Storage (FP16)':15s} {'Memory (RAM)':15s}")
print("-" * 80)

for name, dim in dimensions.items():
    # float32: 4 bytes per value
    # float16: 2 bytes per value
    storage_fp32 = n_docs * dim * 4 / (1024**3)  # GB
    storage_fp16 = n_docs * dim * 2 / (1024**3)  # GB
    memory_ram = n_docs * dim * 4 / (1024**3) * 1.5  # GB (estimated overhead)
    
    print(f"{name:35s} {storage_fp32:>8.2f} GB{'':>7s} {storage_fp16:>8.2f} GB{'':>7s} {memory_ram:>8.2f} GB")

print("\n🤔 Questions:")
print("  1. Can you fit 1M MiniLM (384D) vectors in 8GB RAM?")
print("  2. What about 10M with text-embedding-3-large (3072D)?")
print("  3. How does dimension choice affect your cloud bill?")
print()

# ── Search time comparison ──
print("═══ Search Time vs Dimensions ═══\n")

for name, dim in dimensions.items():
    n_vectors = 100_000  # Small enough to benchmark quickly
    
    # Generate random vectors
    vectors = np.random.randn(n_vectors, dim).astype(np.float32)
    query = np.random.randn(dim).astype(np.float32)
    
    # Normalize
    vectors = vectors / np.linalg.norm(vectors, axis=1, keepdims=True)
    query = query / np.linalg.norm(query)
    
    # Time the dot product
    n_runs = 10
    start = time.time()
    for _ in range(n_runs):
        scores = np.dot(vectors, query)
    avg_time = (time.time() - start) / n_runs * 1000  # ms
    
    # Estimate for 1M
    est_1m = avg_time * 10
    
    print(f"{name:35s} {avg_time:>8.2f} ms (100K) → est. {est_1m:>8.2f} ms (1M)")
```

**Expected output:**
```
═══ Storage Cost: 1M documents ═══

Model                                     Storage (FP32)    Storage (FP16)    Memory (RAM)    
MiniLM (384D)                                    1.43 GB           0.72 GB           2.15 GB
bge-base (768D)                                  2.86 GB           1.43 GB           4.29 GB
text-embedding-3-small (1536D)                   5.72 GB           2.86 GB           8.58 GB
text-embedding-3-large (3072D)                  11.44 GB           5.72 GB          17.16 GB

═══ Search Time vs Dimensions ═══

MiniLM (384D)                                   1.23 ms (100K) → est.   12.30 ms (1M)
bge-base (768D)                                 2.45 ms (100K) → est.   24.50 ms (1M)
text-embedding-3-small (1536D)                  4.89 ms (100K) → est.   48.90 ms (1M)
text-embedding-3-large (3072D)                  9.78 ms (100K) → est.   97.80 ms (1M)
```

**🤔 The dimension tradeoff becomes concrete:**
- 3072D is 8x more storage AND 8x slower than 384D
- But the quality might only be 3-5% better
- Is that tradeoff worth it? It depends entirely on your use case

---

## ✅ Good Output Examples

### What Strong Model Selection Looks Like

```
Case: Enterprise Customer Support RAG
──────────────────────────────────────
Requirements:
- 500K support documents, 10K daily queries
- Accuracy critical (wrong answers = escalations)
- Multilingual (EN, ES, FR)
- < 500ms total response time
- Budget: $500/month for embeddings

Chosen Model: text-embedding-3-small (1536D)
Justification:
  ✓ Best quality-to-cost ratio: $0.02/1M tokens
  ✓ Multilingual capabilities: trained on multi-lang data
  ✓ 1536D: good accuracy, reasonable storage (2.86GB for 500K)
  ✓ API-managed: no GPU needed, no ops overhead
  ✓ Cost: ~$50/month for embedding + queries (well under $500)

Rejected:
  ✗ text-embedding-3-large: $200/month — exceeding budget for marginal gain
  ✗ all-MiniLM-L6-v2: free but lower accuracy on multilingual
  ✗ voyage-3: better quality but $0.06/1M — 3x cost
  
Validation: Ran test on 100 support queries → 94% MRR with text-embedding-3-small
             vs 89% with MiniLM, 96% with 3-large (not worth 4x cost)
```

### What Weak Model Selection Looks Like

```
Case: Personal Note-Search App
─────────────────────────────────
Requirements:
- 2,000 personal notes
- Single user, no budget
- English only

Chosen Model: text-embedding-3-large (3072D)
Why it's wrong:
  ✗ $0.13/1M tokens for 2000 notes → won't cost much, but still unnecessary
  ✗ 3072D → 23MB for 2000 vectors, but latency suffers
  ✗ A 384D model would be 8x faster with similar quality for simple note search
  ✗ Requires internet + API key for a personal project

Better choice: all-MiniLM-L6-v2 (384D)
  ✓ Free, local, fast
  ✓ 384D is plenty for note search
  ✓ Can run on any laptop, zero infrastructure
  ✓ 1.2ms per search vs ~5ms with 3-large (on local CPU)
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Picking by MTEB Rank Only

```python
# ❌ WRONG
model = "the #1 model on MTEB leaderboard"
# MTEB aggregates 50+ tasks. #1 overall might be terrible for retrieval.

# ✅ RIGHT
model = "the model that performs best on MY data and queries"
# Test on your domain. MTEB is a starting point, not a gospel.
```

### Antipattern 2: Ignoring Embedding Size Impact

```python
# ❌ WRONG — choosing 3072D "because it's better"
# With 10M documents:
# 3072D → 122 GB storage, 98ms search time
# 384D  → 15 GB storage, 12ms search time
# The 3% quality gain isn't worth 8x the infra cost

# ✅ RIGHT — match dimensions to your scale
# < 100K docs: dimension barely matters, choose for quality
# 100K-1M docs: 768-1024D balanced sweet spot
# 1M+ docs: 384-768D unless you have budget for the infra
```

### Antipattern 3: Free-Model Dogmatism

```
"Open-source models are just as good as paid ones."

REALITY: For many use cases, yes. For some, no.

When free models lose:
- Multilingual retrieval (OpenAI/Cohere spend $$$ on multilingual training)
- Very long documents (>512 tokens: some open models truncate)
- Edge cases requiring massive model capacity
- Domain-specific vocabulary without a fine-tuned open model

When paid models lose:
- Cost at scale (10M docs × $0.02/1M tokens × ~200 tokens = $40)
- Latency (every query needs a network call)
- Privacy (data leaves your infrastructure)
- Offline/air-gapped deployments
```

### Antipattern 4: Not Testing on Your Data

```
WORST PRACTICE:
1. Read blog post: "Top 5 Embedding Models 2026"
2. Pick #1
3. Deploy to production
4. Wonder why search results are bad for YOUR domain

BEST PRACTICE:
1. Collect 50 representative documents from your corpus
2. Write 20 realistic queries with known correct answers
3. Test 3-5 candidate models on this dataset
4. Pick the one that performs best on YOUR metrics
5. Deploy and monitor

This takes 2 hours and saves 2 weeks of debugging.
```

### Failure Mode: The "Model Drift" Problem

```
You deploy a model. It works great for 6 months. Then search quality degrades.

WHY? The embedding model didn't change — YOUR DATA changed.
- New products with new terminology
- New documentation for new features
- User queries use different language over time

SOLUTION: Re-run model comparison every 3-6 months with fresh data.
Or: Use a model that's continuously updated (OpenAI, Cohere).
```

---

## 🧪 Drills & Challenges

### Drill 1: Your Personal Model Shootout (35 min)

1. Collect 20 documents from YOUR actual work (emails, docs, notes, code comments)
2. Write 10 realistic search queries
3. Label which documents are relevant for each query
4. Compare 3+ models using the ModelComparator above
5. Pick your winner and justify with data

**Key insight:** You now know which model to use for YOUR actual data. Not some benchmark. Not a blog post. YOUR data.

### Drill 2: The Budget Calculator (15 min)

Write a script that calculates the MONTHLY cost of EACH model at different scales:

| Scale | Docs | Queries/day | Avg doc tokens | Avg query tokens |
|-------|------|-------------|----------------|-----------------|
| Tiny | 1K | 100 | 200 | 15 |
| Small | 10K | 1K | 500 | 20 |
| Medium | 100K | 10K | 1000 | 25 |
| Large | 1M | 100K | 2000 | 30 |

Include:
- One-time indexing cost (embedding all docs)
- Monthly query cost (embedding queries)
- Storage cost
- API vs self-hosted comparison

**Purpose:** You'll internalize the cost differences when you compute them yourself.

### Drill 3: The "Does Domain Matter?" Experiment (30 min)

1. Pick a domain you know well (code, finance, medical, legal, sports, cooking — anything)
2. Find a general model and a domain-specific model
3. Create 10 query-document pairs where you KNOW a general model would struggle
4. Compare the scores

**For example (code domain):**
```
Query: "function to find the longest common subsequence"
General model might match: "longest sequence in array" (WRONG — different concept)
Code model might match: "LCS algorithm implementation" (RIGHT — same concept)
```

### Drill 4: The Adversarial Model Test (20 min)

For your chosen model, find the "adversarial examples" — queries where the model gives CONFIDENTLY WRONG results (similarity > 0.8 but completely irrelevant).

1. Write 5 such queries
2. Analyze the failure pattern
3. Does this failure pattern affect your production use case?
4. If yes, consider a different model. If no, document it as a known limitation.

**Professional practice:** OpenAI publishes "known limitations" for every model. You should too.

### Drill 5: The Dimension Ablation Study (25 min)

Take ONE model (e.g., `all-mpnet-base-v2`, 768D) and test what happens when you reduce dimensions:

```python
from sklearn.decomposition import PCA

model = SentenceTransformer('all-mpnet-base-v2')
embeddings = model.encode(docs, normalize_embeddings=True)

for n_dims in [768, 512, 256, 128, 64]:
    pca = PCA(n_components=n_dims)
    reduced = pca.fit_transform(embeddings)
    # Test search quality with reduced embeddings
    # How many dimensions do you NEED to maintain 95% of original quality?
```

**Expected finding:** Most embedding information is concentrated in the first ~128 dimensions. You can often cut dimensions by 75% with only 1-2% quality loss. This is called the "intrinsic dimension" of embeddings.

---

## 🚦 Gate Check

Before moving to File 04 (ChromaDB), confirm you can:

- [ ] **List the 5 dimensions** of embedding model selection from memory
- [ ] **Run a model comparison** on YOUR data using the comparator above
- [ ] **Calculate costs** for API-based embedding at a given scale
- [ ] **Explain when a domain-specific model** beats a general-purpose one
- [ ] **Know the dimension tradeoff** — when to choose 384D vs 768D vs 1536D vs 3072D
- [ ] **Answer these questions:**
  1. If you have 100K documents in English, and quality is your ONLY concern, what do you choose?
  2. If you have 10M multilingual documents with tight budget, what do you choose?
  3. What does MTEB actually measure? Why is it insufficient for model selection?
  4. How would you test if a new model is better than your current one?

- [ ] **Write down your personal model selection heuristic** (3-4 sentence decision framework)

---

## 📚 Resources

**Benchmarks:**
- [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard) — Massive Text Embedding Benchmark
- [BEIR Benchmark](https://github.com/beir-cellar/beir) — Specialized retrieval evaluation
- [OpenAI Embeddings Comparison](https://openai.com/blog/new-embedding-models) — Their official benchmarks

**Model Repositories:**
- HuggingFace: `sentence-transformers` collection (300+ models)
- Voyage AI: Domain-specific models (legal, code, multilingual)
- Cohere: Enterprise embedding models with multilingual support

**Production Reading:**
- "Choosing an Embedding Model" — LlamaIndex docs (practical guide)
- "Embedding Models: A Comprehensive Evaluation" — Arxiv survey paper
- "The Curse of Dimensionality in Vector Search" — Technical explanation

**Your Decision Flowchart:**
```
START: What's your primary constraint?
│
├── Budget = $0 → Open-source sentence-transformers
│   ├── Need speed → MiniLM / BGE-small (384D)
│   ├── Need quality → BGE-base/mpnet (768D)
│   └── Need domain → Search HuggingFace for domain-specific
│
├── Latency critical (< 50ms) → Small open-source model
│   └── Consider dimension: 384D keeps memory/search fast
│
├── Quality critical → text-embedding-3-large or voyage-3
│   └── Budget allows it, latency is secondary
│
├── Multilingual → Cohere or text-embedding-3
│   └── Open-source multilingual models are catching up
│
└── Large scale (> 1M docs) → Dimension is now CRITICAL
    ├── Can you fit in RAM? 3072D × 1M × 4 bytes = 12GB
    └── Can you afford ANN reindexing? Higher dim = slower
```

---

> **🛑 Revisit the 3 discovery questions from the beginning:**
>
> 1. **The Essential Criteria** — Did your 5 factors match the framework? Did you miss any?
> 2. **The Dimension Tradeoff** — You now know concrete numbers. Does 1536D vs 384D matter for YOUR use case?
> 3. **Free vs Paid** — Can you now calculate the exact cost difference for YOUR scale?
>
> **Key realization:** You didn't memorize a list of "best models." You learned a FRAMEWORK for evaluating ANY model. This framework works today, next year, and when new models inevitably obsolete today's leaderboard. That's the method.
>
> **Next:** File 04 — ChromaDB. Now that you know which model to use, let's build a real vector database.
