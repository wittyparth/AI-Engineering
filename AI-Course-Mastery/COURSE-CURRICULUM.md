# 📖 AI Engineering Mastery — Complete Curriculum

**11 Phases · ~90 Topic Files · 11 Portfolio Projects · ~8 Months**

---

# Phase 0: AI Engineering Mindset
**⏱ 8–10 hours · 7 files**

> Install the mental models of a senior AI engineer before writing code.

## 00-Module-Overview.md
- Phase goals: think like a senior AI engineer, evaluation-first, cost-aware
- 5-day daily schedule with time budgets
- Build Loop concept introduction
- Gate Check requirements

## 01-Senior-vs-Junior-Mindset.md
- **Sub-topics:** Core difference between junior and senior AI engineers
- Junior: framework-focused, "works on my laptop", vibes-based success
- Senior: owns production outcomes, asks about failure modes upfront
- Real examples: building RAG with LangChain vs owning a production RAG system
- Framework proficiency ≠ seniority
- **🛠 Skills:** Mental models, production thinking, architectural awareness

## 02-Production-vs-Academic-Thinking.md
- **Sub-topics:** The gap between benchmarks and production reality
- The 99% trap — academic benchmarks don't measure production failure modes
- Case studies: what breaks in production that doesn't break in notebooks
- Production-think exercises — questions to ask before writing code
- "Works on my machine" gap analysis
- **🛠 Skills:** Critical evaluation, skepticism toward benchmarks, failure mode analysis

## 03-Evaluation-First-Philosophy.md
- **Sub-topics:** "If you can't measure it, you can't ship it"
- Why evaluation is THE senior skill — most AI engineers skip it
- What to evaluate at each layer: retrieval → generation → agent → system
- The evaluation pipeline: dataset → run → score → decide
- Cost of NOT evaluating: silent degradation, regressions, bad data
- Evaluation is infrastructure, not a project
- **🛠 Skills:** Eval design, metric selection, quality measurement

## 04-Decision-Framework.md
- **Sub-topics:** When to build raw vs use frameworks vs buy
- Decision tree: complexity, team size, customization needs, switching cost
- When to build raw (even in production): core differentiators, debugging
- When to use a framework: boilerplate, commoditized layers, speed
- Framework maturity model: toy → team → org → ecosystem
- Course pattern: always build raw first, then add frameworks
- **🛠 Skills:** Architectural decision-making, tradeoff analysis, framework evaluation

## 05-Cost-Awareness-Culture.md
- **Sub-topics:** Token economics — every token costs money
- Cost breakdown: input vs output tokens, caching vs fresh generate
- Cost optimization levers: model selection, prompt length, caching, batching
- The Model Cascade Pattern (Klarna, Uber): try cheap model first, escalate
- Cost tracking as non-negotiable infrastructure
- Cost is a feature, not an afterthought
- **🛠 Skills:** Cost modeling, optimization strategy, budget management

## 06-The-Build-Loop.md
- **Sub-topics:** The 6-step loop used in EVERY phase
  1. Problem — define the concrete problem
  2. Build Raw — implement without frameworks
  3. Add Framework — see how frameworks solve the pain
  4. Productionize — monitoring, cost, security, scaling
  5. Evaluate — measure quality metrics
  6. Debug — find and fix failures
- What each step produces as an artifact
- Why this builds transferable intelligence (not framework knowledge)
- **🛠 Skills:** Structured learning methodology, systematic building

---

# Phase 1: Foundations — LLM Gateway
**⏱ 20–25 hours · 13 files**

> How LLMs work, tokenization, API integration, streaming + Build a Multi-Provider LLM Gateway.

## 00-Module-Overview.md
- Phase goals, 10-day schedule, build path overview
- Framework integration: LiteLLM
- Gate Check requirements

## 01-How-LLMs-Work.md
- **Sub-topics:** Token prediction, autoregressive generation
- Training pipeline conceptually: pretraining → instruction tuning → RLHF
- What an LLM CANNOT do: count, reason reliably, know its own limits
- Key implications for engineering: probabilistic ≠ deterministic, output quality varies
- **🛠 Skills:** LLM fundamentals, realistic expectations, system design constraints

## 02-Tokenizers-Deep-Dive.md
- **Sub-topics:** What is a tokenizer? BPE, WordPiece, SentencePiece
- Critical facts: tokens ≠ words, token count is bidirectional, tokenizers are model-specific
- Practical engineering: counting tokens before sending, token budgeting
- Common traps: multi-byte characters, code formatting, numbered lists
- **🛠 Skills:** Token estimation, cost calculation, prompt length optimization

## 03-Context-Windows-Attention.md
- **Sub-topics:** What is a context window? Attention mechanism conceptually
- The attention problem: quadratic complexity, lost-in-the-middle
- Practical strategies: position matters, context budgeting, sliding windows
- Positional encoding breakdown: absolute vs relative (RoPE, ALiBi)
- **🛠 Skills:** Context management, attention understanding, LLM memory model

## 04-Model-Families-Tradeoffs.md
- **Sub-topics:** Provider landscape 2026 — frontier models (GPT-4o, Claude 4, Gemini 2.5)
- Open-source models: Llama 3, Mistral, Phi, DeepSeek
- Decision framework: capability vs speed vs cost vs control vs latency
- Model Router Pattern: classify query → route to right model
- **🛠 Skills:** Model selection, provider evaluation, routing architecture

## 05-Structured-Outputs-Basics.md
- **Sub-topics:** Three strategies — prompt engineering (fragile)
- Function/tool calling (better): works across providers
- Structured output / constrained decoding (best): guaranteed schema
- Validation + retry pattern: check output, retry with error feedback
- Reliability hierarchy: prompt → function → constrained decoding
- **🛠 Skills:** Structured output design, validation patterns, schema enforcement

## 06-Python-AI-Patterns.md
- **Sub-topics:** Async patterns for AI workloads
- Retry with exponential backoff + jitter
- Rate limiting — token bucket, sliding window
- Streaming patterns — async generators, partial parsing
- Error handling for LLM API calls (timeouts, rate limits, server errors)
- **🛠 Skills:** Async Python, resilient API clients, error handling patterns

