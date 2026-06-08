# 01 — Agent Foundations: Core Loop, Planning & Architecture

## 🎯 Purpose & Goals

> **🛑 STOP. Before I define agents — think.**
>
> You have a RAG engine from Phase 5. It can find documents and answer questions. But what if you need it to:
> - *"Research Project Alpha's Q4 impact — search internal docs, check recent news, cross-reference financial data, and write a report"*
> - *"Monitor our API error logs every hour, identify patterns, page the right team, and create a GitHub issue"*
>
> A RAG system answers questions. An AGENT **accomplishes goals** — it decides WHAT to do, in what ORDER, and when to STOP.

### 🤔 Discovery Question 1: Before You Read the Definition

You have:
- An LLM that can generate text and reason
- A set of functions (search web, run code, query DB, send email)

Your boss says: *"I need a competitive analysis report on our top 3 competitors by Friday. Include their pricing, recent funding, product changes this quarter, and how we compare. Go."*

**Your challenge:** Design a system where the LLM **decides the steps.** Not you writing a script. The LLM looks at the goal and figures out: "First I should search for each competitor's pricing, then search for recent funding news, then compare them, then write the report."

What's the **loop** that enables this? How does the LLM know when it's done vs. when to keep going? What happens when it makes a mistake?

---

### 🤔 Discovery Question 2: The Tool Problem

Your LLM has these capabilities:
1. `search_web(query)` → returns search results
2. `fetch_page(url)` → returns page content
3. `query_database(sql)` → returns query results
4. `send_email(to, subject, body)` → sends an email
5. `write_file(path, content)` → writes to disk

A user tells the agent: *"Email my team the sales data from our database."*

**Before reading on, think about:**
- How does the LLM know what arguments to pass to `query_database`?
- How does it know which `search_web` queries to run?
- What if it decides to `send_email` to the wrong person?
- What if it calls `write_file` to overwrite an important file?

**The core problem:** The LLM can reason, but it needs a **structured way** to call tools and use their results. This is the agent loop — the bridge between LLM reasoning and real-world action.

---

### 🤔 Discovery Question 3: The Planning Trap

An agent needs to plan. But here's the problem:

> *"I need to analyze our customer churn data"*

A naive agent might:
1. Search for "customer churn data location"
2. Read the data
3. Try to analyze it
4. Realize it needs to know what "churn" means in this context
5. Search for "churn definition"
6. Re-read the data with this context
7. Write the analysis

**That's 7 steps. What if step 4 reveals step 2 was wrong?** The agent needs to **re-plan** — go back, search for more information, and adjust.

**Your challenge:** How do you design an agent that can change its plan mid-execution? What's the difference between a plan and a "stubborn script that can't adapt"?

---

> **Keep these 3 questions in mind. By the end of this file, you'll have answers to all of them.**

**By the end of this file, you will:**
1. Understand the **agent loop** — the fundamental cycle of reasoning, acting, and observing
2. Build a **zero-shot agent** from scratch that plans before acting
3. Implement **tool calling** — the bridge between LLM reasoning and external systems
4. Understand **planning strategies** — when to plan fully vs. plan step-by-step
5. See exactly where simple agents break (and why File 02 fixes it)

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 15 min |
| What Is an Agent? | 20 min |
| The Agent Loop — Core Architecture | 30 min |
| Code: Zero-Shot Agent from Scratch | 60 min |
| Code: Tool Calling Implementation | 45 min |
| Planning Strategies (Plan-then-Execute vs. Step-by-Step) | 30 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 20 min |
| Drills & Challenges | 30 min |
| Gate Check | 10 min |
| **Total** | **~5 hours** |

---

## 📖 Concept Deep-Dive

### 1. What Is an Agent?

An **AI agent** is an LLM wrapped in a loop that gives it three capabilities a raw LLM doesn't have:

1. **Tool use** — call functions to interact with the world (search, compute, act)
2. **Memory** — maintain state across multiple reasoning steps
3. **Planning** — decide what to do next based on what happened before

**The fundamental agent loop:**

```
┌─────────────────────────────────────────────────┐
│                  AGENT LOOP                      │
│                                                   │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│   │  THINK   │──→│   ACT    │──→│ OBSERVE  │    │
│   │ (reason) │   │ (tool)   │   │ (result) │    │
│   └──────────┘   └──────────┘   └──────────┘    │
│         ↑                            │           │
│         └────────────────────────────┘           │
│             (loop until goal achieved)            │
└─────────────────────────────────────────────────┘
```

**The critical distinction:**

| Raw LLM | Agent |
|---------|-------|
| One-shot response | Multi-step reasoning |
| No tool access | Can call APIs, run code, query DBs |
| No memory of past steps | Full history of actions and results |
| Can't self-correct | Can retry, re-plan, and adapt |
| Passive (user drives) | Active (decides what to do) |
| Returns text | Accomplishes goals |

