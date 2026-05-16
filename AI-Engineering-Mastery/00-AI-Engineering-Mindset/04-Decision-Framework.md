# Decision Framework: When to Build Raw vs Use Frameworks

## The Core Principle

> **Build from scratch until the pain of building from scratch exceeds the pain of learning a framework.**
> **Use a framework until the pain of the framework exceeds the pain of raw code.**

## Decision Tree

```
Starting a new AI feature:

1. Do I understand the fundamental concept?
   ├── No → Build from scratch first (learn the concept)
   └── Yes → Go to 2

2. Is this a common pattern (RAG, agent, chatbot)?
   ├── Yes → Use a framework (LangChain, Pydantic AI, LangGraph)
   ├── Novel/experimental → Build from scratch (flexibility matters)
   └── Simple API call → No framework needed (direct API)

3. Do I need production reliability?
   ├── Yes → Use framework with proven production track record
   ├── Prototype → Build from scratch for learning, then port to framework
   └── Learning → Build from scratch (this is a course, not a startup)

4. Do I need framework-specific features?
   ├── Type safety, DI, OTel → Pydantic AI
   ├── Complex state machines, HITL → LangGraph
   ├── 100+ integrations, fast prototype → LangChain
   ├── Data-heavy RAG → LlamaIndex
   ├── Multi-provider abstraction → LiteLLM
   └── Prompt optimization → DSPy
```

## When to Build Raw (Even in Production)

| Scenario | Why Raw |
|----------|---------|
| You need < 3 tools/endpoints | Framework overhead isn't worth it |
| The framework's abstraction doesn't fit | Fighting the framework costs more |
| You need max performance | Framework overhead matters at scale |
| You're learning | You won't understand the framework until you've felt the pain |
| The problem is unique | No framework has a pre-built solution |

## When to Use a Framework

| Scenario | Why Framework |
|----------|--------------|
| Common pattern (RAG, agent, chatbot) | Don't reinvent the wheel |
| Need 10+ integrations | Framework already built them |
| Team velocity matters | Less code to write and maintain |
| You need specific features | Checkpointing, HITL, DI, OTel |
| The framework has solved edge cases you haven't encountered | Years of community testing |

## The Framework Maturity Model

```
Level 0: Raw Python + direct API calls (no framework)
Level 1: Light wrapper (function to call LLM + parse response)
Level 2: Single-purpose framework (just RAG, just agents)
Level 3: Full ecosystem framework (LangChain, Pydantic AI)
Level 4: Composed framework stack (Pydantic AI + LangGraph + LiteLLM)
Level 5: Custom framework (built for your specific use case)
```

**Senior engineers move fluidly between levels.**

## Course Pattern

Every phase in this course follows:
1. **Build Raw** — Understand the concept by building without frameworks
2. **Pain Emerges** — You'll feel exactly why frameworks exist
3. **Add Framework** — Now the framework makes sense (you know what problem it solves)
4. **Productionize** — Add monitoring, cost controls, security
5. **Evaluate** — Measure and improve
6. **Debug** — Find and fix the common failure modes

## Reflection

- For your last project, which level of the maturity model were you at?
- Would a framework have saved you time, or added complexity?
- When should you drop the framework and go raw?
