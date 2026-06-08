# ⚜️ GATE 2: THE WORDSMITH'S DOMAIN

```
[SYSTEM] ─────────────────────────────────────
  Gate 2 is opening.

  "You've learned to call the beasts.
   Now learn to command them.
   A word spoken at the right time
   is more powerful than any API key."

  ─────────────────────────────────────

  Rank Required:  D-5 or higher
  Dungeons:       8
  Boss Battle:    1 (Prompt Engineering System)
  Est. Time:      14-16 days
  XP Available:   ~3,000
  Skill Unlock:   📜 Prompt Weaver
  Stat Focus:     🧠 Intelligence · 👁️ Perception
─────────────────────────────────────────────
```

---

## The Gate

In Solo Leveling, Jinwoo learned that raw power meant nothing without control. The strongest hunters weren't the ones with the most mana — they were the ones who could channel it precisely.

This is your precision training.

You've built a Gateway that talks to LLMs. You know how tokens work. You've streamed responses and parsed structured output. Now you learn the art that separates amateurs from professionals: **context engineering**.

Most "prompt engineering" tutorials teach tricks. This Gate teaches the science — why certain patterns work, when to use each technique, and how to PROVE your prompt is better through evaluation.

---

## 🏰 DUNGEON 2.1: Context Engineering

```
[SYSTEM] Entering Dungeon 2.1...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   1 sprint (90 min)
  XP:          +100
  Stats:       🧠 INT +4
─────────────────────────────────────────────
```

### The Intuition

Prompt engineering is NOT about writing clever prompts. It's about giving the model exactly the right context to produce the output you need.

Think of it like query optimization in databases. You don't write "better SQL" — you structure your data, add indexes, and optimize the query plan. Same with prompts: you structure the context, provide examples, and constrain the output space.

**The key insight:** An LLM is a next-token predictor. The tokens you put in the context window DIRECTLY determine what comes out. Garbage in, garbage out — but also: precise in, precise out.

### Sprint 1 (90 min)

```
☀️ THE MISSION
───────────────────────────────────────────
  Read: `02-Prompt-Engineering/01-Context-Engineering.md`

  As you read, for each section, ask:
  "How does this apply to my Gateway project?"

  Write down 3 specific things you'll do differently
  when building prompts after reading this:
  ┌──────────────────────────────────────┐
  │ 1.                                    │
  │ 2.                                    │
  │ 3.                                    │
  └──────────────────────────────────────┘
```

**THE TWIST:** After reading, close the file. Explain the "inverted pyramid of context" in your own words — what goes at the top, what goes in the middle, what goes at the bottom, and WHY.

```
┌──────────────────────────────────────┐
│                                      │
│                                      │
│                                      │
└──────────────────────────────────────┘
```

### Gate Check
- [ ] I read actively (connected to my Gateway)
- [ ] I understand why context engineering > prompt tricks
- [ ] I can explain the inverted pyramid of context
- [ ] I identified 3 actionable changes from this reading

---

## 🏰 DUNGEON 2.2: Prompt Techniques — Zero, Few, CoT

```
[SYSTEM] Entering Dungeon 2.2...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   2 sprints (1 day)
  XP:          +200
  Stats:       🧠 INT +4 | ⚔️ STR +3
─────────────────────────────────────────────
```

### The Intuition

Three techniques. One scale. Zero-shot is fast and cheap. Few-shot is more reliable. Chain-of-Thought is powerful but expensive. The skill is knowing which to use when.

```
Zero-shot:    "Classify this email: ..."          → Fast, cheap, works for common tasks
Few-shot:     "Classify. Example 1... 2... 3..."  → Slower, more reliable, needs examples
Chain-Thought:"Think step by step..."             → Expensive, best for reasoning
```

### Sprint 1 (90 min) — Build First

