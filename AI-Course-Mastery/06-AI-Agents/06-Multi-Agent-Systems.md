# 06 — Multi-Agent Systems: Collaboration, Delegation & Debate

## 🎯 Purpose & Goals

### 🛑 STOP. Think about why ONE agent isn't always enough.

Your ReAct agent from File 02 is capable. It can search, calculate, and reason. But give it this:

> *"Write a Python script that downloads our sales data, cleans it, generates a visualization, and emails it to the team."*

Your agent needs to:
1. **Write code** (requires Python expertise)
2. **Test the code** (requires debugging skills)
3. **Review for safety** (requires security awareness)
4. **Send the email** (requires communication judgment)

That's 4 different MINDSETS. Your single agent tries to do all 4. And it does... okay. But what if you had SEPARATE agents, each specialized in one thing?

---

### 🤔 Scenario 1: The Jack-of-All-Trades Problem

Your single agent handles: coding, research, writing, data analysis, customer support.

A user asks: *"Debug this Python error and explain it to our non-technical client."*

The agent needs to:
1. Understand the Python traceback (expert-level coding)
2. Find the root cause (deep debugging)
3. Translate to plain English (communication)
4. Write an email to the client (professional writing)

**The problem:** Each of these requires a different "personality." The debugging mindset is analytical and precise. The client communication mindset is empathetic and simple. Your single agent has ONE system prompt. It can't switch mindsets fluidly.

**Your challenge before reading on:** When does it make sense to have SEPARATE agents with DIFFERENT system prompts, each handling one phase of the work?

---

### 🤔 Scenario 2: The Rubber Stamp Problem

Your agent writes code. But you need someone to REVIEW it before it runs.

```
Agent writes: 
  def process_payment(amount):
      db.execute(f"INSERT INTO payments VALUES ({amount})")
      # ← SQL injection vulnerability!

Nobody reviews it → Payment table gets hacked
```

**🤔 Solution:** A separate REVIEWER agent that checks the coder agent's output:

```
Writer Agent: Writes the code
Reviewer Agent: Checks for bugs, security issues, style problems
Approval Gate: Only runs code if Reviewer approves
```

**Your challenge:** Design the protocol between Writer and Reviewer. What information does Reviewer need? What happens when Reviewer rejects? Can Writer appeal?

---

### 🤔 Scenario 3: The Infinite Debate

You have two agents — one optimistic, one pessimistic:

```
User: "Should we invest in this startup?"
Optimist Agent: "Yes! Growing 200% YoY, strong team, huge market!"
Pessimist Agent: "No! Burn rate is high, market is crowded, competitors have deeper pockets."

Optimist: "But the technology is groundbreaking!"
Pessimist: "So was Webvan. They went bankrupt."
Optimist: "Different era, different business model."
Pessimist: "The fundamentals are the same."
... (continues forever)
```

**The problem:** Two agents with opposing views will debate indefinitely. Each has valid points. Neither can "win" because the question is subjective.

**Your challenge:**
1. How do you structure a debate that converges on insight, not endless argument?
2. What stopping criterion makes sense for a multi-agent debate?
3. How does the user get a SYNTHESIS of both views, not just the "winner"?

---

> **These 3 scenarios define the multi-agent landscape.**
>
> By the end of this file, you'll have implemented solutions for ALL three.

**By the end of this file, you will:**
1. Understand the **4 multi-agent patterns** — Supervisor, Debate, Swarm, Pipeline
2. Know when each pattern is appropriate (and when a single agent is better)
3. Build a **Supervisor pattern** — one agent delegates work to specialized workers
4. Build a **Debate pattern** — agents with different views converge on insight
5. Understand **Swarm patterns** — many agents coordinate without central control
6. Measure the cost/benefit of multi-agent vs. single-agent systems

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Scenarios | 15 min |
| The 4 Multi-Agent Patterns | 25 min |
| When Multi-Agent Makes Sense | 15 min |
| Code: Supervisor Pattern | 45 min |
| Code: Debate Pattern | 35 min |
| Code: Pipeline Pattern | 30 min |
| Coordination Protocols | 25 min |
| Cost Analysis: Single vs Multi | 20 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 20 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~5 hours** |

---

