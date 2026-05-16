# Security & Guardrails

## Threat Model for AI Systems

| Threat | Impact | Likelihood | Priority |
|--------|--------|-----------|----------|
| Prompt injection | Data leakage, unwanted actions | High | Critical |
| PII leakage | Compliance violation | Medium | Critical |
| Rate limit abuse | Cost spike, DoS | High | High |
| Model theft | IP loss | Low | High |
| Training data extraction | Private data leak | Low | Medium |

## Guardrail Implementation

### 1. Input Guardrails
```python
class InputGuardrail:
    def __init__(self):
        self.blocked_patterns = [
            r"ignore (all |previous )?instructions",
            r"you are now",
            r"pretend (you are|to be)",
            r"<\|im_start\|>",
            r"system prompt",
            # Add domain-specific patterns
        ]

    async def check(self, user_input: str) -> dict:
        for pattern in self.blocked_patterns:
            if re.search(pattern, user_input, re.IGNORECASE):
                return {"blocked": True, "reason": f"Detected: {pattern}"}

        # LLM-based classification for subtle attacks
        classification = await self.llm_classify(user_input)
        return classification

    async def llm_classify(self, text: str) -> dict:
        prompt = f"""Classify this user input:
        - "safe" : Normal question
        - "suspicious" : Potential injection attempt
        - "malicious" : Clear attack

        Input: {text}
        Classification:"""
        response = await self.llm.generate(prompt)
        return {"blocked": response.strip() != "safe", "reason": response}
```

### 2. Output Guardrails
```python
class OutputGuardrail:
    def __init__(self):
        self.pii_detector = PIIDetector()
        self.toxicity_classifier = ToxicityClassifier()

    async def check(self, model_output: str) -> dict:
        issues = []

        # Check for PII
        pii = self.pii_detector.detect(model_output)
        if pii:
            issues.append(f"PII detected: {pii}")
            model_output = self.pii_detector.redact(model_output)

        # Check for toxicity
        toxicity = self.toxicity_classifier.classify(model_output)
        if toxicity["score"] > 0.8:
            issues.append(f"Toxic content: {toxicity['category']}")

        return {
            "safe": len(issues) == 0,
            "issues": issues,
            "sanitized_output": model_output,
        }
```

### 3. Cost Control Guardrails
```python
class CostGuardrail:
    def __init__(self, max_daily_cost: float = 100.0):
        self.max_daily = max_daily_cost
        self.daily_spend = 0.0
        self.reset_time = datetime.utcnow().replace(hour=0, minute=0, second=0)

    async def check(self, estimated_cost: float) -> bool:
        self._reset_if_new_day()

        if self.daily_spend + estimated_cost > self.max_daily:
            logger.warning(f"Daily cost limit would be exceeded: "
                          f"{self.daily_spend + estimated_cost} > {self.max_daily}")
            return False

        self.daily_spend += estimated_cost
        return True
```

## 🔴 Senior: Defense in Depth

```
Layer 1: WAF (rate limiting, IP blocking)
Layer 2: Input validation (pattern matching, length limits)
Layer 3: LLM-based classification (nuanced attack detection)
Layer 4: Prompt design (delimiter-based separation)
Layer 5: Output validation (PII, toxicity, content policy)
Layer 6: Audit logging (post-hoc analysis)
Layer 7: Human review (for high-risk actions)
```

No single layer is sufficient. Each layer catches what others miss.

## Drill: Build a Security Test Suite

Create tests for:
1. 10 different prompt injection patterns
2. 5 PII types (email, phone, SSN, credit card, address)
3. 3 cost abuse scenarios (runaway loops, max budget bypass)
4. Rate limit bypass attempts
5. Output policy violations (NSFW, violent content)