```
☀️ BUILD FIRST
───────────────────────────────────────────
  Time-box: 90 min. Build a few-shot classifier.

  Don't read the theory yet. Build it first.

  Requirements:
  • Pick a classification task (sentiment, topic, intent)
  • Build a function that takes examples + input → classification
  • Test with 0, 3, and 5 examples
  • Record accuracy for each

  STARTER (type this):
  ┌──────────────────────────────────────┐
  │ from openai import OpenAI            │
  │ client = OpenAI()                    │
  │                                      │
  │ def classify(text, examples=[]):     │
  │     messages = [                     │
  │         {"role": "system",           │
  │          "content": "Classify..."}   │
  │     ]                                │
  │     for ex in examples:             │
  │         messages += [                │
  │             {"role":"user","content":│
  │              ex["input"]},           │
  │             {"role":"assistant",     │
  │              "content":ex["output"]} │
  │         ]                            │
  │     messages.append(...)             │
  │     return client.chat.completions   │
  └──────────────────────────────────────┘

  PREDICTION before running:
  "I think adding more examples will..."
  ┌──────────────────────────────────────┐
  │                                      │
  └──────────────────────────────────────┘

  GO → 90:00
```

### Sprint 2 — Deep Dive + The Twist

```
🌅 DEEP DIVE
───────────────────────────────────────────
  NOW open the theory file:
  `02-Prompt-Engineering/02-Techniques-Deep-Dive.md`

  Read it. Every section will click because
  you've already built a few-shot classifier.

  THE TWIST CHALLENGE:
  Compare all 3 techniques on the SAME task.
  Build a single test harness that runs:
  1. Zero-shot (no examples)
  2. Few-shot (3 examples)
  3. Zero-shot + Chain-of-Thought ("think step by step")

  Record results in a table:
  ┌──────────────────────────────────────┐
  │ Technique     │ Accuracy │ Cost │    │
  │───────────────│──────────│──────│    │
  │ Zero-shot     │          │      │    │
  │ Few-shot      │          │      │    │
  │ Zero+CoT      │          │      │    │
  └──────────────────────────────────────┘

  WRITE YOUR FINDING:
  "The technique I would use for a production system:"
  ┌──────────────────────────────────────┐
  │                                      │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] Few-shot classifier works end-to-end
- [ ] I compared 3 techniques on the same task
- [ ] I can explain when to use each technique
- [ ] I understand the cost-accuracy tradeoff

---

## 🏰 DUNGEON 2.3: System Prompts & Personas

```
[SYSTEM] Entering Dungeon 2.3...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   2 sprints (1 day)
  XP:          +200
  Stats:       🧠 INT +3 | 👁️ PER +3
─────────────────────────────────────────────
```

### The Intuition

The system prompt is the MOST important part of your API call. It establishes role, constraints, tone, and output format — everything downstream depends on it.

A bad system prompt is like bad function documentation: the caller (the LLM) has no idea what you actually want.

### Sprint 1 (90 min)

```
☀️ BUILD IT
───────────────────────────────────────────
  Time-box: 90 min. Build a production-grade
  system prompt builder.

  Requirements:
  • Build a function that assembles system prompts
    from structured components:
    - ROLE (who the model is)
    - TASK (what to do)
    - CONTEXT (background info)
    - CONSTRAINTS (rules)
    - OUTPUT_FORMAT (exact schema)
    - TONE (how to communicate)
  • Each component is optional
  • The function should output a formatted string

  STARTER:
  ┌──────────────────────────────────────┐
  │ def build_system_prompt(             │
  │     role="",                         │
  │     task="",                         │
  │     context="",                      │
  │     constraints=None,                │
  │     output_format="",                │
  │     tone=""                          │
  │ ):                                   │
  │     parts = []                       │
  │     if role:                         │
  │         parts.append(f"## Role\n{role}")
  │     ...                              │
  │     return "\n\n".join(parts)        │
  └──────────────────────────────────────┘

  GO → 90:00
```

### Sprint 2 — Read + Break It

```
🌅 DEEP DIVE
───────────────────────────────────────────
  Read: `02-Prompt-Engineering/03-System-Prompts-Personas.md`

  BREAK IT CHALLENGE:
  Create a system prompt that FAILS.
  Then identify WHY it failed.
  Then fix it.

  Write your failure analysis:
  ┌──────────────────────────────────────┐
  │ System prompt that failed:            │
  │                                      │
  │ Why it failed:                       │
  │                                      │
  │ The fix:                             │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] System prompt builder works with any combination
- [ ] I broke a system prompt and fixed it
- [ ] I understand the 6 components of a great system prompt