## 07-API-Integration-Patterns.md
- **Sub-topics:** Unified client pattern across providers
- Retry strategies: exponential backoff with jitter
- Rate limiting: client-side vs server-side, token bucket algorithm
- Streaming patterns: SSE, async iterators, buffer management
- **🛠 Skills:** Multi-provider integration, client design, rate management

## 08-Streaming-SSE-WebSockets.md
- **Sub-topics:** Why streaming matters — perceived latency, early display
- Server-Sent Events (SSE): implementation, FastAPI StreamingResponse
- WebSockets: bidirectional streaming, use cases
- Streaming gotchas: token-level ≠ character-level, error handling during stream
- Connection management, backpressure, cost tracking during stream
- **🛠 Skills:** Streaming architecture, SSE implementation, real-time UI integration

## 09-AI-Production-Framework-Landscape.md
- **Sub-topics:** Framework categories overview
  - Agent frameworks: LangGraph, Pydantic AI, CrewAI, AutoGen
  - RAG & data: LangChain, LlamaIndex, Haystack
  - LLM gateways: LiteLLM, Portkey, Openrouter
  - LLM serving: vLLM, Ollama, TGI, Llama Stack
  - Prompt optimization: DSPy, Weave, PromptLayer
  - Observability: Langfuse, LangSmith, Logfire, Prometheus
- Decision tree for framework selection
- How this course uses frameworks integrated into build narrative
- **🛠 Skills:** Framework landscape awareness, tool selection, ecosystem navigation

## Drill-Build-CLI-Chat.md
- **Sub-topics:** Build CLI chat client with streaming responses
- Async input/output, event loop, signal handling
- Extension: multi-turn conversations, provider switching
- **🛠 Skills:** CLI development, async I/O, LLM API integration

## Drill-Tokenizer-Explorer.md
- **Sub-topics:** Explore how different text gets tokenized
- Compare token counts across languages, formats, content types
- Build cost estimation table
- **🛠 Skills:** Tokenization analysis, cost estimation

## Project: Multi-Provider LLM Gateway
- **Sub-topics:** FastAPI service routing to OpenAI, Anthropic, Ollama
- LiteLLM Router with automatic fallback, cost tracking
- SSE streaming, unified OpenAI-compatible API
- Rate limiting, retry logic, cost budgets per endpoint
- **🛠 Skills:** API gateway architecture, multi-provider abstraction, production FastAPI

---

# Phase 2: Prompt Engineering — Systematic & Scientific
**⏱ 10–14 days · 10 files**

> Engineer prompts like code — with metrics, versioning, and automated optimization.

## 00-Module-Overview.md
- Phase goals, 14-day schedule
- Prompt Engineering Hierarchy: Level 1 (copy-paste) → Level 5 (adaptive)
- Framework integration: DSPy
- Portfolio project: Prompt Engineering System

## 01-Context-Engineering.md
- **Sub-topics:** The anatomy of a prompt — instruction, context, input, output format
- Why context matters more than instructions
- Attention distribution across prompt sections
- Context engineering principles: specificity, relevance, structure
- **🛠 Skills:** Prompt architecture, context design, attention-aware prompting

## 02-Techniques-Deep-Dive.md
- **Sub-topics:** Zero-shot prompting, strengths and weaknesses
- Few-shot prompting: example selection, ordering, label balance
- Chain-of-Thought (CoT): why it works, when it fails
- Structured output prompting
- When each technique works — decision matrix
- **🛠 Skills:** Technique selection, prompt strategy, output quality

## 03-System-Prompts-Personas.md
- **Sub-topics:** System prompt design — role, constraints, behavior
- Persona engineering: expertise level, tone, constraints
- Behavioral guardrails: what the model MUST and MUST NOT do
- System prompt patterns: role + context + constraints + output spec
- **🛠 Skills:** System prompt design, persona crafting, behavioral control

## 04-Advanced-Techniques.md
- **Sub-topics:** Self-consistency — sample multiple CoT paths, vote
- Tree-of-Thought (ToT): explore multiple reasoning branches
- Reflexion: LLM evaluates its own output, self-corrects
- Generated knowledge prompting: LLM generates facts before answering
- **🛠 Skills:** Advanced prompting patterns, reasoning enhancement, self-evaluation

## 05-Red-Teaming-Antipatterns.md
- **Sub-topics:** 9 prompt attack patterns (injection, jailbreak, leak, etc.)
- Common failure modes: overfitting to examples, recency bias, format brittleness
- Bias in prompts: order bias, label bias, persona bias
- Antipattern catalog with examples and fixes
- **🛠 Skills:** Red-teaming, failure mode identification, prompt hardening

## 06-Evaluation-AB-Testing.md
- **Sub-topics:** Unit tests for prompts — expected outputs, edge cases
- Regression test suite: versioned, auto-run, pass/fail per test
- A/B testing prompt variants with statistical significance
- The Prompt Test Harness — framework for automated prompt evaluation
- What to test: correctness, safety, style, latency, cost
- **🛠 Skills:** Eval design for prompts, A/B methodology, regression detection

## 07-DSPy-Optimization.md
- **Sub-topics:** DSPy philosophy: programming, not prompting
- Signatures: typed input/output definitions
- Modules: Predict, ChainOfThought, ReAct
- Optimizers: BootstrapFewShot, MIPROv2, Bayesian
- Assertions: runtime guards, validation constraints
- DSPy vs manual prompting: when each wins
- **🛠 Skills:** DSPy programming, automated optimization, prompt compilation

## 08-Specialized-Prompt-Patterns.md
- **Sub-topics:** Code generation prompts — constraints, language, style
- Reasoning prompts — step-by-step, verification, confidence
- Creative writing prompts — tone, structure, constraints
- Analysis prompts — structured output, comparison, synthesis
- **🛠 Skills:** Domain-specific prompting, pattern adaptation

