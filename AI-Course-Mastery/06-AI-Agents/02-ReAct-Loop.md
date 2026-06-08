# 02 — The ReAct Loop: Thinking and Acting in Cycles

## 🎯 Purpose & Goals

### 🛑 STOP. Don't read anything yet. Think about this.

**The Scenario That Broke Your File 01 Agent**

Your Zero-Shot agent from the last module works great for simple tasks. Now give it this:

> *"Research the impact of the 2024 US presidential election results on AI regulation. Find specific policy changes, industry responses, and projected impact on AI companies."*

Your agent creates a beautiful plan:
```
1. search_web("2024 US election AI regulation impact")
2. search_web("AI policy changes 2025 after election")
3. fetch_page([top result for industry responses])
4. Synthesize findings
```

Then it executes Step 1. The search returns:

> *"Tech industry braces for AI regulation changes after 2024 election"* — a speculative article from before the election.

Now here's the problem: Your agent has a plan. The plan says Step 2 comes next. It doesn't know that Step 1's result is stale. It doesn't know it should search for *post-election* results specifically. It just follows the plan.

**By Step 4, it produces an analysis based entirely on pre-election speculation. The whole report is wrong.**

---

### 🤔 Your First Real Challenge

Before you read the solution — think:

**Question 1: What's the fundamental flaw in the plan-then-execute agent?**

Look at the trace again. The agent:
1. Made a plan before knowing what the search would reveal
2. Executed the plan step by step
3. Never re-evaluated whether the plan was still correct
4. Never noticed that a search result was stale/unreliable
5. Never adjusted its approach based on what it found

**What would YOU change about the loop to fix this?**

> Think about this for 2 minutes. Literally. Pause reading and think.
>
> **Hint:** What if the agent didn't just "act → observe" but also "thought" between each action?
>
> What if every action was followed by a reasoning step: *"That result was disappointing. I need to try a different approach. What if I search for..."*?

---

### 🤔 Question 2: The Contradiction Problem

Your agent runs this research. It finds two conflicting results:

- Source A: "New AI regulation bill passed in March 2025 — requires transparency in training data"
- Source B: "AI transparency bill STALLED in committee, industry lobbyists fight back"

**Which is correct? How does the agent decide?**

```
Senior engineer: "This is the real test of an agent. Anyone can write code that searches 
and summarizes. The hard part is handling CONFLICTING information. In production, 
you'll get contradictory data constantly — different sources, different timelines, 
different biases. Your agent needs to detect conflicts and resolve them."

Junior engineer: "Should I just pick the more recent source?"

Senior engineer: "That's one strategy. But what if the more recent article is a 
less reliable source? What if both are from the same day? What if one cites data 
and the other doesn't? There's no single rule. The agent needs to REASON about 
which to trust. That's why the 'Think' step matters."
```

---

### 🤔 Question 3: The Stopping Problem

Your agent is researching. It searches, reads, searches more, reads more. At some point, it needs to STOP and deliver an answer.

But how does it decide?

- After 3 searches? What if the 4th search would have found the key insight?
- After 10 minutes? What if the answer is in the very next result?
- When it feels "confident"? LLMs don't actually feel confidence.

