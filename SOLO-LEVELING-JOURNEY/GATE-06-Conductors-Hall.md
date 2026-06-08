# ⛩️ GATE 06 — Conductor's Hall (Agents & Tool Use)

**Rank Requirement:** B-
**Theme:** Building autonomous agents — tool calling, planning, memory, and multi-agent systems
**Total Days:** 8 Dungeons + 1 Boss Battle
**Core Skills:** Function calling, agent loops, tool design, planning, agent memory, multi-agent orchestration

---

## Dungeon 1 — What Makes an Agent?

**Day 1 | Concept:** An agent is an LLM + tools + loop. The LLM decides, the tools act, the loop persists.

**Agent Loop:**
```
while task_not_done:
    1. Observe (current state + user input)
    2. Think (what should I do next?)
    3. Act (call tool or respond)
    4. Observe result
```

**Agent vs RAG:**
| | RAG | Agent |
|--|-----|-------|
| Decision making | None (fixed pipeline) | Dynamic (LLM decides) |
| Tool use | One tool (retriever) | Many tools |
| State | Stateless | Stateful |
| Loop | Linear | While loop |

**🗡️ Dungeon Mastery:** Implement the simplest possible agent loop: an LLM that can call `get_weather(city)` or respond directly. Run 3 turns.

**💻 Code — Minimal Agent Loop:**
```python
import json

class MinimalAgent:
    def __init__(self, llm, tools: dict):
        self.llm = llm
        self.tools = tools  # {"tool_name": callable}
        self.messages = [{"role": "system", "content": self._system_prompt()}]

    def run(self, user_input: str, max_turns: int = 5) -> str:
        self.messages.append({"role": "user", "content": user_input})

        for turn in range(max_turns):
            response = self.llm.chat(self.messages)
            self.messages.append({"role": "assistant", "content": response})

            action = self._parse_action(response)
            if not action:  # No tool call = final response
                return response

            # Execute tool
            result = self._execute(action)
            self.messages.append({"role": "tool", "content": str(result)})

        return "Max turns reached."

    def _parse_action(self, response: str) -> dict | None:
        try:
            if "```tool" in response:
                block = response.split("```tool")[1].split("```")[0].strip()
                return json.loads(block)
            return None
        except:
            return None

    def _execute(self, action: dict) -> str:
        tool_name = action.get("tool")
        args = action.get("args", {})
        if tool_name in self.tools:
            return self.tools[tool_name](**args)
        return f"Unknown tool: {tool_name}"

    def _system_prompt(self) -> str:
        return """You are an agent with tools. Available:

- get_weather(city: str) -> str
- search_web(query: str) -> str

To use a tool, respond with:
```tool
{"tool": "get_weather", "args": {"city": "London"}}
```

Otherwise respond normally."""

# Usage:
# agent = MinimalAgent(llm=my_llm, tools={"get_weather": get_weather_fn})
# print(agent.run("What's the weather in Tokyo?"))
```

**👹 Boss Lesson:** An agent is just a loop. The magic is in the tools and the LLM's ability to choose correctly.

---

## Dungeon 2 — Function Calling (Native)

**Day 2 | Concept:** Modern LLMs support native function calling — structured tool definitions the model can invoke directly.

**Function Definition Format:**
```json
{
  "name": "get_weather",
  "description": "Get current temperature for a city",
  "parameters": {
    "type": "object",
    "properties": {
      "city": {"type": "string", "description": "City name"},
      "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
    },
    "required": ["city"]
  }
}
```

**🗡️ Dungeon Mastery:** Define 3 function definitions (search, calculator, send_email). Pass them to an LLM with a query that requires chaining 2 tools. Verify the model calls them correctly.

**💻 Code — Function Calling:**
```python
from typing import Any

