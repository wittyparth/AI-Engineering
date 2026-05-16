# Prompt Design Patterns

## The Anatomy of a Prompt

```
[System Message] → Sets role, constraints, behavior
[Context]         → Documents, data, retrieved info
[Examples]        → Few-shot demonstrations
[Instruction]     → What to do, format, constraints
[Input]           → The user's actual query
```

## Pattern Catalog

### 1. Zero-Shot
```
Classify this email as spam or not spam:
Email: {email_text}
Classification:
```
Works for simple, well-defined tasks. Fails for nuanced or novel tasks.

### 2. Few-Shot
```
Classify emails:
Good: "Meeting tomorrow at 3pm?" → Not spam
Bad: "You won $10,000,000!!!" → Spam
Good: "Can you review the PR?" → Not spam
Bad: "BUY NOW LIMITED OFFER" → Spam

Email: {email_text}
Classification:
```
Dramatically improves accuracy (~15-30%). Example selection matters.

### 3. Chain-of-Thought (CoT)
```
Problem: A bat and ball cost $1.10. The bat costs $1.00 more than the ball.
How much does the ball cost?
Let's think step by step:
1. Let the ball cost x
2. Then the bat costs x + 1.00
3. Total: x + (x + 1.00) = 1.10
4. 2x + 1.00 = 1.10
5. 2x = 0.10
6. x = 0.05
Answer: $0.05
```
The biggest single quality improvement for reasoning tasks.

### 4. Persona/Role Prompting
```
You are a senior backend engineer reviewing a Python PR.
Your review must cover:
- Performance implications
- Security vulnerabilities
- Code maintainability
- Testing coverage
```
Constrains the model's "perspective" and improves domain-specific quality.

### 5. Structured Output Prompting
```
Extract the following fields as JSON:
- name: string
- age: integer (must be between 0-150)
- symptoms: array of strings
- severity: enum["mild", "moderate", "severe"]

Return ONLY valid JSON, no explanation.
```

## 🔧 Framework Integration: DSPy for Automated Optimization

After mastering manual prompt design, DSPy can automate prompt optimization against metrics. Instead of hand-tuning 5-shot examples, you define a module + metric and compile.

See `07-DSPy-Prompt-Optimization.md` for the deep-dive.

## 🔴 Senior: When Patterns Fail

| Pattern | Failure Mode | Fix |
|---------|-------------|-----|
| Zero-shot | Ambiguous outputs | Add constraint examples |
| Few-shot | Example bias | Diversify examples, shuffle order |
| CoT | Verbose, expensive | Limit reasoning steps |
| Persona | Over-acting | Soften the role definition |
| Structured | Invalid JSON | Use function calling + validation |