### 2. The Agent Architecture

Every agent has four components:

```
┌──────────────────────────────────────────────────────────┐
│                    AGENT SYSTEM                            │
│                                                            │
│  ┌──────────────────────────────────────────────────┐     │
│  │ 1. CORE LLM (the reasoning engine)               │     │
│  │    • GPT-4o, Claude, Gemini — the "brain"        │     │
│  │    • Given: system prompt + conversation history  │     │
│  │    • Returns: text or tool call instructions      │     │
│  └──────────────────────────────────────────────────┘     │
│                                                            │
│  ┌──────────────────────────────────────────────────┐     │
│  │ 2. TOOL LAYER (the hands)                        │     │
│  │    • Structured definitions of available APIs    │     │
│  │    • Function schemas the LLM can "call"         │     │
│  │    • Execution layer that runs the actual code   │     │
│  └──────────────────────────────────────────────────┘     │
│                                                            │
│  ┌──────────────────────────────────────────────────┐     │
│  │ 3. MEMORY / STATE (the context)                  │     │
│  │    • Full history: every thought, action, result │     │
│  │    • Working memory: current conversation        │     │
│  │    • Long-term memory: persisted knowledge       │     │
│  └──────────────────────────────────────────────────┘     │
│                                                            │
│  ┌──────────────────────────────────────────────────┐     │
│  │ 4. CONTROL LOOP (the orchestration)              │     │
│  │    • Decide: LLM chooses action or final answer  │     │
│  │    • Execute: run the chosen tool                 │     │
│  │    • Observe: feed result back to LLM            │     │
│  │    • Loop: repeat until done or max iterations    │     │
│  └──────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

### 3. The Agent Loop in Detail

The control loop is where everything happens. Here's the exact sequence:

```
ITERATION N:
   1. LLM receives: system prompt + full conversation history
   2. LLM outputs EITHER:
      a) A tool call (name + arguments) → "I need to search for X"
      b) A final answer → "Here's the report"
   3. IF tool call:
      a) Validate the tool exists and arguments are valid
      b) Execute the tool function with arguments
      c) Append tool result to conversation history
      d) GO TO step 1 (next iteration)
   4. IF final answer:
      a) Return the answer to the user
      b) EXIT loop
```

**⚠️ SAFETY LIMIT:** Every agent loop must have a `max_iterations` cap. Without it:
- An agent can loop indefinitely ($$$ cost)
- An agent can get stuck in a "think → act → observe → think" cycle forever
- **Rule of thumb:** Start with 10 iterations. Increase only if you have specific reasons.

### 4. Planning Strategies

There are fundamentally two approaches to agent planning:

#### Strategy A: Plan-Then-Execute (Zero-Shot Planning)

The agent creates a full plan BEFORE taking any actions:

```
Plan: "To write a competitive analysis report, I will:
  1. Search for Competitor A's pricing and recent news
  2. Search for Competitor B's pricing and recent news
  3. Search for Competitor C's pricing and recent news
  4. Fetch the top results from each search
  5. Synthesize findings into a comparison table
  6. Write the final report"

Then execute step-by-step, checking off completed items.
```

**Good for:** Well-understood tasks with clear steps
**Bad for:** Exploratory tasks where later steps depend on earlier findings

#### Strategy B: ReAct (Step-by-Step Planning)

The agent plans ONE step at a time, based on what it just learned:

```
Thought: "I need competitor pricing data. Let me start with Competitor A."
Action: search_web("Competitor A pricing 2025")
Observation: "Pricing page shows $49/month for basic plan..."

Thought: "Interesting. Basic is $49. Let me check if they have an enterprise tier."
Action: search_web("Competitor A enterprise pricing")
Observation: "Custom pricing, no public info..."

Thought: "Competitor A doesn't publish enterprise pricing. Let me check Competitor B."
Action: search_web("Competitor B pricing 2025")
...
```

**Good for:** Exploratory tasks, research, debugging
**Bad for:** Simple, linear tasks (adds unnecessary overhead)

#### Which Should You Use?

| Factor | Plan-Then-Execute | ReAct |
|--------|-------------------|-------|
| Task clarity | Well-defined | Exploratory |
| Step dependency | Independent | Sequential |
| Cost | Lower (fewer LLM calls) | Higher (LLM call per step) |
| Flexibility | Rigid | Adaptive |
| Best for | Data processing, report generation | Research, analysis, debugging |

**Senior engineer insight:** Start with ReAct. It's more robust and handles surprises better. Switch to plan-then-execute ONLY when you have strong evidence that the task steps are predictable.

```
Senior engineer: "Plan-then-execute looks good in demos because the plan is always 
perfect. In production, plans change the moment you hit reality. ReAct adapts. 
Plan-then-execute breaks."

