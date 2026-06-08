# 09 — Project: Production Customer Support Bot

## 🎯 Project Overview

This is your Phase 4 capstone. You will build a **production-grade customer support chatbot** powered by RAG. This isn't a toy — it's a complete system with ingestion pipelines, retrieval, context optimization, evaluation, production deployment, and monitoring.

By the end of this project, you'll have a portfolio piece that demonstrates:
- **RAG architecture**: You designed and built a complete RAG system from first principles
- **Production engineering**: Caching, streaming, error handling, monitoring
- **Evaluation rigor**: You can measure and prove your system works
- **Engineering judgment**: You made tradeoff decisions and can explain them

### Learning Objectives

1. Design a complete RAG architecture from ingestion to production deployment
2. Build each component yourself before reaching for frameworks
3. Evaluate your system quantitatively and make data-driven improvements
4. Handle real-world edge cases: contradictions, missing information, multi-turn conversations
5. Deploy with observability, caching, and graceful degradation

### Time Budget

| Milestone | Estimated Time |
|-----------|---------------|
| Dataset preparation & ingestion | 3-4 hours |
| Milestone 1: Chunking & embedding | 4-6 hours |
| Milestone 2: Retrieval system | 4-6 hours |
| Milestone 3: Context & generation | 6-8 hours |
| Milestone 4: Evaluation & optimization | 6-8 hours |
| Milestone 5: Production deployment | 4-6 hours |
| Documentation & portfolio prep | 2-3 hours |
| **Total** | **30-40 hours** |

---

### ⏹ STOP. Before you start, answer these questions.

**Question 1 — Your Design Ethos**

*You're building a customer support bot. A user asks a question that's technically out of your knowledge base — your docs don't cover it. The LLM can probably answer it from its training data (it's a common question). Do you let the LLM answer from memory, or do you strictly refuse because the docs don't cover it?*

*There's no right answer — different companies make different choices. But YOUR bot needs a consistent policy. Write your policy in one sentence. Then write the system prompt instruction that enforces it.*

**Question 2 — Support vs. Sales**

*A user asks: "Which of your plans would be best for my team of 15 developers?" This is half support question (plan features) and half sales question (recommendation). Your knowledge base has detailed plan comparisons. Your RAG system retrieves the right docs. But the generated answer sounds robotic because it's just listing features.*

*What's missing from the RAG system to handle this kind of question well? Is it a retrieval problem (need different chunks), a context problem (need different prompt structure), or a generation problem (need different voice)? How would you design the system to detect and handle recommendation-style questions?*

**Question 3 — The Support Escalation Path**

*Your bot handles Tier 1 support (common questions, simple issues). When the bot can't help, it should escalate to a human. But users hate being transferred. They'd rather the bot "try harder."*

*Design your bot's escalation policy. What signals trigger escalation? (Not just "I don't know" — what are the subtle signals that a human is needed?) How does the bot hand off context to the human agent so the user doesn't have to repeat themselves? How do you measure whether your escalation rate is appropriate (too high = bot isn't useful, too low = users are frustrated)?*

**Write down your answers. These design decisions will shape every line of code you write.**

---

## 📋 Project Specification

### Dataset

You need a customer support knowledge base. You have several options:

