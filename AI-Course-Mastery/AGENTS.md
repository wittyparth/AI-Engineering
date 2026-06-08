# AI Course Mastery — Agent Instructions

This file helps AI coding agents understand this course's structure, philosophy, and what high-quality output looks like.

---

## Course Structure

11 Phases, each building on the last. Every phase has:
- **Module Overview** — what, why, time budget, deliverables
- **Topic Files** — deep teaching with code
- **Drills** — focused 30-60 min exercises
- **Portfolio Project** — ships to GitHub
- **Gate Check** — must pass before advancing

## Output Quality Standards

Every file in this course must meet these bars:

1. **Production code only.** No toy examples. Every code block should be copy-paste-runnable and use proper error handling, typing, config management.

2. **Explain the WHY.** Don't just show code — explain why this approach, what the tradeoffs are, and when you'd choose differently.

3. **Show BAD and GOOD.** Every technique must include:
   - ✅ What good looks like (expected output, working example)
   - ❌ What bad looks like (common mistakes, failure modes, wrong output)

4. **Time budgets are real.** Each module has exact hours, not "~1 week vague". Daily breakdowns with deadlines.

5. **Cost awareness everywhere.** Every API call, every model choice, every infrastructure decision includes cost analysis.

6. **Senior vs Junior callouts.** Explicit callouts showing how a senior engineer thinks differently.

7. **Every project is portfolio-worthy.** Must have: GitHub-ready code, README with architecture decisions, and evaluation results.

## File Template (Mandatory)

```
🎯 PURPOSE & GOALS
⏱ TIME BUDGET & SCHEDULE  
📖 CONCEPT DEEP-DIVE
💻 CODE EXAMPLES
✅ GOOD OUTPUT EXAMPLES
❌ ANTIPATTERNS & FAILURE MODES
🧪 DRILLS & CHALLENGES
🚦 GATE CHECK
📚 RESOURCES
```

## Tech Stack Preferences

- **Backend**: FastAPI, Pydantic v2, async/await, httpx
- **AI SDKs**: OpenAI, Anthropic, Google Generative AI, LiteLLM
- **Agent Framework**: Pydantic AI (preferred), LangGraph (complex state machines)
- **RAG**: LangChain (prototyping), LlamaIndex (data-centric)
- **Prompt Optimization**: DSPy
- **Structured Outputs**: Instructor (API), Outlines (local)
- **Vector DB**: Qdrant (preferred), ChromaDB (dev), pgvector (if already using Postgres)
- **Embedding Models**: text-embedding-3-small/large, nomic-embed-text (local)
- **Serving**: vLLM (open models), OpenAI/Anthropic API (managed)
- **Deployment**: Docker Compose → AWS ECS/Fargate
- **Observability**: Langfuse, Prometheus, Grafana, OpenTelemetry
- **Caching**: Redis
- **Evals**: RAGAS, DeepEval, custom LLM-as-judge

## When Helping With This Course

1. **Read the module overview first** to understand the build narrative
2. **Follow the build loop**: Problem → Build Raw → Add Framework → Productionize → Evaluate → Debug
3. **Use production patterns**: proper error handling, typing, env config, testing
4. **Explain design decisions** — don't just code, tell WHY
5. **Call out senior-vs-junior thinking** at least once per file
6. **Warn about common mistakes** before the user makes them
7. **Include cost estimates** for every API-based solution
8. **Show what "good" output looks like** — don't just describe it
