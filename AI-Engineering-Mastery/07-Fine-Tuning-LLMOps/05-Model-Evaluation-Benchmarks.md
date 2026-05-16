# Model Evaluation & Benchmarks

## Before and After Comparison

```python
class FineTuneEvaluation:
    def __init__(self, base_model, finetuned_model, test_set: list[dict]):
        self.base = base_model
        self.finetuned = finetuned_model
        self.test_set = test_set

    async def compare(self) -> dict:
        results = {"base": [], "finetuned": []}
        for example in self.test_set:
            base_response = await self.base.generate(example["prompt"])
            ft_response = await self.finetuned.generate(example["prompt"])

            results["base"].append({
                "prompt": example["prompt"],
                "response": base_response,
                "expected": example["expected"],
                "score": await self._score(example["expected"], base_response),
            })
            results["finetuned"].append({
                "response": ft_response,
                "score": await self._score(example["expected"], ft_response),
            })

        return {
            "base_avg_score": mean(r["score"] for r in results["base"]),
            "finetuned_avg_score": mean(r["score"] for r in results["finetuned"]),
            "improvement": results["finetuned_avg_score"] - results["base_avg_score"],
        }
```

## Domain-Specific Evals

For fine-tuning, general benchmarks (MMLU, HellaSwag) are less important than DOMAIN-SPECIFIC metrics:

```python
domain_evals = {
    "format_adherence": "...",
    "domain_accuracy": "...",
    "tone_consistency": "...",
    "instruction_following": "...",
    "refusal_rate": "...",  # For safety fine-tuning
}
```

## MT-Bench Style Evaluation

```python
async def mt_bench_eval(model, questions: list[dict]) -> float:
    total_score = 0
    for q in questions:
        response = await model.generate(q["prompt"])
        score = await judge_llm.generate(
            f"Rate the quality of this response (1-10):\n"
            f"Question: {q['prompt']}\n"
            f"Response: {response}\n"
            f"Category: {q['category']}\n"
            f"Score:"
        )
        total_score += int(score.strip())
    return total_score / len(questions)
```

## 🔴 Senior: The Eval Trap

The most common mistake: comparing fine-tuned model against base model using the SAME prompts that were in the training data. The fine-tuned model will appear to win because it memorized.

**Always use a held-out test set — data the model has NEVER seen during training.**

## Drill: Model Comparison Framework

1. Create 30 test prompts (not in training data)
2. Generate responses from base model AND fine-tuned model
3. Blind test: have a human (or LLM judge) rate each response without knowing which is which
4. Report: which model wins, by how much, on which types of questions
