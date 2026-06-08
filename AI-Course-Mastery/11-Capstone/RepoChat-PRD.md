# RepoChat — Complete Product Specification
### *NotebookLM for Your GitHub Repository*

> **Tagline:** Every codebase has stories. RepoChat lets you ask them.

---

## Table of Contents

1. [Vision & Philosophy](#1-vision--philosophy)
2. [The Problem — In Depth](#2-the-problem--in-depth)
3. [Product Overview](#3-product-overview)
4. [User Personas](#4-user-personas)
5. [Core Features — Full Detail](#5-core-features--full-detail)
6. [User Experience & Flows](#6-user-experience--flows)
7. [Technical Architecture](#7-technical-architecture)
8. [AI Pipeline Design](#8-ai-pipeline-design)
9. [Data Model](#9-data-model)
10. [API Design](#10-api-design)
11. [Evaluation & Quality](#11-evaluation--quality)
12. [Security & Privacy](#12-security--privacy)
13. [Observability & Monitoring](#13-observability--monitoring)
14. [MVP Scope & Phased Roadmap](#14-mvp-scope--phased-roadmap)
15. [Monetization](#15-monetization)
16. [Launch Strategy](#16-launch-strategy)
17. [Success Metrics](#17-success-metrics)
18. [Tech Stack](#18-tech-stack)

---

## 1. Vision & Philosophy

### The Core Insight

Cursor and Claude Code are exceptional at *writing* code. But there is a completely separate, equally painful problem that no tool solves well: *understanding* code you didn't write, can't fully remember, or inherited from someone who left.

RepoChat is not a coding assistant. It is a **knowledge interface for codebases** — the difference between a library (where you retrieve known books) and a librarian (who understands what you're really asking and finds the answer across everything in the building).

### The NotebookLM Analogy — Why It Works

NotebookLM's breakthrough was not technical. It was a design philosophy: **ground every answer exclusively in the sources the user provides, with citations to the exact source passage.** No hallucination. No general knowledge bleed. Pure faithfulness to the source material.

RepoChat applies this philosophy to code:

- Every answer is grounded in the actual repository
- Every claim cites the exact file, line range, and commit
- The system refuses to speculate beyond what the codebase shows
- Users can trust the output enough to act on it

### Design Principles

**Principle 1: Understanding over generation.** RepoChat never writes code. It only explains, explores, and connects. This keeps it deeply focused and avoids competing with Cursor.

**Principle 2: Citations are non-negotiable.** Every answer surfaces the exact file:line reference. Clicking it opens the file in the right place. This is what separates RepoChat from "ask ChatGPT about your code."

**Principle 3: Questions, not search.** Developers shouldn't have to remember filenames or function names to explore a codebase. They should be able to ask in natural language: "Why does auth work differently for admin users?" and get an intelligent, grounded answer.

**Principle 4: Context accumulates.** Like a real conversation, follow-up questions build on prior answers within a session. "And what would break if we changed that?" works naturally.

**Principle 5: Multi-source truth.** Code alone is incomplete. The *why* lives in PR descriptions, issue comments, commit messages, and changelogs. RepoChat synthesizes across all of them.

---

## 2. The Problem — In Depth

### The Hidden Tax on Engineering Teams

Studies consistently show engineers spend **40–60% of their time reading and understanding code** rather than writing it. This number is higher for:

- New hires onboarding to a codebase
- Developers working with unfamiliar modules
- Open source contributors trying to understand library internals
- Engineers debugging a dependency they didn't write
- Tech leads doing code review across multiple services

This is not a small inconvenience. For a 10-person engineering team, it represents thousands of hours per year of time where the developer is stuck in the understanding phase, not the building phase.

### Why Existing Tools Fail

**GitHub Search:** Keyword-based. You need to already know what you're looking for. Can't answer "why does this exist" or "what would break if I change this."

**Cursor / Copilot:** Built for writing code, not reading it. Chat with codebase features exist but are chat-first UX bolted onto an IDE, not a dedicated understanding interface. Also require the IDE to be open.

**Grep / ripgrep:** Fast but dumb. No semantic understanding. No synthesis. Returns lines, not explanations.

**Documentation:** When it exists, it's always out of date. There's always a gap between what the docs say and what the code actually does.

**Asking a senior engineer:** The most effective current solution — and completely unscalable. The senior engineer who wrote the code has left. Or is in a different timezone. Or is busy.

**Confluence / Notion:** Where knowledge goes to die. Static documents that are never updated when code changes.

### The Real Pain Points (Exact Scenarios)

1. *"I joined this team 3 weeks ago and I'm afraid to touch the payment service because I don't understand how it connects to auth and billing."*
2. *"This PR touches 14 files and I have no idea if it's safe to merge. I need to understand what these modules do before I approve."*
3. *"The bug is somewhere in the order processing flow but I can't tell where because I don't know which service owns what."*
4. *"I want to add a new feature but I don't know if we already have a utility for this somewhere in the codebase."*
5. *"Why is this implemented with Redis here instead of the database? There must have been a reason. The engineer who wrote it is gone."*

RepoChat solves all five. Directly.

---

## 3. Product Overview

### What RepoChat Is

RepoChat is a web application that lets you connect a GitHub repository and immediately start having an intelligent conversation about it. You ask questions in plain English. It answers with grounded explanations, exact file citations, code snippets, and reasoning chains — synthesizing across code, PRs, issues, commits, and documentation simultaneously.

### What Makes It Different — The 5 Differentiators

**1. Multi-source synthesis.** Most "chat with codebase" tools only index source files. RepoChat indexes code + commit messages + PR titles and descriptions + issue threads + README and docs + CHANGELOG + inline comments. This is what enables it to answer "why" questions, not just "what" questions.

**2. Citation-first design.** Every answer includes clickable citations that take you to the exact file:line in GitHub. This is the core trust mechanism.

**3. Semantic code understanding.** Beyond text search, RepoChat understands code semantically — it knows that `user_id` in one file refers to the same concept as `userId` in another, that a function called `processPayment` is related to the `Payment` model even if they don't share a filename, that an error handling pattern in one service should probably match another.

**4. Faithfulness guardrails.** The system is designed to say "I don't see evidence of that in this codebase" rather than speculate. It validates answers against retrieved context before returning them.

**5. Developer UX, not product UX.** The interface is fast, keyboard-first, and built for developers. Not a chatbot with a profile picture. A serious research tool.

---

## 4. User Personas

### Persona 1: Alex — The New Engineer
*Software engineer, 2 years experience, just joined a startup with a 3-year-old codebase*

**Pain:** Spends 3–4 hours every day reading code trying to understand the system before touching anything. Too embarrassed to ask the same questions repeatedly.

**Use case:** Onboarding accelerator. Asks RepoChat to explain every service, the data flow between them, and the history behind unusual architectural decisions.

**Core jobs:**
- "Explain what this service does and how it connects to the others"
- "Walk me through what happens when a user signs up"
- "Why is this module structured differently from the others?"

### Persona 2: Priya — The OSS Contributor
*Senior engineer, contributing to a large open source project she didn't build*

**Pain:** The codebase has 200k lines. Documentation is patchy. Issues reference PRs from 2 years ago. Understanding context takes days.

**Use case:** Research companion. Asks targeted questions about specific behaviors before submitting a PR she's confident about.

**Core jobs:**
- "Has anyone already tried to add support for X?"
- "What's the expected behavior of this function when input is null?"
- "Show me all the places this interface is implemented"

### Persona 3: Marcus — The Tech Lead
*Principal engineer doing code review and architecture decisions*

**Pain:** Reviews 5–10 PRs a week across services he doesn't always know well. Has to context-switch constantly. Often approves code he doesn't fully understand.

**Use case:** Code review research. Before reviewing a PR, asks RepoChat about the module being modified to understand the existing patterns and invariants.

**Core jobs:**
- "What are the invariants in this module that this PR might violate?"
- "Has this approach been tried before? What happened?"
- "What other parts of the codebase depend on this interface?"

### Persona 4: Zara — The Debugging Engineer
*Mid-level engineer chasing a production bug across services*

**Pain:** The bug crosses service boundaries. She needs to trace the flow of data across 4 services and 12 files to find where it breaks.

**Use case:** Debugging compass. Asks RepoChat to trace data flows and identify where a specific field is transformed or dropped.

**Core jobs:**
- "Trace the flow of `order_id` from the API endpoint to the database write"
- "Where could the `status` field be set to null unexpectedly?"
- "Which services write to the `orders` table?"

---

## 5. Core Features — Full Detail

### Feature 1: Repository Indexing Engine

**What it does:** Connects to a GitHub repository (public or private via OAuth) and builds a rich, multi-source index optimized for semantic retrieval.

**What gets indexed:**
- All source code files (with language-aware chunking — functions, classes, and modules are kept as semantic units rather than arbitrary token windows)
- Git commit history — message, author, timestamp, files changed, diff summary
- Pull request titles, descriptions, review comments, and linked issues
- Issue titles and bodies, with comment threads
- README, docs folder, CHANGELOG, CONTRIBUTING guides
- Inline code comments (extracted separately from code, weighted differently)
- GitHub release notes

**Chunking strategy:**
- Code: AST-aware chunking. Functions are never split mid-body. Class definitions keep their method list. Import blocks are kept together. Average chunk: 30–80 lines.
- Prose (docs, PR descriptions, issues): Paragraph-level chunking with overlap. Average chunk: 150–300 tokens.
- Commits: Each commit is a single document with metadata (hash, date, author, message, list of files changed).
- Comments: Each inline comment is a single document linked to its file and line.

**Incremental indexing:** After initial index, the system webhooks GitHub for new commits, PRs, and issues. Only changed files and new events are re-indexed. Full re-index is triggered on force-push.

**Index metadata stored per chunk:**
```
{
  "id": "uuid",
  "repo_id": "uuid",
  "source_type": "code|commit|pr|issue|doc|comment",
  "file_path": "src/auth/middleware.py",
  "line_start": 45,
  "line_end": 89,
  "language": "python",
  "commit_sha": "abc123",
  "author": "priya@company.com",
  "timestamp": "2024-01-15T14:23:00Z",
  "embedding": [0.023, ...],  // 1536-dim
  "text": "...",
  "bm25_tokens": [...]
}
```

**Supported languages (Phase 1):** Python, JavaScript, TypeScript, Go, Rust, Java, Ruby, PHP, C#
**Supported languages (Phase 2):** All languages with tree-sitter grammars

### Feature 2: Hybrid Search Retrieval

**What it does:** Given a natural language question, retrieves the most relevant chunks across all source types using a hybrid approach that combines semantic similarity and keyword precision.

**Search pipeline:**

```
User Question
     │
     ├─── Query Expansion (LLM generates 3 alternative phrasings)
     │
     ├─── Semantic Search (dense retrieval via embeddings)
     │         └── Qdrant HNSW index, top-30 results
     │
     ├─── Keyword Search (BM25 over code tokens and identifiers)
     │         └── Elasticsearch/Typesense, top-30 results
     │
     ├─── Reciprocal Rank Fusion (merge and re-rank)
     │         └── Combined top-15 candidates
     │
     ├─── Cross-Encoder Re-ranking (semantic relevance scoring)
     │         └── Top-8 final context chunks
     │
     └─── Context Assembly (structured prompt with citations)
```

**Query expansion rationale:** "Why does auth fail for admin users?" expands to include "authentication error admin role," "permission denied administrator," and "auth middleware admin bypass" — capturing different ways the codebase might reference this concept.

**Source type boosting:** By default, code chunks and PR descriptions are boosted slightly over issue comments. The user can override this ("focus only on code" or "show me what PRs said about this").

**Identifier-aware tokenization:** Code identifiers are tokenized both as whole tokens (`processPayment`) and split by convention (`process`, `Payment`) so BM25 catches both exact function name matches and semantic keyword matches.

### Feature 3: Grounded Answer Generation

**What it does:** Takes the retrieved context chunks and generates a precise, grounded answer with inline citations.

**Answer structure:**
```
Answer: [Natural language explanation]

Referenced in:
  [1] src/auth/middleware.py:45-89 — AuthMiddleware class
  [2] PR #247 (2024-01-10) — "Refactor admin auth flow"
  [3] commit abc123 (2024-01-08) — "Fix admin permission check"

Related:
  • src/auth/permissions.py:12 — Permission model definition
  • src/users/models.py:78 — Admin flag on User model
```

**Faithfulness enforcement:**
- The LLM is explicitly instructed: "Only answer based on the provided context. If the context does not contain enough information to answer, say so explicitly."
- A faithfulness validator runs on every answer: embeds the answer and checks that all factual claims are traceable to retrieved chunks. Claims without grounding trigger a warning or regeneration.
- Confidence scoring: answers are tagged HIGH / MEDIUM / LOW confidence based on context relevance scores.

**Answer types the system handles:**
- **Explanatory:** "What does this module do?" → Prose explanation with structure
- **Trace:** "How does X flow through the system?" → Step-by-step with each step cited
- **Historical:** "Why was this changed?" → Synthesizes commits + PRs + issue threads
- **Relational:** "What depends on this?" → Dependency map with citations
- **Comparative:** "How is this implemented differently from X?" → Side-by-side with both citations
- **Search:** "Where is X used?" → File list with line references

### Feature 4: Multi-Turn Conversation with Context Memory

**What it does:** Maintains session state so follow-up questions build on prior answers naturally.

**How it works:**
- Conversation history is maintained in-session (not persisted across sessions by default)
- Prior exchanges are summarized and injected into context window for follow-up questions
- Entity tracking: if the user asked about `AuthMiddleware`, follow-up references to "it" or "the middleware" resolve correctly
- The retrieval query for follow-up questions includes the conversation summary to capture implicit context

**Example conversation:**
```
User: "How does authentication work in this codebase?"
AI:   [explains auth flow, cites 5 files]

User: "What happens if the token is expired?"
AI:   [answers specifically about token expiry within the auth system
       already established, adds 3 new citations]

User: "And who's responsible for refreshing it?"
AI:   [finds the refresh logic, explains the responsibility split,
       notes this was refactored in PR #312]
```

### Feature 5: Visual Codebase Explorer

**What it does:** Generates visual maps of the codebase structure on demand, not as a static sidebar but as a dynamic output of specific questions.

**Visual outputs available:**
- **Dependency graph:** "Show me how the services connect" → Interactive node graph
- **Call graph:** "Show me who calls this function" → Tree diagram with file:line labels
- **Data flow diagram:** "Trace order_id through the system" → Sequence/flow diagram
- **Module ownership map:** "Which modules did each team member primarily write?" → Color-coded file tree
- **Change heatmap:** "What areas of the codebase changed most in the last 3 months?" → Heatmap overlay on file tree

All visualizations are generated as interactive SVG/D3 diagrams rendered inline in the chat. They can be exported as PNG, SVG, or copied as Mermaid syntax.

### Feature 6: PR Review Intelligence

**What it does:** Given a PR URL or number, generates a pre-review brief that helps reviewers understand the context before they look at the diff.

**PR Brief includes:**
- Summary of what the PR changes (from the diff + description)
- The modules affected and what they currently do
- Historical context: last time these files were touched, by whom, and why
- Potential risk areas: "This PR touches the payment flow, which has had 3 bugs in the last 6 months"
- Questions to ask the author based on apparent gaps in the PR description
- Related PRs and issues for context

**Input:** Just a PR URL. No configuration needed.
**Output:** A structured markdown brief, rendered in the UI, shareable via link.

### Feature 7: Onboarding Mode

**What it does:** A guided walkthrough mode specifically designed for engineers new to a codebase.

**How it works:**
1. User selects "Onboarding Mode" and specifies their role (frontend / backend / fullstack / data / devops)
2. RepoChat analyzes the repository and generates a structured onboarding curriculum:
   - "Day 1: System overview" — high-level architecture explanation
   - "Day 2: Core data models" — what the main entities are and how they relate
   - "Day 3: Entry points" — where does a request enter the system?
   - "Day 4: Key services" — deep dive per service/module
   - "Day 5: The tricky parts" — where the technical debt and unusual decisions live
3. Each section is interactive — embedded questions the new engineer can ask to go deeper
4. Generates a shareable "Codebase Guide" document at the end

**Why this is valuable:** The best onboarding experience currently is "sit with a senior engineer for a week." This approximates that experience for codebases of any size at any time.

### Feature 8: Smart Code Search

**What it does:** Semantic code search that understands intent, not just keywords.

**Capabilities:**
- "Find all places where we handle payment failures" → Returns all error handling patterns related to payments, even if they use different variable names and patterns
- "Show me examples of how we do background job processing" → Finds all workers, tasks, and queue consumers
- "Find any SQL that might be vulnerable to injection" → Security-pattern search
- "Where do we have N+1 query patterns?" → Anti-pattern detection search

**How it differs from grep:** The query is understood semantically. "Payment failures" doesn't require the word "payment" or "failure" to appear in the code — it retrieves code that *conceptually* handles payment failure scenarios.

---

## 6. User Experience & Flows

### Primary Flow: Connect and Ask

```
Landing Page
     │
     └── "Connect Repository" button
              │
              └── GitHub OAuth (or paste public repo URL)
                       │
                       └── Indexing screen
                                │
                                ├── Progress bar: "Indexing source files... (847 files)"
                                ├── "Indexing commit history... (2,341 commits)"
                                ├── "Indexing pull requests... (412 PRs)"
                                └── "Indexing documentation... (23 files)"
                                         │
                                         └── Chat interface — ready in ~2-5 min
```

### Chat Interface Design

```
┌────────────────────────────────────────────────────────────┐
│  RepoChat  │  my-org/my-repo  ▾  │  [New Session]  [Share] │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  SUGGESTED QUESTIONS (disappears after first message)      │
│  ┌─────────────────────┐  ┌─────────────────────────────┐ │
│  │ 🗺 Give me an       │  │ 🔍 How does authentication  │ │
│  │   overview of this  │  │    work?                    │ │
│  │   codebase          │  └─────────────────────────────┘ │
│  └─────────────────────┘  ┌─────────────────────────────┐ │
│  ┌─────────────────────┐  │ 📋 What changed most in     │ │
│  │ ⚡ Trace a request  │  │    the last month?          │ │
│  │   from API to DB    │  └─────────────────────────────┘ │
│  └─────────────────────┘                                   │
│                                                            │
│ ──────────────────────────────────────────────────────── │
│                                                            │
│  [Answer appears here with inline citations]               │
│                                                            │
│  Referenced in:                                            │
│  ▶ src/auth/middleware.py:45-89                            │
│  ▶ PR #247 — "Refactor admin auth flow" (Jan 10)           │
│  ▶ commit abc123 — "Fix admin permission" (Jan 8)          │
│                                                            │
│ ──────────────────────────────────────────────────────── │
│                                                            │
│  ┌──────────────────────────────────────────────┐  [Send] │
│  │ Ask anything about this codebase...          │         │
│  └──────────────────────────────────────────────┘         │
│  /onboard  /pr 247  /diagram  /search  [?]                │
└────────────────────────────────────────────────────────────┘
```

### Slash Commands

| Command | Action |
|---|---|
| `/onboard` | Start Onboarding Mode |
| `/pr [number]` | Generate PR Review Brief for PR #N |
| `/diagram` | Generate visual diagram of last answer |
| `/search [query]` | Switch to semantic code search mode |
| `/focus [path]` | Restrict retrieval to a specific directory |
| `/history` | Show what was changed most in this area |
| `/export` | Export current session as markdown |

### Citation Click Behavior

When a user clicks a citation like `src/auth/middleware.py:45-89`, the right panel opens (or a new tab) showing:
- The file content with lines 45-89 highlighted
- The git blame for those lines (who wrote it, when, in what commit)
- The PR that introduced this code
- Option to "Ask about this file" to start a focused conversation

---

## 7. Technical Architecture

### System Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT (Browser)                          │
│  React + Vite │ Streaming Chat UI │ D3 Visualization Engine      │
└───────────────────────────┬─────────────────────────────────────┘
                            │ HTTPS / WebSocket (SSE)
┌───────────────────────────▼─────────────────────────────────────┐
│                    API GATEWAY (FastAPI)                          │
│  Auth │ Rate Limiting │ Request Routing │ Session Management      │
└─────┬──────────────┬───────────────┬───────────────┬────────────┘
      │              │               │               │
┌─────▼──┐    ┌──────▼────┐  ┌──────▼──────┐  ┌───▼──────────┐
│Indexing│    │  Search   │  │   Answer    │  │  Session &   │
│Service │    │  Service  │  │  Generator  │  │  History Svc │
└─────┬──┘    └──────┬────┘  └──────┬──────┘  └───┬──────────┘
      │              │               │              │
┌─────▼──────────────▼───────────────▼──────────────▼──────────┐
│                    MESSAGE QUEUE (RabbitMQ)                    │
│  index.repo │ index.incremental │ search.query │ answer.gen   │
└─────┬──────────────┬───────────────┬───────────────┬──────────┘
      │              │               │               │
┌─────▼──┐    ┌──────▼────┐  ┌──────▼──────┐  ┌───▼──────────┐
│GitHub  │    │  Qdrant   │  │  OpenAI /   │  │   Redis      │
│API     │    │  Vector   │  │  Anthropic  │  │   Cache      │
│Client  │    │  DB       │  │  Claude API │  │   + Sessions │
└────────┘    └───────────┘  └─────────────┘  └──────────────┘
                    │
              ┌─────▼──────────────────┐
              │  PostgreSQL            │
              │  Repos │ Users │ Jobs  │
              │  Chunks │ Sessions     │
              └────────────────────────┘
```

### Service Responsibilities

**Indexing Service:**
- Receives indexing jobs from the queue
- Fetches repository content via GitHub API (with rate limit management)
- Runs AST parsing per language via tree-sitter
- Generates embeddings (batched via OpenAI embeddings API)
- Writes chunks to Qdrant and metadata to PostgreSQL
- Sets up GitHub webhooks for incremental updates

**Search Service:**
- Receives search queries
- Runs query expansion via LLM
- Executes parallel dense + sparse retrieval
- Applies RRF fusion and cross-encoder re-ranking
- Returns ranked context chunks with metadata

**Answer Generator:**
- Receives question + retrieved context
- Constructs structured prompt with conversation history
- Streams response via LLM API (Claude or GPT-4)
- Runs faithfulness validation on completed answer
- Extracts and formats citations

**Session Service:**
- Manages conversation history per session
- Handles session persistence (opt-in)
- Generates session summaries for context injection

---

## 8. AI Pipeline Design

### Embedding Strategy

**Model:** `text-embedding-3-large` (OpenAI) — 3072 dimensions, truncated to 1536 for cost/performance balance. Voyage AI's `voyage-code-2` as alternative for code-specific embeddings.

**Code-specific embedding approach:**
- Function signatures are embedded separately from function bodies
- Comments are embedded alongside the code they describe
- Import statements are embedded with a prefix that indicates they're dependency declarations
- Test files are embedded with a flag indicating test context

**Embedding batch size:** 2048 chunks per batch. Estimated index time for a 100k LOC repo: 8–12 minutes on first index.

### Retrieval Configuration

```python
retrieval_config = {
    "dense_top_k": 30,          # Qdrant semantic search
    "sparse_top_k": 30,          # BM25 keyword search
    "rerank_top_k": 8,           # Cross-encoder final selection
    "rrf_k_constant": 60,        # RRF fusion parameter
    "query_expansion_variants": 3,
    "source_type_weights": {
        "code": 1.0,
        "pr_description": 0.9,
        "commit_message": 0.8,
        "issue": 0.7,
        "doc": 0.85,
        "inline_comment": 0.75
    }
}
```

### Answer Generation Prompt Architecture

```
SYSTEM:
You are RepoChat, an expert codebase analyst. You have been given
context retrieved from a GitHub repository. Your job is to answer
the developer's question faithfully and precisely.

RULES:
1. Only use information from the provided context
2. Every factual claim must reference a specific [N] citation
3. If context is insufficient, say exactly what you don't know
4. Use technical language appropriate to the codebase's patterns
5. Never hallucinate function names, file paths, or behaviors

CONTEXT:
[Retrieved chunks with IDs, file paths, and content]

CONVERSATION HISTORY:
[Summarized prior turns]

USER QUESTION:
[Current question]

OUTPUT FORMAT:
Answer in clear technical prose. Use code blocks for code.
End with: "Sources: [list of chunk IDs used]"
```

### Faithfulness Validator

After each generated answer, a validation pass runs:

```python
def validate_faithfulness(answer: str, context_chunks: list[Chunk]) -> FaithfulnessResult:
    """
    1. Extract all factual claims from the answer
    2. For each claim, check if it's grounded in at least one context chunk
    3. Return confidence score and list of ungrounded claims
    """
    claims = extract_claims(answer)  # LLM-based claim extraction
    
    results = []
    for claim in claims:
        claim_embedding = embed(claim)
        best_match_score = max(
            cosine_similarity(claim_embedding, chunk.embedding)
            for chunk in context_chunks
        )
        results.append(ClaimResult(claim=claim, score=best_match_score))
    
    # Flag answers where any claim scores below threshold
    ungrounded = [r for r in results if r.score < 0.72]
    return FaithfulnessResult(
        confidence="HIGH" if not ungrounded else "LOW",
        ungrounded_claims=ungrounded
    )
```

---

## 9. Data Model

```sql
-- Core entities

CREATE TABLE repositories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    github_repo_id BIGINT UNIQUE NOT NULL,
    full_name TEXT NOT NULL,  -- "owner/repo"
    default_branch TEXT NOT NULL,
    last_indexed_at TIMESTAMPTZ,
    index_status TEXT DEFAULT 'pending',  -- pending|indexing|ready|failed
    total_chunks INTEGER DEFAULT 0,
    webhook_id BIGINT,
    is_private BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE chunks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repo_id UUID REFERENCES repositories(id) ON DELETE CASCADE,
    source_type TEXT NOT NULL,  -- code|commit|pr|issue|doc|comment
    file_path TEXT,
    line_start INTEGER,
    line_end INTEGER,
    language TEXT,
    commit_sha TEXT,
    author_email TEXT,
    authored_at TIMESTAMPTZ,
    content TEXT NOT NULL,
    content_hash TEXT NOT NULL,  -- for dedup
    qdrant_id TEXT UNIQUE,  -- reference to vector store
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repo_id UUID REFERENCES repositories(id),
    user_id UUID REFERENCES users(id),
    title TEXT,  -- auto-generated from first question
    summary TEXT,  -- rolling summary for context injection
    created_at TIMESTAMPTZ DEFAULT NOW(),
    last_active_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID REFERENCES sessions(id) ON DELETE CASCADE,
    role TEXT NOT NULL,  -- user|assistant
    content TEXT NOT NULL,
    citations JSONB DEFAULT '[]',  -- [{chunk_id, file_path, lines}]
    faithfulness_score FLOAT,
    retrieval_latency_ms INTEGER,
    generation_latency_ms INTEGER,
    model_used TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE index_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repo_id UUID REFERENCES repositories(id),
    job_type TEXT NOT NULL,  -- full|incremental
    status TEXT DEFAULT 'queued',
    progress JSONB DEFAULT '{}',
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_chunks_repo_source ON chunks(repo_id, source_type);
CREATE INDEX idx_chunks_file_path ON chunks(repo_id, file_path);
CREATE INDEX idx_messages_session ON messages(session_id, created_at);
CREATE INDEX idx_sessions_user ON sessions(user_id, last_active_at DESC);
```

---

## 10. API Design

### REST Endpoints

```
Authentication
  POST /auth/github          → OAuth initiation
  GET  /auth/github/callback → OAuth callback + JWT issue
  POST /auth/refresh         → Refresh access token

Repositories
  GET  /repos                → List user's connected repos
  POST /repos                → Connect new repo {github_url | repo_id}
  GET  /repos/{id}           → Repo details + index status
  DEL  /repos/{id}           → Disconnect repo
  POST /repos/{id}/reindex   → Trigger full re-index

Sessions
  GET  /repos/{id}/sessions       → List sessions for repo
  POST /repos/{id}/sessions       → Create new session
  GET  /sessions/{id}             → Session detail + messages
  DEL  /sessions/{id}             → Delete session
  GET  /sessions/{id}/export      → Export session as markdown

Messaging (SSE Streaming)
  POST /sessions/{id}/messages    → Send message, stream response

Search
  POST /repos/{id}/search         → Semantic code search
  GET  /repos/{id}/files/{path}   → Get file content with blame

PR Briefs
  POST /repos/{id}/pr-brief       → Generate PR brief {pr_number}

Visualizations
  POST /sessions/{id}/diagram     → Generate diagram from context

Index Status
  GET  /repos/{id}/index-status   → SSE stream of indexing progress
```

### Streaming Response Format

```typescript
// Each SSE event is one of:

// Token chunk
{ type: "token", content: "The authentication middleware..." }

// Citation discovered
{ type: "citation", ref: "[1]", file: "src/auth/middleware.py",
  line_start: 45, line_end: 89, github_url: "https://github.com/..." }

// Answer complete with metadata
{ type: "done", faithfulness: "HIGH", latency_ms: 1240,
  model: "claude-sonnet-4-6", chunks_used: 7 }

// Error
{ type: "error", code: "INSUFFICIENT_CONTEXT",
  message: "Not enough information in this codebase to answer confidently" }
```

---

## 11. Evaluation & Quality

### Retrieval Evaluation

**Metrics tracked:**
- `recall@5` — is the relevant chunk in the top 5 results?
- `recall@10` — top 10?
- `MRR` — Mean Reciprocal Rank
- `NDCG@10` — Normalized Discounted Cumulative Gain

**Evaluation dataset:** A golden dataset of 50 question-answer pairs per supported repository type (Python web app, Node microservice, Go service, etc.) with human-annotated relevant chunks. Run nightly against any retrieval changes.

### Answer Quality Evaluation

**Metrics tracked:**
- `Faithfulness` — % of claims grounded in retrieved context (using LLM-as-judge)
- `Answer Relevance` — does the answer address the question asked?
- `Citation Accuracy` — do citations actually contain what the answer claims?
- `User Satisfaction` — thumbs up/down feedback per answer (captured in UI)

**LLM-as-Judge Prompt (Faithfulness):**
```
Given this answer and this context, rate faithfulness 1-5.
Score 5: every claim is directly supported by context.
Score 1: answer contains hallucinated information.
Explain which claims, if any, are not grounded.
```

### Automated Eval Pipeline

```
Every PR to main triggers:
1. Retrieval eval suite → must pass recall@5 > 0.75
2. Faithfulness eval on 20 representative questions → must score > 0.85
3. Latency benchmark → P95 answer latency < 8 seconds
4. Cost estimate → average cost per answer < $0.04
```

---

## 12. Security & Privacy

### Repository Access

- **Public repos:** No authentication required. Fetched via unauthenticated GitHub API.
- **Private repos:** Require GitHub OAuth with `repo` scope. Access tokens stored encrypted (AES-256) in PostgreSQL. Rotated automatically.
- **Org repos:** Respect GitHub team permissions. RepoChat only accesses what the authenticated user can access.

### Data Handling

- All repository content is stored encrypted at rest
- Embeddings are stored in Qdrant with repo-scoped namespaces — no cross-repo leakage
- Users can delete a repository at any time → all chunks, embeddings, and sessions are purged within 24 hours
- No repository content is used for model training
- GitHub webhook payloads are validated via HMAC-SHA256 signature

### Secret Detection

Before indexing any file, a secret scanner runs (using detect-secrets or trufflehog patterns) to identify API keys, passwords, tokens, and private keys in committed code. These chunks are:
1. Flagged in the UI ("this file contains potential secrets")
2. Redacted before being embedded or stored
3. Never included in LLM context

### Multi-Tenancy Isolation

- Every database query is scoped by `user_id` at the service layer
- Qdrant collections are namespaced by `repo_id` with tenant isolation
- Row-level security enforced in PostgreSQL

---

## 13. Observability & Monitoring

### Metrics (Prometheus + Grafana)

```
# Business metrics
repochat_repos_indexed_total
repochat_sessions_created_total
repochat_messages_sent_total
repochat_answers_rated_positive_total
repochat_answers_rated_negative_total

# Performance metrics
repochat_retrieval_latency_ms{quantile="0.5|0.95|0.99"}
repochat_answer_latency_ms{quantile="0.5|0.95|0.99"}
repochat_indexing_duration_seconds{repo_size="small|medium|large"}

# Quality metrics
repochat_faithfulness_score_avg
repochat_retrieval_recall_at5

# Cost metrics
repochat_embedding_tokens_total
repochat_llm_tokens_total{model="claude|gpt4"}
repochat_cost_per_answer_usd
```

### Langfuse Integration

Every LLM call is traced in Langfuse:
- Input prompt (with retrieved context)
- Output response
- Latency per component
- Token usage
- Faithfulness score
- User feedback (thumbs up/down)

This enables:
- Identifying specific questions that produce low-quality answers
- Detecting retrieval failures (good question, wrong chunks retrieved)
- Tracking quality over time as the model or retrieval changes

### Alerting

| Alert | Threshold | Action |
|---|---|---|
| Answer P95 latency | > 15s | Page on-call |
| Faithfulness score | < 0.75 (7-day avg) | Slack alert |
| Indexing failure rate | > 5% | Slack alert |
| LLM API errors | > 1% | Page on-call |
| Daily cost | > $50 | Slack alert |

---

## 14. MVP Scope & Phased Roadmap

### MVP (Week 1–3): The Core Loop

**Goal:** One thing done perfectly. Connect a public GitHub repo, ask a question, get a grounded answer with citations.

**Included:**
- GitHub OAuth + public repo connection
- Full indexing: code files + README + basic commit messages
- Hybrid search (semantic + BM25)
- Streaming answer generation with citations
- 5 suggested starter questions per repo
- Simple session management (in-memory, not persisted)
- React frontend with streaming chat UI

**Excluded from MVP:**
- Private repos
- PR/issue indexing
- Visual diagrams
- Onboarding mode
- Incremental indexing (full re-index only)

**MVP success criteria:**
- Indexes a 10k LOC Python repo in < 3 minutes
- Answers a factual question about the codebase correctly 80%+ of the time
- P95 answer latency < 10 seconds
- Zero hallucinated file paths or function names

### Phase 2 (Week 4–6): Depth

- Add PR + issue + commit message indexing
- Incremental indexing via webhooks
- Session persistence + history
- Private repo support
- `/pr [number]` PR brief generation
- Slash command system
- Improve retrieval with cross-encoder re-ranking

### Phase 3 (Week 7–10): Intelligence

- Onboarding Mode
- Visual diagrams (dependency, call graph, data flow)
- Smart code search with anti-pattern detection
- Multi-repo sessions (ask across multiple connected repos)
- Evaluation dashboard (faithfulness scores, user feedback trends)

### Phase 4 (Week 11–14): Scale & Monetization

- Team workspaces (shared repos, shared sessions)
- VS Code extension (RepoChat sidebar in editor)
- Slack integration (ask @repochat in your Slack)
- Enterprise private deployment option
- Usage analytics for team leads

---

## 15. Monetization

### Pricing Model

**Free Tier:**
- 3 public repositories
- 50 questions/month
- Sessions not persisted
- Community support

**Pro — $19/month:**
- Unlimited public repositories
- 5 private repositories
- 1,000 questions/month
- Session history (90 days)
- PR briefs
- Visual diagrams
- Priority support

**Team — $49/month per seat:**
- Everything in Pro
- Unlimited private repositories
- Shared team sessions
- Onboarding mode
- Slack integration
- Admin dashboard + usage analytics

**Enterprise — Custom:**
- Self-hosted deployment
- SSO/SAML
- Audit logs
- SLA guarantees
- Custom integrations

### Unit Economics (Estimate)

| Component | Cost per Answer |
|---|---|
| Embeddings (query) | ~$0.0001 |
| Qdrant queries | ~$0.0002 |
| LLM generation (Claude Sonnet) | ~$0.015 |
| Infrastructure overhead | ~$0.005 |
| **Total per answer** | **~$0.02** |

At $19/month Pro with 1,000 questions: $20 cost vs $19 revenue → need to optimize or cap.
**Lever:** Semantic caching. If 30% of questions hit cache (common for identical questions about the same repo), cost drops to ~$0.014/answer → positive unit economics.

---

## 16. Launch Strategy

### Pre-Launch

1. Build RepoChat on a popular open source Python repo (FastAPI itself is a great choice — it's well-known, well-structured, and developers know what the correct answers are)
2. Record a 90-second demo: connect the repo, ask "How does dependency injection work in FastAPI?", watch it trace through the source and produce a cited answer
3. Write a technical post: "I built NotebookLM for GitHub repos — here's the architecture"

### Launch Channels

**Hacker News:** "Show HN: RepoChat — ask your codebase questions, get cited answers" — target a Tuesday morning post

**X/Twitter:** Threaded technical post + screen recording. Target: engineering audience. Parthu's existing content series makes this natural.

**Product Hunt:** Full launch page with 90-second demo video

**Reddit:** r/programming, r/ExperiencedDevs, r/MachineLearning — post the architecture article, not the product

**Dev.to / Hashnode:** Cross-post the technical article

### The Hook That Works

The single most compelling demo: connect a repo, ask "Why is this implemented this way?" about something non-obvious, and show it surface the specific PR from 18 months ago that explains the decision. This is the wow moment. Every developer has experienced this exact frustration.

---

## 17. Success Metrics

### North Star Metric
**Questions answered per week** — captures both acquisition (new repos connected) and engagement (existing users returning)

### Supporting Metrics

| Metric | Target (Month 3) |
|---|---|
| Repos indexed | 500+ |
| Weekly active users | 200+ |
| Questions answered/week | 2,000+ |
| Answer positive rating | > 75% |
| D7 retention | > 40% |
| P95 answer latency | < 8s |
| Faithfulness score | > 0.85 |
| Avg cost per answer | < $0.025 |

---

## 18. Tech Stack

### Frontend
```
Framework:     Vite + React 18
Language:      TypeScript
Styling:       Tailwind CSS
Chat UI:       Custom SSE streaming handler
Visualization: D3.js for diagrams, Mermaid for code diagrams
Code Display:  Shiki (syntax highlighting)
State:         Zustand
HTTP:          Axios + EventSource for SSE
```

### Backend
```
Framework:     FastAPI (Python 3.11+)
Language:      Python
Task Queue:    RabbitMQ + Celery
Cache:         Redis (sessions, semantic cache, rate limiting)
Auth:          GitHub OAuth 2.0 + JWT (PyJWT)
```

### AI / ML
```
Embeddings:    OpenAI text-embedding-3-large (primary)
               Voyage voyage-code-2 (code-specific alternative)
LLM:           Anthropic Claude Sonnet 4.6 (primary)
               OpenAI GPT-4o (fallback)
Reranker:      Cohere Rerank 3 (cross-encoder)
Tracing:       Langfuse
```

### Data
```
Primary DB:    PostgreSQL 16 (Supabase or self-hosted)
Vector DB:     Qdrant (self-hosted Docker or Qdrant Cloud)
Code Parsing:  tree-sitter (AST-aware chunking)
BM25:          rank_bm25 (Python) or Typesense
Secrets scan:  detect-secrets
```

### Infrastructure
```
Hosting:       AWS ECS (Fargate) or Railway
Container:     Docker (multi-stage builds)
CI/CD:         GitHub Actions
Monitoring:    Prometheus + Grafana
Logging:       Langfuse + structured JSON logs
CDN:           CloudFront
```

---

## Appendix: Open Questions & Decisions

| Question | Options | Recommendation |
|---|---|---|
| Primary LLM | Claude vs GPT-4o | Claude Sonnet — better long-context, better at following citation instructions |
| Vector DB | Qdrant vs Pinecone vs pgvector | Qdrant — self-hostable, fast, good filtering support |
| Code embedding | OpenAI vs Voyage | Start OpenAI, A/B test Voyage code model in Phase 2 |
| Chunking | Token-based vs AST-based | AST-based — worth the complexity for code quality |
| Multi-tenancy | Shared collection vs per-repo | Per-repo Qdrant collections — cleaner isolation, easier deletion |
| Private repo storage | Encrypt content or embeddings only | Encrypt both — user trust is paramount |

---

*Document version: 1.0 | Built for production-level portfolio showcase and real product launch*
