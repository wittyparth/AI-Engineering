# Phase 11 — Capstone (Days 134-154)

**Goal:** Build and deploy your own production AI product end-to-end.
**Files:** 1 overview + 1 options detail + 3 PRDs (5 files)
**Total days:** 21 study days + 1 final review

---

## Phase Structure

The Capstone is divided into 5 milestones:

| Milestone | Days | Deliverable |
|-----------|------|-------------|
| 1 — Selection + Planning | 134-136 | PRD, architecture, project plan |
| 2 — Core Build | 137-143 | Working prototype with core features |
| 3 — Advanced Features | 144-148 | Full feature set + guardrails |
| 4 — Evaluation + Polish | 149-151 | Testing, eval, docs |
| 5 — Deployment + Launch | 152-154 | Deployed, monitoring, README |

---

### Day 134 — Project Selection

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Read all capstone materials | `00-Capstone-Overview.md`, `01-Project-Options-Details.md` |
| 🌤️ Afternoon | Read all 3 PRDs | `Recall-PRD.md`, `Aria-PRD.md`, `RepoChat-PRD.md` |
| 🌙 Evening | Decision + initial ideas | Choose your project. Write: "Why this project? What will success look like?" |

**Checklist:**
- [ ] All capstone materials read
- [ ] All 3 PRDs read
- [ ] Project selected: ________
- [ ] Reasoning documented

---

### Day 135 — PRD Deep Dive + Architecture

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Re-read your chosen PRD deeply | Highlight architecture decisions, data model, AI pipeline |
| 🌤️ Afternoon | Architecture design | Draw system architecture, component diagram, data flow |
| 🌙 Evening | Document architecture | Write ARCHITECTURE.md — decisions, tradeoffs, open questions |

**Checklist:**
- [ ] PRD fully understood with no open questions
- [ ] Architecture diagram drawn
- [ ] ARCHITECTURE.md written
- [ ] GitHub repo initialized with structure

---

### Day 136 — Project Plan + Environment Setup

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Build project plan | Break into tasks, estimate days, set milestones |
| 🌤️ Afternoon | Environment setup | Docker, dependencies, CI/CD, dev scripts |
| 🌙 Evening | Final prep | Everything ready to build tomorrow |

**Checklist:**
- [ ] Task breakdown written (can use the day-by-day in this file as template)
- [ ] Dev environment working
- [ ] CI/CD pipeline set up (minimal: lint + test on push)
- [ ] Secrets/API keys configured

---

### Day 137 — Core: Data Layer + Schemas

**All day = 🏗️ Build**

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Core data models | Pydantic schemas, database models, migrations |
| 🌤️ Afternoon | Data ingestion pipeline | Document loading, processing, storage |
| 🌙 Evening | Test + commit | Schema tests pass → git commit |

**Checklist:** Schemas defined ☐ | Data pipeline working ☐ | Tests pass ☐ | GitHub updated ☐

---

### Day 138 — Core: AI Pipeline Foundation

**All day = 🏗️ Build**

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | LLM integration | API client, provider abstraction, streaming |
| 🌤️ Afternoon | Embedding + retrieval pipeline | Vector DB, embedding function, search |
| 🌙 Evening | Test + commit | AI pipeline end-to-end test → git commit |

**Checklist:** LLM integration ☐ | Embedding/retrieval ☐ | E2E test passes ☐ | GitHub updated ☐

---

### Day 139 — Core: Core Feature Implementation

**All day = 🏗️ Build**

| Session | Focus | Tasks |
|-------|-------|-------|
| ☀️ Morning | Primary feature implementation | Your app's main AI feature (e.g., Q&A, search, generation) |
| 🌤️ Afternoon | Secondary features | Supporting features for MVP |
| 🌙 Evening | Test + commit | Feature tests pass → git commit |

**Checklist:** Primary feature working ☐ | Secondary features working ☐ | Tests ☐ | GitHub updated ☐

---

### Day 140 — Core: API Layer

**All day = 🏗️ Build**

| Session | Focus | Tasks |
|-------|-------|-------|
| ☀️ Morning | API endpoints | FastAPI routes, request/response models, error handling |
| 🌤️ Afternoon | API testing | Integration tests, response format validation |
| 🌙 Evening | Test + commit | All API endpoints tested → git commit |

