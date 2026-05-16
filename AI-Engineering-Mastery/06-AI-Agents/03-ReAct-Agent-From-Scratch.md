# ReAct Agent From Scratch

## The ReAct Loop

Reason → Act → Observe → Repeat

```python
class ReActAgent:
    def __init__(self, tools: dict[str, Tool], llm):
        self.tools = tools
        self.llm = llm

    async def run(self, task: str, max_steps: int = 10) -> str:
        messages = [{"role": "system", "content": self._system_prompt()}]
        messages.append({"role": "user", "content": task})

        for step in range(max_steps):
            # 1. THINK (Reason)
            response = await self.llm.generate(messages)
            messages.append({"role": "assistant", "content": response})

            # Check if agent is done
            if "<FINAL_ANSWER>" in response:
                return self._extract_answer(response)

            # 2. ACT
            action = self._parse_action(response)
            if not action:
                messages.append({
                    "role": "user",
                    "content": "Invalid action format. Use: <TOOL>tool_name</TOOL>\n<PARAMS>args</PARAMS>"
                })
                continue

            # 3. OBSERVE
            tool = self.tools.get(action["tool"])
            if not tool:
                observation = f"Error: Tool '{action['tool']}' not found. Available: {list(self.tools.keys())}"
            else:
                try:
                    observation = await tool.run(**action["params"])
                except Exception as e:
                    observation = f"Error executing {action['tool']}: {e}"

            messages.append({
                "role": "user",
                "content": f"Observation: {observation}"
            })

        return "Max steps reached without final answer."

    def _system_prompt(self) -> str:
        tool_descriptions = "\n".join([
            f"- {name}: {tool.description}" for name, tool in self.tools.items()
        ])
        return f"""You are an agent that solves tasks by using tools.

Available tools:
{tool_descriptions}

For each step, respond with:
<THOUGHT>your reasoning about what to do next</THOUGHT>
<TOOL>tool_name</TOOL>
<PARAMS>{{"param1": "value1"}}</PARAMS>

When you have the final answer:
<FINAL_ANSWER>your answer</FINAL_ANSWER>"""
```

## Tool Implementation

```python
class SearchTool(Tool):
    def __init__(self):
        self.description = "Search the knowledge base. Use for finding information."
        self.parameters = {"query": {"type": "string"}}

    async def run(self, query: str) -> list[dict]:
        # Implementation
        return results
```

## The Complete Loop

```
Step 1: User: "What's the capital of France and what's the weather there?"
Step 2: Agent: <THOUGHT>I need to find the capital of France and its weather.
                    First, search for the capital.</THOUGHT>
         <TOOL>search_documentation</TOOL>
         <PARAMS>{"query": "capital of France"}</PARAMS>
Step 3: Tool: Paris
Step 4: Agent: <THOUGHT>The capital is Paris. Now get the weather.</THOUGHT>
         <TOOL>get_weather</TOOL>
         <PARAMS>{"location": "Paris"}</PARAMS>
Step 5: Tool: 22°C, sunny
Step 6: Agent: <FINAL_ANSWER>The capital of France is Paris, where it's currently 22°C and sunny.</FINAL_ANSWER>
```

## 🔴 Senior: ReAct Failure Modes

| Failure | Symptom | Fix |
|---------|---------|-----|
| Infinite loop | Same action repeated | Max step limit + detect loops |
| Tool confusion | Wrong tool chosen | Better tool descriptions |
| Hallucinated observations | Agent makes up tool output | Validate tool output |
| Context overflow | Hits token limit | Summarize history |
| Overly verbose | 10 steps for simple task | Penalize unnecessary actions |
