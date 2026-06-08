# 09 — Project: The Advanced RAG Engine — Multi-Strategy, Self-Improving

## 🎯 Project Purpose

This is your Phase 5 portfolio project — a production-ready Advanced RAG Engine that combines ALL the techniques from this phase into a single, intelligent system.

The engine is **multi-strategy**: it doesn't use one retrieval approach. It classifies each query, routes to the optimal strategy, and continuously improves based on evaluation feedback.

**What makes this project different from Phase 4's Customer Support Bot:**

| Phase 4 Bot | Phase 5 Engine |
|-------------|----------------|
| Single retrieval strategy | Multi-strategy routing (standard, HyDE, multi-hop, graph, vision) |
| Fixed chunking | Dynamic chunking with contextual enrichment |
| No query understanding | Query classification + entity extraction |
| Single generator | Strategy-specific generation with citation control |
| Manual evaluation | Automated component-level evaluation |
| Static | Self-improving (eval feedback → strategy tuning) |
| Text-only | Multimodal (text + images + diagrams) |
| Vector-only | Hybrid vector + knowledge graph |

---

## ⏱ Time Budget

| Milestone | Estimated Time |
|-----------|---------------|
| Architecture design & planning | 1.5h |
| Milestone 1: Core ingestion pipeline | 2h |
| Milestone 2: Multi-strategy retrieval | 3h |
| Milestone 3: Query routing & generation | 2h |
| Milestone 4: Evaluation & self-improvement | 2h |
| Milestone 5: Production deployment | 1.5h |
| Portfolio preparation | 1h |
| **Total** | **~13 hours** |

---

## 📖 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                      ADVANCED RAG ENGINE                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  INGESTION LAYER                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │  Text    │  │  Image   │  │  Entity  │  │  Community       │   │
│  │  Chunker │  │  Proc.   │  │  Extract │  │  Detection       │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────────┬─────────┘   │
│       │              │             │                  │              │
│       ▼              ▼             ▼                  ▼              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │  Vector  │  │  CLIP    │  │ Knowledge│  │  Community       │   │
│  │  Index   │  │  Index   │  │  Graph   │  │  Summaries       │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘   │
│                                                                      │
│  ROUTING LAYER                                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Query Classifier                                              │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────────┐ │   │
│  │  │Simple  │ │Vocab   │ │Multi-  │ │Visual  │ │Relationship│ │   │
│  │  │ Fact   │ │Mismatch│ │ Doc    │ │ Query  │ │  Query     │ │   │
│  │  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └──────┬─────┘ │   │
│  │      │          │          │          │             │         │   │
│  │      ▼          ▼          ▼          ▼             ▼         │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────────┐ │   │
│  │  │Standard│ │ HyDE   │ │ Agentic│ │ Vision │ │  GraphRAG  │ │   │
│  │  │  RAG   │ │        │ │ Multi  │ │  RAG   │ │            │ │   │
│  │  │        │ │        │ │  Hop   │ │        │ │            │ │   │
│  │  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └──────┬─────┘ │   │
│  └──────┼──────────┼──────────┼──────────┼──────────────┼───────┘   │
│         │          │          │          │              │           │
│         ▼          ▼          ▼          ▼              ▼           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  GENERATION LAYER                                             │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │   │
│  │  │  Context     │  │  LLM Gen.   │  │  Citation        │   │   │
│  │  │  Builder     │─▶│  (strategy-  │─▶│  Verification    │   │   │
│  │  │              │  │  specific)   │  │                  │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  EVALUATION LAYER                                                    │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────┐  ┌──────────┐   │   │
│  │  │ Component  │  │ E2E Eval   │  │Regress.│  │ Strategy │   │   │
│  │  │  Eval      │  │ (RAGAS)    │  │ Tests  │  │ Optimizer│   │   │
│  │  └────────────┘  └────────────┘  └────────┘  └──────────┘   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📦 Project Structure

