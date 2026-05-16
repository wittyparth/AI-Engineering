# AI Engineering Mastery — Agent Instructions

This file helps AI coding agents understand this course's structure and the user's preferences.

## Course Structure

- **Phase 0**: AI Engineering Mindset (mental models, decision frameworks)
- **Phase 1**: Build an LLM Gateway (Foundations + LiteLLM)
- **Phase 2**: Build a Prompt Engineering System (Prompting + DSPy)
- **Phase 3**: Build a Semantic Search Engine (Embeddings + Vector DBs)
- **Phase 4**: Build a Production RAG Engine (RAG + LangChain + LlamaIndex)
- **Phase 5**: Build an Advanced Knowledge System (Adv RAG + LlamaIndex Workflows)
- **Phase 6**: Build a Multi-Agent System (Agents + Pydantic AI + LangGraph + LiteLLM)
- **Phase 7**: Fine-Tuning & LLMOps
- **Phase 8**: Deploy Everything to Production (vLLM + Llama Stack + AWS)
- **Phase 9**: Build an Observability & Evaluation Platform (Langfuse + RAGAS)
- **Phase 10**: Advanced Topics (Outlines + Instructor + Multimodal + Security)
- **Phase 11**: Capstone

## Core Philosophy

This course teaches DEEP principles first, then shows how frameworks solve those problems. The user should understand WHY before learning HOW. Every phase follows:

```
PROBLEM → BUILD RAW → ADD FRAMEWORKS → PRODUCTIONIZE → EVALUATE → DEBUG
```

## User Proficiency

- Expert: Python, FastAPI, Docker, AWS, full-stack, Postgres, Git
- Familiar: Redis, CI/CD, Kubernetes (basic)
- Learning: AI Engineering (LLMs, RAG, agents, embeddings, fine-tuning, production deployment)

## Learning Preferences

- **Build production systems, not tutorials** — every project must be portfolio-worthy
- **From scratch first** — build without frameworks before using them
- **Production mindset** — cost, latency, security, monitoring matter
- **Hard mode** — challenge the user, don't dumb things down
- **Senior callouts** — always highlight junior vs senior thinking
- **Evaluation** — if it can't be measured, it's not done
- **Frameworks integrated, not separate** — frameworks should emerge from the build narrative

## When Helping

1. **Read the phase overview first** to understand the build narrative
2. **Follow the build loop** — Problem → Raw → Framework → Productionize → Evaluate → Debug
3. **Use production patterns** — proper error handling, typing, config, testing
4. **Explain design decisions** — don't just write code, tell WHY
5. **Call out senior-vs-junior thinking** when relevant
6. **Warn about common mistakes** before the user makes them
7. **Prefer FastAPI**, async patterns, and clean architecture
8. **Help measure** — suggest metrics and evals for whatever is being built
9. **Integrate frameworks into the narrative** — don't treat them as separate topics

## Stack Preferences

- **Backend**: FastAPI, Pydantic, async, httpx
- **AI**: OpenAI, Anthropic, Ollama, sentence-transformers
- **Agent Frameworks**: Pydantic AI (preferred), LangGraph (for complex state machines)
- **RAG Frameworks**: LangChain (prototyping), LlamaIndex (data-centric)
- **Provider Abstraction**: LiteLLM
- **Prompt Optimization**: DSPy
- **Structured Generation**: Outlines (local), Instructor (API)
- **Model Serving**: vLLM, Llama Stack
- **Vector DB**: Qdrant (preferred) or pgvector
- **Deployment**: Docker Compose → AWS ECS
- **Observability**: Langfuse, Prometheus, Grafana, OpenTelemetry
- **Caching**: Redis
- **Evals**: RAGAS, custom LLM-as-judge