## 09-Production-Prompt-Management.md
- **Sub-topics:** Prompt template registry — versioned, environment-aware
- File structure for prompt management: templates/, tests/, versions/
- YAML-based prompt definitions with metadata
- Deployment strategy: canary prompts, gradual rollout, rollback
- Prompt CI/CD: test on commit, gate on quality, deploy on pass
- **🛠 Skills:** Prompt ops, versioning, CI/CD for prompts, release management

## Project: Prompt Engineering System
- **Sub-topics:** Prompt template registry with versioning
- Input sanitization and injection detection (10+ attack patterns)
- DSPy compilation pipeline for automated optimization
- A/B testing of prompt variants with statistical significance
- **🛠 Skills:** Full-stack prompt management, production prompt infrastructure

---

# Phase 3: Embeddings & Vector Databases
**⏱ 12–16 hours · 9 files**

> Build a semantic search engine — from scratch to production.

## 00-Module-Overview.md
- Phase goals, build path overview
- Search quality philosophy: most RAG failures trace back to bad retrieval
- Gate Check requirements

## 01-Embeddings-Deep-Dive.md
- **Sub-topics:** What is an embedding? Dense vector representation of meaning
- How embeddings are created: tokenization → context → pooling → normalization
- Properties: semantic similarity, linear relationships, dimensionality
- Quality dimensions: richness, consistency, discrimination
- The embedding trap: good on benchmarks ≠ good on YOUR data
- **🛠 Skills:** Embedding understanding, quality assessment, vector representation

## 02-Semantic-Search-From-Scratch.md
- **Sub-topics:** Distance metrics — cosine similarity, Euclidean (L2), dot product
- Brute force search: O(n·d) complexity, fine for small datasets
- Inverted File Index (IVF): partition → search nearest centroids
- Build vector search from scratch: embeddings → index → search → rank
- **🛠 Skills:** Search algorithm implementation, distance metric selection

## 03-Embedding-Model-Selection.md
- **Sub-topics:** Model landscape 2026: OpenAI, Cohere, Voyage, sentence-transformers
- Step 1: Define requirements — domain, language, dimensionality, latency
- Step 2: Benchmark on YOUR data — recall@K, latency, cost
- The embedding trap revisited: leaderboard scores ≠ your use case
- **🛠 Skills:** Model evaluation, benchmark design, embedding selection

## 04-ChromaDB.md
- **Sub-topics:** Setup and configuration, collections management
- CRUD operations: add, update, delete, query
- Metadata filtering, batch operations
- Local development workflow: persist, load, reset
- **🛠 Skills:** Vector DB setup, local development, CRUD operations

## 05-Qdrant-Pinecone.md
- **Sub-topics:** Qdrant — HNSW index configuration, quantization, filtering
- Pinecone — serverless, pod-based, hybrid search
- Scaling considerations: millions of vectors, latency P95
- Production deployment: Docker, cloud, authentication
- **🛠 Skills:** Production vector DB deployment, scaling, configuration

## 06-pgvector.md
- **Sub-topics:** PostgreSQL as vector DB — extension setup
- Indexing: HNSW, IVF, exact search
- Hybrid search with SQL: WHERE + ORDER BY distance
- When pgvector is the right choice: already using Postgres, small-medium scale
- **🛠 Skills:** pgvector setup, SQL vector search, hybrid query patterns

## 07-Advanced-Vector-Search.md
- **Sub-topics:** Hybrid search — BM25 (keyword) + dense (semantic) fusion
- Reciprocal Rank Fusion (RRF): combine rankings from multiple strategies
- Cold start problem: no labeled data, no query logs
- When to use each search strategy: precision vs recall tradeoffs
- **🛠 Skills:** Hybrid search implementation, ranking fusion, cold start handling

## 08-Drills-Embeddings.md
- **Sub-topics:** Build vector search from scratch (no FAISS, no Qdrant)
- Embedding benchmark — compare models on your data
- Report template: recall@K, precision@K, latency, cost
- **🛠 Skills:** Hands-on embedding/search implementation, benchmarking

## Project: Knowledge Search Engine
- **Sub-topics:** FastAPI service with multiple embedding model support
- HNSW index for fast approximate search (FAISS/Qdrant)
- Hybrid search (semantic + BM25 via RRF)
- Search quality dashboard: recall@K, precision@K, latency P95
- A/B comparison of embedding models
- **🛠 Skills:** Production search system, evaluation dashboard, multi-model architecture

---

# Phase 4: RAG Foundations
**⏱ 20–25 hours · 9 files**

> Make LLMs answer from YOUR data — build RAG 3 ways (raw, LangChain, LlamaIndex).

## 00-Module-Overview.md
- Phase goals, 3-implementation approach
- RAG is NOT search — retrieval quality bounds generation quality
- Framework integration: LangChain, LlamaIndex

## 01-What-Is-RAG.md
- **Sub-topics:** The full RAG pipeline: ingest → chunk → embed → retrieve → generate
- The RAG tax: retrieval latency, context window management, cost per query
- When to use LangChain vs LlamaIndex: prototyping speed vs data-centric
- **🛠 Skills:** RAG architecture understanding, framework selection

## 02-Chunking-Strategies.md
- **Sub-topics:** Why chunking is the single most impactful RAG parameter
- The tradeoff: small chunks (precision) vs large chunks (context)
- Strategy 1: Recursive character split (good baseline, 512-1024 tokens, 10-20% overlap)
- Strategy 2: Semantic chunking (better) — embedding similarity to detect topic shifts
- Strategy 3: Document structure-aware (best) — headers, sections, paragraphs
- Chunking for different content: code, prose, tables, mixed media
- **🛠 Skills:** Chunking strategy design, parameter tuning, content-aware splitting

## 03-Retrieval-Deep-Dive.md
- **Sub-topics:** Embedding → similarity search → top-K selection
- Ingestion pipelines: source handlers, document model, orchestration
- Ingestion failure modes: corrupted files, encoding issues, duplicate detection
- Metadata extraction and filtering
- **🛠 Skills:** Ingestion pipeline design, extraction, metadata management

## 04-Context-Integration.md
- **Sub-topics:** Prompt construction for RAG — instruction + context + query
- From-scratch RAG implementation: chunk → embed → retrieve → construct prompt → generate
- The vanilla RAG trap: no reranking, no query transformation, no evaluation
- **🛠 Skills:** RAG prompt design, context assembly, raw RAG implementation