```
advanced-rag-engine/
├── README.md                       # Architecture decisions, setup, usage
├── requirements.txt
├── .env.example
│
├── ingest/
│   ├── __init__.py
│   ├── text_processor.py           # Chunking, embedding, indexing
│   ├── image_processor.py          # OCR, vision description, CLIP
│   ├── graph_builder.py            # Entity extraction, graph construction
│   ├── community_summarizer.py     # Community detection + summarization
│   └── document_loader.py          # Multi-format document reader
│
├── retrieve/
│   ├── __init__.py
│   ├── query_router.py             # Query classifier → strategy selection
│   ├── standard_retriever.py       # Baseline vector search
│   ├── hyde_retriever.py           # HyDE-enhanced retrieval
│   ├── multi_hop_retriever.py      # Multi-hop agentic retrieval
│   ├── graph_retriever.py          # Graph traversal + community search
│   ├── multimodal_retriever.py     # Vision + text retrieval
│   └── reranker.py                 # Cross-encoder reranking
│
├── generate/
│   ├── __init__.py
│   ├── context_builder.py          # Build strategy-specific context
│   ├── generator.py                # LLM generation with citations
│   └── citation_verifier.py        # Verify claims against sources
│
├── evaluate/
│   ├── __init__.py
│   ├── component_evaluator.py      # Isolate retriever/generator
│   ├── ragas_evaluator.py          # RAGAS metrics + aspect critique
│   ├── regression_suite.py         # Regression test runner
│   ├── bias_detector.py            # LLM-as-judge bias analysis
│   └── strategy_optimizer.py       # Tune routing based on eval results
│
├── engine.py                       # Main entry point (FastAPI app)
├── config.py                       # Strategy configs per deployment
│
├── tests/
│   ├── test_retrievers.py
│   ├── test_routing.py
│   ├── test_generation.py
│   ├── test_evaluation.py
│   └── fixtures/                   # Test documents & images
│
├── data/
│   ├── documents/                  # Source documents
│   ├── images/                     # Source images/screenshots
│   ├── indices/                    # Saved vector indices
│   ├── graph/                      # Serialized knowledge graph
│   └── eval_results/              # Evaluation history
│
├── scripts/
│   ├── run_ingestion.py
│   ├── run_evaluation.py
│   ├── run_regression.py
│   └── benchmark_strategies.py
│
└── docker-compose.yml              # For production deployment
```

---

## 🏗️ Milestone 1: Ingestion Pipeline

### What to Build

A document ingestion pipeline that:
1. **Reads multiple formats:** Markdown, PDF (with images), HTML, plain text
2. **Chunks text intelligently:** With contextual enrichment (from File 02)
3. **Processes images:** OCR, vision descriptions, CLIP embeddings (from File 05)
4. **Extracts entities + builds graph:** Knowledge graph from extracted entities (from File 06)
5. **Detects communities + generates summaries:** For global search (from File 06)
6. **Stores everything:** Vector index (text), CLIP index (images), knowledge graph, community summaries

### Core Classes

```python
class DocumentLoader:
    """Read documents from multiple formats."""

class TextProcessor:
    """Chunk text with contextual enrichment."""

class ImageProcessor:
    """OCR, describe, CLIP-embed images."""

class GraphBuilder:
    """Entity extraction → graph → community summaries."""

class IngestionPipeline:
    """Orchestrate ingestion of a document collection."""
```

### Ingestion Flow

```python
async def run_ingestion(doc_dir: str):
    pipeline = IngestionPipeline(
        text_processor=TextProcessor(chunk_size=512, overlap=128),
        image_processor=ImageProcessor(ocr_engine="easyocr"),
        graph_builder=GraphBuilder(llm_client=client),
        vector_store=QdrantClient(...),
    )

    # Process all documents
    result = await pipeline.ingest_directory(doc_dir)

    print(f"Ingested: {result.text_chunks} text chunks, "
          f"{result.images} images, "
          f"{result.entities} entities, "
          f"{result.communities} communities")
```

### Key Decisions to Make

- **Chunk size:** 512 for general text, 256 for code-heavy docs, 1024 for narrative docs
- **Overlap strategy:** Fixed 10% overlap vs. semantic boundary detection
- **Image preprocessing:** OCR-only vs. OCR + vision description vs. all three (OCR, description, CLIP)
- **Entity extraction frequency:** Every document vs. once per batch vs. incremental
- **Graph storage:** NetworkX (dev) vs. Neo4j (prod)

