# Context Windows & Attention

## What Is a Context Window?

The maximum number of tokens the model can "see" at once. Everything outside this window does not exist to the model.

| Model | Context Window |
|-------|---------------|
| GPT-4o | 128K tokens |
| Claude 3.5 Sonnet | 200K tokens |
| Gemini 1.5 Pro | 2M tokens |
| Llama 3.1 405B | 128K tokens |
| Mistral Large 2 | 128K tokens |

## The Attention Problem

O(n²) complexity means long contexts are expensive:

| Context Length | Attention Ops | Relative Cost |
|---------------|---------------|---------------|
| 4K | 16M | 1x |
| 32K | 1B | 64x |
| 128K | 16B | 1024x |

🔴 **Senior**: Just because the model supports 128K doesn't mean you should send 128K. "Needle in a haystack" benchmarks measure retrieval, not understanding. Models reliably lose information in the middle of long contexts (the "lost in the middle" problem).

## Practical Strategies

### 1. Position Matters
- Beginning and end of context have highest recall
- Middle is where information gets lost
- Put the MOST important information at the START and END

### 2. Context Budgeting
```
Total context = system_prompt + conversation_history + retrieved_docs + user_query
You must manage ALL of these simultaneously.
```

### 3. The Lost in the Middle Problem
[Research paper](https://arxiv.org/abs/2307.03172): When relevant information is in the middle of a long context, retrieval accuracy drops by up to 20% compared to beginning/end placement.

### 4. Positional Encoding Breakdown
- Absolute position (original Transformer): Hard limits
- RoPE (Rotary Position Embedding): Better extrapolation
- ALiBi: Linear bias, not position (handles length extrapolation)
- YaRN / NTK-aware scaling: Extending pre-trained context windows

## Drill: Context Window Stress Test

Write a script that:
1. Creates documents of varying lengths
2. Places the answer at beginning, middle, and end of context
3. Measures retrieval accuracy at each position
4. Plots the "lost in the middle" curve for 3 different models
5. Reports the effective context window (where accuracy drops below 80%)
