# Recall — Complete Product Specification
### *Passive Knowledge Base for Everything You Read*

> **Tagline:** Stop remembering. Start recalling.

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
19. [Competitive Analysis](#19-competitive-analysis)

---

## 1. Vision & Philosophy

### The Core Insight

Knowledge workers spend **28% of their workweek** — roughly 11.2 hours — reading: articles, documentation, RFCs, code reviews, newsletters, technical blogs, research papers, Slack threads, Notion docs, GitHub issues. They do this every day. And they remember almost none of it.

The problem is not capture. The problem is **retrieval at the moment of need**.

When you're writing a design doc and think "I read something about exactly this problem last week," you have two options:
1. Spend 15 minutes digging through browser history, bookmarks, and open tabs
2. Give up and write from scratch, maybe re-discovering the same article next week

Neither is acceptable. The information is in your head — you know you saw it. But the technological substrate to bridge that gap doesn't exist. Read-it-later apps require manual saving. Bookmarks require manual organization. Browser history is URL-based and garbage for semantic retrieval.

**Recall is the bridge.** A browser extension that silently indexes everything you read — article body, technical content, documentation, research papers — into a local vector knowledge base. When you're writing, coding, or researching, it surfaces what's relevant. When you need to know "have I read about X?", you ask and get an answer with citations.

### Design Principles

**Principle 1: Zero friction.** The user never saves, clips, highlights, or tags anything. Reading is the capture mechanism. If you opened a page and read it for more than N seconds, it's indexed. No buttons, no gestures, no mental overhead.

**Principle 2: Local-first by default.** All content, embeddings, and indexes live on the user's machine. No cloud upload, no training on user data, no server-side processing. Privacy is not a feature — it's the architecture.

**Principle 3: Contextual, not interruptive.** Recall never pops up notifications, never shows a "did you know?" card, never interrupts flow. It waits in the sidebar. It responds when called. Like a librarian who sits silently until you ask a question.

**Principle 4: The recall experience must feel like magic.** "I read something about..." → type a vague description → exact passage with source URL, 2 seconds. The retrieval quality must be high enough that checking Recall becomes reflexive before hitting "Google it again."

**Principle 5: The system improves with use.** Every page read strengthens the index. Every search query (even failed ones) trains the retrieval model. Over weeks, Recall becomes a true second brain that knows what you've seen, what you know, and what you've forgotten.

---

## 2. The Problem — In Depth

### The Knowledge Worker's Reading Tax

Knowledge workers read an estimated 2,000-5,000 words per day across disparate sources. A senior engineer at a fast-moving company might read:

- 3-5 technical blog posts/engineering blogs per week
- 2-3 RFCs or design documents
- 10+ GitHub PR descriptions and diffs
- Countless Slack messages, Notion pages, and internal wikis
- Documentation for dependencies, frameworks, and tools
- Research papers, architecture decision records (ADRs), post-mortems

The current workflow is: read → maybe bookmark → forget. The cost manifests as:

| Problem | Frequency | Cost |
|---------|-----------|------|
| Re-reading something you already read | 2-3x/week | 15-30 min each |
| Failing to find a known resource | 3-5x/week | 10 min each + frustration |
| Writing from scratch when source exists | 1-2x/week | 2-4 hours of rework |
| Making decisions without full context | Ongoing | Suboptimal outcomes |

### Why Existing Solutions Fail

**Browser history** is URL-based, not content-based. You can find "that Stripe blog post about idempotency" only if you remember it was Stripe, and it was a blog, and the title had "idempotency" in it. (Spoiler: it probably didn't.)

**Bookmarks** require intentional action. Even with folders and tags, retrieval breaks down past ~100 items. Most people have 200+ bookmarked articles they've never re-visited.

**Read-it-later apps (Pocket, Instapaper, Readwise Reader)** require you to save first. This creates a separate workflow from reading. Most articles get read in the source — a tab, an email, a link clicked from Slack. The friction of saving to a read-it-later app before or after reading means most content never enters the system.

**Note-taking apps (Notion, Obsidian, Mem)** require active note-taking during or after reading. This is the highest friction option. Even disciplined note-takers only capture a fraction of what they read.

**AI memory layers (Mem0, personal-ai-memory)** focus on AI conversation memory — what you discussed with ChatGPT or Claude. They don't capture general web reading.

**Rewind AI** comes closest — it captures everything on your screen via periodic screenshots and OCR. But it's a full desktop app ($30-50/mo), doesn't work in the browser only context, and the screen capture model is overkill for the read-and-recall use case. It indexes meetings, messages, and casual browsing — noise that dilutes the signal for reading-specific recall.

### The Technical Gap

No existing product combines:
- ✅ Zero-effort passive capture (no manual save needed)
- ✅ Content-aware indexing (full-text + embeddings, not just URLs)
- ✅ Contextual sidebar recall (relevant passages surfaced while you write)
- ✅ Local-first privacy (all data on device)
- ✅ Semantic QA ("have I read about X?")

That is the gap Recall fills.

---

## 3. Product Overview

Recall is a Chrome/Firefox browser extension that:

1. **Silently indexes** every page you spend meaningful time reading (configurable: >15s, >30s, scroll-depth threshold)
2. **Stores** full page content, extracted text, metadata (title, URL, publish date, author), and vector embeddings in a **local** ChromaDB / SQLite + vector database
3. **Surfaces** relevant passages in a sidebar when you're writing, coding, or researching in any web app (Google Docs, Notion, GitHub, linear, email clients)
4. **Answers questions** about what you've read — "What did I read about RAG evaluation?" — returning passages with source links and context
5. **Delivers a weekly digest** of forgotten-but-relevant content based on what you searched for that week

### Core Loop

```
Read a page normally → Indexed silently → Continue working
                                       ↓
                                  Weeks later:
                          Writing a doc → Recall sidebar lights up
                          with 3 relevant passages you read months ago
                                       ↓
                          "Oh right, that article had exactly this!"
                          Click → Source opens → Citation inserted
```

### Positioning

Recall is to **what you read** what Rewind is to **what you see on screen** — but lighter, browser-native, and focused exclusively on the read-and-recall workflow that dominates knowledge work.

---

## 4. User Personas

### Persona 1: The Senior Engineer (Primary)

**Name:** Priya
**Role:** Staff Engineer at a Series B startup
**Reading profile:** 15-20 long-form technical articles/week, 3-5 RFCs, countless PR descriptions, Slack threaded discussions with architectural decisions
**Pain:** "I know I read a blog post about exactly this Kafka partitioning issue, but I can't find it. I spent 20 minutes on Google and gave up. I'll just figure it out myself."
**How Recall helps:** While writing a design doc on event-driven architecture, Recall's sidebar surfaces 4 passages from Engineering blogs she read months ago. She cites one directly. The doc goes from draft to review in half the time.
**Technical comfort:** High. Comfortable with local-first tools, understands embedding and RAG concepts.

### Persona 2: The Technical PM / Product Researcher

**Name:** Marcus
**Role:** Senior Product Manager, developer tools
**Reading profile:** Competitive analysis, market research reports, blog posts, product docs, customer interview notes
**Pain:** "I interviewed 15 customers and read 30 market reports. When I'm writing the PRD, I know there's data somewhere but I can't surface it."
**How Recall helps:** Types "what did customers say about pricing?" into Recall. Returns 6 passages from different sources with customer quotes. Marcus builds the pricing section in 10 minutes instead of 2 hours.
**Technical comfort:** Medium. Needs clean UX, doesn't want to configure databases.

### Persona 3: The Researcher / Lifelong Learner

**Name:** Aisha
**Role:** PhD student / Independent researcher
**Reading profile:** 5-10 academic papers/week, book excerpts, newsletter deep-dives, documentation
**Pain:** "I read 40 papers last month on transformer attention mechanisms. I can barely remember which paper said what."
**How Recall helps:** "Which papers discussed multi-query attention?" → Returns 5 papers with highlighted passages. Aisha builds her related works section without re-reading everything.
**Technical comfort:** High. May want self-hosted options.

### Persona 4: The Consultant / Fractional Executive

**Name:** David
**Role:** Independent consultant working with 3-4 clients
**Reading profile:** Client-specific docs, industry reports, competitive intel, news, each client has a separate context
**Pain:** "I read extensively for Client A on Monday and Client B on Tuesday. By Thursday, I can't remember which insight was for which client."
**How Recall helps:** Tag-based organization (auto-tagged by domain), client-specific search, "show me everything I read about [Client A] last month."

---

## 5. Core Features — Full Detail

### 5.1 Passive Content Indexing (The Engine)

**What it does:** Automatically captures and indexes web page content when the user reads it, with zero intentional action.

**Detection criteria (configurable):**
- Time threshold: Page open for >15s (default), configurable to 5s/30s/60s
- Scroll depth: >50% of page scrolled (optional secondary gate)
- Content length: >500 characters of readable text (filters landing pages, login screens, captchas)
- Excluded patterns: Configurable list (e.g., `mail.google.com`, `calendar.google.com`, known SAAS dashboards)

**Capture pipeline:**
1. Content script detects page visit meets criteria
2. Extracts: `title`, `url`, `byline`, `published_date`, `content` (readable text via Mozilla Readability or similar), `word_count`, `estimated_read_time`
3. Sends to background service worker for processing
4. Processing: text cleaning → chunking → embedding generation → upsert to local vector DB → metadata index

**What is NOT captured:**
- Pages visited for < threshold time
- Password fields, iframes, embeds
- Pages matching exclusion patterns
- Pages with no readable text (mostly JS app shells)

### 5.2 Semantic Sidebar (The Interface)

**What it does:** A collapsible sidebar that appears on any page, showing relevant passages from your reading history.

**Trigger modes:**
- **Manual:** User clicks the Recall icon in the browser toolbar
- **Active recall:** Sidebar auto-opens with contextual suggestions when Recall detects you're writing (text input focused in Google Docs, Notion, GitHub, etc.)
- **Search:** User types a question or keywords into the sidebar search bar

**Sidebar content:**
- Search bar at top with placeholder: "What have I read about..."
- Results appear below grouped by relevance score
- Each result shows: passage excerpt (2-3 sentences), source title, source URL, date read
- Clicking a result opens the source page (or scrolls to the relevant section if available)
- "Copy as citation" button per result (formats: Markdown link, plain URL, APA-style)

**Active recall mode (contextual):**
- When user is focused on a textarea/input field in supported apps, Recall analyzes the current page/document content
- Generates embedding of the current context
- Retrieves top-k semantically similar passages from the index
- Shows "Based on what you're working on..." section at top of sidebar with 2-3 suggestions
- Confidence indicator: "High relevance" / "Somewhat relevant" / "You might have forgotten"

### 5.3 Question Answering (The "Have I Read" Engine)

**What it does:** Answer natural-language questions grounded exclusively in the user's reading history.

**Capabilities:**
- `"What did I read about chunking strategies?"` → Returns passages about text chunking with sources
- `"What was that article about MongoDB indexing?"` → Handles vague queries
- `"Show me everything from Stripe's blog about idempotency"` → Filters by source domain + topic
- `"What papers discussed LoRA vs full fine-tuning?"` → Comparative answers from multiple sources

**Quality requirements:**
- Answers MUST include verbatim source passages (not paraphrased from model knowledge)
- Every claim cites the source title, URL, and approximate date read
- If no relevant content exists in the index, answer MUST be "I haven't found anything about that in your reading history" — never hallucinate
- Retrieval latency: <2s for top-k (target: 500ms)

### 5.4 Weekly Digest (The Resurfacing Engine)

**What it does:** A weekly email or in-extension summary of what you read, what you might have forgotten, and what's relevant to your current work.

**Content:**
- **Top 5 forgotten gems:** Passages from your reading history with highest semantic distance from your recent searches + highest initial relevance score
- **Last week in review:** What you read last week (stats: articles, topics, time spent)
- **Trending topics:** What topics you read most about this month
- **Unresearched questions:** Queries that returned no results — a prompt to go find those answers

**Delivery:** Weekly email digest + in-extension notification (configurable)

### 5.5 Collections & Auto-Tagging

**What it does:** Automatically organizes your reading history into topical collections, no manual effort.

**Automatic organization:**
- Content is auto-tagged by topic (e.g., "RAG", "Kubernetes", "product management", "Python")
- Collections form based on topical clusters (e.g., "All articles about RAG evaluation")
- Cross-article connections identified: "You read this article about vector search — it connects to the one about hybrid search you saved last month"

**Manual overrides:**
- User can manually tag articles
- User can create custom collections
- User can exclude articles from specific collections
- User can delete articles from the index entirely

### 5.6 Export & Interop

**What it does:** The knowledge base is never a silo. Export in standard formats.

**Export formats:**
- JSON (full metadata + content + embeddings)
- CSV (title, url, date read, tags, snippet)
- Markdown (formatted reading list with links)
- Obsidian-compatible markdown (frontmatter + body)
- Notion-compatible CSV

**Integrations:**
- Obsidian plugin (sync highlights + metadata as daily notes)
- Readwise export (surfaced passages flow into Readwise review)
- API access for power users (REST API to query the local index)

---

## 6. User Experience & Flows

### 6.1 Onboarding

**Step 1: Install & confirm**
- Install from Chrome Web Store
- One-time permissions screen explaining what Recall accesses (page content on all sites — explain WHY)
- Privacy pledge: "Everything stays on your device. We never see what you read."

**Step 2: Index your history (optional)**
- Option to batch-index the last N days of browser history
- Progress bar: indexing 500 pages from history...
- Estimated time: 2-5 minutes for 500 pages
- User can skip this and start fresh

**Step 3: First recognition moment**
- Within the first day, user opens a new tab and sees: "We've already indexed 15 articles you read today."
- The a-ha moment comes when user asks "What did I read about X?" and gets a perfect answer

**Step 4: Sidebar introduction**
- Guided tour of the sidebar: search bar, active recall mode, weekly digest toggle
- "Try asking: what have I read recently?"

### 6.2 Daily Use Flow

```
Morning:
  - User reads 5 articles during morning research
  - Recall silently indexes all 5

Afternoon:
  - User starts writing a design doc in Google Docs
  - Recall sidebar auto-opens with "Based on what you're writing..." suggestions
  - User sees a passage from an article read 3 weeks ago
  - Inserts citation with one click

Evening:
  - User recalls another article but can't remember the title
  - Opens Recall sidebar, types "that thing about rate limiting at Stripe"
  - First result: the exact Stripe engineering blog post from 2 months ago
  - "Oh right, that's the one"
```

### 6.3 Search Flow

```
User types: "vector search performance benchmarks"

1. Embed query with local embedding model (Xenova/all-MiniLM-L6-v2 or similar)
2. Search vector DB with cosine similarity (top-15)
3. Rerank by: semantic score × recency multiplier × source authority signal
4. Return top-8 results with passages, titles, URLs, dates
5. User scans results, clicks one → opens original page

If results are poor:
  - "I don't have anything on that yet. Try a different search?"
  - Log: failed query + context → used for tuning reranker
```

### 6.4 Active Recall (Contextual Suggestion) Flow

```
User on Notion, writing a page about "event-driven architecture":

1. On text input focus + 5s pause in typing → trigger context capture
2. Extract last 200 words written from the Notion page
3. Embed context → retrieve top-5 from reading index
4. Filter: relevance > 0.75 threshold, not already seen in last 24h
5. Show in sidebar: "Based on your writing" with 2-3 suggestions
6. Each suggestion: "From [Source] (read 3 weeks ago): [passage excerpt]"

User has three options:
  - Click to open source
  - Dismiss (suggestion gone for this session)
  - "Tell me more" → expand to full Q&A mode
```

---

## 7. Technical Architecture

### 7.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Browser Extension                      │
│                                                           │
│  ┌──────────────┐   ┌─────────────────────────────────┐  │
│  │ Content Script│   │    Background Service Worker    │  │
│  │ (per-tab)     │   │                                 │  │
│  │ - Readability │──▶│ - Content extraction            │  │
│  │ - Scroll hook │   │ - Chunking pipeline             │  │
│  │ - Timer logic │   │ - Embedding queue               │  │
│  └──────────────┘   │ - DB operations                  │  │
│                      │ - Search orchestration           │  │
│  ┌──────────────┐   └──────────┬──────────────────────┘  │
│  │  Sidebar UI  │              │                         │
│  │  (React)     │◀─────────────┘                         │
│  │  - Search    │                                         │
│  │  - Results   │                                         │
│  │  - Contextual│                                         │
│  └──────────────┘                                         │
│                                                           │
│  ┌──────────────────────────────────────────────────┐    │
│  │           Local Storage Layer                     │    │
│  │                                                    │    │
│  │  ┌─────────┐  ┌──────────────┐  ┌──────────────┐  │    │
│  │  │ SQLite   │  │  Vector DB   │  │  Embedding   │  │    │
│  │  │(metadata)│  │  (ChromaDB   │  │  Model       │  │    │
│  │  │ tags     │  │   or DuckDB)│  │  (Ort/ONNX)  │  │    │
│  │  │ fulltext │  │              │  │              │  │    │
│  │  └─────────┘  └──────────────┘  └──────────────┘  │    │
│  └──────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### 7.2 Component Details

#### Content Script (Per-Tab)

**Purpose:** Detect reading activity and extract page content.

**Responsibilities:**
- Inject into every page via manifest content_scripts or dynamic injection
- Monitor: page visibility (tab active), scroll events, time on page
- When read threshold met: extract page content using Mozilla Readability (or custom extractor)
- Send extracted content to background worker via runtime messaging (chunked if >1MB)
- Handle SPA navigation (pushState/popState changes)

**Key constraints:**
- Minimal performance impact: no DOM operations after extraction
- Memory: release extracted content after sending
- Timing: extraction starts after threshold, not on page load

#### Background Service Worker

**Purpose:** Orchestrate indexing, search, and data management.

**Responsibilities:**
- Receive content from content script
- Queue processing (debounce rapid page visits)
- Text cleaning: strip nav, ads, boilerplate (trafilatura or similar)
- Chunking: split into passages (512-1024 token chunks with overlap)
- Embedding: generate vectors via ONNX runtime (local, no network)
- Indexing: write embeddings to Vector DB, metadata + full text to SQLite
- Search: receive query → embed → retrieve → rerank → return
- Scheduled tasks: weekly digest generation, index cleanup, expired entry purging

**Key constraints:**
- MV3 service worker: 30s startup, 5min idle timeout (use persistent storage + alarms API)
- Offline-capable: all operations must work without internet
- Memory budget: <500MB heap for indexing pipeline

#### Sidebar UI (React)

**Purpose:** User interface for search, recall, and configuration.

**Responsibilities:**
- Renders as iframe or web component in a sidebar panel
- Search input with debounced autocomplete
- Results list with infinite scroll
- Active recall cards (contextual suggestions)
- Settings page: index config, exclusions, export, about
- Theme support: light/dark/system

**Key constraints:**
- Must not interfere with page layout (fixed sidebar, not injected into page DOM)
- Must work on all major web apps (Google Docs, Notion, GitHub, Linear)
- Bundle size: <200KB gzipped (sidebars are performance-sensitive)

### 7.3 Data Flow Diagrams

#### Indexing Flow

```
Page Load ──→ Content Script injected
                │
                ▼
           Timer starts (15s default)
                │
                ▼
           User scrolls >50%?
                │
                ▼
           Extract with Readability
                │
                ▼
           Send to Background Worker
                │
                ▼
           Text cleaning pipeline
                │
                ▼
           Chunk into passages (512 tokens, 128 overlap)
                │
                ▼
           Generate embeddings (ONNX, local)
                │
                ▼
           SQLite: metadata + full text
           Vector DB: passage_embedding[]
                │
                ▼
           Indexing complete (async, non-blocking)
```

#### Search Flow

```
User types query in sidebar
        │
        ▼
   Embed query (local ONNX model)
        │
        ▼
   Vector search: top-15 candidates
        │
        ▼
   Rerank: semantic × recency × source_trust
        │
        ▼
   SQLite: fetch metadata for top-8 results
        │
        ▼
   Return to sidebar: passage + title + URL + date
```

---

## 8. AI Pipeline Design

### 8.1 Embedding Model

**Choice:** Local ONNX embedding model (Xenova/all-MiniLM-L6-v2 or BAAI/bge-small-en-v1.5)

**Why local:**
- Privacy: no data leaves the device
- Latency: <100ms per embedding on modern hardware (WebNN/WASM)
- Offline: works without internet
- Cost: free

**Fallback:** If ONNX runtime unavailable or too slow, use `chrome.ai.onDevice` API (when available in Chrome) or browser-native ML APIs

**Embedding dimensions:** 384 (MiniLM) or 768 (BGE-small)

### 8.2 Chunking Strategy

**Default:** Fixed-size chunks with overlap

```
Passage 1: [0-512 tokens]
Passage 2: [384-896 tokens]  (128 token overlap)
Passage 3: [768-1280 tokens]
...
```

**Sliding window approach:**
- Chunk size: 512 tokens (configurable: 256-1024)
- Overlap: 128 tokens (configurable: 0-256)
- Sentence boundary detection: prefer chunk breaks at sentence boundaries (regex + NLTK-style splitting)
- Minimum chunk length: 50 tokens (discard boilerplate fragments)

### 8.3 Retrieval Pipeline

```
Query ──→ Embed ──→ Vector Search ──→ Rerank ──→ Return
```

**Stage 1: Embedding**
- Model: all-MiniLM-L6-v2 (384-dim)
- Input: raw query text (no reformulation)
- Latency target: <50ms

**Stage 2: Vector Search**
- Top-k: 15 (configurable)
- Distance metric: cosine similarity
- Index type: IVF (inverted file index) for >10K vectors, brute-force for <10K
- Latency target: <100ms

**Stage 3: Reranking**
- Cross-encoder model (optional, can be disabled to save resources):
  - Model: ms-marco-MiniLM-L6-v2 (local ONNX)
  - Only reranks top-15 candidates (not all)
  - Latency: ~200ms for 15 pairs
- Scoring formula (if no cross-encoder): `score = 0.6 * cosine_sim + 0.2 * recency_score + 0.1 * source_authority + 0.1 * read_time_score`

**Stage 4: Return**
- Top-8 results with passages, title, URL, date, tags
- Include relevance confidence label: "High" (>0.85), "Medium" (0.7-0.85), "Low" (<0.7)

### 8.4 Q&A Pipeline (for Natural Language Questions)

```
Question ──→ Retrieval ──→ Context Assembly ──→ [Optional: Local LLM] ──→ Answer
```

**Direct retrieval mode (default):**
- Retrieve top-5 most relevant passages
- Return as search results with snippets
- User reads and decides

**LLM-enhanced mode (for capability+ hardware):**
- Only if user has WebGPU + enough memory + downloads a small local model
- Retrieve top-10 passages
- Instruct tuned small model (Phi-3-mini, Llama-3.2-1B) via ONNX/WebLLM
- Prompt: "Given these passages from the user's reading history, answer their question. If no passage answers it, say you don't know."
- Return synthesized answer with numbered citations
- Latency target: <3s total (retrieval + generation)

**Fallback (no LLM):**
- Return search results only
- "No local model available. Here's what I found in your reading history..."

### 8.5 Weekly Digest Generation

**Algorithm:**
1. From the last 7 days of reading, select articles user spent >2 min on
2. For each article, extract key passages (highest TF-IDF + position-weighted)
3. Rank passages by: `interestingness = (1 - recent_search_overlap) × (read_time / avg_read_time) × smooth(recency)`
   - `recent_search_overlap`: how much the user has already searched about this topic → penalize if already found
   - `read_time`: relative engagement → reward deep reads
   - `smooth(recency)`: sigmoid over days → reward slightly older (more likely forgotten)
4. Top-5 passages → formatted digest
5. Include: "You read X articles this week" with topic breakdown

---

## 9. Data Model

### 9.1 SQLite Schema (Metadata + Full Text)

```sql
-- Core table: articles indexed
CREATE TABLE articles (
    id TEXT PRIMARY KEY,              -- hash of normalized URL (sha256)
    url TEXT NOT NULL,                 -- original URL
    normalized_url TEXT NOT NULL,       -- normalized (remove UTM, fragments)
    title TEXT,
    author TEXT,
    published_date TEXT,               -- ISO 8601, nullable
    domain TEXT NOT NULL,              -- extracted domain
    content_text TEXT NOT NULL,        -- full extracted readable text
    word_count INTEGER,
    estimated_read_time_seconds INTEGER,
    readability_score REAL,            -- 0-1, quality of extracted content
    created_at TEXT NOT NULL,          -- when first indexed
    updated_at TEXT NOT NULL,          -- when last read/updated
    last_read_at TEXT NOT NULL,        -- when user last visited
    read_count INTEGER DEFAULT 1,      -- how many times visited
    total_time_spent_seconds INTEGER,  -- cumulative reading time
    is_deleted INTEGER DEFAULT 0       -- soft delete
);

-- Chunks/passages with vectors
CREATE TABLE passages (
    id TEXT PRIMARY KEY,               -- uuid
    article_id TEXT NOT NULL,          -- FK to articles
    passage_index INTEGER NOT NULL,    -- position in article
    content TEXT NOT NULL,             -- passage text
    token_count INTEGER,               -- approximate token count
    embedding BLOB,                    -- binary embedding vector (float32 array)
    FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE
);

-- Tags (auto-generated + user-added)
CREATE TABLE tags (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE
);

CREATE TABLE article_tags (
    article_id TEXT NOT NULL,
    tag_id INTEGER NOT NULL,
    source TEXT DEFAULT 'auto',  -- 'auto' | 'user'
    PRIMARY KEY (article_id, tag_id),
    FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE,
    FOREIGN KEY (tag_id) REFERENCES tags(id) ON DELETE CASCADE
);

-- Search history (for improvement + digest)
CREATE TABLE search_queries (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    query_text TEXT NOT NULL,
    embedding BLOB,
    result_count INTEGER,
    was_satisfied INTEGER,           -- user clicked a result?
    created_at TEXT NOT NULL
);

-- User preferences
CREATE TABLE preferences (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
);

-- Default preferences
-- time_threshold_seconds: 15
-- scroll_depth_threshold: 0.5
-- max_indexed_per_day: 500
-- excluded_domains: []
-- excluded_url_patterns: []
-- weekly_digest_enabled: true
-- sidebar_shortcut: "Ctrl+Shift+R"
-- active_recall_enabled: true
-- embedding_model: "all-MiniLM-L6-v2"
-- chunk_size: 512
-- chunk_overlap: 128

-- FTS5 index for full-text search (fallback + hybrid)
CREATE VIRTUAL TABLE articles_fts USING fts5(
    title,
    content_text,
    content='articles',
    content_rowid='rowid'
);
```

### 9.2 Vector Index (ChromaDB or DuckDB)

**Collection: `recall_passages`**

```
Schema:
  id: str (passage.id from SQLite)
  embedding: float[384] (or 768)
  metadata:
    article_id: str
    passage_index: int
    domain: str
    created_at: str
    title: str
    url: str
```

**Operations:**
- `upsert(ids, embeddings, metadatas)` — batch insert/update
- `query(query_embedding, n_results=15, where={})` — similarity search
- `delete(where={"article_id": ...})` — remove article and all passages

### 9.3 Storage Budget

| Data | Estimated Size per 1K Articles | Notes |
|------|-------------------------------|-------|
| SQLite metadata + text | ~50 MB | Compressed text, indexes |
| Embeddings (384-dim float32) | ~6 MB | 1,536 bytes per passage × ~4 passages/article |
| ChromaDB index overhead | ~10 MB | IVF index structures |
| **Total per 1K articles** | **~66 MB** | |
| **Total per 10K articles (1 year heavy use)** | **~660 MB** | Still fits comfortably on modern laptops |

---

## 10. API Design

### 10.1 Internal Extension API (Chrome Runtime)

All communication between Content Script ↔ Background Worker ↔ Sidebar UI happens via Chrome runtime messaging.

```
// Content Script → Background Worker
interface IndexRequest {
  type: 'INDEX_PAGE';
  payload: {
    url: string;
    title: string;
    content: string;  // Readability output
    readability: {
      textContent: string;
      length: number;
      byline?: string;
      publishedDate?: string;
    };
    metadata: {
      wordCount: number;
      scrollPercentage: number;
      timeSpentSeconds: number;
    };
  };
}

// Sidebar → Background Worker
interface SearchRequest {
  type: 'SEARCH';
  payload: {
    query: string;
    limit?: number;      // default: 8
    minScore?: number;   // default: 0.5
    filters?: {
      domains?: string[];
      tags?: string[];
      dateFrom?: string;
      dateTo?: string;
    };
  };
}

interface SearchResponse {
  type: 'SEARCH_RESULTS';
  payload: {
    results: SearchResult[];
    query: string;
    latency: number;     // ms
  };
}

interface SearchResult {
  passageId: string;
  content: string;
  score: number;
  title: string;
  url: string;
  domain: string;
  author?: string;
  publishedDate?: string;
  dateRead: string;
  tags: string[];
  passageIndex: number;
}

// Sidebar → Background Worker (contextual)
interface ContextualSuggestRequest {
  type: 'CONTEXTUAL_SUGGEST';
  payload: {
    pageUrl: string;
    pageTitle: string;
    recentText: string;  // last 200 words typed
  };
}

// Background Worker → Sidebar
interface BadgeUpdate {
  type: 'BADGE_UPDATE';
  payload: {
    indexedCount: number;
    lastIndexedAt?: string;
  };
}
```

### 10.2 External REST API (for Power Users & Export)

```
Base URL: http://localhost:21379/api/v1  (local server for export/integration)
     (Only started on demand, not persistent)

GET  /api/v1/status
  → { indexedCount, storageUsed, version }

GET  /api/v1/search?q={query}&limit=8&minScore=0.5
  → { results: SearchResult[], query, latency }

GET  /api/v1/articles?page=1&perPage=20&sort=last_read_at
  → { articles: Article[], total, page, perPage }

GET  /api/v1/articles/{id}
  → { article: Article, passages: Passage[] }

DELETE /api/v1/articles/{id}
  → { success: true }

GET  /api/v1/export?format=json|csv|markdown
  → File download

GET  /api/v1/tags
  → { tags: Tag[] }

POST /api/v1/tags/{articleId}
  → Body: { tags: string[] }

GET  /api/v1/stats
  → {
      totalArticles,
      totalPassages,
      totalReadTime,
      topDomains: [{domain, count}],
      topTags: [{tag, count}],
      readingByDay: [{date, count}]
    }
```

---

## 11. Evaluation & Quality

### 11.1 Retrieval Quality Metrics

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Recall@5 | >0.85 | % of relevant results in top-5 for test queries |
| Precision@5 | >0.75 | % of top-5 results that are relevant |
| Mean Reciprocal Rank (MRR) | >0.80 | Average reciprocal rank of first relevant result |
| Normalized Discounted Cumulative Gain (nDCG@10) | >0.80 | Ranking quality weighted by relevance |
| Latency p50 | <500ms | Time from query to results |
| Latency p95 | <2s | Time from query to results (slow queries) |
| User satisfaction (explicit) | >4.0/5.0 | After-search rating prompt (1-5 stars) |
| User satisfaction (implicit) | >60% | % of searches where user clicked a result |

### 11.2 Indexing Quality Metrics

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Content capture rate | >90% | % of qualifying pages with complete text extraction |
| Noise pass-through rate | <5% | % of indexed pages with >50% non-content (ads, nav) |
| Duplicate detection | >99% | % of identical pages detected and deduplicated |
| Embedding freshness | <24h | Max time between page read and embedding completion |
| False positive reads | <2% | % of indexed pages user did not actually "read" |

### 11.3 Evaluation Dataset

Build a test corpus of 500 articles across 10 domains (tech blogs, documentation, news, academic papers, product docs, etc.). Create 100 queries with known relevant passages. Use this to:

1. Tune chunking strategy (size, overlap)
2. Compare embedding models (MiniLM vs BGE vs text-embedding-3-small via cloud)
3. Validate reranking improvements
4. Regression test after each model change

### 11.4 Feedback Loop

For every search, log:
- Query text
- Results returned
- Which result was clicked (if any)
- Time to click
- Subsequent actions (opened source, copied citation, dismissed)

Use this data to:
- Identify failed queries (no clicks) for index improvement
- Train recency weighting per user
- Suggest new content the user should read (if many queries on a topic with zero results)

---

## 12. Security & Privacy

### 12.1 Privacy Architecture

**Zero-knowledge by design:**
- All data processing happens on-device (embedding, indexing, search)
- No user data ever transmitted to external servers
- No analytics, no telemetry, no crash reporting with personal data
- Optional: privacy-first usage metrics (aggregate, anonymized, user-opt-in)

**Data at rest:**
- SQLite database in browser's extension storage area
- Vector DB in IndexedDB (inaccessible to other extensions or websites)
- No cloud sync (by design; future optional encrypted sync)

**Permissions justification:**
- `activeTab` + `scripting`: Inject content script for extraction
- `storage`: Store preferences and queue state
- `alarms`: Weekly digest scheduling
- `unlimitedStorage`: Large reading history
- Host permissions: `<all_urls>` (required for content extraction on any page)
  - **Transparency:** Clearly disclosed in onboarding. User can restrict to specific sites.

### 12.2 Security Considerations

| Threat | Mitigation |
|--------|-----------|
| Malicious page injects content | Content extraction uses Readability library (DOM-based, no JS execution) |
| Extension data theft | No network-facing API; data in extension sandboxed storage |
| Timing side-channel on visited pages | No external network calls during indexing |
| Extension masquerading | Published through Chrome Web Store with 2FA developer account |
| XSS in sidebar | Sidebar renders in isolated iframe, no access to page DOM |
| Compromised web store account | Mandatory extension signing, code integrity checks |

### 12.3 Data Retention

| Data Type | Retention | Rationale |
|-----------|-----------|-----------|
| Page content (full text) | 90 days default (configurable) | Space management; embeddings persist longer |
| Embeddings | Indefinite (user can delete) | Small size, high value |
| Metadata (title, URL, date) | Indefinite | Essential for recall |
| Search history | 30 days | Only used for quality improvement |
| User preferences | Until changed | No size concern |

---

## 13. Observability & Monitoring

### 13.1 Local Telemetry (Privacy-First)

All monitoring is local-only. No external services.

```
Dashboard at chrome-extension://recall/debug.html:

Indexing:
  - Total articles indexed
  - Articles indexed today
  - Average indexing time per page
  - Queue depth
  - Failed extractions (reason breakdown)

Search:
  - Total searches today
  - Average search latency
  - P95 search latency
  - Zero-result queries (count + last 10)
  - Click-through rate

Health:
  - Storage used (MB)
  - Storage free (estimated)
  - Memory usage (heap)
  - ONNX model loaded
  - ChromaDB connection status
  - Last sync / last export

Errors:
  - Extraction failures (by domain)
  - Embedding failures
  - DB connection errors
  - CRASH reports (stored locally)
```

### 13.2 User-Facing Health Indicators

- Sidebar status dot: 🟢 Green (all systems normal), 🟡 Yellow (indexing behind schedule), 🔴 Red (storage almost full, suggest export)
- Storage warning at 80% capacity
- "Recall has indexed N articles this week" (subtle counter in sidebar footer)

---

## 14. MVP Scope & Phased Roadmap

### Phase 1: MVP — "Silent Indexer" (6-8 weeks)

**Goal:** Core indexing works reliably. Search works. Privacy-first.

**Features:**
- ✅ Passive content indexing (Chrome, then Firefox)
- ✅ Readability extraction + text cleaning
- ✅ Local ONNX embedding (all-MiniLM-L6-v2)
- ✅ Local ChromaDB vector storage
- ✅ Sidebar with search
- ✅ Basic result display (title, URL, passage, score)
- ✅ Manual sidebar toggle (keyboard shortcut)
- ✅ Chrome Web Store publish

**Non-goals:**
- ❌ Active recall / contextual suggestions
- ❌ Weekly digest
- ❌ LLM-enhanced Q&A
- ❌ History import
- ❌ Export/integrations
- ❌ Collections & auto-tagging
- ❌ Safari/Firefox (Chrome only)

**Success criteria:**
- <1% crash rate per session
- P95 indexing latency <5s (from page open to embedded)
- P95 search latency <1s
- 90%+ content extraction success rate
- User can find articles they read >1 week ago

### Phase 2: Active Recall & Context (4-6 weeks)

**Goal:** Proactive suggestions when they matter most.

**Features:**
- ✅ Active recall mode — sidebar suggests passages based on current writing context
- ✅ Google Docs, Notion, GitHub, Linear support
- ✅ Improved reranking (cross-encoder optional)
- ✅ Sidebar auto-open on supported pages
- ✅ "Copy as citation" button
- ✅ Badge counter (articles indexed today)
- ✅ Search filters (date range, domain)

### Phase 3: Intelligence Layer (4-6 weeks)

**Goal:** Answer questions, not just search keyword matches.

**Features:**
- ✅ Local LLM for Q&A (Phi-3-mini or similar via WebLLM)
- ✅ Natural language question answering with citations
- ✅ Weekly digest email/in-extension
- ✅ Auto-tagging (topic extraction per article)
- ✅ Basic collections (auto-clustered topics)
- ✅ Analytics dashboard (local: what you read, when, trends)

### Phase 4: Ecosystem (6-8 weeks)

**Goal:** Recall is part of the user's broader knowledge workflow.

**Features:**
- ✅ Export to Obsidian, Readwise, Markdown, JSON
- ✅ Local REST API for integrations
- ✅ Firefox support
- ✅ Safari support
- ✅ History import (last N days of browser history)
- ✅ Excluded domains UI
- ✅ Sync via encrypted file (manual, user-managed)
- ✅ Optional cloud sync via end-to-end encryption (paid tier)
- ✅ Mobile companion (read later → auto-indexed on desktop)

### Phase 5: Power User & Scale (ongoing)

**Goal:** Handle heavy users with years of reading history.

**Features:**
- ✅ Hybrid search (vector + BM25 full-text)
- ✅ Custom embedding models (user can bring own model)
- ✅ Multi-profile support (work vs personal reading)
- ✅ Advanced search: boolean operators, proximity search
- ✅ Smart deduplication (same content, different URLs)
- ✅ Reading insights (trending topics, knowledge gaps, weekly recs)
- ✅ Team sharing (optional, future)

---

## 15. Monetization

### Freemium Model

| Tier | Price | Limits |
|------|-------|--------|
| **Free** | $0 | Index up to 500 articles, search only, sidebar suggestions (basic), no Q&A |
| **Pro** | $5/mo or $50/yr | Unlimited articles, active recall, local LLM Q&A, weekly digest, auto-tagging, all export formats |
| **Power** | $12/mo or $120/yr | Everything in Pro + multi-profile, encrypted cloud sync (cross-device), custom models, local REST API, priority features |

### Why Free Tier Exists

1. **Zero friction onboarding.** User installs, uses for weeks, becomes dependent, outgrows limits → converts.
2. **Network effects for retention.** The more you've indexed, the more valuable the product is. Churn rate decreases with reading history size.
3. **Privacy as differentiator.** Even the free tier is local-first. No data extraction, no ads, no training on user data.

### Cost Structure

| Cost | Per User/Month (approx) | Notes |
|------|------------------------|-------|
| Storage | $0 | Local-only |
| Compute (embedding) | $0 | Local ONNX |
| LLM queries (Phase 3+) | $0 | Local WebLLM |
| Chrome Web Store fee | $0 | One-time $5 dev fee |
| Infrastructure (website, docs) | <$0.01/user | Scales efficiently |
| **Total marginal cost** | **~$0** | |

This means even the free tier is sustainable. Revenue is pure margin after development costs.

### Why $5/mo Pro?
- Readwise Reader charges $9.99/mo for highlight resurfacing (requires manual action)
- Rewind charges $30-50/mo for full screen capture (overkill for reading)
- Mem charges $15/mo for auto-organizing notes (requires saving)
- Recall's passive indexing + sidebar recall is clearly differentiated and cheaper

---

## 16. Launch Strategy

### Pre-Launch (Weeks -4 to 0)

- **Landing page:** `recall.so` or `recall-app.com`
  - Tagline, waitlist signup, privacy pledge
  - Demo GIF showing: read → weeks later → sidebar recall
- **Waitlist:** Collect emails with a question: "What's the one thing you read recently that you wish you could find faster?"
- **Content marketing:**
  - "The Cost of Re-Reading: How Knowledge Workers Lose 2+ Hours Per Week"
  - "Passive Knowledge Bases: Why Readwise, Rewind, and Mem are All Missing the Same Thing"
  - "Local-First AI: The Privacy Revolution Nobody's Talking About"
- **Community building:**
  - r/selfhosted, r/LocalLLaMA, Hacker News
  - Product Hunt launch scheduled
  - Technical blog on architecture (Rust/WASM + ONNX in browser)

### Launch (Weeks 0-2)

- **Chrome Web Store launch**
- **Hacker News:** "Show HN: Recall — A passive knowledge base for everything you read in the browser"
- **Product Hunt** with demo video
- **Twitter/X launch thread:** GIF-heavy, technical audience
- **Pricing page live** (30-day free Pro trial)

### Post-Launch (Weeks 2-8)

- **Reddit AMA** in r/MachineLearning or r/programming
- **Developer guides:** "Embed a local RAG pipeline in your browser in 10 lines of config"
- **Partnerships:** Obsidian plugin, Readwise integration
- **Iterate on feedback:** Most requested features after launch → Phase 2

---

## 17. Success Metrics

### Product Metrics

| Metric | 3-Month Target | 12-Month Target |
|--------|---------------|-----------------|
| Active users (weekly) | 1,000 | 25,000 |
| Articles indexed per active user | 200/month | 500/month |
| Searches per active user per week | 5 | 15 |
| Search click-through rate | >50% | >65% |
| Weekly digest open rate | >40% | >60% |
| Free → Pro conversion | 5% | 12% |
| D7 retention | >40% | >60% |
| D30 retention | >20% | >35% |
| NPS (users with >100 indexed articles) | 40+ | 60+ |

### Quality Metrics

| Metric | Target |
|--------|--------|
| P95 search latency | <1.5s |
| Index success rate | >95% |
| Extension crash-free session rate | >99.5% |
| Average memory usage (idle) | <100MB |
| Average memory usage (indexing) | <300MB |

---

## 18. Tech Stack

### Browser Extension

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Extension framework | Plasmo (or WXT) | Modern MV3 build tooling, React integration, cross-browser |
| UI | React 18 + Tailwind CSS | Familiar, small bundle, fast renders |
| State management | Zustand | Lightweight, TypeScript-native |
| Content extraction | Mozilla Readability | Battle-tested, handles 90%+ of pages |
| Text cleaning | Trafilatura (WASM) | Better boilerplate removal than Readability alone |

### Local AI / ML

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Embedding model | ONNX Runtime Web (Xenova/all-MiniLM-L6-v2) | Local, fast, works in browser |
| Vector DB | ChromaDB (embedded mode) or DuckDB | Local, persistent, no external process |
| ONNX runtime | ONNX Runtime Web / Xenova Transformers.js | Browser-native WASM/WebGL inference |
| LLM (Phase 3+) | WebLLM / llama.cpp via WASM (MLC-LLM) | Run small models locally in browser |
| Cross-encoder | ms-marco-MiniLM-L6-v2 ONNX | Local reranking |

### Data Storage

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Metadata + FTS | SQLite via sql.js (WASM) | Reliable, well-understood, FTS5 for hybrid search |
| Vector index | ChromaDB embedded or DuckDB with vector extension | Purpose-built for vectors |
| Full text fallback | SQLite FTS5 | Simple, no external dependency |
| File export | Custom | Generate JSON/CSV/Markdown from SQLite |

### Build & Distribution

| Tool | Purpose |
|------|---------|
| Vite / Plasmo bundler | Build extension |
| WXT | Cross-browser build (Chrome, Firefox, Edge) |
| Chrome Web Store | Primary distribution |
| GitHub Actions | CI, lint, test, build |

---

## 19. Competitive Analysis

### Competitive Landscape Matrix

| Feature | **Recall** (proposed) | **Rewind AI** | **Readwise Reader** | **Mindweave** | **Mem AI** | **Sider AI** | **Personal AI Memory** | **Firefox History** |
|---------|----------------------|--------------|-------------------|-------------|----------|------------|----------------------|-------------------|
| **Capture method** | Passive (auto) | Passive (screen record) | Manual (save/clip) | 1-click save | Manual save/import | 1-click save | Auto (AI convos only) | Auto (history) |
| **Capture scope** | All web reading | Full screen + audio | Saved articles | Saved content | Saved notes/chats | Saved pages | AI chat conversations | URLs + titles |
| **Content indexing** | Full text + vectors | OCR + transcripts | Highlights only | Full text + vectors | Full text | Full text | Full text + vectors | URLs only |
| **Semantic search** | ✅ | ✅ | ❌ (keyword) | ✅ | ✅ | ✅ | ✅ | ❌ (URL-based) |
| **Contextual sidebar** | ✅ (on any page) | ❌ (desktop app) | ❌ (separate app) | ❌ (Chrome ext) | ❌ (web app) | ✅ (sidebar) | ❌ (popup) | ❌ |
| **Active recall** | ✅ (auto-suggest) | ❌ | ✅ (Spaced Rep) | ❌ | ❌ (manual search) | ❌ | ❌ | ❌ |
| **QA on history** | ✅ | ✅ ("Ask Rewind") | ❌ | ✅ (Q&A) | ✅ (Chat) | ✅ | ❌ | ❌ |
| **Local-first** | ✅ | ✅ | ❌ (cloud) | ❌ (cloud) | ❌ (cloud) | ❌ (cloud) | ✅ | ✅ |
| **LLM answer gen** | ✅ (local, Phase 3) | ✅ (cloud) | ❌ | ✅ (Gemini) | ✅ (GPT) | ✅ (multi-model) | ❌ | ❌ |
| **Cross-browser** | Chrome → FF/Safari | Desktop app only | Web app | Chrome only | Web + mobile | Chrome + mobile | Chrome | Firefox only |
| **Pricing** | Free / $5-12/mo | $30-50/mo | $5.59-9.99/mo | Free (open source) | $12-15/mo | Freemium (credit-based) | Free (open source) | Free (built-in) |
| **Privacy model** | 100% local | 100% local | Server-side | Server-side | Server-side | Server-side | 100% local | 100% local |
| **Requires action?** | ❌ No | ❌ No | ✅ Yes (save) | ✅ Yes (clip) | ✅ Yes (import) | ✅ Yes (clip) | ✅ Yes (auto-capture AI) | ❌ No |
| **Open source** | ✅ Planned | ❌ | ❌ | ✅ MIT | ❌ | ❌ | ✅ Apache 2.0 | ❌ (proprietary) |

### Competitive Position Map

```
                          PASSIVE CAPTURE
                               │
                               │
                    Recall ← ──┼── → Rewind AI
                    (reading   │     (everything on screen)
                     focused)  │
                               │
          ─────────────────────┼──────────────────────
                               │     LOCAL-FIRST
                               │
                    Personal   │
                    AI Memory  │
                    (AI convos │
                     only)     │
                               │
                          MANUAL SAVE
                               │
                               │
                    Sider AI ──┼── Readwise Reader
                    (sidebar   │     (highlight
                     AI)       │      resurfacing)
                               │
                    Mem AI ────┼── Mindweave
                    (notes)    │     (knowledge hub)
```

### Key Competitive Insights

1. **No one does passive + reading-only + local-first.** Rewind is passive but captures everything (screen noise). Personal AI Memory is passive and local-first but only captures AI conversations. Firefox history is passive but only URL-based. **Recall occupies a unique quadrant.**

2. **The action barrier is real.** Readwise/Mem/Mindweave/Sider all require the user to intentionally save or highlight. The majority of reading happens without that action. Passive capture eliminates this failure mode entirely.

3. **Contextual recall is the killer feature no one has.** Active recall — the sidebar lighting up with relevant passages while you write — is completely absent from every competitor. Readwise has spaced repetition (time-based), not context-based.

4. **Local LLM Q&A will be a lock-in.** Once a user has 5,000 indexed articles + local Q&A, the retrieval quality and answers become irreplaceable. Switching costs are extremely high.

5. **Rewind is the closest competitor but overpriced for the use case.** Rewind at $30-50/mo captures far more than reading (meetings, calls, social media) — the signal-to-noise ratio for reading-specific recall is poor. Recall at $5-12/mo wins on price + focus.

6. **Firefox is the incumbency threat.** Firefox's built-in local AI semantic search over browsing history (Beta 142+) could expand. However, it's only on Firefox, only for search suggestions (not contextual recall), and not extensible. Competing with features, not distribution.

---

### Appendix A: UX Mockups (Textual)

#### Sidebar — Empty State

```
┌──────────────────────┐
│ 🔍 Recall            │
│ ───────────────────── │
│                       │
│  What have I read     │
│  about...             │
│  ┌─────────────────┐  │
│  │ Search your     │  │
│  │ reading history │  │
│  └─────────────────┘  │
│                       │
│  No searches yet.     │
│  Try: "RAG            │
│  evaluation"          │
│                       │
│  ───────────────────── │
│  📊 47 articles indexed│
│  Today: 3 new         │
│  ⚙️ Settings          │
└──────────────────────┘
```

#### Sidebar — Search Results

```
┌──────────────────────┐
│ 🔍 Recall            │
│ ───────────────────── │
│                       │
│  RAG evaluation       │
│  ┌─────────────────┐  │
│  │ 6 results  (0.4s)│  │
│  └─────────────────┘  │
│                       │
│  ┌────────────────────┐│
│  │ ★ HIGH MATCH      ││
│  │                    ││
│  │ "RAG evaluation    ││
│  │ remains surprisingly││
│  │ under-standardized  ││
│  │ ..."               ││
│  │                    ││
│  │ 📄 Evaluating RAG  ││
│  │    Systems         ││
│  │ 🌐 blog.llamaindex.││
│  │    io              ││
│  │ 📅 Read 3 weeks ago  ││
│  │ [Open] [Cite]      ││
│  └────────────────────┘│
│                       │
│  ┌────────────────────┐│
│  │ MEDIUM MATCH       ││
│  │ "RAGAS provides a  ││
│  │ comprehensive      ││
│  │ framework for..."  ││
│  │ 📄 RAGAS docs      ││
│  │ 🌐 docs.ragas.io   ││
│  │ 📅 Read 2 months    ││
│  │    ago              ││
│  │ [Open] [Cite]      ││
│  └────────────────────┘│
└──────────────────────┘
```

#### Sidebar — Active Recall Mode

```
┌──────────────────────┐
│ 🔍 Recall            │
│ ───────────────────── │
│                       │
│ 📋 Based on your      │
│    writing...         │
│                       │
│  You're writing about │
│  "event-driven        │
│  architecture"        │
│                       │
│  ┌────────────────────┐│
│  │ 💡 From your       ││
│  │    reading history ││
│  │                    ││
│  │ "Kafka provides    ││
│  │ at-least-once      ││
│  │ delivery guarantees"││
│  │ 📄 Event-Driven    ││
│  │    with Kafka      ││
│  │ 🌐 confluent.io    ││
│  │ [Insert Citation]  ││
│  │ [Open Full] [Dismiss]│
│  └────────────────────┘│
│  ┌────────────────────┐│
│  │ "Choosing between  ││
│  │ SQS and Kafka..."  ││
│  │ 📄 AWS Blog        ││
│  │ [Insert Citation]  ││
│  └────────────────────┘│
│                       │
│  [Ask: "What did I    │
│   read about..."]     │
└──────────────────────┘
```

---

### Appendix B: Glossary

| Term | Definition |
|------|-----------|
| **Active recall** | Proactive surfacing of relevant passages based on the user's current context (what they're writing/working on) |
| **Passive capture** | Indexing content without any intentional user action — reading IS the capture mechanism |
| **Contextual suggestion** | Recall sidebar showing "Based on your writing..." cards with passages from reading history |
| **Readability** | Library for extracting readable content from web pages (strips ads, nav, sidebars) |
| **Chunking** | Splitting a document into smaller passages for embedding and retrieval |
| **Reranking** | Re-scoring initial retrieval results with a more expensive/accurate model |
| **Cross-encoder** | A model that jointly encodes query + passage for more accurate relevance scoring |
| **Vector DB** | Database that stores and indexes embedding vectors for semantic similarity search |
| **Hybrid search** | Combining vector similarity with keyword/BM25 search for better retrieval |

---

> **Next:** [Add Recall as Option 4 in 01-Project-Options-Details.md →](file:///C:/Users/MunakalaParthaSaradh/Desktop/AI%20course/AI-Course-Mastery/11-Capstone/01-Project-Options-Details.md)