### Deliverables

- [ ] `DocumentLoader` handles text + PDF + HTML + Markdown
- [ ] `TextProcessor` produces contextually enriched chunks
- [ ] `ImageProcessor` extracts OCR text + generates descriptions + creates CLIP embeddings
- [ ] `GraphBuilder` extracts entities, resolves duplicates, detects communities
- [ ] Everything is persisted (vector index, CLIP index, graph, summaries)
- [ ] Ingestion can run incrementally (re-process only changed documents)

---

## 🏗️ Milestone 2: Multi-Strategy Retrieval

### What to Build

Five retrieval strategies, each implemented as a standalone module:

1. **Standard Retriever:** `text-embedding-3-small` + cosine similarity (baseline)
2. **HyDE Retriever:** LLM generates hypothetical document → embed that → search
3. **Multi-Hop Retriever:** Decompose query → sequential retrieval with gap analysis
4. **Graph Retriever:** Extract entities from query → traverse graph → find connected info
5. **Multimodal Retriever:** CLIP-embed query → search image index → supplement with text

All strategies implement a common interface:

```python
class RetrievalStrategy(ABC):
    @abstractmethod
    async def retrieve(
        self,
        query: str,
        top_k: int = 10,
    ) -> RetrievalResult:
        """
        Returns:
            RetrievalResult with:
            - chunks: List[Dict] — retrieved text chunks
            - images: List[Dict] — retrieved images (if multimodal)
            - graph_path: List[Dict] — traversal path (if GraphRAG)
            - scores: List[float] — relevance scores
            - metadata: Dict — latency, strategy used, etc.
        """
        pass
```

### Core Classes

```python
class StandardRetriever(RetrievalStrategy):
    """Standard vector search."""

class HyDERetriever(RetrievalStrategy):
    """HyDE-enhanced retrieval."""

class MultiHopRetriever(RetrievalStrategy):
    """Multi-hop with gap analysis."""

class GraphRetriever(RetrievalStrategy):
    """Graph traversal + community search."""

class MultimodalRetriever(RetrievalStrategy):
    """Vision + CLIP retrieval."""
```

### Strategy Comparison Harness

Build a comparison framework:

```python
strategies = {
    "standard": StandardRetriever(vector_store),
    "hyde": HyDERetriever(llm_client, vector_store),
    "multi_hop": MultiHopRetriever(llm_client, vector_store),
    "graph": GraphRetriever(graph, vector_store),
    "multimodal": MultimodalRetriever(clip, vector_store),
}

# Run all strategies on the same query
for name, strategy in strategies.items():
    result = await strategy.retrieve(query)
    print(f"{name}: recall@{5} = {result.recall_at_5:.3f}, "
          f"latency = {result.latency_ms:.0f}ms, "
          f"cost = ${result.cost:.5f}")
```

### Deliverables

- [ ] All 5 strategies implement the common interface
- [ ] Each strategy is independently testable
- [ ] Comparison harness measures recall, precision, latency, cost
- [ ] At least 20 test queries across all strategy types
- [ ] Strategy comparison documented in README

---

## 🏗️ Milestone 3: Query Routing & Generation

### What to Build

A query router that classifies each incoming query and routes to the optimal strategy:

```python
class QueryRouter:
    """
    Route queries to the optimal retrieval strategy.

    Classification:
    - SIMPLE_FACT: One chunk answers it → Standard RAG
    - VOCAB_MISMATCH: Query language ≠ doc language → HyDE
    - MULTI_DOC: Needs multiple sources → Multi-Hop
    - VISUAL: Needs image understanding → Multimodal
    - RELATIONSHIP: Needs entity connections → GraphRAG
    - COMPLEX: Multiple signals → Combined strategies
    """

    async def classify(self, query: str) -> QueryClassification:
        """Determine query type and recommended strategy."""

    async def retrieve(self, query: str) -> RetrievalResult:
        """Route query and execute the optimal strategy."""
```

And a generator that builds strategy-specific context:

