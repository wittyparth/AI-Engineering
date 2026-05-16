# Phase 2: Build a Prompt Engineering System

**The Problem**: You're hand-tuning prompts in a notebook. They work once, fail on edge cases, break when you switch models, and you have no idea why. Prompt quality degrades silently in production.

**What You'll Build**: A systematic prompt engineering platform with versioning, testing, injection defense, and automated optimization — like what companies like Klarna and Credit Genie use to manage prompts at scale.

**Difficulty**: Easy-Medium | **Time**: ~8 hours

## The Build Path

```
1. PROBLEM: "Prompt engineering is vibes-based, not engineering"
2. BUILD RAW: Design patterns, CoT, few-shot — craft prompts by hand
3. ADD FRAMEWORK: DSPy — compile prompts against metrics automatically
4. PRODUCTIONIZE: Prompt templates, versioning, injection detection 
5. EVALUATE: A/B test prompt variants, regression test suite
6. DEBUG: Prompt injection lab — break 10 attack patterns, then fix them
```

## Concepts (Built Into the Flow)

| # | Step | What You Learn | Why It Matters |
|---|------|---------------|----------------|
| 1 | Problem | Prompt design patterns, roles, personas | Foundation of ALL AI interaction |
| 2 | Build Raw | Chain-of-thought, few-shot engineering | The biggest quality levers you control |
| 3 | Build Raw | Systematic prompt testing, versioning | Treat prompts like code — test them |
| 4 | Add Framework | DSPy — automated prompt optimization | Level 4 in the prompt engineering hierarchy |
| 5 | Productionize | Prompt templates, environments, rollback | Enterprise prompt management |
| 6 | Productionize | Input sanitization, injection detection | Security is non-negotiable |
| 7 | Evaluate | A/B testing, regression detection | Know if your prompt change helps or hurts |
| 8 | Debug | Injection lab — 10 attack patterns | Understand how attackers think |

## The Project: Injection-Resistant Prompt System

A FastAPI service with:
- Prompt template registry (versioned, environment-aware, rollback)
- Input sanitization and injection detection (blocks 10+ attack patterns)
- Automated prompt testing suite (CI gate)
- DSPy compilation pipeline (auto-optimize prompts)
- A/B testing of prompt variants with statistical significance

**This is what production prompt management looks like at companies processing millions of queries.**

## 🔴 Senior: The Prompt Engineering Hierarchy

```
Level 1: Copy-paste prompts in a notebook
Level 2: Prompt templates with variables
Level 3: Prompt versioning + testing suite
Level 4: Automated prompt optimization (DSPy)
Level 5: Adaptive prompts that change based on context
```

Most teams are at Level 1-2. This phase gets you to Level 4.

## Gate Check

- [ ] Prompt passes 100% of test cases in the suite
- [ ] Template system supports versioning and rollback
- [ ] Injection detection catches 10 different attack patterns
- [ ] DSPy compilation improves accuracy by 10%+ over hand-tuned
- [ ] A/B test shows statistically significant improvement
