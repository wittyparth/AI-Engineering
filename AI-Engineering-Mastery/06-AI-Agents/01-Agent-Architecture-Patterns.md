# Agent Architecture Patterns

## 1. Single Agent (Simplest)

```
User → Agent → Tool → Agent → User
```

One agent with multiple tools. Good for simple automation.

**When**: Single domain, well-defined tasks, few tools.

## 2. Supervisor Pattern

```
User → Supervisor Agent
         ├── Specialist Agent A (tools: ...)
         ├── Specialist Agent B (tools: ...)
         └── Specialist Agent C (tools: ...)
      → Supervisor → User
```

Supervisor delegates to specialists, synthesizes results.

**When**: Multi-domain tasks, need for specialization.

## 3. Orchestrator-Worker

```
User → Orchestrator
         ├── Planner: breaks task into subtasks
         ├── Workers: execute subtasks (parallel)
         └── Synthesizer: combines results
      → User
```

**When**: Complex tasks that can be parallelized.

## 4. Pipeline

```
User → Agent 1 (research) → Agent 2 (analyze) → Agent 3 (write) → User
```

Each agent does one thing and passes to the next.

**When**: Deterministic workflows, each step depends on previous.

## 5. Swarm

```
User → Agent A ←→ Agent B ←→ Agent C
         ↓           ↓           ↓
        Tool        Tool        Tool
```

Agents communicate freely, no central controller.

**When**: Emergent behavior desired, open-ended problems.

## 🔴 Senior: Pattern Selection Guide

| Pattern | Complexity | Reliability | Flexibility | Best For |
|---------|-----------|-------------|-------------|----------|
| Single | Low | High | Low | Simple automation |
| Supervisor | Medium | High | Medium | Domain routing |
| Orchestrator | High | Medium | High | Complex tasks |
| Pipeline | Medium | High | Low | Fixed workflows |
| Swarm | Very High | Low | Very High | Research/experimental |

For production: start with single or supervisor. Only move to more complex patterns when the simpler one fails.

## 🔧 Framework Integration: Agent Frameworks in This Phase

You'll build agents from scratch first (ReAct loop, tool design, memory). Then add frameworks:

| Framework | Best For | When to Use |
|-----------|----------|-------------|
| **Pydantic AI** | Type-safe single agents with structured output | Agent-as-a-function with DI, OTel, validation |
| **LangGraph** | Complex state machines, HITL, checkpointing | Multi-agent orchestration, durable execution |
| **LiteLLM** | Provider abstraction, cost tracking, routing | Every agent should use this for model calls |
| **CrewAI** | Role-based multi-agent teams | Business process automation with specialized roles |

**The pro stack**: Pydantic AI (agent logic) + LangGraph (orchestration) + LiteLLM (provider abstraction).

**Deep dives**: See `09-Pydantic-AI-Agents.md`, `10-LangGraph-for-Agents.md`, and `11-LiteLLM-Provider-Abstraction.md`.