## 📖 Concept Deep-Dive

### 1. The 4 Multi-Agent Patterns

```
PATTERN        │ STRUCTURE                        │ BEST FOR
───────────────┼──────────────────────────────────┼────────────────────────
SUPERVISOR     │ One agent delegates to workers   │ Complex tasks with clear
               │ Supervisor decides who does what  │ subtask boundaries

DEBATE         │ Agents argue different positions │ Decision-making where
               │ Moderator synthesizes             │ multiple perspectives help

PIPELINE       │ Each agent passes output to next │ Linear workflows where
               │ Agent A → Agent B → Agent C       │ each step builds on prior

SWARM          │ No central control               │ Highly dynamic tasks
               │ Agents self-organize              │ where roles aren't fixed
```

### 2. When Multi-Agent Makes Sense

**🤔 Critical question: When is multi-agent ACTUALLY better than single-agent?**

Let's be honest about the costs:

```
FACTOR              │ SINGLE AGENT          │ MULTI-AGENT
────────────────────┼───────────────────────┼──────────────────────────
Cost per task       │ $0.01-0.05           │ $0.05-0.50 (5-10x more)
Latency             │ 2-10 seconds         │ 30 seconds - 5 minutes
Code complexity     │ Simple loop          │ Graph with N agents + routing
Debugging           │ Print statements     │ Need tracing across agents
Failure modes       │ One point of failure │ N points of failure
Output quality      │ Good                 │ Better (if designed well)
```

**Multi-agent is worth it when ONE of these is true:**
1. The task has **clearly separable subtasks** (research → analyze → write)
2. Different **perspectives add value** (debate finds blind spots)
3. The task would exceed **context limits** if done by one agent
4. **Specialization** dramatically improves quality (a coder agent vs. a writer agent)

**Multi-agent is NOT worth it when:**
1. The task is simple (one search → one answer)
2. Coordination overhead dominates actual work
3. You haven't verified that a single agent fails at this task

> **Senior engineer insight:** I've seen teams add multi-agent complexity to problems that a simple ReAct agent solved fine. The result: higher costs, more bugs, harder debugging. Start with a single agent. Add agents ONLY when you have MEASURED evidence that the single agent fails. Do NOT pre-optimize for multi-agent.

---

## 💻 Code Examples

### Example 1: Supervisor Pattern

The most practical multi-agent pattern. One agent directs specialized workers:

