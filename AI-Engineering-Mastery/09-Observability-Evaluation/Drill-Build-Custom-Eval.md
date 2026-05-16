# Drill: Build a Custom Eval

**Time**: 30 min  **Difficulty**: Medium

## Task

Build an evaluation metric specifically for YOUR use case.

## Example: Citation Accuracy Eval

```python
async def eval_citation_accuracy(response: str, context: list[str]) -> float:
    """Check if every factual claim in the response is supported by the context."""
    # 1. Extract claims from response
    claims = await extract_claims(response)

    # 2. For each claim, check if it's supported by context
    supported = 0
    for claim in claims:
        # Use LLM-as-judge
        verdict = await llm.generate(
            f"Context: {' '.join(context)}\n"
            f"Claim: {claim}\n"
            f"Is this claim directly supported by the context? Answer YES or NO."
        )
        if "YES" in verdict.upper():
            supported += 1

    return supported / len(claims) if claims else 1.0
```

## Your Eval Challenge

Pick ONE metric that matters for your system and build a custom eval:
- **For RAG**: "What percentage of answers include an unsupported claim?"
- **For Agents**: "Did the agent pick the correct tool on the first try?"
- **For Chatbots**: "How many turns until the user's issue is resolved?"
- **For Code Gen**: "Does the generated code compile and pass tests?"

## Deliverable

```python
class CustomEval:
    name = "my_custom_metric"

    async def score(self, input: dict, output: str) -> float:
        # Your logic here
        pass

    async def explain(self, input: dict, output: str) -> str:
        # Explain why the score is what it is
        pass
```

Test it against 10 cases and report: average score, variance, and edge cases where it fails.