**Option A: Use real public docs**
- Stripe API docs (https://stripe.com/docs/api)
- Twilio docs
- Docker docs
- FastAPI docs
- Any SaaS product with public documentation

**Option B: Synthetic dataset**
Create a synthetic support knowledge base covering a fictional SaaS product:
- Product documentation (features, specifications)
- Billing and pricing guide
- API reference
- FAQ
- Changelog (with version changes)
- Troubleshooting guide

**Option C: Your own product**
If you have a product with documentation, use that. This gives you domain expertise.

**Requirements:**
- At least 50 pages/documents of support content
- At least 3 document types (e.g., docs, pricing, guides)
- Content should have some internal contradictions (e.g., old vs. new pricing) for challenge
- Content should span multiple difficulty levels (simple FAQs, complex troubleshooting)

### Architecture

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   DOCUMENT       │     │   INGESTION      │     │   VECTOR STORE   │
│   SOURCE         │────▶│   PIPELINE       │────▶│   (Qdrant/       │
│   (files, web)   │     │   (chunk, embed) │     │    ChromaDB)     │
└──────────────────┘     └──────────────────┘     └──────────────────┘
                                                            │
                                                            │
┌──────────────────┐     ┌──────────────────┐              │
│   USER           │────▶│   RAG SERVER     │──────────────┘
│   (chat UI)      │     │   (FastAPI)      │
└──────────────────┘     │                  │
        ▲                │   - Retrieval    │
        │                │   - Context      │
        │                │   - Generation   │
        │                │   - Caching      │
        │                │   - Streaming    │
        │                │   - Evaluation   │
        │                └──────────────────┘
        │                           │
        │                ┌──────────────────┐
        └────────────────│   LLM API        │
                         │   (OpenAI/etc)   │
                         └──────────────────┘
```

### Tech Stack Options

**Required:**
- Python 3.11+
- FastAPI (API server)
- Pydantic (validation, settings)
- httpx or openai Python SDK (LLM API)
- A vector database (pick one):
  - **ChromaDB** (simplest, good for dev)
  - **Qdrant** (production-grade, preferred)
  - **pgvector** (if you want PostgreSQL integration)
- tiktoken (token counting)

**Optional:**
- Redis (caching)
- Langfuse (observability)
- Docker (containerization)
- Next.js (if you want a chat UI — reuse your frontend skills)

---

## Milestone 1: Data Ingestion & Chunking

### Goal

Build the pipeline that ingests documents and prepares them for retrieval.

### Requirements

1. **Document loader**: Read documents from multiple formats (at least 2: markdown, HTML, plain text)
2. **Chunker**: Implement at least 3 chunking strategies:
   - Fixed-size (configurable token count with overlap)
   - Paragraph/section-based (respects document structure)
   - Recursive (attempts semantic boundaries, falls back to fixed-size)
3. **Metadata extraction**: For each chunk, extract and store:
   - Source document title
   - Chunk position in document (section heading)
   - Document type (docs, pricing, changelog, etc.)
   - Last updated date (from document metadata)
4. **Embedding**: Embed all chunks using an embedding model
5. **Indexing**: Store embeddings in your chosen vector database

### Edge Cases to Handle

- Document with no clear section breaks (how do you chunk?)
- Very small document (shorter than chunk_size — single chunk)
- Very large document (thousands of pages — progressive loading)
- Document with tables, code blocks, lists (structure-aware chunking)
- Duplicate documents (how do you detect and handle?)

### Deliverables

```python
# ingestion_pipeline.py
class DocumentLoader:
    def load(self, path: str) -> List[Document]:
        """Load documents from file or directory."""
        pass

class Chunker:
    def chunk(self, document: Document) -> List[Chunk]:
        """Chunk a document into pieces."""
        pass

    def chunk_fixed_size(self, text: str, size: int, overlap: int) -> List[str]:
        """Fixed-size chunking."""
        pass

    def chunk_by_section(self, text: str) -> List[str]:
        """Section-based chunking."""
        pass

    def chunk_recursive(self, text: str, target_size: int) -> List[str]:
        """Recursive chunking with semantic boundaries."""
        pass

class EmbeddingService:
    async def embed(self, texts: List[str]) -> List[List[float]]:
        """Embed texts using the embedding API."""
        pass

class IndexingPipeline:
    async def run(self, source_path: str):
        """Full pipeline: load → chunk → embed → index."""
        pass
```

### Validation

After completing this milestone:
- [ ] At least 200 chunks indexed
- [ ] At least 3 different chunking strategies implemented
- [ ] Chunks have complete metadata (title, position, type, date)
- [ ] No chunks are empty or corrupted
- [ ] Ingestion handles errors gracefully (bad files, API failures, duplicates)
- [ ] Ingestion is repeatable (re-running doesn't duplicate)

### Discovery Questions

After implementing, ask yourself:
- Which chunking strategy produces the most useful chunks for YOUR documents?
- How did you handle documents that didn't fit your chunking assumptions?
- What metadata turned out to be most useful for retrieval later?

---

## Milestone 2: Retrieval System

### Goal

Build a multi-strategy retrieval system that finds the right chunks for any query.

### Requirements

1. **Dense retrieval**: Vector similarity search using embeddings
2. **Sparse retrieval**: Keyword-based search (BM25 or TF-IDF) as fallback
3. **Hybrid fusion**: Combine dense and sparse results with configurable weighting
4. **Metadata filtering**: Support filtering by document type, date, or category
5. **Query rewriting**: Optional query expansion/rewriting for better retrieval
6. **Top-K configurability**: Support different numbers of chunks per query type

### Implementation

```python
# retrieval_system.py
class DenseRetriever:
    """Vector similarity-based retrieval."""

    async def retrieve(self, query: str, top_k: int = 5, filters: Dict = None) -> List[Chunk]:
        """Retrieve chunks by vector similarity."""
        pass


class SparseRetriever:
    """Keyword-based retrieval (BM25)."""

    async def retrieve(self, query: str, top_k: int = 5) -> List[Chunk]:
        """Retrieve chunks by keyword matching."""
        pass


class HybridRetriever:
    """Combine dense and sparse retrieval results."""

    def __init__(self, dense_weight: float = 0.7, sparse_weight: float = 0.3):
        self.dense_weight = dense_weight
        self.sparse_weight = sparse_weight

    async def retrieve(
        self, query: str, top_k: int = 5, filters: Dict = None
    ) -> List[Chunk]:
        """Retrieve using hybrid fusion."""
        # Run dense + sparse in parallel
        # Fuse results with weighted scoring
        # Apply filters
        # Return top-k
        pass


class QueryRewriter:
    """Rewrite queries for better retrieval."""

    async def rewrite(self, query: str, conversation_history: List[str] = None) -> str:
        """Rewrite query to improve retrieval quality."""
        pass
```

### Edge Cases to Handle

- Query with no matching results (zero-shot retrieval)
- Query that's too short (1-2 words) — expand or flag?
- Query that's too long (entire paragraph) — summarize first?
- Query mixing multiple intents ("How do I reset my password and update billing?")
- Query in different language than documents
- Query asking about something that doesn't exist in any document

### Validation

After completing this milestone:
- [ ] Dense retrieval returns relevant results for 8/10 test queries
- [ ] Sparse retrieval catches results dense misses (at least 2 test cases)
- [ ] Hybrid fusion outperforms either individual method (measured)
- [ ] Metadata filtering works correctly (returns only documents matching filter)
- [ ] Retrieval latency < 500ms (including embedding time)
- [ ] Zero-results case returns gracefully (no crash, meaningful message)

### Tuning

- Experiment with dense_weight (0.3, 0.5, 0.7, 0.9) — which works best for YOUR data?
- Experiment with top-K (3, 5, 8, 10) — at what point does adding more chunks hurt?
- Test retrieval with and without query rewriting — measure the improvement

---

## Milestone 3: Context Integration & Generation

### Goal

Build the "A" (Augment) and "G" (Generation) steps — assemble optimal context and generate answers.

### Requirements

1. **Context builder**: Assemble retrieved chunks into an optimized prompt
   - Chunk ordering: best first, rest sorted, second-best last (Lost in the Middle mitigation)
   - Citation formatting: clear [SOURCE N] markers
   - Token budget management: dynamic chunk selection within limits
   - Context exhaustion notes: indicate when chunks were excluded
2. **System prompt**: A well-crafted system prompt that:
   - Sets expectations for how to use sources
   - Handles contradictions explicitly
   - Defines the "I don't know" policy
   - Specifies output format
3. **Conversation handler**: For multi-turn conversations
   - History compression (summarize older turns)
   - Context carry-over (what info from prior turns matters)
   - Resolve cross-turn contradictions
4. **Citation extractor**: Parse citations from the generated answer and verify them

### Implementation

```python
# context_integration.py
class ContextBuilder:
    def build(
        self,
        chunks: List[Chunk],
        query: str,
        max_tokens: int = 3000,
        conversation_history: List[Dict] = None
    ) -> str:
        """Build optimized context prompt."""
        pass

    def position_chunks(self, chunks: List[Chunk]) -> List[Chunk]:
        """Position chunks to mitigate Lost in the Middle."""
        pass

    def format_citations(self, chunks: List[Chunk]) -> str:
        """Format chunks with citation markers."""
        pass

    def select_chunks_for_budget(
        self, chunks: List[Chunk], budget_tokens: int
    ) -> List[Chunk]:
        """Select chunks within token budget."""
        pass


# generation.py
class AnswerGenerator:
    async def generate(
        self,
        query: str,
        context: str,
        system_prompt: str,
        stream: bool = False
    ) -> AnswerResult:
        """Generate answer using LLM."""
        pass

    async def generate_stream(
        self,
        query: str,
        context: str,
        system_prompt: str
    ) -> AsyncGenerator[str, None]:
        """Stream answer tokens."""
        pass


# citation_verifier.py
class CitationVerifier:
    def extract_citations(self, answer: str) -> List[Citation]:
        """Extract [SOURCE N] references from answer."""
        pass

    def verify_citations(
        self, citations: List[Citation], chunks: List[Chunk]
    ) -> VerificationResult:
        """Check that cited sources actually support the claims."""
        pass
```

### Edge Cases to Handle

- Context + query exceed token budget (truncate gracefully)
- Single chunk retrieved (skip positioning complexity)
- Zero chunks retrieved (fallback message, no LLM call)
- Conversation history too long (compress oldest turns)
- User asks about something OUTSIDE the knowledge base
- Answer contains hallucinated citation (verify and flag)

### System Prompt Design

Create 3 versions of your system prompt and A/B test them:

```python
system_prompts = {
    "strict": """Answer ONLY using the provided sources. If sources don't
contain the answer, say so. Never use your own knowledge.""",

    "balanced": """Answer based on the provided sources. If the sources don't
contain the answer, you may use your own knowledge but clearly indicate
when you're doing so.""",

    "helpful_first": """Provide the most helpful answer possible. Use the
provided sources as your primary information. If you know the answer
from your training and the sources don't contradict, you may include
that information with appropriate context."""
}
```

**Your task**: Choose ONE and justify your choice with evaluation data.

### Validation

After completing this milestone:
- [ ] Context builder produces well-structured prompts with citations
- [ ] Token budget management works (no over-limit errors)
- [ ] System prompt produces desired behavior (you can test this)
- [ ] Multi-turn conversations work (at least 5-turn test)
- [ ] Citation extraction works on sample answers
- [ ] Empty retrieval case returns graceful message

---

## Milestone 4: Evaluation & Optimization

### Goal

Build a rigorous evaluation framework and use it to optimize your system.

### Requirements

1. **Evaluation dataset**: Create at least 30 test queries with gold answers
   - 10 easy (single chunk, direct answer)
   - 10 medium (2-3 chunks, some synthesis)
   - 10 hard (cross-document, contradictions, multi-step)
2. **Metrics implementation**: Implement at least 3 metrics
   - Faithfulness (LLM-as-judge)
   - Answer relevance
   - Context precision (or recall)
3. **A/B testing framework**: Compare configurations
   - Different chunk sizes
   - Different retrieval strategies (dense vs. hybrid)
   - Different models (gpt-4o-mini vs. gpt-4o)
   - Different context strategies (with/without position optimization)
4. **Regression detection**: Compare results across runs, flag degradations

### What to Optimize

Run experiments to determine:

1. **Optimal chunk size**: Test [200, 500, 1000] tokens. Measure faithfulness and recall.
2. **Optimal top-K**: Test [3, 5, 8]. Measure precision@K and answer completeness.
3. **Dense vs. hybrid**: Compare retrieval quality with and without BM25 fusion.
4. **With vs. without position optimization**: Does Lost in the Middle mitigation help?
5. **Model comparison**: gpt-4o-mini vs. gpt-4o — is the quality difference worth the cost?

### Implementation

```python
# evaluation.py
class Evaluator:
    def __init__(self, eval_dataset: List[EvalExample]):
        self.dataset = eval_dataset
        self.results = []

    async def evaluate(self, rag_system) -> EvalReport:
        """Run full evaluation on a RAG system."""
        pass

    def compare(self, report_a: EvalReport, report_b: EvalReport) -> ComparisonReport:
        """Compare two evaluation runs."""
        pass


# experiments.py
class ExperimentRunner:
    """Run A/B experiments on RAG configurations."""

    async def experiment_chunk_size(self, sizes: List[int]) -> List[EvalReport]:
        """Compare different chunk sizes."""
        pass

    async def experiment_retrieval_strategy(self) -> ComparisonReport:
        """Compare dense vs. hybrid retrieval."""
        pass

    async def experiment_model_selection(self) -> ComparisonReport:
        """Compare different LLM models."""
        pass
```

### Validation

After completing this milestone:
- [ ] Evaluation dataset has 30+ queries across 3 difficulty levels
- [ ] At least 3 metrics implemented and producing reasonable scores
- [ ] At least 3 A/B experiments completed with documented results
- [ ] You can identify which configuration performs best for YOUR data
- [ ] You have evidence (not intuition) for your design decisions

### Document Your Findings

Create a table:

| Experiment | Config A | Config B | Winner | Faithfulness | Latency | Cost |
|-----------|----------|----------|--------|-------------|---------|------|
| Chunk size | 200 | 500 | ? | ? | ? | ? |
| Top-K | 3 | 5 | ? | ? | ? | ? |
| Retrieval | Dense | Hybrid | ? | ? | ? | ? |
| Position opt | No | Yes | ? | ? | ? | ? |
| Model | mini | full | ? | ? | ? | ? |

---

## Milestone 5: Production Deployment

### Goal

Deploy your RAG system with production-grade infrastructure.

### Requirements

1. **REST API**: FastAPI server with endpoints:
   - `POST /api/chat` — Non-streaming chat
   - `POST /api/chat/stream` — Streaming chat (SSE)
   - `GET /api/health` — Health check
   - `POST /api/feedback` — User feedback collection
2. **Caching**: At least exact query cache (Redis or in-memory)
3. **Streaming**: SSE-based response streaming
4. **Error handling**: Graceful degradation for all failure modes
5. **Logging**: Structured JSON logging for every query
6. **Configuration**: Environment-based config (dev/staging/production)

### Production Hardening

```python
# config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    app_name: str = "Customer Support Bot"
    debug: bool = False

    # Model config
    llm_model: str = "gpt-4o-mini"
    eval_model: str = "gpt-4o"
    embedding_model: str = "text-embedding-3-small"

    # RAG config
    chunk_size: int = 500
    chunk_overlap: int = 50
    top_k: int = 5
    max_context_tokens: int = 3000
    dense_weight: float = 0.7

    # API keys (from environment)
    openai_api_key: str
    qdrant_url: Optional[str] = None
    redis_url: Optional[str] = None

    # Production
    cache_ttl_seconds: int = 3600
    max_history_turns: int = 10
    enable_streaming: bool = True
    daily_budget_usd: float = 100.0

    class Config:
        env_file = ".env"
```

### Deployment Options

**Local development:**
```bash
# Run with uvicorn
uvicorn app.main:app --reload --port 8000
```

**Docker:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Monitoring

Implement at minimum:
- Health endpoint (`/api/health`)
- Query logging (structured JSON stdout)
- Feedback collection endpoint
- Basic latency tracking

### Validation

After completing this milestone:
- [ ] API starts and responds to health checks
- [ ] Non-streaming chat returns answers with sources
- [ ] Streaming chat returns tokens progressively
- [ ] Cache returns cached responses (measurably faster)
- [ ] Server handles errors gracefully (wrong input, missing API key)
- [ ] Logs contain structured data for every query
- [ ] Configuration can be changed via environment variables

---

## Stretch Goals

If you complete the milestones and want more:

### Stretch 1: Multi-Provider Support
Support multiple LLM providers (OpenAI, Anthropic, Together) with automatic failover. If OpenAI is rate-limited, fall back to Anthropic.

### Stretch 2: Reranking
Implement a cross-encoder reranker between retrieval and generation. Compare retrieval quality with and without reranking.

### Stretch 3: Advanced Evaluation
Implement all 4 RAGAS metrics (not just 3) and build a dashboard that tracks them over time.

### Stretch 4: Chat UI
Build a simple chat frontend (Next.js or plain HTML/JS) that:
- Shows streaming responses
- Displays source citations inline
- Allows feedback (thumbs up/down)
- Shows conversation history

### Stretch 5: Automated CI/CD
Set up a GitHub Actions pipeline that:
- Runs evaluation on every PR
- Compares against baseline
- Blocks PRs that degrade quality metrics
- Deploys automatically on merge to main

### Stretch 6: Semantic Cache
Implement semantic caching (embedding-based similarity lookup) in addition to exact cache.

### Stretch 7: A/B Testing in Production
Implement a system that routes 50% of traffic to Config A and 50% to Config B, then compares user satisfaction metrics.

---

## 📦 Deliverables Checklist

### Repository Structure

```
customer-support-bot/
├── README.md                    # Project overview, architecture, setup
├── requirements.txt             # Python dependencies
├── .env.example                 # Environment variable template
├── docker-compose.yml           # (optional) Container orchestration
│
├── app/
│   ├── __init__.py
│   ├── main.py                  # FastAPI app entry point
│   ├── config.py                # Pydantic settings
│   ├── ingestion/               # Milestone 1
│   │   ├── loader.py            # Document loading
│   │   ├── chunker.py           # Chunking strategies
│   │   ├── embedder.py          # Embedding service
│   │   └── pipeline.py          # Ingestion orchestrator
│   ├── retrieval/               # Milestone 2
│   │   ├── dense.py             # Dense vector retrieval
│   │   ├── sparse.py            # BM25 retrieval
│   │   ├── hybrid.py            # Hybrid fusion
│   │   └── rewriter.py          # Query rewriting
│   ├── context/                 # Milestone 3
│   │   ├── builder.py           # Context assembly
│   │   ├── templates.py         # Prompt templates
│   │   └── conversation.py      # Multi-turn handling
│   ├── generation/              # Milestone 3
│   │   ├── generator.py         # LLM answer generation
│   │   ├── streaming.py         # Streaming support
│   │   └── citations.py         # Citation extraction & verification
│   ├── evaluation/              # Milestone 4
│   │   ├── metrics.py           # Faithfulness, relevance, etc.
│   │   ├── dataset.py           # Eval dataset management
│   │   ├── runner.py            # Evaluation orchestrator
│   │   └── experiments.py       # A/B test framework
│   └── production/              # Milestone 5
│       ├── cache.py             # Caching layer
│       ├── logging.py           # Structured logging
│       ├── monitoring.py        # Metrics collection
│       └── resilience.py        # Retry, circuit breaker
│
├── data/
│   ├── documents/               # Source documents
│   │   ├── docs/                # Product documentation
│   │   ├── pricing/             # Pricing guides
│   │   ├── changelog/           # Version history
│   │   └── faq/                 # FAQ documents
│   └── evaluation/              # Eval dataset
│       ├── queries.json         # Test queries with gold answers
│       └── results/             # Evaluation results
│
├── tests/
│   ├── test_chunker.py
│   ├── test_retrieval.py
│   ├── test_context.py
│   ├── test_generation.py
│   └── test_evaluation.py
│
└── scripts/
    ├── ingest.py                # One-shot ingestion
    ├── evaluate.py              # Run evaluation
    ├── ab_test.py               # Run A/B experiment
    └── seed_cache.py            # Pre-warm cache
```

### What to Submit

1. **Working code**: All milestones implemented
2. **README**: Architecture, setup instructions, design decisions, tradeoffs
3. **Evaluation report**: Results from Milestone 4 with data-driven conclusions
4. **Sample conversations**: 5-10 example Q&A pairs showing your bot in action
5. **Post-mortem**: One thing that went wrong and how you fixed it

---

## 🚦 Success Criteria

Your project passes when you can demonstrate:

| Criteria | Baseline | Target | Stretch |
|----------|----------|--------|---------|
| **Total queries in eval set** | 15 | 30 | 50+ |
| **Avg faithfulness** | 0.80 | 0.88 | 0.93 |
| **Avg answer relevance** | 0.75 | 0.85 | 0.90 |
| **p95 latency** | <5s | <3s | <1.5s |
| **Cache hit rate** | 10% | 25% | 40%+ |
| **Multi-turn handling** | 3 turns | 8 turns | 15 turns |
| **Error rate** | <5% | <1% | <0.1% |
| **Citation accuracy** | 70% | 85% | 95% |

### Must-Have Features

- [ ] Ingestion pipeline with at least 3 chunking strategies
- [ ] Hybrid retrieval (dense + sparse with fusion)
- [ ] Context optimization (positioning, citations, token budget)
- [ ] Multi-turn conversation support
- [ ] Streaming responses
- [ ] Evaluation framework with 3+ metrics
- [ ] At least 3 A/B experiments with documented results
- [ ] Caching (at minimum: exact query cache)
- [ ] Graceful error handling for all failure modes
- [ ] Structured logging

### Nice-to-Have Features

- [ ] Semantic cache
- [ ] Reranking
- [ ] Chat UI
- [ ] Docker deployment
- [ ] CI/CD pipeline with evaluation gates
- [ ] A/B testing in production
- [ ] Multi-provider LLM support with failover

---

## 📚 Resources

### Reference Implementations
- The Phase 4 modules (01-08) — Your primary reference for HOW to build each component
- Phase 3 project (Knowledge Search Engine) — Retrieval patterns carry over
- Phase 1 project (LLM Gateway) — Provider abstraction, streaming patterns
- Phase 2 project (Prompt Engineering System) — System prompt design patterns

### Example Support Knowledge Bases
- Stripe API Reference: https://stripe.com/docs/api
- Twilio Docs: https://www.twilio.com/docs
- Docker Docs: https://docs.docker.com
- FastAPI Docs: https://fastapi.tiangolo.com

### Architecture Inspiration
- Glean's RAG architecture (blog)
- Notion AI's approach to citation
- Intercom's Fin (customer support AI) — observe their patterns

### Tools to Consider
- **Qdrant**: Self-hosted vector DB, excellent for production
- **Langfuse**: Open-source LLM observability
- **Prometheus + Grafana**: For production monitoring
- **Docker Compose**: For orchestrating multiple services

---

## 🎓 Portfolio Notes

This project demonstrates:

- **Full-stack RAG**: You built every component from ingestion to production deployment
- **System design**: You made architecture decisions and can explain tradeoffs
- **Evaluation rigor**: You measured your system's performance, not just demoed it
- **Production engineering**: Caching, streaming, error handling, monitoring
- **Technical writing**: Your README and documentation explain a complex system clearly

**In an interview, you can say:**
> "I built a production customer support bot using RAG. I implemented chunking, hybrid retrieval, context optimization, and LLM generation from scratch before adding any framework. I built an evaluation pipeline with 3 automated metrics and ran 5 experiments to optimize chunk size, retrieval strategy, and model selection. The system handles multi-turn conversations, caches responses, streams answers, and degrades gracefully when components fail."

**Then you show them the repo.**

---

## 🚦 Final Gate

Before declaring Phase 4 complete, verify:

1. **All 5 milestones are implemented** (even if some are minimal)
2. **Your evaluation shows the system works** (metrics meet baseline thresholds)
3. **You can demonstrate 3 sample conversations** that show different capabilities
4. **You wrote a README** that someone could use to understand and run your system
5. **You can answer**: What's the weakest part of your system and what would you improve next?

**What's next:** Phase 5 (Advanced RAG) builds directly on this project. You'll add HyDE, agentic retrieval, GraphRAG, and multimodal capabilities — to this exact customer support bot. Save your code well.

---

*"The projects in this course aren't exercises. They're the first entries in your AI engineering portfolio. Build them like you're shipping to production — because one day, you'll point a hiring manager at this repo."*
