# Capstone Project Options — Details

This capstone is the course's centerpiece. Unlike tutorial projects, these options are **market-validated** — each one maps to a real product category with measurable production metrics from companies shipping today.

---

## Option 1: AI Customer Support System

**Market Validation:** Zendesk AI ($55/agent/mo), Intercom Fin, Ada (enterprise), Airbnb's support AI. AI customer service market is $4B+ with 300-400% ROI over 3 years documented at scale.

**Real companies shipping this:**
| Company | Result |
|---------|--------|
| Amtrak ("Julie") | Reduced average handling times significantly |
| Bradesco (bank) | 280K questions/month at ~95% accuracy, response from 10min → seconds |
| Airbnb | Seamless AI + human handoff, gold-standard UX |
| SAP | AI-enabled support that anticipates needs, predicts failures |

**Scenario:** A SaaS company gets 500 support tickets/day. Build an AI system that handles Level 1 support autonomously and escalates complex issues to humans.

### Required Components

| Phase | Component | Real-World Equivalent |
|-------|-----------|-----------------------|
| 1 | Multi-provider LLM gateway | LiteLLM, Portkey |
| 2 | Prompt templates per query type | Intercom's Fin |
| 3 | Embeddings for ticket routing | Ada's semantic routing |
| 4 | RAG over support docs | Zendesk Answer Bot |
| 5 | Advanced retrieval + reranking | Glean's enterprise search |
| 6 | Multi-agent (triage, research, respond) | Salesforce Agentforce |
| 7 | Fine-tune on past tickets (optional) |   |
| 8 | Docker + AWS deploy |   |
| 9 | Observability + quality monitoring | Langfuse tracing |
| 10 | Guardrails (no false promises, escalate PII) | Guardrails AI |

### Evaluation Metrics (Industry Benchmarks)

| Metric | Production Target | Why |
|--------|-------------------|-----|
| Auto-resolution rate | >60% | Industry average for L1 support |
| Customer satisfaction | >4.0/5.0 | Matches human-only CSAT |
| Escalation accuracy | >90% | Right issues reach humans, wrong ones don't |
| Response time | <15s | AI handles in seconds (Bradesco: 10min → seconds) |
| Cost per ticket | <$0.10 | AI costs vs $5-15 human cost |
| Human agent handle time reduction | 30-40% | AI copilot assists agents (Assembled data) |

### Architecture

```
User (Chat/Email/API)
    ↓
[Guardrails Layer] → Input filtering, PII redaction, injection detection
    ↓
[Triage Agent] → Categorizes intent, urgency, sentiment
    ↓
┌──────────────────┐
│  Router Agent    │──→ Simple query → [RAG + Generate Answer]
│  (Multi-LLM)     │──→ Complex issue → [Escalate to Human + Context Handoff]
└──────────────────┘    └→ Multi-turn → [Research Agent: search KB, history, tools]
    ↓
[Response Agent] → Formats answer, cites sources, checks quality
    ↓
[Output Guardrails] → Harmful content check, topic relevance
    ↓
User + [Observability: Langfuse trace every step]
```

### Market Positioning

Pricing model: $0.10-0.50 per resolved ticket, or $199/month flat for 2000 tickets.
Target: Mid-market SaaS companies spending $20K+/month on support agents.

---

## Option 2: AI Code Review Assistant

**Market Validation:** CodeRabbit, Qodo, GitHub Copilot ($4.7M paid users), Cursor ($2B ARR). AI coding market hit $12.8B in 2026, projected $30.1B by 2032.

**Real companies shipping this:**
| Product | Metric |
|---------|--------|
| GitHub Copilot | 4.7M paid subscribers, 75% YoY growth |
| Cursor | $2B ARR, 1M+ paying users |
| Claude Code | 46% "most loved" per JetBrains survey |
| CodeRabbit | AI PR reviews catching bugs pre-merge |
| Qodo | Code review + test generation in CI/CD |

