# Project 14: Multi-Agent Content Pipeline

- **Tier:** 4 — Portfolio-Level Complex Systems
- **Project #:** 14 of 16
- **Tech Stack:** LangGraph (supervisor + workers), MCP tools, FastAPI, Langfuse tracing, human-in-the-loop workflows
- **Concepts:** Multi-agent orchestration, role-based agents, human-in-the-loop (interrupt/resume), failure handling, parallel worker execution
- **Quality Gate:** ✅ APPROVED when system produces 10 publishable articles with verified facts, passing all editorial quality checks

---

## Phase 1: Brief (Priya)

> *Priya: "A media company wants an AI content pipeline. This is the kind of project that either works beautifully or produces chaos."*

**Client:** **Veritas Media** — a digital publisher producing 100+ articles/month across tech, business, and science beats.

**Their problem:** They have 5 senior writers but need to scale to 200+ articles/month. Each article goes through: topic research → draft → fact-check → SEO optimization → publish. The bottleneck is NOT writing — it's the 3-4 editing passes after writing.

**What we're building:** A multi-agent content production pipeline:
1. **Trend Researcher** — finds trending topics, surfaces data, identifies angles
2. **Writer** — produces the first draft in the publication's style guide
3. **Fact-Checker** — verifies every claim, flags unverifiable statements
4. **SEO Optimizer** — optimizes title, meta, headings, internal links, keywords
5. **Editor** (human-in-the-loop) — reviews and approves final article

**Non-negotiable:**
- Every article must pass fact-checking before publication
- Human must approve final version before publishing (human-in-the-loop)
- If a specialist agent fails, the system must recover (not lose the article)
- Must produce articles indistinguishable from the publication's human-written content
- Must handle topic variations (tech news, business analysis, science explainers)

**Deadline:** 14 days. They want to pilot with 10 articles before committing.

**Definition of done:** Editor enters a topic → 5-agent pipeline produces a draft → editor reviews/approves → publishes. 10 articles produced, all passing editorial quality checks.

---

## Phase 2: Learning Path (Maya)

> *Maya: "This project adds two things you haven't done before: (1) human-in-the-loop with pause/resume, and (2) production multi-agent with failure recovery."*

### What's New

1. **Human-in-the-loop.** The editor needs to review and approve. The agent pauses and waits. LangGraph supports this with `interrupt()` — the graph suspends execution, the human reviews, the graph resumes.

2. **Failure recovery.** What happens when the fact-checker flags a claim that can't be verified? The pipeline should: (a) pause, (b) ask the writer to revise, (c) re-fact-check, (d) continue. Not crash.

3. **Parallel execution.** The trend researcher can work independently of the writer's first pass. Use LangGraph's `Send` API for parallel fan-out.

### First Concrete Step

> "Read about LangGraph's `interrupt()` mechanism. Write a 20-line graph that: starts → pauses (waiting for human input) → continues → finishes. Get the human-in-the-loop pattern working in isolation before you add it to the full pipeline."

---

## Phase 3: The Build

> *5 agents. 1 pipeline. Human approval at the end. Failure recovery throughout.*

### Milestone 1: Single-Agent Pipeline

Build each agent independently and test:

```python
from langgraph.prebuilt import create_react_agent

trend_researcher = create_react_agent(
    model=ChatOpenAI(model="gpt-4o-mini"),
    tools=[search_web, fetch_trending_topics, analyze_competitor_content],
    prompt="""You are a trend researcher. Given a topic:
1. Search for current trends, data, and recent developments
2. Analyze what competitors are covering on this topic
3. Identify underserved angles the publication could take
4. Return: topic brief with data points, angle recommendations, source list"""
)

writer = create_react_agent(
    model=ChatOpenAI(model="gpt-4o"),
    prompt="""You are a professional writer for Veritas Media. Write in the publication's voice:
- Clear, authoritative, accessible
- Short paragraphs (2-3 sentences max)
- Data-backed claims with inline citations
- Engaging title and subheadings
- 800-1500 words

Style guide: Use active voice. Avoid jargon without explanation. 
Open with a hook. End with a takeaway."""
)

fact_checker = create_react_agent(
    model=ChatOpenAI(model="gpt-4o-mini"),
    tools=[search_web],  # Can independently verify claims
    prompt="""You are a fact-checker. Review the article:
1. Extract every factual claim (dates, stats, quotes, names)
2. Verify each claim against reliable sources
3. Flag: ✅ VERIFIED / ⚠️ UNVERIFIED / ❌ CONTRADICTED
4. For unverified claims, suggest removal or revision
5. Return: verification report"""
)

seo_optimizer = create_react_agent(
    model=ChatOpenAI(model="gpt-4o-mini"),
    prompt="""You are an SEO specialist. Optimize the article:
1. Title: optimize for search (include primary keyword, <60 chars)
2. Meta description: compelling, <160 chars, includes keyword
3. Headings: ensure H1-H2 structure, include secondary keywords
4. Internal links: suggest 3-5 links to other articles
5. Keywords: primary + secondary keyword placement
6. Return: optimized article + SEO report"""
)
```

