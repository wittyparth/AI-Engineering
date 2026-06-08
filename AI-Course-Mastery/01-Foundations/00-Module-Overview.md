# Phase 1: Foundations — How LLMs Work & Your First AI Gateway

## 🎯 Purpose & Goals

Welcome to the first real building phase. By now you have the **senior engineer mindset** installed. Now we learn what's actually happening under the hood of these models, and build your first production AI system.

**By the end of this phase, you will:**
- Understand how LLMs actually work (token prediction, attention, embeddings — no math degree required)
- Be able to count tokens, estimate costs, and choose the right model for any task
- Write async, streaming, error-resilient Python code for AI systems
- Have a working **Multi-Provider LLM Gateway** that can use OpenAI, Anthropic, or local models with one config change
- Understand the production framework landscape (LangChain, LlamaIndex, LiteLLM, DSPy) and when to use what

**⏱ Total Time Budget:** 20–25 hours over 10 days

| Day | Topic | Time | ✅ Done |
|-----|-------|------|---------|
| 1 | 01-How-LLMs-Work.md + Drill | 2.5h | ☐ |
| 2 | 02-Tokenizers-Deep-Dive.md + Drill | 2.5h | ☐ |
| 3 | 03-Context-Windows-Attention.md | 2.5h | ☐ |
| 4 | 04-Model-Families-Tradeoffs.md | 2h | ☐ |
| 5 | 05-Structured-Outputs-Basics.md + 06-Python-AI-Patterns.md | 3h | ☐ |
| 6 | 07-API-Integration-Patterns.md | 2.5h | ☐ |
| 7 | 08-Streaming-SSE-WebSockets.md | 2.5h | ☐ |
| 8 | 09-AI-Production-Framework-Landscape.md | 2h | ☐ |
| 9 | Drills + Start Project | 3h | ☐ |
| 10 | Finish Project + Phase Gate | 3h | ☐ |

**📅 Deadline:** Day 10 end-of-day. Your LLM Gateway must be working end-to-end.

---

## 📖 Real Talk — Why This Phase Exists

> *"Partha, I've seen senior engineers with 10 years of backend experience fail at AI engineering because they treated LLMs like regular APIs. They didn't understand why the same prompt gives different answers. They didn't know why their app was slow. They got surprised by a $5,000 API bill."*

**This phase prevents all of that.**

You don't need to understand the math of transformers (attention scores, feed-forward networks, backpropagation). But you ABSOLUTELY need to understand:
- **Tokens** — the atomic unit of everything (cost, performance, context limits)
- **Context windows** — your most constrained resource
- **Temperature & sampling** — why the same input produces different outputs
- **Embeddings** — the foundation of RAG, search, and memory
- **Streaming** — required for good UX (nobody wants a spinner)
- **Error handling** — APIs fail, rate limits hit, models return garbage

**Cross-reference:** Everything you learn here connects to:
- **Phase 2 (Prompt Engineering)** — you need to understand tokens and context to engineer prompts
- **Phase 3 (Embeddings)** — you'll use embeddings everywhere
- **Phase 4 (RAG)** — RAG is built on token limits, context windows, and embeddings
- **Phase 6 (Agents)** — agents are just LLMs + tools + memory (all covered here)
- **Phase 10 (Deployment)** — cost optimization starts with understanding token economics

---

## 📋 Phase Progress Tracker

Copy this into your learning journal. Track EVERY module:

```
PHASE 1: FOUNDATIONS
═══════════════════════════════════════════
☐ 01-How-LLMs-Work.md         [  /  ]
☐ 02-Tokenizers-Deep-Dive.md  [  /  ]
☐ 03-Context-Windows-Attention.md [  /  ]
☐ 04-Model-Families-Tradeoffs.md [  /  ]
☐ 05-Structured-Outputs-Basics.md [  /  ]
☐ 06-Python-AI-Patterns.md    [  /  ]
☐ 07-API-Integration-Patterns.md [  /  ]
☐ 08-Streaming-SSE-WebSockets.md [  /  ]
☐ 09-AI-Production-Framework-Landscape.md [  /  ]
☐ Drill: CLI Chat              [  /  ]
☐ Drill: Tokenizer Explorer    [  /  ]
☐ Project: LLM Gateway         [  /  ]
───────────────────────────────────────────
Phase Gate: ___ / 12 complete
```

---

## 🧠 Questions to Think About During This Phase

Don't just read. These questions will make you actually think:

1. **Before starting:** "If an LLM just predicts the next word, how can it answer complex questions correctly?"
2. **After tokenizers:** "Why does 'I love AI' cost the same as 'I ❤️ AI'? (hint: it doesn't — one costs more)"
3. **After context windows:** "If I put 100 pages of context in, does the model actually read all of it? What happens to the middle pages?"
4. **After APIs:** "Why do all major providers have slightly different APIs? What would a unified API look like?"
5. **Before the project:** "I can call 3 different AI APIs. What's the point of a 'gateway'? Just call them directly."

> 🤔 *Pause here. Actually write down your answers before moving on. I'll wait.*

---

## 🚧 Debug Challenge (Pre-Phase Warmup)

Before you start, here's a buggy piece of code. Can you spot what's wrong?

```python
from openai import OpenAI

client = OpenAI()

def get_weather(city: str) -> str:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "user", "content": f"What is the weather in {city}?"}
        ]
    )
    return response

weather = get_weather("Mumbai")
print(f"Temperature: {weather['temperature']}")
```

**🔍 Find at least 5 things wrong with this code.** (Answers at the end of Phase 1)

---

## 🚦 Phase Gate (What You Need to Pass)

Before moving to Phase 2, you must:

- [ ] Complete ALL 9 topic files + 2 drills + 1 project
- [ ] LLM Gateway project works end-to-end with at least 2 providers
- [ ] Gateway handles: streaming, errors, rate limits, and cost tracking
- [ ] Gateway supports swapping providers via config (no code changes)
- [ ] You can explain how tokens work to a non-technical person
- [ ] You can estimate the cost of any AI feature before building it
- [ ] Gateway is pushed to GitHub with a README

---

**Start with → `01-How-LLMs-Work.md`**
