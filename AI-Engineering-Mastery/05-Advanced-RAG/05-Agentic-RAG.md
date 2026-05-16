# Agentic RAG

## The Core Idea

Instead of a fixed retrieval → generate pipeline, an AGENT decides WHEN and HOW to retrieve.

```
Agent receives query
  ↓
Is the query clear? → If no → Ask clarifying question
Is retrieval needed?  → If no → Answer from knowledge
What to retrieve?     → Vector DB? Web search? SQL? Code?
Is info sufficient?   → If no → Try a different strategy
Generate answer
```

## Agentic RAG Loop

```python
class AgenticRAG:
    def __init__(self, tools: dict, llm):
        self.tools = tools  # {"vector_search": ..., "web_search": ..., "sql_query": ...}
        self.llm = llm

    async def answer(self, query: str, max_steps: int = 5) -> str:
        context = []
        step = 0

        while step < max_steps:
            # Decide next action
            action = await self._decide_action(query, context)

            if action["type"] == "answer":
                return await self._generate(query, context)

            elif action["type"] == "retrieve":
                tool = self.tools[action["tool"]]
                results = await tool.search(action["sub_query"])
                context.extend(results)

            elif action["type"] == "clarify":
                return action["question"]  # Ask user for clarification

            step += 1

        # Fallback: generate with whatever we have
        return await self._generate(query, context, fallback=True)

    async def _decide_action(self, query: str, context: list) -> dict:
        return await self.llm.generate_structured(
            f"""Given the query and current context, choose the next action.
Query: {query}
Context length: {len(context)} chunks
Current context: {context[-3:] if context else 'none'}

Options:
- {{"type": "retrieve", "tool": "vector_search", "sub_query": "..."}}
- {{"type": "retrieve", "tool": "web_search", "sub_query": "..."}}
- {{"type": "clarify", "question": "..."}}
- {{"type": "answer"}}

Next action:""",
            response_format=ActionSchema
        )
```

## Retrieval Strategies the Agent Can Choose

| Strategy | When | Tool |
|----------|------|------|
| Vector search | General knowledge | Qdrant/Pinecone |
| Web search | Current events | Tavily/Exa |
| SQL query | Structured data | PostgreSQL |
| Code execution | Computation | Python sandbox |
| Document crawl | Specific source | Custom scraper |
| Knowledge graph | Relationships | Neo4j |

## 🔴 Senior: Agentic RAG is NOT Always Better

| Scenario | Vanilla RAG | Agentic RAG |
|----------|------------|-------------|
| Simple Q&A | ✅ Faster, cheaper | ❌ Overkill |
| Multi-part questions | ❌ Bad | ✅ Good |
| Ambiguous queries | ❌ Bad | ✅ Clarifies |
| High throughput | ✅ Predictable | ❌ Variable latency |
| Debugging | ✅ Simple | ❌ Complex |

Use agentic RAG when you need to handle diverse, complex queries. Use vanilla RAG for simple, predictable scenarios.

## Drill: Agentic RAG From Scratch

Build an agent that has 3 tools:
1. `vector_search` — search your document corpus
2. `web_search` — search the web (use Tavily or DuckDuckGo)
3. `sql_query` — query a SQLite database

Test it on queries that require multiple tools:
- "What does the documentation say about X, and what's the latest news about it?"
- "Find similar incidents to this database error and tell me the resolution"