**In production, this is the #1 agent failure mode.** Agents either:
- Stop too early (shallow, incomplete answers)
- Loop forever (chasing perfect information that doesn't exist)

**Your challenge before reading on:** Design a stopping criterion. Not a hand-wavy "when it feels done" — an actual, implementable rule or set of rules that determines when an agent should stop searching and answer.

---

> **Keep these 3 questions in your head. The entire ReAct pattern is designed to solve them.**
>
> By the end of this file, you'll have implemented solutions to ALL three.

**By the end of this file, you will:**
1. Understand the **ReAct pattern** — the #1 most important agent architecture
2. Build a ReAct agent from scratch that interleaves reasoning with action
3. Implement **dynamic re-planning** — the agent changes its approach based on results
4. Build **contradiction detection** — the agent identifies and resolves conflicting information
5. Implement **stopping criteria** — the agent knows when it has enough information
6. See exactly why ReAct is the default choice for production agents

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 20 min |
| Why Plan-Then-Execute Fails | 20 min |
| The ReAct Pattern — Thinking Out Loud | 30 min |
| Code: Building a ReAct Agent from Scratch | 75 min |
| Code: Adding Dynamic Re-Planning | 45 min |
| Code: Contradiction Detection & Resolution | 30 min |
| Code: Stopping Criteria | 25 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 20 min |
| Drills & Challenges | 45 min |
| Gate Check | 10 min |
| **Total** | **~5.5 hours** |

---

## 📖 Concept Deep-Dive

### 1. Why Plan-Then-Execute Fails in Production

Let's be precise about the failure. It's not that planning is bad. The problem is **when** the planning happens.

```
Plan-Then-Execute Flow:
┌────────────────────────────────────────────────────┐
│ PLAN PHASE (happens once)                          │
│   "I will: Search → Fetch → Analyze → Write"      │
│   ⚠️ Made WITHOUT knowing what searches return    │
└────────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────┐
│ EXECUTION PHASE (happens blindly)                  │
│   Step 1: Search → "Pre-election speculation"     │
│   Step 2: Fetch → "Industry lobbying report"       │
│   Step 3: Analyze → "Based on steps 1-2..."        │
│   ⚠️ Never re-evaluates the plan                  │
└────────────────────────────────────────────────────┘
```

The plan is a FROZEN document. The world changes when you execute. Your plan doesn't.

**The fix is obvious once you see it:** What if the agent re-plans after EVERY action? What if it thinks, then acts, then thinks again before the next action?

This is the **ReAct** pattern.

### 2. The ReAct Pattern — Reason + Act

**ReAct** = **Rea**soning + **Act**ing

The paper that introduced it (Yao et al., 2022) showed something simple but profound:

> When an LLM alternates between generating reasoning traces (thinking out loud) and taking actions (calling tools), it performs dramatically better than doing either alone.

```
ReAct Flow:
┌────────────────────────────────────────────────────┐
│ THINK: "I need to understand X. Let me start       │
│         by searching for background info."         │
├────────────────────────────────────────────────────┤
│ ACT: search_web("X background")                    │
├────────────────────────────────────────────────────┤
│ OBSERVE: "Results show X was founded in 2020..."   │
├────────────────────────────────────────────────────┤
│ THINK: "Founded in 2020 — that's recent. I need   │
│         to check their funding. Let me also        │
│         check competitors founded around then."    │
├────────────────────────────────────────────────────┤
│ ACT: search_web("X funding rounds"),               │
│      search_web("competitors similar to X 2020")  │
├────────────────────────────────────────────────────┤
│ OBSERVE: "X raised $50M Series B..."               │
├────────────────────────────────────────────────────┤
│ THINK: "Good, I have background and funding.       │
│         Now I need to check their products.        │
│         But wait — should I also verify the        │
│         funding info? Let me cross-check."         │
├────────────────────────────────────────────────────┤
│ ACT: fetch_page("Crunchbase article on X Series B")│
├────────────────────────────────────────────────────┤
│ OBSERVE: "$50M confirmed, led by Sequoia..."       │
├────────────────────────────────────────────────────┤
│ THINK: "I have enough verified information to     │
│         compile the report. ✅ Verified funding.    │
│         ✅ Have company background.                 │
│         Need product details. One more search."    │
├────────────────────────────────────────────────────┤
│ ...                                                │
└────────────────────────────────────────────────────┘
```

The key difference: **The agent REASONS after every observation.** It doesn't follow a frozen plan — it dynamically responds to what it finds.

### 3. The Anatomy of a ReAct Step

Each ReAct cycle has 3 parts:

```
THOUGHT
├── What did I just learn? (analysis of observation)
├── What does this mean for my goal? (relevance)
├── What should I do next? (decision)
├── What information am I still missing? (gap analysis)
└── Should I stop? (completeness check)

ACTION
├── Which tool to call
├── What arguments to pass
└── Why this tool (linked to the thought)

OBSERVATION
├── Raw result from the tool
├── Success/failure of the action
└── Key data points extracted
```

**What makes ReAct powerful:**
1. **Transparency** — every decision is explained in natural language
2. **Debuggability** — you can read the agent's reasoning and find where it went wrong
3. **Adaptability** — the agent changes course based on results
4. **Self-correction** — if an action fails, the agent reasons about why and tries differently

### 4. The Thinking Instruction — Why It Matters

Here's a critical insight most tutorials skip:

The LLM needs to be EXPLICITLY told to think out loud. Without this instruction, it tends to:
- Jump to conclusions (no reasoning trace)
- Make tool calls without explaining why
- Give final answers without showing work

**Compare these system prompts:**

```python
# ❌ WEAK — Agent doesn't think out loud
system_prompt = "You are a helpful assistant with access to tools."

# ✅ STRONG — Agent reasons step by step
system_prompt = """You are a research assistant. For every step:
1. THINK: Explain what you know, what you need, and why
2. ACT: Call a tool if you need more information
3. OBSERVE: Analyze the tool result
4. REPEAT until you have enough information

CRITICAL: Your thinking is NOT just for show. Actually reason about:
- What did this result tell me?
- Is this information reliable?
- What am I still missing?
- Should I try a different approach?
- Do I have enough to answer?
"""
```

**From Phase 2, you learned Chain-of-Thought prompting makes LLMs reason better.** ReAct is Chain-of-Thought for actions — the LLM doesn't just think, it acts on its thinking.

> **Cross-reference: Phase 2 (Prompt Engineering)**
>
> Remember CoT prompting? You told the model to "think step by step" before answering. The result was dramatically better reasoning.
>
> ReAct is the same principle, but extended: "Think step by step, AND take an action after each thought, AND observe the result, AND think again."
>
> The insight from Phase 2 applies here too: **The thinking isn't just for show.** When the LLM articulates its reasoning, it actually reasons better. The act of verbalizing creates logical structure.

---

## 💻 Code Examples

### 🔧 The Build Loop

I'm going to show you a BROKEN ReAct agent first. Your job: look at it, find the bugs, and understand WHY it fails. Then I'll show you the fixed version.

#### Example 1: The "Almost Right" ReAct Agent (with 4 deliberate bugs)

Read this carefully. It looks correct. But it has 4 bugs that make it fail in production:

```python
"""
ReAct Agent v1 — looks right, works wrong.

Read this code. Find the 4 bugs BEFORE running it.
Think about what happens at each step.
"""

import os
import json
from openai import OpenAI
from typing import Callable, Optional


class Tool:
    def __init__(self, name: str, description: str, parameters: dict, function: Callable):
        self.name = name
        self.description = description
        self.parameters = parameters
        self.function = function


class BuggyReActAgent:
    """
    This agent has 4 deliberate bugs.
    
    🐛 BUG 1: No system prompt awareness — the agent doesn't know it should 
              think out loud, so it just calls tools without reasoning
    
    🐛 BUG 2: Tool results are NOT added back to messages — the agent 
              executes, but never sees the results
    
    🐛 BUG 3: No stopping criterion — the agent keeps looping forever
    
    🐛 BUG 4: Error handling throws exceptions instead of returning error 
              messages the LLM can learn from
    """
    
    def __init__(self, tools: list[Tool], model: str = "gpt-4o-mini"):
        self.tools = {t.name: t for t in tools}
        self.model = model
        self.client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
        self.messages = []
        self.max_iterations = 10
    
    def _tool_schemas(self):
        return [
            {
                "type": "function",
                "function": {
                    "name": t.name,
                    "description": t.description,
                    "parameters": t.parameters,
                }
            }
            for t in self.tools.values()
        ]
    
    def run(self, user_input: str) -> str:
        # 🐛 BUG 1: No system prompt! The agent doesn't know
        # to think out loud. It will call tools without reasoning.
        self.messages.append({"role": "user", "content": user_input})
        
        iteration = 0
        while iteration < self.max_iterations:
            iteration += 1
            print(f"\n[Iteration {iteration}]")
            
            response = self.client.chat.completions.create(
                model=self.model,
                messages=self.messages,
                tools=self._tool_schemas(),
                tool_choice="auto",
                temperature=0.0,
            )
            
            message = response.choices[0].message
            
            if not message.tool_calls:
                return message.content
            
            for tool_call in message.tool_calls:
                tool_name = tool_call.function.name
                tool_args = json.loads(tool_call.function.arguments)
                
                print(f"  🔧 Calling {tool_name}({tool_args})")
                
                # 🐛 BUG 2: The tool result is NOT added back!
                # The agent calls the tool but never sees the result.
                # Next iteration, it has no context about what happened.
                result = self.tools[tool_name].function(**tool_args)
                print(f"  📥 Got: {str(result)[:100]}...")
                # ← RESULT IS DROPPED HERE!
                
                # 🐛 BUG 4: No error handling wrapper
                # If the tool throws, the WHOLE AGENT crashes
        
        # 🐛 BUG 3: No "why did I stop?" context
        return "Max iterations reached."
```

---

### 🤔 Your Turn: Find the 4 Bugs

Before scrolling down — actually write down:

1. **Bug 1**: What happens without a system prompt?
2. **Bug 2**: What happens when tool results aren't fed back?
3. **Bug 3**: What's wrong with the stopping behavior?
4. **Bug 4**: Why is no error handling dangerous?

> **Think for 3 minutes. Actually write your answers.**
>
> Then compare with my analysis below.

---

### Debug Analysis

**Bug 1 — No system prompt:**
The agent gets no instruction to "think out loud." Without this, the LLM tends to call tools directly without reasoning. You get:
```
[Iteration 1] 🔧 search_web({"query": "..."}) 
[Iteration 2] 🔧 search_web({"query": "..."}) 
[Iteration 3] 🔧 search_web({"query": "..."}) 
[Iteration 4] "Here's the answer."
```
No thoughts. No reasoning. Just blind tool calls. The LLM doesn't explain WHY it's searching for what it's searching for.

**Bug 2 — Results not fed back:**
This is the KILLER bug. The agent calls `search_web("AI regulation 2025")`, gets results, but NEVER sees them. Next iteration, the agent sees:
```
User: Research AI regulation
Assistant: [tool call — search_web]
Assistant: [tool call — search_web] (again!)
```
It has NO memory of previous results. It searches the same thing again. Or it searches something unrelated. It's flying blind.

**Bug 3 — No stopping context:**
When max iterations hit, the response is just "Max iterations reached." No partial results. No explanation of what was found. The user gets nothing useful. A better approach: return whatever was gathered so far.

**Bug 4 — No error handling:**
If `search_web` throws a network error, the whole agent crashes. In production, API calls fail constantly. The agent should handle errors gracefully and try alternatives.

---

### Example 2: The Fixed ReAct Agent

Now let's build the CORRECT version:

```python
"""
Correct ReAct Agent — thinks out loud, learns from results, handles errors.
"""

import os
import json
import time
from typing import Callable, Optional, Any
from dataclasses import dataclass, field
from openai import OpenAI


@dataclass
class Tool:
    name: str
    description: str
    parameters: dict
    function: Callable


@dataclass
class ReActStep:
    """Records a single ReAct cycle for traceability."""
    iteration: int
    thought: str
    action: Optional[tuple[str, dict]] = None  # (tool_name, arguments)
    observation: Optional[str] = None
    is_final: bool = False
    answer: Optional[str] = None
    error: Optional[str] = None


class ReActAgent:
    """
    The correct ReAct agent implementation.
    
    Three things make this work:
    1. SYSTEM PROMPT: Explicitly tells the LLM to think out loud
    2. RESULT FEEDBACK: Every tool result goes back into the conversation
    3. STOPPING CRITERIA: Clear rules for when to stop
    
    This is the PATTERN. Not a library, not a framework — the actual pattern
    that frameworks like LangGraph implement under the hood.
    """
    
    def __init__(
        self,
        tools: list[Tool],
        system_prompt: Optional[str] = None,
        model: str = "gpt-4o-mini",
        max_iterations: int = 15,
    ):
        self.tools = {t.name: t for t in tools}
        self.model = model
        self.max_iterations = max_iterations
        self.client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
        
        # Default system prompt with ReAct instructions
        self.system_prompt = system_prompt or """You are a research assistant that solves problems through iterative reasoning and tool use.

For every step, follow this pattern:

1. THINK: Analyze what you know and what you need.
   - What did I just learn from the last result?
   - What information am I still missing?
   - What's the best next step?
   - Should I try a different approach?

2. ACT: Call a tool if you need more information.
   - Choose the right tool for what you need
   - Be specific with your arguments
   
3. OBSERVE: Process the result.
   - What does this tell me?
   - Is this information reliable?
   - Does it contradict anything I found earlier?

CRITICAL RULES:
- You MUST think before acting. Never call tools without reasoning.
- If you find contradictory information, investigate before deciding.
- When you have sufficient verified information, provide your answer.
- If a tool fails, try an alternative approach.
- Cite your sources when possible.
- Be honest about uncertainty.

STOP when:
- You have verified answers to all aspects of the question
- You've tried 3+ approaches and can't find the information
- The user's question is fully addressed"""
        
        self.messages = [{"role": "system", "content": self.system_prompt}]
        self.steps: list[ReActStep] = []
        self.total_cost: float = 0.0
    
    def _tool_schemas(self) -> list[dict]:
        return [
            {
                "type": "function",
                "function": {
                    "name": t.name,
                    "description": t.description,
                    "parameters": t.parameters,
                }
            }
            for t in self.tools.values()
        ]
    
    def _llm_call(self, messages: list[dict]) -> Any:
        """Make an LLM call with full tool access."""
        response = self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            tools=self._tool_schemas(),
            tool_choice="auto",
            temperature=0.0,
        )
        
        # Track cost
        in_tokens = response.usage.prompt_tokens
        out_tokens = response.usage.completion_tokens
        self.total_cost += (in_tokens * 0.15 / 1_000_000) + (out_tokens * 0.60 / 1_000_000)
        
        return response.choices[0].message
    
    def _execute_tool(self, tool_name: str, arguments: dict) -> str:
        """
        Execute tool with comprehensive error handling.
        
        The KEY insight: we return errors AS RESULTS, not exceptions.
        This lets the LLM see the error and decide what to do.
        """
        if tool_name not in self.tools:
            return f"⚠️ Tool '{tool_name}' not found. Available: {list(self.tools.keys())}"
        
        tool = self.tools[tool_name]
        
        try:
            result = tool.function(**arguments)
            if isinstance(result, (dict, list)):
                return json.dumps(result, indent=2, default=str)
            return str(result)
        except TypeError as e:
            return f"⚠️ Invalid arguments for {tool_name}: {str(e)}"
        except Exception as e:
            return f"⚠️ Tool {tool_name} failed: {str(e)}"
    
    def run(self, user_input: str, verbose: bool = True) -> str:
        """Run the ReAct agent loop."""
        self.messages.append({"role": "user", "content": user_input})
        self.steps = []
        
        if verbose:
            print(f"\n{'='*60}")
            print(f"🎯 {user_input}")
            print(f"{'='*60}\n")
        
        iteration = 0
        while iteration < self.max_iterations:
            iteration += 1
            
            if verbose:
                print(f"\n{'─'*40}")
                print(f"  Iteration {iteration}")
                print(f"{'─'*40}")
            
            # ── THINK + ACT ──
            message = self._llm_call(self.messages)
            
            # Record thought (the message content IS the thought)
            thought = message.content or "(no explicit thought)"
            if verbose:
                print(f"\n💭 {thought}")
            
            # ── CHECK: Is agent done? ──
            if not message.tool_calls:
                step = ReActStep(
                    iteration=iteration,
                    thought=thought,
                    is_final=True,
                    answer=thought,
                )
                self.steps.append(step)
                
                if verbose:
                    print(f"\n✅ Task complete after {iteration} iterations")
                    print(f"💰 Cost: ~${self.total_cost:.6f}")
                
                # Summarize what happened
                if verbose:
                    print(f"\n📊 Summary:")
                    print(f"   Total iterations: {iteration}")
                    print(f"   Tools used: {len([s for s in self.steps if s.action])}")
                    print(f"   Final cost: ${self.total_cost:.6f}")
                
                return thought
            
            # ── EXECUTE TOOLS ──
            for tool_call in message.tool_calls:
                tool_name = tool_call.function.name
                tool_args = json.loads(tool_call.function.arguments)
                
                if verbose:
                    print(f"  \n🔧 {tool_name}({json.dumps(tool_args)})")
                
                result = self._execute_tool(tool_name, tool_args)
                
                if verbose:
                    result_preview = result[:300] + "..." if len(result) > 300 else result
                    print(f"  📥 {result_preview}")
                
                # ── CRITICAL: Feed result back ──
                # This is what makes ReAct work — the LLM sees every result
                self.messages.append(message)
                self.messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result,
                })
                
                # Record step
                self.steps.append(ReActStep(
                    iteration=iteration,
                    thought=thought,
                    action=(tool_name, tool_args),
                    observation=result[:500],  # Truncated for logging
                ))
        
        # ── MAX ITERATIONS — Return partial results ──
        summary = self._get_partial_summary()
        msg = f"⚠️ Reached max iterations ({self.max_iterations}).\n\nPartial findings so far:\n{summary}"
        if verbose:
            print(f"\n⚠️ Max iterations reached")
        return msg
    
    def _get_partial_summary(self) -> str:
        """Extract key findings from completed steps."""
        findings = []
        for step in self.steps:
            if step.observation:
                findings.append(f"- {step.observation[:200]}")
        return "\n".join(findings[:5])  # Top 5 observations
```

---

### 🤔 Question 4: Design Your Tools (Pause and Think)

The ReAct agent above works with any tools. Before you run it, think about this:

You need tools for the research task: *"Compare GPT-4o, Claude 3.5 Sonnet, and Gemini 1.5 Pro on: context window size, pricing, multimodal support, and release date."*

**Don't look at my tools below. First, design your own:**

What tool or tools would you give the agent? Consider:
- Do you need one flexible search tool, or multiple specialized tools?
- What parameters does each tool need?
- How will the agent know WHICH tool to use?
- How do you prevent the agent from searching too broadly?

> Write down your tool design. Then compare with mine below.

---

### My Tool Design for the Research Agent

```python
# ── Tools for the comparative research task ──

def search_web(query: str, max_results: int = 5) -> str:
    """
    Search the web for information.
    
    Parameters:
      query: The search query
      max_results: Number of results to return (1-10)
    """
    # ... implementation (same as File 01)
    return results


def fetch_page(url: str, max_chars: int = 3000) -> str:
    """
    Fetch and extract text content from a URL.
    
    Parameters:
      url: The URL to fetch
      max_chars: Maximum characters to return
    """
    # ... implementation
    return content


def extract_comparison_data(topic: str, items: list[str], attributes: list[str]) -> str:
    """
    Search for and extract comparison data across multiple items.
    
    This is a HIGHER-LEVEL tool that does multiple searches internally.
    It saves the agent from making N×M individual calls.
    """
    results = {}
    for item in items:
        item_data = {}
        for attr in attributes:
            r = search_web(f"{item} {attr} 2025")
            item_data[attr] = extract_value(r)
        results[item] = item_data
    return json.dumps(results, indent=2)


tools = [
    Tool(
        name="search_web",
        description="Search the web for current information. Use for finding facts, news, prices, and updates.",
        parameters={
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "The search query (be specific)"
                },
                "max_results": {
                    "type": "integer",
                    "description": "Number of results (default 5, max 10)",
                    "default": 5
                }
            },
            "required": ["query"]
        },
        function=search_web
    ),
    Tool(
        name="fetch_page",
        description="Fetch the text content of a URL. Use after search_web to read full articles.",
        parameters={
            "type": "object",
            "properties": {
                "url": {
                    "type": "string",
                    "description": "The full URL to fetch"
                }
            },
            "required": ["url"]
        },
        function=fetch_page
    ),
    Tool(
        name="extract_comparison_data",
        description="Compare multiple items across given attributes. Use for research comparisons.",
        parameters={
            "type": "object",
            "properties": {
                "topic": {"type": "string", "description": "The topic being researched"},
                "items": {"type": "array", "items": {"type": "string"}, "description": "Items to compare"},
                "attributes": {"type": "array", "items": {"type": "string"}, "description": "Attributes to compare"}
            },
            "required": ["topic", "items", "attributes"]
        },
        function=extract_comparison_data
    ),
]
```

**Why this tool design?**
- `search_web` handles initial discovery
- `fetch_page` digs deeper into specific sources
- `extract_comparison_data` is an OPTIMIZED tool for the specific task — it does N×M searches internally, saving agent iterations

**Senior engineer insight:** The best tool design anticipates what the agent needs to do and provides the RIGHT level of abstraction. Too granular (just "search" and "fetch") and the agent wastes iterations. Too high-level (just "write report") and the agent loses flexibility. Design tools at the level of the TASK, not the level of the API.

---

### Example 3: Running the ReAct Agent

```python
if __name__ == "__main__":
    agent = ReActAgent(
        tools=tools,
        model="gpt-4o-mini",
        max_iterations=15,
    )
    
    result = agent.run(
        "Compare GPT-4o, Claude 3.5 Sonnet, and Gemini 1.5 Pro on: "
        "context window size, pricing, and multimodal support. "
        "Which is best for a document analysis application?"
    )
    
    print(f"\n{'='*60}")
    print("FINAL ANSWER:")
    print(f"{'='*60}")
    print(result)
```

**Expected trace (abbreviated):**
```
🎯 Compare GPT-4o, Claude 3.5 Sonnet, and Gemini 1.5 Pro on...

────────────────────────────────────────
  Iteration 1
────────────────────────────────────────

💭 I need to compare these 3 models. Let me start by searching for each 
model's specs. I'll use extract_comparison_data to get all the data at once.

🔧 extract_comparison_data({"topic": "LLM model comparison", 
     "items": ["GPT-4o", "Claude 3.5 Sonnet", "Gemini 1.5 Pro"],
     "attributes": ["context window", "pricing", "multimodal"]})
📥 Results for GPT-4o: 128K context, $2.50/$10 per 1M tokens, multimodal (text+image+audio)
   Results for Claude 3.5 Sonnet: 200K context, $3/$15 per 1M tokens, multimodal (text+image)
   Results for Gemini 1.5 Pro: 1M context, $1.25/$5 per 1M tokens, multimodal (text+image+audio+video)

────────────────────────────────────────
  Iteration 2
────────────────────────────────────────

💭 I have the raw data. Let me verify the most surprising finding — 
Gemini's pricing being the cheapest AND largest context. Let me double-check.

🔧 search_web({"query": "Gemini 1.5 Pro pricing per token 2025"})
📥 "Gemini 1.5 Pro: $1.25 per 1M input tokens, $5.00 per 1M output tokens..."

💭 Confirmed. Now let me also check if there are specific features relevant 
to document analysis — like PDF handling and long-document support.

🔧 search_web({"query": "GPT-4o vs Claude vs Gemini document analysis comparison"})
📥 "Claude excels at long-document analysis due to 200K context..."

────────────────────────────────────────
  Iteration 3
────────────────────────────────────────

💭 I have verified data from multiple sources. All 3 models support 
multimodal document analysis. Let me synthesize the findings.

✅ Final answer synthesizing the comparison...
```

---

## ✅ Good Output Examples

### What a GREAT ReAct Trace Looks Like

```
Goal: "Find the latest funding round for Anthropic and compare to OpenAI's valuation"

AGENT TRACE:
───────────────────────────────────────────────────────

💭 "I need Anthropic's latest funding. Let me start with a search."
🔧 search_web({"query": "Anthropic latest funding round 2025"})
📥 "Anthropic raised $3.5B in Series E at $61.5B valuation (March 2025)"

💭 "$61.5B valuation. That's significant. Let me verify with a second source."
🔧 search_web({"query": "Anthropic Series E 2025 valuation"})
📥 "Confirmed: Anthropic's $3.5B Series E led by Lightspeed, valuation $61.5B"
✅ Sources agree.

💭 "Now I need OpenAI's valuation for comparison. Let me search."
🔧 search_web({"query": "OpenAI latest valuation 2025"})
📥 "OpenAI valued at $300B in SoftBank-led round (February 2025)"

💭 "Significant gap: OpenAI at $300B vs Anthropic at $61.5B. Let me check
    if there are more recent updates for either."
🔧 search_web({"query": "OpenAI valuation 2025 latest funding"})
📥 "Same result: $300B. ✅ Verified."

💭 "I have verified data from multiple sources for both companies. 
    Enough to answer. Let me compile."

✅ FINAL ANSWER:
  Anthropic: $61.5B valuation after $3.5B Series E (March 2025)
  OpenAI: $300B valuation after SoftBank-led round (February 2025)
  Both verified from 2+ sources.

✅ TRACE METRICS:
  Iterations: 5
  Tools used: 4 (web search × 4)
  Verification rate: 100% (all facts double-checked)
  Contradictions found: 0
```

### What a BAD ReAct Trace Looks Like

```
Goal: "Find the latest funding round for Anthropic"

AGENT TRACE:
───────────────────────────────────────────────────────

💭 "Let me search."
🔧 search_web({"query": "Anthropic"})
📥 "Anthropic is an AI safety company..."

💭 "Let me search more."
🔧 search_web({"query": "Anthropic funding"})
📥 "Anthropic raised $750M in 2024..."

💭 "More search."
🔧 search_web({"query": "Anthropic Series D"})
📥 "Anthropic raised $750M Series D..."

💭 "Let me search again."
🔧 search_web({"query": "Anthropic funding round 2025"})
📥 Outcome uncertain...

🔴 PROBLEM:
  ✗ No explicit reasoning — "Let me search" gives no strategy
  ✗ No verification — accepts first result without cross-checking
  ✗ Repetitive searches — different phrasings of the same query
  ✗ No stopping criterion — just keeps searching
  ✗ No analysis — never says "I have enough" or "I need more"

ROOT CAUSE:
  Weak system prompt that didn't enforce "think before act"
  No verification instruction
  No stopping criteria
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: The "Why did it do that?" Agent

When you DON'T have explicit thinking in your trace, debugging becomes impossible:

```python
# ❌ WRONG — No reasoning trace
messages = [
    {"role": "system", "content": "You have tools. Use them."},
    {"role": "user", "content": goal}
]
# Output: search_web({"query": "..."})
# Why this search? What did it expect to find? No idea.

# ✅ RIGHT — Reasoning is mandatory
messages = [
    {"role": "system", "content": """For EVERY step:
    - Explain what you know
    - Explain what you need
    - Explain why this tool call will help
    - Explain what you'll do with the result"""},
    {"role": "user", "content": goal}
]
# Output: "I need to find X. I already know Y. I'm searching for Z..."
```

### Antipattern 2: Tool Call Spam

The agent makes dozens of tiny, similar tool calls instead of batching:

```python
# ❌ WRONG — One search per data point
search_web({"query": "GPT-4o context window"})
search_web({"query": "GPT-4o pricing"})
search_web({"query": "GPT-4o multimodal"})
search_web({"query": "Claude context window"})
# ... Agent does 12 iterations for 3 models × 4 attributes

# ✅ RIGHT — Batch related searches
extract_comparison_data({
    "items": ["GPT-4o", "Claude", "Gemini"],
    "attributes": ["context window", "pricing", "multimodal"]
})
# 1 call instead of 12. Same data.
```

**The fix:** Design tools at the right granularity. If your agent repeatedly makes similar calls, create a batch/higher-level tool.

### Antipattern 3: The Verification Disaster

The agent accepts the FIRST result without checking:

```
Agent: searches "OpenAI revenue 2024"
Result: "$3.7B annualized revenue" (from an unreliable blog)
Agent: "OpenAI's revenue is $3.7B"
Reality: OpenAI's actual 2024 revenue was ~$4.5B+
```

**The fix:** System prompt should include verification rules:
```
"Any factual claim must be verified from 2+ independent sources.
If sources disagree, mention the disagreement and explain which you trust and why.
If you can only find one source, state that explicitly."
```

### Antipattern 4: The Infinite Loop of Despair

The agent keeps searching because it never decides "this is enough":

```
search → "AI regulation bill passed"
search → "AI regulation bill details"  
search → "AI regulation bill text"
search → "AI regulation bill vote count"
search → "AI regulation bill impact"
search → "AI regulation bill analysis"
... (never ending)
```

**The fix:** Implement explicit stopping criteria (see next section).

### Antipattern 5: Ignoring Phase 2 Lessons

Remember from Phase 2: **Temperature matters.** For agent reasoning, temperature=0 is usually correct:

```python
# ❌ WRONG — Agent gets "creative" with tool calls
agent = ReActAgent(temperature=0.7)  
# The LLM might randomly decide to search for unrelated things
# It might hallucinate tool arguments

# ✅ RIGHT — Agent is deterministic
agent = ReActAgent(temperature=0.0)
# Consistent reasoning, consistent tool choices
```

---

## 🧪 Deep Dive: Stopping Criteria (The #1 Production Problem)

Let's solve **Question 3** from the beginning — **when should the agent stop?**

This is NOT a trivial question. It's the most common agent failure in production.

### Strategy 1: Exhaustive Checklist (Good for Factual Questions)

The agent maintains a checklist of what it needs:

```
System prompt addition:
"Before answering, verify you have all of the following:
✓ Entity identification (what/who are we researching?)
✓ Key facts gathered (at least 3 data points)
✓ Verified from 2+ sources (no single-source claims)
✓ All sub-questions addressed
✓ Conflicting information resolved
If anything is missing, continue researching."
```

### Strategy 2: Diminishing Returns (Good for Research)

The agent tracks how valuable each new search is:

```
Search 1: Found company founded in 2020 — NEW information
Search 2: Found funding details — NEW information
Search 3: Found product details — NEW information
Search 4: Found same info as search 2 — NO NEW information
Search 5: Found same info as search 1 — NO NEW information
→ STOP after 2 consecutive searches return no new information
```

### Strategy 3: Time-Boxed (Good for Cost Control)

Hard limit on iterations or time:

```python
# Time-based stopping
import time
start_time = time.time()
max_duration = 120  # 2 minutes max

while iteration < max_iterations:
    if time.time() - start_time > max_duration:
        return "Time limit reached. Here's what I found so far: ..."
    # ... continue loop
```

### Strategy 4: Question Coverage (Best for Complex Tasks)

The agent decomposes the goal into sub-questions and tracks coverage:

```
Goal: "Analyze competitor X"
Sub-questions:
  [✅] What does X do?
  [✅] How much funding have they raised?
  [✅] Who are their customers?
  [❌] What's their pricing model?
  [❌] Recent product launches?

Stop when: ALL sub-questions are answered OR 3 attempts per question fail.
```

---

### 🤔 Question 5: Implement Your Stopping Criterion

**Your challenge:** You're building an agent for customer support ticket triage. It reads a ticket, searches the knowledge base, and drafts a response.

The cost constraint: Each agent run must cost **under $0.02**.

The quality constraint: The response must address ALL parts of the customer's question.

**Design the stopping criterion.** Think about:
- What happens if the search finds nothing?
- What if the customer asks 3 things but you only found answers for 2?
- How do you trade off "more searches" vs "cost limit"?
- What does the agent do when it hits the cost limit but hasn't answered everything?

> **This is a real production problem.** Every AI support platform (Intercom, Zendesk AI, etc.) solves this. There's no perfect answer — there are tradeoffs. Think about what YOU would do.
>
> After you've thought about it, compare with the solution below.

---

### A Production Stopping Criterion Example

```python
class StoppingCriterion:
    """
    Hybrid stopping criterion for a customer support agent.
    
    Combines: coverage + cost + diminishing returns
    """
    
    def __init__(
        self,
        max_iterations: int = 10,
        max_cost: float = 0.02,      # $0.02 per run
        min_coverage: float = 0.8,   # 80% sub-questions answered
        max_empty_searches: int = 2,  # Stop after 2 useless searches
    ):
        self.max_iterations = max_iterations
        self.max_cost = max_cost
        self.min_coverage = min_coverage
        self.max_empty_searches = max_empty_searches
        self.consecutive_empty_searches = 0
    
    def should_stop(
        self,
        iteration: int,
        current_cost: float,
        coverage: float,
        last_search_was_empty: bool,
    ) -> tuple[bool, str]:
        """
        Decide whether the agent should stop.
        Returns (should_stop, reason).
        """
        # Hard limits
        if iteration >= self.max_iterations:
            return True, f"Max iterations ({self.max_iterations}) reached"
        
        if current_cost >= self.max_cost:
            return True, f"Cost limit (${self.max_cost}) reached"
        
        # Coverage met
        if coverage >= self.min_coverage:
            return True, f"Sufficient coverage ({coverage:.0%} ≥ {self.min_coverage:.0%})"
        
        # Diminishing returns
        if last_search_was_empty:
            self.consecutive_empty_searches += 1
            if self.consecutive_empty_searches >= self.max_empty_searches:
                return True, f"Last {self.max_empty_searches} searches returned nothing new"
        else:
            self.consecutive_empty_searches = 0
        
        return False, ""
```

---

## 🧪 Drills & Challenges

### Drill 1: The Contradiction Agent (25 min)

Your agent needs to research a topic where sources CONTRADICT each other.

**The task:**
> "What is the recommended daily sugar intake according to major health organizations?"

**The trap:** Different organizations recommend different amounts. WHO says <25g, AHA says <36g for men, ADA has different guidance for diabetics.

**Your challenge:**
1. Modify the ReAct agent to detect contradictions
2. When it finds conflicting numbers, it should search for WHY they differ
3. The final answer should explain the disagreement, not pick one side

**Extension:** What if both sources agree? How does the agent verify?

### Drill 2: The "Broken Tool" Drill (20 min)

Your agent has a tool that sometimes fails:

```python
def unreliable_search(query: str) -> str:
    """30% chance of failure — simulates real API flakiness."""
    import random
    if random.random() < 0.3:
        raise ConnectionError("Search API timed out")
    return real_search(query)
```

Run the agent with this tool. Watch what happens.

**Your challenge:**
1. Does the agent handle the failure gracefully?
2. Does it retry? Does it try a different approach?
3. Modify the agent to handle failures better.

### Drill 3: Design the Tool Set (25 min)

You need an agent that helps developers debug production issues. It has access to:

- PagerDuty API (see active incidents)
- Datadog API (query metrics)
- GitHub API (search code, read issues)
- Slack API (send messages)
- AWS CloudWatch (read logs)

**Design the tool set:**
1. What tools do you create? (don't say "one per API" — think about what the agent NEEDS)
2. What safety constraints does each tool need?
3. What if the agent tries to page the entire on-call team at 3 AM?

### Drill 4: The Stopping Game (15 min)

For each scenario, decide: **Should the agent stop or continue?**

```
Scenario A: The agent has found 3 sources that all say the same thing.
            One more search would cost $0.001. Stop or continue?

Scenario B: The agent has 2 conflicting sources. It's been searching for 
            5 minutes. The user is waiting. Stop or continue?

Scenario C: The agent found an answer in the first search. Low confidence.
            Second search confirms it. High confidence. Stop or continue?

Scenario D: The agent has searched 8 times. Every search returned unique,
            relevant information. Cost is $0.015 so far. Limit is $0.02.
            Stop or continue?
```

### Drill 5: The "ReAct or Not" Decision (15 min)

For each task, decide: **ReAct agent or simpler approach?**

```
Task 1: Extract all phone numbers from a 1000-page document.
Task 2: Research competitors and write a market analysis report.
Task 3: Monitor a stock price and alert when it drops 5%.
Task 4: Answer customer questions from a fixed FAQ document.
Task 5: Debug a server error by checking logs, configs, and recent deploys.
```

**For each task:**
- Would a ReAct agent be overkill?
- What simpler approach would work?
- When would a simpler approach fail?

---

## 🚦 Gate Check

Before moving to File 03 (Tool Design), ensure you can:

- [ ] **Explain the ReAct pattern** in 2 sentences: Reason → Act → Observe → Repeat
- [ ] **Build a ReAct agent from scratch** that interleaves reasoning with tool calls
- [ ] **Implement error recovery** — the agent handles tool failures gracefully
- [ ] **Implement information verification** — the agent cross-checks facts
- [ ] **Design stopping criteria** — the agent knows when it has enough information
- [ ] **Trace and debug** a ReAct agent run

### The Real Test

Set up this scenario:

```python
agent = ReActAgent(tools=[search_web, fetch_page, calculate])

# The trick: "Pi" could be 3.14159... OR the raspberry Pi
result = agent.run("What is the value of Pi and who invented it?")
```

**Does your agent:**
1. Recognize the ambiguity of "Pi"?
2. Search for BOTH meanings?
3. Clarify with you before assuming one meaning?
4. Produce a correct answer covering both?

**If your agent fails at any of these — that's a real production issue.** Ambiguity handling is one of the hardest problems in agent design.

### Final Questions (Think, Don't Just Read)

1. **Your agent finds 3 sources:**
   - Source A: "Revenue was $10M" (company press release)
   - Source B: "Revenue was $8.5M" (independent analyst)
   - Source C: "Revenue was $10M" (news article citing Source A)
   
   How many independent confirmations do you actually have? What's the RIGHT answer?

2. **Your agent has been running for 2 minutes and 12 iterations.** It found partial information but not everything. The user is waiting. The cost is $0.03. Do you:
   - Let it continue?
   - Stop and return partial results?
   - Return what it has and continue in the background?
   
   **This is a REAL decision you'll make in production.** There's no perfect answer — think about the tradeoffs.

3. **From Phase 1 (LLM Gateway):** You built a multi-provider gateway. Now you're building agents. How would you integrate your gateway into the ReAct agent so it can choose between GPT-4o and Claude depending on the task?

---

## 📚 Resources

**Foundational:**
- 📄 [ReAct: Synergizing Reasoning and Acting (Yao et al., 2022)](https://arxiv.org/abs/2210.03629) — The paper. Read sections 1-3 at minimum.
- 📄 [Lilian Weng — LLM Powered Autonomous Agents](https://lilianweng.github.io/posts/2023-06-23-agent/) — Comprehensive survey of agent patterns

**Code References:**
- 📄 [OpenAI Function Calling Documentation](https://platform.openai.com/docs/guides/function-calling)
- 📄 [Anthropic Tool Use Documentation](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)

**Production Reading:**
- 📄 [Building Effective Agents — Anthropic](https://docs.anthropic.com/en/docs/build-with-claude/agentic) — Their practical guide
- 📄 [The Practical Guide to ReAct Agents](https://www.analyticsvidhya.com/blog/2024/01/react-agents/) — Implementation patterns

**Your ReAct Progress Checklist:**
```
☐ ReAct pattern understood (this file)
☐ ReAct agent built and tested
☐ Error recovery implemented
☐ Contradiction detection works
☐ Stopping criteria implemented
☐ Agent handles ambiguous queries
☐ Trace debugging practiced
☐ Cost model understood for ReAct
```

---

> **Revisit the 3 questions from the beginning:**
>
> 1. **The planning flaw** — Your Zero-Shot agent followed a frozen plan and failed. The ReAct agent re-plans after EVERY action, adapting to what it finds.
>
> 2. **The contradiction problem** — ReAct gives you a reasoning trace. The agent can (and should) investigate conflicts by searching for more sources and explaining why it trusts one over another.
>
> 3. **The stopping problem** — This is still hard, but now you have strategies: coverage checklists, diminishing returns detection, cost limits, and hybrid approaches.
>
> **Key realization:** ReAct is not a framework or a library. It's a PATTERN — think, act, observe, repeat. This pattern is the foundation of every production agent system. LangGraph, LangChain agents, AutoGPT, BabyAGI — they all implement some version of this loop.
>
> **Next:** File 03 — Tool Design. How to build tools that are powerful, safe, and reliable. Every attack surface, every safety pattern, every validation technique.

---

> **Senior engineer closing thought:**
>
> *"The ReAct pattern looks simple on paper — just think, act, observe, repeat. But the magic isn't in the loop. It's in what you put INTO the thinking. A good system prompt that makes the LLM genuinely reason about what it's doing is worth 100x more than any framework optimization. I've seen agents with perfect infrastructure produce garbage because the prompt didn't force real reasoning. And I've seen agents with simple loops produce brilliant work because every thought was genuine analysis. Write the prompt first. Then build the loop."*
