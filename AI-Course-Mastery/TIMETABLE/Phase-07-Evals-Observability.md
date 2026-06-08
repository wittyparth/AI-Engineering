# Phase 7 — Production Evals & Observability (Days 84-95)

**Goal:** Build evaluation-first mindset into everything — LLM-as-judge, eval pipelines, production observability, regression detection — build an eval platform.
**Files:** 9 concept + 1 project (10 files — all concept files >1000 lines)
**Total days:** 12 study days + 1 phase-end review

---

### Day 84 — Evals-First Mindset

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Module overview + Evals-First Mindset | `00-Module-Overview.md`, `01-Evals-First-Mindset.md` |
| 🌤️ Afternoon | Code: build a minimal eval harness | Implement: test case → model → judge → score pipeline |
| 🌙 Evening | Flashback | Write: "What does 'evals-first' mean in practice? How does it change development workflow?" |

**Key Concepts:** Eval-driven development, eval as CI/CD gate, cost of NOT evaluating
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Eval-first vs test-first vs code-first, eval ROI calculation

---

### Day 85 — LLM-as-Judge (Full Day — Large File)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | LLM-as-Judge | `02-LLM-as-Judge.md` (concepts: judge prompting, bias, calibration) |
| 🌤️ Afternoon | Code: implement 3 judge types | Pairwise comparison, single-point grading, reference-based grading |
| 🌙 Evening | Flashback | Write: "What biases do LLM judges have? How do you mitigate them?" |

**Key Concepts:** Judge architectures, position bias, self-enhancement bias, judge calibration, multi-judge
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Judge bias categories, judge selection (same model vs different, GPT-4 judge vs Claude)

---

### Day 86 — Metrics That Matter

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Metrics That Matter | `03-Metrics-That-Matter.md` |
| 🌤️ Afternoon | Code: implement a metrics dashboard | Track: accuracy, latency, cost, hallucination rate, user satisfaction proxy |
| 🌙 Evening | Flashback | Write: "What 5 metrics would you track for a production chatbot? Why those 5?" |

**Key Concepts:** Metrics hierarchy (proxy vs outcome), leading vs lagging indicators, cost-aware metrics
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Metric selection framework, vanity metrics vs actionable metrics

---

### Day 87 — Evaluation Datasets

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Evaluation Datasets | `04-Evaluation-Datasets.md` |
| 🌤️ Afternoon | Code: build eval dataset generator | Golden test set creation, synthetic data generation, adversarial examples |
| 🌙 Evening | Flashback | Write: "What makes a good evaluation dataset? How do you avoid data leakage?" |

**Key Concepts:** Golden dataset design, synthetic data, data leakage, dataset versioning
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Dataset quality vs quantity, human-curated vs synthetic, dataset drift

---

### Day 88 — Building Eval Pipelines (Full Day — Large File)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Building Eval Pipelines | `05-Building-Eval-Pipelines.md` (architecture) |
| 🌤️ Afternoon | Code: build production eval pipeline | Automated eval runs, reporting, regression detection |
| 🌙 Evening | Flashback | Write: "Architecture of my eval pipeline — walk through from trigger to report" |

**Key Concepts:** CI/CD for evals, parallel eval execution, result aggregation, regression alerts
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Eval triggers (commit, schedule, manual), pipeline stages, parallelization

---

### Day 89 — Production Observability

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Production Observability | `06-Production-Observability.md` |
| 🌤️ Afternoon | Code: integrate Langfuse | Tracing, logging, cost tracking, custom event tracking |
| 🌙 Evening | Flashback | Write: "What's the difference between monitoring and observability for LLM systems?" |

**Key Concepts:** Traces, spans, logging, observability vs monitoring, Langfuse architecture
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Observability pillars (traces, metrics, logs), sampling strategies, cost of observability

---

### Day 90 — Regression Detection

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Regression Detection | `07-Regression-Detection.md` |
| 🌤️ Afternoon | Code: implement regression detector | Automated comparison of eval runs, statistical significance, alerting |
| 🌙 Evening | Flashback | Write: "How do you detect when a model update silently degrades quality?" |

**Key Concepts:** Regression types (quality, cost, latency), statistical significance, A/B comparison in evals
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Detecting regressions without golden data, when a change isn't a regression

---

### Day 91 — Custom Eval Framework

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Custom Eval Framework | `08-Custom-Eval-Framework.md` |
| 🌤️ Afternoon | Code: build a reusable eval framework | Pluggable judges, test runners, report generators |
| 🌙 Evening | Flashback | Write: "If you had to design an eval framework from scratch, what would the core abstractions be?" |

**Key Concepts:** Eval framework architecture, plugin system, result schemas, reporter abstractions
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Eval framework vs ad-hoc evals, framework flexibility vs complexity tradeoff

---

### Day 92-93 — Project: Eval Platform (2 days)

**All days = 🏗️ Project** — `09-Project-Eval-Platform.md`

#### Day 92 — Project: Core Framework

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Read project spec + design | Architecture: eval runner, judges, reporters, storage |
| 🌤️ Afternoon | Build eval runner + judge system | Implement test execution, multiple judge types, scoring |
| 🌙 Evening | Flashback + commit | Framework architecture → git commit |

**Checklist:** Spec read ☐ | Eval runner working ☐ | Judges working ☐ | GitHub updated ☐

#### Day 93 — Project: Storage + Reports + Gate Check

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Eval history + trends | Store eval results, track trends over time |
| 🌤️ Afternoon | Reports + Gate Check | 🎯 Report generation, visualization, gate checks |
| 🌙 Evening | Final commit + reflection | How would I extend this in production? |

**🎯 Gate Check — Can you:**
- [ ] Run automated eval suites on demand?
- [ ] Use LLM-as-judge with at least 2 judge architectures?
- [ ] Track and report eval metrics over time?
- [ ] Detect regressions by comparing eval runs?
- [ ] Integrate with Langfuse for tracing?
- [ ] Generate human-readable eval reports?
- [ ] Handle adversarial eval datasets?
- [ ] Explain eval-first development workflow?

---

### Day 94-95 — Phase 7 Review

#### Day 94 — Phase-End Review

| Session | Focus | Activities |
|---------|-------|------------|
| ☀️ Morning | Scan + recall | Re-read ALL 10 file headings. Re-type LLM-as-judge from memory. |
| 🌤️ Afternoon | Mind map + weak spots | ✍️ Mind map Phase 7. Deep-dive on weak areas. |
| 🌙 Evening | 🧪 Self-test | Quiz yourself. |

#### Day 95 — Eval Practice Day

| Session | Focus | Activities |
|---------|-------|------------|
| ☀️ Morning | Build an eval from scratch | Choose an earlier project (RAG bot, agent) and write an eval for it |
| 🌤️ Afternoon | Run + report | Run the eval, generate a report, identify 1 regression |
| 🌙 Evening | Reflection | How has my thinking about evaluation changed? |

**🧪 Self-Test Questions:**
1. What is eval-first development? How does it change the workflow?
2. How does LLM-as-judge work? What biases exist and how do you mitigate them?
3. Name 5 metrics you'd track for an LLM system and why each matters
4. What makes a good eval dataset? How do you avoid data leakage?
5. Architecture of an eval pipeline — walk through from trigger to report
6. What's the difference between observability and monitoring for LLM systems?
7. How do you detect a regression without golden data?
8. What would a custom eval framework's core abstractions be?
9. How do you integrate evals into CI/CD?
10. When is human evaluation better than LLM-as-judge?

**Self-Assessment:**
- Total score (out of 10): ________
- Weakest area: ________
