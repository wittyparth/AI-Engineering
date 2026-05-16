# Reasoning Models (o3, R1, Gemini Thinking)

## What Makes Them Different

Standard LLM: "What's the next token?"
Reasoning model: Internal chain-of-thought before producing the answer.

```
Standard: "Q: 24×37 = ?  A: 888"
Reasoning: "Q: 24×37 = ?  <think>Hmm, 24×37. Let me break this down.
  24×30 = 720, 24×7 = 168. 720 + 168 = 888.</think> 888"
```

## Test-Time Compute

The key insight: **spend more compute at inference time to get better answers**.

| Model | Thinking Tokens | Accuracy (Math) | Cost |
|-------|----------------|-----------------|------|
| GPT-4o | 0 | 60% | 1x |
| o3-mini | 2K-10K | 85% | 3-5x |
| o3 (full) | 10K-50K | 95% | 20-50x |

## When Reasoning Models Matter

```
Complex math and logic       ✅ Dramatically better
Code generation              ✅ Better for complex algorithms
Multi-step planning          ✅ Much better
Simple Q&A                   ❌ Waste of compute
Creative writing             ❌ Worse (too analytical)
Translation                  ❌ No benefit
```

## Engineering for Reasoning Models

```python
# Different prompting strategy needed
# BAD for reasoning models:
response = llm.generate("What's the capital of France?")
# → thinks for 2000 tokens about whether Paris is REALLY the capital

# GOOD for reasoning models:
response = llm.generate(
    "What's the capital of France? Answer concisely.",
    thinking_budget=0,  # Skip thinking for simple questions
)
# → "Paris"

# For hard problems:
response = llm.generate(
    "Solve this complex math problem...",
    thinking_budget=10000,  # Allow substantial thinking
)
```

## 🔴 Senior: Cost Management

Reasoning models cost 3-50x more per query. Strategies:
1. **Route simple queries away**: Use a classifier to decide if reasoning is needed
2. **Set max thinking budget**: `thinking_budget=5000` limits cost
3. **Cache reasoning traces**: If the same reasoning works for similar queries, cache it
4. **Distill reasoning**: Fine-tune a cheaper model on reasoning traces (OpenAI's "distillation")