```python
class ContextBuilder:
    """
    Build LLM context based on the retrieval strategy used.

    Different strategies produce different context formats:
    - Standard RAG: list of chunks
    - Multi-Hop: reasoning chain + chunks
    - GraphRAG: entity descriptions + traversal path + chunks
    - Multimodal: text chunks + base64-encoded images
    """

class Generator:
    """Generate answers with citations, strategy-aware."""

class CitationVerifier:
    """
    Verify that every claim in the answer is supported by
    the retrieved context. Flag unsupported claims.
    """
```

### Query Classification Accuracy

Build a test set to measure routing accuracy:

| Query | Correct Route | Your Router |
|-------|--------------|-------------|
| "What's the refund policy?" | Standard | ? |
| "How do I get my money back?" | HyDE | ? |
| "Who manages the team that built X?" | Multi-Hop | ? |
| "What does the architecture diagram show?" | Multimodal | ? |
| "What projects does Dr. Chen work on?" | GraphRAG | ? |
| "Compare mobile and web onboarding" | Multi-Hop | ? |

**Target: 90%+ routing accuracy.**

### Deliverables

- [ ] QueryRouter with ≥90% classification accuracy on 30 test queries
- [ ] ContextBuilder generates strategy-specific context formatting
- [ ] Generator produces answers with source citations
- [ ] CitationVerifier flags unsupported claims
- [ ] Router falls back gracefully (if strategy fails, try next best)

---

## 🏗️ Milestone 4: Evaluation & Self-Improvement

### What to Build

An automated evaluation loop that:

1. **Runs every query through component-level evaluation** (isolate retriever vs. generator)
2. **Tracks strategy performance over time** — which strategy works for which query type?
3. **Detects regressions** — score drops between versions
4. **Adjusts routing thresholds** — if HyDE isn't helping for X query type, stop routing there

### Core Classes

```python
class StrategyMonitor:
    """
    Track performance of each retrieval strategy over time.

    For each (strategy, query_type) pair, track:
    - Average faithfulness, recall, precision
    - Average latency, cost
    - Success rate (score > threshold)

    Use this to identify:
    - Which strategies perform best for which query types
    - Which strategies are degrading over time
    - Which queries would benefit from a different strategy
    """

class SelfOptimizer:
    """
    Automatically adjust routing and strategy parameters
    based on evaluation results.

    Adjustable parameters:
    - Routing thresholds (when to use HyDE vs. standard)
    - HyDE temperature (more/less creative hypotheticals)
    - Multi-hop max depth (stop earlier if early hops suffice)
    - Reranking cutoff (keep top-N after reranking)
    - Chunk size (if certain doc types need different sizes)
    """
```

### Evaluation Dashboard (CLI)

Build a command-line dashboard:

```
╔═══════════════════════════════════════════════════════════════╗
║  ADVANCED RAG ENGINE — EVALUATION REPORT                      ║
║  Version: 2.1.0    Date: 2025-05-16    Run: a1b2c3d4         ║
╠═══════════════════════════════════════════════════════════════╣
║  OVERALL                                                      ║
║  Composite Score:  0.874  ▲ +2.1% from last run              ║
║  Faithfulness:     0.892  ▲ +1.4%                             ║
║  Answer Relevancy: 0.856  ▲ +0.8%                             ║
║  Avg Latency:      1.24s  ▼ -12%  (optimization working)     ║
║  Avg Cost:         $0.0039 ▼ -18%                              ║
╠═══════════════════════════════════════════════════════════════╣
║  BY STRATEGY                                                  ║
║  Standard:  0.921  |  used: 54%  |  latency: 0.41s  |  C:0.001║
║  HyDE:      0.903  |  used: 18%  |  latency: 0.89s  |  C:0.003║
║  Multi-Hop: 0.845  |  used: 12%  |  latency: 2.31s  |  C:0.008║
║  GraphRAG:  0.812  |  used: 8%   |  latency: 1.82s  |  C:0.005║
║  Multimodal:0.721  |  used: 5%   |  latency: 4.15s  |  C:0.015║
║  Complex:   0.788  |  used: 3%   |  latency: 5.42s  |  C:0.022║
╠═══════════════════════════════════════════════════════════════╣
║  REGRESSION TESTS: 18/20 passed                               ║
║  ❌ "Enterprise rate limit" — Faithfulness dropped 0.91→0.72  ║
║  ❌ "Screenshot error E1042" — Vision retrieval failed        ║
╠═══════════════════════════════════════════════════════════════╣
║  ROUTING OPTIMIZATION RECOMMENDATIONS                         ║
║  • Move "vocab mismatch" queries from Multi-Hop to HyDE      ║
║    (HyDE scores 0.91 vs. Multi-Hop 0.84 for these)           ║
║  • Increase multimodal fallback threshold                     ║
║    (too many queries without images routed to vision)         ║
║  • Reduce GraphRAG max depth from 3 to 2                      ║
║    (depth 3 adds 800ms but rarely adds new info)              ║
╚═══════════════════════════════════════════════════════════════╝
```