**Market stats:**
- 84% of developers use AI tools, 51% daily
- 22% of all merged code is now AI-authored
- AI-coauthored PRs have ~1.7× more issues (opportunity for review!)
- 91% AI adoption across DX developer sample of 135K+

**Scenario:** A development team of 20 engineers merges 30 PRs/week. Build an automated code review assistant that catches bugs, security issues, and style violations before merge.

### Required Components

| Phase | Component | Real-World Equivalent |
|-------|-----------|-----------------------|
| 1 | LLM integration for code analysis | Claude Code, Copilot |
| 2 | Prompt patterns for code review | CodeRabbit prompts |
| 3 | Embeddings for codebase search | Sourcegraph Cody |
| 4 | RAG over best practices + style guides | Qodo knowledge base |
| 5 | Reranking + hybrid search for code |   |
| 6 | Agent that reads files, runs linters, checks deps | Devin agent |
| 7 | Fine-tuned code review model (optional) |   |
| 8 | GitHub App deployment with webhooks |   |
| 9 | Eval: precision/recall of bug detection |   |
| 10 | Security: prevent suggesting vulnerable code |   |

### Evaluation Metrics

| Metric | Production Target | Source |
|--------|-------------------|--------|
| Bug catch rate | >30% of real bugs | CodeRabbit benchmarks |
| False positive rate | <20% | Industry acceptable threshold |
| Review time per PR | <2 min | Human review: 30min-2hr |
| Developer satisfaction | >4.0/5.0 NPS | Survey-based |
| Security issue detection | >40% of OWASP Top 10 | SAST tool benchmarks |
| PR merge rate with AI review green | >90% |   |

### Architecture

```
GitHub Webhook (PR opened / updated)
    ↓
[PR Fetcher] → Get diff, context, changed files
    ↓
[Code Analysis Agent]
    ├── Static Analysis → Run linters (ESLint, PyLint, etc.)
    ├── Security Scan → Check for OWASP patterns, secrets
    ├── RAG Review → Compare against best practices in vector DB
    └── Architecture Review → Check for pattern violations
    ↓
[Comment Generator] → Specific, actionable, line-level feedback
    ↓
[GitHub API] → Post comments on PR
    ↓
[Quality Gate] → Block merge if critical issues found
```

### Market Positioning

Pricing: Free for open-source, $12/developer/month for private repos.
Target: Engineering teams of 10-100 developers currently using manual code review.

---

## Option 3: AI Document Intelligence Platform

**Market Validation:** Glean (enterprise search), Hebbia (financial/legal AI), CustomGPT.ai (used by MIT), Notion AI. Enterprise knowledge management with AI is a rapidly growing segment with proven ROI.

**Real companies shipping this:**
| Company/Product | Result |
|-----------------|--------|
| Cintas + Vertex AI | GenAI-powered internal knowledge center, search across contracts + product docs + customer data in real time |
| Hebbia | AI document Q&A for financial/legal, used by top firms |
| CustomGPT.ai | MIT's entrepreneurship center uses it (ChatMTC). Anti-hallucination focus |
| Glean | Enterprise search across 100+ SaaS apps, grounded answers with sources |
| NotebookLM | Google's research assistant — upload docs, get grounded Q&A + podcast summaries |

**Scenario:** A mid-size company (500 employees) has 50,000+ internal documents across PDFs, Google Docs, Slack, Confluence, and emails. Information is scattered. Build a platform that lets anyone ask questions and get answers with citations from the company's actual documents.

### Required Components

