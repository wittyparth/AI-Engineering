# Capstone Project Options — Full Details

Each project option is based on a real product category with production companies already shipping in this space. You're not building a tutorial project — you're building something that could compete with actual products.

---

## Option 1: AI Customer Support System

**Based on real products:** Zendesk AI, Intercom Fin, Ada, Airbnb Support
**Market size:** $4B+ | **Best for:** Full-stack + API design skills

### Real-World Benchmarks

| Metric | What production systems achieve | Your target |
|--------|-------------------------------|-------------|
| Auto-resolution rate | Intercom Fin: 50-70% L1 resolution | >60% |
| Response time | Bradesco's bot: 10min → seconds | <15s |
| CSAT | Zendesk AI: 4.0/5.0 average | >4.0/5.0 |
| Cost per ticket | Human: $5-15 | AI: <$0.10 |
| Human time saved | Assembled data: 30-40% reduction | >30% |

### Real Case Studies to Reference

- **Amtrak "Julie"** — conversational AI that reduced average handling times across US rail support
- **Bradesco (Brazilian bank)** — virtual agent handles 280K questions/month at ~95% accuracy
- **SAP** — AI-enabled support that predicts failures and anticipates needs before customers report them
- **Airbnb** — Gold-standard AI chatbot UX with seamless human escalation

### Required Components (Mapped to Course Phases)

| Phase | Component | What You'll Build |
|-------|-----------|-------------------|
| 1 | Multi-provider LLM gateway | Route to cheapest provider per query type |
| 2 | Prompt templates per query type | System prompts for billing, tech support, account issues |
| 3 | Vector DB for product knowledge | Index all support docs, FAQs, policy pages |
| 4 | RAG over support docs | Retrieve relevant docs per ticket |
| 5 | Advanced retrieval + reranking | Improve RAG quality with HyDE + reranker |
| 6 | Multi-agent system | Triage agent → Research agent → Response agent |
| 7 | Fine-tune on past tickets (optional) | Domain adaptation for company-specific language |
| 8 | Production deployment | Docker Compose → AWS ECS |
| 9 | Observability | Trace every ticket resolution through Langfuse |
| 10 | Guardrails | Block false promises, detect PII, escalate sensitive topics |

### Architecture

```
User (Chat widget / Email / API)
    ↓
[Guardrails] → Input filter, PII redaction, injection check
    ↓
[Triage Agent] → Classifies: "billing" / "tech" / "account" / "other"
               → Scores urgency: LOW / MED / HIGH
               → Detects sentiment: positive / neutral / frustrated
    ↓
┌──────────────────────────────────────────┐
│         Supervisor Agent                 │
│  (Decides: self-resolve or escalate)     │
└──────────────────────────────────────────┘
    ↓                          ↓
[Self-Serve Path]         [Escalation Path]
  RAG → Answer             Context package → Human queue
  Citation check           Sentiment + history attached
  Quality gate             Agent sees full context
    ↓                          ↓
[Response Agent] ← ← ← ← [Human Agent responds]
  Formats final answer
  Polishes tone
    ↓
[Output Guardrails] → Harmful content check, topic relevance
    ↓
User receives answer
```

---

## Option 2: AI Code Review Assistant

**Based on real products:** CodeRabbit, Qodo, GitHub Copilot, Cursor
**Market size:** $12.8B (fastest-growing AI category) | **Best for:** Your SWE background

### Real-World Benchmarks

| Metric | What production systems achieve | Your target |
|--------|-------------------------------|-------------|
| Bug catch rate | CodeRabbit: catches bugs pre-merge | >30% of real bugs |
| False positive rate | Industry acceptable: <20% | <20% |
| Review time | Human: 30min-2hr per PR | <2 min |
| AI adoption | 84% developers use AI tools, 51% daily | N/A |
| AI-authored code | 22% of all merged code (DX sample, 2025) | N/A |

### Market Context

> "The AI coding assistant market reached $12.8B in 2026. 85% of developers use AI tools. GitHub Copilot has 4.7M paid subscribers. Cursor hit $2B ARR."
> — IdeaPlan Market Report, 2026