Junior engineer: "But ReAct makes more API calls and costs more?"

Senior engineer: "Correct. But a broken plan costs more — wasted API calls, bad results,
and your time debugging. Robustness over cost, always. Optimize cost AFTER you have 
correctness."
```

---

## 💻 Code Examples

### Example 1: Zero-Shot Agent from Scratch

Let's build the simplest possible agent — one that plans before executing:

```python
"""
Zero-Shot Agent: Plan first, then execute step by step.

This agent:
1. Takes a goal
2. Creates a plan (list of steps with tool calls)
3. Executes each step
4. Returns the result

No frameworks. Just Python + OpenAI API.
"""

import os
import json
import time
from typing import Callable, Any
from dataclasses import dataclass, field
from openai import OpenAI


@dataclass
class Tool:
    """A tool an agent can use."""
    name: str
    description: str
    parameters: dict        # JSON Schema for parameters
    function: Callable       # The actual Python function
    

@dataclass
class Step:
    """A single step in the agent's plan."""
    step_number: int
    tool: str
    arguments: dict
    reasoning: str
    result: str = ""
    status: str = "pending"  # pending | running | completed | failed
    

@dataclass
class Plan:
    """The agent's execution plan."""
    goal: str
    steps: list[Step] = field(default_factory=list)
    current_step: int = 0
    
    def to_string(self) -> str:
        lines = [f"Goal: {self.goal}", ""]
        for i, step in enumerate(self.steps):
            marker = "→" if i == self.current_step else " "
            lines.append(f"[{marker}] Step {step.step_number}: {step.reasoning}")
            lines.append(f"    Tool: {step.tool}({json.dumps(step.arguments)})")
            if step.result:
                lines.append(f"    Result: {step.result[:100]}...")
            if step.status == "failed":
                lines.append(f"    ⚠️ FAILED")
        return "\n".join(lines)


