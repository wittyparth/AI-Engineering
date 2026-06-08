# Phase 1 — Foundations & LLM Gateway (Days 5-21)

**Goal:** How LLMs work under the hood, API integration patterns, streaming, structured outputs — then build a Multi-Provider LLM Gateway.
**Files:** 9 concept + 2 drills + 1 project (13 files total, including 3303-line project)
**Total days:** 17 study days + 1 phase-end review

---

### Day 5 — How LLMs Work

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | How LLMs Work | `01-How-LLMs-Work.md` |
| 🌤️ Afternoon | Module overview + reinforcement | `00-Module-Overview.md` (scan), re-read LLM internals section |
| 🌙 Evening | Flashback | Write: "Explain how an LLM generates text from scratch" |

**Key Concepts to Lock:** Autoregressive generation, next-token prediction, transformer basics
**Checklist:**
- [ ] Morning session done — typed key code examples
- [ ] Afternoon session done — module overview scanned
- [ ] 🌙 Flashback completed
- [ ] Progress log updated

**Revision points:** Transformer architecture, training vs inference, scaling laws intuition

---

### Day 6 — Tokenizers (Concept + Drill)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Tokenizers Deep Dive | `02-Tokenizers-Deep-Dive.md` |
| 🌤️ Afternoon | Tokenizer Drill | `Drill-Tokenizer-Explorer.md` (build a tokenizer explorer CLI) |
| 🌙 Evening | Flashback | Write: "Why do tokenizers matter for production?" + 1 insight from the drill |

**Key Concepts to Lock:** BPE/WordPiece, tokenization edge cases, token-count-cost relationship
**Checklist:**
- [ ] Morning — read + typed BPE example
- [ ] Afternoon — drill completed (code running)
- [ ] 🌙 Flashback completed
- [ ] Progress log updated

**Revision points:** BPE merge rules, common tokenization pitfalls, multilingual tokenization

---

### Day 7 — Context Windows & Model Families

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Context Windows & Attention | `03-Context-Windows-Attention.md` |
| 🌤️ Afternoon | Model Families & Tradeoffs | `04-Model-Families-Tradeoffs.md` |
| 🌙 Evening | Flashback | Write: "What is the attention mechanism in 3 sentences?" + compare 2 model families |

**Key Concepts to Lock:** Attention mechanism, context window limits, model family selection criteria
**Checklist:**
- [ ] Morning session done
- [ ] Afternoon session done
- [ ] 🌙 Flashback completed
- [ ] Progress log updated

**Revision points:** KV cache, positional encoding, RoPE, model comparison table

---

### Day 8 — Structured Outputs (Full Day)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Structured Outputs Basics | `05-Structured-Outputs-Basics.md` |
| 🌤️ Afternoon | Code heavy — build JSON parser + schema enforcement | Re-implement key examples from the file |
| 🌙 Evening | Flashback | Write: "3 ways to get structured output from an LLM and when to use each" |

**Key Concepts to Lock:** JSON mode, function calling, constrained decoding, Pydantic integration
**Checklist:**
- [ ] Morning — read + typed examples
- [ ] Afternoon — working structured output code
- [ ] 🌙 Flashback completed
- [ ] Progress log updated

**Revision points:** Constrained decoding vs JSON mode vs function calling tradeoffs

---

### Day 9 — Python Patterns & API Integration

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Python AI Patterns | `06-Python-AI-Patterns.md` |
| 🌤️ Afternoon | API Integration Patterns | `07-API-Integration-Patterns.md` |
| 🌙 Evening | Flashback | Write: "What is the retry+fallback pattern for LLM APIs?" |

**Key Concepts to Lock:** Async patterns, rate limiting, retry with exponential backoff, provider SDK patterns
**Checklist:**
- [ ] Morning session done
- [ ] Afternoon session done — typed API integration code
- [ ] 🌙 Flashback completed
- [ ] Progress log updated

**Revision points:** Circuit breaker pattern, async vs sync, error types (retryable vs not)

---

### Day 10 — Streaming & CLI Drill

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Streaming, SSE & WebSockets | `08-Streaming-SSE-WebSockets.md` |
| 🌤️ Afternoon | Build CLI Chat Drill | `Drill-Build-CLI-Chat.md` |
| 🌙 Evening | Flashback | Write: "How does SSE streaming work end-to-end?" |

**Key Concepts to Lock:** SSE protocol, streaming token by token, WebSocket vs SSE tradeoffs
**Checklist:**
- [ ] Morning — read + typed streaming server
- [ ] Afternoon — CLI chat application working
- [ ] 🌙 Flashback completed
- [ ] Progress log updated

**Revision points:** Chunk parsing, backpressure, abort signals, partial token handling

---

### Day 11 — Framework Landscape + Project Kickoff

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | AI Production Framework Landscape | `09-AI-Production-Framework-Landscape.md` |
| 🌤️ Afternoon | Read project spec + plan architecture | `Project-Cornerstone-Multi-Provider-LLM-Gateway.md` (read through architecture section) |
| 🌙 Evening | Flashback + project notes | Sketch your project architecture diagram from memory |