class FunctionCaller:
    """Wraps native function calling API."""

    @staticmethod
    def define_tool(name: str, description: str, parameters: dict) -> dict:
        return {
            "type": "function",
            "function": {
                "name": name,
                "description": description,
                "parameters": parameters,
            },
        }

    @staticmethod
    def execute_function_call(func_call: dict, tool_implementations: dict) -> str:
        name = func_call["name"]
        args = json.loads(func_call.get("arguments", "{}"))
        if name in tool_implementations:
            result = tool_implementations[name](**args)
            return json.dumps({"result": result})
        return json.dumps({"error": f"Unknown tool: {name}"})


# Tool definitions:
TOOLS = [
    FunctionCaller.define_tool(
        "search_docs",
        "Search through documentation",
        {"type": "object", "properties": {
            "query": {"type": "string", "description": "Search query"},
            "max_results": {"type": "integer", "description": "Max results"},
        }, "required": ["query"]},
    ),
    FunctionCaller.define_tool(
        "calculate",
        "Evaluate a mathematical expression",
        {"type": "object", "properties": {
            "expression": {"type": "string", "description": "Math expression"},
        }, "required": ["expression"]},
    ),
]

# Tool implementations:
def search_docs(query: str, max_results: int = 3) -> list:
    return [f"Result {i} for {query}" for i in range(max_results)]

def calculate(expression: str) -> float:
    return eval(expression)  # Note: use safer eval in production

TOOL_IMPLS = {"search_docs": search_docs, "calculate": calculate}
```

**👹 Boss Lesson:** Native function calling is 10x more reliable than parsing tool calls from freeform text. Always prefer it when available.

---

## Dungeon 3 — Tool Design Patterns

**Day 3 | Concept:** Tool design = agent capability. Bad tool design = confused agent.

**Tool Design Principles:**
1. **Granularity:** One tool = one capability. Don't make a swiss-army tool.
2. **Self-explanatory names:** `send_email_to_user(id, subject, body)` not `process(id, data)`
3. **Descriptive parameters:** Use enums, descriptions, and constraints
4. **Error-returning:** Tools should return useful error messages, not throw
5. **Idempotency where possible:** Same input = same output for reads

**Common Tool Categories:**
| Category | Examples |
|----------|----------|
| Retrieval | search_docs, get_page, list_files |
| Computation | calculate, code_interpreter, translate |
| External | send_email, post_tweet, create_issue |
| Memory | save_note, recall, list_memories |
| System | read_file, write_file, execute_command |

**🗡️ Dungeon Mastery:** Design a set of 5 tools for a "personal assistant" agent. Write their definitions. Explain why each tool's granularity is correct.

**💻 Code — Tool Design Templates:**
```python
_TOOL_TEMPLATES = {
    "read": {
        "name": "read_file",
        "description": "Read contents of a file at given path. Returns text content or error.",
        "parameters": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "Absolute file path"},
                "max_length": {"type": "integer", "description": "Max chars to return"},
            },
            "required": ["path"],
        },
    },
    "search": {
        "name": "search_codebase",
        "description": "Full-text search across the codebase. Returns matching file paths with snippets.",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Search query"},
                "file_pattern": {"type": "string", "description": "Filter by pattern, e.g. *.py"},
            },
            "required": ["query"],
        },
    },
    "execute": {
        "name": "run_command",
        "description": "Execute a shell command in the workspace. Returns stdout, stderr, exit code.",
        "parameters": {
            "type": "object",
            "properties": {
                "command": {"type": "string", "description": "Command to execute"},
                "timeout_seconds": {"type": "integer", "description": "Max execution time"},
            },
            "required": ["command"],
        },
    },
}
```

**👹 Boss Lesson:** Your tools ARE your agent's capabilities. Design them carefully — a confused tool leads to a confused agent.

---

## Dungeon 4 — Planning & Reasoning

**Day 4 | Concept:** Agents need to plan before acting. ReAct, Plan-and-Solve, and Tree-of-Thought are the key paradigms.

**Paradigms:**
| Approach | Flow | Best For |
|----------|------|----------|
| ReAct | Think → Act → Observe → Repeat | Simple multi-step tasks |
| Plan-and-Solve | Plan first → Execute plan step by step | Complex tasks with dependencies |
| Tree-of-Thought | Explore N plans in parallel → Pick best | Open-ended problems |
| Reflexion | Act → Evaluate → Reflect → Improve | Code generation, writing |

**💻 Code — Plan-and-Solve Agent:**
```python
class PlanAndSolveAgent:
    def __init__(self, llm, tools: dict, max_steps: int = 15):
        self.llm = llm
        self.tools = tools
        self.max_steps = max_steps

    def run(self, task: str) -> str:
        # Step 1: Create plan
        plan = self._create_plan(task)
        # Step 2: Execute plan step by step
        for step in plan:
            result = self._execute_step(step)
            # Check if done
            if self._is_complete(task, result):
                return result
        return "Task incomplete."

    def _create_plan(self, task: str) -> list[str]:
        prompt = f"""Break this task into 3-5 steps. Each step should be a concrete action.

Task: {task}

Format each step as: STEP N: description"""
        response = self.llm.generate(prompt)
        return [line.strip() for line in response.split("\n")
                if line.strip().startswith("STEP")]

    def _execute_step(self, step: str) -> str:
        prompt = f"""Current step: {step}

Use available tools to complete this step. Available: {list(self.tools.keys())}

Describe what you did and what you found:"""
        # In reality, this would involve tool calling
        return self.llm.generate(prompt)

    def _is_complete(self, task: str, result: str) -> bool:
        prompt = f"""Task: {task}
Result so far: {result}

Is the task complete? Reply YES or NO only."""
        return "YES" in self.llm.generate(prompt).upper()
