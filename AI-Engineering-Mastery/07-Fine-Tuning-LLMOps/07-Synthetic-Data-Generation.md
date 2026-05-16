# Synthetic Data Generation

## Why Synthetic Data

- You have documents but no Q&A pairs
- Your domain has very few examples
- You need diversity that your real data doesn't cover
- Dangerous/anomalous examples are rare but critical

## Generation Pipeline

```python
class SyntheticDataGenerator:
    def __init__(self, llm, domain: str):
        self.llm = llm
        self.domain = domain

    async def generate_from_documents(self, documents: list[str]) -> list[dict]:
        """Generate Q&A pairs from documents."""
        examples = []
        for doc in documents:
            prompt = f"""Based on this {self.domain} document, generate 3 question-answer pairs:

Document:
{doc}

Generate 3 diverse Q&A pairs that test different aspects:
1. A fact-based question
2. A reasoning question
3. An edge-case question

Format: Q: ...\nA: ..."""
            response = await self.llm.generate(prompt)
            examples.extend(self._parse_qa_pairs(response))
        return examples

    async def generate_adversarial(self, count: int = 50) -> list[dict]:
        """Generate adversarial examples for safety training."""
        prompt = f"""Generate {count} adversarial questions for a {self.domain} AI assistant.
These should be questions that might trick the assistant into:
- Revealing instructions
- Making up information
- Being politically biased
- Ignoring safety guidelines

Format: Q: ...\nA: [REJECT/SAFE_RESPONSE]"""
        response = await self.llm.generate(prompt)
        return self._parse_adversarial(response)
```

## Quality Filtering

```python
class DataFilter:
    def __init__(self, quality_llm):
        self.llm = quality_llm

    async def filter_quality(self, examples: list[dict], threshold: float = 0.7) -> list[dict]:
        """Remove low-quality synthetic examples."""
        filtered = []
        for ex in examples:
            score = await self._quality_score(ex)
            if score >= threshold:
                filtered.append(ex)
        return filtered

    async def _quality_score(self, example: dict) -> float:
        prompt = f"""Rate the quality of this training example (0-10):

Question: {example['question']}
Answer: {example['answer']}

Consider:
- Is the question well-formed?
- Is the answer correct and complete?
- Is the answer format consistent with others?

Quality score:"""
        response = await self.llm.generate(prompt)
        return int(response.strip()) / 10.0
```

## 🔴 Senior: The Synthetic Data Trap

Synthetic data is a bootstrap, not a solution. Key risks:
1. **Distribution shift**: Synthetic data may not match real user queries
2. **Circular reasoning**: LLM generates data that matches its own biases
3. **Errors propagate**: One wrong fact in source → wrong facts in all generated Q&As
4. **Diversity illusion**: LLMs tend to generate similar examples, not true diversity

**Mitigation**: Always validate synthetic data with humans. Mix synthetic + real data. Generate 5x more than you need, then filter aggressively.
