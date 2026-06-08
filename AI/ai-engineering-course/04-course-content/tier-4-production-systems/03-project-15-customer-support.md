# Project 15: Autonomous Customer Support System

- **Tier:** 4 — Portfolio-Level Complex Systems
- **Project #:** 15 of 16
- **Tech Stack:** LangGraph, RAG with hybrid search, FastAPI, Langfuse tracing, PostgreSQL for conversation state, Redis for caching
- **Concepts:** Intent classification, agentic RAG, confidence thresholds, escalation workflows, feedback loops, observability at scale, human-in-the-loop
- **Quality Gate:** ✅ APPROVED when system resolves 80% of tickets without human intervention, maintains CSAT > 4.0, costs < $0.50/resolution

---

## Phase 1: Brief (Priya)

> *Priya: "This is the one the entire support industry is waiting for. Get it right."*

**Client:** **CloudNest** — a B2B SaaS company with 5,000+ customers. They handle 1,000+ support tickets/day across billing, technical support, account management, and feature requests.

**Their problem:** 40-person support team. Tickets arrive 24/7 across email, chat, and in-app messaging. Peak hours see 80+ tickets/hour. Average first response time is 45 minutes — customers are frustrated. The support team is burned out answering the same questions about password resets, billing inquiries, and basic troubleshooting. Their documentation is excellent but customers don't read it.

**What we're building:** An autonomous support agent that:
1. Classifies every incoming ticket by intent, urgency, and sentiment
2. Searches the knowledge base with hybrid retrieval (vector + keyword)
3. Attempts resolution autonomously when confidence is high
4. Escalates to human agents when: confidence is low, sentiment is negative, or the topic is sensitive
5. Learns from resolved tickets — if a human agent resolves a ticket differently, the system should adapt
6. Tracks resolution rate, CSAT, cost per ticket, and escalation patterns

**Non-negotiable:**
- Must process 1,000+ tickets/day with < 5 second response time for auto-resolved tickets
- Must achieve 80%+ auto-resolution rate within 30 days of deployment
- Must maintain CSAT > 4.0 (on 1-5 scale) for bot-handled tickets
- Must cost < $0.50 per ticket (all in: LLM calls, vector DB, infrastructure)
- Sentiment-negative tickets MUST route to human — no exceptions
- All escalation decisions must be auditable: why did the bot escalate this?
- Must have a feedback loop: when a human changes the bot's answer, the system should learn

**Deadline:** 14 days. They're scaling from 5,000 to 8,000 customers next quarter and the support team can't keep up.

**Definition of done:** 80% of incoming tickets resolve without human touch, CSAT holds above 4.0, and you can show a dashboard with resolution rate, cost/ticket, and escalation breakdown over a 7-day simulated run.

---

## Phase 2: Learning Path (Maya)

> *Maya: "This is the most architecturally complex project so far. Multiple subsystems that all need to work in concert. Let's break it down."*

### What Makes This Hard

"Four challenges you haven't tackled at this scale before:

1. **Scale.** 1,000+ tickets/day means ~42 tickets/hour. Each ticket needs classification, retrieval, and response generation in under 5 seconds. You can't afford slow retrievals or expensive model calls on every ticket.

2. **Sentiment-aware routing.** This isn't just about accuracy — it's about emotional intelligence. A technically correct answer to an angry customer is worse than no answer. You need to detect emotional state BEFORE deciding the response strategy.

3. **Feedback loops.** When a human agent changes the bot's answer, that's a training signal. The system needs to capture it and improve. This is where most support bots die — they never get better after launch.

4. **Observability at scale.** At 1,000 tickets/day, you can't spot-check quality. You need dashboards, automated eval, and alerting. If resolution rate drops below 75%, you need to know immediately and know WHY."

### Learning Order (Scratch-First)

**Step 1: Build the intent classifier.**
Before anything else, build a classifier that takes a ticket and outputs: `{category, sentiment, urgency, requires_human}`. Start simple — a dedicated LLM call with structured output. This is your routing brain.

> *"Production reality: Most teams use a small, fast model (GPT-4o-mini / Claude Haiku) for classification and reserve the big model for response generation. Classification is high-frequency, low-stakes — optimize for speed and cost."*