### Milestone 2: Supervisor + Pipeline Graph

```python
from langgraph.graph import StateGraph, START, END, Command
from langgraph.checkpoint import MemorySaver
from typing import TypedDict, List, Optional

class ArticleState(TypedDict):
    topic: str
    research_brief: Optional[str]
    draft: Optional[str]
    fact_check_report: Optional[str]
    seo_report: Optional[str]
    final_article: Optional[str]
    editor_feedback: Optional[str]
    status: str  # drafting, fact_checking, seo, editor_review, approved, rejected
    iteration_count: int

MAX_ITERATIONS = 3

def supervisor_router(state: ArticleState) -> Command:
    """Route based on current status and iteration count."""
    if state["status"] == "drafting":
        return Command(goto="writer")
    elif state["status"] == "fact_checking":
        return Command(goto="fact_checker")
    elif state["status"] == "seo":
        return Command(goto="seo_optimizer")
    elif state["status"] == "editor_review":
        return Command(goto="editor_review")
    elif state["status"] == "rejected":
        if state["iteration_count"] < MAX_ITERATIONS:
            return Command(goto="writer", update={"status": "drafting", "iteration_count": state["iteration_count"] + 1})
        else:
            return Command(goto="END")
    elif state["status"] == "approved":
        return Command(goto="END")

# Build the graph
builder = StateGraph(ArticleState)

builder.add_node("trend_researcher", research_topic)
builder.add_node("writer", write_article)
builder.add_node("fact_checker", fact_check_article)
builder.add_node("seo_optimizer", optimize_seo)
builder.add_node("editor_review", human_review)  # This is where interrupt() happens

builder.add_edge(START, "trend_researcher")
builder.add_edge("trend_researcher", "writer")

# After writing → fact check → SEO → editor
builder.add_edge("writer", "fact_checker")
builder.add_edge("fact_checker", "seo_optimizer")
builder.add_edge("seo_optimizer", "editor_review")

# Editor can approve, reject (→ rewrite), or reject permanently
builder.add_conditional_edges(
    "editor_review",
    lambda state: "rejected" if state["status"] == "rejected" else "approved" if state["status"] == "approved" else "writer",
    {"writer": "writer", "approved": END, "rejected": "writer"}
)

# Enable persistence for human-in-the-loop
checkpointer = MemorySaver()
pipeline = builder.compile(checkpointer=checkpointer)
```

### Milestone 3: Human-in-the-Loop with Interrupt

```python
from langgraph.types import interrupt

def human_review(state: ArticleState) -> dict:
    """Pause and wait for human editor review."""
    article = state["draft"]
    
    # Present article to editor and WAIT
    editor_decision = interrupt({
        "type": "editor_review",
        "article": article,
        "topic": state["topic"],
        "fact_check_report": state.get("fact_check_report", ""),
        "seo_report": state.get("seo_report", ""),
        "question": "Please review this article. Approve, request changes, or reject?"
    })
    
    # Editor's response
    if editor_decision["action"] == "approve":
        return {"status": "approved", "editor_feedback": editor_decision.get("notes", "")}
    elif editor_decision["action"] == "revise":
        return {"status": "rejected", "editor_feedback": editor_decision["revision_notes"]}
    else:
        return {"status": "rejected", "editor_feedback": "Permanently rejected"}

# Running the pipeline with human-in-the-loop
config = {"configurable": {"thread_id": "article-42"}}

# Initial run (pauses at editor_review)
initial_state = {"topic": "The future of edge computing in healthcare", "status": "drafting", "iteration_count": 0}
for event in pipeline.stream(initial_state, config):
    if "__interrupt__" in event:
        # Editor sees this and responds
        human_review_data = event["__interrupt__"][0].value
        print(f"Editor needs to review article on: {human_review_data['topic']}")
        # ... editor reviews and responds ...
        
        # Resume
        pipeline.invoke(Command(resume={
            "action": "revise",
            "revision_notes": "Add more specific use cases. The intro is too generic."
        }), config)
```

**Expected stuck point:** The interrupt/resume cycle doesn't preserve the full state. When the editor sends feedback, the writer needs to see the ORIGINAL article, the editor's feedback, and the fact-check report — not just the feedback.