class ZeroShotAgent:
    """
    Agent that creates a complete plan before executing.
    
    🤔 Why zero-shot planning?
    For tasks with well-understood steps, planning upfront is more efficient
    than reasoning at every step (ReAct). The LLM "thinks ahead" and creates
    a roadmap, then executes without needing to re-think at each step.
    
    Tradeoff: If the plan is wrong, the agent executes a wrong plan.
    """
    
    def __init__(
        self,
        system_prompt: str,
        tools: list[Tool],
        model: str = "gpt-4o-mini",
        max_iterations: int = 15,
    ):
        self.system_prompt = system_prompt
        self.tools = {t.name: t for t in tools}
        self.model = model
        self.max_iterations = max_iterations
        self.client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
        self.messages: list[dict] = []
        self.total_cost: float = 0.0
    
    def _tool_schemas(self) -> list[dict]:
        """Convert tools to OpenAI function calling schema."""
        return [
            {
                "type": "function",
                "function": {
                    "name": tool.name,
                    "description": tool.description,
                    "parameters": tool.parameters,
                }
            }
            for tool in self.tools.values()
        ]
    
    def _llm_call(self, messages: list[dict], tools: bool = True) -> Any:
        """Make an LLM call with optional tool access."""
        kwargs = {
            "model": self.model,
            "messages": messages,
            "temperature": 0.0,  # Agents should be deterministic
        }
        if tools:
            kwargs["tools"] = self._tool_schemas()
            kwargs["tool_choice"] = "auto"
        
        response = self.client.chat.completions.create(**kwargs)
        
        # Track cost (approximate)
        input_tokens = response.usage.prompt_tokens
        output_tokens = response.usage.completion_tokens
        # gpt-4o-mini pricing as of 2025
        self.total_cost += (input_tokens * 0.15 / 1_000_000) + (output_tokens * 0.60 / 1_000_000)
        
        return response.choices[0].message
    
    def _create_plan(self, goal: str) -> Plan:
        """
        Step 1: Create a detailed execution plan.
        
        The LLM generates a structured plan with:
        - What steps are needed
        - Which tool to use at each step
        - What arguments to pass
        - Why this step is necessary
        """
        planning_prompt = f"""You are a planning agent. Your job is to create a detailed, 
step-by-step plan to accomplish the user's goal.

Available tools:
{chr(10).join(f"  - {t.name}: {t.description}" for t in self.tools.values())}

For each step, specify:
1. What reasoning leads to this step
2. Which tool to call
3. What arguments to pass

Return a JSON plan with this structure:
{{
    "reasoning": "Overall strategy for accomplishing this goal",
    "steps": [
        {{
            "step_number": 1,
            "reasoning": "Why this step is needed",
            "tool": "tool_name",
            "arguments": {{"arg1": "value1"}}
        }}
    ]
}}

Goal: {goal}

IMPORTANT: 
- Each step must use ONE of the available tools
- Steps should be independent where possible
- List steps in execution order
- Do NOT include a "final answer" step — the system handles that
"""
        
        message = self._llm_call(
            [{"role": "system", "content": planning_prompt}],
            tools=False  # Planning doesn't need tool access
        )
        
        try:
            plan_data = json.loads(message.content)
        except json.JSONDecodeError:
            # Fallback: extract JSON from the response
            import re
            json_match = re.search(r'\{.*\}', message.content, re.DOTALL)
            if json_match:
                plan_data = json.loads(json_match.group())
            else:
                raise ValueError(f"LLM returned non-JSON plan: {message.content[:200]}")
        
        steps = []
        for s in plan_data["steps"]:
            steps.append(Step(
                step_number=s["step_number"],
                tool=s["tool"],
                arguments=s["arguments"],
                reasoning=s["reasoning"],
            ))
        
        return Plan(goal=goal, steps=steps)
    
    def _execute_tool(self, tool_name: str, arguments: dict) -> str:
        """Execute a tool and return its result."""
        if tool_name not in self.tools:
            return f"Error: Unknown tool '{tool_name}'. Available: {list(self.tools.keys())}"
        
        tool = self.tools[tool_name]
        
        try:
            result = tool.function(**arguments)
            # Convert to string for LLM consumption
            if isinstance(result, (dict, list)):
                return json.dumps(result, indent=2)
            return str(result)
        except Exception as e:
            return f"Error executing {tool_name}: {str(e)}"
    
    def run(self, goal: str) -> str:
        """
        Execute the full agent pipeline:
        1. Plan → 2. Execute each step → 3. Synthesize final answer
        """
        print(f"\n{'='*60}")
        print(f"🎯 Goal: {goal}")
        print(f"{'='*60}\n")
        
        # Phase 1: Plan
        print("📋 PHASE 1: PLANNING")
        print("-" * 40)
        
        plan = self._create_plan(goal)
        print(f"Strategy: {plan.to_string()}\n")
        
        if not plan.steps:
            return "I couldn't create a plan for this goal. Please be more specific."
        
        print(f"Plan has {len(plan.steps)} steps. Executing...\n")
        
        # Phase 2: Execute
        print("⚙️ PHASE 2: EXECUTION")
        print("-" * 40)
        
        self.messages.append({"role": "system", "content": self.system_prompt})
        self.messages.append({
            "role": "user",
            "content": f"Goal: {goal}\n\nPlan: {plan.to_string()}\n\nExecute each step and provide the final result."
        })
        
        iteration = 0
        while iteration < self.max_iterations:
            iteration += 1
            print(f"\n[Iteration {iteration}/{self.max_iterations}]")
            
            message = self._llm_call(self.messages)
            
            if not message.tool_calls:
                # LLM is done — return final answer
                print(f"[Agent] Task complete after {iteration} iterations")
                print(f"[Cost] ~${self.total_cost:.6f}")
                return message.content
            
            # Execute each tool call (usually 1, but parallel possible)
            for tool_call in message.tool_calls:
                tool_name = tool_call.function.name
                tool_args = json.loads(tool_call.function.arguments)
                
                print(f"  🔧 {tool_name}({json.dumps(tool_args)})")
                
                result = self._execute_tool(tool_name, tool_args)
                result_preview = result[:200] + "..." if len(result) > 200 else result
                print(f"  📥 Result: {result_preview}")
                
                # Add tool result to conversation history
                self.messages.append({
                    "role": "assistant",
                    "content": None,
                    "tool_calls": [
                        {
                            "id": tool_call.id,
                            "type": "function",
                            "function": {
                                "name": tool_name,
                                "arguments": json.dumps(tool_args)
                            }
                        }
                    ]
                })
                self.messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result
                })
        
        return f"Max iterations ({self.max_iterations}) reached. Here's what I have so far."
```

### Example 2: Real Tools the Agent Can Use

```python
"""
Example tools for the ZeroShotAgent.
Each tool is a real, working function.
"""

import httpx
import datetime
import json
import math
from typing import Optional