```python
"""
Supervisor Pattern: One coordinator delegates to specialized workers.

The flow:
1. User gives a complex task
2. Supervisor agent breaks it into subtasks
3. Each subtask → assigned to a specialized worker agent
4. Workers report back
5. Supervisor synthesizes final answer

Built on LangGraph (File 05).
"""

from typing import TypedDict, Annotated, Literal
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
import json
import operator


# ── Worker Agents ──

RESEARCHER_SYSTEM_PROMPT = """You are a RESEARCHER agent. You find and gather information.
Your strengths: thorough web searches, cross-referencing sources, finding data.
Your weaknesses: you don't analyze or make recommendations.
Output: factual findings with sources. Do NOT draw conclusions."""

ANALYST_SYSTEM_PROMPT = """You are an ANALYST agent. You interpret data and find patterns.
Your strengths: identifying trends, comparing options, quantifying tradeoffs.
Your weaknesses: you don't search for new information.
Output: analysis and insights based on data you're given. Do NOT research."""

WRITER_SYSTEM_PROMPT = """You are a WRITER agent. You produce clear, polished output.
Your strengths: structuring information, explaining complex topics, professional tone.
Your weaknesses: you don't research or analyze data.
Output: well-formatted document based on research and analysis you're given."""


def create_worker_agent(system_prompt: str, model: str = "gpt-4o-mini"):
    """Create a specialized worker agent with a fixed system prompt."""
    llm = ChatOpenAI(model=model, temperature=0)
    
    def worker(task: str, context: str = "") -> str:
        """Worker function: takes a task, returns output."""
        messages = [
            SystemMessage(content=system_prompt),
        ]
        if context:
            messages.append(HumanMessage(content=f"Context from previous work:\n{context}"))
        messages.append(HumanMessage(content=f"Your task:\n{task}"))
        
        response = llm.invoke(messages)
        return response.content
    
    return worker


researcher = create_worker_agent(RESEARCHER_SYSTEM_PROMPT)
analyst = create_worker_agent(ANALYST_SYSTEM_PROMPT)
writer = create_worker_agent(WRITER_SYSTEM_PROMPT)


# ── Supervisor Agent ──

supervisor_llm = ChatOpenAI(model="gpt-4o", temperature=0)  # Stronger model for coordination


# ── Graph State ──

class SupervisorState(TypedDict):
    """State for the supervisor multi-agent system."""
    task: str                          # Original user task
    subtasks: list[dict]               # [{agent, task, status, result}]
    current_agent: str                 # Who should work next
    final_output: str
    iteration_count: int


# ── Graph Nodes ──

def plan_subtasks(state: SupervisorState) -> SupervisorState:
    """
    Node: Supervisor plans how to decompose the task.
    
    The supervisor decides:
    1. What subtasks are needed
    2. Which agent should do each
    3. In what order
    """
    task = state["task"]
    
    response = supervisor_llm.invoke([
        SystemMessage(content="""You are a supervisor managing a team of AI specialists.

Available agents:
- RESEARCHER: Finds information, searches the web, gathers data
- ANALYST: Interprets data, finds patterns, compares options
- WRITER: Produces polished documents, explanations, reports

Given a user's task, decide:
1. What subtasks are needed
2. Which agent should do each
3. What order makes sense

Respond with a JSON array of subtasks:
[
    {"agent": "researcher", "task": "Find X", "depends_on": []},
    {"agent": "analyst", "task": "Analyze X", "depends_on": [0]},
    {"agent": "writer", "task": "Write report about X", "depends_on": [0, 1]}
]

RULES:
- Researchers can run in parallel if they're independent
- Analysts need research results first
- Writers need analysis results first
- Keep it simple: 2-4 subtasks max"""),
        HumanMessage(content=task),
    ])
    
    try:
        subtasks = json.loads(response.content)
    except json.JSONDecodeError:
        # Fallback to a simple plan
        subtasks = [
            {"agent": "researcher", "task": f"Research: {task}", "depends_on": []},
            {"agent": "analyst", "task": f"Analyze findings for: {task}", "depends_on": [0]},
            {"agent": "writer", "task": f"Write final output for: {task}", "depends_on": [0, 1]},
        ]
    
    # Add status tracking
    for subtask in subtasks:
        subtask["status"] = "pending"
        subtask["result"] = ""
    
    print(f"\n📋 SUPERVISOR PLAN: {len(subtasks)} subtasks")
    for i, s in enumerate(subtasks):
        print(f"   {i}. [{s['agent']}] {s['task'][:60]}...")
    
    return {
        "subtasks": subtasks,
        "current_agent": "researcher",  # Start with research
    }


def route_to_worker(state: SupervisorState) -> Literal["worker", "synthesize", "end"]:
    """
    Route: Decide which agent should work next, or if we're done.
    
    The supervisor checks which subtasks are:
    - pending (but dependencies are met) → assign to worker
    - all complete → synthesize
    - none can proceed → end with partial results
    """
    subtasks = state.get("subtasks", [])
    
    # Check: are all subtasks complete?
    all_done = all(s["status"] == "completed" for s in subtasks)
    if all_done:
        return "synthesize"
    
    # Check: can any pending subtask proceed?
    for i, s in enumerate(subtasks):
        if s["status"] == "pending":
            deps_met = all(
                subtasks[d]["status"] == "completed"
                for d in s.get("depends_on", [])
            )
            if deps_met:
                return "worker"  # Found work to do
    
    # No work can proceed (probably dependency chain broken)
    return "end"


def execute_worker(state: SupervisorState) -> SupervisorState:
    """
    Node: Execute the next available worker task.
    
    Finds the first pending subtask whose dependencies are met,
    and routes it to the appropriate worker agent.
    """
    subtasks = state["subtasks"]
    current_task = None
    current_idx = None
    
    for i, s in enumerate(subtasks):
        if s["status"] == "pending":
            deps_met = all(
                subtasks[d]["status"] == "completed"
                for d in s.get("depends_on", [])
            )
            if deps_met:
                current_task = s
                current_idx = i
                break
    
    if not current_task:
        return {"iteration_count": state.get("iteration_count", 0) + 1}
    
    # Gather context from dependencies
    context_parts = []
    for d in current_task.get("depends_on", []):
        dep_result = subtasks[d].get("result", "")
        if dep_result:
            context_parts.append(f"From previous step ({subtasks[d]['agent']}):\n{dep_result[:500]}")
    
    context = "\n\n".join(context_parts)
    
    # Route to the right worker
    agent_map = {
        "researcher": researcher,
        "analyst": analyst,
        "writer": writer,
    }
    
    worker_fn = agent_map.get(current_task["agent"])
    if not worker_fn:
        result = f"Unknown agent type: {current_task['agent']}"
    else:
        print(f"\n🔧 [{current_task['agent']}] Working on: {current_task['task'][:80]}...")
        result = worker_fn(current_task["task"], context)
        print(f"   ✅ Done. Output: {result[:100]}...")
    
    # Update subtask status
    updated_subtasks = list(subtasks)
    updated_subtasks[current_idx] = {
        **current_task,
        "status": "completed",
        "result": result,
    }
    
    return {
        "subtasks": updated_subtasks,
        "iteration_count": state.get("iteration_count", 0) + 1,
        "current_agent": current_task["agent"],
    }


def synthesize(state: SupervisorState) -> SupervisorState:
    """
    Node: Supervisor collects all worker outputs and produces final answer.
    """
    subtasks = state["subtasks"]
    
    results_text = ""
    for i, s in enumerate(subtasks):
        results_text += f"\n## Step {i} ({s['agent']}): {s['task']}\n"
        results_text += s.get("result", "No output")[:1000] + "\n"
    
    response = supervisor_llm.invoke([
        SystemMessage(content="""You are a supervisor. Synthesize the work from your team 
into a coherent final answer. Your job is to:
1. Connect findings from different agents
2. Resolve any contradictions
3. Present the information clearly
4. Add your own assessment of quality/confidence"""),
        HumanMessage(content=f"Original task: {state['task']}\n\nTeam results:\n{results_text}\n\nProvide the final synthesized answer."),
    ])
    
    print(f"\n📊 SUPERVISOR SYNTHESIS COMPLETE")
    
    return {
        "final_output": response.content,
        "current_agent": "__end__",
    }


# ── Build the Graph ──

def build_supervisor_system():
    workflow = StateGraph(SupervisorState)
    
    workflow.add_node("planner", plan_subtasks)
    workflow.add_node("worker", execute_worker)
    workflow.add_node("synthesizer", synthesize)
    
    workflow.set_entry_point("planner")
    
    workflow.add_conditional_edges(
        "planner",
        route_to_worker,
        {
            "worker": "worker",
            "synthesize": "synthesizer",
            "end": END,
        }
    )
    workflow.add_conditional_edges(
        "worker",
        route_to_worker,
        {
            "worker": "worker",
            "synthesize": "synthesizer",
            "end": END,
        }
    )
    workflow.add_edge("synthesizer", END)
    
    return workflow.compile()


# ── Run ──

def run_multi_agent(task: str):
    """Run the multi-agent system on a task."""
    app = build_supervisor_system()
    
    result = app.invoke({
        "task": task,
        "subtasks": [],
        "current_agent": "",
        "final_output": "",
        "iteration_count": 0,
    })
    
    print(f"\n{'='*60}")
    print("FINAL OUTPUT:")
    print(f"{'='*60}")
    print(result["final_output"])
    print(f"\n📊 Iterations: {result['iteration_count']}")
    
    return result["final_output"]


# Example usage:
# result = run_multi_agent("Analyze the impact of AI on the Indian fintech sector")
```

