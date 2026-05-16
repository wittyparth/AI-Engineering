# Systematic Prompt Testing

## The Problem

"Does this prompt work?" — if you can't answer this with data, you're guessing.

## Test Types

### 1. Unit Tests for Prompts
```python
def test_classification_prompt():
    prompt = classifier_prompt("Buy now!!! Click here!!!")
    response = llm.generate(prompt, temperature=0)
    assert "spam" in response.lower()

def test_extraction_prompt():
    prompt = extract_prompt("John is 30 years old and lives in NY")
    response = llm.generate(prompt, temperature=0)
    data = json.loads(response)
    assert data["name"] == "John"
    assert data["age"] == 30
```

### 2. Regression Test Suite
Run every prompt change against a fixed dataset of 50-100 examples.

```python
class PromptRegressionTest:
    def __init__(self):
        self.test_cases = [
            {"input": "What is AI?", "expected_topics": ["artificial intelligence"]},
            {"input": "How do I cook pasta?", "expected_topics": ["cooking", "pasta"]},
        ]

    def run(self, prompt_template: str) -> dict:
        results = {}
        for case in self.test_cases:
            prompt = prompt_template.format(query=case["input"])
            response = llm.generate(prompt)
            results[case["input"]] = self._evaluate(response, case)
        return results

    def _evaluate(self, response: str, expected: dict) -> bool:
        # Check if expected topics appear in response
        return all(topic in response.lower() for topic in expected["expected_topics"])
```

### 3. A/B Testing
```python
variant_a = "Summarize: {text}"
variant_b = "Provide a concise summary of the following text: {text}"

results = {"a": [], "b": []}
for text in test_texts:
    results["a"].append(evaluate(llm.generate(variant_a.format(text=text))))
    results["b"].append(evaluate(llm.generate(variant_b.format(text=text))))
```

## The Prompt Test Harness

Build a system that:
1. Takes a prompt template + test cases
2. Generates responses at temperature=0
3. Validates against expected outputs
4. Reports pass/fail and error types
5. Versions prompt changes (git-friendly)

## 🔴 Senior: What to Test

| Test | Why | Frequency |
|------|-----|-----------|
| Format compliance | Always | Every change |
| Content accuracy | Critical | Every change + weekly |
| Edge cases | Important | Every change |
| Injection resistance | Critical | Every change |
| Performance (latency) | Important | Monitor |
| Cost per call | Important | Monitor |