**Step 2: Build the retrieval layer.**
Create the hybrid search pipeline (vector + keyword) against your knowledge base. This should be the same pattern from Project 4 but at higher reliability — every query needs results, and results need relevance scores you can threshold against.

**Step 3: Build the response generator.**
A node that takes the classified intent + retrieved chunks + conversation history and generates a response. Must include confidence scoring: "how sure am I that this answer is correct?"

> *"Confidence scoring is the single most important design decision in this system. The entire escalation logic depends on it. Don't just use LLM self-reported confidence — combine it with: retrieval score, query-intent match quality, and historical resolution rate for similar tickets."*

**Step 4: Build the router with confidence threshold.**
The conditional edge: if confidence >= threshold AND sentiment is neutral/positive → auto-resolve. If confidence < threshold OR sentiment is negative → escalate.

**Step 5: Build the escalation system.**
When escalation happens, package: original ticket, what the bot attempted, why it wasn't confident enough, suggested resolution. Route to the right human agent queue.

**Step 6: Build the feedback loop.**
When a human agent resolves an escalated ticket, compare their approach to the bot's attempted approach. If there's a delta, log it for eval. This becomes your continuous improvement signal.

**Step 7: Build the observability dashboard.**
Track: tickets handled, auto-resolve rate, CSAT, cost/ticket, average handle time, escalation reasons breakdown, model performance by intent category.

### Memory Triage

**Memorize cold:**
- Intent classifier schema — category, sentiment, urgency, requires_human fields
- Confidence threshold pattern — the conditional edge that routes auto vs escalate
- Feedback loop capture pattern — log every human override with context
- LangGraph state structure for support flows

**Look up when needed:**
- Langfuse/LangSmith dashboard configuration for support metrics
- Qdrant/Pinecone hybrid search API details
- Redis caching configuration for KB queries
- PostgresSaver checkpointer setup for conversation persistence

**Understand deeply:**
- Why sentiment-aware routing matters more than retrieval accuracy — *"A perfect answer to a furious customer is a failed interaction. The emotional state IS part of the routing criteria."*
- The economics of auto-resolution — *"Every percentage point of auto-resolution above 70% is thousands of dollars/month in reduced support headcount. But below 70%, you're just adding latency before the human."*
- Feedback loop design — *"The most common failure: the loop captures data but nobody acts on it. The eval dashboard needs to be the first thing a support manager sees every morning."*
- Confidence composition — *"Combining multiple signals (retrieval score, model probability, historical pattern) into a single confidence score is more reliable than any single signal."*

### First Concrete Step

> Write a Python script that reads 100 sample tickets from a JSON file, sends each one to a small model (GPT-4o-mini, temp 0) with a structured output schema for `{category, sentiment, urgency, requires_human}`, and prints the classification distribution. How many fall into each category? How many get flagged as `requires_human`?

### Resources (Just-in-Time)

- **LangGraph State Machine Architecture** (CallSphere blog, 2026) — state machine patterns for support agents, conditional routing, interrupts
- **LangGraph Customer Support with Escalation tutorial** (machinelearningplus) — worked example of sentiment + routing + HITL
- **Langfuse Documentation** — tracing setup, custom eval for support metrics, dashboard configuration
- **Relevance AI / RAGAS** — for building automated support quality eval
- **OWASP Top 10 for LLM Applications** — security patterns for customer-facing agents (prompt injection, data leakage)

---

## Phase 3: The Build

### Milestone 1: Ticket Classification Engine

Build the entry point — every ticket goes through this first:

```python
from pydantic import BaseModel
from typing import Literal

class TicketClassification(BaseModel):
    category: Literal[
        "billing", "technical", "account", "feature_request",
        "general_inquiry", "complaint", "other"
    ]
    sentiment: Literal["positive", "neutral", "negative", "angry"]
    urgency: Literal["low", "medium", "high", "critical"]
    requires_human: bool
    reasoning: str  # why the classifier decided this
```

Build the classifier as a FastAPI endpoint: `POST /classify` takes `{ticket_id, message, customer_history_summary}` and returns the classification.

**Expected stuck point:** The classifier is inconsistent — same ticket gets different categories on repeated runs.

