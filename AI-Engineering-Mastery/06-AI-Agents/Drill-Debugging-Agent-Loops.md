# Drill: Debugging Agent Loops

**Time**: 30 min  **Difficulty**: Medium

## The Problem

Your agent is stuck in an infinite loop. Here's the trace:

```
Step 1: Thought → Use search tool → Searches "X" → No results
Step 2: Thought → Use search tool → Searches "X" → No results
Step 3: Thought → Use search tool → Searches "X" → No results
... (repeats forever until max_steps)
```

## Find the Bug

```python
class BuggyReActAgent:
    async def run(self, task: str, max_steps: int = 10):
        messages = [self._system_prompt(), {"role": "user", "content": task}]

        for step in range(max_steps):
            response = await self.llm.generate(messages)
            messages.append({"role": "assistant", "content": response})

            if "<FINAL_ANSWER>" in response:
                return response

            action = self._parse_action(response)  # Bug 1: returns None on invalid format
            observation = await self._execute_tool(action)  # Bug 2: doesn't handle None
            messages.append({"role": "user", "content": f"Observation: {observation}"})
            # Bug 3: No detection of repeated actions
```

## Find and Fix All 5 Bugs

1. **No loop detection**: If the same action is taken 3+ times, force a different approach
2. **No tool error handling**: If a tool returns an error, the agent should try a different tool
3. **No progress tracking**: If the agent hasn't made progress in N steps, summarize and restart
4. **No tool output validation**: If the tool returns empty results, the agent should try a different query
5. **No dead-end detection**: If the agent has exhausted all tools, it should give up gracefully

## Fixed Version

```python
class FixedReActAgent:
    async def run(self, task: str, max_steps: int = 10):
        messages = [self._system_prompt(), {"role": "user", "content": task}]
        action_history = []
        no_progress_count = 0

        for step in range(max_steps):
            response = await self.llm.generate(messages)
            messages.append({"role": "assistant", "content": response})

            if "<FINAL_ANSWER>" in response:
                return response

            action = self._parse_action(response)
            if not action:
                messages.append({"role": "user", "content": "Invalid format. Use the exact tool format."})
                continue

            # Detect repeated actions
            action_key = f"{action['tool']}:{action['params']}"
            if action_history.count(action_key) >= 2:
                messages.append({
                    "role": "user",
                    "content": f"You've tried {action_key} {action_history.count(action_key)} times. Try a different approach."
                })
                continue
            action_history.append(action_key)

            # Execute with error handling
            try:
                observation = await self._execute_tool(action)
            except Exception as e:
                observation = f"Tool error: {e}. Try a different tool."

            messages.append({"role": "user", "content": f"Observation: {observation}"})

        return "I was unable to complete this task. Here's what I tried: " + str(action_history)
```
