# 🧠 AI Engineering Mastery — The Definitive Course

### From Senior Software Engineer to Production AI Architect

**Designed for:** Partha — Senior Full-Stack Engineer (FastAPI, Next.js, AWS, Docker, Postgres)  
**Goal:** Build production-grade AI applications that ship to real users. Land an AI Engineering role.  
**Philosophy:** Learn by building — from scratch first, then with frameworks. Every concept has a drill, every module has a project, and everything ships.  
**Standard:** Adrian Cantrill depth + production engineering rigor + AI research currency  
**Duration:** ~8 months at 2–3 hours/day (flexible — accelerate or decelerate)  
**Total Modules:** 11 Phases → 90+ learning files → 11 portfolio projects  

---

> *"The engineers extracting real value from AI aren't those with the most innovative demos — they're the ones doing the less glamorous engineering work: building evaluation pipelines, implementing guardrails, designing for uncertainty, and treating LLM systems with the same rigour they'd apply to any critical infrastructure."*
> — ZenML, analysis of 1,200 production LLM deployments (2025)

---

## 📋 What Makes This Course Different

| Typical AI Tutorial | This Course |
|---------------------|-------------|
| Teaches framework syntax | Teaches **DEEP principles** — understand WHY before HOW |
| Toy projects (chatbots, todo lists) | Production systems (gateways, RAG engines, multi-agent platforms) |
| Skips evaluation | **Evaluation in EVERY phase** — if it can't be measured, it's not done |
| Skips debugging | **Debug challenges in EVERY module** — broken code you must fix |
| Focuses on code only | Teaches **SENIOR thinking** — tradeoffs, cost, architecture, failure modes |
| You watch/read | You **BUILD** — every phase produces a portfolio-worthy project |
| Separate concept + framework | **Frameworks integrated into the build narrative** |
| No time boundaries | **Exact time budgets** per module — real-work pressure |
| One-shot learning | **Spaced repetition** built in — revision checkpoints weave through |

---

## 🗺️ Course Map — 11 Phases

```
PHASE 0: AI ENGINEERING MINDSET (Week 1)
  Install the mental models of a senior AI engineer before writing code

PHASE 1: FOUNDATIONS & LLM GATEWAY (Weeks 2–3)
  How LLMs work → Tokenizers → Context windows → APIs → Streaming → Build a Multi-Provider LLM Gateway

PHASE 2: PROMPT ENGINEERING SYSTEMATIC (Weeks 4–5)
  Context engineering → Techniques (zero/few/CoT) → System prompts → Red-teaming → Evaluation → DSPy → Build a Prompt Engineering System

PHASE 3: EMBEDDINGS & VECTOR DATABASES (Weeks 6–7)
  Embeddings deep dive → Semantic search from scratch → ChromaDB → Pinecone/Qdrant → pgvector → Build a Personal Knowledge Search Engine

PHASE 4: RAG FOUNDATIONS (Weeks 8–10)
  Naive RAG → Chunking strategies → Retrieval → Hybrid search → Reranking → Failure modes → Build a Production Customer Support Bot

PHASE 5: ADVANCED RAG (Weeks 11–13)
  HyDE → Contextual retrieval → Agentic RAG → Multimodal RAG → GraphRAG → RAGAS evaluation → Build an Advanced RAG Engine

PHASE 6: AI AGENTS (Weeks 14–17)
  Agent foundations → ReAct loop → Tool design → Memory systems → LangGraph → Multi-agent → MCP → Agent evaluation → Build a Multi-Agent Research System

PHASE 7: PRODUCTION EVALS & OBSERVABILITY (Weeks 18–20)
  Evals as unit tests → LLM-as-judge → Langfuse tracing → Custom eval framework → CI/CD for AI → Regression testing → Build an Eval Platform

PHASE 8: GUARDRAILS, SAFETY & SECURITY (Weeks 21–22)
  Input guardrails → Output guardrails → Prompt injection defense → PII redaction → Content moderation → Build a Safe AI Gateway

PHASE 9: FINE-TUNING & LLMOPS (Weeks 23–25)
  When to fine-tune → LoRA deep dive → Unsloth → Dataset preparation → Evaluation → Build a Fine-Tuned Custom Model

PHASE 10: DEPLOYMENT & SCALE (Weeks 26–28)
  Model serving (vLLM) → Docker Compose → AWS Bedrock → Caching (Redis) → Cost optimization → Latency optimization → Build Production Infrastructure

PHASE 11: CAPSTONE (Weeks 29–32)
  Plan → Design → Build → Evaluate → Deploy → Your own production AI product
```

---

## 🛠️ Your Tech Stack (By End of Course)

**Core AI & LLMs:**
- OpenAI API (GPT-4o, GPT-4o-mini, text-embedding-3-small/large)
- Anthropic Claude API (Claude 3.5 Sonnet, Claude 3 Haiku)
- Google Gemini API (Gemini 1.5 Pro/Flash)
- Ollama (local LLMs — Llama 3, Mistral, Phi)
- LiteLLM (provider abstraction)

**Prompt Engineering & Optimization:**
- DSPy (programmatic prompt optimization)
- Instructor (structured outputs across providers)
- Outlines (local structured generation)

**Frameworks & Orchestration:**
- LangChain (RAG prototyping)
- LlamaIndex (data-centric RAG)
- LangGraph (stateful agents)
- Pydantic AI (agent framework — preferred)

