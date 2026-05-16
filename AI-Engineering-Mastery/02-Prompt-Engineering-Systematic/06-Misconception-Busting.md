# Misconception Busting

## What Most People Get Wrong

### 1. "Longer prompts are better"
Reality: Longer prompts increase cost, increase latency, dilute signal-to-noise ratio, and increase the chance of hallucination. The shortest prompt that passes your test suite is the best prompt.

### 2. "Temperature controls creativity"
Reality: Temperature controls token probability distribution. High temperature makes all tokens more equally likely (more "random"). Low temperature concentrates probability on the most likely tokens. It doesn't add creativity — it adds noise.

```python
# temperature=0: deterministic (mostly)
# temperature=0.7: balanced
# temperature=1.5: near-random gibberish
```

### 3. "Prompt engineering is about getting the perfect first try"
Reality: Prompt engineering is an iterative process. The first prompt is always wrong. Plan for iteration.

### 4. "System prompt is more important than user prompt"
Reality: The most recent instructions (user prompt) often override system instructions. This is called "instruction dominance."

### 5. "You can prompt your way out of any problem"
Reality: Some problems need RAG (not enough knowledge), fine-tuning (not the right behavior), or human review (too risky). Know when prompting isn't the answer.

### 6. "Few-shot examples teach the model"
Reality: Few-shot examples prime the model's output distribution. They don't teach. The model already "knows" — examples just guide it toward the right part of its distribution.

## 🔴 Senior: The Prompt Trap

The most dangerous belief: "I can fix this with a better prompt."

Symptoms:
- Adding more and more constraints to the system prompt
- "Please, please, please output valid JSON"
- Five-paragraph system prompt with contradictory instructions

The fix: If the prompt is longer than 2000 tokens, you probably need a different approach (RAG, fine-tuning, or a different model).
