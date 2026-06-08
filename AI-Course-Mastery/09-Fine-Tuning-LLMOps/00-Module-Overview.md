# 00 — Module Overview: Fine-Tuning & LLMOps

## 🎯 Purpose & Goals

**Phase 9 is where you stop being a consumer of AI models and start becoming a CREATOR of them.**

Everything you've done so far — prompt engineering, RAG, agents, guardrails — treats the model as a FIXED thing. You work around its limitations. You prompt it better. You give it better context. You guard against its failures.

Fine-tuning changes the equation. Instead of working around the model's limitations, you CHANGE the model itself.

But here's the trap: **fine-tuning is the most OVERHYPED and MISUNDERSTOOD technique in AI engineering.**

Every tutorial makes it sound like magic:
- "Fine-tune on your data and watch accuracy soar!"
- "Custom models outperform GPT-4 on your specific task!"
- "LoRA makes fine-tuning accessible to everyone!"

The reality is MUCH more nuanced:
- Most of the time, prompt engineering + RAG will outperform a fine-tuned model
- Fine-tuning REGULARLY makes models worse at things they were good at (catastrophic forgetting)
- Fine-tuning requires HIGH-QUALITY data — and "high quality" is harder than it sounds
- Fine-tuning can introduce SAFETY REGRESSIONS that your Phase 8 guardrails need to catch
- Fine-tuning is EXPENSIVE and TIME-CONSUMING — and often doesn't pay off

**The most important skill this phase will teach you: knowing WHEN NOT to fine-tune.**

---

### 🤔 Discovery Question 1: The Wrong Decision

You're building a customer support AI for a SaaS company. You have 10,000 support tickets with answers.

**Option A**: Prompt engineer GPT-4 with 5 carefully crafted few-shot examples.
- Cost: $50/month in API calls, 2 hours of work
- Accuracy: 87% on test set

**Option B**: Fine-tune GPT-3.5 on your 10,000 tickets.
- Cost: $500 for training + $200/month in API calls, 1 week of work
- Accuracy: 89% on test set

**Option C**: Build a RAG system (Phase 4) with your knowledge base.
- Cost: $100/month in API + vector DB, 3 days of work
- Accuracy: 93% on test set

**🤔 Before reading on, answer these:**

1. Option B (fine-tuning) costs 10x more than Option A (prompt engineering) for a 2% improvement. Option C (RAG) beats BOTH with less effort. Given this data, is fine-tuning EVER the right choice? When would you choose B over A or C?

2. The accuracy numbers above assume ideal conditions. In practice, fine-tuning often has HIDDEN costs:
   - Data cleaning takes 2-3x longer than expected
   - First training run fails (wrong format, wrong parameters)
   - The fine-tuned model regresses on unrelated tasks
   - You need 3-5 training runs to get it right
   
   If you add these real-world costs, Option B might take 3 weeks and cost $3,000. Does that change your decision?

3. **The hardest question:** Six months later, your product changes. The support tickets now cover new features. Your RAG system (Option C) just needs a document update — done in an hour. Your fine-tuned model (Option B) needs RETRAINING — another $500 and a week of work. **How does MAINTENANCE cost change the fine-tuning decision?** Should fine-tuning only be used for tasks that DON'T change over time? What tasks are those?

---

### 🤔 Discovery Question 2: The Capability Regression Trap

You fine-tune a model on legal document summarization. Before fine-tuning, it scores:
- General knowledge: 92% (MMLU benchmark)
- Code generation: 88% (HumanEval)
- Legal summarization: 65% (your test set)

After fine-tuning on 5,000 legal summaries:
- General knowledge: 87% (-5% 😱)
- Code generation: 81% (-7% 😱)
- Legal summarization: 91% (+26% 🎉)

**🤔 Before reading on, answer these:**

1. The fine-tuned model is AMAZING at legal summarization — 91%! But it got WORSE at general knowledge and coding. Your users notice: "The AI used to help me with Python, now it's terrible." Was the fine-tuning WORTH it? How do you measure "overall value" when the model improves in one dimension but degrades in others?

2. What CAUSED the regression? The model weights changed. The legal data "overwrote" some of the general knowledge. But why did coding ability drop 7% more than general knowledge? Is there a structural reason why some capabilities are more fragile than others during fine-tuning?

