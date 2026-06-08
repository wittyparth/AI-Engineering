# Project 8: Real-Time Sales Intelligence Agent

- **Tier:** 3 — Real Constraints
- **Project #:** 8 of 12 (first of Tier 3)
- **Tech Stack:** LangGraph (create_react_agent), FastAPI streaming, external APIs (simulated), Langfuse tracing
- **Concepts:** Agent loop, tool use, real-time processing, personalization, streaming, latency optimization, agent evaluation
- **Quality Gate:** ✅ APPROVED when agent completes 100 test scenarios with < 3s latency, < $0.05/run, and you have tracing for every step

---

## Phase 1: Brief (Priya)

> *Priya: "This one's fast. Client expects answers in seconds, not minutes. No pressure."*

**Client:** **SailFast** — a B2B SaaS company selling to mid-market manufacturing firms ($50M-500M revenue). Their sales team of 20 reps handles 50+ active deals.

**Their problem:** Sales reps spend 3 hours/day researching prospects before calls. They check the company website, Crunchbase, recent news, LinkedIn, mutual connections — 6 different tabs. By the time they've done all the research, they have 15 minutes of prep time for a 30-minute call.

**What we're building:** An AI sales intelligence agent. A sales rep opens a prospect's LinkedIn page, copies the company name, pastes it into our agent. The agent:
1. Researches the company (industry, size, recent news, funding)
2. Identifies the decision-maker's background and interests
3. Generates a personalized outreach email

**Non-negotiable:**
- Must complete in under **3 seconds**. The rep needs answers before the call starts
- Must cost less than **$0.05 per run**. At 50 runs/day, that's $75/month
- Every fact must be verifiable — no hallucinated company info
- Must work for at least 80% of mid-market companies (not just well-known ones)

**Deadline:** 8 days. Sales team has a Q2 push starting next week.

**Definition of done:** Rep types "Acme Manufacturing, CEO: Jane Smith" → agent returns company summary, CEO background, personalization angle, draft email. All in < 3 seconds.

---

## Phase 2: Learning Path (Maya)

> *Maya: "This is your first real agent. Not a RAG pipeline. Not a structured extraction script. An agent that thinks, acts, observes, and decides. The ReAct loop."*

### What Makes This Hard

"Three challenges that don't exist in RAG systems:

1. **Latency.** Each LLM call in an agent loop adds 500ms-2s. If your agent makes 3 reasoning steps, you've already lost the < 3s requirement. You need to optimize the LOOP, not just individual calls.

2. **Tool design.** Your agent needs tools to search the web, look up company info, and generate emails. If the tools are badly designed, the agent will misuse them — calling the wrong tool, passing wrong arguments, ignoring useful results.

3. **Evaluation.** How do you test 'did the agent do a good job?' It's not as simple as RAG metrics. You need to evaluate the full trajectory — did it call the right tools in the right order? Did it use the results correctly?

### Learning Order (Scratch-First)

**Step 1: Build a ReAct loop manually.**
Write raw Python with `while True`: LLM decides → call tool → observe → loop. No LangGraph yet. Feel the pain of state management.

> *"This is the most important step. A ReAct loop is 40 lines of Python. Write it yourself. When LangGraph breaks, you'll know why."*

**Step 2: Add tool schemas.**
Define your tools with clear descriptions and typed parameters. Bad tool descriptions → agent misuses them. Great tool descriptions → agent uses them perfectly.

**Step 3: Introduce LangGraph.**
Replace your manual loop with `create_react_agent`. Notice how LangGraph handles state persistence, message history, and conditional routing for you.

**Step 4: Optimize for latency.**
Measure every step. Where is time spent? Can you parallelize independent calls? Can you use cheaper models for some steps? Can you cache common queries?

**Step 5: Add observability.**
Integrate Langfuse tracing. Every step of the agent's reasoning, every tool call, every latency metric.

### Memory Triage

**Memorize cold:**
- The ReAct loop structure: `LLM decides → calls tool → sees result → decides again → ... → final answer`
- LangGraph's `create_react_agent` signature: `(model, tools, prompt)`
- Tool function pattern: docstring IS the tool description — make it clear
- Streaming pattern with `app.stream()` for real-time output

**Look up when needed:**
- LangGraph `StateGraph` manual API (for when you need custom control)
- Langfuse Python SDK for custom trace creation
- Specific LLM latency benchmarks for model selection

**Understand deeply:**
- Why tool design matters more than agent architecture — *"a badly designed tool makes the agent look stupid. But it's not the agent's fault — it's the tool. The best agent in the world can't use a screwdriver to drive a nail."*
- Why latency optimization is architectural, not tactical — *"you can't optimize your way out of too many LLM calls. If the agent needs 4 sequential calls to answer, no amount of caching solves that. You must redesign the LOOP."*
- When NOT to use an agent — *"Anthropic's research shows most production use cases are solved by simple prompting or prompt chaining. Agents are for when you need dynamic tool selection. Don't over-agent."*

