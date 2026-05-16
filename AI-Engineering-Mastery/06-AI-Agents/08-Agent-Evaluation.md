# Agent Evaluation

## The Hardest Problem

Evaluating agents is fundamentally harder than evaluating RAG because:
1. **Non-deterministic**: Same task, different paths each time
2. **Multi-step**: Which step failed? The tool call? The reasoning? The generation?
3. **Open-ended**: No single "correct" answer
4. **Compound errors**: One bad step cascades

## Evaluation Dimensions

### 1. Task Completion
```python
async def eval_task_completion(agent, tasks: list[dict]) -> dict:
    results = {"success": 0, "partial": 0, "failed": 0, "details": []}
    for task in tasks:
        result = await agent.run(task["prompt"])
        success = await judge_agent(task, result)
        results[success] += 1
        results["details"].append({"task": task, "result": result, "success": success})
    return results
```

### 2. Step-Level Evaluation
```python
class StepEvaluator:
    def __init__(self, llm):
        self.llm = llm

    async def evaluate_step(self, step: dict) -> dict:
        return {
            "reasoning_quality": await self._score_reasoning(step["thought"]),
            "tool_selection": await self._score_tool_choice(step["action"]),
            "tool_execution": step["observation"]["success"],
            "efficiency": 1.0 if step["step_number"] <= self._expected_steps(step["task"]) else 0.5,
        }

    async def _score_reasoning(self, thought: str) -> float:
        prompt = f"Rate the quality of this agent's reasoning (0-10):\n{thought}"
        score = await self.llm.generate(prompt)
        return int(score.strip()) / 10.0
```

### 3. Cost & Efficiency
```python
def eval_efficiency(trajectory: list[dict]) -> dict:
    return {
        "total_steps": len(trajectory),
        "total_tokens": sum(s["tokens"] for s in trajectory),
        "total_cost": sum(s["cost"] for s in trajectory),
        "unnecessary_steps": count_unnecessary(trajectory),
        "tool_errors": count_errors(trajectory),
    }
```

### 4. Robustness Testing
```python
class RobustnessEval:
    def __init__(self, agent):
        self.agent = agent

    async def test_tool_failure(self):
        """Agent should handle tool failures gracefully."""
        task = "Find information about X"
        # Disable a tool mid-task
        result = await self.agent.run(task, disable_tool=["search"])
        return "alternative" in result.lower() or "fallback" in result.lower()

    async def test_ambiguous_query(self):
        """Agent should ask for clarification."""
        task = "Tell me about it"  # No context
        result = await self.agent.run(task)
        return "?" in result or "clarify" in result.lower()

    async def test_max_steps(self):
        """Agent should terminate gracefully at step limit."""
        task = "Do something very complex that will hit the limit"
        result = await self.agent.run(task, max_steps=3)
        return "limit" in result.lower() or "sorry" in result.lower()
```

## 🔴 Senior: Evaluation as a Service

```python
class AgentEvalPipeline:
    """Continuous evaluation pipeline for agents."""

    def __init__(self):
        self.suite = EvalSuite()
        self.regression_db = RegressionDB()

    async def run_full_suite(self, agent_version: str) -> EvalReport:
        baseline = await self.regression_db.get_baseline()
        results = await self.suite.run(agent_version)

        report = EvalReport(
            version=agent_version,
            results=results,
            regressions=self._find_regressions(results, baseline),
        )

        if report.has_regressions:
            await self._alert_team(report)
            return None  # Don't deploy

        await self.regression_db.store(agent_version, results)
        return report
```

## Drill: Build an Agent Eval Suite

Create eval cases for:
1. **Correct tool selection** (3 test cases)
2. **Graceful error handling** (3 scenarios: tool down, bad input, timeout)
3. **Efficiency** (should not take more than N steps for simple tasks)
4. **Edge cases** (empty input, very long input, adversarial prompts)
5. **Cost awareness** (shouldn't call expensive tool unnecessarily)
