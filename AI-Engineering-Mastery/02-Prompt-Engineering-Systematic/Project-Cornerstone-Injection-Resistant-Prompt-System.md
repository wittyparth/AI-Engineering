# Cornerstone Project: Injection-Resistant Prompt System

**Time**: 2-3 days  **Difficulty**: Medium-Hard  **Portfolio**: Yes

## The Problem

You're building a customer-facing chatbot. Users will try to trick it. Executives will blame you when it reveals internal instructions. You need a hardened system.

## Requirements

### Must Have
- [ ] Prompt template registry (YAML-based, versioned, git-friendly)
- [ ] Input sanitization pipeline (strip injection patterns)
- [ ] Injection detection (classify input as safe/suspicious/malicious)
- [ ] Response guardrail (check output before sending to user)
- [ ] FastAPI endpoints for rendering + generating
- [ ] Template rendering with variable substitution
- [ ] Audit logging (which template + version for each request)
- [ ] Test suite with 15+ injection attack patterns

### Should Have
- [ ] A/B testing between prompt variants
- [ ] Canary deployment for new prompt versions
- [ ] Rollback capability
- [ ] Token + cost tracking per template

### Nice to Have
- [ ] Dashboard showing prompt performance over time
- [ ] Auto-detection of prompt drift (output quality changing)
- [ ] Integration with Langfuse for traceability

## Architecture

```
User Input → Sanitizer → Injection Detector → Prompt Renderer → LLM → Guardrail → User
                ↓               ↓                                        ↓
              Audit          Alert                                  Block/Modify
```

## 🔴 Senior: Defense In Depth

Single-layer defense is insufficient. You need:
1. **Input layer**: Reject or sanitize
2. **Prompt layer**: Delimiter-based separation
3. **Output layer**: Check response before sending
4. **Audit layer**: Log everything for post-hoc analysis
5. **Human layer**: Escalate suspicious cases

## Design Decisions

1. **Block vs. sanitize**: When a user tries injection, do you block or sanitize? (Blocking tells attackers they're on the right track)
2. **Detection granularity**: Three tiers (safe/suspicious/malicious) or binary?
3. **False positive tolerance**: Better to block a legitimate user or risk an injection?
4. **Template variable escaping**: How do you handle user input that contains template syntax?
5. **Evaluation metrics**: What's your threshold for "safe enough"?

## Code Review Rubric

- Input sanitization catches all 15 attack patterns
- Template versioning works end-to-end
- Guardrail catches common output violations
- Audit log is complete and queryable
- Tests exist for each defense layer
- System degrades gracefully when LLM fails

## Reflection

1. What attack patterns surprised you?
2. How would you handle zero-day injection attacks?
3. What's the cost in latency for each defense layer?
4. How do you balance security with user experience?
