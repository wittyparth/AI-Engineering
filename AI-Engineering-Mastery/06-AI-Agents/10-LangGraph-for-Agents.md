# LangGraph for Stateful Agents

## Why LangGraph

LangGraph is a low-level orchestration framework for building stateful, multi-actor agent workflows. Think of it as a DAG (Directed Acyclic Graph) execution engine with cycles, state management, and durable execution.

## Core Concepts

```python
from langgraph.graph import StateGraph, END, START
from typing import TypedDict, Literal

# 1. Define state
class AgentState(TypedDict):
    query: str
    documents: list[str]
    answer: str
    steps: int

# 2. Define nodes (just functions that modify state)
def retrieve(state: AgentState) -> AgentState:
    docs = vector_search(state["query"])
    return {**state, "documents": docs}

def generate(state: AgentState) -> AgentState:
    answer = llm.generate(state["query"], state["documents"])
    return {**state, "answer": answer, "steps": state["steps"] + 1}

# 3. Define edges (conditional routing)
def should_continue(state: AgentState) -> Literal["generate", "retry"]:
    if len(state["documents"]) == 0:
        return "retry"  # No docs → rewrite query
    return "generate"

# 4. Build graph
builder = StateGraph(AgentState)
builder.add_node("retrieve", retrieve)
builder.add_node("generate", generate)
builder.add_edge(START, "retrieve")
builder.add_conditional_edges("retrieve", should_continue)
builder.add_edge("generate", END)

# 5. Compile and run
graph = builder.compile()
result = graph.invoke({"query": "What is RAG?", "steps": 0})
```

## LangGraph for Multi-Agent

```python
from langgraph.graph import StateGraph, MessagesState

# Each agent can have its own tools and model
class ResearchState(MessagesState):
    research_notes: list[str]
    final_report: str

def research_agent(state: ResearchState) -> ResearchState:
    """Researcher agent with web search tool."""
    results = web_search(state["messages"][-1].content)
    state["research_notes"].extend(results)
    return state

def analysis_agent(state: ResearchState) -> ResearchState:
    """Analysis agent synthesizes research."""
    analysis = llm.generate(f"Synthesize: {state['research_notes']}")
    state["final_report"] = analysis
    return state

def human_review(state: ResearchState):
    """Optional human-in-the-loop checkpoint."""
    return {"needs_review": state["final_report"]}

builder = StateGraph(ResearchState)
builder.add_node("research", research_agent)
builder.add_node("analyze", analysis_agent)
builder.add_node("review", human_review)
builder.add_edge(START, "research")
builder.add_edge("research", "analyze")
builder.add_edge("analyze", "review")
builder.add_conditional_edges(
    "review",
    lambda s: "analyze" if not s.get("approved") else END,
)
```

## LangGraph vs Pydantic AI

| Feature | LangGraph | Pydantic AI |
|---------|-----------|-------------|
| State management | First-class TypedDict | RunContext + deps |
| Graph cycles | ✅ Native | ⚠️ Via pydantic-graph |
| Durable execution | ✅ Checkpoints | ✅ Durable execution |
| Human-in-the-loop | ✅ Built-in | ✅ Via hooks |
| Streaming | ✅ Via events | ✅ Native async |
| Type safety | ⚠️ TypedDict (runtime) | ✅ Full Pydantic (compile-time) |
| Learning curve | Steep (graph mental model) | Low (Pydantic mental model) |
| Best for | Complex state machines | Agent-as-a-function |

## 🔴 Senior: When to Use LangGraph

LangGraph is overkill for simple agents. Use it when:
1. You need durable execution (agent survives crashes)
2. You have complex branching logic (conditional routes)
3. You need human-in-the-loop checkpoints mid-execution
4. You want checkpoint/rewind capabilities
5. You're building a complex multi-agent system with shared state

For simple agents (tool call → respond): use Pydantic AI or a raw ReAct loop.
For complex state machines: use LangGraph.

## Drill: Port Your Custom Agent to LangGraph

1. Take your ReAct agent from scratch
2. Define its state as a TypedDict
3. Break the loop into discrete graph nodes (think, act, observe)
4. Add a conditional edge for retry logic
5. Add a human-in-the-loop checkpoint
6. Compare: which was easier to debug? Which was easier to extend?
