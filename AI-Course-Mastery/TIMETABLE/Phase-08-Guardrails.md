# Phase 8 — Guardrails, Safety & Security (Days 96-106)

**Goal:** Master prompt injection defense, input/output guardrails, PII redaction, content moderation — build a Safe AI Gateway.
**Files:** 9 concept + 1 project (10 files — many large, up to 2102 lines)
**Total days:** 11 study days + 1 phase-end review

---

### Day 96 — Threat Modeling

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Module overview + Threat Modeling | `00-Module-Overview.md`, `01-Threat-Modeling.md` |
| 🌤️ Afternoon | Code: build a threat model for your LLM Gateway project | Identify threats, rank by severity, document mitigations |
| 🌙 Evening | Flashback | Write: "What are the 4 biggest security threats to an LLM application?" |

**Key Concepts:** Threat modeling framework, trust boundaries, attack surface, OWASP LLM top 10
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** OWASP LLM categories, threat modeling as continuous practice, trust boundary identification

---

### Day 97 — Prompt Injection (Full Day — Large File)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Prompt Injection | `02-Prompt-Injection.md` (attack types + theory) |
| 🌤️ Afternoon | Code: implement + test injection defenses | Build: input sanitization, instruction separation, prompt shield |
| 🌙 Evening | Flashback | Write: "Describe 3 types of prompt injection attacks and a defense for each." |

**Key Concepts:** Direct vs indirect injection, jailbreak patterns, defense layers, delimiter-based separation
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Injection taxonomy, defense-in-depth, known jailbreak patterns

---

### Day 98 — Input Guardrails

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Input Guardrails | `03-Input-Guardrails.md` |
| 🌤️ Afternoon | Code: build input guardrail system | Content filtering, prompt validation, user intent classification |
| 🌙 Evening | Flashback | Write: "What inputs should you block vs transform vs allow through?" |

**Key Concepts:** Input validation pipeline, content classification, allow/deny lists, semantic filtering
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Guardrail placement (pre-LLM vs post-LLM), false positive/negative tradeoffs

---

### Day 99 — Output Guardrails

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Output Guardrails | `04-Output-Guardrails.md` |
| 🌤️ Afternoon | Code: build output guardrail system | Output validation, content policy enforcement, output transformation |
| 🌙 Evening | Flashback | Write: "What's harder to guard — input or output? Why?" |

**Key Concepts:** Output validation strategies, structured output enforcement, content policy checking
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Input vs output guardrail asymmetry, output transformation strategies

---

### Day 100 — PII Redaction (Full Day — Large File)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | PII Redaction | `05-PII-Redaction.md` (PII types, detection strategies, legal context) |
| 🌤️ Afternoon | Code: build PII redaction system | Regex + NER + custom patterns, redaction vs masking vs anonymization |
| 🌙 Evening | Flashback | Write: "What types of PII exist? How do you choose between redaction, masking, and anonymization?" |

**Key Concepts:** PII taxonomy, detection (regex, NER, LLM-based), redaction strategies, compliance requirements
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** PII detection accuracy tradeoffs, context-aware redaction, regulatory considerations

---

### Day 101 — Content Moderation

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Content Moderation | `06-Content-Moderation.md` |
| 🌤️ Afternoon | Code: build moderation pipeline | OpenAI moderation API, custom classifier, escalation workflows |
| 🌙 Evening | Flashback | Write: "What content categories should you moderate? How do you handle gray areas?" |

**Key Concepts:** Content moderation categories, moderation APIs, classifier pipelines, human-in-the-loop
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Automated vs human moderation, false positive handling, category taxonomy

---

### Day 102 — Guardrails Architecture

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Guardrails Architecture | `07-Guardrails-Architecture.md` |
| 🌤️ Afternoon | Code: design + implement guardrail chain | Input guardrails → LLM call → output guardrails — as a composable pipeline |
| 🌙 Evening | Flashback | Write: "Draw the architecture of a complete guardrail system from memory" |

**Key Concepts:** Composable guardrail pipeline, guardrail ordering, guardrail performance, fail-open vs fail-closed
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Guardrail composition patterns, fail-closed vs fail-closed decision, latency budget

---

### Day 103 — Testing Guardrails

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Testing Guardrails | `08-Testing-Guardrails.md` |
| 🌤️ Afternoon | Code: build guardrail test suite | Red-team tests, regression tests, adversarial test dataset |
| 🌙 Evening | Flashback | Write: "How do you test a guardrail system? What makes a good guardrail test?" |

**Key Concepts:** Red-teaming methodology, adversarial testing, guardrail eval metrics, regression detection
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Guardrail metrics (precision, recall, F1), adversarial dataset creation

---

### Day 104-105 — Project: Safe AI Gateway (2 days)

**All days = 🏗️ Project** — `09-Project-Safe-AI-Gateway.md`

#### Day 104 — Project: Core Guardrails

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Read project spec + design | Architecture: guardrail chain, PII system, moderation |
| 🌤️ Afternoon | Build input + output guardrails | Implement guardrail pipeline with at least 3 guardrail types |
| 🌙 Evening | Flashback + commit | Architecture decisions → git commit |

**Checklist:** Spec read ☐ | Input guardrails working ☐ | Output guardrails working ☐ | GitHub updated ☐

#### Day 105 — Project: PII + Moderation + Gate Check

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | PII + moderation | Implement PII redaction + content moderation pipeline |
| 🌤️ Afternoon | Testing + docs + Gate Check | 🎯 Red-team testing, README, gate checks |
| 🌙 Evening | Final commit + reflection | What attack would still get through my guardrails? |

**🎯 Gate Check — Can you:**
- [ ] Block prompt injection attacks?
- [ ] Redact PII from both input and output?
- [ ] Moderate content (block harmful, allow safe)?
- [ ] Chain guardrails in a configurable pipeline?
- [ ] Test guardrails with adversarial inputs?
- [ ] Report guardrail metrics (block rate, false positive rate)?
- [ ] Handle fail-closed vs fail-open decisions?
- [ ] Measure guardrail latency overhead?

---

### Day 106 — Phase 8 Review

**🔄 Phase-End + Weekly Review combined**

| Session | Focus | Activities |
|---------|-------|------------|
| ☀️ Morning | Scan + recall | Re-read ALL 10 file headings. Re-type guardrail pipeline from memory. |
| 🌤️ Afternoon | Mind map + weak spots | ✍️ Mind map Phase 8. Re-read weak sections. |
| 🌙 Evening | 🧪 Self-test | Quiz yourself. |

**🧪 Self-Test Questions:**
1. What are the 4 biggest security threats to an LLM application?
2. How does prompt injection work? Name 3 defense strategies
3. What's the difference between input guardrails and output guardrails?
4. How do you detect and redact PII? What are the tradeoffs of different methods?
5. What content moderation categories matter for a general-purpose AI app?
6. Architecture of a guardrail pipeline — walk through from request to response
7. How do you test guardrails? What metrics matter?
8. What's the difference between fail-closed and fail-open? When would you choose each?
9. How do you handle false positives in guardrails?
10. What's the latency budget for guardrails in a production system?

**Self-Assessment:**
- Total score (out of 10): ________
- Weakest area: ________