## 05-RAG-Failure-Modes.md
- **Sub-topics:** 5 common RAG failures
  1. Retrieval miss — relevant document not in top-K
  2. Bad chunks — chunk boundaries cut through key information
  3. Missing context — not enough relevant text in context window
  4. Hallucination — model ignores context and makes up answer
  5. Irrelevant context — retrieved chunks aren't related to query
- Diagnosis and mitigation for each failure mode
- **🛠 Skills:** Failure diagnosis, RAG debugging, systematic troubleshooting

## 06-Evaluating-RAG.md
- **Sub-topics:** RAGAS metrics: faithfulness, answer relevancy, context precision, context recall
- Building eval datasets: golden Q&A pairs, human annotation
- LLM-as-Judge: judge design, scoring rubric, bias mitigation
- Eval-driven development: set metric targets before writing code
- **🛠 Skills:** RAG evaluation, LLM-as-judge design, metric-driven development

## 07-Production-RAG.md
- **Sub-topics:** Reranking with cross-encoder for precision improvement
- Hybrid search for recall improvement (from Phase 3)
- Query transformation: rewriting, expansion, decomposition
- Caching: exact match + semantic caching for latency/cost
- Streaming generation for better UX
- **🛠 Skills:** Production RAG optimization, reranking, caching, streaming

## 08-Drills-RAG.md
- **Sub-topics:** Build RAG without LangChain — understand every component
- Chunking optimization lab — experiment with strategies, measure impact
- RAG with LangChain — rapid prototyping
- **🛠 Skills:** Raw framework implementation, experimental methodology

## Projects: RAG Framework Comparison + Production RAG System
- **Sub-topics:** Same RAG task implemented 3 ways — raw vs LangChain vs LlamaIndex
- Comparison: code complexity, flexibility, performance, debuggability
- Production RAG: multi-strategy chunking, hybrid search, reranking, query rewriting, RAGAS
- **🛠 Skills:** Framework evaluation, production RAG architecture, systematic comparison

---

# Phase 5: Advanced RAG
**⏱ 18–22 hours · 9 files**

> Beyond naive retrieval — HyDE, agentic RAG, GraphRAG, multimodal, self-RAG.

## 00-Module-Overview.md
- Phase goals: solve 5 problems (vocabulary mismatch, multi-hop, entity understanding, etc.)
- Framework integration: LlamaIndex Workflows
- Senior: RAG strategy selection — choose strategy per query

## 01-HyDE-Query-Transformation.md
- **Sub-topics:** HyDE — generate hypothetical document, embed that, search with it
- Multi-query — ask same question 5 ways, aggregate results
- Query decomposition — break complex question into sub-queries
- Strategy comparison: when each works best
- LlamaIndex Workflows integration
- **🛠 Skills:** Query transformation, HyDE implementation, multi-query strategies

## 02-Contextual-Retrieval.md
- **Sub-topics:** Context-aware chunking — embed chunk + surrounding context
- Parent-child retrieval: retrieve child chunk, return parent document
- Contextual embedding: prepend document summary to each chunk
- When contextual retrieval helps vs adds cost
- **🛠 Skills:** Contextual chunking, parent-child patterns, context enhancement

## 03-Reranking-Deep-Dive.md
- **Sub-topics:** Cross-encoder reranking — why it's more accurate than bi-encoder
- Cohere Rerank API vs local models (BAAI/bge-reranker)
- Performance impact: 10-20% recall improvement at top-5
- Reranking vs second retrieval — when to do each
- **🛠 Skills:** Reranking implementation, cross-encoder usage, precision optimization

## 04-Agentic-RAG.md
- **Sub-topics:** Agent decides WHEN and HOW to retrieve
- Agentic RAG loop: analyze query → select strategy → retrieve → evaluate → iterate
- Reflexion: agent reflects on retrieved info, decides if sufficient
- Retrieval strategies: vector, web, code, SQL, Graph
- When agentic RAG is NOT worth the complexity
- **🛠 Skills:** Agentic retrieval design, strategy selection, autonomous iteration

## 05-Multimodal-RAG.md
- **Sub-topics:** Three approaches ranked by quality
  1. Multimodal embedding models (Jina CLIP, nomic-embed-vision) — best balance
  2. Image captioning + text RAG — good fallback, no special infrastructure
  3. Multimodal LLM + raw images — most powerful, most expensive
- Cost analysis: each approach at 100K queries/month
- **🛠 Skills:** Multimodal architecture, embedding selection, cost modeling

## 06-GraphRAG.md
- **Sub-topics:** Knowledge graph construction — entity extraction, relationship mapping
- Graph query patterns: traversal, community detection, centrality
- Microsoft's GraphRAG pattern: entity extraction → community detection → summarization
- When Graph RAG vs Vector RAG: multi-hop vs similarity
- **🛠 Skills:** Entity extraction, graph construction, GraphRAG query patterns

## 07-Advanced-Evaluation.md
- **Sub-topics:** Multi-hop accuracy — can the system answer questions needing 2+ documents
- Retrieval precision at each step in multi-hop queries
- Cache hit rate for semantic caching
- Faithfulness evaluation for complex answers
- **🛠 Skills:** Multi-hop eval, advanced faithfulness, cache effectiveness measurement

## 08-Advanced-Drills.md
- **Sub-topics:** Hands-on implementation of advanced RAG patterns
- Agentic retrieval implementation from scratch
- Multi-hop query resolution patterns
- **🛠 Skills:** Advanced RAG implementation, pattern application

## Project: Advanced RAG Engine
- **Sub-topics:** Agentic research assistant with multi-strategy retrieval
- HyDE + multi-query for comprehensive coverage
- Self-RAG: evaluate retrieval, retry if insufficient
- Graph RAG for entity-relationship questions
- Semantic caching (40%+ cost reduction)
- **🛠 Skills:** Multi-strategy RAG architecture, cost optimization, advanced evaluation

---

# Phase 6: AI Agents
**⏱ 25–30 hours · 9 files**

