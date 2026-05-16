# Stateful Agents with LangGraph

## Why State Matters

Basic agents are loops without state. LangGraph gives you:
- **State**: Shared data across agent steps
- **Nodes**: Individual processing steps
- **Edges**: Conditional transitions between nodes
- **Cycles**: Loops with termination conditions

## Basic Graph

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, List

class AgentState(TypedDict):
    query: str
    retrieved_docs: List[str]
    generated_answer: str
    step_count: int

def retrieve(state: AgentState) -> AgentState:
    docs = vector_search(state["query"])
    return {**state, "retrieved_docs": docs, "step_count": state["step_count"] + 1}

def generate(state: AgentState) -> AgentState:
    answer = llm.generate(state["query"], state["retrieved_docs"])
    return {**state, "generated_answer": answer}

def check_quality(state: AgentState) -> str:
    """Conditional edge: decide next step."""
    if len(state["retrieved_docs"]) == 0:
        return "retry_retrieval"  # Go back to retrieve
    return "proceed"  # Go to generate

# Build graph
graph = StateGraph(AgentState)
graph.add_node("retrieve", retrieve)
graph.add_node("generate", generate)
graph.add_edge("retrieve", "generate", condition=check_quality)
graph.set_entry_point("retrieve")
graph.add_edge("generate", END)
```

## Multi-Agent with LangGraph

```python
class MultiAgentState(TypedDict):
    task: str
    research_notes: List[str]
    analysis: str
    report: str
    approved: bool

research_node = ResearchAgent()
analysis_node = AnalysisAgent()
writing_node = WritingAgent()
review_node = HumanReviewAgent()

builder = StateGraph(MultiAgentState)
builder.add_node("research", research_node)
builder.add_node("analyze", analysis_node)
builder.add_node("write", writing_node)
builder.add_node("review", review_node)

builder.add_edge("research", "analyze")
builder.add_edge("analyze", "write")
builder.add_edge("write", "review")

# Conditional: if not approved, go back to write
builder.add_conditional_edges(
    "review",
    lambda state: "write" if not state["approved"] else END,
)
```

## 🔴 Senior: LangGraph vs Custom

| Factor | LangGraph | Custom Loop |
|--------|-----------|-------------|
| Complexity | Medium | Low |
| State management | Built-in | Manual |
| Persistence | Built-in (checkpoints) | Manual |
| Debugging | LangSmith tracing | Print statements |
| Flexibility | Framework constraints | Full control |
| Learning curve | Medium | Low |

Start with a custom loop. Move to LangGraph when you need checkpointing, persistence, or complex branching.