---

### 🤔 Checkpoint: Design Your Own Multi-Agent Team

You're building a multi-agent system for **code review**. What agents would you include?

Consider these roles:
- Coder Agent (writes the code)
- Reviewer Agent (finds bugs)
- Security Agent (checks vulnerabilities)
- Style Agent (checks formatting)
- Tester Agent (writes tests)

**Your challenge:** 
1. Which agents are NECESSARY vs. nice-to-have?
2. What's the ORDER of operations? (Can review happen before tests are written?)
3. What happens when agents DISAGREE? (Reviewer says "bad code" but Coder says "it works")
4. What's the cost of adding each agent? (How many extra LLM calls?)

> **Think about this. Then build it in Drill 1.**

---

### Example 2: Debate Pattern

Two agents with different perspectives converge on a balanced conclusion:

```python
"""
Debate Pattern: Two agents argue opposite positions, moderated by a third.

The flow:
1. Proposer argues POSITION A
2. Opposer argues POSITION B
3. Moderator reads both arguments and asks follow-up questions
4. Each agent responds to the other's points
5. After N rounds, Moderator synthesizes

This is NOT about "winning." It's about surfacing ALL perspectives.
"""

def debate(topic: str, position_a: str, position_b: str, rounds: int = 2) -> str:
    """
    Run a structured debate between two AI agents.
    
    Args:
        topic: The question being debated
        position_a: What side A argues
        position_b: What side B argues
        rounds: How many rounds of back-and-forth
    
    Returns: Moderator's synthesis
    """
    llm = ChatOpenAI(model="gpt-4o", temperature=0.3)  # Slight temp for varied arguments
    
    # System prompts
    proposer_prompt = f"""You are arguing FOR: {position_a}
    
Be persuasive but fair. Use evidence and logic.
Acknowledge valid points from the other side.
Your goal: present the strongest case for {position_a}."""

    opposer_prompt = f"""You are arguing AGAINST: {position_a}

Be persuasive but fair. Use evidence and logic.
Acknowledge valid points from the other side.
Your goal: present the strongest case for {position_b}."""

    moderator_prompt = f"""You are a moderator. You have two debaters arguing about: {topic}

Your job:
1. After each round, identify the STRONGEST point from each side
2. Ask ONE follow-up question that challenges BOTH sides
3. After {rounds} rounds, synthesize a balanced conclusion
4. Your synthesis should NOT pick a winner — it should articulate 
   when each position is valid and what the tradeoffs are"""

    proposer = create_worker_agent(proposer_prompt)
    opposer = create_worker_agent(opposer_prompt)
    moderator = create_worker_agent(moderator_prompt)
    
    # Track the debate
    transcript = []
    
    # Round 1: Opening statements
    print(f"\n🎯 DEBATE: {topic}")
    print(f"   FOR: {position_a}")
    print(f"   AGAINST: {position_b}")
    
    a_opening = proposer(f"Present your opening argument for {position_a}. Be specific and cite evidence.")
    b_opening = opposer(f"Present your opening argument for {position_b}. Be specific and cite evidence.")
    
    transcript.append(f"=== Round 1 ===")
    transcript.append(f"\n[FOR {position_a}]: {a_opening[:500]}")
    transcript.append(f"\n[FOR {position_b}]: {b_opening[:500]}")
    
    print(f"\n📢 Round 1:")
    print(f"   A: {a_opening[:100]}...")
    print(f"   B: {b_opening[:100]}...")
    
    for round_num in range(1, rounds):
        # Moderator asks a challenge question
        challenge = moderator(f"""Current transcript:
{chr(10).join(transcript[-4:])}

What is the most important question that BOTH sides need to address?
Ask ONE specific question that tests both positions.""")
        
        print(f"\n❓ Moderator: {challenge[:150]}...")
        
        # Each side responds
        a_response = proposer(f"""The moderator asks: {challenge}

Your opponent said: {b_opening[:300]}

Respond to the challenge AND address your opponent's point.""")
        
        b_response = opposer(f"""The moderator asks: {challenge}

Your opponent said: {a_opening[:300]}

Respond to the challenge AND address your opponent's point.""")
        
        transcript.append(f"\n=== Round {round_num + 1} ===")
        transcript.append(f"\n[Moderator]: {challenge}")
        transcript.append(f"\n[FOR {position_a}]: {a_response[:500]}")
        transcript.append(f"\n[FOR {position_b}]: {b_response[:500]}")
        
        print(f"\n📢 Round {round_num + 1}:")
        print(f"   A: {a_response[:100]}...")
        print(f"   B: {b_response[:100]}...")
    
    # Final: Moderator synthesis
    synthesis = moderator(f"""Full debate transcript:
{chr(10).join(transcript)}

Provide a balanced synthesis that:
1. Summarizes the strongest argument from each side
2. Identifies areas of agreement
3. Identifies key disagreements that remain
4. Gives practical guidance: when would you choose {position_a} vs {position_b}?
5. Does NOT declare a winner""")
    
    print(f"\n{'='*40}")
    print("📊 MODERATOR SYNTHESIS:")
    print(f"{'='*40}")
    print(synthesis)
    
    return synthesis
```

