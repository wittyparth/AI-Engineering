# Real Company AI Stacks: Case Studies

> Research analysis of how real companies deploy AI in production (2025-2026). These patterns directly inform the syllabus projects.

## 1. Enterprise Legal Q&A (Fortune 500 Manufacturer)

**The Problem:** Legal department had 50,000+ documents across contracts, regulations, and case law. Lawyers spent hours searching for precedents manually.

**The Solution (Production RAG):**
- **Ingestion pipeline** (80% of effort): PDF parsing with OCR fallback, metadata extraction, parent-child chunking (512-token child chunks, 2000-token parent chunks)
- **Chunking:** Semantic chunking (not fixed-size) using sentence boundaries with 10% overlap
- **Embeddings:** OpenAI text-embedding-3-small (cost optimized)
- **Vector DB:** Qdrant with HNSW indexing
- **Search:** Hybrid — 70% dense (embedding) + 30% sparse (BM25) weighting, plus cross-encoder reranking
- **Generation:** GPT-4o-mini with strict citation enforcement
- **Evaluation:** RAGAS metrics (faithfulness, answer relevancy, context precision)
- **Observability:** Langfuse tracing for every query

**Key lessons learned:**
- Hybrid search was the single biggest quality improvement
- OCR quality was the biggest bottleneck (not AI, physical document scanning)
- Chunking strategy mattered more than model choice
- Evaluation was the hardest part to get right
- Cost optimization (caching, model selection) saved ~60% vs naive implementation

**Relevance:** This is the basis for **Project 4 (Legal Document Q&A System)** in Tier 2.

---

## 2. Wix's Multi-Agent RAG System (Anna)

**The Problem:** Wix had data spread across multiple silos. Users couldn't get answers that required joining information across datasets.

**The Solution (Multi-Agent RAG):**
- **7 specialized agents** for different data domains
- **Agent roles:** Root intent detection → Question validation → Query division → Data retrieval across 7 indexes → Response synthesis
- **Orchestrator-worker pattern** — a central orchestrator decides which agents to call
- **Accuracy improvement:** 72% → 96.8% across progressive optimizations

**Key lessons learned:**
- Multi-agent doesn't mean fully autonomous — orchestrator-worker pattern dominated
- Each agent needed a clear, narrow role
- Inter-agent data contracts were critical for reliability
- Evaluation required per-agent metrics + holistic system metrics

**Relevance:** This architecture informs **Projects 13-14 (Deep Research Agent, Multi-Agent Content Pipeline)** in Tier 4.

---

## 3. Anthropic's Field Research on AI Agents

**The Finding (from Anthropic's 2025 engineering blog):** After analyzing hundreds of production agent deployments:

**Most effective patterns (ordered by complexity):**
1. **Augmented LLM** — LLM + retrieval + tools (80% of use cases stop here)
2. **Prompt chaining** — deterministic decomposition into steps
3. **Routing** — classify → route to specialized handler
4. **Parallelization** — multiple simultaneous LLM calls
5. **Orchestrator-workers** — central coordinator + sub-agents
6. **Evaluator-optimizer** — generator + evaluator loop

**Key findings:**
- The most effective agents are **simple** — augmented LLMs doing one thing well
- Most production systems use **prompt chaining**, not autonomous agents
- Multi-agent is **overkill until a single agent demonstrably fails**
- The secret to good agents is **well-designed tools**, not clever prompting
- **Hard limits on steps and cost** are non-negotiable in production

**Relevance:** This is the core teaching for **Module 3.1 (AI Agents from Scratch)** and **Tier 3** projects.

---

## 4. Financial Services RAG (Regulatory Compliance)

**The Problem:** A financial firm needed to query earnings reports, regulatory filings, and market analysis. Zero tolerance for hallucinated figures. Must handle tables, cross-reference documents, and abstain from answering when uncertain.

**The Solution (Advanced Agentic RAG):**
- **Query rewriting** — LLM rewrites user queries to be more retrieval-friendly
- **HyDE** — generates hypothetical documents as better search queries
- **Multi-hop retrieval** — some answers required 2-3 rounds of retrieval
- **Vision model** for table extraction from PDFs
- **Strict abstention** — system trained to say "I don't know" when insufficient evidence
- **Citation verification** — cross-references each claim with source document

**Key lessons learned:**
- Query rewriting improved retrieval by 15-30%
- System abstenstion built user trust (accepting "I don't know" > hallucination)
- Table extraction from PDFs required vision models — OCR alone wasn't enough
- Cross-document questions required agentic (multi-step) retrieval

**Relevance:** This is the basis for **Project 9 (Advanced Financial RAG System)** in Tier 3.

---

## 5. MCP: From Experiment to Industry Standard

**The Timeline:**
- **Nov 2024:** Anthropic announces Model Context Protocol (MCP)
- **2025:** Every major AI provider adopts it (OpenAI, Google, AWS, Microsoft)
- **Dec 2025:** Donated to Linux Foundation as industry standard
- **Early 2026:** 13,000+ public MCP servers available

**What MCP Provides:**
- Standardized interface: Host → Client → Server
- Built-in security model (OAuth 2.1 with PKCE)
- Resource exposure (files, APIs, databases) through uniform API
- Tool standardization — any MCP-compatible agent can use any MCP server

**Relevance:** MCP is covered in **Module 3.4 (Model Context Protocol)** and used in Tier 4 projects.