```

**🗡️ Dungeon Mastery:** Give an agent the task "Research RAG vs Fine-tuning and create a comparison table." Watch it plan, execute sub-tasks, and produce the final output.

**👹 Boss Lesson:** Planning separates toy agents from useful agents. A plan is just a structured sequence of tool calls.

---

## Dungeon 5 — Agent Memory

**Day 5 | Concept:** Agents need memory to be coherent across turns and sessions. Three types: short-term, long-term, and episodic.

**Memory Types:**
```
Short-term (context window):
  Current conversation, last N tool results

Long-term (vector DB):
  Facts learned about the user, project context

Episodic (log):
  What happened in previous sessions, past mistakes
```

**🗡️ Dungeon Mastery:** Implement an agent that remembers the user's name, preferred language, and previous task across sessions. Save to JSON; load on restart.

**💻 Code — Agent Memory System:**
```python
import json
from pathlib import Path
from datetime import datetime

class AgentMemory:
    def __init__(self, storage_path: str = "agent_memory.json"):
        self.storage_path = Path(storage_path)
        self.data = self._load()

    def _load(self) -> dict:
        if self.storage_path.exists():
            return json.loads(self.storage_path.read_text())
        return {"user_info": {}, "episodes": [], "facts": []}

    def _save(self):
        self.storage_path.write_text(json.dumps(self.data, indent=2))

    def remember_fact(self, fact: str, source: str = "conversation"):
        self.data["facts"].append({
            "fact": fact,
            "source": source,
            "timestamp": datetime.now().isoformat(),
        })
        self._save()

    def recall_facts(self, query: str, top_k: int = 3) -> list[str]:
        """Simple keyword-based recall. In production, use embeddings."""
        query_words = query.lower().split()
        scored = []
        for entry in self.data["facts"]:
            score = sum(1 for w in query_words if w in entry["fact"].lower())
            if score > 0:
                scored.append((score, entry["fact"]))
        scored.sort(reverse=True)
        return [fact for _, fact in scored[:top_k]]

    def log_episode(self, action: str, result: str, success: bool):
        self.data["episodes"].append({
            "action": action,
            "result": result[:200],
            "success": success,
            "timestamp": datetime.now().isoformat(),
        })
        self._save()

    def remember_user_info(self, key: str, value: str):
        self.data["user_info"][key] = value
        self._save()