| Phase | Component | Real-World Equivalent |
|-------|-----------|-----------------------|
| 1 | Multi-provider LLM gateway | LiteLLM |
| 2 | Prompt engineering for Q&A with citations | Glean's grounded answers |
| 3 | Embeddings + vector DB | Qdrant / Pinecone |
| 4 | RAG pipeline with hybrid search | Hebbia's retrieval |
| 5 | Advanced RAG (HyDE, reranking, Contextual Retrieval) | Anthropic's Contextual Retrieval |
| 6 | Agent: when to search, when to ask, when to clarify | CustomGPT.ai agent |
| 7 | Fine-tune on company-specific terms (optional) |   |
| 8 | Docker + AWS deploy with auth |   |
| 9 | Observability + eval (faithfulness, recall) | RAGAS, Langfuse |
| 10 | Guardrails: only answer from indexed docs |   |

### Evaluation Metrics

| Metric | Production Target | Why |
|--------|-------------------|-----|
| Faithfulness (RAGAS) | >0.95 | Answer must be grounded in source docs |
| Context recall | >0.90 | Retrieved docs must contain the answer |
| Answer relevance | >0.90 | Answer must address the question |
| Avg latency per query | <3s | User patience threshold |
| Citation accuracy | >95% | Citations must point to correct source |
| Cost per query | <$0.05 | Enterprise scale (1000s of queries/day) |

### Architecture

```
Document Ingestion Pipeline
    PDFs, DOCX, web pages, Slack export, Confluence
    ↓
[Document Processor]
    ├── Text extraction (pdfplumber, python-docx)
    ├── Image detection + GPT-4V description (multimodal)
    ├── Chunking (semantic + recursive)
    └── Contextual Retrieval enrichment
    ↓
[Vector DB] (Qdrant)
    └── Indexed chunks with metadata
    ↓

Query Flow
    User Question
    ↓
[Guardrails] → Off-topic check, input validation
    ↓
[Query Understanding]
    ├── Rewrite ambiguous queries ← HyDE-generated answer
    └── Multi-query generation
    ↓
[Hybrid Search] → Semantic (embedding) + Keyword (BM25) + Metadata filter
    ↓
[Reranker] → Cohere / BGE cross-encoder → Top 3-5 chunks
    ↓
[Generation] → LLM answers with [Source 1], [Source 2] citations
    ↓
[Faithfulness Check] → Verify every claim has a source
    ↓
User ← Answer + Citations
```

### Market Positioning

Pricing: $19/user/month for teams, $custom for enterprise (500+ users).
Target: Any company that has >10,000 internal documents and spends >$50K/year on knowledge management tools.

---

## Comparison Matrix

| Dimension | Customer Support (Option 1) | Code Review (Option 2) | Document Intel (Option 3) |
|-----------|----------------------------|----------------------|--------------------------|
| **Market size** | $4B+ | $12.8B (fastest growing) | ~$2B (enterprise search) |
| **Best for your background** | Full-stack + API design | SWE instincts (native fit) | RAG expertise showcase |
| **Portfolio impression** | "Built a product like Zendesk AI" | "Built what CodeRabbit does" | "Built a competitor to Glean" |
| **Skills demonstrated** | Multi-agent + RAG + eval + deployment | Code analysis + agents + CI/CD | Deep RAG + search + ingestion |
| **Hardest engineering challenge** | Escalation accuracy, multi-turn | False positive rate, latency | Retrieval quality at scale |
| **Cheapest to run** | Medium (LLM calls per ticket) | Cheap (code is short context) | Medium (embedding all docs) |
| **Most shippable in 4 weeks** | Yes | Yes | Yes (with scope control) |

## Recommendation

- **Option 2 (Code Review Assistant)** plays directly to your SWE background, has the hottest market ($12.8B), and is cheapest to operate (code is short context = low token cost). Best portfolio signal.
- **Option 1 (Customer Support)** is the safest bet — most documented, most real-world references, clearest architecture.
- **Option 3 (Document Intelligence)** is the best RAG portfolio piece but requires more data preparation.

**My pick:** Option 2 (Code Review Assistant). You already understand the problem domain deeply, the market is exploding, and it showcases every skill from this course in a package only a senior engineer could build.
