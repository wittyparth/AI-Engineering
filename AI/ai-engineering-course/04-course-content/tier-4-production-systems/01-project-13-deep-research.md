# Project 13: Deep Research Agent

- **Tier:** 4 — Portfolio-Level Complex Systems
- **Project #:** 13 of 16 (first of Tier 4)
- **Tech Stack:** LangGraph (supervisor pattern), web search API, MCP tools, long-context LLM, Langfuse tracing + evaluation
- **Concepts:** Multi-agent orchestration (supervisor + workers), autonomous planning, web research, source synthesis, citation tracking, trajectory evaluation
- **Quality Gate:** ✅ APPROVED when output is indistinguishable from a junior analyst's research report on 10 test topics, with all claims verifiable from cited sources

---

## Phase 1: Brief (Priya)

> *Priya: "This is the one clients ASK for. Every consulting firm wants this. Build it right."*

**Client:** **StratBridge** — a management consulting firm. They produce 50+ research reports/month for Fortune 500 clients.

**Their problem:** Junior analysts spend 2-3 days researching a question, reading 50+ sources, and writing a structured report. The partner wants answers in hours, not days.

**What we're building:** A deep research agent that:
1. Takes a research question (e.g., *'What are the top 3 trends in automotive supply chain resilience post-2025?'*)
2. Autonomously plans a research strategy (what to search for, which sources to prioritize)
3. Searches the web, reads documents, extracts key findings
4. Cross-references sources, identifies contradictions
5. Produces a structured report with citations — executive summary, key findings, analysis, conflicting views, methodology