> "AI-coauthored PRs show ~1.7× more issues than human-only PRs. This is the product opportunity — AI creates code faster, but it needs AI review even more."
> — Panto AI Coding Statistics, 2026

### Required Components (Mapped to Course Phases)

| Phase | Component | What You'll Build |
|-------|-----------|-------------------|
| 1 | LLM integration for code analysis | Analyze PR diffs with GPT-4o-mini |
| 2 | Prompt patterns for code review | Review prompts: correctness, security, style, performance |
| 3 | Embeddings for codebase search | Index your codebase for context-aware review |
| 4 | RAG over best practices | Retrieve relevant patterns from a best-practices vector DB |
| 5 | Reranking for code snippets | Rank findings by severity |
| 6 | Agent: read files, run linters, check deps | Multi-tool agent that audits the PR |
| 7 | Fine-tune on your codebase (optional) | Adapt to team conventions |
| 8 | GitHub App deployment | Webhook-driven, posts line-level comments |
| 9 | Eval suite: precision, recall, user satisfaction | Track every review against human reviewer findings |
| 10 | Security: block vulnerable suggestions | OWASP pattern check in generated suggestions |

### Architecture

```
PR opened / updated (GitHub webhook)
    ↓
[PR Analyzer] → Fetch diff, base branch, changed files, commit messages
    ↓
┌─────────────────────────────────────────────┐
│         Code Review Agent Suite             │
│                                             │
│  [Static Analysis Agent]                    │
│    → Run ESLint, PyLint, Bandit             │
│    → Check for secrets in diff              │
│                                             │
│  [Security Agent]                           │
│    → OWASP Top 10 pattern matching          │
│    → SQL injection, XSS, hardcoded keys     │
│                                             │
│  [Best Practices Agent]                     │
│    → RAG over team's style guide + patterns │
│    → Naming, structure, error handling      │
│                                             │
│  [Test Coverage Agent]                      │
│    → Are there tests for this change?       │
│    → Do existing tests need updating?       │
└─────────────────────────────────────────────┘
    ↓
[Severity Classifier] → Critical / Major / Minor / Nit
    ↓
[GitHub API] → Post line-level comments + summary
    ↓
[Quality Gate] → Block merge if critical issues found
```

---

## Option 3: AI Document Intelligence Platform

**Based on real products:** Glean, Hebbia, CustomGPT.ai, Notion AI, NotebookLM
**Market size:** Enterprise knowledge management ($2B+) | **Best for:** Deep RAG showcase

### Real-World Benchmarks

| Metric | What production systems achieve | Your target |
|--------|-------------------------------|-------------|
| Faithfulness | CustomGPT.ai: anti-hallucination focus | >0.95 RAGAS faithfulness |
| Context recall | Need to find relevant doc for every query | >0.90 |
| Citation accuracy | Glean: source-grounded answers | >95% valid citations |
| Latency | User patience threshold: <3s | <3s p95 |
| Cost | Enterprise scale: 1000s queries/day | <$0.05/query |

### Real Case Studies to Reference

- **Cintas + Google Cloud** — Built a GenAI-powered internal knowledge center using Vertex AI, enabling service/sales teams to search contracts, product docs, and customer interaction data in real time
- **MIT Entrepreneurship Center** — Uses CustomGPT.ai (ChatMTC) for accurate answers from their knowledge base with no hallucination
- **NotebookLM** — Google's research assistant: upload docs, get grounded Q&A + generates podcast-style audio summaries

### Required Components (Mapped to Course Phases)