def search_web(query: str) -> str:
    """
    Search the web for current information.
    Uses DuckDuckGo's API (no API key required).
    
    🤔 Why DuckDuckGo? 
    For development and prototyping, it's free and requires no auth.
    For production, replace with SerpAPI, Bing API, or a custom crawler.
    """
    try:
        r = httpx.get(
            "https://api.duckduckgo.com/",
            params={"q": query, "format": "json", "no_html": 1},
            timeout=10.0
        )
        data = r.json()
        
        results = []
        if data.get("AbstractText"):
            results.append(f"Summary: {data['AbstractText']}")
        
        if data.get("RelatedTopics"):
            for topic in data["RelatedTopics"][:3]:
                if "Text" in topic:
                    results.append(f"- {topic['Text']}")
        
        return "\n".join(results) if results else f"No results found for: {query}"
    except Exception as e:
        return f"Search failed: {str(e)}"


def get_current_time() -> str:
    """Get the current date and time."""
    now = datetime.datetime.now()
    return now.strftime("%Y-%m-%d %H:%M:%S")


def calculate(expression: str) -> str:
    """
    Safely evaluate a mathematical expression.
    
    ⚠️ SAFETY: eval() is dangerous. We restrict it to only math functions.
    The __builtins__ dict prevents access to dangerous Python features.
    
    Senior engineer note: This is still not perfectly safe. A determined
    attacker might find ways around it. In production, use a sandboxed
    environment or a dedicated expression parser like `asteval`.
    """
    allowed = {
        "abs": abs, "round": round, "min": min, "max": max,
        "sum": sum, "pow": pow, "sqrt": math.sqrt,
        "pi": math.pi, "e": math.e,
        "sin": math.sin, "cos": math.cos, "tan": math.tan,
    }
    try:
        result = eval(expression, {"__builtins__": {}}, allowed)
        return str(result)
    except Exception as e:
        return f"Calculation error: {str(e)}"


def fetch_page(url: str) -> str:
    """
    Fetch and extract text content from a URL.
    """
    try:
        r = httpx.get(url, timeout=15.0, follow_redirects=True)
        r.raise_for_status()
        
        # Basic HTML tag stripping (in production, use BeautifulSoup)
        import re
        text = re.sub(r'<[^>]+>', ' ', r.text)
        text = re.sub(r'\s+', ' ', text).strip()
        
        # Limit to first 3000 characters
        return text[:3000]
    except Exception as e:
        return f"Failed to fetch page: {str(e)}"


def get_weather(city: str) -> str:
    """
    Get current weather for a city.
    Free API — no key needed for wttr.in.
    """
    try:
        r = httpx.get(f"https://wttr.in/{city}?format=%C+%t+%w+%h", timeout=10.0)
        return f"Weather in {city}: {r.text.strip()}"
    except Exception as e:
        return f"Weather lookup failed: {str(e)}"


# ── Define tools for the agent ──

tools = [
    Tool(
        name="search_web",
        description="Search the web for current information. Use for facts, news, and research.",
        parameters={
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "The search query"
                }
            },
            "required": ["query"]
        },
        function=search_web
    ),
    Tool(
        name="get_current_time",
        description="Get the current date and time. Use when you need to know when it is now.",
        parameters={
            "type": "object",
            "properties": {}
        },
        function=get_current_time
    ),
    Tool(
        name="calculate",
        description="Evaluate a mathematical expression. Use for calculations, comparisons, and data analysis.",
        parameters={
            "type": "object",
            "properties": {
                "expression": {
                    "type": "string",
                    "description": "Mathematical expression to evaluate (e.g., '150 * 0.15')"
                }
            },
            "required": ["expression"]
        },
        function=calculate
    ),
    Tool(
        name="fetch_page",
        description="Fetch the text content of a URL. Use to read articles, docs, or web pages.",
        parameters={
            "type": "object",
            "properties": {
                "url": {
                    "type": "string",
                    "description": "The URL to fetch"
                }
            },
            "required": ["url"]
        },
        function=fetch_page
    ),
]
```

### Example 3: Running the Agent

```python
if __name__ == "__main__":
    agent = ZeroShotAgent(
        system_prompt="""You are a research assistant. You help answer questions 
by searching the web and analyzing information.

Guidelines:
1. Always verify facts by searching
2. Cite your sources when possible
3. If search results are insufficient, try different search queries
4. For calculations, use the calculate tool, don't guess
5. When you have enough information, provide a clear, concise answer
6. If you can't find reliable information, say so honestly""",
        tools=tools,
        model="gpt-4o-mini",
        max_iterations=15,
    )
    
    result = agent.run(
        "What companies developed GPT-4o, Claude 3.5 Sonnet, and Gemini 1.5 Pro? "
        "Which one has the largest context window?"
    )
    
    print(f"\n{'='*60}")
    print("FINAL ANSWER:")
    print(f"{'='*60}")
    print(result)
```

**Expected output:**
```
════════════════════════════════════════════════════════
🎯 Goal: What companies developed GPT-4o, Claude 3.5 Sonnet, and Gemini 1.5 Pro? Which one has the largest context window?
════════════════════════════════════════════════════════