---

### 🤔 Checkpoint: The Debate Cost-Benefit

A single debate with 2 rounds costs:

```
Round 1: 2 LLM calls (opening statements)
Challenge: 1 LLM call (moderator)
Round 2: 2 LLM calls (responses)
Synthesis: 1 LLM call

Total: 6 LLM calls = ~$0.03-0.12 depending on model

Same insight from a single agent:
1 LLM call = ~$0.005-0.02
```

**Question:** Is the debate worth 3-6x the cost?

**When it IS:**
- High-stakes decisions (investment, strategy, legal)
- When bias matters (single agent has blind spots)
- When the answer isn't obvious

**When it ISN'T:**
- Simple factual questions
- Time-sensitive requests
- Tasks where accuracy doesn't vary much

---

## ✅ Good Output Examples

### What a Great Multi-Agent Trace Looks Like

```
Task: "Write a competitive analysis report on our top 3 competitors"

SUPERVISOR PLAN:
  Step 0: [Researcher] Find latest pricing, funding, products for each competitor
  Step 1: [Analyst] Compare and identify trends, strengths, weaknesses
  Step 2: [Writer] Produce polished report

EXECUTION:
  🔧 [Researcher] Searching 3 competitors... Done (4 searches)
  🔧 [Analyst] Analyzing data... Done (identified 3 key trends)
  🔧 [Writer] Writing report... Done (1500 words with sections)

COST BREAKDOWN:
  Supervisor planning: $0.002 (gpt-4o)
  Researcher: 4 searches + 1 LLM call = $0.008
  Analyst: 1 LLM call = $0.005
  Writer: 1 LLM call = $0.005
  Supervisor synthesis: $0.002
  Total: $0.022

COMPARED TO SINGLE AGENT:
  Single agent: 6-8 iterations, 6-8 LLM calls = $0.015
  Multi-agent: specialized workers, fewer iterations = $0.022
  
  Extra cost: $0.007 (47% more)
  Quality improvement: Measurably better (verified by human review)
  → Worth it for this task type
```