**Checklist:** All endpoints implemented ☐ | Request validation ☐ | Response format correct ☐ | Tests ☐ | GitHub updated ☐

---

### Day 141 — Core: Frontend/Interface (If applicable)

**All day = 🏗️ Build**

| Session | Focus | Tasks |
|-------|-------|-------|
| ☀️ Morning | UI setup + core components | Next.js app, main views, API integration |
| 🌤️ Afternoon | Streaming + interactivity | SSE streaming, loading states, error handling |
| 🌙 Evening | Polish + commit | UI functional → git commit |

**Checklist:** Core UI working ☐ | API integration ☐ | Streaming ☐ | Error states ☐ | GitHub updated ☐

---

### Day 142 — Core: Integration + Bug Fixes

**All day = 🏗️ Build**

| Session | Focus | Tasks |
|-------|-------|-------|
| ☀️ Morning | End-to-end integration | Wire everything together: UI → API → AI → DB |
| 🌤️ Afternoon | Bug hunt | Find and fix all visible bugs |
| 🌙 Evening | Working prototype milestone | 🏆 Milestone 2 complete. Run the whole thing yourself. |

**Checklist:** E2E working ☐ | Bugs fixed ☐ | Working prototype demonstrable ☐ | GitHub updated ☐

---

### Day 143 — Buffer / Makeup Day

Use this day if behind. If ahead: start advanced features early.

**If on track:** Take the day off or work on advanced features.

---

### Day 144 — Advanced: Guardrails + Safety

**All day = 🏗️ Build**

| Session | Focus | Tasks |
|-------|-------|-------|
| ☀️ Morning | Input guardrails | Prompt injection defense, input validation |
| 🌤️ Afternoon | Output guardrails | Content moderation, PII redaction, output validation |
| 🌙 Evening | Test + commit | Guardrail tests → git commit |

**Checklist:** Input guardrails ☐ | Output guardrails ☐ | Guardrail tests ☐ | GitHub updated ☐

---

### Day 145 — Advanced: Evaluation System

**All day = 🏗️ Build**

| Session | Focus | Tasks |
|-------|-------|-------|
| ☀️ Morning | Evaluation harness | Build eval system for your app: test cases, judges, scoring |
| 🌤️ Afternoon | Run baseline eval | Establish quality baseline, identify weak spots |
| 🌙 Evening | Analysis + commit | Eval results documented → git commit |

**Checklist:** Eval harness built ☐ | Baseline scores recorded ☐ | Weak areas identified ☐ | GitHub updated ☐

---

### Day 146 — Advanced: Monitoring + Observability

**All day = 🏗️ Build**

| Session | Focus | Tasks |
|-------|-------|-------|
| ☀️ Morning | Langfuse/Observability integration | Tracing, logging, cost tracking |
| 🌤️ Afternoon | Production monitoring | Metrics, dashboards, alerting basics |
| 🌙 Evening | Test + commit | Monitoring pipeline tested → git commit |

**Checklist:** Langfuse integration ☐ | Cost tracking ☐ | Metrics dashboard ☐ | GitHub updated ☐

---

### Day 147 — Advanced: Performance Optimization

**All day = 🏗️ Build**

| Session | Focus | Tasks |
|-------|-------|-------|
| ☀️ Morning | Profile + identify bottlenecks | Latency profiling, cost analysis |
| 🌤️ Afternoon | Optimize top 3 bottlenecks | Cache, query optimization, model tiering, batching |
| 🌙 Evening | Benchmark + commit | Before/after comparison → git commit |

**Checklist:** Bottlenecks identified ☐ | Top 3 optimized ☐ | Improvement measured ☐ | GitHub updated ☐

---

### Day 148 — Advanced: Polish + Edge Cases

**All day = 🏗️ Build**

| Session | Focus | Tasks |
|-------|-------|-------|
| ☀️ Morning | Edge case handling | Empty states, error states, rate limiting, long-running operations |
| 🌤️ Afternoon | UX polish | Loading states, error messages, empty states, responsive design |
| 🌙 Evening | Full feature milestone | 🏆 Milestone 3 complete. Full feature set working. |

**Checklist:** Edge cases handled ☐ | UX polish done ☐ | Full feature set ☐ | GitHub updated ☐

---

### Day 149 — Eval: Deep Testing