```

**👹 Boss Lesson:** No memory = no learning. Every agent session should leave traces for the next one.

---

## Dungeon 6 — Multi-Agent Systems

**Day 6 | Concept:** One agent is a tool. Multiple agents collaborating solve problems no single agent can.

**Architecture Patterns:**
| Pattern | Structure | Best For |
|---------|-----------|----------|
| Supervisor | One agent delegates to specialist agents | Complex workflows |
| Debate | Two agents argue, third judges | Decision making |
| Swarm | Many agents, same role, vote | Research, data collection |
| Pipeline | Agent A → Agent B → Agent C | Sequential processing |

**🗡️ Dungeon Mastery:** Build a supervisor agent with 3 specialist agents (Researcher, Writer, Reviewer). Give it a task: "Write a blog post about vector databases."

**💻 Code — Supervisor Agent:**
```python
class SupervisorAgent:
    def __init__(self, llm, specialists: dict):
        self.llm = llm
        self.specialists = specialists

    def run(self, task: str) -> str:
        prompt = f"""You are a supervisor. Delegate work to specialists.
Specialists: {list(self.specialists.keys())}

Task: {task}

Create a delegation plan. Output format:
DELEGATE to [specialist]: [specific instructions]"""

        plan = self.llm.generate(prompt)

        results = {}
        for line in plan.split("\n"):
            if line.startswith("DELEGATE"):
                parts = line.replace("DELEGATE to ", "").split(":", 1)
                if len(parts) == 2:
                    specialist = parts[0].strip()
                    instructions = parts[1].strip()
                    if specialist in self.specialists:
                        results[specialist] = self.specialists[specialist](instructions)

        # Synthesize results
        synthesis_prompt = f"""Task: {task}

Results from specialists:
{json.dumps(results, indent=2)}

Synthesize these into a final response:"""
        return self.llm.generate(synthesis_prompt)

# Specialists are just functions:
def researcher(query: str) -> str:
    return f"Researched: {query}"

def writer(topic: str) -> str:
    return f"Written content about: {topic}"

def reviewer(text: str) -> str:
    return f"Reviewed and approved: {text[:50]}..."
```

**👹 Boss Lesson:** Multi-agent systems are powerful but complex. Start with a single agent, add specialists only when you need them.

---

## Dungeon 7 — Error Handling & Recovery

**Day 7 | Concept:** Agents will fail. Tool calls will error. The mark of a good agent is graceful recovery.

**Failure Patterns:**
| Failure | Symptom | Recovery |
|---------|---------|----------|
| Tool timeout | No response | Retry with simpler params |
| Bad tool choice | Wrong result | Let agent reflect and retry |
| Infinite loop | Same action repeatedly | Max retries + escalate |
| Context overflow | Truncated response | Summarize and continue |
| Hallucination | Confident wrong answer | Cross-check with tools |

**💻 Code — Resilient Agent:**
```python
class ResilientAgent:
    def __init__(self, llm, tools: dict, max_retries: int = 3):
        self.llm = llm
        self.tools = tools
        self.max_retries = max_retries
        self.action_history = []

    def run(self, task: str) -> str:
        context = task
        retries = 0

        while retries < self.max_retries:
            try:
                result = self._execute_with_tools(context)
                if self._is_good_result(result):
                    return result
                # Bad result — reflect and retry
                context = self._reflect(result)
                retries += 1
            except Exception as e:
                context = f"Error occurred: {e}\nTry a different approach."
                retries += 1

        return f"Failed after {self.max_retries} attempts. Partial result: {result}"

    def _execute_with_tools(self, context: str) -> str:
        # Implementation with tool calls
        pass

    def _is_good_result(self, result: str) -> bool:
        # Check for error indicators
        error_indicators = ["error", "failed", "invalid", "timeout", "exception"]
        return not any(indicator in result.lower() for indicator in error_indicators)

    def _reflect(self, previous_result: str) -> str:
        prompt = f"""The previous attempt produced a poor result.
Result: {previous_result}

Analyze what went wrong and propose a different approach.
Next attempt should:"""
        return self.llm.generate(prompt)