### What a BAD Multi-Agent Trace Looks Like

```
Task: "What's the weather in Tokyo?"

SUPERVISOR PLAN:
  Step 0: [Researcher] Find Tokyo weather
  Step 1: [Analyst] Analyze weather data
  Step 2: [Writer] Write weather report

🔴 PROBLEM:
  The task is SIMPLE. A single search answers it.
  Multi-agent adds 400% overhead for ZERO quality gain.

COST BREAKDOWN:
  Supervisor planning: $0.002
  Researcher: $0.005 (1 search)
  Analyst: $0.005 (analyzing a simple weather result!)
  Writer: $0.005 (writing "it's 22°C and sunny")
  Supervisor synthesis: $0.002
  Total: $0.019

  Single agent cost: $0.003
  Multi-agent overhead: 533% more expensive
  Quality difference: IDENTICAL
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Agent Spam

```python
# ❌ WRONG — Adding agents because "more is better"
team = [
    ResearchAgent(), AnalysisAgent(), WritingAgent(),
    FactCheckAgent(), StyleAgent(), FormatAgent(),
    TranslationAgent(), SummaryAgent(), ReviewAgent(),
]
# Each adds cost. Each adds latency. Most add no value.

# ✅ RIGHT — Start minimal, add based on evidence
team = [ResearchAgent(), WritingAgent()]
# Add AnalysisAgent ONLY if research → write is missing analysis
# Add FactCheckAgent ONLY if writing has hallucinations
```

### Antipattern 2: The Ping-Pong Problem

```
Agent A: "I found the data"
Agent B: "Let me analyze it"
Agent A: "Actually, I found more data"
Agent B: "Let me re-analyze with the new data"
Agent A: "Wait, I also found..."
Agent B: "Let me re-analyze again..."
... (infinite loop)
```

**Fix:** Use LangGraph with explicit edges and iteration limits. Workers don't decide when to work — the graph does.

### Antipattern 3: No Shared Context

```
Researcher finds: "Company revenue: $10M"
Analyst analyzes: "Assuming revenue of $5M..." ← Different number!
Writer writes: "Revenue was $5M based on analysis." ← Compound error!

