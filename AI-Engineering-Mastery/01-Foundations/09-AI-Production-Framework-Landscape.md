# AI Production Framework Landscape (2026)

## The Big Picture

Raw concepts teach you *what* happens. Frameworks teach you *how to ship*. Use both. This file maps every production AI framework to where it fits in your stack.

## Framework Categories

### 1. Agent Frameworks (Build Agents)

| Framework | Philosophy | Best For | Provider Support | Key Strength |
|-----------|-----------|----------|-----------------|--------------|
| **Pydantic AI** | Type-safe, DI, minimal | Production agents with structured output | 30+ providers via LiteLLM | Type safety, DI, OTel built-in |
| **LangChain** | Ecosystem breadth | Prototyping, 1000+ integrations | 70+ providers | Largest ecosystem |
| **LangGraph** | Stateful DAG orchestration | Complex multi-agent state machines | Any via LangChain | Durable execution, checkpointing |
| **Deep Agents** | Batteries-included agents | Autonomous agents with planning | Any via LangChain | Planning, subagents, filesystem |
| **CrewAI** | Role-based agent teams | Multi-agent collaboration | 20+ providers | Role abstraction |
| **Griptape** | Enterprise governance | Regulated industries | 15+ providers | Trust boundaries |

### 2. RAG & Data Frameworks

| Framework | Philosophy | Best For | Key Strength |
|-----------|-----------|----------|-------------|
| **LlamaIndex** | Data-first RAG | Complex data ingestion + retrieval | 100+ data connectors, advanced RAG patterns |
| **LangChain** | General RAG | Quick RAG prototyping | Retrieval chains, document loaders |
| **SynapseKit** | Async-first, minimal | Performance-critical RAG | 2 deps only, async-native |

### 3. LLM Gateways & Proxies

| Framework | Philosophy | Best For | Key Strength |
|-----------|-----------|----------|-------------|
| **LiteLLM** | One interface, 100+ providers | Provider abstraction, cost tracking | Consistent API, 100+ providers |
| **OpenRouter** | Hosted API gateway | Pay-per-token, multi-model | No infrastructure needed |
| **Portkey** | Observability + gateway | Enterprise AI ops | A/B testing, guardrails |
| **Pydantic AI Gateway** | Unified LLM proxy | Spend controls, audit | Cloudflare edge, OTel |

### 4. LLM Serving

| Framework | Philosophy | Best For | Key Strength |
|-----------|-----------|----------|-------------|
| **vLLM** | High-throughput inference | Self-hosted LLMs | PagedAttention, continuous batching |
| **Ollama** | Local development | Running models locally | Easiest setup |
| **Llama Stack** | Standardized API server | Multi-model, multi-infra | OpenAI-compatible APIs |
| **TGI** | HuggingFace serving | HuggingFace models | Text Generation Inference |

### 5. Prompt Optimization

| Framework | Philosophy | Best For | Key Strength |
|-----------|-----------|----------|-------------|
| **DSPy** | Programming, not prompting | Automated prompt optimization | Compile-time optimization |
| **Outlines** | Constrained decoding | Guaranteed structured output | Generation-time constraints |

### 6. Observability & Evaluation

| Framework | Philosophy | Best For | Key Strength |
|-----------|-----------|----------|-------------|
| **Langfuse** | Open-source observability | Trace + eval together | Open source, self-host |
| **LangSmith** | Enterprise observability | LangChain ecosystem | Deep LangChain integration |
| **Pydantic Logfire** | OTel-native observability | OTel anywhere | No vendor lock-in, evals |
| **RAGAS** | RAG evaluation | RAG quality metrics | Specialized RAG metrics |

## Decision Tree

```
Starting a new AI project?

Do you need:
  → Type safety + structured output? → Pydantic AI
  → Fast prototyping + 1000 integrations? → LangChain
  → Complex stateful agent workflows? → LangGraph
  → Data-heavy RAG applications? → LlamaIndex
  → Just provider abstraction? → LiteLLM
  → Self-hosted model serving? → vLLM + Llama Stack
  → Prompt optimization without manual work? → DSPy
  → All of the above? → Start raw, add frameworks where needed
```

## 🔴 Senior: How to Use This Course + Framework

This course teaches you raw concepts FIRST in every phase, then adds frameworks. Here's why:

```
Phase N raw concepts → You understand WHAT the framework does internally
                ↓
Phase N + 1 frameworks → You understand WHY the framework exists
                ↓
Production project → You CHOOSE when to use framework vs raw
```

**Rule of thumb**: 
- Build without framework first (understand the problem)
- Add framework when you hit pain (too much boilerplate, need scaling)
- NEVER use a framework before you understand the problem it solves
- NEVER avoid a framework when it clearly saves time
