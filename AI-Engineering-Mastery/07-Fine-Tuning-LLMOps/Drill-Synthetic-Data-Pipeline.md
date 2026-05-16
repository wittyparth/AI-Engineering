# Drill: Synthetic Data Pipeline

**Time**: 30 min  **Difficulty**: Medium

## Task

Build a pipeline that generates, filters, and validates synthetic training data.

```python
class SyntheticPipeline:
    def __init__(self, llm, domain: str):
        self.llm = llm
        self.domain = domain
        self.generator = SyntheticDataGenerator(llm, domain)
        self.filter = DataFilter(llm)

    async def run(self, documents: list[str], target: int = 500) -> list[dict]:
        # 1. Generate 5x target
        print(f"Generating from {len(documents)} documents...")
        raw_examples = await self.generator.generate_from_documents(documents)

        # 2. Diversify (ensure coverage)
        diverse = await self._diversify(raw_examples)

        # 3. Filter quality
        print(f"Filtering {len(diverse)} examples...")
        filtered = await self.filter.filter_quality(diverse)

        # 4. Deduplicate
        deduped = self._deduplicate(filtered)

        # 5. Validate format
        validated = self._validate_format(deduped)

        print(f"Generated {len(raw_examples)} → Filtered {len(filtered)} → Validated {len(validated)}")
        return validated[:target]

    def _deduplicate(self, examples: list[dict]) -> list[dict]:
        seen = set()
        unique = []
        for ex in examples:
            q_hash = hashlib.md5(ex["question"].encode()).hexdigest()
            if q_hash not in seen:
                seen.add(q_hash)
                unique.append(ex)
        return unique

    def _validate_format(self, examples: list[dict]) -> list[dict]:
        valid = []
        for ex in examples:
            if "question" in ex and "answer" in ex:
                if len(ex["question"]) > 10 and len(ex["answer"]) > 10:
                    valid.append(ex)
        return valid
```

## Test

Run on 10 documents. How many valid examples do you get? What's the quality?
Manually review 20 examples and categorize: excellent / acceptable / poor.