### Deliverables

- [ ] Component-level evaluation runs on every query
- [ ] StrategyMonitor tracks per-strategy performance
- [ ] Regression suite runs before every deployment
- [ ] SelfOptimizer adjusts routing based on evaluation results
- [ ] CLI dashboard shows real-time evaluation status
- [ ] At least one self-improvement cycle demonstrated (router changes its behavior based on data)

---

## 🏗️ Milestone 5: Production Deployment

### What to Build

Deploy the engine as a FastAPI application:

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(title="Advanced RAG Engine")

class QueryRequest(BaseModel):
    query: str
    strategy: Optional[str] = "auto"  # auto-routing or force-specific
    top_k: int = 10
    include_images: bool = False

class QueryResponse(BaseModel):
    answer: str
    citations: List[Citation]
    strategy_used: str
    latency_ms: float
    cost: float
    eval_scores: Optional[Dict[str, float]]

@app.post("/query")
async def query(request: QueryRequest) -> QueryResponse:
    ...

@app.post("/ingest")
async def ingest_document(file: UploadFile) -> Dict:
    ...

@app.get("/evaluate")
async def get_evaluation_report() -> Dict:
    ...

@app.get("/strategies")
async def list_strategies() -> Dict:
    ...
```

### Docker Deployment

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "engine:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
version: "3.8"
services:
  engine:
    build: .
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    volumes:
      - ./data:/app/data
      - ./indices:/app/indices
```

### Deliverables

- [ ] FastAPI application with `/query`, `/ingest`, `/evaluate`, `/strategies` endpoints
- [ ] Docker Compose configuration for production deployment
- [ ] Environment configuration (`.env.example`)
- [ ] API documentation (auto-generated by FastAPI)
- [ ] Health check endpoint
- [ ] Request logging with latency tracking
- [ ] Error handling with graceful degradation

---

## 📊 Evaluation & Success Criteria

### Quantitative Gates

| Metric | Target | How to Measure |
|--------|--------|---------------|
| Composite quality score | ≥ 0.85 | ComponentEvaluator + RAGAS |
| Faithfulness | ≥ 0.90 | AspectCritic + CitationVerifier |
| Routing accuracy | ≥ 90% | QueryRouter accuracy on 30 labeled queries |
| Strategy comparison | HyDE, multi-hop, graph all improve over standard | A/B test on 50 queries each |
| Latency avg | ≤ 2 seconds | Per-request timing |
| Cost avg | ≤ $0.005/query | Token tracking |
| Regression pass rate | 100% | Regression test suite |
| Self-improvement | Router changes strategy allocation based on data | Before/after comparison |

### Qualitative Gates

- [ ] The engine correctly handles vocabulary mismatch (HyDE improves recall by 15%+)
- [ ] The engine correctly performs multi-hop retrieval (full reasoning chain)
- [ ] The engine finds relevant images for visual queries
- [ ] The engine traverses knowledge graph for entity relationship questions
- [ ] The engine correctly says "I don't know" when information is missing
- [ ] The engine detects and surfaces contradictions when sources disagree
- [ ] The engine's routing adapts based on evaluation feedback

### Hypothesis Testing

Test these hypotheses with YOUR engine:

