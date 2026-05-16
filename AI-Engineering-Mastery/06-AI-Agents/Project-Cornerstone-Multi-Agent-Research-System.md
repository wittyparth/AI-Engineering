# Cornerstone Project: Multi-Agent Research System

**Time**: 4-5 days  **Difficulty**: Hard  **Portfolio**: Yes

## The Problem

Build a system where multiple specialized AI agents collaborate to research a topic and produce a structured report. The orchestrator decomposes the task, specialist agents execute in parallel, and a writer synthesizes the output.

## Requirements

### Must Have
- [ ] Orchestrator agent that decomposes complex questions into sub-tasks
- [ ] Research agent (RAG over a document corpus + web search tool)
- [ ] Analysis agent that evaluates and synthesizes research findings
- [ ] Writing agent that produces structured reports with citations
- [ ] Shared state/context across agents (blackboard or message bus)
- [ ] Human-in-the-loop approval at key decision points
- [ ] Agent evaluation suite (task completion, step quality, efficiency)
- [ ] Streaming final answer generation

### Should Have
- [ ] Memory system (agents remember past research sessions)
- [ ] Feedback loop (agents can request clarification from user)
- [ ] Cost tracking per agent per session
- [ ] Failure recovery (if one agent fails, redistribute its work)
- [ ] Multi-format output (markdown, PDF, JSON)

### Nice to Have
- [ ] Swarm mode (agents bid on which subtasks they take)
- [ ] Custom agent personalities/styles
- [ ] Quality scoring per section (LLM-as-judge)
- [ ] Iterative refinement (re-write based on feedback)

## Architecture

```
User Query
    ↓
Orchestrator Agent
    ↓
┌──────────────────────────────────────────────┐
│ Research Agent │ Analysis Agent │ Writer      │
│ - Vector DB    │ - Fact checker │ - Outline   │
│ - Web search   │ - Cross-ref    │ - Draft     │
│ - Code search  │ - Gap finder   │ - Polish    │
└───────────────┴───────────────┴─────────────┘
    ↓
Human Review (optional)
    ↓
Final Report
```

## Tools Each Agent Needs

- **Orchestrator**: Task decomposition tool, agent handoff tool
- **Researcher**: RAG search, web search (Tavily/Exa), code search
- **Analyst**: LLM-as-judge, comparison tool, citation checker
- **Writer**: Report template tool, formatting tool

## 🔴 Senior: Production Considerations

1. **Agent timeouts**: Each agent has a max runtime. If exceeded, orchestrator picks up the partial work.
2. **Cost budgets**: Each research session has a token/cost budget. Agents must optimize.
3. **Citation requirements**: Every claim in the final report must cite a source.
4. **Version tracking**: Each research session produces a versioned report.
5. **Observability**: Langfuse tracing across ALL agents (not just one).

## Evaluation Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Task completion | 90%+ | Human eval of final report |
| Factual accuracy | >0.85 | LLM-as-judge + human spot-check |
| Citation quality | 95%+ valid | Automated citation checker |
| Report structure | Acceptable | Human rating 1-5 |
| Cost per report | <$0.50 | Token tracking |
| End-to-end time | <5 min | Wall clock |

## Design Decisions

1. **Agent communication**: Direct messages, shared blackboard, or queue?
2. **Parallel vs sequential**: Which subtasks run in parallel, which depend on others?
3. **Human involvement**: At what points should humans approve/review? Always or exceptions only?
4. **Report template**: Fixed structure or agent-decides structure?
5. **Error strategy**: If one agent fails, do we retry, redistribute, or degrade?

## Code Review Rubric

- Multi-agent coordination works reliably
- Each agent's purpose and boundaries are clear
- System handles agent failures gracefully
- Cross-agent communication is structured and type-safe
- Evaluations exist for task completion and efficiency
- Costs are tracked and bounded per session
- Final report quality exceeds single-agent baseline

## Reflection

1. What communication pattern worked best between agents?
2. How did you handle conflicting information from different agents?
3. What was the most common failure mode and how did you fix it?
4. How would you scale this to 20+ agents?
5. If you added a "reflection" agent, what would it check?