**Non-negotiable:**
- Must search and synthesize 50+ sources per query
- Every claim must be verifiable from cited sources — no hallucinated findings
- Must identify contradictions between sources (not just cherry-pick agreement)
- Output must be indistinguishable from a human analyst's report
- Must handle both broad questions (*'state of AI in healthcare'*) and narrow ones (*'what was company X's revenue in Q3 2025?'*)

**Deadline:** 14 days. Pilot with a consulting partner who needs it for a live client engagement.

**Definition of done:** Give the agent a research question → it returns a 3-5 page report with executive summary, findings, citations, and methodology. A partner reads it and cannot tell it was written by an AI.

---

## Phase 2: Learning Path (Maya)

> *Maya: "This is your first multi-agent system. One agent can't do everything well. A supervisor + specialist agents can."*

### What Makes This Hard

"Three new challenges vs single-agent systems:

1. **Multi-agent coordination.** The supervisor needs to decide WHO to delegate to, WHEN the task is complete, and HOW to combine results from different specialists. If the supervisor is bad at routing, the system fails regardless of how good the specialists are.

2. **Long-form synthesis.** The output is a 3-5 page report, not a paragraph. The agent needs to organize information hierarchically — executive summary → findings → evidence → methodology. This requires planning the structure BEFORE writing.

3. **Citation integrity.** Every claim in the report must be traced back to a specific source URL + passage. If the citation chain breaks, the report loses credibility.

### Learning Order

**Step 1: Build the research planner.**
Given a question, generate a structured research plan: sub-questions to investigate, search queries to run, types of sources to prioritize (news, academic, industry reports, company filings).

**Step 2: Web search + extraction tools.**
Build tools that search the web, fetch page content, and extract relevant passages. Handle paywalled sites (note: access), PDFs, and structured data.

**Step 3: Single-agent research (baseline).**
A single agent with all tools. Measure quality, time, cost. This is your BASELINE to beat with multi-agent.

**Step 4: Multi-agent with supervisor.**
Supervisor agent + worker agents (researcher, fact-checker, writer, quality-reviewer). The supervisor plans and coordinates. Workers execute specialized tasks.

**Step 5: Trajectory evaluation.**
Build an evaluation pipeline that scores: route accuracy (did supervisor pick the right worker?), tool efficiency (optimal number of calls?), end-to-end quality (LLM-as-judge on final report).

### First Concrete Step

> "Write a 'research planner' function. Takes a broad question, returns 5 sub-questions and 15 search queries. Test it on 3 different questions. Refine the planning prompt until the plans are consistently good."

---

## Phase 3: The Build

> *Multi-agent systems fail in interesting ways. Expect the unexpected.*

### Milestone 1: Research Planner + Tools

```python
class ResearchPlan(BaseModel):
    question: str
    sub_questions: list[str]
    search_queries: list[str]
    source_types: list[str]  # ["news", "academic", "industry_report", "company_filing"]
    target_sources: int = 50

def create_research_plan(question: str) -> ResearchPlan:
    """Generate a structured research plan from a broad question."""
    response = client.beta.chat.completions.parse(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "You are a senior research analyst. Given a research question, create a structured plan."},
            {"role": "user", "content": question}
        ],
        response_format=ResearchPlan,
        temperature=0.7  # Slight creativity for diverse search strategies
    )
    return response.choices[0].message.parsed

# Web search tool
@tool
def search_web(query: str, num_results: int = 10) -> str:
    """Search the web and return results with snippets and URLs."""
    results = web_search_api(query, num_results)
    return format_search_results(results)

# Page reading tool
@tool
def read_page(url: str) -> str:
    """Fetch and extract the main content from a URL."""
    content = fetch_url_content(url)
    return extract_main_content(content)
```

### Milestone 2: Multi-Agent Supervisor Architecture

```python
from langgraph_supervisor import create_supervisor
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI

# Specialist agents
researcher = create_react_agent(
    model=ChatOpenAI(model="gpt-4o-mini"),
    tools=[search_web, read_page, extract_quotes],
    prompt="""You are a research specialist. Your job:
1. Search for information relevant to your assigned sub-question
2. Read pages and extract key facts, statistics, and quotes
3. Always save the source URL with each finding
4. Flag contradictions between sources
5. Return structured findings with full citations"""
)

fact_checker = create_react_agent(
    model=ChatOpenAI(model="gpt-4o-mini"),
    tools=[search_web, read_page],
    prompt="""You are a fact-checker. Your job:
1. Verify every claim the researcher made
2. Cross-reference claims across multiple sources
3. Flag claims that cannot be verified
4. Rate confidence (high/medium/low) for each verified claim
5. If a claim is contradicted by other sources, document the disagreement"""
)

writer = create_react_agent(
    model=ChatOpenAI(model="gpt-4o"),
    tools=[],  # Writer doesn't need tools — it synthesizes
    prompt="""You are a senior report writer. Given research findings:
1. Structure the report: Executive Summary → Key Findings → Analysis → Conflicting Views → Methodology
2. Every claim MUST have a citation [Source: URL]
3. Flag disagreements between sources — don't hide them
4. Write in professional, analytical tone — no marketing language
5. Executive summary must capture all key findings in 3-5 sentences"""
)

quality_reviewer = create_react_agent(
    model=ChatOpenAI(model="gpt-4o"),
    tools=[search_web],  # Can verify citations independently
    prompt="""You are a quality reviewer. Check the report for:
1. Citation integrity — does every claim have a source?
2. Source diversity — are findings from multiple independent sources?
3. Contradiction handling — are disagreements noted?
4. Completeness — does it answer the original question?
5. Hallucination — are there claims not supported by cited sources?

Return: APPROVED / REVISE with specific issues."""
)

# Supervisor
supervisor = create_supervisor(
    agents=[researcher, fact_checker, writer, quality_reviewer],
    model=ChatOpenAI(model="gpt-4o-mini"),
    prompt="""You are a research supervisor. Decompose the user's question and coordinate specialists.

Process:
1. First, have the RESEARCHER investigate each sub-question
2. Then, have the FACT_CHECKER verify the findings
3. Then, have the WRITER produce the report
4. Finally, have the QUALITY_REVIEWER check the report

Guidelines:
- Route to ONE specialist at a time
- After each specialist returns, decide the next step
- If the quality reviewer finds issues, route back to writer
- When quality is approved, produce the final answer
- Never do specialist work yourself"""
).compile(name="deep_research_team")
```

**Expected stuck point:** The supervisor routes to the wrong specialist. Researcher returns findings → supervisor asks writer to write → but writer hasn't seen the raw sources. The supervisor needs to pass context between specialists.

**Maya's Socratic question:**
> *"Your supervisor routed researcher→writer with no fact-checking step. The writer wrote a report with a claim that the researcher got wrong. There was no verification between research and writing. How do you enforce the correct sequence?"*

> They should: (1) define the process explicitly in the supervisor prompt (research → fact-check → write → review), (2) add state tracking so the supervisor knows which steps have been completed, (3) implement a hard-coded workflow for the first pass before allowing dynamic routing.

### Milestone 3: Report Generation + Citation Management

```python
class ResearchReport(BaseModel):
    title: str
    executive_summary: str = Field(..., description="3-5 sentences capturing all key findings")
    key_findings: list[Finding]
    analysis: str
    conflicting_views: list[Conflict]  # Important: contradictions are surfaced, not hidden
    methodology: str
    citations: list[Citation]

class Finding(BaseModel):
    claim: str
    evidence: str
    confidence: Literal["high", "medium", "low"]
    sources: list[str]  # URLs

class Citation(BaseModel):
    claim: str
    source_url: str
    source_excerpt: str  # The exact text that supports the claim
```

### Milestone 4: Trajectory Evaluation

```python
class TrajectoryEvaluation(BaseModel):
    """Three-level agent evaluation."""
    
    # Level 1: Final response quality
    end_to_end_score: float  # LLM-as-judge: does the report answer the question?
    
    # Level 2: Trajectory (path correctness)
    route_accuracy: float  # Did supervisor pick right worker at each step?
    tool_efficiency: float  # Were tools used optimally? (vs excessive calls)
    
    # Level 3: Single step quality
    research_quality: float
    fact_check_accuracy: float
    writing_quality: float

def evaluate_research_agent(test_set: list[dict], agent) -> dict:
    """Evaluate the deep research agent on a test set."""
    results = []
    
    for test in test_set:
        question = test["question"]
        expected = test.get("expected_findings", [])
        
        # Run the agent
        report = agent.invoke({"messages": [{"role": "user", "content": question}]})
        
        # Level 1: End-to-end quality
        quality = judge_report_quality(report, expected)
        
        # Level 2: Trajectory quality
        trajectory = analyze_trajectory(report, question)
        
        # Level 3: Citation verification
        citations_verified = verify_all_citations(report)
        
        results.append({
            "question": question,
            "e2e_score": quality,
            "route_accuracy": trajectory["route_accuracy"],
            "citations_valid": citations_verified,
            "num_sources": count_unique_sources(report),
        })
    
    return aggregate_results(results)
```

### Rohan's Mid-Build Interruption

> *Rohan appears after reading one of your research reports.*

"Your report on 'automotive supply chain trends' is well-written. But I found 3 claims that the cited sources don't actually support. One of them — *'80% of automotive suppliers have adopted AI for demand forecasting'* — the cited source says NO such thing. It says *'AI adoption in supply chain is growing.'*

Your LLM writer fabricated a specific statistic and attributed it to a real source. The citation verifier didn't catch it because the SOURCE EXISTS — just doesn't say what the report claims.

This is the hardest problem in AI-generated research: **citation integrity = source exists AND source says what you claim.** You're only checking the first half. How will you fix the second?"

### Priya's Requirement Change

> *Priya: "StratBridge loves the pilot. Now they want it to also HANDLE PDF attachments. Their clients send PDF research reports, whitepapers, and industry analyses. The agent should read them alongside web search results. Three extra days."*

---

## Phase 4: Review (Rohan)

> *You submit: multi-agent research system with supervisor, 10 test reports, trajectory evaluation, citation verification, PDF support.*

### Rohan's Review

| # | Check | Status | Notes |
|---|---|---|---|
| 1 | **Does it work?** | ✅ PASS | Produces well-structured research reports from broad questions. |
| 2 | **Edge cases?** | ✅ PASS | Handles narrow fact questions, broad analysis, contradictory sources. PDF support works. |
| 3 | **Cost-aware?** | 🔄 REVISE | Each report costs $2-5 (multiple LLM calls, web searches, fact-check passes). For 50 reports/month that's $100-250. Acceptable for a consulting firm. But the trajectory eval reveals excessive tool calls — your researcher makes 8-12 search calls when 5-6 would suffice. Optimize. |
| 4 | **Observable?** | ✅ PASS | Full Langfuse trace with trajectory. I can see every routing decision. |
| 5 | **Right approach?** | ✅ PASS | Supervisor + specialists is the correct architecture for this problem. |
| 6 | **Decisions justified?** | ✅ PASS | Architecture doc explains every design choice. |
| 7 | **Measurable quality?** | ✅ PASS | Trajectory evaluation: route accuracy 88%, e2e quality 8.2/10, citation validity 92%. Understood where it fails. |

### Verdict: 🔄 REVISE

*Rohan closes the citation integrity report.*

"Two things:

1. **Tool efficiency.** Your researcher makes too many redundant search calls. The trajectory shows it searches for 'supply chain AI trends,' reads 3 pages, then searches 'AI trends supply chain 2025,' reads 2 more pages that overlap with the first search. Add a deduplication step: before searching, check if similar queries have already been searched. This should cut 30-40% of tool calls.

2. **Citation integrity — second layer.** Your citation verifier checks that the SOURCE exists. It doesn't check that the source ACTUALLY SAYS what the report claims. Add a verification step: for each `[Source: URL]` claim, fetch the source page and ask an LLM: 'Does this page actually say [claim]?' If not, flag for rewrite or removal."

---

## Phase 5: Debrief (Maya)

> *After APPROVED.*

**Maya:** "You just built a multi-agent research system. This is what people picture when they say 'AI agent.' Let me tell you what you actually learned:

- **Multi-agent = routing + specialists.** The supervisor's routing decisions matter MORE than the quality of individual specialists. A great researcher wasted by a bad routing decision. Invest in the supervisor.
- **Citation integrity is a multi-layer problem.** Source exists? Source says what claim says? Claim is interpreted correctly? This is a hard problem with no perfect solution. Layer your defenses.
- **Trajectory evaluation surfaces what end-to-end scores hide.** Your e2e quality was 8.2/10. Your trajectory showed the researcher using 2x the optimal tool calls. The cost over time is significant.
- **Simple multi-agent beats complex multi-agent.** 4 specialists + supervisor is the right size. More agents would add routing complexity without proportional quality gain.

**What you'll see again:**
- **Supervisor pattern** — Project 14 (Content Pipeline) uses the exact same supervisor pattern with different specialists
- **Citation verification** — Every research agent needs this. Project 15 (Customer Support) uses it for policy citation enforcement.
- **Trajectory evaluation** — You'll build on this for the content pipeline's quality gates.

> *Maya: "Project 14 takes multi-agent to production — a content pipeline with 5 specialist agents, human-in-the-loop approval, and failure recovery. Your supervisor pattern is about to get a workout."*