| Phase | Component | What You'll Build |
|-------|-----------|-------------------|
| 1 | Multi-provider LLM gateway | Route queries to cheapest capable model |
| 2 | Prompt engineering for grounded Q&A | System prompt enforcing citation-only answers |
| 3 | Embeddings + vector DB | Index all documents with semantic search |
| 4 | RAG pipeline with hybrid search | Semantic + BM25 keyword search |
| 5 | Advanced RAG (HyDE, reranking, contextual retrieval) | Anthropic's contextual retrieval technique |
| 6 | Agent: decides when to search, ask, or clarify | Smart query understanding |
| 7 | Fine-tune on domain-specific terms (optional) | Legal/medical/financial terminology |
| 8 | Production deployment | Docker + AWS with auth |
| 9 | Observability + RAGAS eval | Faithfulness, recall, precision dashboards |
| 10 | Guardrails: only answer from indexed docs | Topic boundaries, hallucination prevention |

### Architecture

```
Document Ingestion Pipeline:
    PDFs / DOCX / Web Scrape / Slack Export / Confluence
    ↓
[Document Processor]
    ├── Text extraction (pdfplumber, python-docx, beautifulsoup)
    ├── Image analysis (GPT-4V describes charts/diagrams)
    ├── Semantic chunking (split on topic changes, not character count)
    └── Contextual enrichment (Anthropic's technique: add document context to each chunk)
    ↓
[Qdrant / ChromaDB] → Indexed chunks with metadata (source, date, author, tags)

Query Flow:
    User: "What's our refund policy for international orders?"
    ↓
[Guardrails] → Topic check, input validation, PII redaction
    ↓
[Query Understanding Agent]
    ├── Intent classification: factual / comparative / procedural
    ├── Entity extraction: "international orders" → search filter
    ├── Query rewriting: "refund policy for orders outside US"
    └── HyDE generation: LLM generates hypothetical answer first
    ↓
[Hybrid Search] → Embedding similarity + BM25 keyword + metadata filter
    ↓
[Reranker] → Cohere / BGE cross-encoder → Top 3-5 most relevant chunks
    ↓
[Generation Agent] → Answer with [Source 1], [Source 2] citations
    ↓
[Faithfulness Check] → LLM-as-judge: every claim must trace to a source
    ↓
User ← "International orders can be refunded within 30 days [Source 1]. Contact support@company.com for return labels [Source 3]."
```

---

## Option 4: Recall — Passive Knowledge Base Browser Extension

**Based on real products:** Rewind AI, Readwise Reader, Mindweave, Mem AI, Sider AI (Wisebase)
**Market size:** Personal AI knowledge ($500M+ emerging) | **Best for:** Local-first AI, browser extension architecture, on-device ML

### The Core Insight

Every knowledge worker reads thousands of articles, docs, and posts per year. They remember almost none of it. Existing tools require manual action to save or highlight — which means most reading never gets captured.

Recall is the **passive, local-first answer**: a browser extension that silently indexes everything you read into a private vector knowledge base. When you're writing, coding, or researching, a sidebar surfaces relevant passages from your reading history. No buttons. No clips. No highlighting. Reading IS the capture mechanism.

### Real-World Benchmarks

| Metric | What production systems achieve | Your target |
|--------|-------------------------------|-------------|
| Search latency | Rewind: <1s for local search | <500ms p50, <2s p95 |
| Retrieval precision | Readwise: keyword-only (no semantic) | >85% precision@5 |
| Capture rate | Readwise: needs manual save (captures <10% of reading) | >90% passive capture (auto of qualifying pages) |
| Privacy | Rewind: 100% local | 100% local — zero external data flow |
| Cost per user | Rewind: $30-50/mo | $0-5/mo (local compute is free) |
| Active recall | No competitor does contextual sidebar | Suggested passages while writing |

### Competitive Landscape at a Glance

| Product | Capture | Semantic Search | Sidebar Recall | Local-First | Pricing |
|---------|---------|----------------|---------------|-------------|---------|
| **Recall (you)** | ✅ Passive (auto) | ✅ Full-text + vector | ✅ Contextual auto-suggest | ✅ 100% local | Free / $5-12/mo |
| Rewind AI | ✅ Passive (screen) | ✅ | ❌ Desktop app only | ✅ | $30-50/mo |
| Readwise Reader | ❌ Manual highlight | ❌ Keyword only | ❌ Separate app | ❌ Cloud | $9.99/mo |
| Mindweave | ❌ 1-click clip | ✅ | ❌ | ❌ Cloud | Free (OSS) |
| Mem AI | ❌ Manual save | ✅ | ❌ | ❌ Cloud | $15/mo |
| Sider AI | ❌ 1-click clip | ✅ | ✅ Manual search | ❌ Cloud | Freemium |
| Personal AI Memory | ✅ Auto (AI convos only) | ✅ | ❌ | ✅ | Free (OSS) |

