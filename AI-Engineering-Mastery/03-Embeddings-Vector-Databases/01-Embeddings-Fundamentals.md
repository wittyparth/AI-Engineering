# Embeddings Fundamentals

## What Is an Embedding?

A vector (list of numbers) that represents the "meaning" of text. Semantically similar text has numerically similar vectors.

```
"king"     → [0.32, -0.12, 0.87, ..., 0.04]  (1536 dimensions)
"queen"    → [0.31, -0.10, 0.85, ..., 0.06]  (close to king)
"bicycle"  → [-0.45, 0.72, -0.33, ..., 0.12] (far from king)
```

## How Embeddings Are Created

1. **Input text** → tokenized
2. **Transformer encoder** → contextualized representations
3. **Pooling layer** → single vector from token representations
4. **Normalization** → unit vector (optional but common)

## Properties

- **Dimensionality**: 384 (all-MiniLM-L6-v2) to 3072 (text-embedding-3-large)
- **Fixed length**: Same dimension regardless of input length
- **Direction matters**: Cosine similarity measures angle, not magnitude
- **Context matters**: "bank" (river) ≠ "bank" (money) — different embeddings

## Embedding Quality Dimensions

| Quality | Good | Bad |
|---------|------|-----|
| Semantic similarity | "car" ≈ "automobile" | "car" ≈ "apple" |
| Out-of-domain | Works on legal text too | Only works on training data |
| Cross-lingual | "chat" (fr) ≈ "cat" (en) | Different languages separate |
| Length robustness | Short + long texts comparable | Short texts dominate |

## 🔴 Senior: Embedding Trap

Embedding models are trained on general data. If your data is domain-specific (medical, legal, code), general embedding models underperform by 10-20%. Always benchmark on YOUR data.
