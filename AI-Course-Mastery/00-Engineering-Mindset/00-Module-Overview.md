# Phase 0: AI Engineering Mindset

## 🎯 Purpose & Goals

Before you write a single line of AI code, you must install the correct mental models. Most AI engineers fail not because they can't code, but because they think about AI systems the wrong way — treating them like deterministic software when they're probabilistic, or treating them like magic when they're just statistics.

**By the end of this phase, you will:**
- Think like a senior AI engineer, not a junior playing with APIs
- Understand why production AI fails differently than traditional software
- Have an evaluation-first mindset — you measure before you build
- Know how to make architectural decisions with tradeoffs explicitly weighed
- Be cost-aware in every design choice
- Master the Build Loop that every phase of this course follows

**⏱ Total Time Budget:** 8–10 hours over 5 days

| Day | Topic | Time |
|-----|-------|------|
| 1 | Senior vs Junior Mindset + Production vs Academic Thinking | 2 hrs |
| 2 | Evaluation-First Philosophy | 2 hrs |
| 3 | Decision Framework + Drill | 2 hrs |
| 4 | Cost Awareness Culture + Drill | 2 hrs |
| 5 | The Build Loop + Phase Gate | 2 hrs |

**📅 Deadline:** Day 5 end-of-day. You must pass the Gate Check to proceed.

---

## 📖 Why This Phase Exists

### The Hardest Problem in AI Engineering Is Not Technical

The hardest problem is **uncertainty management**.

Traditional software is deterministic: input X → function Y → output Z. You can unit test it, you can predict behavior, you can sleep at night.

AI software is probabilistic: input X → model Y → output Z... maybe. Or garbage. Or a hallucination. Or a security breach. Or it costs 10x more than expected.

Senior AI engineers don't try to eliminate uncertainty — they **design for it**.

They build:
- **Guardrails** that catch bad outputs before reaching users
- **Evals** that measure quality degradation over time
- **Observability** that traces every model call
- **Fallbacks** for when the model fails
- **Cost controls** that prevent bill shock

This is the difference between "I can call an API" (junior) and "I can ship a reliable AI product" (senior).

### The Three Pillars of AI Engineering Maturity

| Pillar | Junior | Senior |
|--------|--------|--------|
| **Technical** | Knows API parameters | Knows when NOT to use AI, knows failure modes |
| **Economic** | Has no idea what a query costs | Can estimate cost per user per day before writing code |
| **Evaluation** | "Looks good to me" | Has a test suite, regression benchmarks, and can prove improvement |

---

## 🗺️ Module Map

| File | What You'll Learn |
|------|-------------------|
| `01-Senior-vs-Junior-Mindset` | The cognitive framework that separates architects from API-callers |
| `02-Production-vs-Academic-Thinking` | Why academic AI fails in production — the 80/20 trap |
| `03-Evaluation-First-Philosophy` | You cannot improve what you cannot measure |
| `04-Decision-Framework` | A repeatable process for every architecture choice |
| `05-Cost-Awareness-Culture` | Token economics, latency budgets, cost per user |
| `06-The-Build-Loop` | The engineering cycle used in every phase of this course |

---

## 📚 Cross-Phase References

This mindset phase connects to:
- **Phase 4 (RAG)** → You'll need the evaluation-first mindset to judge RAG quality
- **Phase 6 (Agents)** → The decision framework helps you choose agent architectures
- **Phase 7 (Evals)** → Direct application of the evaluation-first philosophy
- **Phase 10 (Deployment)** → Cost awareness pays off here

---

## 🚦 Phase Gate

Before moving to Phase 1 (Foundations), you must:

- [ ] Complete all 6 topic files in this phase
- [ ] Complete the Decision Framework drill
- [ ] Complete the Cost Analysis drill  
- [ ] Write 1-page learning journal entry: "What I thought AI engineering was vs what it actually is"
- [ ] Be able to answer: "What's the FIRST thing you should build before any AI feature?"

**Answer to the gate question:** Your evaluation pipeline. Always. Without evals, you're guessing.

---

**Proceed to → `01-Senior-vs-Junior-Mindset.md`**