**Maya's Socratic question:**
> *"Your classifier is non-deterministic. What temperature are you using? What happens if you set temperature to 0? Is there a way to make classification decisions without calling the LLM at all for obvious cases?"*

> They should discover: temperature 0 for classification, pre-check patterns (regex for password resets, billing keywords, all-caps), and caching identical tickets.

**Edge case to handle:** Multi-intent tickets. A user says "I can't log in AND I was charged twice." The classifier needs to handle compound intent — either split into sub-tickets or route to the higher-urgency category.

### Milestone 2: Hybrid Retrieval with Confidence Scoring

Build the knowledge base retrieval layer:

1. Set up Qdrant with hybrid search (dense + sparse vectors)
2. Index your support documentation as chunks with metadata (category, product area, last updated)
3. Implement a retrieval function that returns chunks + a relevance score for each

Then build the **confidence scoring**:

```python
class RetrievalResult(BaseModel):
    chunks: list[dict]
    max_relevance: float  # 0.0 to 1.0
    chunk_count: int
    category_match: bool  # does the ticket category match chunk metadata?
```

Combine retrieval confidence with classification signals to produce a composite confidence score.

**Expected stuck point:** The retrieval returns great results for billing and account queries but consistently fails for technical troubleshooting. The relevance scores are misleading — they're high but the chunks aren't actually helpful.

**Maya's Socratic question:**
> *"Your retrieval is performing unevenly across categories. How would you detect this in production? And once you detect it, what levers do you pull — different chunking for technical docs? Higher retrieval count for certain categories? Route certain categories to different retrieval strategies?"*

> They should learn: category-specific retrieval strategies, the importance of slicing eval data by category to see per-category performance, and that uniform retrieval doesn't work for diverse knowledge bases.

### Milestone 3: Response Generation + Router

Build the response generation node:

```python
class SupportResponse(BaseModel):
    answer: str
    confidence: float  # composite confidence
    citations: list[str]  # source article IDs
    suggested_action: str | None  # e.g., "reset password", "issue refund"
    requires_follow_up: bool
```

Build the conditional router:

- If `confidence >= 0.8` AND `sentiment != "angry"` → auto-resolve
- If `confidence >= 0.8` AND `sentiment == "angry"` → auto-resolve with empathetic tone + offer escalation
- If `confidence < 0.8` → escalate
- If `requires_human == True` → always escalate

**Expected stuck point:** The confidence threshold of 0.8 is arbitrary. Some tickets with 0.75 confidence resolve fine. Some with 0.85 confidence fail spectacularly.

**Maya's Socratic question:**
> *"Why 0.8? Where did that number come from? How would you find the RIGHT threshold for YOUR knowledge base and YOUR customers?"*

> They should learn: threshold tuning via eval on historical tickets, ROC curves, precision-recall tradeoffs, and that the optimal threshold varies by category.

### Rohan's Mid-Build Interruption

> *Rohan leans against your desk. "I've been watching your confidence threshold approach. One question: what happens when a ticket that SHOULD have been auto-resolved gets escalated? And what happens when a ticket that SHOULD have been escalated gets auto-resolved? I want to see your false positive AND false negative numbers. And I want to see how you plan to trend them over time."*

This forces the learner to think about the **confusion matrix** of their escalation decisions — and to build the monitoring that tracks both directions.

### Priya's Requirement Change

> *Priya: "Great news — we're piloting with a second client. Different domain. Different knowledge base. Your system needs to handle multi-tenant support — different knowledge bases for different clients. And they want a per-client dashboard. One more day."*

This forces the learner to make the knowledge base tenant-aware, implement per-client metrics, and verify that the classification + retrieval + routing works across domains.

---

## Phase 4: Review (Rohan)

> *You submit with a 7-day simulation: 7,000 tickets processed, 82% auto-resolution, CSAT 4.2, cost $0.38/ticket. Full dashboard with per-category breakdown, escalation reason analysis, and feedback loop examples.*

Rohan opens your submission. Studies the numbers for a full two minutes.

### Decision Documentation Required

