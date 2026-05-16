# Phase 6: Build a Multi-Agent System

**The Problem**: A single LLM call can't research, analyze, write, and fact-check a complex report. You need MULTIPLE specialized agents collaborating — like how Klarna's AI assistant handles 2.5M conversations with 700 FTEs worth of work.

**What You'll Build**: A multi-agent research system using ReAct (from scratch), Pydantic AI (type-safe agents), LangGraph (stateful orchestration), and LiteLLM (provider abstraction). You'll understand EVERY layer of production agent architecture.

**Difficulty**: Hard | **Time**: ~18 hours

## The Build Path

```
1. PROBLEM: "Single LLM calls can't do complex multi-step work reliably"
2. BUILD RAW: ReAct agent from scratch — observe → think → act loop
3. BUILD RAW: Tool design patterns, error recovery, infinite loop prevention
4. ADD FRAMEWORK: Pydantic AI — type-safe agents with DI, OTel, structured output
5. ADD FRAMEWORK: LangGraph — state machines, checkpointing, human-in-the-loop
6. ADD FRAMEWORK: LiteLLM — 100+ providers, routing, cost tracking
7. BUILD RAW: Multi-agent patterns — supervisor, orchestrator, swarm
8. FRAMEWORK COMPARISON: Same agent, 3 implementations (raw, Pydantic AI, LangGraph)
9. PRODUCTIONIZE: MCP (Model Context Protocol) — standardized tool interface
10. EVALUATE: Agent evaluation — task completion, trajectory quality, cost
11. DEBUG: Agent in infinite loop — find and fix the bug
```

## Concepts (Built Into the Flow)

| # | Step | What You Learn | Why It Matters |
|---|------|---------------|----------------|
| 1 | Build Raw | Agent architecture patterns, ReAct loop | Understand how agents WORK internally |
| 2 | Build Raw | Tool design, error handling, rate limiting | Tools are the agent's interface to the world |
| 3 | Build Raw | Memory systems (short-term, long-term, episodic) | Agents without memory are useless |
| 4 | Add Framework | **Pydantic AI** — type-safe agents | Compile-time validation, DI, OTel built-in |
| 5 | Add Framework | **LangGraph** — stateful workflows | Checkpointing, HITL, durable execution |
| 6 | Add Framework | **LiteLLM** — provider abstraction | 100+ providers, routing, cost tracking |
| 7 | Build Raw | Multi-agent orchestration | Communication, delegation, consensus |
| 8 | Productionize | MCP (Model Context Protocol) | Standardized tool interface (industry standard) |
| 9 | Evaluate | Agent evaluation | The hardest unsolved problem in AI engineering |
| 10 | Debug | Infinite loops, tool failures, hallucination cascades | Real agent failures |

## Framework Integration

| Framework | When It Shines | Used Here |
|-----------|---------------|-----------|
| **Pydantic AI** | Type-safe single agents with structured output | Core agent logic, DI for testing |
| **LangGraph** | Complex state machines, HITL, checkpointing | Orchestration, multi-agent coordination |
| **LiteLLM** | Provider abstraction, cost tracking | Every agent uses it for model calls |

## The Project: Multi-Agent Research System

A system with:
- Orchestrator agent (breaks down the task)
- Research agent (searches documents + web)
- Analysis agent (synthesizes findings)
- Writing agent (produces structured reports)
- Fact-checking agent (verifies claims)
- Human-in-the-loop for verification
- LiteLLM Router for cost-optimized model selection
- LangGraph for state management and checkpointing

**This is the architecture Klarna, Vodafone, and Uber use in production.**

## 🔴 Senior: Agent Maturity Model

```
Level 1: Single tool call (function calling)
Level 2: Sequential tool calls (deterministic)
Level 3: ReAct loop (LLM decides next action)
Level 4: Stateful agents (memory + context)
Level 5: Multi-agent (specialized agents collaborate)
Level 6: Self-improving (agents evaluate and refine)
```

Most production systems are at Level 2-3. This phase takes you to Level 5.

## Gate Check

- [ ] Built ReAct agent from scratch (no frameworks)
- [ ] Pydantic AI agent with DI passes tests without API calls
- [ ] LangGraph state machine handles HITL workflow
- [ ] Multi-agent system completes a task no single agent could
- [ ] LiteLLM Router reduces costs by 40%+ vs using GPT-4o for everything
- [ ] Agent evaluation pipeline catches regression
- [ ] Debug challenge: found the infinite loop cause