Your product is uniquely positioned: **passive + reading-only + local-first + contextual sidebar recall.** No competitor occupies this quadrant.

### Required Components (Mapped to Course Phases)

| Phase | Component | What You'll Build |
|-------|-----------|-------------------|
| 1 | Multi-provider LLM gateway | Route Q&A queries to cheapest capable model (Phase 3+) |
| 2 | Prompt engineering for citation | System prompt enforcing "only answer from indexed passages" |
| 3 | Embeddings + vector DB (local) | Index every page's readable content with semantic search — all in-browser |
| 4 | RAG pipeline (browser-executed) | Retrieve relevant passages from personal reading history |
| 5 | Advanced retrieval + reranking | Hybrid search (vector + BM25), cross-encoder reranking (local ONNX) |
| 6 | Simple agent: understand query intent | Decide if user is searching, asking, or needs context suggestions |
| 7 | Fine-tuning (optional - Phase 3) | Adapt embedding model to user's reading domains |
| 8 | Extension deployment | Chrome Web Store → Firefox → Safari |
| 9 | Observability (local dashboard) | Retrieval latency, index status, storage usage, zero-result queries |
| 10 | Guardrails: never hallucinate | Block answers not grounded in user's index; "I don't know" default |

### Architecture

```
Browser Extension Architecture:

┌─────────────────────────────────────────────────────────┐
│                    Browser Extension                      │
│                                                           │
│  ┌──────────────┐   ┌─────────────────────────────────┐  │
│  │ Content Script│   │    Background Service Worker    │  │
│  │ (per-tab)     │   │                                 │  │
│  │ - Readability │──▶│ - Text cleaning + chunking      │  │
│  │ - Scroll hook │   │ - Embedding (ONNX/WASM)         │  │
│  │ - Timer logic │   │ - DB operations                 │  │
│  └──────────────┘   │ - Search orchestration           │  │
│                      └──────────┬──────────────────────┘  │
│  ┌──────────────┐              │                         │
│  │  Sidebar UI  │◀─────────────┘                         │
│  │  (React)     │                                         │
│  │  - Search    │   ┌──────────────────────────────────┐  │
│  │  - Results   │   │       Local Storage Layer         │  │
│  │  - Contextual│   │  ┌─────────┐  ┌──────────────┐   │  │
│  └──────────────┘   │  │ SQLite  │  │  Vector DB   │   │  │
│                      │  │(metadata│  │ (ChromaDB    │   │  │
│                      │  │ + FTS)  │  │  or DuckDB)  │   │  │
│                      │  └─────────┘  └──────────────┘   │  │
│                      └──────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Data Flow

```
READ FLOW:
1. User opens technical article → Content script injected
2. Timer starts (15s default) → user scrolls >50%
3. Readability extracts clean content (strips nav/ads)
4. Background worker: chunks → ONNX embeddings → local vector DB
5. SQLite stores: title, URL, metadata, full text, tags

SEARCH FLOW:
1. User types in sidebar: "Kafka partitioning strategies"
2. Query embedded locally (same ONNX model)
3. Vector search: top-15 similar passages
4. Rerank: semantic similarity × recency × source authority
5. Return top-8 results with highlighted excerpts