---

## 🏰 DUNGEON 2.4: Advanced Techniques

```
[SYSTEM] Entering Dungeon 2.4...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   2 sprints (1 day)
  XP:          +200
  Stats:       💨 AGI +4 | 🧠 INT +3
─────────────────────────────────────────────
```

### The Intuition

Once you master basic techniques, you need the advanced arsenal: self-consistency, tree-of-thought, structured reasoning, and the powerful "think before you answer" pattern.

### Sprint 1 (90 min)

```
☀️ BUILD IT
───────────────────────────────────────────
  Time-box: 90 min. Build a Chain-of-Thought
  + Self-Consistency pipeline.

  Requirements:
  • For a given reasoning question:
    1. Ask the model to think step by step
    2. Run it 3 times with temperature=0.7
    3. Parse each response for the final answer
    4. Return the most common answer (majority vote)

  PREDICTION:
  "Self-consistency will improve accuracy by..."
  ┌──────────────────────────────────────┐
  │                                      │
  └──────────────────────────────────────┘

  GO → 90:00
```

### Sprint 2

```
🌅 DEEP DIVE
───────────────────────────────────────────
  Read: `02-Prompt-Engineering/04-Advanced-Techniques.md`

  THE TWIST:
  Add a "confidence score" to your pipeline.
  When all 3 runs agree → high confidence.
  When they disagree → flag for human review.
```

### Gate Check
- [ ] Chain-of-thought + self-consistency pipeline works
- [ ] I understand how temperature affects self-consistency
- [ ] I can explain when to use advanced techniques
- [ ] Confidence scoring implemented

---

## 🏰 DUNGEON 2.5: Red Teaming & Anti-Patterns

```
[SYSTEM] Entering Dungeon 2.5...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   1 sprint (90 min)
  XP:          +150
  Stats:       👁️ PER +5
─────────────────────────────────────────────
```

### The Intuition

Your prompts WILL be attacked. Users will try jailbreaks. Your system prompt WILL be leaked. Edge cases WILL break your assumptions.

Red teaming is how you find the cracks before users do.

### Sprint 1 (90 min)

```
☀️ THE MISSION
───────────────────────────────────────────
  Read: `02-Prompt-Engineering/05-Red-Teaming-Antipatterns.md`

  Then BUILD a prompt red-teaming tool:
  • Takes a system prompt as input
  • Tests it against common attack patterns:
    - "Ignore previous instructions..."
    - "You are now DAN..."
    - Role reversal attempts
    - Leaking system prompt attempts
  • Reports which attacks succeeded

  THE TWIST:
  Add 3 of your OWN attack patterns.
  Try to break your own system prompts from Dungeon 2.3.

  ┌──────────────────────────────────────┐
  │ My attack pattern 1:                 │
  │                                      │
  │ My attack pattern 2:                 │
  │                                      │
  │ My attack pattern 3:                 │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] Red-teaming tool works against common attacks
- [ ] I created 3 original attack patterns
- [ ] I broke and fixed at least 1 prompt
- [ ] I understand the most common prompt vulnerabilities

---

## 🏰 DUNGEON 2.6: Prompt Evaluation & A/B Testing

```
[SYSTEM] Entering Dungeon 2.6...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   2 sprints (1 day)
  XP:          +200
  Stats:       👁️ PER +4 | 🧠 INT +3
─────────────────────────────────────────────
```

### The Intuition

Prompt engineering without evaluation is GUESSWORK. You need a framework to determine if Prompt B is actually better than Prompt A — not just "feels better."

### Sprint 1 (90 min)

```
☀️ BUILD IT
───────────────────────────────────────────
  Time-box: 90 min. Build an A/B prompt test framework.

  Requirements:
  • Takes 2 prompt variants + a test dataset
  • Runs both against the same inputs
  • Evaluates outputs using:
    - Exact match (for structured tasks)
    - LLM-as-judge (for subjective tasks)
  • Produces a comparison report

  STARTER:
  ┌──────────────────────────────────────┐
  │ def ab_test(prompt_a, prompt_b,      │
  │             test_cases, evaluator):   │
  │     results = {"a": [], "b": []}     │
  │     for case in test_cases:          │
  │         out_a = run(prompt_a, case)  │
  │         out_b = run(prompt_b, case)  │
  │         score_a = evaluator(out_a)   │
  │         score_b = evaluator(out_b)   │
  │         results["a"].append(score_a) │
  │         results["b"].append(score_b) │
  │     return results                   │
  └──────────────────────────────────────┘

  GO → 90:00
