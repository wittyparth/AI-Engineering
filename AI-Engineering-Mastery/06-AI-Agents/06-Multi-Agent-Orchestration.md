# Multi-Agent Orchestration

## Communication Patterns

### 1. Direct Messaging
```python
class AgentMessage(BaseModel):
    sender: str
    recipient: str
    message_type: str  # "request", "response", "update", "error"
    content: str
    metadata: dict
```

### 2. Shared Blackboard
```python
class Blackboard:
    """Shared state that all agents can read/write."""
    def __init__(self):
        self.state = {}
        self.lock = asyncio.Lock()

    async def write(self, key: str, value: Any, agent: str):
        async with self.lock:
            self.state[key] = {"value": value, "author": agent, "timestamp": time.time()}

    async def read(self, key: str) -> Any:
        return self.state.get(key)

    async def watch(self, pattern: str) -> AsyncIterator:
        """Subscribe to changes matching a pattern."""
        while True:
            # Poll or event-based
            await asyncio.sleep(0.1)
```

### 3. Queue-Based
```python
class AgentQueue:
    """Agents publish tasks, other agents consume them."""
    def __init__(self):
        self.queues = defaultdict(asyncio.Queue)

    async def publish(self, to_role: str, message: AgentMessage):
        await self.queues[to_role].put(message)

    async def consume(self, role: str) -> AgentMessage:
        return await self.queues[role].get()
```

## Orchestrator-Worker Example

```python
class OrchestratorAgent:
    def __init__(self):
        self.workers = {
            "researcher": ResearchWorker(),
            "analyst": AnalysisWorker(),
            "writer": WritingWorker(),
        }

    async def run(self, task: str) -> str:
        # 1. Plan: break task into subtasks
        plan = await self._create_plan(task)

        # 2. Dispatch: send subtasks to workers
        tasks = []
        for subtask in plan["subtasks"]:
            worker = self.workers[subtask["worker"]]
            tasks.append(worker.run(subtask["description"]))

        # 3. Gather: collect all results
        results = await asyncio.gather(*tasks)

        # 4. Synthesize: combine into final output
        return await self._synthesize(task, results)
```

## Handoff Pattern (Anthropic MCP Style)

```python
class AgentHandoff:
    """One agent can hand off control to another."""

    async def process(self, task: str, context: dict) -> str:
        if self._needs_specialist(task):
            specialist = self._select_specialist(task)
            # Hand off with context
            result = await specialist.process(
                task=task,
                context={**context, "handoff_from": self.name}
            )
            return result
        # Handle directly
        return await self._handle(task, context)
```

## 🔴 Senior: Multi-Agent Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Too many agents | Coordination overhead | Start with 2-3 agents max |
| No shared state | Agents work in silos | Blackboard pattern |
| All agents can talk | Communication chaos | Supervisor pattern |
| No termination | Infinite debates | Max rounds, timeouts |
| Identical agents | No specialization | Give each agent specific tools + role |