1. **HyDE improves recall by ≥15%** for vocabulary-mismatch queries vs. standard RAG
2. **Multi-hop improves answer completeness by ≥20%** for multi-document questions vs. standard RAG
3. **GraphRAG improves reasoning accuracy by ≥25%** for relationship questions vs. standard RAG
4. **Tiered routing reduces cost by ≥50%** vs. running all queries through the most expensive strategy
5. **Component-level evaluation** catches failures that end-to-end evaluation misses (identify at least 2 cases)

---

## 📝 Portfolio README Template

Your project's README is as important as the code. It's what hiring managers read first.

```markdown
# Advanced RAG Engine — Multi-Strategy, Self-Improving

## One-Liner
A production-grade RAG engine that classifies queries, routes to the optimal 
retrieval strategy (standard, HyDE, multi-hop, GraphRAG, multimodal), evaluates 
its own output, and improves its routing over time.

## Architecture
                 ┌─────────────┐
                 │   Query     │
                 │  Classifier │
                 └──────┬──────┘
          ┌─────────────┼──────────────┐
          ▼              ▼              ▼
    ┌──────────┐  ┌──────────┐  ┌──────────┐
    │ Standard │  │   HyDE   │  │ Multi-   │
    │  RAG     │  │          │  │ Hop      │
    └──────────┘  └──────────┘  └──────────┘
    ┌──────────┐  ┌──────────┐  ┌──────────┐
    │ GraphRAG │  │Multimodal│  │ Complex  │
    └──────────┘  └──────────┘  └──────────┘
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                   ┌───────────┐
                   │  Context  │
                   │  Builder  │
                   └─────┬─────┘
                         ▼
                   ┌───────────┐
                   │ LLM Gen.  │
                   └───────────┘
                         ▼
                   ┌───────────┐
                   │  Eval +   │
                   │ Optimize  │
                   └───────────┘

## Key Technical Decisions

### Why HyDE instead of query rewriting?
- HyDE converts queries into document-language, fixing vocabulary mismatch
- Query rewriting (LLM rephrases) doesn't change the embedding type
- Benchmarked: HyDE improved recall by 22% vs. 8% for query rewriting

### Why NetworkX instead of Neo4j?
- This project's scale (10K entities, 50K relationships) fits in memory
- Neo4j adds operational complexity for marginal benefit at this scale
- Migration path: NetworkX → Neo4j when graph exceeds 100K nodes

### Why tiered routing instead of one strategy?
- 54% of queries are simple facts → standard RAG ($0.001/query)
- Only 5% need vision → multimodal RAG ($0.015/query)
- Tiered routing reduced costs by 67% with only 3% quality drop
- Full cost-quality analysis in docs/

## Performance

| Strategy | Recall@5 | Precision@5 | Latency | Cost/Query | Usage |
|----------|----------|-------------|---------|------------|-------|
| Standard | 0.88 | 0.72 | 410ms | $0.001 | 54% |
| HyDE | 0.91 | 0.78 | 890ms | $0.003 | 18% |
| Multi-Hop | 0.85 | 0.81 | 2310ms | $0.008 | 12% |
| GraphRAG | 0.79 | 0.84 | 1820ms | $0.005 | 8% |
| Multimodal | 0.72 | 0.76 | 4150ms | $0.015 | 5% |
| Complex | 0.81 | 0.82 | 5420ms | $0.022 | 3% |

**Overall: Composite 0.874, Avg Latency 1.24s, Avg Cost $0.0039/query**

## Self-Improvement Demo

### Before optimization:
- 30% of vocabulary-mismatch queries routed to Multi-Hop (wrong strategy)
- Multi-Hop scored 0.84 on those queries at 3× the cost of HyDE

### After optimization (auto-detected by StrategyMonitor):
- Router now sends vocab-mismatch queries to HyDE
- HyDE scores 0.91 on same queries at 1/3 the cost
- +7% quality, -67% cost for this query type

## Setup

```bash
git clone <repo>
pip install -r requirements.txt
cp .env.example .env
# Add your API keys
python scripts/run_ingestion.py --data-dir ./data/documents/
uvicorn engine:app
```

## API

### POST /query
```json
{
  "query": "What research does Project Alpha's leader focus on?",
  "strategy": "auto"
}
→
{
  "answer": "Dr. Sarah Chen leads Project Alpha...",
  "citations": [...],
  "strategy_used": "graphrag",
  "latency_ms": 1820,
  "cost": 0.0047
}
```

## Lessons Learned

1. **HyDE helps most when queries are short and documents are formal**
   - Long queries already have good embeddings — HyDE adds noise
   - Rule: HyDE for queries <10 words, skip for >20 words

2. **Multi-hop's gap analyzer is the hardest component to tune**
   - Too aggressive → infinite loop
   - Too conservative → shallow reasoning
   - Sweet spot: stop after 3 hops or when 80% of sub-questions answered

3. **GraphRAG quality depends entirely on entity extraction**
   - Bad extraction → bad graph → bad retrieval
   - Invest in entity resolution before graph traversal
   - Our resolver caught 94% of duplicates after fine-tuning

4. **Self-improvement works but needs guardrails**
   - Router can oscillate if eval data is noisy
   - Solution: require 100+ samples before changing routing rules
   - Implemented in StrategyOptimizer with configurable confidence thresholds
```

