# Data Preparation for Fine-Tuning

## The 80% Rule

80% of fine-tuning success is data quality. 20% is training configuration. Most failed fine-tuning attempts fail because of bad data, not bad training.

## Data Format

### Chat Template Format
```json
{
    "messages": [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is FastAPI?"},
        {"role": "assistant", "content": "FastAPI is a modern Python web framework..."}
    ]
}
```

### Completion Format
```json
{
    "prompt": "Q: What is FastAPI?\nA:",
    "completion": " FastAPI is a modern Python web framework..."
}
```

## Data Quality Checklist

### ✅ Good Data
- Diverse examples (cover all use cases)
- Correct formatting (consistent across all examples)
- Appropriate length (matches expected usage)
- Clean (no typos, no contradictions)
- Representative (matches production distribution)

### ❌ Bad Data
- Repetitive examples (model memorizes, doesn't generalize)
- Formatting inconsistencies (mixes json/yaml/text)
- Too short/long (doesn't match usage patterns)
- Contains errors (teaches bad behavior)
- Leaked test data (invalid evaluation)

## Data Augmentation

```python
def augment_dataset(dataset: list[dict], llm) -> list[dict]:
    """Generate additional training examples from existing ones."""
    augmented = []
    for example in dataset:
        # Vary the phrasing
        variants = llm.generate(
            f"Rewrite this question 3 ways: {example['question']}"
        )
        for variant in parse_list(variants):
            augmented.append({
                "question": variant,
                "answer": example["answer"],
            })
    return augmented
```

## 🔴 Senior: Data Quality Over Quantity

| Dataset Size | Quality | Result |
|-------------|---------|--------|
| 100 | High | Surprisingly good for narrow tasks |
| 1,000 | High | Good for most tasks |
| 10,000 | High | Excellent, but diminishing returns |
| 100,000 | Low | May make things WORSE (noise) |
| 1,000,000 | Low | Definitely worse (signal drowns in noise) |

100 high-quality examples > 10,000 low-quality examples.

## Drill: Dataset Creation Pipeline

Build a pipeline that:
1. Takes raw domain documents
2. Extracts Q&A pairs (using LLM)
3. Validates format and quality
4. Deduplicates
5. Splits train/validation/test
6. Outputs in Hugging Face Dataset format