> From RAG to autonomous action — ReAct, tools, memory, multi-agent, MCP.

## 00-Module-Overview.md
- Phase goals, agent vs RAG comparison table
- Agent Maturity Model: Level 1 (tool call) → Level 6 (self-improving)
- Framework integration: Pydantic AI, LangGraph, LiteLLM

## 01-Agent-Foundations.md
- **Sub-topics:** Single agent pattern — simplest, tool call → response
- Supervisor pattern — supervisor delegates to specialized agents
- Orchestrator-worker pattern — one orchestrator, many workers
- Pipeline pattern — sequential agent handoffs
- Swarm pattern — decentralized agents with shared goal
- Pattern selection guide: match pattern to task complexity
- **🛠 Skills:** Agent architecture design, pattern selection, orchestration understanding

## 02-ReAct-Loop.md
- **Sub-topics:** ReAct loop from scratch — observe → think → act cycle
- Thought generation: what to do next and why
- Action execution: call tool with structured parameters
- Observation: tool response → continue loop or produce final answer
- Failure modes: infinite loops, hallucinated tool calls, stuck states
- **🛠 Skills:** ReAct implementation, loop control, failure detection

## 03-Tool-Design.md
- **Sub-topics:** What makes a good tool — clear name, descriptive docstring, constrained params
- Tool schema: name, description, parameters (required/optional)
- Description engineering — the description is what the LLM uses to choose
- Parameter constraints: types, enums, ranges, regex
- Input validation at tool boundary
- **🛠 Skills:** Tool design, schema engineering, validation patterns

## 04-Memory-Systems.md
- **Sub-topics:** Short-term memory — conversation history within session
- Long-term memory — vector store across sessions, retrieval-augmented
- Episodic memory — specific past experiences, success/failure recall
- Memory-augmented agent architecture
- Memory strategy guide: what type for which use case
- **🛠 Skills:** Memory system design, vector memory, episodic recall, multi-tier architecture

## 05-LangGraph-Deep-Dive.md
- **Sub-topics:** Why state matters — agents need persistent, check-pointable state
- Basic graph: State → Nodes → Edges → Compile → Run
- State management: shared state, reducer patterns, branching
- Multi-agent with LangGraph: agent nodes, conditional routing, parallel execution
- Human-in-the-loop: interrupt graph, wait for approval, resume
- LangGraph vs custom state machines: when to use which
- **🛠 Skills:** LangGraph development, state graph design, HITL patterns

## 06-Multi-Agent-Systems.md
- **Sub-topics:** Communication patterns — direct messaging, shared blackboard, queue-based
- Orchestrator-worker example: decompose task, delegate, synthesize
- Handoff pattern (Anthropic MCP style): agent → context → next agent
- Multi-agent anti-patterns: over-delegation, circular dependencies, race conditions
- **🛠 Skills:** Multi-agent architecture, communication design, anti-pattern avoidance

## 07-MCP-Protocol.md
- **Sub-topics:** What is MCP? Model Context Protocol — standardized tool interface
- Core concepts: MCP Server (exposes tools/resources/prompts), MCP Client (connects)
- MCP + agent: agent discovers available MCP servers, uses their tools
- Building your own MCP server: tools, resources, prompts, transports
- When MCP vs custom tools: standardization, ecosystem, switching cost
- **🛠 Skills:** MCP implementation, server development, tool standardization

## 08-Agent-Evaluation.md
- **Sub-topics:** The hardest unsolved problem in AI engineering
- Evaluation dimensions:
  1. Task completion — binary or graded success
  2. Step-level evaluation — each action correct?
  3. Cost & efficiency — tokens/step, time/step, total cost
  4. Robustness testing — bad inputs, edge cases, failure recovery
- Eval as a service: trace → score → aggregate → alert
- **🛠 Skills:** Agent eval design, multi-dimensional assessment, automation

## Project: Multi-Agent Research System
- **Sub-topics:** 5 specialized agents — orchestrator, research, analysis, writing, fact-checking
- Pydantic AI for type-safe agent logic
- LangGraph for state management and checkpointing
- LiteLLM Router for cost-optimized model selection
- Human-in-the-loop for verification
- Agent evaluation pipeline
- **🛠 Skills:** Multi-agent system design, framework composition, production orchestration

---

# Phase 7: Production Evals & Observability
**⏱ 18–22 hours · 9 files**

> Know your AI actually works — continuously, measurably, at scale.

## 00-Module-Overview.md
- Phase goals: evals-first mindset, proactive not reactive
- Framework integration: Langfuse, Pydantic Logfire, RAGAS, Prometheus + Grafana

## 01-Evals-First-Mindset.md
- **Sub-topics:** Evals before implementation — design evaluation before writing code
- The eval pyramid: unit → component → integration → system → production
- Eval-driven development: metric targets → implementation → measure → iterate
- **🛠 Skills:** Eval-first design, test-driven AI development, quality culture

## 02-LLM-as-Judge.md
- **Sub-topics:** Judge design — prompt, rubric, scoring scale
- Bias mitigation: position bias, length bias, verbosity bias, self-enhancement
- Pairwise scoring vs absolute scoring
- Rubric-based evaluation: criteria, levels, examples
- Judge reliability: agreement rate, calibration, human correlation
- **🛠 Skills:** LLM-as-judge design, bias detection, rubric engineering, reliability measurement

## 03-Metrics-That-Matter.md
- **Sub-topics:** Performance metrics — latency (P50/P95/P99), throughput, error rates
- Quality metrics — user feedback, retrieval quality, generation quality
- Cost metrics — per-query, per-model, per-user, trends
- Composite metrics: cost per good outcome, quality-weighted throughput
- **🛠 Skills:** Metric selection, KPI definition, composite metric design

## 04-Evaluation-Datasets.md
- **Sub-topics:** Golden datasets — curated, verified, representative
- Synthetic data generation: using LLMs to create eval data
- Dataset versioning: track changes, rollback, diff between versions
- Dataset curation: deduplication, quality filtering, adversarial examples
- **🛠 Skills:** Eval dataset design, synthetic generation, versioning, curation