📋 PHASE 1: PLANNING
────────────────────────────────────────
Strategy: ...
Plan has 4 steps. Executing...

⚙️ PHASE 2: EXECUTION
────────────────────────────────────────

[Iteration 1/15]
  🔧 search_web({"query": "GPT-4o developer company"})
  📥 Result: GPT-4o is developed by OpenAI...

[Iteration 2/15]
  🔧 search_web({"query": "Claude 3.5 Sonnet developer company"})
  📥 Result: Claude 3.5 Sonnet is developed by Anthropic...

[Iteration 3/15]
  🔧 search_web({"query": "Gemini 1.5 Pro developer company"})
  📥 Result: Gemini 1.5 Pro is developed by Google DeepMind...

[Iteration 4/15]
  🔧 search_web({"query": "GPT-4o vs Claude 3.5 vs Gemini 1.5 context window comparison"})
  📥 Result: Gemini 1.5 Pro has 1M tokens, Claude 3.5 has 200K, GPT-4o has 128K...

════════════════════════════════════════════════════════
FINAL ANSWER:
════════════════════════════════════════════════════════
- GPT-4o: Developed by OpenAI (context window: 128K tokens)
- Claude 3.5 Sonnet: Developed by Anthropic (context window: 200K tokens)
- Gemini 1.5 Pro: Developed by Google DeepMind (context window: 1M tokens)

**Winner: Gemini 1.5 Pro** with a 1M token context window — nearly 5x larger than Claude 3.5 and 8x larger than GPT-4o.
```

---

## ✅ Good Output Examples

### What a Well-Designed Agent Trace Looks Like

```
Goal: "How many days until the next total solar eclipse, and where can I see it?"

Agent output:
─────────────────────────────────────────

📋 PLAN:
  Search for next total solar eclipse date and location
  Calculate days from today
  Fetch detailed viewing guide
  Synthesize final answer

📊 EXECUTION:
  [1/3] search_web("next total solar eclipse date 2025 2026")
        → Results: Next total solar eclipse is on August 12, 2026
  [2/3] calculate("(date(2026,8,12) - date.today()).days")
        → 453 days from today
  [3/3] fetch_page("https://eclipse.nasa.gov/2026-path")
        → Visible from: Spain, Iceland, Greenland, Russia

✅ FINAL ANSWER:
  The next total solar eclipse is on August 12, 2026 (453 days from today).
  Path of totality: Spain → Iceland → Greenland → Russia.
  Best viewing: Northern Spain (longest duration: 2m18s).

  Source: NASA Eclipse Web Site, national eclipse calculations.

KEY METRICS:
  Total LLM calls: 4 (1 plan + 3 execute)
  Cost (gpt-4o-mini): ~$0.002
  Latency: ~8 seconds
  Verification: ✔ All sources cited
```

### What a POOR Agent Trace Looks Like

```
Goal: "How many days until the next total solar eclipse?"

Agent output:
─────────────────────────────────────────

📋 PLAN:
  search_web("eclipse date")
  fetch_page("some astronomy site")
  Write answer

📊 EXECUTION:
  [1/2] search_web("eclipse date")
        → "Solar eclipse 2024 was in April"
  [2/2] fetch_page("random astronomy blog")
        → Page about lunar eclipses (wrong!)
  
