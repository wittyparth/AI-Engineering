# Memory Systems

## The Problem

LLMs have no memory between calls. Agents need memory to persist state, learn from interactions, and maintain context across sessions.

## Memory Types

### 1. Short-Term Memory (Within Session)
```python
class ShortTermMemory:
    def __init__(self, max_turns: int = 20):
        self.history = []
        self.max_turns = max_turns

    def add(self, role: str, content: str):
        self.history.append({"role": role, "content": content})
        if len(self.history) > self.max_turns * 2:
            # Summarize old turns
            self._summarize_old()

    def get_context(self) -> list[dict]:
        return self.history[-self.max_turns * 2:]
```

### 2. Long-Term Memory (Across Sessions)
```python
class LongTermMemory:
    def __init__(self, vector_db):
        self.vector_db = vector_db

    async def remember(self, key: str, content: str, metadata: dict = None):
        """Store a memory with a key for later retrieval."""
        await self.vector_db.store(
            content=content,
            metadata={"type": "memory", "key": key, **(metadata or {})}
        )

    async def recall(self, query: str, k: int = 5) -> list[str]:
        """Retrieve relevant memories."""
        results = await self.vector_db.search(query, k=k)
        return [r.content for r in results]

    async def forget(self, key: str):
        """Delete a specific memory."""
        await self.vector_db.delete(filter={"key": key})
```

### 3. Episodic Memory (Specific Experiences)
```python
class EpisodicMemory:
    """Remembers specific past episodes and their outcomes."""

    async def store_episode(self, task: str, actions: list, outcome: str):
        await self.db.store({
            "type": "episode",
            "task": task,
            "actions": actions,
            "outcome": outcome,
            "timestamp": datetime.utcnow(),
        })

    async def recall_similar(self, task: str) -> list[dict]:
        """Find similar past experiences."""
        return await self.db.search(task, filter={"type": "episode"})
```

## Memory-Augmented Agent

```python
class MemoryAugmentedAgent:
    def __init__(self, tools, llm, short_term, long_term, episodic):
        self.tools = tools
        self.llm = llm
        self.stm = short_term
        self.ltm = long_term
        self.episodic = episodic

    async def run(self, task: str) -> str:
        # 1. Recall relevant memories
        past_memories = await self.ltm.recall(task)
        similar_episodes = await self.episodic.recall_similar(task)

        # 2. Build context with memories
        context = self._build_context(task, past_memories, similar_episodes)

        # 3. Run agent loop
        result = await self._react_loop(context)

        # 4. Store this episode
        await self.episodic.store_episode(task, self._actions_taken, result)

        return result
```

## 🔴 Senior: Memory Strategy Guide

| Scenario | Memory Type | Why |
|----------|-------------|-----|
| Chatbot | Short-term only | Sessions are independent |
| Personal assistant | Short + Long-term | Needs user preferences |
| Code agent | Short + Episodic | Learns from past errors |
| Research agent | All three | Builds knowledge over time |
| Customer support | Short + Long-term (KB) | Needs product knowledge |
