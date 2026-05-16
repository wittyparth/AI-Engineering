# Tokenizers Deep Dive

## What Is a Tokenizer?

Maps text → integers (token IDs) for the model to process.

```
"Hello, world!" → [15496, 11, 995, 0]
```

## Critical Facts

### 1. Tokens ≠ Words
- 1 token ≈ 0.75 words for English
- "hello" = 1 token, "he" + "llo" = 2 tokens
- "I love AI" = 3 tokens, "I love artificial intelligence" = 6 tokens
- Different languages tokenize very differently

### 2. Token Count Is Bidirectional
- Input tokens cost ~1/3 of output tokens (OpenAI pricing)
- A 2000-token prompt ≠ 2000-token response
- Cost = (input_tokens × input_price) + (output_tokens × output_price)

### 3. Tokenizers Are Model-Specific
- GPT-4 uses `cl100k_base`
- Claude uses SentencePiece
- Llama uses a different BPE tokenizer
- Switching models changes token counts

## Practical Engineering

```python
# Count tokens before sending
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4")
tokens = enc.encode("Your prompt here")
len(tokens)  # Know your costs upfront

# 🔴 Senior: Always count tokens before sending.
# A 5% cost savings across millions of requests is significant.
# Implement token budgeting in your pipeline.
```

## Common Tokenization Traps

| Trap | Reality |
|------|---------|
| "My 500-word document will fit in the context" | 500 words ≈ 667 tokens + overhead |
| "All models use the same tokenizer" | Completely false. Token counts differ 20%+ |
| "Numbers are one token each" | "42" might be 1 token, "421" might be 2 |
| "Adding whitespace doesn't matter" | Tokenizers treat whitespace differently |

## Drill: Tokenizer Explorer

Build a CLI tool that:
- Takes text input
- Shows token IDs, segments, and counts for 3 different tokenizers
- Estimates cost for GPT-4o, Claude 3.5, and Llama 3
- Highlights surprising tokenizations
- Flags when you're approaching context limits
