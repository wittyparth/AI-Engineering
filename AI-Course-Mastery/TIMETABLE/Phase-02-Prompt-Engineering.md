# Phase 2 — Prompt Engineering Systematic (Days 22-32)

**Goal:** Move from ad-hoc prompting to systematic prompt engineering with evaluation, optimization, and production management.
**Files:** 9 concept + 1 project (10 teaching files + 1 overview)
**Total days:** 11 study days + 1 phase-end review

---

### Day 22 — Context Engineering

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Module overview + Context Engineering | `00-Module-Overview.md`, `01-Context-Engineering.md` |
| 🌤️ Afternoon | Code: build a context window calculator + prompt structure optimizer | Type key examples from the file |
| 🌙 Evening | Flashback | Write: "What is context engineering and why is it the foundation of prompting?" |

**Key Concepts:** Context window management, prompt structure, instruction hierarchy, delimiter usage
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Context window budget, instruction placement, delimiter strategies

---

### Day 23 — Techniques Deep Dive (Full Day — Large File)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Zero-shot, Few-shot, Chain-of-Thought | `02-Techniques-Deep-Dive.md` (first half) |
| 🌤️ Afternoon | Advanced techniques + code all examples | `02-Techniques-Deep-Dive.md` (second half) — type every code example |
| 🌙 Evening | Flashback | Write: "Compare zero-shot, few-shot, and CoT — when would you use each?" |

**Key Concepts:** Zero-shot vs few-shot vs CoT, when each works, failure modes of each
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Technique comparison table, CoT variants, technique selection decision tree

---

### Day 24 — System Prompts & Advanced Techniques

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | System Prompts & Personas | `03-System-Prompts-Personas.md` |
| 🌤️ Afternoon | Advanced Techniques | `04-Advanced-Techniques.md` |
| 🌙 Evening | Flashback | Draft a system prompt for an AI coding tutor (from memory, not looking) |

**Key Concepts:** System prompt structure, persona crafting, advanced prompting techniques (tree-of-thought, self-consistency, etc.)
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** System prompt building blocks, advanced technique selection guide

---

### Day 25 — Red Teaming & Antipatterns

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Red Teaming & Antipatterns | `05-Red-Teaming-Antipatterns.md` |
| 🌤️ Afternoon | Practical: break your own prompts | Take yesterday's system prompt, attack it with red-teaming techniques |
| 🌙 Evening | Flashback | Write: "What 3 prompt vulnerabilities did I discover in my own prompting?" |

**Key Concepts:** Common prompt vulnerabilities, jailbreak patterns, defensive prompting
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Injection categories, defense layers, red-teaming methodology

---

### Day 26 — Evaluation & A/B Testing

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Evaluation & A/B Testing | `06-Evaluation-AB-Testing.md` |
| 🌤️ Afternoon | Code: build a prompt evaluation harness | Build an A/B testing framework for comparing prompt variants |
| 🌙 Evening | Flashback | Write: "How do you measure whether one prompt is better than another?" |

**Key Concepts:** Prompt evaluation metrics, A/B test design, statistical significance
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Evaluation metrics (accuracy, coherence, instruction following), test design methodology

---

### Day 27 — DSPy Optimization (Full Day — Large File)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | DSPy Optimization | `07-DSPy-Optimization.md` (concepts + first half of code) |
| 🌤️ Afternoon | Code: run DSPy optimizer on a prompt task | Type and run all DSPy optimization examples |
| 🌙 Evening | Flashback | Write: "How does DSPy differ from manual prompt engineering? When would you use it?" |

**Key Concepts:** Programmatic prompt optimization, DSPy modules (ChainOfThought, ReAct), teleprompters
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** DSPy architecture (Signature → Module → Teleprompter → Optimizer), when NOT to use DSPy

---

### Day 28 — Specialized Patterns & Production Management

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Specialized Prompt Patterns | `08-Specialized-Prompt-Patterns.md` |
| 🌤️ Afternoon | Production Prompt Management | `09-Production-Prompt-Management.md` |
| 🌙 Evening | Flashback | Write: "What would a production prompt management system need?" (versioning, testing, deployment) |

**Key Concepts:** Specialized patterns (few-shot with retrieval, dynamic prompting), production prompt lifecycle
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Prompt versioning strategies, A/B testing in production, prompt CI/CD

---

### Day 29 — Project Part 1: Evaluation System

**All day = 🏗️ Project**

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Read project spec + design | `10-Project-Prompt-Engineering-System.md` — architecture phase |
| 🌤️ Afternoon | Build eval harness | Implement prompt evaluation system with test cases, scorers, and reporting |
| 🌙 Evening | Flashback + commit | Architecture decision log entry → git commit |

**Checklist:**
- [ ] Project spec read and architecture planned
- [ ] Eval harness implemented (runs multiple prompt variants, scores results)
- [ ] Code runs without errors
- [ ] Code pushed to GitHub

---

### Day 30 — Project Part 2: Prompt Management Dashboard

**All day = 🏗️ Project**

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Prompt versioning system | Build prompt storage, versioning, diffing |
| 🌤️ Afternoon | Management interface | Build the comparison/viewing layer for prompt versions |
| 🌙 Evening | Flashback + commit | What was the hardest part of prompt versioning? → git commit |

**Checklist:**
- [ ] Prompt versioning working (save, load, compare)
- [ ] Management UI/interface functional
- [ ] Code pushed to GitHub

---

### Day 31 — Project Part 3: Polish + Gate Check

**All day = 🏗️ Project**

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Testing | Unit tests + integration tests for the whole system |
| 🌤️ Afternoon | Documentation + Gate Check | 🎯 README, run through all gate checks |
| 🌙 Evening | Final commit + reflection | What would I do differently next time? |

**🎯 Gate Check — Can you:**
- [ ] Run an A/B comparison between 2 prompt variants and get a score?
- [ ] Version prompts and roll back to a previous version?
- [ ] Test a prompt against red-teaming attempts?
- [ ] Use DSPy to optimize a prompt automatically?
- [ ] Export evaluation results?
- [ ] Demonstrate at least 3 prompting techniques working?
- [ ] Identify and fix at least 2 prompt vulnerabilities?

---

### Day 32 — Phase 2 Review

**🔄 Phase-End + Weekly Review combined**

| Session | Focus | Activities |
|---------|-------|------------|
| ☀️ Morning | Scan + recall | Re-read headings of ALL 10 files. Re-type DSPy optimizer code from memory. |
| 🌤️ Afternoon | Mind map + weak spots | ✍️ Mind map Phase 2 from memory. Deep-dive on weak areas. |
| 🌙 Evening | 🧪 Self-test | Quiz yourself on all gate check questions. Note your score. |

**🧪 Self-Test Questions:**
1. What is context engineering and why is it more important than prompt phrasing?
2. Compare zero-shot, few-shot, and chain-of-thought — when does each fail?
3. What makes a good system prompt? Give 3 principles.
4. How do you systematically evaluate a prompt? What metrics matter?
5. What is DSPy and how does it optimize prompts differently than manual tuning?
6. Name 3 common prompt injection patterns and how to defend against each.
7. What's the difference between tree-of-thought and chain-of-thought?
8. How would you version prompts in a production system?
9. When should you NOT use DSPy?
10. What is the red-teaming methodology for prompts?

**Self-Assessment:**
- Total score (out of 10): ________
- Weakest area: ________
- Plan for weak area: ________
