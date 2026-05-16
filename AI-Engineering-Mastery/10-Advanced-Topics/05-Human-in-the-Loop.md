# Human-in-the-Loop

## When to Involve Humans

| Scenario | Automation | Human Review |
|----------|-----------|--------------|
| "What's the weather?" | ✅ Full auto | Never |
| "Approve this loan" | ⚠️ Suggest | ✅ Required |
| "Diagnose this patient" | ⚠️ Suggest | ✅ Required |
| "Summarize a meeting" | ✅ Auto with review | Optional |
| "Write production code" | ⚠️ Generate draft | ✅ Required |

## HITL Architecture

```python
class HumanInTheLoop:
    def __init__(self, auto_threshold: float = 0.95):
        self.auto_threshold = auto_threshold
        self.pending_reviews = Queue()

    async def process(self, task: dict) -> dict:
        # 1. Auto-generate
        result = await self.ai_generate(task)

        # 2. Confidence check
        confidence = await self.estimate_confidence(result)

        if confidence >= self.auto_threshold:
            return {"status": "auto", "result": result}

        # 3. Escalate to human
        review_id = await self.create_review(task, result)
        return {"status": "pending_review", "review_id": review_id}

    async def submit_review(self, review_id: str, approved: bool, edits: str = None):
        if approved:
            return {"status": "approved"}
        else:
            # Human provided corrections
            return {"status": "corrected", "edits": edits}

    async def estimate_confidence(self, result: dict) -> float:
        """Score how confident we are in the result."""
        scores = []
        # Model confidence
        scores.append(result.get("logprobs", 0.5))
        # Eval score (if available)
        scores.append(await self.quick_eval(result))
        # Fallback: 0.5
        return sum(scores) / len(scores) if scores else 0.5
```

## Feedback Loop

```python
class FeedbackLoop:
    """Learn from human corrections to reduce future human involvement."""

    def __init__(self):
        self.corrections = []

    async def record_correction(self, task: dict, ai_output: str, human_output: str):
        self.corrections.append({
            "task": task,
            "ai": ai_output,
            "human": human_output,
            "timestamp": datetime.utcnow(),
        })

        # Fine-tune the model on corrections
        if len(self.corrections) >= 100:
            await self.fine_tune_on_corrections()

    async def should_escalate(self, task: dict) -> bool:
        """Decide if a task needs human review based on past corrections."""
        similar_corrections = self._find_similar(task)
        if not similar_corrections:
            return True  # No history → escalate
        correction_rate = sum(c["different"] for c in similar_corrections) / len(similar_corrections)
        return correction_rate > 0.2  # Escalate if model is wrong >20% of the time
```

## 🔴 Senior: Designing HITL Systems

1. **Default to automate, escalate on uncertainty**: Never start with "all tasks need human review"
2. **Make review easy**: Show the AI's reasoning, highlight where it's uncertain
3. **Track agreement rate**: If AI and humans agree 99% of the time, the human is probably unnecessary
4. **Learn from corrections**: Every human edit is a training example for future improvement
5. **Measure human time**: If humans spend >30s per review, the system is too slow
