# Cornerstone Project: Secure Enterprise AI

**Time**: 3-4 days  **Difficulty**: Hard  **Portfolio**: Yes

## The Problem

Build an enterprise-grade AI system that can handle sensitive data, resist attacks, and comply with security policies. This project combines everything from Phase 8 (Deployment Security) and Phase 10 (Security Deep-Dive).

## Requirements

### Must Have
- [ ] Multi-layer defense against prompt injection (input + prompt + output)
- [ ] PII detection and redaction on inputs and outputs
- [ ] Rate limiting (per-user, per-IP, per-endpoint)
- [ ] Cost budget enforcement (per-user, daily limits)
- [ ] Audit logging (every request logged with user, query, response, cost)
- [ ] API key authentication
- [ ] Content policy enforcement (block certain topics)

### Should Have
- [ ] Automated security test suite in CI
- [ ] Token budget (max tokens per response)
- [ ] Allow/block list for models
- [ ] Human review queue for flagged requests
- [ ] Data retention policies (auto-delete logs after N days)

### Nice to Have
- [ ] Penetration test report
- [ ] SOC2/ISO27001 readiness documentation
- [ ] Data sovereignty support (EU data stays in EU)
- [ ] Model access controls (which users can use which models)

## Architecture

```
User → Auth (API Key) → Rate Limiter → Input Guardrail → Prompt Builder
                                                              ↓
LLM → Output Guardrail → PII Redactor → Audit Log → Response
                                    ↓
                              Human Review
                            (for flagged items)
```

## Guardrail Implementation

```python
class GuardrailSystem:
    def __init__(self, config: SecurityConfig):
        self.input_guard = InputGuardrail(config)
        self.pii_detector = PIIDetector()
        self.content_filter = ContentFilter(config.blocked_topics)
        self.output_guard = OutputGuardrail(config)
        self.audit = AuditLogger()

    async def process(self, request: Request) -> Response:
        # 1. Authenticate
        user = await self.authenticate(request)

        # 2. Rate limit
        await self.rate_limiter.check(user.id)

        # 3. Cost check
        await self.cost_guard.check(user.id, request.estimated_cost)

        # 4. Input guardrail
        safe_input = await self.input_guard.check(request.query)

        # 5. Generate
        response = await self.llm.generate(request.query)

        # 6. Output guardrail
        safe_response = await self.output_guard.check(response)

        # 7. PII redaction
        clean_response = await self.pii_detector.redact(safe_response)

        # 8. Log everything
        await self.audit.log(user, request, response, clean_response)

        return clean_response
```

## 🔴 Senior: Security Is a System Property

Security isn't a feature you add. It's a property of the entire system design:
- Every input is a potential attack vector
- Every output is a potential leak
- Every dependency is a potential vulnerability
- Every human is a potential insider threat

**Design for failure**: Assume the attacker will get through one layer. Have the next layer ready.

## Design Decisions

1. **Block vs sanitize**: When input is suspicious, block it or clean it?
2. **Hard vs soft blocks**: Hard error or graceful degradation?
3. **Logging detail**: Full query text or hashed? (PII compliance)
4. **Review queue**: Auto-escalate or manual review?
5. **Cost limits**: Hard stop or warning?
6. **Model access**: All users get all models or tiered access?

## Reflection

1. What attack vectors did you discover that you hadn't considered?
2. How did security measures affect latency and UX?
3. What would a sophisticated attacker still be able to bypass?
4. How would you respond to a confirmed security incident?
5. What's your top priority improvement?
