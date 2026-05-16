# Chain-of-Thought Reasoning

## Why CoT Works

CoT effectively increases "test-time compute" — the model spends more tokens reasoning, which improves accuracy on complex tasks. It creates a scratchpad that allows the model to defer committing to an answer.

## CoT Variants

### 1. Standard CoT
```
Q: {question}
A: Let's think step by step.
```
Works well. Simple. Can be verbose.

### 2. Few-Shot CoT
```
Q: Roger has 5 tennis balls. He buys 2 more cans of tennis balls.
Each can has 3 tennis balls. How many does he have?
A: Roger starts with 5 balls. 2 cans of 3 each = 6 balls. 5 + 6 = 11.
The answer is 11.

Q: {question}
A:
```
Significantly more reliable than zero-shot CoT.

### 3. Self-Consistency
Generate multiple CoT paths, then majority-vote the answer:
```python
responses = [await llm.generate(CoT_prompt) for _ in range(5)]
answers = [parse_answer(r) for r in responses]
final = Counter(answers).most_common(1)[0][0]
```
Improves accuracy 5-15% at 5x cost. Worth it for critical decisions.

### 4. Tree-of-Thought (ToT)
Explore multiple reasoning branches and evaluate each:
```
Branch 1: {path A}
  Evaluation: {score}
Branch 2: {path B}
  Evaluation: {score}
Best path: {branch with highest score}
```
Expensive but powerful for complex planning tasks.

## 🔴 Senior: CoT Cost Analysis

| Strategy | Tokens per query | Accuracy | Cost multiplier |
|----------|-----------------|----------|-----------------|
| Direct | ~100 | 70% | 1x |
| CoT | ~500 | 85% | 3-5x |
| Self-consistency (5) | ~2500 | 92% | 15-20x |
| ToT | ~5000+ | 95% | 30-50x |

**Decision rule**: Use the cheapest strategy that passes your eval threshold. Only escalate for harder problems.

## Drill: CoT Quality Test

Create a test suite of 20 reasoning problems. Measure accuracy with:
1. Direct prompting
2. Zero-shot CoT
3. Few-shot CoT
4. Self-consistency (3 paths)

Report accuracy vs. cost for each. Where's the sweet spot for YOUR use case?
