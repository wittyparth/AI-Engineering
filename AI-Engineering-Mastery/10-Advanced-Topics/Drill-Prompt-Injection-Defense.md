# Drill: Prompt Injection Defense

**Time**: 30 min  **Difficulty**: Hard

## Task

Build a multi-layer defense system against prompt injection:

```python
class InjectionDefense:
    def __init__(self):
        self.layers = []

    async def defend(self, user_input: str, system_prompt: str) -> tuple[str, bool]:
        """Returns (processed_input, blocked)."""
        # Layer 1: Pattern matching (fast)
        if self._matches_known_patterns(user_input):
            return ("Input blocked: suspicious pattern detected", True)

        # Layer 2: Delimiter-based separation
        safe_input = self._wrap_with_delimiters(user_input)

        # Layer 3: LLM-based classification
        classification = await self._classify_input(user_input)
        if classification == "malicious":
            return ("Input blocked: LLM classified as malicious", True)

        # Layer 4: Output guardrail
        response = await self._generate(safe_input, system_prompt)
        safe_response = await self._check_output(response)

        return (safe_response, False)

    def _matches_known_patterns(self, text: str) -> bool:
        patterns = [
            r"ignore.*instructions",
            r"skip.*previous",
            r"you are now",
            r"system.*prompt",
            r"DAN|do anything now",
        ]
        return any(re.search(p, text, re.IGNORECASE) for p in patterns)

    def _wrap_with_delimiters(self, text: str) -> str:
        return f"\n[USER_INPUT_START]\n{text}\n[USER_INPUT_END]\n"

    async def _classify_input(self, text: str) -> str:
        response = await self.classifier.generate(
            f"Classify as safe/suspicious/malicious:\n{text}"
        )
        return response.strip().lower()
```

## Test Your Defense

Create 15 test inputs:
- 5 normal queries (should pass)
- 5 injection attempts (should be blocked)
- 5 borderline cases (test your false positive rate)

Report: precision, recall, false positive rate.
