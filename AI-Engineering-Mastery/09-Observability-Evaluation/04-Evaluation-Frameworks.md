# Evaluation Frameworks

## The Eval Pyramid

```
            ┌─────────────────────┐
            │  Online Evaluation   │ ← A/B testing, user feedback
            ├─────────────────────┤
            │  Production Metrics  │ ← Latency, cost, error rate
            ├─────────────────────┤
            │  Regression Testing  │ ← Automated, per-deployment
            ├─────────────────────┤
            │  Offline Evaluation  │ ← RAGAS, LLM-as-judge
            ├─────────────────────┤
            │  Unit Tests          │ ← Prompt format, tool selection
            └─────────────────────┘
```

## Building an Eval Pipeline

```python
class EvalPipeline:
    def __init__(self):
        self.test_cases = self._load_test_cases()
        self.results = []

    async def run(self, system) -> EvalReport:
        for case in self.test_cases:
            result = await self._evaluate(system, case)
            self.results.append(result)

        return EvalReport(
            total=len(self.results),
            passed=sum(1 for r in self.results if r.passed),
            failed=[r for r in self.results if not r.passed],
            metrics=self._aggregate_metrics(),
        )

    async def _evaluate(self, system, case) -> EvalResult:
        response = await system.answer(case["question"])
        scores = {
            "faithfulness": await self._faithfulness(response, case["contexts"]),
            "relevancy": await self._relevancy(case["question"], response),
            "accuracy": await self._accuracy(response, case["expected"]),
        }
        return EvalResult(
            question=case["question"],
            response=response,
            scores=scores,
            latency=response.latency,
            cost=response.cost,
            passed=all(s >= case["threshold"] for s in scores.values()),
        )
```

## Regression Detection

```python
class RegressionDetector:
    def __init__(self, baseline: dict, threshold: float = 0.05):
        self.baseline = baseline
        self.threshold = threshold

    def check(self, current: dict) -> list[str]:
        regressions = []
        for metric, value in current.items():
            if metric in self.baseline:
                drop = (self.baseline[metric] - value) / self.baseline[metric]
                if drop > self.threshold:
                    regressions.append(
                        f"{metric} dropped by {drop:.1%}: "
                        f"{self.baseline[metric]:.3f} → {value:.3f}"
                    )
        return regressions
```

## Drill: Build Eval Pipeline

Take your Phase 4 RAG project and:
1. Create 30 eval test cases (with ground truth)
2. Implement faithfulness, relevancy, and accuracy metrics
3. Run eval against your RAG system
4. Make one change (chunk size, top-k, prompt) and re-run
5. Report which change improved which metric