3. **The hardest question:** You need BOTH good legal summarization AND good general knowledge. Options:
   - Fine-tune a model and use it ONLY for legal tasks (cost: 2 models to maintain)
   - Fine-tune with EWC (Elastic Weight Consolidation) to protect general knowledge (cost: more complex training)
   - Use a RAG system for legal + base model for general (cost: latency, two calls)
   - Accept the regression and tell users "this model is legal-specific" (cost: user trust)
   
   **What do you choose? Why? How do you even MEASURE which option is best?**

---

### 🤔 Discovery Question 3: The Data Quality Illusion

You collect 10,000 examples for fine-tuning. They look great:
- All correctly formatted
- All verified by domain experts
- All cover the full range of tasks you need

You train. The model trains successfully (loss goes down). You evaluate... and the model performs WORSE than prompting.

**🤔 Before reading on, answer these:**

1. Your training loss went DOWN. The model learned SOMETHING. But it performs WORSE. What could it have learned that makes it worse? (Hint: memorization without understanding, spurious correlations, noise fitting)

2. You had domain experts verify every example. But domain experts have DISAGREEMENTS. If 3 experts reviewed the same example and chose different answers, which one is "correct"? If your dataset has inconsistent labeling, the model learns the AVERAGE of all answers — which is wrong for everyone. How do you detect and fix labeling inconsistency?

3. **The hardest question:** 5,000 of your 10,000 examples are about password reset (your most common support issue). 500 are about API integration (rare but complex). The model becomes GREAT at password reset but terrible at API integration. Your dataset distribution matches your production distribution — but the model optimizes for FREQUENCY, not IMPORTANCE. **How do you construct a training dataset that balances frequency (what happens most) with importance (what matters most when it fails)?**

---

### Phase 9 Roadmap

```
PHASE 9: FINE-TUNING & LLMOPS

 00 ─ Module Overview ──── When to fine-tune, when NOT to
  │                         (Discovery-driven decision framework)
  │
 01 ─ Fine-Tuning ───────── What actually happens during training?
      Fundamentals           Loss curves, overfitting, forgetting
  │
 02 ─ LoRA & PEFT ────────── How LoRA works, rank selection,
      │                        adapter math, merging, routing
  │
 03 ─ Dataset ────────────── Collection, cleaning, formatting,
      Preparation             quality filtering, deduplication
  │
 04 ─ Training with ──────── Training loop, hyperparameters,
      Unsloth/TRL             monitoring, checkpointing
  │
 05 ─ Evaluation & ───────── Comparing FT vs base, benchmarks,
      Comparison              regression testing (links Phase 7)
  │
 06 ─ Function-Calling ───── Specialized FT for tool use,
      FT                      structured outputs, agents
  │
 07 ─ LLMOps ─────────────── Model registry, versioning,
      │                        experiment tracking, routing
  │
 08 ─ Deployment ─────────── vLLM serving, quantization,
      │                        A/B testing, scaling
  │
 09 ─ PROJECT ────────────── Fine-tune a custom model for
      Custom Model            a real use case (Portfolio)
```

### What This Phase Connects To

| Phase | Connection |
|-------|-----------|
| Phase 1 (Foundations) | You built a gateway. Now you'll build the model it serves. |
| Phase 2 (Prompt Eng) | Fine-tuning REPLACES prompt engineering for some tasks — but prompt engineering still matters for guiding fine-tuned models |
| Phase 4 (RAG) | Fine-tuning vs RAG is a CRITICAL design decision. This phase shows you when each wins. |
| Phase 7 (Evals) | Your Phase 7 eval platform evaluates fine-tuned models. Fine-tuning WITHOUT evaluation is blind. |
| Phase 8 (Guardrails) | Fine-tuning can introduce safety regressions. Your guardrails catch them. |

### Time Budget

| Module | Estimated Time |
|--------|---------------|
| 00 — Module Overview | 30 min |
| 01 — Fine-Tuning Fundamentals | 4 hours |
| 02 — LoRA & PEFT | 5 hours |
| 03 — Dataset Preparation | 6 hours |
| 04 — Training with Unsloth/TRL | 5 hours |
| 05 — Evaluation & Comparison | 4 hours |
| 06 — Function-Calling FT | 4 hours |
| 07 — LLMOps | 3 hours |
| 08 — Deployment | 4 hours |
| 09 — Project: Custom Model | 10 hours |
| **Total** | **~45 hours** |