## 05-Building-Eval-Pipelines.md
- **Sub-topics:** Eval pipeline architecture: dataset → runner → scorer → aggregator
- CI/CD eval gates: run on every push, block on regression
- Regression detection: statistical comparison between runs
- Automated benchmarking: scheduled runs, trend tracking
- **🛠 Skills:** Eval pipeline implementation, CI/CD integration, regression detection

## 06-Production-Observability.md
- **Sub-topics:** OpenTelemetry — industry standard, auto-instrumentation
- Langfuse — traces, spans, observations, scores, cost tracking
- Trace context propagation — trace_id across services
- Custom trace for RAG: retrieval → context → generation → score
- Logfire — OTel-native observability with built-in AI tracing
- **🛠 Skills:** Observability implementation, Langfuse integration, distributed tracing

## 07-Regression-Detection.md
- **Sub-topics:** Drift detection — input drift, output drift, concept drift
- Score drift — evaluation scores degrading over time
- Automated alerting — Slack/PagerDuty when quality drops
- Incident response playbook: detect → triage → fix → verify → post-mortem
- On-call for AI systems: different failure modes than traditional software
- **🛠 Skills:** Drift detection, alerting design, incident response, AI on-call

## 08-Custom-Eval-Framework.md
- **Sub-topics:** Build your own evaluation framework
- Example: citation accuracy eval — claim extraction → source verification → score
- Test suites, automated runners, reporting
- **🛠 Skills:** Eval framework design, custom metric implementation, reporting

## Project: AI Eval Platform
- **Sub-topics:** Every LLM call traced (prompt, response, latency, cost, model)
- Automated LLM-as-judge evaluation on every deployment
- Grafana dashboards: latency P50/P95/P99, error rates, cost trends
- Alerting on regression (Slack/PagerDuty)
- A/B comparison of prompt variants
- **🛠 Skills:** Full observability platform, monitoring, alerting, dashboarding

---

# Phase 8: Guardrails, Safety & Security
**⏱ 16–20 hours · 9 files**

> Make your AI system safe by design — prevent harm, not just detect it.

## 00-Module-Overview.md
- Phase goals: prevent harm before it happens, defense in depth
- Eval vs safety mindset — two sides of same coin

## 01-Threat-Modeling.md
- **Sub-topics:** Prompt injection — direct (user input), indirect (retrieved content), multi-turn
- Data poisoning — training data contamination, fine-tuning attacks
- Model inversion — extracting training data from model responses
- Model theft — stealing model capabilities via API
- Supply chain attacks — compromised dependencies, poisoned open-source models
- **🛠 Skills:** Threat modeling, attack taxonomy, security awareness

## 02-Prompt-Injection.md
- **Sub-topics:** 9 injection attack types: roleplay, jailbreak, leak, override, etc.
- Direct injection: user input overrides system prompt
- Indirect injection: retrieved content contains attack
- Multi-turn injection: build up context over multiple messages
- Defense strategies for each attack type
- **🛠 Skills:** Injection detection, attack classification, defense implementation

## 03-Input-Guardrails.md
- **Sub-topics:** Classify and filter harmful inputs before they reach the LLM
- Input classification: harm categories, severity scoring
- Block/flag/warn decisions based on severity
- Fast path: simple classifier (block obvious attacks)
- Deep path: LLM-based analysis (detect subtle attacks)
- **🛠 Skills:** Input guardrail design, classification, multi-tier filtering

## 04-Output-Guardrails.md
- **Sub-topics:** Filter unsafe outputs before they reach users
- Content moderation: hate, harassment, violence, self-harm, sexual
- Topic restrictions: stay in allowed topics, reject off-topic
- Output validation: schema compliance, constraint checking
- **🛠 Skills:** Output filtering, content moderation, constraint enforcement

## 05-PII-Redaction.md
- **Sub-topics:** PII detection: names, emails, phones, SSNs, addresses, credit cards
- Redaction strategies: masking, hashing, replacement, deletion
- Context-aware PII: "John said..." vs "John Smith, SSN: 123-45-6789"
- PII in RAG context: retrieved documents may contain sensitive data
- **🛠 Skills:** PII detection, redaction implementation, context-aware filtering

## 06-Content-Moderation.md
- **Sub-topics:** Safety classification across multiple harm categories
- Severity scoring: safe → sensitive → unsafe → prohibited
- Multi-model moderation: classifier + LLM + rules cascade
- Custom moderation categories for your use case
- **🛠 Skills:** Moderation system design, severity classification, custom categories

## 07-Guardrails-Architecture.md
- **Sub-topics:** Multi-layer defense: input → LLM → output
- Guardrail router: route to appropriate guardrail based on content type
- Latency-aware routing: fast path for obvious cases, deep path for edge cases
- Defense in depth: no single point of failure, layered security
- **🛠 Skills:** Guardrail architecture, defense-in-depth design, routing patterns

## 08-Testing-Guardrails.md
- **Sub-topics:** Adversarial testing — generate attack variations, measure defense rate
- Red teaming: systematic attack simulation, blind spots identification
- Safety evaluation: automated test suites for each guardrail
- Regression testing: ensure new attacks don't bypass existing defenses
- **🛠 Skills:** Adversarial testing, red teaming, safety automation

## Project: Safe AI Gateway
- **Sub-topics:** Multi-layer guardrail system: input guardrails → LLM → output guardrails
- Input filtering (harm classification), output filtering (content moderation)
- PII redaction (detect + mask)
- Injection defense (direct + indirect + multi-turn)
- Guardrail evaluation suite (accuracy, latency, bypass rate)
- **🛠 Skills:** Production guardrail system, security infrastructure, defense evaluation

---

# Phase 9: Fine-Tuning & LLMOps
**⏱ 16–20 hours · 9 files**

> Stop consuming models — become a creator of them. LoRA, data prep, training, deployment.

## 00-Module-Overview.md
- Phase goals: when NOT to fine-tune is the most important skill
- Cost analysis: fine-tuning vs RAG vs prompting comparison
- Fine-tuning is overhyped and misunderstood — most of the time, prompting + RAG wins

