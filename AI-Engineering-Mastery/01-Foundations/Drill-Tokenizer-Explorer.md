# Drill: Tokenizer Explorer

**Time**: 30 min  **Difficulty**: Easy

## Task

Build a CLI tool that:
1. Takes a text string as input
2. Tokenizes it with 3 different tokenizers (tiktoken, sentencepiece, and a third)
3. Shows the breakdown: token IDs, token text, counts per model
4. Estimates cost for GPT-4o, Claude 3.5, Llama 3
5. Flags if any context limit is exceeded

## Starter

```python
import tiktoken
import json

def explore_tokenizers(text: str):
    # OpenAI tokenizer
    enc = tiktoken.encoding_for_model("gpt-4o")
    tokens = enc.encode(text)
    print(f"GPT-4o: {len(tokens)} tokens")
    print(f"  IDs: {tokens[:20]}{'...' if len(tokens) > 20 else ''}")
    print(f"  Decoded: {[enc.decode([t]) for t in tokens[:10]]}")

    # Claud tokenizer (using tiktoken cl100k_base approximation)
    enc2 = tiktoken.get_encoding("cl100k_base")
    tokens2 = enc2.encode(text)
    print(f"Claude (approx): {len(tokens2)} tokens")

    # Cost estimation
    cost_gpt4o = (len(tokens) / 1_000_000) * 2.50
    cost_claude = (len(tokens2) / 1_000_000) * 3.00
    print(f"Estimated cost: GPT-4o=${cost_gpt4o:.6f}, Claude=${cost_claude:.6f}")

if __name__ == "__main__":
    text = input("Enter text to tokenize: ")
    explore_tokenizers(text)
```

## Challenge Mode
Add a "surprising tokenizations" section that finds cases where common words split weirdly.

## Cost Table
| Model | Input $/M tokens | Output $/M tokens |
|-------|-----------------|------------------|
| GPT-4o | $2.50 | $10.00 |
| GPT-4o-mini | $0.15 | $0.60 |
| Claude 3.5 Sonnet | $3.00 | $15.00 |
| Llama 3 70B (self-hosted) | ~$0.20 (compute) | ~$0.20 |
