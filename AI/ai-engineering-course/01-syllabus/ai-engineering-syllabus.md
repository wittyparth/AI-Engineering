# AI Engineering Mastery Syllabus
## Based on NexaAI Studio's Learn-by-Building System

> **Your goal:** Become an AI Engineer who ships production systems — RAG pipelines, autonomous agents, evaluation harnesses, and multi-agent orchestration — not just a "prompt engineer" who can call an API.
>
> **How this works:** This syllabus is designed to be delivered through the NexaAI Studio roleplay system. Each phase maps to NexaAI's tier structure. You learn by doing real client projects with Rohan (CTO) holding the quality bar, Maya (Senior AI Engineer) guiding the learning path, and Priya (PM) defining business requirements.
>
> **Estimated timeline:** 6-12 months at 10-15 hours/week (faster if you're already a software engineer)

---

## How the NexaAI System Teaches This Syllabus

The NexaAI Studio system is NOT a traditional course. It's a simulated AI consultancy where every project is a real client need. Here's how each part of the system delivers this syllabus:

| NexaAI Role | How They Teach |
|---|---|
| **Priya (PM)** | Opens every project with a **client brief** — business problem, deadline, non-negotiable requirements, definition of done. Creates real stakes. Translates "I need to learn RAG" into "The client's support team spends 4 hours/day answering questions in their docs. Your job: build a system that cuts that to 30 minutes." |
| **Maya (Senior AI Engineer)** | Teaches **scratch first, framework second**. Before touching LangChain, you build retrieval manually so you feel the pain it solves. She says exactly what to memorize vs what to look up. Links resources at the precise moment needed — not a dump upfront. Asks Socratic questions that force deep thinking. |
| **Rohan (CTO)** | Reviews every submission against a **7-point production quality bar**: Does it work? Will it break in production? Is it cost-aware? Is it observable? Is the approach right? Are decisions justified? Is quality measurable? APPROVED / REVISE / REJECTED with specific line-level feedback. |
| **Projects build on each other** | A shortcut in Project 1 comes back to bite in Project 5. Rohan references past mistakes. This enforces genuine learning — you can't fake your way through. |
| **Scaffolded difficulty** | You start with internal studio tools (no client pressure), graduate to real clients with Maya heavily present, then to independent work with real production constraints, and finally to portfolio-level complex systems. |

**The golden rule:** Frameworks are introduced ONLY after you've done it manually and felt the pain they solve.

---

## Syllabus Overview

```
TIER 1: Internal Studio Tools (Projects 1-3)
    ↓
TIER 2: First Real Clients, Guided Heavily (Projects 4-7)
    ↓
TIER 3: Real Constraints, Independent (Projects 8-12)
    ↓
TIER 4: Portfolio-Level Complex Systems (Projects 13+)
```

Each Tier has **concepts to learn** (with specific courses/resources), **projects to build** (with real business context), and **quality gates** (what Rohan checks before you advance).

---

# TIER 1: Internal Studio Tools
*"Get comfortable with the AI engineering workflow. No client pressure. You're building tools for your own team."*

**Duration:** ~1.5-2 months
**When you finish:** You can call LLM APIs, handle structured outputs, build a basic chatbot with memory, and deploy a simple API.

---

## Module 1.1: Python for AI Engineering

**What you need to be able to do:**
- Write Python with classes, async/await, error handling, type hints
- Use Pydantic v2 for data validation and schemas
- Make HTTP requests, parse JSON, handle API authentication
- Work with Git: commit, branch, PR, resolve conflicts
- Set up Python virtual environments and manage dependencies

**What Maya says about this:** *"You don't need to be a Python expert to start. But you need enough that the language isn't fighting you while you're trying to learn AI concepts. Focus on: functions, classes, async, type hints, and Pydantic. You'll learn the rest as you go."*

**Resources:**
- [ ] **Automate the Boring Stuff with Python** (free online book) — for Python fundamentals
- [ ] **FastAPI Official Tutorial** (docs) — focus on path ops, Pydantic models, dependency injection
- [ ] **Pydantic v2 docs** — model definitions, validators, serialization
- [ ] Real Python's `asyncio` walkthrough — you'll need async for LLM calls

**Memorize cold:** Pydantic model syntax, async/await pattern, Git add-commit-push cycle
**Look up when needed:** Specific FastAPI middleware config, Git advanced commands

---

## Module 1.2: LLM API Fundamentals

**What you need to be able to do:**
- Call OpenAI and Anthropic APIs from Python
- Understand the messages array structure (system/user/assistant)
- Handle streaming responses
- Manage API keys, rate limits, and error handling
- Count tokens and estimate costs
- Understand temperature, top-p, max_tokens, stop sequences

**Production reality:** In 2026, companies use a mix of OpenAI, Anthropic, and open-source models. 70% use RAG. Most use a hybrid approach — API-based for external, open models for internal data-sensitive workloads.

**Maya's teaching sequence:**
1. Call the API via raw HTTP first (feel the boilerplate)
2. Then use the SDK (OpenAI Python SDK / Anthropic Python SDK)
3. Add streaming
4. Add error handling + retries
5. Add token counting + cost tracking

**Resources:**
- [ ] **OpenAI API Reference** — focus on Chat Completions
- [ ] **Anthropic API Docs** — focus on Messages API
- [ ] **OpenAI Cookbook** — production-tested recipes
- [ ] DeepLearning.AI: **Building Systems with the ChatGPT API** (short course)

**Memorize cold:** Messages array structure, streaming loop pattern, error handling pattern
**Look up when needed:** Specific model pricing, rate limit tiers

---

## Module 1.3: Structured Outputs & Prompt Engineering

**What you need to be able to do:**
- Write system prompts that control behavior reliably
- Use few-shot prompting effectively
- Get LLMs to return valid JSON consistently
- Use function calling / tool use for structured outputs
- Implement chain-of-thought reasoning
- Version and test prompts systematically

**Production reality:** Structured outputs are foundational — in production AI systems, you ALWAYS need the model to return parseable, typed data. Pydantic + structured outputs is the standard pattern.

**Maya's teaching sequence:**
1. Get the model to return JSON in markdown blocks (brittle — feel the pain)
2. Use response_format / JSON mode (better)
3. Use function calling / tool use for structured outputs (production standard)
4. Combine with Pydantic for schema validation + retry logic
5. Systematic prompt testing with Promptfoo or similar

**Resources:**
- [ ] **OpenAI Structured Outputs Guide**
- [ ] **Anthropic Tool Use Docs**
- [ ] **DAIR.AI Prompt Engineering Guide** (72K stars on GitHub)
- [ ] DeepLearning.AI: **Prompt Engineering for ChatGPT** (short course)

**Memorize cold:** System prompt structure, JSON mode pattern, tool/function schema format
**Look up when needed:** Specific prompting techniques, chain-of-thought variants

---

## Module 1.4: Building & Deploying AI APIs

**What you need to be able to do:**
- Build a FastAPI application with proper structure
- Serve LLM responses through REST endpoints
- Handle CORS, authentication, rate limiting
- Containerize with Docker
- Deploy to cloud (Railway, Render, or Fly.io)
- Set up basic monitoring and logging

**Maya's teaching sequence:**
1. FastAPI hello world with a /chat endpoint
2. Add streaming SSE endpoint
3. Add conversation memory (in-memory first)
4. Containerize with Docker
5. Deploy

**Resources:**
- [ ] **FastAPI Official Tutorial** — full walkthrough
- [ ] **Docker Official Tutorial** — containers for AI apps
- [ ] **Railway / Render docs** — quick deployment

**Memorize cold:** FastAPI app structure, Dockerfile pattern
**Look up when needed:** Specific deployment configs, CORS settings

---

### TIER 1: Projects & Quality Gates

**Project 1: Internal Prompt Testing Harness**
*"NexaAI's team keeps tweaking prompts for different clients and we can't tell if changes make things better or worse. Build us an internal tool to test prompt versions against expected outputs."*
- **Concepts:** Prompt engineering, structured outputs, basic eval (manual comparison)
- **Tech stack:** Python, OpenAI/Anthropic API, basic file storage
- **Rohan's gate:** ✅ APPROVED when you can demonstrate that changing one word in a prompt produces measurably different quality

**Project 2: Meeting Transcript Summarization Pipeline**
*"Our PM Priya has 20 hours of client meeting recordings every week. She needs a tool that takes raw transcripts and produces: (1) a 3-bullet executive summary, (2) action items with owners, (3) key decisions made."*
- **Concepts:** Structured outputs, Pydantic validation, batch processing, cost awareness
- **Tech stack:** Python, LLM API, Pydantic, basic file I/O
- **Rohan's gate:** ✅ APPROVED when processing 10 transcripts costs less than $1 total AND extracts consistently structured output

**Project 3: Internal Semantic Search Tool**
*"We have 500+ pages of internal documentation spread across Notion, Google Docs, and Slack. No one can find anything. Build a search tool that lets us find the right doc by describing what we need in natural language."*
- **Concepts:** Embeddings, vector similarity search (in-memory), semantic vs keyword search
- **Tech stack:** Python, sentence-transformers or OpenAI embeddings, NumPy/FAISS for vector search, FastAPI
- **Rohan's gate:** ✅ APPROVED when you can explain why cosine similarity was the right choice (or when it wasn't), and demonstrate your search returns better results than CTRL+F

