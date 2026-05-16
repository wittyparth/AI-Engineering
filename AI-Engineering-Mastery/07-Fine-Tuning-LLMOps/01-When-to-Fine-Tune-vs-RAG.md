# When to Fine-Tune vs RAG

## Decision Matrix

```
                     Can the model learn this from context?
                              /       \
                            YES        NO
                            /           \
                      Use RAG     Does this require the model
                                  to BE something different?
                                        /       \
                                      YES        NO
                                      /           \
                              Fine-Tune     Is this a knowledge
                                            that doesn't change?
                                                  /       \
                                                YES        NO
                                                /           \
                                           RAG (+ FT    Fine-Tune
                                           for format)  + RAG
```

## Comparison

| Factor | RAG | Fine-Tuning |
|--------|-----|-------------|
| Knowledge freshness | ✅ Always current | ❌ Frozen at training time |
| Private data security | ✅ Data stays in your DB | ⚠️ Data goes to model weights |
| Implementation time | Days | Weeks |
| Cost to run | Per-query (tokens) | Per-inference (model size) |
| Cost to build | Low (DB + pipeline) | High (GPU + data + training) |
| Quality for rare facts | High (exact docs retrieved) | Low (may hallucinate) |
| Quality for tone/style | Low (prompt engineering) | High (model learns the style) |
| Iteration speed | Fast (change prompt/db) | Slow (re-train) |
| Scalability | Easy (scale DB) | Complex (multi-GPU) |

## Common Patterns

### Pattern 1: RAG Only
```
- Knowledge changes frequently
- You need citations
- Limited budget
- Standard output format is fine
```

### Pattern 2: Fine-Tuning Only
```
- Consistent behavior needed (tone, format, structure)
- Knowledge is stable
- Low latency requirements
- Offline/edge deployment
```

### Pattern 3: Both (Most Production Systems)
```
- Fine-tune for behavior (tone, format, instruction following)
- RAG for knowledge (current data, private docs)
- Results: Best of both worlds
```

🔴 **Senior**: Most teams waste months debating this. Do RAG first. It's faster, cheaper, and you'll learn what your model actually needs. Only fine-tune when RAG hits a clear wall.