## 01-Fine-Tuning-Fundamentals.md
- **Sub-topics:** Decision matrix: fine-tuning vs RAG vs prompt engineering
- Common patterns:
  1. RAG only — for knowledge-heavy tasks, frequently changing data
  2. Fine-tuning only — for format/style tasks, low data needs
  3. Both (most production systems) — learn format via fine-tuning, facts via RAG
- **🛠 Skills:** Decision framework, pattern selection, hybrid architecture

## 02-LoRA-and-PEFT.md
- **Sub-topics:** Why parameter-efficient fine-tuning — LoRA costs $10, full fine-tuning costs $10K+
- How LoRA works: low-rank matrices, adapter injection, rank selection
- QLoRA: LoRA + 4-bit quantization — fine-tune 8B models on single GPU
- Training script: dataset → tokenize → LoRA config → trainer → save adapter
- LoRA configuration guide: rank, alpha, target modules, dropout
- **🛠 Skills:** LoRA configuration, adapter training, PEFT methodology

## 03-Dataset-Preparation.md
- **Sub-topics:** The 80% rule — data quality is 80% of fine-tuning success
- Data formats: chat template format, completion format
- Data quality checklist: ✅ good data characteristics, ❌ bad data characteristics
- Data augmentation: paraphrasing, back-translation, perturbation
- Quality > quantity — 500 high-quality examples > 10K noisy examples
- **🛠 Skills:** Dataset preparation, quality assessment, augmentation techniques

## 04-Training-with-Unsloth-TRL.md
- **Sub-topics:** The training stack: datasets → tokenizer → model → LoRA → trainer
- Unsloth: 2x faster training, 50% less memory than standard implementations
- Hugging Face TRL: SFTTrainer, DPOTrainer, configuration
- Multi-GPU training: DeepSpeed, FSDP, tensor parallelism
- Training monitoring: loss curves, gradient norms, learning rate
- Common training failures: loss spikes, NaN gradients, OOM errors
- **🛠 Skills:** Training infrastructure, Unsloth usage, failure diagnosis

## 05-Evaluation-and-Comparison.md
- **Sub-topics:** Before/after comparison framework
- Domain-specific evals: create test set, run both models, compare metrics
- MT-Bench style evaluation: multi-turn conversation quality scoring
- The eval trap: fine-tuned model scores high on training distribution, fails on OOD
- **🛠 Skills:** Model comparison, domain-specific evaluation, generalization testing

## 06-Fine-Tuning-for-Function-Calling.md
- **Sub-topics:** Tool-calling fine-tuning: train model to use specific tool set
- Structured output training data: input → expected tool call format
- Dataset creation for function calling
- Evaluation: tool selection accuracy, parameter correctness
- **🛠 Skills:** Function-calling fine-tuning, structured output training, tool-use evaluation

## 07-LLMOps-Model-Registry.md
- **Sub-topics:** Model versioning — track every training run
- Experiment tracking: hyperparameters, datasets, metrics, artifacts
- Evaluation database: store all model evaluation results, queryable
- Model lineage: which data → which training → which eval → which deployment
- **🛠 Skills:** LLMOps infrastructure, experiment management, model governance

## 08-Deployment-of-Fine-Tuned-Models.md
- **Sub-topics:** vLLM serving for fine-tuned models
- Quantization: AWQ, GPTQ, GGUF — when to use each
- Model merging: SLERP, TIES, DARE — combine multiple fine-tuned adapters
- Serving infrastructure: Docker, GPU allocation, autoscaling
- **🛠 Skills:** Model deployment, quantization, merge techniques

## Project: Fine-Tuned Domain Expert
- **Sub-topics:** Fine-tune Llama 3.2 8B on a domain (legal, medical, code, support)
- Synthetic data generation using GPT-4o
- LoRA training with Unsloth (< 2 hours on single GPU)
- Evaluation: base model vs fine-tuned vs RAG-augmented base
- Quantize and deploy with vLLM
- **🛠 Skills:** End-to-end fine-tuning pipeline, domain adaptation, deployment

---

# Phase 10: Deployment & Scale
**⏱ 20–25 hours · 9 files**

> Ship AI that survives production — infrastructure, caching, cost, latency, monitoring.

## 00-Module-Overview.md
- Phase goals: production failure scenarios, cost traps
- 90% of AI projects fail at this step — not because the model is bad
- Framework integration: vLLM, Llama Stack, LiteLLM

## 01-Production-Infrastructure.md
- **Sub-topics:** Docker multi-stage builds for AI services (< 500MB image)
- Docker Compose for AI stack: app + vector DB + model server + Redis + monitoring
- Production patterns: health checks, graceful shutdown, resource limits
- Non-root user for security
- **🛠 Skills:** Docker optimization, compose orchestration, production container patterns

## 02-Model-Serving-at-Scale.md
- **Sub-topics:** vLLM — PagedAttention, continuous batching, KV cache management
- Basic vLLM server setup, programmatic usage
- Tensor parallelism for multi-GPU serving
- Llama Stack — standardized multi-provider serving API
- vLLM vs TGI vs Llama Stack: when to use each
- **🛠 Skills:** Model serving infrastructure, vLLM configuration, serving comparison

## 03-Caching-with-Redis.md
- **Sub-topics:** Exact match cache — Redis, simple key-value, TTL
- Semantic cache — embed query, find similar cached results
- Cache-aside pattern: check → return or compute → store
- Cache invalidation: TTL, webhook-based, version-based
- **🛠 Skills:** Redis caching, semantic caching, cache strategy design

## 04-Cost-Optimization.md
- **Sub-topics:** Where the money goes: inference (80%+), embedding, storage, infra
- Optimization strategies:
  1. Model selection (biggest impact) — cascade: cheap → expensive
  2. Caching — exact + semantic can save 40%+
  3. Prompt optimization — shorter prompts = lower cost
  4. Batch processing — OpenAI batch API offers 50% discount
- Cost tracking dashboard: per-query, per-model, per-user
- **🛠 Skills:** Cost optimization strategy, model cascades, batch processing

