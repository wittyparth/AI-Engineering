# How LLMs Actually Work

## The Core Idea

An LLM is a next-token predictor. Given a sequence of tokens, it predicts the most likely next token. That's it. Everything else — reasoning, coding, creativity — emerges from scale and training data.

## Training Pipeline (Conceptual)

```
Raw Text → Tokenization → Pre-training → Fine-tuning → Alignment (RLHF/DPO)
```

1. **Pre-training**: Predict next token on trillions of words (unsupervised)
2. **Fine-tuning**: Teach specific behaviors (supervised)
3. **Alignment**: Make outputs helpful, harmless, honest (RLHF/DPO)

## What an LLM Cannot Do

- It cannot "think" — it generates plausible tokens
- It cannot "know" — it has no internal state beyond the forward pass
- It cannot "remember" — the context window is not memory
- It cannot "verify" — it will confidently hallucinate

🔴 **Senior**: The distinction between "predicting tokens" and "reasoning" is the single most important mental model. Every AI engineering failure traces back to forgetting this.

## Key Implications for Engineering

| Property | Engineering Impact |
|----------|-------------------|
| Deterministic-ish (temp=0) | Caching works for exact repeats |
| Context window is limited | Must manage what goes in |
| No inherent truth | RAG, citations, grounding |
| Token cost ≠ compute cost | Input tokens are cheaper than output |
| Latency scales with output length | Streaming is essential for UX |
