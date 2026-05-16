# Few-Shot Engineering

## The Hidden Complexity

Most people think few-shot = "just add examples." In reality, few-shot quality depends on:

### 1. Example Selection
- **Random** examples: +10-15% over zero-shot
- **Similar** examples (semantically close to query): +20-30%
- **Diverse** examples (cover edge cases): +15-25%
- **Adversarial** examples (hard cases): +5-10% but better robustness

🔴 **Senior**: Dynamic example selection (retrieve similar examples from a database) beats static few-shot by 10-15%.

### 2. Example Ordering
- Putting the most relevant example LAST (closest to query) improves accuracy
- Random ordering can vary accuracy by 5-10%
- Cluster similar examples together vs. interleave them?

### 3. Label Balance
If you have 3 classes and give 2 examples of class A and 1 of class B, the model biases toward A.

### 4. Format Consistency
```
Bad: Example 1 - spam | Example 2 - NOT SPAM
Good: "Buy now!!!" → spam
       "Meeting at 3" → not spam
```

## Dynamic Few-Shot Selection

```python
class DynamicFewShot:
    def __init__(self, example_db: list[dict], embedder):
        self.examples = example_db
        self.embedder = embedder

    def select(self, query: str, k: int = 3) -> list[dict]:
        query_emb = self.embedder.embed(query)
        scored = [
            (ex, cosine_similarity(query_emb, self.embedder.embed(ex["input"])))
            for ex in self.examples
        ]
        scored.sort(key=lambda x: x[1], reverse=True)
        return [ex for ex, _ in scored[:k]]
```

## Drill: Few-Shot Comparison

Build a test harness that compares:
1. Zero-shot
2. Static few-shot (same 3 examples every time)
3. Dynamic few-shot (semantically similar examples)
4. Random few-shot (3 random examples per query)

Test on 100 queries. Report accuracy per strategy.
