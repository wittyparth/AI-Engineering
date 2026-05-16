# RAG Evaluation (RAGAS)

## Why Traditional Metrics Don't Work

- Accuracy: binary, doesn't capture partial correctness
- BLEU/ROUGE: n-gram overlap, meaningless for generative answers
- Human eval: expensive, not reproducible

## RAGAS Metrics

### 1. Faithfulness
Does the answer stay true to the retrieved context? (0-1)

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

result = evaluate(
    dataset=dataset,
    metrics=[faithfulness, answer_relevancy, context_precision],
)
print(result)  # {'faithfulness': 0.89, 'answer_relevancy': 0.92, ...}
```

### 2. Answer Relevancy
How relevant is the answer to the question?

### 3. Context Precision
How much of the retrieved context is actually relevant?

### 4. Context Recall
How much relevant information was retrieved?

## Building Your Eval Dataset

```python
from datasets import Dataset

eval_data = {
    "question": ["What is RAG?", "How does chunking work?", ...],
    "answer": [..., ..., ...],  # from your RAG system
    "contexts": [[...], [...], ...],  # retrieved chunks
    "ground_truth": [..., ..., ...],  # ideal answer (gold standard)
}

dataset = Dataset.from_dict(eval_data)
```

## LLM-as-Judge

```python
async def faithfulness_eval(question: str, answer: str, context: str) -> float:
    prompt = f"""Rate the faithfulness of this answer on a scale of 0-5.
Faithfulness means: does the answer stay true to the context, without hallucination?

Context: {context}
Question: {question}
Answer: {answer}

Faithfulness score (0-5):"""
    response = await llm.generate(prompt)
    return int(response.strip()) / 5.0
```

## 🔴 Senior: Eval-Driven Development

1. Build 50-100 eval cases (with ground truth)
2. Run baseline RAG → record scores
3. Make one change (chunking, embedding, prompt, etc.)
4. Re-run eval → compare scores
5. Only keep changes that IMPROVE scores

This is the scientific method for RAG engineering. Without evals, you're guessing.