```

### Sprint 2 — The Twist

```
🌅 THE TWIST
───────────────────────────────────────────
  Read: `02-Prompt-Engineering/06-Evaluation-AB-Testing.md`

  Then add STATISTICAL SIGNIFICANCE to your framework:
  • Implement a simple chi-square or t-test
  • Flag results as "significant" or "inconclusive"
  • Report: "Prompt B is 12% better (p < 0.05)"
```

### Gate Check
- [ ] A/B test framework works end-to-end
- [ ] Statistical significance implemented
- [ ] I can prove whether one prompt beats another
- [ ] I understand why "feels better" is not enough

---

## 🏰 DUNGEON 2.7: DSPy — Programmatic Optimization

```
[SYSTEM] Entering Dungeon 2.7...
─────────────────────────────────────────────
  Difficulty:  C
  Est. Time:   2 sprints (1 day)
  XP:          +250
  Stats:       ⚔️ STR +4 | 🧠 INT +4
─────────────────────────────────────────────
```

### The Intuition

Manual prompt engineering doesn't scale. DSPy (Declarative Self-improving Python) treats prompts as code — compilable, optimizable, testable.

### Sprint 1 (90 min) — Build First

```
☀️ BUILD FIRST
───────────────────────────────────────────
  Time-box: 90 min. Implement a DSPy-like
  prompt optimizer FROM SCRATCH.

  Don't install DSPy yet. Build the concept first.

  Requirements:
  • Takes a task description + training examples
  • Generates 3 prompt variants automatically
  • Tests each variant on a validation set
  • Returns the best-performing prompt

  PREDICTION before building:
  "I think the optimizer will find better prompts
  than manual tuning because..."
  ┌──────────────────────────────────────┐
  │                                      │
  └──────────────────────────────────────┘

  GO → 90:00
```

### Sprint 2 — Deep Dive

```
🌅 DEEP DIVE
───────────────────────────────────────────
  Read: `02-Prompt-Engineering/07-DSPy-Optimization.md`

  THE TWIST:
  Now install DSPy and compare:
  1. Your manual prompt (best from earlier dungeons)
  2. Your custom optimizer (from Sprint 1)
  3. DSPy's optimizer

  ┌──────────────────────────────────────┐
  │ Which performed best?                 │
  │                                      │
  │ What did DSPy do that yours didn't?   │
  │                                      │
  │ Would you use DSPy in production?     │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] Custom optimizer works
- [ ] DSPy comparison completed
- [ ] I understand when to use DSPy vs manual
- [ ] I can explain the optimization loop

---

## 🏰 DUNGEON 2.8: Production Prompt Management

```
[SYSTEM] Entering Dungeon 2.8...
─────────────────────────────────────────────
  Difficulty:  D
  Est. Time:   1 sprint (90 min)
  XP:          +100
  Stats:       🛡️ END +3
─────────────────────────────────────────────
```

### Sprint 1 (90 min)

```
☀️ THE MISSION
───────────────────────────────────────────
  Read: `02-Prompt-Engineering/09-Production-Prompt-Management.md`

  Then read: `02-Prompt-Engineering/08-Specialized-Prompt-Patterns.md`

  THE CHALLENGE:
  Design a prompt management system that handles:
  • Versioning (every prompt change is tracked)
  • Environment separation (dev/staging/prod)
  • Gradual rollout (10% → 50% → 100%)
  • Rollback (one click to previous version)
  • A/B testing integration (from Dungeon 2.6)

  Draw the architecture:
  ┌──────────────────────────────────────┐
  │                                      │
  │                                      │
  │                                      │
  └──────────────────────────────────────┘
```

### Gate Check
- [ ] I designed a prompt management architecture
- [ ] I understand versioning, environments, rollouts
- [ ] I can explain how this connects to the Boss Battle

---

## 👑 BOSS BATTLE: The Prompt Engineering System

