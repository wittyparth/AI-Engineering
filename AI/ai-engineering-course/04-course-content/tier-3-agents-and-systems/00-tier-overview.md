# Tier 3: Real Constraints — Agents & Production Systems

> *"Clients now have production constraints: latency budgets, cost ceilings, accuracy requirements. Maya gives less hand-holding."*

## Why Tier 3 Exists

Tier 2 taught you RAG — the most common pattern in production AI. Tier 3 teaches you **agents** — LLMs that can **do things**, not just **answer things**.

Instead of "retrieve text and generate an answer," an agent can:
- Search the web, read pages, and synthesize findings
- Write and execute code, observe results, and iterate
- Call APIs, transform data, and take actions
- Decide what to do next based on changing information

**This is where you stop being a "RAG engineer" and become an "AI Engineer."**

## What Changes From Tier 2

| Dimension | Tier 2 | Tier 3 |
|---|---|---|
| **Pattern** | RAG (passive retrieval) | Agents (active tool use + reasoning) |
| **Guidance** | Goal + hints from Maya | Minimal — "here's the problem, figure it out" |
| **Constraints** | Cost + accuracy | Latency + cost + accuracy + reliability + security |
| **Framework** | Build from scratch first | Framework from start (you've earned it) |
| **Evaluation** | Basic metrics + manual spot-check | Automated eval suites with CI integration |
| **Observability** | Logging | Full tracing stack (Langfuse) |
| **Rohan's bar** | Points 1-4 (works + edge cases + cost + observable) | All 7 points, every time |

## The 5 Projects

| Project | What You Build | The Real Lesson |
|---|---|---|
| **8** | Real-Time Sales Intelligence Agent | Agent loop design, tool use, latency optimization (<3s) |
| **9** | Advanced Financial RAG System | Agentic RAG, query rewriting, vision for tables, anti-hallucination |
| **10** | GitHub Code Review Agent | Tool use (GitHub API), structured code analysis, precision/recall eval |
| **11** | Natural Language to SQL Agent | Text-to-SQL, schema grounding, safe execution, query validation |
| **12** | Cost-Aware Model Router (Production) | Production routing, semantic caching, fallback chains, cost dashboards |

## The Agent Design Principles (Anthropic, 2025-2026)

Based on Anthropic's field research analyzing hundreds of production agent deployments:

1. **The most effective agents are simple** — augmented LLMs doing one thing well
2. **Most production systems use prompt chaining** (deterministic), not autonomous agents
3. **Multi-agent is overkill** until a single agent demonstrably fails
4. **The key to good agents is well-designed tools**, not clever prompting
5. **Hard limits on steps and cost** are non-negotiable in production

## What You'll Be Able to Do After Tier 3

- ✅ Build agents with tool use and multi-step reasoning (ReAct loop)
- ✅ Implement advanced RAG patterns (HyDE, agentic retrieval, parent-child chunking)
- ✅ Design and implement tool schemas (MCP servers)
- ✅ Set up full observability stack (Langfuse tracing, monitoring, alerts)
- ✅ Implement cost-aware systems with semantic caching and model routing
- ✅ Deploy with CI/CD, auth, and security
- ✅ Evaluate agent systems with appropriate metrics

**Portfolio:** 5 production-grade projects with tracing, evaluation, and documentation

> *Maya's voice: "In Tier 1 you called the API. In Tier 2 you architected the system. In Tier 3 you build things that THINK. This is where it gets fun."*
