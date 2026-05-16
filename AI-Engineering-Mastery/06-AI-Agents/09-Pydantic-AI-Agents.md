# Pydantic AI: Production Agent Framework

## Why Pydantic AI

Think FastAPI for AI agents. Same philosophy: type safety, dependency injection, automatic validation, and OTel observability out of the box.

Built by the Pydantic team — the validation layer already used by OpenAI SDK, Anthropic SDK, LangChain, LlamaIndex, and most other frameworks.

## Core Concepts

### Agent Definition
```python
from pydantic_ai import Agent, RunContext
from pydantic import BaseModel
from dataclasses import dataclass

# 1. Define dependencies (injected via DI)
@dataclass
class SupportDeps:
    db: Database
    customer_id: int

# 2. Define structured output
class SupportResult(BaseModel):
    ticket_id: int
    status: str
    resolution: str | None = None

# 3. Define agent
support_agent = Agent(
    model="openai:gpt-4o",
    deps_type=SupportDeps,
    result_type=SupportResult,
    system_prompt="You are a support agent...",
)
```

### Dependency Injection
```python
@support_agent.system_prompt
async def add_customer_name(ctx: RunContext[SupportDeps]) -> str:
    name = await ctx.deps.db.get_customer_name(ctx.deps.customer_id)
    return f"Customer name: {name}"

@support_agent.tool
async def get_balance(ctx: RunContext[SupportDeps]) -> float:
    """Get the customer's current balance."""
    return await ctx.deps.db.get_balance(ctx.deps.customer_id)

# Run with dependencies injected
deps = SupportDeps(db=Database(), customer_id=42)
result = await support_agent.run("What's my balance?", deps=deps)
# result.data is a typed SupportResult
```

## Key Features

### Structured Outputs (Compile-Time Safe)
```python
class Invoice(BaseModel):
    invoice_number: str
    total: float
    line_items: list[dict]

agent = Agent("openai:gpt-4o", result_type=Invoice)
result = await agent.run("Extract from email...")
# result.data is GUARANTEED to be valid Invoice
# Type checker catches schema mismatches at write time
```

### Streaming
```python
async with agent.run_stream("Tell me a story") as result:
    async for chunk in result.stream():
        print(chunk, end="", flush=True)
    # streamed_text = await result.get_text()
```

### Multi-Agent Workflows
```python
from pydantic_ai import Agent

router = Agent("openai:gpt-4o")
research = Agent("anthropic:claude-sonnet-4")
writer = Agent("openai:gpt-4o")

@router.tool
async def delegate_research(topic: str) -> str:
    """Research a topic and return findings."""
    result = await research.run(f"Research: {topic}")
    return result.data

@router.tool
async def delegate_writing(outline: str) -> str:
    """Write a report from an outline."""
    result = await writer.run(f"Write report: {outline}")
    return result.data
```

## Pydantic AI vs LangChain vs Raw

| Aspect | Pydantic AI | LangChain | Raw API |
|--------|-------------|-----------|---------|
| Type safety | ✅ Compile-time validation | ⚠️ Runtime validation | ❌ Manual |
| Dependency injection | ✅ Built-in | ❌ Manual wiring | ❌ Manual |
| Structured output | ✅ Declarative BaseModel | ⚠️ OutputParser + validation | ❌ JSON parsing |
| Model switching | ✅ Change string | ✅ Change config | ❌ Rewrite client |
| Observability | ✅ Built-in OTel | ✅ LangSmith | ❌ Manual |
| Testing | ✅ TestModel (zero API) | ⚠️ Mock chains | ❌ Mock HTTP |
| Learning curve | Low (Pydantic knowledge) | Steep | Low |
| Bundle size | Minimal | Heavy | Minimal |

## 🔴 Senior: When to Use Pydantic AI

**Use it when:**
- You want type-safe agents (structured outputs guaranteed)
- You need dependency injection for testability
- You already love Pydantic and FastAPI
- You want OTel observability without vendor lock-in
- You're building a new production system from scratch

**Don't use it when:**
- You need a specific LangChain integration not yet supported
- You want LangGraph's mature state machine for complex agents
- Your team already has deep LangChain expertise and production code

## Drill: Convert Your ReAct Agent to Pydantic AI

Take your ReAct agent from earlier and port it to Pydantic AI:
1. Define dependencies as a dataclass
2. Convert tools to `@agent.tool` decorated functions
3. Define a structured output model
4. Add a `TestModel` to verify without API calls
5. Compare lines of code and error handling

```python
from pydantic_ai import Agent
from pydantic_ai.models.test import TestModel

# Test agent runs deterministically — zero API calls
test_agent = Agent(
    TestModel(),
    result_type=str,
    system_prompt="Test agent",
)
result = test_agent.run_sync("Test query")
assert result.data == "Test query"  # TestModel echoes input
```