Submit with your code:
1. Your confidence threshold and how you arrived at it (show the ROC curve or precision-recall tradeoff)
2. How you handle multi-intent tickets — what's your strategy for splitting vs routing to higher urgency?
3. Your feedback loop design — how does the system learn from human agent corrections?
4. Multi-tenant architecture — how does the system isolate knowledge bases and metrics per client?

### Rohan's Review

| # | Check | Status | Notes |
|---|---|---|---|
| 1 | **Does it work?** | ✅ PASS | End-to-end: ticket in → classification → retrieval → auto-resolve or escalate. Numbers on the board. |
| 2 | **Edge cases handled?** | 🔄 REVISE | Multi-intent tickets need clearer handling. What about empty messages? Non-English? What if the vector DB is down mid-day? What if classification returns `requires_human` but all human agents are busy? |
| 3 | **Cost-aware?** | ✅ PASS | $0.38/ticket target met. Classification model is the small one. Response model only fires for auto-resolve path. Good caching on KB retrieval. |
| 4 | **Observable?** | 🔄 REVISE | Dashboard is good but I need per-ticket tracing. Give me the Langfuse trace for any given ticket_id. I need to see: what was classified, what was retrieved, what confidence was computed, and why the router decided what it decided. |
| 5 | **Right approach?** | ✅ PASS | Sentiment-aware routing is the correct architecture. Classification-first, then retrieval, then response — this is how production support bots are designed. |
| 6 | **Decisions justified?** | ✅ PASS | Threshold analysis is clear. Multi-tenant design documented. |
| 7 | **Measurable quality?** | ✅ PASS | 7-day simulation with all metrics tracked. CSAT, resolution rate, cost per ticket, escalation breakdown. |

### Verdict: 🔄 REVISE

*Rohan leans back.*

"82% auto-resolution with CSAT 4.2 and cost under $0.50/ticket is genuinely good. But here's what needs attention:

1. **Multi-intent handling.** I tested with 'cancel my subscription AND can you help me reset my password' and it classified as billing only. The password reset half was ignored. You need a compound intent strategy — either split into sub-tickets or handle the higher-urgency first with an offer to address the second.

2. **Per-ticket tracing.** Your dashboard shows aggregate numbers but I need to debug individual failures. Hook Langfuse tracing at EVERY node — classification, retrieval, response generation, router decision. Each trace should be queryable by `ticket_id`. When support quality drops Tuesday at 3pm, I need to find the bad tickets in 30 seconds.

3. **KB-down fallback.** What happens if Qdrant is unreachable? Your current code crashes. Add a fallback: keyword-only search via a local index (SQLite FTS5, for example). The auto-resolve rate will drop but the system doesn't go down.

Fix these three and resubmit with explanations."

---

## Phase 5: Debrief (Maya)

> *After Rohan APPROVES your resubmission.*

**Maya:** "This project is the culmination of almost everything you've learned. Let's connect the dots:

**What you should feel confident about now:**
- Building multi-stage classification pipelines with confidence scoring
- Architecting sentiment-aware routing — not just 'is it right?' but 'is it appropriate given the customer's state?'
- Designing feedback loops that capture improvement signals from human operators
- Building at scale — 1,000+ tickets/day with cost constraints
- Observability that makes a complex system debuggable in production

**What you'll see again:**
- Intent classification + routing — comes back in Project 16 for routing code generation tasks
- Human-in-the-loop — was a key pattern in Project 14, is critical here, and will be central in Project 16
- Feedback loops — this is the #1 thing that separates production AI systems from demos. Every system you build from now on should ask: 'how does this learn from its mistakes?'
- Multi-tenant architecture — you'll use this pattern whenever you build a platform serving multiple clients

**What made this project different from all the others:**

Every project before this had a well-defined scope. This one required you to design a system where the QUALITY of decisions matters as much as the correctness. The router deciding 'keep or escalate' is a harder problem than 'answer this question' because it involves uncertainty, emotional state, cost tradeoffs, and customer trust.

> *Rohan's parting words: 'Most support chatbots fail because teams chase accuracy and ignore the confidence-aware routing. You built one that knows when NOT to answer. That's the harder skill.'*

**You're ready for the final project. Priya's waiting with the last brief — and this one's a beast."*