### First Concrete Step

> "Write a 40-line Python ReAct loop. Two tools: `search_web(query)` and `get_company_info(company_name)`. Hardcode the simulated responses. Get the loop working: user asks a question → LLM decides to search → gets result → decides to look up company → gets result → synthesizes answer."

> *"Make it work with mock tools first. Then replace mocks with real APIs. This is how you build agents — start with the loop, not the tools."*

---

## Phase 3: The Build

> *Latency is the boss of this project. Every decision is measured in milliseconds.*

### Milestone 1: Manual ReAct Loop

```python
import json

def react_agent(query: str, tools: dict, model: str = "gpt-4o-mini", max_steps: int = 5):
    """Minimal ReAct loop. ~40 lines. No LangGraph."""
    messages = [
        {"role": "system", "content": "You are a sales intelligence agent. Use tools to research companies and generate insights."},
        {"role": "user", "content": query}
    ]
    
    for step in range(max_steps):
        response = client.chat.completions.create(model=model, messages=messages)
        msg = response.choices[0].message
        
        if not msg.tool_calls:
            # Final answer
            return msg.content
        
        messages.append(msg)
        
        for tool_call in msg.tool_calls:
            func = tools[tool_call.function.name]
            args = json.loads(tool_call.function.arguments)
            result = func(**args)
            
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result)
            })
    
    return "Max steps reached. Please refine your query."
```

**Expected stuck point:** The LLM calls a tool with slightly wrong arguments — `search_web(query="Acme Manufacturing")` instead of `search_web(company_name="Acme Manufacturing")`. The tool errors.

**Maya's Socratic question:**
> *"Your agent called `search_web` with the parameter name `query` but your function expects `company_name`. Whose fault is this — the LLM, the function definition, or the tool schema?"*

> They should: fix the tool description to match what the LLM expects, or use OpenAI's function calling format with proper parameter names and descriptions. This leads to the insight that tool schema = prompt engineering for agents.

### Milestone 2: LangGraph Agent

Convert to LangGraph using `create_react_agent`:

```python
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

@tool
def search_company_news(company: str) -> str:
    """Search for recent news about a company. Use this for latest updates, funding, product launches."""
    # Real implementation would use a search API
    return f"Recent news about {company}: ..."

@tool
def get_company_profile(company: str) -> str:
    """Get company details: industry, size, revenue range, headquarters. Use this for basic company research."""
    # Simulated
    return f"{company}: B2B manufacturing, 500-1000 employees, $50M-100M revenue"

@tool  
def get_executive_background(name: str) -> str:
    """Get background on an executive: previous roles, education, known interests."""
    return f"{name}: CEO since 2020, previously VP at ..."

# Build the agent
model = ChatOpenAI(model="gpt-4o-mini")
agent = create_react_agent(
    model,
    tools=[search_company_news, get_company_profile, get_executive_background],
    prompt="You are a sales intelligence agent. Research companies and executives to generate personalized outreach."
)

# Run
result = agent.invoke({
    "messages": [{"role": "user", "content": "Research Acme Manufacturing and its CEO Sarah Chen"}]
})
```

**Expected stuck point:** The LLM calls unnecessary tools. For "Acme Manufacturing," it calls `get_company_profile`, then `search_company_news`, then `get_executive_background` — even when the user only asked for company info.

**Maya's Socratic question:**
> *"Your agent makes 3 sequential LLM calls for a simple query. That's 3x latency, 3x cost. How do you make the agent more efficient?"*

> They should: (1) improve tool descriptions so the agent picks the RIGHT tool on first try, (2) use a single tool that returns more comprehensive data, (3) add a cheaper classification step before routing.

### Milestone 3: Latency Optimization

The < 3s constraint forces tough decisions:

```python
# Strategy 1: Parallel tool calls
# LangGraph supports parallel execution via multiple tool nodes
# But the LLM needs to initiate all tool calls in ONE response

# Strategy 2: Caching
# Common companies get cached results
cache = {}

def get_company_info_cached(company: str) -> dict:
    if company not in cache:
        cache[company] = fetch_company_info(company)
    return cache[company]

# Strategy 3: Cheaper model for simple steps
# Use GPT-4o-mini for research, GPT-4o only for email generation
```

**Expected stuck point:** Even with caching, a brand-new query takes 4-5 seconds. The LLM call (1.5s), then tool execution (1s), then second LLM call (1.5s), then email generation (1s) = 5s total.

**Maya's Socratic question:**
> *"Your agent still takes 5 seconds for new queries. You need 3 seconds. You can't make the model faster. You can't make the API faster. What CAN you change?"*

> They should: (1) reduce to 2 LLM calls max (combine research + email into one prompt if tools return rich data), (2) stream the response so the user sees progress, (3) pre-fetch common companies, (4) use a faster model for classification + retrieval, only use expensive model for generation.

### Milestone 4: Observability with Langfuse