---

# TIER 2: First Real Clients, Guided Heavily
*"Real business stakes, but Maya is very present. You'll make mistakes but catch them before they reach the client."*

**Duration:** ~2-3 months
**When you finish:** You can build production RAG pipelines, understand chunking strategies, use vector databases, implement hybrid search, and evaluate system quality.

---

## Module 2.1: RAG from Scratch

**What you need to be able to do:**
- Understand why RAG exists (LLMs don't know your data)
- Build a manual RAG pipeline: chunk → embed → store → retrieve → generate
- Implement different chunking strategies (fixed-size, semantic, recursive)
- Choose and use embedding models
- Set up a vector database (Chroma for dev, Qdrant for production)
- Implement hybrid search (BM25 + vector)
- Add reranking (cross-encoder)

**Production reality (from 10 real enterprise deployments analyzed):**
- Hybrid RAG (dense + sparse) outperforms vector-only in EVERY production deployment
- Typical sweet spot: 70% dense, 30% sparse weighting
- Agentic RAG (multi-step retrieval) needed for complex cross-document queries
- Chunking strategy is the #1 lever — 512-token semantic chunks with 10-20% overlap performs best generally
- Caching layer saves ~40% in costs
- RAG in production is 20% AI, 80% engineering (extraction, chunking, eval, monitoring)

**Maya's teaching sequence:**
1. Build RAG with in-memory vectors and NumPy (feel the pain of rolling your own)
2. Add a real vector database (Chroma first, then Qdrant)
3. Add BM25 keyword search alongside vector search
4. Add reranking
5. Measure retrieval quality (Recall@k, MRR)
6. Now introduce LangChain/LlamaIndex — *"you can see exactly what they abstract now"*

**Resources:**
- [ ] **RAG From Scratch** (LangChain video series) — watch after doing it manually
- [ ] DeepLearning.AI: **Building and Evaluating Advanced RAG**
- [ ] **Qdrant documentation** — vector database basics
- [ ] **NirDiamant/RAG_Techniques** (26K stars on GitHub)
- [ ] Chip Huyen's **AI Engineering** book — chapters on retrieval

**Memorize cold:** RAG pipeline steps, vector DB upsert/query patterns, chunking parameters
**Look up when needed:** Specific embedding model benchmarks, vector index tuning (HNSW params)

---

## Module 2.2: Vector Databases & Search

**What you need to be able to do:**
- Choose between vector DB options (Qdrant, Pinecone, pgvector, Weaviate, Chroma)
- Understand HNSW indexing and its parameters
- Implement hybrid search (dense + sparse)
- Add metadata filtering
- Handle document updates and deletions
- Understand scaling considerations

**Production reality (from 2025 survey of 1,000+ AI teams):**
- 65% use a dedicated vector database (not general-purpose with vector extension)
- pgvector is rising fast for teams that want one less service
- Qdrant dominates for high-scale production RAG
- Pinecone is the most popular fully-managed option
- Access control in vector search is a hard problem many teams get wrong

**Resources:**
- [ ] **Weaviate Academy** (free, best learning resource for vector DBs)
- [ ] **Qdrant documentation** — HNSW, filtering, hybrid search
- [ ] **pgvector documentation** — if you want to use PostgreSQL

**Memorize cold:** Upsert/query API, hybrid search pattern, metadata filter syntax
**Look up when needed:** HNSW ef_construction, M parameters, specific index tuning

---

## Module 2.3: Evaluation & Quality Measurement

**What you need to be able to do:**
- Build a golden test dataset for your RAG system
- Measure retrieval metrics (precision, recall, MRR, NDCG)
- Measure generation metrics (faithfulness, answer relevancy, context precision)
- Use RAGAS or DeepEval for automated evaluation
- Implement LLM-as-a-judge evaluation
- Set up regression testing for prompt and model changes

**Why this matters:** Rohan says *"'It feels like it's working' isn't good enough for the client."* In production, evaluation is what separates reliable systems from demos. 70% of teams update prompts monthly — without eval, you're flying blind.

**Resources:**
- [ ] **RAGAS documentation** — faithfulness, relevancy, precision
- [ ] **DeepEval** (14K stars) — 50+ metrics including RAG and agent eval
- [ ] **Promptfoo** — unit tests for prompts, CI integration
- [ ] **Hamel Husain's Field Guide to Evaluating AI Systems**

**Memorize cold:** RAGAS metric definitions, eval dataset structure
**Look up when needed:** Specific metric implementations, eval framework APIs

---

### TIER 2: Projects & Quality Gates

**Project 4: Legal Document Q&A System**
*"A law firm has 500+ PDFs of contracts, case law, and regulatory documents. Their junior lawyers spend hours searching for precedents. Build a system where they can ask questions in plain English and get answers with exact citations."*
- **Concepts:** PDF parsing, chunking strategy, hybrid search, citation rendering
- **Tech stack:** Python, PyMuPDF for PDFs, Qdrant/Chroma, BM25 + embedding hybrid, FastAPI
- **Rohan's gate:** ✅ APPROVED when every answer includes the source document, page number, and excerpt — AND you can prove retrieval precision > 80% on a held-out test set of 50 queries

**Project 5: E-commerce Customer Support Bot**
*"An online retailer's support team handles 200+ return/refund queries daily. The answers are in their policy documents but agents spend 5 minutes searching per query. Build a bot that answers instantly with policy citations."*
- **Concepts:** Multi-tenant RAG, conversation memory, query rewriting, confidence thresholds
- **Tech stack:** FastAPI, Qdrant, LangChain/LlamaIndex (after manual build), Docker
- **Bonus:** Add "I don't know" handling — the bot should escalate to human when confidence is low
- **Rohan's gate:** ✅ APPROVED when the bot correctly handles edge cases (ambiguous queries, multi-part questions, out-of-policy requests) AND costs < $0.02 per query

**Project 6: Structured Data Extraction Pipeline**
*"A recruiting firm needs to parse 1,000+ CVs/week into structured JSON — skills, experience, education, certifications. Their current process is manual data entry."*
- **Concepts:** Batch processing, structured outputs, Pydantic validation with retry, cost optimization
- **Tech stack:** Python, LLM, Pydantic, batch queue
- **Rohan's gate:** ✅ APPROVED when extraction accuracy > 95% on a validation set, cost < $0.10 per CV, and you can show the failure cases where it struggles

**Project 7: Multi-Model API Gateway**
*"NexaAI needs an internal gateway that routes requests to the cheapest adequate model. Simple queries go to Haiku, complex reasoning to Sonnet/Opus. Build the routing layer."*
- **Concepts:** Model routing, cost optimization, fallback strategies, unified API interface, LiteLLM
- **Tech stack:** FastAPI, LiteLLM, prompt classification for routing, cost logging
- **Rohan's gate:** ✅ APPROVED when routing saves at least 40% cost vs always using the best model, with < 5% quality degradation measured by LLM-as-judge

---

# TIER 3: Real Constraints, More Independent
*"Clients now have production constraints: latency budgets, cost ceilings, accuracy requirements. Maya gives less hand-holding."*

**Duration:** ~2.5-3.5 months
**When you finish:** You can build agents with tool use, implement advanced RAG patterns, design evaluation frameworks, and understand full-stack AI deployment.

---

## Module 3.1: AI Agents from Scratch

**What you need to be able to do:**
- Build the ReAct loop manually (observe → think → act → observe...)
- Design and implement tool schemas
- Handle tool call failures and retries
- Implement conversation memory (short-term + long-term)
- Build stateful agents with LangGraph
- Implement guardrails and safety controls (step limits, cost bounds)

**Production reality (from Anthropic's 2025 field research):**
- The most effective agents are simple — augmented LLMs doing one thing well
- Most production systems use prompt chaining (deterministic) not autonomous agents
- Multi-agent orchestration is reserved for when a single agent demonstrably fails
- The key to good agents is well-designed tools, not clever prompting
- Hard limits on steps and cost are non-negotiable in production

**Anthropic's Agent Patterns (in order of complexity):**
1. **Augmented LLM** — LLM + retrieval + tools (start here)
2. **Prompt Chaining** — decompose task into sequential steps
3. **Routing** — classify input → route to specialized handler
4. **Parallelization** — run multiple calls simultaneously
5. **Orchestrator-Workers** — central orchestrator delegates to sub-agents
6. **Evaluator-Optimizer** — generator + evaluator in loop

**Maya's teaching sequence:**
1. Build ReAct loop in raw Python (feel the state management pain)
2. Add tools (search, calculator, file reader)
3. Add memory (conversation history, persistent storage)
4. Add guardrails (max steps, cost limits, output validation)
5. Now introduce LangGraph — *"you can see why state machines matter"*

**Resources:**
- [ ] **Anthropic's Building Effective Agents** (must-read engineering guide)
- [ ] **Anthropic Tool Use Documentation** with cookbooks
- [ ] DeepLearning.AI: **AI Agents in LangGraph** (short course)
- [ ] **LangGraph documentation** — state graphs, nodes, edges
- [ ] **NirDiamant/GenAI_Agents** (7K+ stars) — 45 agent notebook implementations

**Memorize cold:** ReAct loop structure, tool schema format, LangGraph state definition
**Look up when needed:** LangGraph node/edge API specifics, tool call error handling patterns

---

## Module 3.2: Advanced RAG Patterns

**What you need to be able to do:**
- Implement query rewriting and expansion
- Build agentic RAG (multi-step retrieval with reasoning)
- Implement HyDE (Hypothetical Document Embeddings)
- Parent-child chunking strategies
- Implement GraphRAG for multi-hop reasoning
- Build feedback loops (user ratings → retrieval improvement)
- Implement temporal filtering for time-sensitive data

**Production reality (from case studies of Wix, Fortune 500 manufacturer, pharma companies):**
- Agentic RAG uses LLM to decide how many retrieval passes and which indexes to query
- Query rewriting improves retrieval accuracy by 15-30%
- Wix's production multi-agent RAG system uses specialized agents for: root intent detection, question validation, query division, and data retrieval
- Advanced RAG evaluation shows improvement from 72% → 96.8% accuracy across progressive optimizations

**Resources:**
- [ ] **NirDiamant/RAG_Techniques** (26K stars) — comprehensive RAG patterns
- [ ] DeepLearning.AI: **Advanced RAG Patterns** (short course)
- [ ] **Microsoft GraphRAG** — knowledge graph + RAG
- [ ] **LlamIndex documentation** — advanced retrieval strategies

**Memorize cold:** Query rewriting pattern, HyDE implementation, parent-child chunking
**Look up when needed:** Specific reranker model benchmarks, GraphRAG configuration

---

## Module 3.3: Observability & LLMOps

**What you need to be able to do:**
- Set up tracing for LLM calls (Langfuse, LangSmith, or OpenTelemetry)
- Track latency, token usage, and cost per request
- Implement prompt versioning and management
- Build dashboards for system monitoring
- Set up alerts for quality degradation
- Implement A/B testing for prompt and model changes

**Production reality (from 2025 AI Engineering Report):**
- 60% of production teams use observability tools
- Only 1 in 3 teams are satisfied with their observability — it's the weakest part of most stacks
- 62% of teams plan to improve observability — it's the #1 investment area
- Langfuse and LangSmith are the dominant tracing platforms
- Poor observability is the root cause of most production AI failures

**Resources:**
- [ ] **Langfuse documentation** — tracing, prompt management, evaluation
- [ ] **LangSmith documentation** — LangGraph-integrated observability
- [ ] **OpenTelemetry GenAI semantic conventions**
- [ ] **James Briggs: LangSmith 101**

**Memorize cold:** Trace structure (span, event, metadata), cost tracking pattern
**Look up when needed:** Specific SDK methods, dashboard configuration

---

## Module 3.4: Model Context Protocol (MCP)

**What you need to be able to do:**
- Understand MCP architecture (Host → Client → Server)
- Build an MCP server exposing tools and resources
- Integrate MCP with Claude/GPT applications
- Understand MCP security model and auth (OAuth 2.1 with PKCE)
- Implement MCP for tool standardization

**Production reality:** MCP went from Anthropic experiment in Nov 2024 to Linux Foundation standard by Dec 2025. Every major AI provider adopted it in 2025. It's now the default way AI agents connect to tools. There are 13,000+ public MCP servers as of early 2026.

**Resources:**
- [ ] **Model Context Protocol documentation** (official spec)
- [ ] DeepLearning.AI: **MCP: Build Rich-Context AI Apps with Anthropic**
- [ ] **Hugging Face MCP Course**
- [ ] **MCP GitHub repositories** — example servers in Python/TypeScript

---

## Module 3.5: Cloud Deployment & Production Infrastructure

**What you need to be able to do:**
- Deploy AI services on cloud platforms (AWS Bedrock, GCP Vertex AI, or Azure AI Foundry)
- Set up CI/CD pipelines for AI systems (GitHub Actions)
- Implement caching (semantic caching with GPTCache, Redis)
- Handle rate limiting and cost controls
- Implement authentication and authorization
- Set up monitoring and alerting

**Resources:**
- [ ] **AWS AIP-C01 certification guide** (AI Practitioner)
- [ ] **FastAPI Production Deployment** guide
- [ ] **Docker Compose** for multi-service AI deployments
- [ ] **GitHub Actions** CI/CD for ML

---

### TIER 3: Projects & Quality Gates

**Project 8: Real-Time Sales Intelligence Agent**
*"A B2B SaaS company needs an agent that monitors prospect signals (website visits, job changes, funding news), generates personalized outreach, and drafts emails. Must respond in under 3 seconds."*
- **Concepts:** Agent loop, tool use, real-time processing, personalization
- **Tech stack:** LangGraph, tool APIs (Clearbit, Crunchbase, etc.), FastAPI streaming
- **Rohan's gate:** ✅ APPROVED when agent completes 100 test scenarios with < 3s latency, < $0.05/run, and you have tracing for every step

**Project 9: Advanced Financial RAG System**
*"A financial services firm has regulatory documents, earnings reports, and market analysis. The system must cite sources, handle tables in PDFs, never hallucinate figures, and answer cross-document queries."*
- **Concepts:** Agentic RAG, query rewriting, hybrid search, vision for table extraction, citation enforcement, answer abstention for unknown
- **Tech stack:** Qdrant, LangGraph, vision model for table extraction, RAGAS evaluation
- **Rohan's gate:** ✅ APPROVED when faithfulness score > 0.95 on test set, zero hallucinated figures in 100-test evaluation, and system correctly abstains from answering on 20% of "unanswerable" test queries

**Project 10: GitHub Code Review Agent**
*"A dev team wants an agent that reviews PRs — checks for style guide violations, suggests improvements, catches common bugs. Must not produce false positives."*
- **Concepts:** Tool use (GitHub API), structured code analysis, evaluation with precision/recall
- **Tech stack:** LangGraph, GitHub API, AST parsing, eval framework
- **Rohan's gate:** ✅ APPROVED when false positive rate < 10% on a curated test set of 50 PRs

**Project 11: Natural Language to SQL Agent**
*"A healthcare company wants non-technical staff to ask questions about patient data in plain English. The system writes and executes SQL, interprets results, and handles ambiguous queries safely."*
- **Concepts:** Text-to-SQL, schema grounding, safe execution (read-only), query validation
- **Tech stack:** LLM, PostgreSQL, read-only SQL executor, query validation layer
- **Rohan's gate:** ✅ APPROVED when SQL accuracy > 85% on benchmark, zero destructive queries possible, and ambiguous queries correctly ask for clarification

**Project 12: Cost-Aware Model Router**
*"Build a production model router that classifies query complexity and routes to the optimal model (Haiku/Sonnet/Opus or equivalent). Include caching, fallbacks, and cost dashboard."*
- **Concepts:** Model routing, prompt caching, fallback chains, cost optimization, observability
- **Tech stack:** LiteLLM, Langfuse tracing, Redis caching, FastAPI
- **Rohan's gate:** ✅ APPROVED when 40%+ cost savings vs always-expensive model with < 3% quality degradation measured by comprehensive eval suite

---

# TIER 4: Portfolio-Level Complex Systems
*"These are the projects that matter. Multi-system, real architectural decisions, production-ready. Rohan gets significantly harder."*

**Duration:** Ongoing (3-5 months for first 2 projects)
**When you finish:** You can architect multi-agent systems, deploy at scale, implement human-in-the-loop workflows, and design evaluation frameworks for complex AI systems.

---

## Module 4.1: Multi-Agent Orchestration

**What you need to be able to do:**
- Design multi-agent architectures with clear agent roles
- Implement agent coordination patterns (orchestrator-workers, swarm, pipeline)
- Handle inter-agent communication and state sharing
- Implement human-in-the-loop workflows
- Build agent evaluation frameworks
- Design for failure recovery across distributed agents

**Production reality (from multi-agent case studies):**
- Wix's Anna multi-agent system uses 7 specialized agents for data discovery
- Fortune 500 manufacturing uses 12 parallel agent group chats
- Most teams use orchestrator-worker pattern, NOT fully autonomous agents
- Human-in-the-loop for high-risk actions is standard
- Data contracts between agents are critical for reliability

**Resources:**
- [ ] **LangGraph documentation** — multi-agent patterns
- [ ] **CrewAI documentation** — role-based agent collaboration
- [ ] DeepLearning.AI: **Multi AI Agent Systems with CrewAI**
- [ ] **UC Berkeley: Large Language Model Agents** (research-meets-practice view)
- [ ] **Microsoft AutoGen** — multi-agent conversation framework

---

## Module 4.2: Fine-Tuning & Customization

**What you need to be able to do:**
- Understand when to fine-tune vs use RAG vs prompt engineer
- Implement LoRA/QLoRA fine-tuning
- Prepare datasets for fine-tuning
- Evaluate fine-tuned models vs base models
- Deploy fine-tuned models (via Hugging Face, vLLM, etc.)

**Maya's teaching sequence:**
1. First, understand why RAG usually beats fine-tuning for knowledge tasks
2. Fine-tune only when you need: style/tone adaptation, output format specialization, or task-specific performance gains
3. Start with a small LoRA fine-tune on a domain-specific dataset
4. Measure before/after performance on your eval set
5. Deploy the adapter alongside your base model

**Resources:**
- [ ] **Hugging Face NLP Course** — fine-tuning chapter
- [ ] DeepLearning.AI: **Fine-tuning with LoRA**
- [ ] **Unsloth** — easy local fine-tuning
- [ ] **Sebastian Raschka: Build a Large Language Model (from Scratch)**

---

## Module 4.3: Security & Guardrails

**What you need to be able to do:**
- Understand prompt injection attack vectors
- Implement input/output guardrails
- Build PII detection and redaction
- Implement rate limiting and abuse prevention
- Design least-privilege agent permissions
- Set up audit logging for agent actions

**Resources:**
- [ ] **OWASP Top 10 for LLM Applications** — the real threat model
- [ ] **Guardrails AI** — output validation and enforcement
- [ ] **NVIDIA NeMo Guardrails** — enterprise guardrail framework
- [ ] **Lakera Guard** — prompt injection detection

---

## Module 4.4: Production Performance

**What you need to be able to do:**
- Optimize latency (prompt caching, batching, speculative decoding)
- Implement semantic caching
- Design for cost efficiency at scale
- Build model serving infrastructure (vLLM, TGI)
- Implement streaming effectively
- Design fallback and degradation strategies

**Resources:**
- [ ] **vLLM documentation** — PagedAttention, continuous batching
- [ ] **LiteLLM** — model routing and cost management
- [ ] **GPTCache** — semantic caching

---

### TIER 4: Projects & Quality Gates

**Project 13: Deep Research Agent**
*"A management consulting firm needs an agent that takes a research question, autonomously plans a research strategy, searches the web, reads and synthesizes documents, cross-references sources, identifies contradictions, and produces a structured report with citations. Must handle 50+ sources per query."*
- **Concepts:** Agent planning, web search, document synthesis, citation tracking, multi-source reasoning
- **Tech stack:** LangGraph, web search API, MCP tools, long-context LLM, evaluation with human review
- **Rohan's gate:** ✅ APPROVED when output is indistinguishable from a junior analyst's research report on 10 test topics, with all claims verifiable from cited sources

**Project 14: Multi-Agent Content Pipeline**
*"A media company needs a content production system: one agent researches trending topics, one drafts articles, one fact-checks, one optimizes for SEO, one schedules publishing. Agents must coordinate, handle failures gracefully, with human-in-the-loop for final approval."*
- **Concepts:** Multi-agent orchestration, role-based agents, human-in-the-loop, failure handling
- **Tech stack:** LangGraph/CrewAI, MCP for tool integration, approval workflow
- **Rohan's gate:** ✅ APPROVED when system produces 10 publishable articles with verified facts, passing all editorial quality checks

**Project 15: Autonomous Customer Support System**
*"A SaaS company wants to automate 80% of support tickets. System must: classify intent, retrieve relevant docs, attempt resolution, escalate when confidence is low, learn from resolved tickets, track resolution rates. Must handle 1000+ tickets/day."*
- **Concepts:** Intent classification, agentic RAG, confidence thresholds, escalation workflows, feedback loops, observability at scale
- **Tech stack:** LangGraph, RAG with hybrid search, eval pipeline, Langfuse tracing
- **Rohan's gate:** ✅ APPROVED when system resolves 80% of tickets without human intervention, maintains CSAT > 4.0, costs < $0.50/resolution

**Project 16: AI-Powered Code Generation Platform**
*"An internal dev tools company wants a system that takes a natural language spec, generates code, runs tests, iterates based on failures, explains what it built, and integrates with CI/CD pipeline."*
- **Concepts:** Tool use (sandboxed execution), evaluation (test pass rate), iterative improvement, code analysis
- **Tech stack:** LangGraph, sandboxed code execution, test framework integration
- **Rohan's gate:** ✅ APPROVED when system passes 70%+ of its own generated test suites on first try on a benchmark of 50 programming tasks

---

# Recommended Courses by Difficulty (External Resources)

These are specific courses mapped to the syllabus phases. You take them at the EXACT moment Maya tells you to — not earlier.

## Beginner Friendly (Tier 1 support)

| Course | Provider | What It Teaches | When to Take |
|---|---|---|---|
| **Python for Everybody** | Coursera (free) | Python fundamentals | Before starting Tier 1 |
| **FastAPI Official Tutorial** | FastAPI docs | API building | Module 1.4 |
| **Prompt Engineering for ChatGPT** | DeepLearning.AI (1 hr) | Systematic prompting | Module 1.3 |
| **Building Systems with ChatGPT API** | DeepLearning.AI (2 hrs) | LLM API patterns | Module 1.2 |

## Intermediate (Tier 2 support)

| Course | Provider | What It Teaches | When to Take |
|---|---|---|---|
| **Building and Evaluating Advanced RAG** | DeepLearning.AI | Production RAG patterns | After Project 5 |
| **RAG from Scratch** | LangChain (YouTube series) | RAG internals | Before Module 2.1 |
| **LangChain for LLM App Development** | DeepLearning.AI | LangChain framework | After manual RAG build |
| **Weaviate Academy** | Weaviate (free) | Vector databases | Module 2.2 |

## Advanced (Tier 3 support)

| Course | Provider | What It Teaches | When to Take |
|---|---|---|---|
| **AI Agents in LangGraph** | DeepLearning.AI | Agent building | Before Project 8 |
| **Functions, Tools and Agents with LangChain** | DeepLearning.AI | Tool use patterns | Module 3.1 |
| **MCP: Build Rich-Context AI Apps** | DeepLearning.AI | MCP | Module 3.4 |
| **Multi AI Agent Systems with CrewAI** | DeepLearning.AI | Multi-agent patterns | Tier 4 |

## Expert Level (Tier 4)

| Course | Provider | What It Teaches | When to Take |
|---|---|---|---|
| **UC Berkeley: Large Language Model Agents** | UC Berkeley | Research-level agent design | Before Project 13 |
| **Full Stack Deep Learning: LLM Bootcamp** | Free recordings | Production LLM system design | Tier 4 |
| **Hugging Face Agents Course** | Hugging Face (free) | Multi-framework agents | Tier 4 |
| **Hugging Face MCP Course** | Hugging Face (free) | MCP mastery | Tier 4 |

## Books Worth Reading (at the right moments)

| Book | Author | Why | When |
|---|---|---|---|
| **AI Engineering** | Chip Huyen | Bridges MLOps and AI engineering — production patterns | Tier 3 |
| **Designing Machine Learning Systems** | Chip Huyen | System design for ML/AI | Tier 3 |
| **Build a Large Language Model (from Scratch)** | Sebastian Raschka | Deep transformer understanding | Tier 3 (optional) |
| **Natural Language Processing with Transformers** | Tunstall, von Werra, Wolf | Hugging Face ecosystem | Tier 3 |

---

# What You'll Be Able to Do at Each Milestone

## After Tier 1 (Projects 1-3): Internal Studio Tools
- ✅ Call LLM APIs with proper error handling
- ✅ Build structured output pipelines with Pydantic
- ✅ Deploy a basic AI API with FastAPI + Docker
- ✅ Understand token economics and cost estimation
- ✅ Build a basic semantic search tool with embeddings
- **Portfolio:** 3 internal tools on GitHub

## After Tier 2 (Projects 4-7): First Real Clients
- ✅ Build production RAG pipelines with hybrid search
- ✅ Choose and tune chunking strategies
- ✅ Implement evaluation with RAGAS/DeepEval
- ✅ Handle document ingestion at moderate scale
- ✅ Implement model routing for cost optimization
- **Portfolio:** 4 client-ready RAG/API projects with evaluations

## After Tier 3 (Projects 8-12): Real Constraints
- ✅ Build agents with tool use and multi-step reasoning
- ✅ Implement advanced RAG (HyDE, agentic, parent-child)
- ✅ Set up full observability stack (tracing, monitoring, alerts)
- ✅ Implement MCP for standardized tool integration
- ✅ Design cost-aware systems with caching and routing
- ✅ Deploy with CI/CD, auth, and security
- **Portfolio:** 5 production-grade projects with monitoring and evaluation

## After Tier 4 (Projects 13-16+): Portfolio-Level
- ✅ Architect multi-agent systems with coordination
- ✅ Implement human-in-the-loop workflows
- ✅ Design comprehensive evaluation frameworks
- ✅ Fine-tune models for specific use cases
- ✅ Build systems that handle 1000+ requests/day
- ✅ Full-stack AI deployment with observability
- **Portfolio:** 4 complex multi-system projects that demonstrate architectural depth
- **Job-ready for:** AI Engineer, LLM Engineer, AI Platform Engineer, AI Product Engineer roles

---

# How the NexaAI System Maps Specifically to This Syllabus

## Delivery Rhythm for Every Project:

```
  ┌──────────────────────────────────────────────┐
  │  1. THE BRIEF (Priya - 5 min)                │
  │  Client name, business problem, what to      │
  │  build, deadline, definition of done          │
  ├──────────────────────────────────────────────┤
  │  2. LEARNING PATH (Maya - 10 min)            │
  │  What concepts you need, learning order       │
  │  (scratch→framework), what to memorize vs     │
  │  look up, first concrete step                 │
  ├──────────────────────────────────────────────┤
  │  3. THE BUILD (You - days/weeks)             │
  │  Build, ask questions, get stuck, decide.     │
  │  Maya guides without solving.                 │
  │  Rohan drops in with hard questions.          │
  │  Priya changes requirements (real world).     │
  ├──────────────────────────────────────────────┤
  │  4. SUBMISSION + REVIEW (Rohan)              │
  │  APPROVED / REVISE / REJECTED with specific   │
  │  feedback against 7-point quality bar.        │
  ├──────────────────────────────────────────────┤
  │  5. DEBRIEF (Maya - 5 min)                   │
  │  What you learned, what will come back        │
  │  harder, what you should now be confident in  │
  └──────────────────────────────────────────────┘
```

## Maya's "Memorize vs Look Up" Guidance Per Module:

| Concept Level | Examples | Action |
|---|---|---|
| **Must memorize** | Messages array structure, LLM API call pattern, Pydantic model syntax, ReAct loop structure, embedding query/upsert pattern | Daily use — drill until automatic |
| **Look up when needed** | HNSW ef_construction, LangGraph node/edge API, specific embedding model dimensions, AWS service configs | Existentially aware, lookup at time of need |
| **Understand deeply** | How attention works, why chunking strategy matters, when to use reasoning vs standard model, sparse vs dense retrieval tradeoffs | Invest time in understanding — this makes you senior |

## How Quality Gets Enforced (Rohan's Review Categories):

| Category | What It Means | Next Action |
|---|---|---|
| ✅ **APPROVED** | Meets all 7 quality checks | Move to next project |
| 🔄 **REVISE** | Works but specific issues found (cost, observability, edge cases, missing eval) | Fix listed issues, resubmit |
| ❌ **REJECTED** | Fundamentally wrong approach | Rethink approach, resubmit with explanation of what changed and why |

## Rohan's 7-Point Quality Bar (applied to EVERY project):
1. **Does it actually work?** — functional baseline
2. **Will it break in production?** — edge cases, error handling, input validation
3. **Is it cost-aware?** — unnecessary token usage, missing caching, no cost controls
4. **Is it observable?** — can you debug this when it fails at 2am?
5. **Is the approach right?** — could a simpler approach solve this?
6. **Are the decisions justified?** — can you explain every architectural choice?
7. **Is quality measurable?** — is there evaluation beyond "I tested it manually"?

---

# The Production Stack You'll Learn (Industry-Verified)

Based on research of what 200+ enterprise AI teams actually use in production (2025-2026):

| Layer | Tools You'll Learn | Production Notes |
|---|---|---|
| **LLM Providers** | OpenAI (GPT-4o, GPT-4o-mini), Anthropic (Claude Sonnet, Opus), open-source via Ollama/vLLM | Most orgs use hybrid — API models for external, open models for internal |
| **Embeddings** | OpenAI text-embedding-3, Voyage AI, sentence-transformers, Cohere | Embedding quality is a major lever |
| **Vector Databases** | Qdrant, Chroma (dev), pgvector, Pinecone, Weaviate | 65% use dedicated vector DBs; pgvector rising |
| **Orchestration** | LangChain, LangGraph, CrewAI, custom Python | LangChain most popular; LangGraph for agents |
| **Protocol** | MCP (Model Context Protocol) | Industry standard for tool integration (Linux Foundation) |
| **API Layer** | FastAPI, Pydantic v2, LiteLLM | FastAPI is the standard for AI services |
| **Evaluation** | RAGAS, DeepEval, Promptfoo, LLM-as-judge | Eval is separating junior from senior engineers |
| **Observability** | Langfuse, LangSmith, OpenTelemetry | #1 investment area for production teams |
| **Deployment** | Docker, AWS Bedrock/GCP Vertex AI, GitHub Actions | CI/CD is essential for AI |
| **Security** | OWASP Top 10 for LLMs, Guardrails AI, PII redaction | Non-negotiable for production |
| **Caching** | Redis, GPTCache/semantic caching | ~40% cost savings in production |
| **Fine-tuning** | LoRA/QLoRA via Hugging Face, Unsloth | When RAG isn't enough for style/format control |

---

# Anticipated Timeline (Realistic)

| Phase | Duration | Prerequisites | Outcome |
|---|---|---|---|
| **Tier 1** (Projects 1-3) | 6-8 weeks | Basic Python | Can build AI APIs and semantic search |
| **Tier 2** (Projects 4-7) | 8-12 weeks | Tier 1 completion | Can build production RAG systems with eval |
| **Tier 3** (Projects 8-12) | 10-14 weeks | Tier 2 completion | Can build agents and production AI systems |
| **Tier 4** (Projects 13-16+) | 12-20 weeks | Tier 3 completion | Can architect complex multi-agent systems |

**Total: 9-14 months** to be genuinely job-ready as an AI Engineer.

*Faster path if you have strong software engineering background: 6-9 months (you can accelerate through Tier 1).*

---

# Final Notes on the System

1. **Projects are the curriculum.** You don't learn AI engineering by watching videos. You learn it by building something real, having Rohan reject it, fixing it, and shipping it.

2. **Frameworks are tools, not the subject.** Every framework in this syllabus (LangChain, LangGraph, CrewAI, etc.) is introduced only AFTER you've built the manual version. This means you understand what problems they solve and can debug when they break.

3. **Evaluation is a core competence.** From Tier 2 onward, every project requires measurable quality metrics. This is what separates engineers who ship reliable systems from engineers who ship demos.

4. **Each project earns the next.** You cannot skip ahead. Rohan reviews every submission. Three REJECTEDs trigger a direct conversation: *"I need to understand what's blocking you."*

5. **Maya researches current patterns.** When a topic needs current techniques, she searches the web and brings back what teams are actually using in production. This keeps the syllabus current without you having to track trends.

---

*"The goal is not to finish projects. The goal is to become an AI engineer who can build anything with AI — and knows why every decision they make is the right one."*
