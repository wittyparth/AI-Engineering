# Transformer Architecture Intuition

## The Attention Mechanism

"Attention" lets each token look at every other token and decide what's relevant.

```
"The cat sat on the ___"
                  ↑
            "mat" gets info from "cat" (subject),
            "sat" (action), "on" (position)
```

## Key Components

### 1. Self-Attention
Every token computes:
- **Query** (Q): "What am I looking for?"
- **Key** (K): "What do I contain?"
- **Value** (V): "What do I pass along?"

Relevance = softmax(Q × K^T / sqrt(d)) × V

🔴 **Senior**: This is why quadratic complexity matters. Q×K^T is O(n²). Doubling the context quadruples the compute. This is the fundamental bottleneck all context management strategies fight.

### 2. Multi-Head Attention
8-128 parallel attention computations. Each head learns a different relationship type (syntax, semantics, position).

### 3. Feed-Forward Network
After attention, each token gets independently transformed. Where attention mixes information between tokens, FFN processes what each token "understands."

### 4. Layer Normalization + Residual Connections
Prevents gradients from vanishing in deep networks. Residual path ensures information can flow directly through the network.

## Why This Matters for AI Engineering

You don't need to implement a transformer. But when you:
- Hit context limits → you know it's O(n²) attention, not arbitrary
- See weird behavior with long inputs → you understand positional encoding breakdown
- Read about "Flash Attention" → you understand it's optimizing the Q×K^T bottleneck
- See "RoPE scaling" → you understand it's modifying positional encoding for longer contexts
