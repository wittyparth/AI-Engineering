# Phase 10: Advanced Topics

**The Problem**: You've mastered the core patterns. Now you need the frontier skills that separate senior/staff engineers — structured generation, reasoning models, multimodal, AI security, and emerging protocols.

**What You'll Build**: Projects covering constrained decoding (Outlines, Instructor), reasoning model pipelines, multimodal systems, and AI security defenses.

**Difficulty**: Hard-Very Hard | **Time**: ~12 hours

## The Build Path

```
1. STRUCTURED GENERATION: Outlines (constrained decoding) + Instructor (structured prompting + retry)
2. REASONING MODELS: o3, R1 — test-time compute, chain-of-thought distillation
3. MULTIMODAL: Text + images + audio pipelines
4. LONG-CONTEXT: Strategies when documents exceed context windows
5. HUMAN-IN-THE-LOOP: When to involve humans, designing the loop
6. AI SECURITY DEEP-DIVE: Prompt injection, data poisoning, model theft
7. EMERGING PATTERNS: MCP, A2A, AG-UI, AI governance
```

## Key Concepts

- **Outlines** — Constrained decoding (100% schema compliance for self-hosted models)
- **Instructor** — Structured prompting + retry with validation feedback (for API models)
- **Reasoning Models** — o3, R1, test-time compute, CoT distillation
- **Multimodal** — Beyond text: images, audio, video
- **Long-Context** — Context caching, sliding windows, summarization
- **HITL** — Active learning, annotation pipelines, escalation design
- **AI Security** — Adversarial robustness, red-teaming, model theft prevention
- **Emerging Protocols** — MCP (tool interface), A2A (agent-to-agent), AG-UI (user interface)

## Frameworks Integrated Here

- **Outlines** — Constrained decoding for local models
- **Instructor** — Structured prompting for API models
- **Pydantic AI** — Structured output with type safety

## 🔴 Senior: The Frontier is Moving Fast

Everything in this phase changes every 6 months. The goal is to build mental models and engineering patterns, not memorize APIs. Focus on:
- **What problem does this solve?**
- **When would I reach for this?**
- **What are the failure modes?**

## Gate Check

- [ ] Implemented constrained decoding (Outlines) for a use case
- [ ] Implemented structured prompting (Instructor) for API models
- [ ] Compared Outlines vs Instructor — know when to use each
- [ ] Can explain how reasoning models differ from standard LLMs
- [ ] Built a simple multimodal pipeline (text + image)
- [ ] Understands when to use human-in-the-loop vs fully automated
- [ ] Can describe 3 major AI security threats and their mitigations