## 05-Latency-Optimization.md
- **Sub-topics:** Speculative decoding — draft model + target model, 2-3x speedup
- KV cache optimization: prefix caching, prompt caching
- Streaming — first-token latency, inter-token latency
- Connection pooling, keep-alive, HTTP/2
- **🛠 Skills:** Latency optimization, speculative decoding, streaming optimization

## 06-Monitoring-and-Alerting.md
- **Sub-topics:** Prometheus + Grafana for AI-specific metrics
- Key metrics: latency (P50/P95/P99), throughput, error rate, cost rate
- AI-specific metrics: quality score trend, retrieval recall trend
- Alert thresholds: what to alert on, how to set thresholds
- On-call playbook for AI systems: what to check in what order
- **🛠 Skills:** Monitoring infrastructure, alert design, on-call procedures

## 07-Multi-Provider-Failover.md
- **Sub-topics:** Fallback routing: primary → secondary → tertiary
- Health checking: active probes, passive observation, error tracking
- Graceful degradation: which features degrade and how
- Cost-aware routing: cheapest acceptable provider for each task
- **🛠 Skills:** Failover design, health monitoring, cost-aware routing

## 08-Security-and-Compliance.md
- **Sub-topics:** API security: authentication, authorization, rate limiting
- Encryption: at-rest, in-transit, key management
- Audit logging: who called what, when, with what result
- Compliance requirements: SOC2, HIPAA, GDPR considerations for AI
- **🛠 Skills:** API security, compliance awareness, audit infrastructure

## Project: Production Infrastructure
- **Sub-topics:** Deploy Phase 6's multi-agent system to production
- Dockerized with multi-stage builds (< 500MB)
- AWS ECS with blue/green deployment
- vLLM + Llama Stack for model serving
- CI/CD pipeline with eval gates
- Cost tracking dashboard ($ per query, per model)
- Load tested to 100+ concurrent requests
- Security guardrails in place
- **🛠 Skills:** Full production deployment, infrastructure-as-code, CI/CD, security hardening

---

# Phase 11: Capstone
**⏱ 4 weeks**

> Build & launch a real AI product that integrates ALL 10 previous phases.

## 00-Capstone-Overview.md
- **Sub-topics:** The mission — combine all phases into one production-grade system
- 4 project options with market validation
- Capstone process: Week 1 (plan) → Week 2 (build core) → Week 3 (productionize) → Week 4 (ship & polish)
- Deliverables: design doc, working system, eval report, cost analysis, post-mortem, demo, portfolio entry
- Scoring rubric: architecture (20%), RAG quality (20%), agent logic (15%), evaluation (15%), production (15%), guardrails (10%), polish (5%)

## 01-Project-Options-Details.md
- **Sub-topics:** Deep dives on each option with required components, evaluation metrics, architecture, and market positioning

**Option 1: AI Customer Support System**
- Market: $4B+ (Zendesk AI, Intercom Fin, Ada, Airbnb Support)
- Required: multi-agent system, RAG over KB, sentiment detection, escalation, human handoff
- Metrics: 60%+ auto-resolution, <15s response, <$0.10 per ticket

**Option 2: AI Code Review Assistant**
- Market: $12.8B (CodeRabbit, Qodo, GitHub Copilot)
- Required: GitHub App integration, diff analysis, multi-agent review (security/bug/style)
- Metrics: >30% bug catch rate, <20% false positives, <2 min per PR

**Option 3: AI Document Intelligence Platform**
- Market: Enterprise knowledge management (Glean, Hebbia, CustomGPT.ai, Notion AI)
- Required: multi-format ingestion, hybrid search, citation-grounded answers, faithfulness validation
- Metrics: >0.95 faithfulness, >90% context recall, <$0.05 per query

---

## 🗺 Curriculum Map

```
Phase 0:  Engineering Mindset ═══════════════ Mental models & decision frameworks
         ↓
Phase 1:  LLM Gateway ═══════════════════════ How LLMs work, API integration, streaming
         ↓
Phase 2:  Prompt Engineering System ═════════ Systematic prompting, DSPy, versioning
         ↓
Phase 3:  Semantic Search Engine ════════════ Embeddings, vector DBs, hybrid search
         ↓
Phase 4:  Production RAG Engine ═════════════ RAG from scratch → LangChain → LlamaIndex
         ↓
Phase 5:  Advanced Knowledge System ═════════ HyDE, agentic RAG, GraphRAG, multimodal
         ↓
Phase 6:  Multi-Agent System ════════════════ ReAct, tools, memory, LangGraph, MCP
         ↓
Phase 7:  Eval & Observability Platform ═════ LLM-as-judge, Langfuse, Grafana
         ↓
Phase 8:  Guardrails & Safety ═══════════════ Prompt injection, PII, content moderation
         ↓
Phase 9:  Fine-Tuning & LLMOps ══════════════ LoRA, Unsloth, quantization
         ↓
Phase 10: Deployment & Scale ════════════════ vLLM, AWS, caching, cost optimization
         ↓
Phase 11: Capstone ═══════════════════════════ Integrate everything → ship real product
```

## 📊 Learning Distribution

| Activity | Hours | % |
|----------|-------|---|
| Concept reading | ~60 | 20% |
| Code drills | ~50 | 17% |
| Portfolio projects | ~120 | 41% |
| Debug challenges | ~20 | 7% |
| Evaluation & testing | ~30 | 10% |
| Deployment & ops | ~15 | 5% |
| **Total** | **~295** | **100%** |

## 🛠 Full Tech Stack (By End of Course)

**Core AI:** OpenAI API, Anthropic Claude, Google Gemini, Ollama (Llama 3, Mistral, Phi), LiteLLM
**Prompt Engineering:** DSPy, Instructor, Outlines, custom eval
**Frameworks:** LangChain, LlamaIndex, LangGraph, Pydantic AI
**Vector DBs:** ChromaDB, Qdrant, Pinecone, pgvector, FAISS
**Observability:** Langfuse, Logfire, RAGAS, DeepEval, Prometheus + Grafana
**Infrastructure:** FastAPI, Docker, AWS ECS, vLLM, Redis, GitHub Actions