---

## 🔥 Stretch Goals (Portfolio Differentiators)

If you finish early, add one of these to make your project STAND OUT:

### Stretch 1: Active Learning Loop

The engine identifies queries where it's UNCERTAIN (low faithfulness score, contradictory sources) and flags them for human review. Human corrections are fed back into the evaluation dataset and used to tune routing:

```python
class ActiveLearningLoop:
    """
    Flag uncertain queries for human review.
    Use human corrections to improve routing.
    
    Triggers for human review:
    - Faithfulness score < 0.7
    - Contradictory sources detected
    - Multi-hop gap analyzer "confidence" < 0.5
    - User explicitly clicked "thumbs down"
    """
```

### Stretch 2: Cost-Aware Routing

The router considers NOT JUST query type but also the CURRENT COST BUDGET. If you're nearing your daily budget, lower-cost strategies get priority:

```python
async def route_with_budget(query: str, remaining_budget: float) -> Strategy:
    if remaining_budget < 0.001:
        return Strategy.STANDARD  # Free tier — only cheap queries
    elif remaining_budget < 0.01:
        return self.classify(query)  # Normal routing
    else:
        return Strategy.COMPLEX  # Budget available — use best strategy
```

### Stretch 3: Multi-Model Consensus

For high-stakes queries (detected by query classifier), run the query through MULTIPLE strategies and take a consensus:

```python
class ConsensusEngine:
    """
    For high-stakes queries, run 3 strategies in parallel.
    If they agree → high confidence answer.
    If they disagree → run a meta-evaluation, flag for review.
    
    High-stakes triggers:
    - Query contains "security", "compliance", "legal", "guarantee"
    - Enterprise customer query
    - Query about pricing or contracts
    """
```

---

## 🚦 Project Gate Check

Before declaring Phase 5 complete:

- [ ] All 5 milestones are demonstrably working
- [ ] The engine handles all 10 edge case types from Module 07
- [ ] You can demonstrate at least ONE self-improvement cycle
- [ ] All 5 retrieval strategies have been compared and documented
- [ ] The README is portfolio-ready (architecture diagram, decisions, performance)
- [ ] The project is deployable (Docker Compose works)
- [ ] Regression tests pass
- [ ] You can articulate WHEN to use each strategy and WHY

### The Final Test

Run this query through your engine:

> *"I'm a senior manager in the Singapore office. The documentation says I might be eligible for the cross-regional assignment program and executive benefits. Can you tell me exactly what I qualify for, referencing the specific policies?"*

The system should:
1. Extract entities: ["senior manager", "Singapore office", "cross-regional assignment", "executive benefits"]
2. Identify as a COMPLEX query → routes to appropriate strategy(ies)
3. Find qualification criteria (manager-level → executive benefits)
4. Find Singapore's participation in cross-regional program
5. Find what executive benefits include
6. Synthesize a complete answer with specific policy references
7. Cite all sources
8. Self-evaluate: faithfulness ≥ 0.90

---

**Phase 5 Complete. You are now an Advanced RAG Engineer.**

Next: Phase 6 — AI Agents. Where RAG meets tool use, planning, and autonomous execution.