**All day = 🧪 Testing**

| Session | Focus | Tasks |
|-------|-------|-------|
| ☀️ Morning | Unit + integration tests | Comprehensive test coverage for all features |
| 🌤️ Afternoon | Adversarial testing | Try to break your app: bad inputs, edge cases, concurrent requests |
| 🌙 Evening | Test report + commit | Test coverage report → git commit |

**Checklist:** Unit tests comprehensive ☐ | Integration tests ☐ | Adversarial tests run ☐ | All tests pass ☐ | GitHub updated ☐

---

### Day 150 — Eval: Quality Assessment

**All day = 🧪 Evaluation**

| Session | Focus | Tasks |
|-------|-------|-------|
| ☀️ Morning | Run full eval suite | All test cases, all judges, full report |
| 🌤️ Afternoon | Quality improvements | Fix issues found by eval — iterate |
| 🌙 Evening | Eval report + commit | Quality scores documented → git commit |

**Checklist:** Full eval run ☐ | Quality scores recorded ☐ | Improvements made ☐ | GitHub updated ☐

---

### Day 151 — Eval: Documentation

**All day = 📝 Documentation**

| Session | Focus | Tasks |
|-------|-------|-------|
| ☀️ Morning | README.md | Architecture, setup, usage, environment variables |
| 🌤️ Afternoon | API docs + architecture decision log | Document key decisions, tradeoffs, learnings |
| 🌙 Evening | Documentation milestone | 🏆 Milestone 4 complete. Every document written. |

**Checklist:** README written ☐ | API docs ☐ | Architecture decisions documented ☐ | GitHub updated ☐

---

### Day 152 — Deploy: Infrastructure Setup

**All day = 🚀 Deployment**

| Session | Focus | Tasks |
|-------|-------|-------|
| ☀️ Morning | Docker Compose finalization | Production-ready Docker config for all services |
| 🌤️ Afternoon | Cloud deployment | Deploy to AWS/GCP/your preferred platform |
| 🌙 Evening | Verify deployment | App is live and responding → git commit |

**Checklist:** Docker Compose production-ready ☐ | Deployed ☐ | Live URL working ☐ | GitHub updated ☐

---

### Day 153 — Deploy: Monitoring + Launch Prep

**All day = 🚀 Deployment**

| Session | Focus | Tasks |
|-------|-------|-------|
| ☀️ Morning | Production monitoring | Verify monitoring works on deployed instance |
| 🌤️ Afternoon | Launch checklist | Security review, cost review, backup plan, rollback plan |
| 🌙 Evening | Launch readiness | 🏆 Milestone 5 of 5. Everything is ready. |

**Checklist:** Monitoring working on production ☐ | Security review done ☐ | Rollback plan ☐ | Launch-ready ☐

---

### Day 154 — Launch + Final Review

| Session | Focus | Tasks |
|-------|-------|-------|
| ☀️ Morning | Launch! | Deploy final version, verify everything working |
| 🌤️ Afternoon | Full system walkthrough | Run through EVERY feature yourself |
| 🌙 Evening | Final reflection 🎉 | Write: "What I learned building this. What I'd do differently. What's next." |

**🎯 Capstone Gate Check — Can you:**
- [ ] The app is live and working?
- [ ] User can complete the primary use case end-to-end?
- [ ] Guardrails protect against common attacks?
- [ ] Evaluation pipeline measures quality?
- [ ] Monitoring tracks cost, latency, errors?
- [ ] Documentation explains architecture and setup?
- [ ] Tests cover core functionality?
- [ ] You can explain every major design decision?
- [ ] You know what you'd improve next?

**Final Log:**
- Project URL: ________
- Total build time: ________ days
- Biggest challenge: ________
- Best decision: ________
- What I'd do differently: ________
- What I'm most proud of: ________

---

## 🎉 Congratulations — You Did It!

This capstone is the culmination of everything you've learned. You are now an AI Engineer.

**You now know:**
- How LLMs work under the hood
- Systematic prompt engineering
- Embeddings + vector databases
- RAG from foundations to advanced
- AI agents + multi-agent systems
- Production evaluation + observability
- Guardrails + safety systems
- Fine-tuning + LLMOps
- Deployment + scale

**Your portfolio has 11 production projects.**

The course is complete. Your journey as an AI Engineer is just beginning.