**Maya's Socratic question:**
> *"Your editor sent feedback but the writer only sees 'make it more specific' without the original draft. The writer rewrites from scratch, losing all the good parts. How do you ensure the writer sees the COMPLETE context on revision?"*

> They should: include ALL previous state in the revision prompt — original draft, editor feedback, fact-check report. The writer should revise, not rewrite. Use the full message history, not just the latest feedback.

### Milestone 4: Failure Recovery

```python
def safe_execute_agent(agent_func, state: dict, max_retries: int = 2) -> dict:
    """Execute an agent with retry and graceful failure."""
    for attempt in range(max_retries + 1):
        try:
            result = agent_func(state)
            return result
        except Exception as e:
            if attempt == max_retries:
                # Graceful degradation: return partial state
                return {
                    "error": f"Agent failed after {max_retries} retries: {str(e)}",
                    "partial_result": state,
                    "status": "degraded"
                }
            time.sleep(2 ** attempt)
```

### Rohan's Mid-Build Interruption

> *Rohan appears during testing.*

"Your pipeline successfully produces articles. But I ran a test: I instructed the editor to approve without reading. The pipeline happily published. There's no GUARD against the human-in-the-loop being a rubber stamp.

In production, you need:
1. Minimum review time (editor can't approve in under 30 seconds — they didn't read it)
2. Random sampling (a percentage of 'approved' articles get re-routed for second review)
3. Quality scoring (post-publication, score article quality. If approved articles score low, flag the editor)

Human-in-the-loop is not a magic solution. It only works if the human actually participates."

### Priya's Requirement Change

> *Priya: "Veritas wants to DOUBLE production. 20 articles/week instead of 10. They want parallel article production — multiple articles going through the pipeline simultaneously. You need to handle concurrent pipelines."*

This forces: parallel execution with LangGraph's `Send` API, resource management (don't run 20 LLM calls simultaneously), and queue management.

---

## Phase 4: Review (Rohan)

> *You submit: multi-agent pipeline, human-in-the-loop, failure recovery, parallel execution, 10 test articles.*

### Rohan's Review

| # | Check | Status | Notes |
|---|---|---|---|
| 1 | **Does it work?** | ✅ PASS | Topic in, published article out. Parallel pipelines work. |
| 2 | **Edge cases?** | ✅ PASS | Agent failure → graceful degradation. Fact-check failure → revision loop → max iterations → human override. |
| 3 | **Cost-aware?** | ✅ PASS | $0.80/article with GPT-4o-mini for most agents, GPT-4o only for writer. 2-3x cheaper than a human writer for first draft + fact-check. |
| 4 | **Observable?** | ✅ PASS | Full trace for every article. Per-agent cost, latency, iteration count. |
| 5 | **Right approach?** | ✅ PASS | Supervisor + specialists. Human-in-the-loop with interrupt. Recovery on failure. Parallel execution with Send. |
| 6 | **Decisions justified?** | ✅ PASS | Architecture decisions documented. Tradeoffs explained. |
| 7 | **Measurable quality?** | ✅ PASS | 10 articles: all passed editorial review (after average 1.3 revision cycles). Blind test: 8/10 readers couldn't distinguish from human-written content. |

### Verdict: ✅ APPROVED

*Rohan reads the blind test results.*

"8/10 readers couldn't distinguish from human-written. That's the quality bar. APPROVED.

**Note on what this proves:** This project demonstrates you can architect a multi-agent system with human oversight, failure recovery, and production scale. This is the kind of system that gets brought up in senior engineering interviews."

---

## Phase 5: Debrief (Maya)

> *After APPROVED.*

**Maya:** "You built a production multi-agent system. Here's what matters:

- **Human-in-the-loop is architectural, not optional.** LangGraph's `interrupt()` makes pause/resume native to the graph. The human isn't bolted on — they're a node in the pipeline.
- **Failure recovery must be explicit.** Multi-agent systems fail in more ways than single agents. Each failure mode needs a recovery path: retry, degrade, escalate, skip.
- **Parallel execution changes the economics.** `Send` API enables concurrent agents, which means you can scale without linear latency increases.
- **The supervisor pattern generalizes.** What you built here (supervisor + specialists + human approval) is the same architecture used by Wix's multi-agent data system, Fortune 500 manufacturing agents, and most production multi-agent deployments.

**What's next:**
- **Project 15** — Autonomous customer support at 1000+ tickets/day. More scale, more feedback loops, more evaluation.
- **Project 16** — Code generation with sandboxed execution. The hardest safety problem you'll face.

> *Maya: "Two more projects. You've come further than you realize."*