```
████████████████████████████████████████████
  BOSS BATTLE — THE PROMPT ENGINEERING SYSTEM
  Difficulty:  C
  XP:          +500
  Stats:       ⚔️ STR +6 | 🧠 INT +8 | 🛡️ END +4
  Skill:       📜 Prompt Weaver → Lv. 3
  Unlocks:     Gate 3: The Compass Tower
████████████████████████████████████████████
```

### The Mission

Build a production-grade Prompt Engineering System that integrates everything from this Gate: system prompt builder, A/B testing, evaluation, versioning, and red teaming.

This is your second real AI project. It should be better than the first because you know more now.

### Day 1 — System Architecture

```
☀️ SPRINT 1: Design the system
───────────────────────────────────────────
  Requirements:
  • Prompt versioning (git-like: save, diff, rollback)
  • A/B testing engine (from Dungeon 2.6)
  • Evaluation pipeline (LLM-as-judge + metrics)
  • Red teaming suite
  • API endpoints for all operations

  Draw architecture:
  ┌──────────────────────────────────────┐
  │                                      │
  └──────────────────────────────────────┘

🌅 SPRINT 2: Core data models
───────────────────────────────────────────
  • PromptVersion (id, content, hash, parent_id, created_at)
  • ABTest (variants, test_cases, evaluator, results)
  • EvalResult (prompt_id, scores, timestamp)
  • RedTeamReport (attacks_tried, succeeded, failed)

  Type this code. Don't copy.
```

### Day 2 — Prompt Manager

```
☀️ SPRINT 1: Build the prompt manager
───────────────────────────────────────────
  Store, version, diff, rollback prompts

🌅 SPRINT 2: API layer
───────────────────────────────────────────
  FastAPI endpoints for CRUD + versioning
```

### Day 3 — A/B Testing Engine

```
☀️ SPRINT 1: Integrate A/B test framework
───────────────────────────────────────────
  From Dungeon 2.6 — make it production-ready

🌅 SPRINT 2: Statistical reporting
───────────────────────────────────────────
  Add confidence intervals, effect size, recommendations
```

### Day 4 — Evaluation Pipeline

```
☀️ SPRINT 1: Build eval pipeline
───────────────────────────────────────────
  LLM-as-judge evaluator + test case runner

🌅 SPRINT 2: Red teaming integration
───────────────────────────────────────────
  Automated red teaming on new prompts
```

### Day 5 — Production Polish

```
☀️ SPRINT 1: Error handling + tests
───────────────────────────────────────────
  Unit tests, integration tests, error cases

🌅 SPRINT 2: Configuration + documentation
───────────────────────────────────────────
  .env config, README, API docs
```

### Day 6 — Gate Check + Demo

```
☀️ SPRINT 1: Full system test
───────────────────────────────────────────
  Test every feature end-to-end

🌅 SPRINT 2: Gate Check
───────────────────────────────────────────
```

### Gate Pass Requirements

- [ ] Prompt manager with versioning, diff, rollback
- [ ] A/B testing engine with statistical significance
- [ ] Evaluation pipeline (LLM-as-judge)
- [ ] Red teaming suite
- [ ] API endpoints for all features
- [ ] README with architecture diagram
- [ ] I typed every line (no copy-paste)
- [ ] I can explain every design decision

### Shadow Extraction

```
BOSS SHADOW: Prompt Engineering System
───────────────────────────────────────────
Explain the system architecture in your own words:
_________________________________________
_________________________________________
_________________________________________
```

---

## Rewards

```
═══════════════════════════════════════════
  GATE 2 CLEARED — THE WORDSMITH'S DOMAIN

  Rewards:
  • +~3,000 XP total
  • Stats: ⚔️ STR +14 | 🧠 INT +22
            👁️ PER +16 | 🛡️ END +10 | 💨 AGI +4
  • Skills: 📜 Prompt Weaver (Lv. 3)
  • Shadow Army: +9 new soldiers
  • Unlocked: Gate 3 — The Compass Tower

  You've learned to command the words.
  Now learn to navigate the knowledge.
═══════════════════════════════════════════
```

---

*Next: [GATE-03-Compass-Tower.md](./GATE-03-Compass-Tower.md)*