❌ FINAL ANSWER:
  The next total solar eclipse was on April 8, 2024.
  (WRONG! The agent didn't specify "next" vs "recent")
  (WRONG! The agent didn't verify with a second source)
  (WRONG! No citation for which eclipse it found)

ROOT CAUSE:
  ✗ Ambiguous query — "eclipse date" returned the most RECENT, not the NEXT
  ✗ No cross-referencing — accepted first result without verification
  ✗ Poor planning — didn't include "calculate days from today" step
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: No Iteration Limit

```python
# ❌ WRONG — No safety limit
agent = Agent(tools=tools)
while True:  # INFINITE LOOP RISK!
    # Agent can loop forever, costing $$$$

# ✅ RIGHT — Always set a max
agent = Agent(tools=tools, max_iterations=10)
# Hard limit prevents runaway costs
```

### Antipattern 2: No Error Recovery

```python
# ❌ WRONG — Tool failure kills the agent
def execute_tool(name, args):
    return tools[name](**args)  # Crash on error → whole agent fails

# ✅ RIGHT — Graceful error handling
def execute_tool(name, args):
    try:
        return tools[name](**args)
    except Exception as e:
        return f"⚠️ Tool {name} failed: {str(e)}\nPlease try a different approach."
        # The LLM can see the error and adjust
```

### Antipattern 3: Plan Bloat

```python
# ❌ WRONG — 20-step plan for a simple question
plan = [
    "Search for X",
    "Read first result",
    "Extract key points",
    "Check second result",
    "Cross-reference...",
    "Write summary...",
    "Verify facts...",
    # ... 13 more steps
]

# ✅ RIGHT — Start simple, the LLM will expand if needed
plan = [
    "Research X with 2-3 searches",
    "Synthesize findings",
]
# Simple plans are more robust. Over-planning = over-fitting.
```

### Antipattern 4: Tools That Return Too Much Data

```python
# ❌ WRONG — Returns entire web page (50K tokens)
def fetch_page(url):
    return httpx.get(url).text  # Full HTML — huge, noisy

# ✅ RIGHT — Extract what matters
def fetch_page(url):
    text = extract_meaningful_content(httpx.get(url).text)
    return text[:3000]  # Limit to what the LLM can process
    # Saves tokens, focuses the LLM on relevant content
```

### Failure Mode: The Tool Hallucination

```
The LLM invents a tool call that doesn't exist:

Agent: "Let me search for competitor pricing"
Agent calls: search_databse(query="competitor_pricing")
            ^^^^^^^^^^^^^^^^
            Typo! Tool doesn't exist!

System: "Error: Unknown tool 'search_databse'. Available: search_web, calculate, fetch_page"

Agent: "Oops, let me use search_web instead."
```

**The fix:** Every tool call must be validated against the actual tool list. When a tool call fails (wrong name, invalid args), the error message must include the **correct alternatives** so the LLM can self-correct.

### Failure Mode: The "I'm Done" Lie

```
Agent calls search_web 3 times, gets partial results
Agent says: "I have enough information"
But actually: It didn't find the key data point, it just got TIRED

This happens because:
1. The LLM is trained to be helpful — it wants to give an answer
2. It doesn't "know" what it doesn't know
3. It might prefer a partial answer over "I can't find this"
```

**The fix:** In the system prompt, explicitly reward honesty:
```
"DO NOT give partial answers. If you don't have sufficient information after
searching, say 'I could not find sufficient information about X.' 
It is better to be honest about gaps than to guess."
```

### Failure Mode: Context Window Saturation

```
After 8 iterations of "think → act → observe," the conversation history contains:
- System prompt: 500 tokens
- Planning output: 800 tokens
- 8 tool call messages: 1600 tokens  
- 8 tool results: average 500 each = 4000 tokens
- Total: ~7000 tokens

After 20 iterations: ~17000 tokens
After 50 iterations: ~42000 tokens

Problem: Cost increases, and the LLM's attention to early context degrades.
```

**The fix:**
- Summarize past iterations into a condensed history
- Or use "forget oldest" sliding window on the conversation
- Or cap iterations low (10-15) and force the agent to be efficient

---

## 🧪 Drills & Challenges

### Drill 1: Build an Agent with Custom Tools (25 min)

Create a **Weather & Travel Agent** with these tools:

```python
def get_weather(city: str) -> str:
    """Get current weather."""

def search_flights(from_city: str, to_city: str, date: str) -> str:
    """Search for flights (mock this — return fake data)."""

def convert_currency(amount: float, from_curr: str, to_curr: str) -> str:
    """Convert currency using exchangerate-api.com."""
```

Then test it with: *"Plan a trip to Tokyo next week. Check the weather, find flights, and estimate costs in USD."*

**Challenge:** What happens when one tool fails (e.g., flight search returns no results)? Does your agent adapt or crash?

### Drill 2: The "Stupid Agent" Test (15 min)

Give your agent this tool:

```python
def send_email(to: str, subject: str, body: str) -> str:
    """Sends an email."""
    print(f"⚠️ EMAIL WOULD BE SENT: to={to}, subject={subject}")
    return f"Email sent to {to}"
```

Then ask: *"Send an email to my team about the project status"*

Did the agent:
- Make up email addresses?
- Ask you for the recipients?
- Send to dangerous-looking addresses?
- Check before sending?

**This is the Tool Safety Problem.** We'll fix it properly in File 03.

### Drill 3: The Planning Experiment (20 min)

Test the same goal with different planning approaches:

```python
# Approach A: No plan (ReAct — think at each step)
agent_a = Agent(system_prompt="You can use tools. Figure out each step as you go.")

# Approach B: Full plan first
agent_b = Agent(system_prompt="Before using any tools, create a complete plan with all steps. Then execute.")

# Approach C: Minimal plan
agent_c = Agent(system_prompt="Briefly outline your approach (1-2 sentences), then execute.")
```

Test with: *"What's the population of India and how does it compare to China's? What was the growth rate for both in the last decade?"*

**Document:** Which approach was fastest? Most accurate? Most costly?

### Drill 4: The Cost Calculator (15 min)

Calculate the cost of running your agent at scale:

```
Setup:
- Model: gpt-4o-mini ($0.15/1M input, $0.60/1M output)
- Average iterations per task: 5
- Average input per iteration: 1500 tokens
- Average output per iteration: 200 tokens
- Queries per day: 1000

Calculate:
1. Cost per query = ?
2. Cost per day = ?
3. Cost per month = ?
4. Cost per year = ?

Now switch to gpt-4o ($2.50/1M input, $10/1M output):
5. How much more expensive?
6. When is the quality gain worth the cost?
```

### Drill 5: Debug the Broken Agent (20 min)

Here's an agent with 3 bugs. Find and fix them:

```python
class BrokenAgent:
    def __init__(self, tools):
        self.tools = tools
        self.messages = []
        self.client = OpenAI()
    
    def run(self, goal):
        self.messages.append({"role": "user", "content": goal})
        
        # BUG 1: No iteration limit — infinite loop risk!
        while True:
            response = self.client.chat.completions.create(
                model="gpt-4o-mini",
                messages=self.messages,  # BUG 2: No system prompt!
                tools=[],  # BUG 3: Empty tools list — agent can't call tools!
            )
            msg = response.choices[0].message
            if not msg.tool_calls:
                return msg.content
            # BUG 4: Tool results are never added back to messages!
```

---

## 🚦 Gate Check

Before moving to File 02 (ReAct Loop), confirm you can:

- [ ] **Explain the agent loop** in 2 sentences: Think → Act → Observe → Repeat
- [ ] **Build a zero-shot agent from scratch** that plans before executing
- [ ] **Implement 3 different tool types** (search, calculation, data fetch)
- [ ] **Handle tool errors gracefully** — the agent recovers and tries alternative approaches
- [ ] **Set max iteration limits** and explain why they're necessary
- [ ] **Answer these questions:**
  1. An agent costs $0.05 per run. You have 10,000 users/day. What's your monthly cost?
  2. Your agent searches "latest news" and gets "No results." What should it do? (A. Give up, B. Try a different query, C. Report an error)
  3. Your agent plans 15 steps but iteration 3 fails. Should it: A. Stop the whole plan, B. Skip step 3, C. Re-plan from step 3? Why?
  4. What happens if the LLM hallucinates a tool call with `send_email(to="all@company.com", subject="URGENT: Salary corrections")` — how do you prevent this?
  5. When would you use plan-then-execute vs. step-by-step (ReAct)? Give a concrete example for each.

- [ ] **Write down the #1 thing** you want to learn from File 02 (ReAct Loop)

---

## 📚 Resources

**Foundational:**
- "ReAct: Synergizing Reasoning and Acting in Language Models" (Yao et al., 2022) — The paper that defined the pattern
- "LLM Powered Autonomous Agents" — Lilian Weng's comprehensive survey
- "Building Agents with LLMs" — Google's guide to agent architecture

**Code References:**
- OpenAI Function Calling documentation
- Pydantic AI — The Python framework we prefer for agents
- LangChain Agent Types — Reference for different agent architectures

**Production Reading:**
- "The Practical Guide to Building AI Agents" — Engineering blog post
- "Agent Design Patterns" — Anthropic's guide to agent architecture
- "Cost of Agent Loops" — Analysis of agent economics at scale

**Your Agent Progress Checklist:**
```
☐ Agent foundations understood (this file)
☐ Zero-shot agent built and tested
☐ At least 3 tool types implemented
☐ Tool error handling works
☐ Iteration limits enforced
☐ Cost model understood
☐ Failure modes documented
```

---

> **🛑 Revisit the 3 discovery questions from the beginning:**
>
> 1. **Before You Read the Definition** — You've now built an agent that can plan and execute multi-step tasks. How close was your mental model of the agent loop?
>
> 2. **The Tool Problem** — You've seen how tools bridge LLM reasoning and real-world action. Did you spot the safety issues we haven't solved yet?
>
> 3. **The Planning Trap** — Our zero-shot agent plans once and executes. What happens when the plan is wrong? (File 02 fixes this.)
>
> **Key realization:** Simple agents plan ONCE and hope the plan survives contact with reality. This works for predictable tasks but breaks for research and analysis. The ReAct pattern (File 02) fixes this by letting the agent re-think after every action.
>
> **Next:** File 02 — The ReAct Loop. How agents reason and act in an interleaved cycle.

---

> **Senior engineer closing thought:**
>
> *"The agent you just built will work great for ~60% of tasks. It will fail spectacularly for the other 40% — not because the code is wrong, but because the world doesn't follow your plan. The next file teaches you to build agents that adapt when reality doesn't match the plan. This is what separates toy agents from production agents."*