# Deduplication — prevent repeated identical actions:
class ActionDeduplicator:
    def __init__(self, window: int = 5):
        self.history = []
        self.window = window

    def is_duplicate(self, action: dict) -> bool:
        recent = self.history[-self.window:]
        return action in recent

    def record(self, action: dict):
        self.history.append(action)
```

**👹 Boss Lesson:** Agents fail. Graceful recovery separates production agents from demos.

---

## Dungeon 8 — Agent Observability

**Day 8 | Concept:** You can't debug what you can't see. Every agent action must be logged, traced, and inspectable.

**What to Log:**
- Every thought the agent had
- Every tool call (name, args, result)
- Every retry and why
- Token usage per step
- Timing per step

**🗡️ Dungeon Mastery:** Add comprehensive logging to your agent. Run a 3-step task. Review the full trace. Identify where the agent struggled.

**💻 Code — Agent Tracer:**
```python
from datetime import datetime
import json
import uuid

class AgentTracer:
    def __init__(self, agent_name: str = "agent"):
        self.session_id = str(uuid.uuid4())[:8]
        self.agent_name = agent_name
        self.steps = []
        self.start_time = datetime.now()

    def log_thought(self, thought: str):
        self.steps.append({
            "type": "thought",
            "content": thought,
            "timestamp": datetime.now().isoformat(),
        })

    def log_tool_call(self, tool: str, args: dict, result: str, duration_ms: float = 0):
        self.steps.append({
            "type": "tool_call",
            "tool": tool,
            "args": args,
            "result_preview": str(result)[:200],
            "duration_ms": duration_ms,
            "timestamp": datetime.now().isoformat(),
        })

    def log_error(self, error: str):
        self.steps.append({
            "type": "error",
            "content": error,
            "timestamp": datetime.now().isoformat(),
        })

    def get_trace(self) -> dict:
        total_time = (datetime.now() - self.start_time).total_seconds()
        tool_calls = [s for s in self.steps if s["type"] == "tool_call"]
        return {
            "session": self.session_id,
            "agent": self.agent_name,
            "total_steps": len(self.steps),
            "total_tool_calls": len(tool_calls),
            "total_time_seconds": total_time,
            "steps": self.steps,
        }

    def print_trace(self):
        trace = self.get_trace()
        print(f"\n{'='*50}")
        print(f"TRACE: {trace['session']} | {trace['agent']}")
        print(f"{trace['total_steps']} steps in {trace['total_time_seconds']:.1f}s")
        print(f"{'='*50}")
        for step in trace['steps']:
            if step['type'] == 'thought':
                print(f"  💭 {step['content'][:100]}")
            elif step['type'] == 'tool_call':
                print(f"  🔧 {step['tool']}({json.dumps(step['args'])[:80]}) "
                      f"[{step.get('duration_ms', 0):.0f}ms]")
            elif step['type'] == 'error':
                print(f"  ❌ {step['content'][:100]}")
```

**👹 Boss Lesson:** If your agent is a black box, it's not production-ready. Trace everything.

---

## ⚔️ BOSS BATTLE: The Multi-Tool Orchestrator

**Objective:** Build an agent that can:
1. Accept a complex multi-step task (e.g., "Research topic X, summarize it, save to file, and email me")
2. Break it into a plan autonomously
3. Execute each step using appropriate tools
4. Handle tool failures with retry/alternative approaches
5. Log every action with full tracing
6. Produce a final summary of what was done

**🗡️ Victory Conditions:**
- Task decomposed into 3+ subtasks
- At least 2 different tools used
- Handles at least one error scenario gracefully
- Full trace logged and readable
- Produces coherent final output

**🏆 Rewards:**
- +500 XP
- B- Rank → B Rank
- Skill Unlock: **Agent Weaver** — 2x faster at building and debugging agent systems
- Title: **The Conductor**

---

*"An agent without tools is just a chatbot. Tools without an agent is just an API. Together, they are a workforce."*