ROOT CAUSE: Analyst used a DIFFERENT number than Researcher found.
No shared context was passed between agents.
```

**Fix:** Always pass previous agent outputs as context to the next agent. Use structured state (like we did in the Supervisor example).

### Antipattern 4: The $1.00 Coffee

```
Agent supervises → Agent researches → Agent analyzes → Agent writes
                                            ↓
                                 Agent reviews → Agent approves
                                            ↓
                                 Agent formats → Agent delivers
                    
Latency: 45 seconds
Cost: $0.87
Quality: Same as single agent
```

**🤔 Sometimes a single agent is ENOUGH.** Don't deploy a multi-agent system for tasks a single agent can handle. The overhead is real.

---

## 🧪 Drills & Challenges

### Drill 1: Build a Code Review Multi-Agent System (30 min)

Build a system with:
1. **Coder Agent** — writes Python code based on requirements
2. **Reviewer Agent** — finds bugs, security issues, style problems
3. **Tester Agent** — generates test cases and runs them (in sandbox)

**The flow:**
```
Coder writes code → Reviewer checks → IF issues found → Coder fixes
                                                     → Reviewer re-checks
                  → IF clean → Tester runs tests → IF pass → Done
                                                   → IF fail → Coder fixes → loop
```

**Your challenge:** How many fix-review cycles should you allow before escalating to a human?

### Drill 2: The Debate Moderator (25 min)

Run a debate between two agents on:

> *"Should my startup use a managed vector database (Pinecone/Qdrant) or self-host (ChromaDB/pgvector)?"*

One agent argues FOR managed. One argues FOR self-hosted.

**After the debate, evaluate:**
1. Did the debate surface insights a single agent wouldn't?
2. Was the moderator effective at challenging both sides?
3. Was the final synthesis actually useful?
4. Would you pay 3x for this vs. a single agent answer?

### Drill 3: The Cost-Benefit Calculator (20 min)

For each task, estimate: single-agent cost vs. multi-agent cost vs. quality difference:

```
Task A: "What's the capital of France?" (simple fact)
Task B: "Write a 5-page business plan for an AI startup" (complex creation)
Task C: "Debug this Python traceback" (technical analysis)
Task D: "Should we acquire Company X?" (strategic decision)
Task E: "Summarize this 50-page document" (information extraction)
```

**Your analysis should answer:** When is the extra cost of multi-agent worth it?

### Drill 4: The Escalation Protocol (20 min)

Design the protocol for when a multi-agent system can't complete a task:

```
Agent A (Researcher) can't find enough data on a topic.
What does it do?
  A) Return partial results and say "insufficient data"
  B) Ask the Supervisor for alternative approaches
  C) Try 3 different search strategies before giving up
  D) Ask the human user for guidance

Design the FULL escalation chain for a 3-agent system.
```

### Drill 5: The Single-Agent Baseline (15 min)

Before you ever build a multi-agent system, you MUST establish the single-agent baseline:

```python
# Run the SAME task on a single agent
single_result = single_agent.run(task)
single_cost = single_agent.total_cost
single_time = single_agent.execution_time

# Run on multi-agent
multi_result = multi_agent.run(task)
multi_cost = multi_agent.total_cost
multi_time = multi_agent.execution_time

# Compare
quality_improvement = evaluate_quality(multi_result, single_result)
cost_multiplier = multi_cost / single_cost
time_multiplier = multi_time / single_time