```python
from langfuse import Langfuse
from langfuse.callback import LangfuseCallbackHandler

langfuse = Langfuse()
langfuse_handler = LangfuseCallbackHandler(
    session_id="sales-agent-demo",
    user_id="test-rep"
)

# Run with tracing
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Research Acme Manufacturing"}]},
    config={"callbacks": [langfuse_handler]}
)
```

**Expected stuck point:** LangGraph errors don't always show up in traces. When the agent fails mid-trajectory, you need to capture the partial state.

### Rohan's Mid-Build Interruption

> *Rohan appears during latency testing.*

"3 seconds. That's your budget. Let me see your breakdown:
- LLM call 1: XXXms
- Tool execution: XXXms
- LLM call 2: XXXms
- Email generation: XXXms

Where's your bottleneck? And what's your strategy for the 20% of cases where the response time exceeds 3 seconds — do you fail, degrade, or stream?"

### Priya's Requirement Change

> *Priya: "Sales team loves the prototype but they want it INSIDE their CRM (HubSpot). They need to click a button on a contact page and get the research in a sidebar. Can you expose it as a streaming API endpoint?"*

Add: FastAPI streaming endpoint with Server-Sent Events (SSE), showing each step as it completes: `{"step": "researching_company", "data": {...}}`, `{"step": "generating_email", "data": {...}}`.

---

## Phase 4: Review (Rohan)

> *You submit: LangGraph agent, streaming API, latency report, 100-scenario evaluation, Langfuse dashboard.*

### Decision Documentation Required

1. Your latency optimization strategy — what worked, what didn't, final breakdown
2. Your tool design — why you chose these tools and how you verified they work reliably
3. Your cost strategy — how you stayed under $0.05/run
4. How you evaluate agent quality — what metrics, what test scenarios, failure analysis

### Rohan's Review

| # | Check | Status | Notes |
|---|---|---|---|
| 1 | **Does it work?** | ✅ PASS | End-to-end: company name → research → email. Streaming works. |
| 2 | **Edge cases?** | 🔄 REVISE | What happens when the company doesn't exist? When the executive is unknown? When the search API times out? Your error handling is too basic — the agent just re-prompts instead of failing gracefully. |
| 3 | **Cost-aware?** | ✅ PASS | $0.038/run on average with GPT-4o-mini + caching. Under $0.05. |
| 4 | **Observable?** | ✅ PASS | Langfuse trace for every run. I can see each tool call, timing, and decision. |
| 5 | **Right approach?** | ✅ PASS | Simple ReAct agent. No over-engineering. LangGraph was the right choice. |
| 6 | **Decisions justified?** | ✅ PASS | You measured latency per component. You optimized where it mattered. |
| 7 | **Measurable quality?** | 🔄 REVISE | 100 scenarios passed. But I need to know: how many of the generated emails would a human actually SEND? Run a blind test: have a sales rep rate 20 emails as 'sendable as-is' / 'needs edits' / 'regenerate.' |

### Verdict: 🔄 REVISE

*Rohan looks at your latency trace.*

"Two things:

1. **Edge case handling.** When a tool fails (company not found, API timeout), your agent hangs indefinitely retrying. Add: after 2 tool failures, the agent should respond with 'I couldn't find information about this company. Here's what I know: [partial info]' — and provide a fallback template email with placeholders.

2. **Human evaluation.** Your automated eval says 92% pass rate. I don't trust automated eval for email quality. Run a blind test: give a sales rep 20 generated emails without telling them which are AI-generated. Ask: would you send this? Rate 1-5. If average < 4.0, fix the generation prompt."

---

## Phase 5: Debrief (Maya)

> *After APPROVED.*

**Maya:** "You just built your first production agent. Here's what that means:

- **Agents are about tools, not thinking.** The best agent in the world fails with bad tools. The worst agent succeeds with great tools. Tool design is 80% of agent engineering.
- **Latency is architecture, not optimization.** You can't optimize your way out of too many sequential LLM calls. You have to DESIGN for latency — parallel tool calls, aggressive caching, streaming, early stopping.
- **Agent evaluation is fundamentally different from RAG eval.** You're not measuring retrieval quality. You're measuring TRAJECTORY quality — did the agent make good decisions in sequence?
- **Simple agents beat complex agents.** Your agent is 5 lines of LangGraph with 3 tools. It works better than a 20-tool agent with sub-agents. Anthropic's research confirms this — the most effective production agents are simple.

**What you'll see again:**
- **Agent evaluation** — Project 10 (Code Review Agent) uses precision/recall for agent output. Much harder than email quality eval.
- **Tool design** — Project 11 (NL2SQL) has the hardest tool design challenge: letting the agent write and execute arbitrary SQL safely.
- **Streaming** — Every agent project from now on needs streaming. Users won't wait for full completion.

> *Maya: "Project 9 takes us back to RAG — but it's RAG on steroids. Agentic retrieval, vision models for PDF tables, and the hardest evaluation challenge yet: proving zero hallucination."*