**Vector Databases:**
- ChromaDB (development)
- Qdrant (production — preferred)
- Pinecone (managed cloud)
- pgvector (PostgreSQL integration)
- FAISS (local high-performance)

**Observability & Evaluation:**
- Langfuse (open-source tracing)
- RAGAS (RAG evaluation)
- DeepEval (testing framework)
- Custom LLM-as-judge pipelines

**Infrastructure & Deployment:**
- FastAPI (AI backends)
- Docker Compose → AWS ECS
- vLLM (model serving)
- Redis (caching)
- Prometheus + Grafana (monitoring)

**Frontend (you already own this):**
- Next.js + streaming responses
- Vercel AI SDK
- Server-Sent Events (SSE)

---

## 📐 Course Architecture — Every File Follows This Template

Every topic file in this course is structured the same way. Master this structure once, and every module becomes predictable:

```
┌──────────────────────────────────────────────────┐
│ 🎯 PURPOSE & GOALS                               │
│   What you will achieve. Stated concretely.      │
├──────────────────────────────────────────────────┤
│ ⏱ TIME BUDGET & SCHEDULE                         │
│   Exact hours. Daily breakdown. Deadlines.       │
├──────────────────────────────────────────────────┤
│ 📖 CONCEPT DEEP-DIVE                             │
│   The teaching. Why this matters. Mental models. │
├──────────────────────────────────────────────────┤
│ 💻 CODE EXAMPLES                                 │
│   Production-grade. Full working code.           │
├──────────────────────────────────────────────────┤
│ ✅ GOOD OUTPUT EXAMPLES                          │
│   What "A+ work" looks like.                    │
├──────────────────────────────────────────────────┤
│ ❌ ANTIPATTERNS & FAILURE MODES                  │
│   What breaks. What NOT to do.                  │
├──────────────────────────────────────────────────┤
│ 🧪 DRILLS & CHALLENGES                           │
│   Hands-on practice. 30-min exercises.          │
├──────────────────────────────────────────────────┤
│ 🚦 GATE CHECK                                    │
│   Prove competence before advancing.             │
├──────────────────────────────────────────────────┤
│ 📚 RESOURCES                                     │
│   Go deeper. Videos, papers, repos, docs.        │
└──────────────────────────────────────────────────┘
```

---

## 🚀 Quick Start

```
git clone <this-repo>
cd AI-Course-Mastery

# Start here
start 00-Engineering-Mindset/00-Module-Overview.md

# Then set up your environment
# Then begin Phase 1
```

---

## 📊 Progress Tracking

This course includes built-in tracking for every phase:

- **🎯 Purpose Check** — Did you achieve what this module promised?
- **⏱ Time Log** — How long did each section actually take? (Self-awareness)
- **🧪 Drill Completion** — Did you finish all drills?
- **🏗️ Project Gate** — Did the project ship to GitHub with a README?
- **📝 Learning Journal** — What surprised you? What confused you?

Each module's Gate Check serves as your progression bar. **Do not skip gates.**

---

## 🔁 Spaced Repetition Built In

Concepts from earlier phases reappear in later phases:
- Embeddings (Phase 3) → used in RAG (Phase 4) → used in Agent memory (Phase 6) → evaluated (Phase 7)
- Prompt engineering (Phase 2) → system prompts for agents (Phase 6) → eval prompts (Phase 7) → fine-tuning data (Phase 9)
- Cost awareness (Phase 0) → calculated in every project → optimized in Phase 10

---

## 🏆 Portfolio By End

| # | Project | Phase | What It Shows |
|---|---------|-------|--------------|
| 1 | Multi-Provider LLM Gateway | Phase 1 | API integration, provider abstraction, streaming |
| 2 | Prompt Engineering System | Phase 2 | Systematic prompting, evaluation, optimization |
| 3 | Personal Knowledge Search Engine | Phase 3 | Embeddings, vector search, semantic retrieval |
| 4 | Production Customer Support Bot | Phase 4 | Full RAG pipeline, chunking, reranking |
| 5 | Advanced RAG Engine | Phase 5 | HyDE, agentic RAG, multimodal, GraphRAG |
| 6 | Multi-Agent Research System | Phase 6 | LangGraph, multi-agent orchestration, MCP |
| 7 | AI Eval Platform | Phase 7 | Evals, observability, CI/CD integration |
| 8 | Safe AI Gateway | Phase 8 | Guardrails, PII redaction, content moderation |
| 9 | Fine-Tuned Custom Model | Phase 9 | LoRA, Unsloth, dataset prep, evaluation |
| 10 | Production Infrastructure | Phase 10 | vLLM, Docker, AWS, Redis, monitoring |
| 11 | Capstone (Your Choice) | Phase 11 | End-to-end production AI product |

---

## ⚠️ Ground Rules

1. **Never copy-paste code.** Type every line. Break it. Fix it. Own it.
2. **Every project ships to GitHub** with a README explaining architecture decisions.
3. **If you can't explain it simply, you don't understand it.** Teach it back.
4. **Build in public.** Document your journey. This is how you get hired.
5. **The goal is depth, not speed.** A shallow course taken fast is worthless.
6. **Don't skip the failure modes.** Production breaks in ways tutorials don't show.
7. **Measure everything.** If it can't be measured, it can't be improved.

---

*"The best AI engineer isn't the one who can prompt the best — it's the one who can build systems that work reliably, cost-effectively, and measurably."*

---

[Next: 00-Engineering-Mindset/00-Module-Overview.md →]