ACTIVE RECALL FLOW:
1. User writing design doc in Google Docs
2. Context capture: last 200 words written
3. Embed context → search reading index
4. Sidebar auto-shows: "Based on your writing..." cards
5. User clicks → citation inserted with source link
```

### Key Technical Challenges

**Challenge 1: MV3 Service Worker Limitations**
Extension service workers have a 30s startup and 5min idle timeout. Mitigation: IndexDB persistence, alarms API for scheduled tasks, persistent connections for long operations.

**Challenge 2: Local ONNX Embedding Performance**
Running an embedding model in-browser requires WASM/WebNN. Must handle: GPU vs CPU detection, model download (5-20MB), quantized inference, fallback if hardware unsupported.

**Challenge 3: Content Extraction Quality**
Mozilla Readability works for blogs but fails for: JS-heavy SPAs, paywalled content, academic PDFs, documentation sites. Will need per-site extraction configs and a fallback pipeline.

**Challenge 4: Cross-Domain Sidebar Injection**
Sidebar must work in Google Docs (shadow DOM), Notion (canvas), GitHub (dynamic content). Requires mutation observers and per-app injection strategies.

### MVP Scope

**Deliver in 6-8 weeks:**
- ✅ Passive content indexing (Chrome, Mozilla Readability)
- ✅ Local ONNX embedding (all-MiniLM-L6-v2 via Xenova/Transformers.js)
- ✅ ChromaDB vector storage in IndexedDB
- ✅ Sidebar with semantic search
- ✅ Basic result display (title, URL, passage excerpt, date read)
- ✅ Manual sidebar toggle (Ctrl+Shift+R)
- ✅ Content extraction exclusions config
- ✅ Chrome Web Store publish

**Phase 2 (next 4-6 weeks):**
- Active recall / contextual suggestions while writing
- Google Docs, Notion, GitHub injection support
- Cross-encoder reranking (local ONNX)

**Phase 3 (next 4-6 weeks):**
- Local LLM Q&A (Phi-3-mini via WebLLM)
- Weekly digest of forgotten-but-relevant passages
- Auto-tagging and collections

### Why This Project

1. **You use this every day.** As an engineer who reads extensively, you are the target user. Dogfooding is natural.
2. **Teaches local-first AI.** Most AI projects use cloud APIs. Recall forces you to master on-device ML, ONNX, WASM — skills few engineers have.
3. **Browser extension architecture.** Chrome extensions have unique constraints (MV3, service workers, IndexedDB, content scripts) different from backend services.
4. **Competitive moat.** No product does passive + reading-only + local-first + contextual recall. There's room to win.
5. **Portfolio punch.** "I built a local RAG system that runs entirely in the browser" is a stronger signal than "I called OpenAI's API."

**Full Product Spec:** [Recall-PRD.md](file:///C:/Users/MunakalaParthaSaradh/Desktop/AI%20course/AI-Course-Mastery/11-Capstone/Recall-PRD.md)

---

## Quick Comparison

| | Customer Support | Code Review | Document Intel | Recall |
|---|---|---|---|---|
| **Market** | $4B+ | $12.8B ⚡ | $2B+ | $500M+ emerging |
| **Your advantage** | Full-stack → API + chat UI | SWE → you know code quality | RAG expertise | Local-first + browser ext |
| **Cheapest to operate** | Medium | ✅ Cheapest (code = short) | Medium (embedding costs) | ✅ Zero (local-only) |
| **Portfolio line** | "Built what Zendesk AI does" | "Built what CodeRabbit does" | "Built a competitor to Glean" | "Built what Rewind should have been" |
| **Best if you want to...** | Ship to real users fastest | Build something you'd use daily | Deepen RAG expertise | Master local AI + build an extension |

---

## How to Choose

| If you... | Pick this option |
|-----------|-----------------|
| Want to ship to real users fastest | **Customer Support** — chat UI is familiar, value is immediate |
| Are a SWE who wants to build something you'd use every single day | **Code Review** — your own PRs will never be the same |
| Want the deepest RAG expertise | **Document Intel** — ingest, chunk, retrieve, evaluate at enterprise scale |
| Want to master local AI, browser extensions, and on-device ML | **Recall** — the hardest but most differentiated build |

---

**Pick one. Build it. Ship it. Your portfolio depends on it.**