**Key Concepts to Lock:** Framework comparison (LangChain vs LlamaIndex vs custom), when to use each
**Project Prep Checklist:**
- [ ] Read the full project spec
- [ ] Architecture diagram sketched
- [ ] Decided: which providers to support first
- [ ] Repo initialized on GitHub

---

### Day 12 — Project: Schemas + Provider Abstraction

**All day = 🏗️ Project**

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Core schemas | Define Pydantic models: `LLMRequest`, `LLMResponse`, `ProviderConfig`, `ModelConfig` |
| 🌤️ Afternoon | Abstract provider interface | Build `BaseProvider` with `generate()` and `generate_stream()` abstract methods |
| 🌙 Evening | Flashback + commit | What did the architecture force me to rethink? → git commit |

**Checklist:**
- [ ] Pydantic schemas defined and tested
- [ ] BaseProvider abstract class written
- [ ] Project runs without errors
- [ ] Code pushed to GitHub

---

### Day 13 — Project: Provider Implementations

**All day = 🏗️ Project**

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | OpenAI provider | Implement OpenAI provider (sync + streaming) |
| 🌤️ Afternoon | Anthropic + Gemini providers | Implement at least 2 more providers |
| 🌙 Evening | Flashback + commit | What was different between provider APIs? → git commit |

**Checklist:**
- [ ] OpenAI provider working
- [ ] Anthropic provider working
- [ ] Gemini/other provider working
- [ ] All providers return consistent `LLMResponse` format
- [ ] Code pushed to GitHub

---

### Day 14 — Project: Request Routing + Fallbacks

**All day = 🏗️ Project**

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Router logic | Build `Router` class that selects provider by model name |
| 🌤️ Afternoon | Fallback chain | Implement fallback on failure: primary → secondary → error |
| 🌙 Evening | Flashback + commit | How did I handle error cases? → git commit |

**Checklist:**
- [ ] Router selects correct provider
- [ ] Fallback chain works (test by killing primary API key)
- [ ] All errors handled gracefully
- [ ] Code pushed to GitHub

---

### Day 15 — Project: Streaming + Error Handling

**All day = 🏗️ Project**

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Streaming end-to-end | Wire streaming through the entire chain (provider → router → response) |
| 🌤️ Afternoon | Error handling | Add retry logic, timeout handling, rate limit detection |
| 🌙 Evening | Flashback + commit | What streaming edge cases did I discover? → git commit |

**Checklist:**
- [ ] Streaming works through all providers
- [ ] SSE response format is consistent
- [ ] Retry works on transient errors
- [ ] Timeouts handled gracefully
- [ ] Code pushed to GitHub

---

### Day 16 — Project: Testing

**All day = 🏗️ Project**

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Unit tests | Test each provider individually, test schemas, test router |
| 🌤️ Afternoon | Integration tests | Test end-to-end flow with real API calls (use cheap models) |
| 🌙 Evening | Flashback + commit | What test caught a bug I would have missed? → git commit |

**Checklist:**
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Edge cases tested (empty response, API error, timeout)
- [ ] Code pushed to GitHub

---

### Day 17 — Project: Polish + Gate Check

**All day = 🏗️ Project**

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Documentation | Write README: architecture, setup, usage, provider config |
| 🌤️ Afternoon | Gate Check + cleanup | 🎯 Run through project gate check. Clean up TODOs, add type hints. |
| 🌙 Evening | Final commit + reflection | Write a learning summary in your own words |

**🎯 Gate Check — Can your gateway:**
- [ ] Route requests to at least 3 different providers?
- [ ] Handle streaming responses from all providers?
- [ ] Fall back gracefully when a provider fails?
- [ ] Return consistent response format regardless of provider?
- [ ] Handle rate limits, timeouts, and API errors?
- [ ] Pass all unit and integration tests?
- [ ] Have clear documentation for setup and usage?

**Log this:**
- Project repo URL: ________
- Number of providers implemented: ________
- What was the hardest bug? ________

---

### Day 18 — Phase 1 Review

**🔄 Phase-End + Weekly Review combined**

| Session | Focus | Activities |
|---------|-------|------------|
| ☀️ Morning | Scan + recall | Re-read headings of ALL 13 files. Re-type tokenizer code from memory. |
| 🌤️ Afternoon | Mind map + weak spots | ✍️ Mind map Phase 1 from memory. Identify top 3 fuzzy concepts — re-read those sections. |
| 🌙 Evening | 🧪 Self-test | Quiz yourself on all gate check questions. Note your score. |

**🧪 Self-Test Questions:**
1. Explain autoregressive generation in one paragraph (no notes)
2. What is BPE tokenization and why does it create variable-length tokens?
3. How does the attention mechanism work? Draw the diagram from memory.
4. Compare GPT-4, Claude 3, and Gemini — when would you use each?
5. What's the difference between JSON mode and function calling for structured outputs?
6. Describe the retry+fallback pattern for LLM API calls
7. How does SSE streaming work at the protocol level?
8. When would you use WebSockets instead of SSE?
9. Name 3 production considerations when building a multi-provider gateway
10. What is the circuit breaker pattern and why does it matter for LLM APIs?

**Self-Assessment:**
- Total score (out of 10): ________
- Weakest area: ________
- Plan for weak area: ________ (re-read at next review)