# Decision:
use_multi_agent = quality_improvement > 0.2 and cost_multiplier < 5
```

**Build this comparison function.** Never deploy multi-agent without this analysis.

---

## 🚦 Gate Check

Before moving to File 07 (MCP Protocol), confirm you can:

- [ ] **Explain the 4 multi-agent patterns** with when to use each
- [ ] **Build a Supervisor pattern** with task decomposition and worker routing
- [ ] **Build a Debate pattern** with moderation and synthesis
- [ ] **Implement shared context** between agents
- [ ] **Calculate cost/benefit** of multi-agent vs single-agent
- [ ] **Recognize when NOT to use multi-agent**

### The Real Test

You have a task: *"Research our top competitor, analyze their pricing strategy, and write a recommendation for our pricing team."*

**Design the multi-agent system:**
1. What agents do you need?
2. What's the flow? (parallel? sequential? conditional?)
3. What does each agent see as context?
4. What's the estimated cost?
5. What's the fallback if one agent fails?

**Then: Compare with a single agent doing the same task. Which is better? Why?**

### Final Questions

1. **Cross-reference File 05:** You built a LangGraph agent with conditional routing. A multi-agent supervisor is the SAME pattern but with DIFFERENT agents in different nodes. How is a multi-agent graph fundamentally different from a single-agent graph? (Hint: think about state, tools, and LLM instances.)

2. **The "too many cooks" problem:** You have 5 agents working on a task. Each adds some value. But the coordination overhead means the task takes 10x longer. At what point do the costs of coordination outweigh the benefits of specialization? How do you MEASURE this?

3. **The oracle problem:** In a debate between two agents, who decides when the debate is over? The moderator? A timer? The user? Each choice has tradeoffs. When would you use each?

---

## 📚 Resources

**Foundational:**
- 📄 [Multi-Agent Collaboration in LangGraph](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/multi-agent-collaboration/) — Official tutorial
- 📄 [AutoGen: Multi-Agent Conversations](https://microsoft.github.io/autogen/) — Microsoft's multi-agent framework
- 📄 [CrewAI Documentation](https://docs.crewai.com/) — Popular multi-agent framework

**Patterns:**
- 📄 [Supervisor Pattern](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/agent_supervisor/)
- 📄 [Debate Pattern for Better Reasoning](https://arxiv.org/abs/2305.19118) — "Improving Factuality and Reasoning in Language Models through Multiagent Debate"
- 📄 [Swarm Patterns](https://github.com/openai/swarm) — OpenAI's experimental swarm framework

**Your Multi-Agent Progress Checklist:**
```
☐ 4 multi-agent patterns understood
☐ Supervisor pattern built and tested
☐ Debate pattern implemented
☐ Shared context between agents working
☐ Cost/benefit analysis done
☐ Single-agent baseline established
☐ Escalation protocol designed
☐ When-to-use-multi-agent decision framework clear
```

---

> **Revisit the 3 scenarios from the beginning:**
>
> 1. **The Jack-of-All-Trades Problem** — Solved by the Supervisor pattern. Each agent has a specialized system prompt. The supervisor decomposes tasks and routes them appropriately. Research agents don't write reports. Writer agents don't search web.
>
> 2. **The Rubber Stamp Problem** — Solved by separate agents for writing and reviewing. The reviewer catches issues the writer missed. If they disagree, a third agent (or human) mediates. This is the pattern behind production code review systems.
>
> 3. **The Infinite Debate** — Solved by structured debates with a moderator. Limited rounds. Moderator asks challenge questions. Final synthesis extracts insight from both sides without declaring a "winner." The debate CONVERGES because the moderator forces it to.
>
> **Key realization:** Multi-agent isn't about more agents — it's about the RIGHT agents with the RIGHT coordination. A well-designed 2-agent system (researcher + writer) outperforms a poorly-designed 5-agent system every time. Start small. Add agents only when you can measure the improvement.
>
> **Next:** File 07 — MCP Protocol. The emerging standard for connecting agents to tools, APIs, and data sources.

---

> **Senior engineer closing thought:**
>
> *"The most common mistake I see is premature multi-agent. Engineers hear 'agents' and immediately design a system with 5+ specialized agents. Then they spend weeks debugging coordination issues. The truth: most tasks don't need multi-agent. A single ReAct agent with good tools handles 80% of use cases. Multi-agent shines for the 20% where tasks genuinely need different mindsets or parallel work. Add agents like you add dependencies — reluctantly, with measured evidence."*
